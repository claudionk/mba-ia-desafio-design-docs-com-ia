# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Pronto para implementação (pendências marcadas como *a validar*) |
| **Data** | 2026-10-29 |
| **Responsáveis técnicos** | Bruno (módulo e integração com pedidos), Diego (outbox e worker), Sofia (revisão de segurança) |
| **Base** | [RFC](RFC.md) · [ADRs](adrs/README.md) · [PRD](PRD.md) · [Tracker](TRACKER.md) |

> Convenção: caminhos marcados com *(novo)* são arquivos a criar; todos os demais existem hoje no repositório.

---

## 1. Contexto e motivação técnica

O OMS não tem nenhum mecanismo de notificação externa. Clientes B2B fazem polling em `GET /api/v1/orders` para descobrir mudanças de status, e três deles exigem notificação em menos de 10 s (Marcos, [09:00]–[09:02]).

Toda mudança de status passa por `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), exposto em `PATCH /api/v1/orders/:id/status` (`src/modules/orders/order.routes.ts`). O método roda em `prisma.$transaction` e:

1. carrega o pedido com itens;
2. rejeita transição para o mesmo status (`ConflictError` com código `INVALID_STATUS_TRANSITION`) e transições inválidas (`InvalidStatusTransitionError`, via `canTransition` em `src/modules/orders/order.status.ts`);
3. debita estoque em `PENDING → PAID` ou repõe em `PAID|PROCESSING → CANCELLED`;
4. atualiza `orders.status` e insere em `order_status_history`.

Essa transação é o **único ponto confiável** onde um evento de status pode nascer. As decisões arquiteturais estão em [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md); este documento detalha como implementá-las.

## 2. Objetivos técnicos

| ID | Objetivo | Meta verificável |
| --- | --- | --- |
| OT-01 | Atomicidade entre mudança de status e evento | 0 casos de status alterado sem linha na outbox (teste de rollback) |
| OT-02 | Latência de entrega | Do commit ao 1º envio: ≤ 2 s de polling + tempo de processamento; p95 < 10 s |
| OT-03 | Resiliência a indisponibilidade do cliente | Até 5 retentativas em ≈ 14h36 antes da DLQ |
| OT-04 | Autenticidade e integridade | 100% dos envios com `X-Signature` HMAC-SHA256 |
| OT-05 | Deduplicação possível pelo cliente | `X-Event-Id` idêntico em todas as tentativas e replays do mesmo evento |
| OT-06 | Zero infraestrutura e dependências novas | `package.json` sem novas dependências de runtime; só MySQL existente |

## 3. Escopo e exclusões

**Incluso**

- Módulo `src/modules/webhooks/` *(novo)*: CRUD de configuração, rotação de secret, histórico de entregas, replay de DLQ.
- Tabelas `webhooks`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter` (migration Prisma).
- `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamado dentro de `changeStatus`.
- Entry-point `src/worker.ts` *(novo)* + script `npm run worker`.
- Evento único: `order.status_changed`.

**Excluído** (com origem)

| Item | Motivo | Origem |
| --- | --- | --- |
| Webhooks inbound (cliente → OMS) | Fluxo é só outbound | Marcos [09:02]; Sofia [09:03] |
| E-mail ao cliente após falhas repetidas | Próxima fase | Larissa [09:37] |
| Rate limiting de saída | Observar e decidir depois | Diego [09:39]; Larissa [09:39] |
| Dashboard visual para o cliente | Projeto separado do time de frontend | Larissa [09:40] |
| Arquivamento de linhas entregues (~30 dias) | Fora do escopo desta feature | Diego [09:08] |
| Múltiplos workers / ordenação global | Limitação conhecida; single-worker | Diego [09:12]–[09:13]; Larissa [09:13] |
| Exactly-once | At-least-once decidido | Diego [09:25] |
| Itens do pedido no payload | Payload enxuto; cliente consulta `GET /orders/:id` | Diego [09:43] |
| Evento na **criação** do pedido (`OrderService.create`) | O ponto de integração decidido é `changeStatus`; a criação não é uma mudança de status. Ver RFC Q-11. | Bruno [09:40] |

## 4. Modelo de dados

Acréscimos em `prisma/schema.prisma`, seguindo as convenções do arquivo: IDs `String @id @default(uuid()) @db.Char(36)` (Larissa, [09:51]), `createdAt/updatedAt`, `@@map` em snake_case e `@@index` explícitos.

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  DELIVERED    // entregue
  FAILED       // falhou (movido para webhook_dead_letter)
}

model Webhook {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // array de OrderStatus assinados, ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id])
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @unique @db.Char(36) // enviado em X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)      // "order.status_changed"
  payload       Json                                     // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    Webhook           @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

  @@index([status, createdAt])
  @@index([status, nextAttemptAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  webhookId      String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text   // truncado antes de gravar
  durationMs     Int
  errorCode      String?  @db.VarChar(64)
  createdAt      DateTime @default(now())

  outbox  WebhookOutbox @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook Webhook       @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  outboxId     String    @db.Char(36)
  webhookId    String    @db.Char(36)
  eventId      String    @db.Char(36)
  payload      Json
  reason       String    @db.VarChar(64)   // código WEBHOOK_* da falha final
  lastError    String?   @db.VarChar(500)
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)      // usuário ADMIN (auditoria)

  webhook Webhook @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:

- **Uma linha de outbox por webhook inscrito** em um evento: cada endpoint tem retry, assinatura e `X-Webhook-Id` próprios. O `eventId` é gerado na inserção (Diego, [09:25]).
- `orderId` **sem FK** para `orders`: o histórico de eventos não deve impedir `DELETE /orders/:id` (permitido em `PENDING`/`CANCELLED` por `OrderService.delete`).
- `Customer` ganha a relação `webhooks Webhook[]`.
- `onDelete: Cascade` a partir de `Webhook` é o padrão provisório para `DELETE /webhooks/:id` — *a validar* (RFC Q-10).
- `events` é `Json` (array) porque o MySQL não tem tipo array; o conteúdo é validado por Zod na API.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro de `changeStatus`)

```
PATCH /api/v1/orders/:id/status
  └─ OrderService.changeStatus  ── prisma.$transaction(tx) ───────────────────────────┐
       1. findUnique(order + items)                                                    │
       2. validações (mesmo status / canTransition)  → erro = rollback, sem evento     │
       3. debitStock / replenishStock                                                  │
       4. tx.order.update(status)                                                      │
       5. tx.orderStatusHistory.create(...)                                            │
       6. publishWebhookEvent(tx, order, from, to)                     ◄── NOVO        │
            a. tx.webhook.findMany({ customerId, active: true })                       │
            b. filtra em memória: events inclui `to`                                   │
            c. se nenhum → return (nada inserido)            (Bruno [09:34])           │
            d. para cada webhook: payload = render(order, from, to, eventId=uuid())    │
            e. tx.webhookOutbox.createMany([...])                                      │
            f. logger.info('webhook_event_enqueued', { orderId, count, eventIds })     │
       7. findUnique(refreshed) e retorno                                              │
  ◄─ commit (ou rollback de TUDO se qualquer passo falhar)                  ───────────┘
```

Regras:

- O passo 6 roda **depois** do passo 5 e **antes** do `refreshed`, dentro do mesmo `tx`. Se `createMany` falhar, a exceção sobe e o Prisma desfaz a transação inteira (Bruno, [09:40]; Diego, [09:41]).
- O `payload` usa os dados do `order` já carregado no passo 1 (`id`, `orderNumber`, `customerId`, `totalCents`) mais `from`/`to`. Nenhuma consulta extra de itens.
- `timestamp` do payload = instante da transição (`new Date().toISOString()` no passo 6).
- **Não** verifica o limite de 64 KB aqui: lançar erro neste ponto desfaria a mudança de status. O tamanho é verificado no worker (§5.2).

### 5.2 Processamento pelo worker

`src/worker.ts` *(novo)* faz bootstrap; o loop vive em `src/modules/webhooks/webhook.processor.ts` *(novo)*.

```
startup:
  logger.info('worker_started')
  recoverStale(): UPDATE webhook_outbox SET status=PENDING WHERE status=PROCESSING
     (single-worker ⇒ qualquer PROCESSING no boot é resto de um crash; reenviar é seguro: at-least-once)

loop (a cada POLL_INTERVAL_MS = 2000):
  1. batch = SELECT … WHERE status=PENDING AND nextAttemptAt <= now()
             ORDER BY createdAt ASC LIMIT BATCH_SIZE (10)
  2. UPDATE … SET status=PROCESSING WHERE id IN (batch) AND status=PENDING
  3. agrupa por webhookId; grupos em paralelo, eventos de um mesmo grupo em série (ordem de createdAt)
  4. para cada evento → deliver(event)  (§5.3)
  5. logger.debug('worker_batch_processed', { size, durationMs })
  6. erro de banco no ciclo → logger.error('worker_cycle_failed') e segue no próximo tick

shutdown (SIGINT/SIGTERM):
  para de agendar ticks → aguarda o batch corrente → prisma.$disconnect() → exit(0)
```

`deliver(event)`:

1. Carrega o `Webhook`. Se ele não existe mais ou está `active = false` → DLQ com `WEBHOOK_INACTIVE` (sem retry).
2. Se a secret está vazia → DLQ com `WEBHOOK_SECRET_REQUIRED`.
3. `body = JSON.stringify(event.payload)`; se `Buffer.byteLength(body) > 65_536` → DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem envio e sem retry (Sofia, [09:23]; Diego, [09:24]).
4. Monta os headers (§6.3), com `X-Signature = HMAC-SHA256(secret, body)`.
5. `fetch(url, { method: 'POST', body, headers, signal: AbortSignal.timeout(10_000) })` (Diego, [09:42]).
6. Grava `webhook_deliveries` (attempt, success, status, `durationMs`, corpo da resposta truncado, `errorCode`).
7. Resposta **2xx** → `status = DELIVERED`, `deliveredAt = now()`, log `webhook_delivery_succeeded`.
8. Qualquer outro resultado → fluxo de retry (§5.3).

### 5.3 Retry

`attempts` conta envios realizados. Tabela de espera aplicada **após** cada falha (Diego, [09:17]; Larissa, [09:17]):

| Falha nº (`attempts` após o envio) | Próxima ação | `nextAttemptAt` |
| --- | --- | --- |
| 1 (envio inicial) | retentativa 1 | now + 1 min |
| 2 | retentativa 2 | now + 5 min |
| 3 | retentativa 3 | now + 30 min |
| 4 | retentativa 4 | now + 2 h |
| 5 | retentativa 5 | now + 12 h |
| 6 | **DLQ** com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — |

```ts
// src/modules/webhooks/webhook.constants.ts (novo)
export const POLL_INTERVAL_MS = 2_000;
export const BATCH_SIZE = 10;
export const DELIVERY_TIMEOUT_MS = 10_000;
export const MAX_PAYLOAD_BYTES = 64 * 1024;
export const RETRY_DELAYS_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]; // 1m 5m 30m 2h 12h
export const SECRET_GRACE_PERIOD_MS = 24 * 60 * 60 * 1000;
export const DELIVERIES_HISTORY_LIMIT = 100;
```

Na falha: `attempts += 1`; se `attempts <= RETRY_DELAYS_MS.length`, então `status = PENDING`, `nextAttemptAt = now + RETRY_DELAYS_MS[attempts - 1]`, `lastError = <código>: <mensagem>`, log `webhook_retry_scheduled`. Caso contrário, segue para a DLQ (§5.4).

Classificação: timeout → `WEBHOOK_DELIVERY_TIMEOUT`; não-2xx → `WEBHOOK_DELIVERY_HTTP_ERROR`; erro de rede/DNS/TLS → `WEBHOOK_DELIVERY_NETWORK_ERROR`. Todas são retentáveis.

> Ordem por pedido durante retries (RFC Q-09): no padrão provisório, um evento em espera **não** bloqueia eventos posteriores do mesmo pedido. A ordem é preservada no caminho feliz, mas pode inverter quando há retry. *A validar com Diego/Larissa.*

### 5.4 DLQ e replay

**Mover para DLQ** (em uma transação):

1. `INSERT webhook_dead_letter (outboxId, webhookId, eventId, payload, reason, lastError, failedAt)` (Diego, [09:18]).
2. `UPDATE webhook_outbox SET status = FAILED`.
3. `logger.warn('webhook_moved_to_dead_letter', { eventId, webhookId, reason })`.

**Replay** — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role `ADMIN`):

1. Busca a entrada; se não existe → `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404); se `replayedAt` já está preenchido → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` (409).
2. Se o webhook está inativo → `WEBHOOK_INACTIVE` (409); reprocessar só falharia de novo.
3. Em transação: `webhook_outbox` volta para `status = PENDING`, `attempts = 0` e `nextAttemptAt = now()`, **mantendo o mesmo `eventId`** ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). Na DLQ, `replayedAt = now()` e `replayedById = req.user.id`.
4. `logger.info('webhook_dead_letter_replayed', { deadLetterId, eventId, adminUserId })` — auditoria de quem executou (Sofia, [09:36]).
5. Se o evento falhar de novo, ele percorre todo o ciclo de retry e gera uma **nova** linha na DLQ.

### 5.5 Rotação de secret

1. `POST /api/v1/webhooks/:id/rotate-secret` gera uma nova secret (`crypto.randomBytes(32)` em hex, com prefixo `whsec_`).
2. Em transação: `previousSecret = secret`, `previousSecretExpiresAt = now + 24h`, `secret = nova` (Sofia, [09:21]).
3. Retorna a nova secret **uma única vez** na resposta.
4. Durante a janela, padrão provisório *(a validar — RFC Q-07)*: o worker envia as duas assinaturas em `X-Signature` (`sha256=<nova>,sha256=<antiga>`), e o cliente aceita se qualquer uma bater. Depois de `previousSecretExpiresAt`, só a nova é usada; o worker trata `previousSecret` expirada como inexistente.
5. Uma nova rotação **dentro** da janela substitui `previousSecret` pela secret atual, e a mais antiga deixa de valer na hora. *A validar com a Sofia.*

## 6. Contratos públicos

### 6.1 Convenções

- Base: `/api/v1` (montado em `src/app.ts`). Todos os endpoints exigem `Authorization: Bearer <JWT>` (`authenticate`).
- Gestão de webhooks: **qualquer role autenticada** (Sofia, [09:37]). Replay de DLQ: **`ADMIN`** (Sofia, [09:36]).
- `customerId` vai no **body** (criação) ou na **query** (listagem), e **não** é extraído do JWT, porque o token é de usuário operador, não de cliente (Bruno, [09:32]; Marcos, [09:32]; Larissa, [09:32]). O formato segue `createOrderSchema` e `listOrdersQuerySchema` (`src/modules/orders/order.schemas.ts`).
- A API de gestão usa camelCase (padrão das respostas atuais). O **payload enviado ao cliente** usa snake_case, conforme listado por Diego ([09:43]).
- Erros usam o envelope de `src/middlewares/error.middleware.ts`: `{ "error": { "code", "message", "details?" } }`.
- `secret` **nunca** aparece em listagem nem em `PATCH`; só nas respostas de criação e de rotação.

### 6.2 Endpoints de gestão

#### E1 — `POST /api/v1/webhooks` — cadastrar webhook

Request:

```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "4b1f0c7e-2a51-4d7e-9a43-8f0d2c1b9e10",
  "url": "https://api.atlascomercial.com.br/oms/webhooks",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d",
  "customerId": "4b1f0c7e-2a51-4d7e-9a43-8f0d2c1b9e10",
  "url": "https://api.atlascomercial.com.br/oms/webhooks",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_3f9a1c7d0b8e4f2a6c5d9e1b7a3f0c8d2e6b4a9f1c7d3e5b8a0f2c4d6e9b1a3",
  "createdAt": "2026-11-03T13:02:11.000Z",
  "updatedAt": "2026-11-03T13:02:11.000Z"
}
```

Status: `201`; `400 VALIDATION_ERROR` (URL não-https → detalhe `WEBHOOK_INVALID_URL`; eventos inválidos → detalhe `WEBHOOK_INVALID_EVENT_FILTER`); `401 UNAUTHORIZED`; `404 NOT_FOUND` (customer inexistente, reaproveitando `NotFoundError('Customer')`).

Semântica: a secret é gerada pela plataforma (Marcos, [09:31]). `events` é obrigatório, com pelo menos 1 item, sem duplicatas e com valores de `OrderStatus` **exceto `PENDING`**: nenhuma transição de `order.status.ts` leva a `PENDING`, então esse filtro nunca dispararia.

#### E2 — `GET /api/v1/webhooks?customerId=…` — listar webhooks de um cliente

Request:

```http
GET /api/v1/webhooks?customerId=4b1f0c7e-2a51-4d7e-9a43-8f0d2c1b9e10&page=1&pageSize=20
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK` (formato `paginated()` de `src/shared/http/response.ts`):

```json
{
  "data": [
    {
      "id": "9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d",
      "customerId": "4b1f0c7e-2a51-4d7e-9a43-8f0d2c1b9e10",
      "url": "https://api.atlascomercial.com.br/oms/webhooks",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotationInProgress": false,
      "createdAt": "2026-11-03T13:02:11.000Z",
      "updatedAt": "2026-11-03T13:02:11.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status: `200`; `400 VALIDATION_ERROR` (`customerId` ausente ou não-UUID); `401`.

#### E3 — `PATCH /api/v1/webhooks/:id` — editar webhook

Request:

```http
PATCH /api/v1/webhooks/9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"], "active": true }
```

Response `200 OK`:

```json
{
  "id": "9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d",
  "customerId": "4b1f0c7e-2a51-4d7e-9a43-8f0d2c1b9e10",
  "url": "https://api.atlascomercial.com.br/oms/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "secretRotationInProgress": false,
  "createdAt": "2026-11-03T13:02:11.000Z",
  "updatedAt": "2026-11-05T09:40:00.000Z"
}
```

Status: `200`; `400 VALIDATION_ERROR` (body vazio, URL não-https, eventos inválidos); `401`; `404 WEBHOOK_NOT_FOUND`.

Semântica: campos editáveis são `url`, `events` e `active` (Bruno, [09:33]; estado ativo — Sofia, [09:21]). `customerId` e `secret` não são editáveis (a secret muda só por E5). A mudança vale para transições **futuras**; eventos já na outbox não são refiltrados ([ADR-007](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md)).

#### E4 — `DELETE /api/v1/webhooks/:id` — remover webhook

Request:

```http
DELETE /api/v1/webhooks/9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d
Authorization: Bearer eyJhbGciOi...
```

Response `204 No Content` (sem corpo; mesmo padrão de `DELETE /orders/:id`).

Status: `204`; `401`; `404 WEBHOOK_NOT_FOUND`.

Semântica: remove a configuração. Padrão provisório: cascata sobre outbox, entregas e DLQ do webhook (*a validar* — RFC Q-10).

#### E5 — `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Request:

```http
POST /api/v1/webhooks/9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d/rotate-secret
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK`:

```json
{
  "id": "9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d",
  "secret": "whsec_a8d3f1e9c7b5a2d4f6e8c0b1a3d5f7e9c2b4a6d8f0e1c3b5a7d9f2e4c6b8a0d1",
  "previousSecretExpiresAt": "2026-11-06T09:40:00.000Z"
}
```

Status: `200`; `401`; `404 WEBHOOK_NOT_FOUND`.

Semântica: a secret antiga continua válida por 24 h (Sofia, [09:21]); ver §5.5.

#### E6 — `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Request:

```http
GET /api/v1/webhooks/9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d/deliveries
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK` (últimas 100 tentativas, mais recentes primeiro — Marcos, [09:34]):

```json
{
  "data": [
    {
      "id": "e1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
      "eventId": "7d9e2f4a-1b3c-4d5e-8f60-718293a4b5c6",
      "eventType": "order.status_changed",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "durationMs": 184,
      "errorCode": null,
      "payload": {
        "event_id": "7d9e2f4a-1b3c-4d5e-8f60-718293a4b5c6",
        "event_type": "order.status_changed",
        "timestamp": "2026-11-05T14:32:05.123Z",
        "order_id": "0f6b2a1c-3d4e-4f5a-9b8c-7d6e5f4a3b2c",
        "order_number": "ORD-000123",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "4b1f0c7e-2a51-4d7e-9a43-8f0d2c1b9e10",
        "total_cents": 15990
      },
      "responseBody": "{\"received\":true}",
      "deadLetterId": null,
      "createdAt": "2026-11-05T14:33:07.410Z"
    },
    {
      "id": "b7c8d9e0-f1a2-4b3c-9d4e-5f6a7b8c9d0e",
      "eventId": "7d9e2f4a-1b3c-4d5e-8f60-718293a4b5c6",
      "eventType": "order.status_changed",
      "attempt": 1,
      "success": false,
      "responseStatus": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "payload": { "…": "mesmo snapshot" },
      "responseBody": null,
      "deadLetterId": null,
      "createdAt": "2026-11-05T14:32:06.002Z"
    }
  ]
}
```

Status: `200`; `400 VALIDATION_ERROR` (id não-UUID); `401`; `404 WEBHOOK_NOT_FOUND`.

Semântica: cada linha é uma **tentativa** (sucesso/falha, payload, resposta e tempo de resposta). `payload` vem do snapshot em `webhook_outbox` (join por `outboxId`). `deadLetterId` é preenchido (join por `webhook_dead_letter.outboxId`) quando a tentativa encerrou o ciclo e gerou entrada na DLQ — é por aí que o admin obtém o `:id` do replay, já que uma listagem de DLQ não foi pedida na reunião.

#### E7 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ

Request:

```http
POST /api/v1/admin/webhooks/dead-letter/3e4f5a6b-7c8d-4e9f-a0b1-c2d3e4f5a6b7/replay
Authorization: Bearer <JWT de usuário ADMIN>
```

Response `202 Accepted`:

```json
{
  "deadLetterId": "3e4f5a6b-7c8d-4e9f-a0b1-c2d3e4f5a6b7",
  "eventId": "7d9e2f4a-1b3c-4d5e-8f60-718293a4b5c6",
  "outboxStatus": "PENDING",
  "replayedAt": "2026-11-06T10:00:00.000Z",
  "replayedById": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"
}
```

Status: `202` (o envio é assíncrono, feito pelo worker); `401 UNAUTHORIZED`; `403 FORBIDDEN` (role ≠ `ADMIN`, via `requireRole`); `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`; `409 WEBHOOK_INACTIVE`.

### 6.3 Contrato outbound (OMS → cliente)

Request enviado pelo worker:

```http
POST https://api.atlascomercial.com.br/oms/webhooks
Content-Type: application/json
X-Event-Id: 7d9e2f4a-1b3c-4d5e-8f60-718293a4b5c6
X-Webhook-Id: 9c2d5e8a-0f3b-4a6c-b1d2-7e8f9a0b1c2d
X-Timestamp: 2026-11-05T14:32:06.002Z
X-Signature: sha256=5b1f0e3c9a7d2b8e4f6a1c3d5e7f9b0a2c4e6d8f1a3b5c7d9e0f2a4b6c8d0e1f

{"event_id":"7d9e2f4a-1b3c-4d5e-8f60-718293a4b5c6","event_type":"order.status_changed","timestamp":"2026-11-05T14:32:05.123Z","order_id":"0f6b2a1c-3d4e-4f5a-9b8c-7d6e5f4a3b2c","order_number":"ORD-000123","from_status":"PROCESSING","to_status":"SHIPPED","customer_id":"4b1f0c7e-2a51-4d7e-9a43-8f0d2c1b9e10","total_cents":15990}
```

| Header | Conteúdo | Origem |
| --- | --- | --- |
| `Content-Type` | `application/json` | Diego [09:44] |
| `X-Event-Id` | UUID do evento, estável entre tentativas e replays | Diego [09:25] |
| `X-Webhook-Id` | ID do cadastro do webhook | Sofia [09:44]; Diego [09:45] |
| `X-Timestamp` | Instante do **envio** (ISO 8601), para o cliente detectar replay | Diego [09:44] |
| `X-Signature` | `sha256=` + hex de HMAC-SHA256(secret, corpo bruto) | Sofia [09:20], [09:22] |

| Campo do payload | Tipo | Origem no código |
| --- | --- | --- |
| `event_id` | UUID | gerado em `publishWebhookEvent` |
| `event_type` | `"order.status_changed"` | constante |
| `timestamp` | ISO 8601 (instante da transição) | — |
| `order_id` | UUID | `Order.id` |
| `order_number` | `ORD-NNNNNN` | `Order.orderNumber` (`reserveOrderNumber`) |
| `from_status` / `to_status` | `OrderStatus` | `from` / `to` em `changeStatus` |
| `customer_id` | UUID | `Order.customerId` |
| `total_cents` | inteiro | `Order.totalCents` |

Resposta esperada do cliente: qualquer **2xx** em até 10 s = entregue. O corpo da resposta é ignorado (só gravado, truncado, no histórico). Qualquer outro desfecho = falha retentável.

Verificação no cliente (exemplo para o portal do desenvolvedor):

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

function isValid(rawBody: string, header: string, secret: string): boolean {
  const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
  return header.split(',').some((part) => {
    const sig = part.trim().replace(/^sha256=/, '');
    return sig.length === expected.length && timingSafeEqual(Buffer.from(sig), Buffer.from(expected));
  });
}
```

## 7. Matriz de erros

Erros de API são subclasses de `AppError` (ou de `ConflictError`) e chegam ao cliente pelo error middleware existente. Erros do worker não têm resposta HTTP: viram `errorCode` em `webhook_deliveries`, `reason` em `webhook_dead_letter` e campo `code` nos logs.

| Código | HTTP | Onde | Condição | Tratamento / classe |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | E3, E4, E5, E6 | `:id` inexistente | `WebhookNotFoundError extends AppError` — não dá para usar `NotFoundError`, que fixa o código `NOT_FOUND` |
| `WEBHOOK_INVALID_URL` | 400 | E1, E3 | URL malformada ou não-`https` (Sofia [09:23]) | `.refine()` no schema Zod; chega como `error.code = VALIDATION_ERROR`, com `details: [{ path: "url", message: "WEBHOOK_INVALID_URL: ..." }]` (comportamento de `validate.middleware.ts`) |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | E1, E3 | `events` vazio, duplicado, com valor fora de `OrderStatus` ou com `PENDING` | Mesmo mecanismo Zod → `VALIDATION_ERROR` com detalhe em `events` |
| `WEBHOOK_INACTIVE` | 409 | E7 / worker | Replay de evento de webhook inativo; worker encontra webhook inativo ou removido | API: `ConflictError(..., 'WEBHOOK_INACTIVE')`. Worker: DLQ sem retry |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | E7 | `:id` da DLQ inexistente | `AppError(404)` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | E7 | Entrada da DLQ já reprocessada | `ConflictError` |
| `WEBHOOK_SECRET_REQUIRED` | — | Worker | Webhook sem secret válida no momento de assinar | DLQ sem retry; `logger.error` (indica bug ou dado corrompido) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — | Worker | Corpo > 64 KB (Sofia [09:23]; Diego [09:24]) | DLQ sem envio e sem retry |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | Worker | Sem resposta em 10 s (Diego [09:42]) | Retry (§5.3) |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | — | Worker | Resposta não-2xx | Retry |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | Worker | DNS, conexão recusada, falha de TLS | Retry |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | Worker | 6ª tentativa falhou | DLQ |
| `WEBHOOK_OUTBOX_INSERT_FAILED` | 500 | `PATCH /orders/:id/status` | Falha ao inserir na outbox | `AppError(500)` lançado de `publishWebhookEvent` → **rollback** da mudança de status. Logar com `logger.error` **antes** de lançar, porque o error middleware só loga erros que não são `AppError` |

Códigos existentes reaproveitados sem mudança: `UNAUTHORIZED` (401), `FORBIDDEN` (403, de `requireRole`), `NOT_FOUND` (404, customer inexistente em E1), `VALIDATION_ERROR` (400).

## 8. Estratégias de resiliência

| Aspecto | Estratégia | Origem |
| --- | --- | --- |
| **Isolamento da transação** | Nenhuma I/O externa dentro de `changeStatus`; só um `INSERT` na outbox | Bruno [09:04]; [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **Timeout** | `AbortSignal.timeout(10_000)` por envio | Levantado por Sofia [09:42]; definido por Diego [09:42] |
| **Retries** | 5 retentativas: 1m/5m/30m/2h/12h, agendadas por `nextAttemptAt` (o worker não dorme por evento) | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| **Backoff** | Exponencial com escalonamento fixo (tabela §5.3), sem jitter — com um único worker não há "thundering herd" | Diego [09:17] |
| **Fallback** | DLQ persistente + replay manual por ADMIN. Fallback por e-mail **adiado** | Diego [09:18]; Larissa [09:37] |
| **Crash do worker** | No boot, linhas `PROCESSING` voltam a `PENDING` (reenvio permitido pelo at-least-once) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| **Banco indisponível** | Erro do ciclo é logado; o loop tenta no próximo tick; nenhum estado se perde | — |
| **Cliente lento** | Grupos por `webhookId` processados em paralelo: um cliente lento não atrasa os outros, e a ordem por pedido se mantém no grupo | Diego [09:12] |
| **Restart da API** | Não afeta o worker (processos separados) | Diego [09:11] |
| **Shutdown gracioso** | `SIGINT`/`SIGTERM`: termina o batch, `prisma.$disconnect()`, como em `src/server.ts` | — |
| **Payload anômalo** | > 64 KB não é enviado → DLQ | Sofia [09:23] |

## 9. Observabilidade

O projeto já tem **Pino** (`src/shared/logger/index.ts`) e correlação por `X-Request-Id` (`src/middlewares/request-logger.middleware.ts`). Não há biblioteca de métricas nem de tracing no `package.json`, e a reunião decidiu não adicionar nada novo (Bruno, [09:29]). Por isso, métricas saem de **logs estruturados** e de **consultas SQL**, e o tracing é feito por **IDs de correlação**.

### 9.1 Logs (Pino)

Eventos em snake_case, no mesmo estilo de `server_started` e `http_request`. O worker usa `logger.child({ component: 'webhook-worker' })`.

| Evento | Nível | Campos |
| --- | --- | --- |
| `webhook_event_enqueued` | info | `orderId`, `fromStatus`, `toStatus`, `eventIds[]`, `webhookIds[]` |
| `worker_started` / `worker_shutdown` | info | `pollIntervalMs`, `batchSize` / `signal` |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs` |
| `webhook_delivery_failed` | warn | idem + `errorCode` |
| `webhook_retry_scheduled` | info | `eventId`, `attempt`, `nextAttemptAt` |
| `webhook_moved_to_dead_letter` | warn | `eventId`, `webhookId`, `reason`, `deadLetterId` |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, `adminUserId` (auditoria — Sofia [09:36]) |
| `webhook_secret_rotated` | info | `webhookId`, `userId`, `previousSecretExpiresAt` |
| `worker_cycle_failed` | error | `err` |

**Segurança de logs:** acrescentar `'*.secret'`, `'*.previousSecret'` e `'headers["x-signature"]'` aos `redactPaths` de `src/shared/logger/index.ts`. O motivo é o histórico de vazamento de secret em log (Diego, [09:22]). Payloads de pedido **não** são logados por inteiro, apenas os IDs.

### 9.2 Métricas

| Métrica | Fonte | Uso / alerta |
| --- | --- | --- |
| `webhook_events_enqueued_total` | contagem de `webhook_event_enqueued` | volume |
| `webhook_deliveries_total{result=success\|failure, errorCode}` | logs de entrega / `webhook_deliveries` | taxa de sucesso por cliente |
| `webhook_delivery_duration_ms` (p50/p95/p99) | `durationMs` | clientes lentos |
| `webhook_end_to_end_latency_ms` (p95) | `deliveredAt − createdAt` da outbox (1ª tentativa) | **SLA < 10 s** (Marcos [09:02]) |
| `webhook_outbox_pending_total` | `COUNT(*) WHERE status='PENDING'` | backlog |
| `webhook_outbox_oldest_pending_age_s` | `now − MIN(createdAt) WHERE status='PENDING' AND attempts=0` | **alerta > 60 s ⇒ worker parado ou atrasado** |
| `webhook_dead_letter_total` / não reprocessados | `webhook_dead_letter` | alerta de crescimento; insumo para decidir o e-mail da próxima fase |
| `webhook_events_per_customer_per_minute` | `webhook_event_enqueued` agrupado | insumo para a decisão de rate limiting (RFC Q-01) |

### 9.3 Tracing

Sem OpenTelemetry nesta fase (seria dependência nova). A rastreabilidade ponta a ponta usa IDs de correlação:

1. `PATCH /orders/:id/status` → log `http_request` com `requestId` e caminho contendo o `orderId`.
2. `webhook_event_enqueued` → liga `orderId` a `eventIds[]`.
3. Logs do worker e `webhook_deliveries` → `eventId` em todas as tentativas.
4. Cliente → recebe `X-Event-Id` e `X-Webhook-Id`, e o suporte consegue achar a entrega exata a partir do que o cliente informar.

Consulta típica de suporte: `eventId` → `webhook_deliveries` (todas as tentativas) → `webhook_dead_letter` (se houver).

## 10. Dependências e compatibilidade

| Dependência | Uso | Status |
| --- | --- | --- |
| MySQL 8.0 (`docker-compose.yml`) | Outbox, DLQ, configuração | Existente |
| Prisma 5.22 (`@prisma/client`, `prisma`) | Novos models + migration em `prisma/migrations/` | Existente |
| Node ≥ 20 (`package.json` → `engines`) | `fetch` + `AbortSignal.timeout` nativos; `crypto.createHmac`/`randomBytes` | Existente, sem lib nova |
| `uuid` 11 | `event_id` (`v4`), como em `request-logger.middleware.ts` | Existente |
| `zod` 3.23 | Schemas do módulo | Existente |
| `pino` 9.5 | Logs | Existente |
| Revisão de segurança (Sofia) | ≥ 2 dias úteis antes do deploy | Agendar (Sofia [09:46]) |
| Portal do desenvolvedor (Marcos) | Documentar headers, HMAC, at-least-once | Marcos [09:26], [09:40] |

**Compatibilidade:** nenhuma mudança de contrato em endpoints existentes. A migration só **adiciona** tabelas e a relação `Customer.webhooks`. O `PATCH /orders/:id/status` mantém request e response; o único efeito colateral é a inserção na outbox. O worker precisa das mesmas variáveis de ambiente da API, porque `src/config/env.ts` exige `DATABASE_URL` **e** `JWT_SECRET` na carga do módulo.

## 11. Integração com o sistema existente

| # | Arquivo existente | Integração |
| --- | --- | --- |
| 1 | `src/modules/orders/order.service.ts` | Em `changeStatus`, logo após `tx.orderStatusHistory.create(...)` e antes do `findUnique` de `refreshed`, chamar `await publishWebhookEvent(tx, order, from, to)` (Bruno [09:41]). Não muda o construtor de `OrderService` nem `buildControllers`: é uma função importada de `src/modules/webhooks/webhook.publisher.ts` *(novo)* que recebe o `TxClient` (`Prisma.TransactionClient`), o mesmo tipo já usado por `debitStock`. `create()` e `delete()` não mudam. |
| 2 | `src/modules/orders/order.status.ts` | Não muda. É a referência para os status assináveis: como nenhuma transição leva a `PENDING`, o schema de `events` usa `z.nativeEnum(OrderStatus)` excluindo `PENDING`. |
| 3 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Novas classes no mesmo padrão de `InvalidStatusTransitionError`/`InsufficientStockError`: `WebhookNotFoundError` (404), `WebhookInactiveError` (`ConflictError`), `WebhookDeadLetterNotFoundError` (404), `WebhookDeadLetterAlreadyReplayedError` (`ConflictError`), `WebhookOutboxInsertFailedError` (500). Todas com prefixo `WEBHOOK_` (Bruno [09:28]). |
| 4 | `src/shared/errors/index.ts` | Exporta as novas classes, como já faz com as demais. |
| 5 | `src/middlewares/error.middleware.ts` | **Sem alteração** (Bruno [09:29]): o ramo `err instanceof AppError` já serializa `{ error: { code, message, details } }`. `P2002`/`P2025` do Prisma continuam cobertos. |
| 6 | `src/middlewares/auth.middleware.ts` | Rotas `/webhooks` usam `router.use(authenticate)`, como `order.routes.ts`. A rota de replay usa `requireRole('ADMIN')`, como `user.routes.ts` (Larissa [09:36]). `req.user.id` alimenta `replayedById`. |
| 7 | `src/middlewares/validate.middleware.ts` | `validate({ body, params, query })` com os schemas de `webhook.schemas.ts` *(novo)*. Atenção: `ZodError` vira `VALIDATION_ERROR` (ver matriz, §7). |
| 8 | `src/routes/index.ts` | Adicionar `webhooks: WebhookController` e `adminWebhooks: AdminWebhookController` ao tipo `Controllers` e montar `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildAdminWebhookRouter(...))`. |
| 9 | `src/app.ts` | Em `buildControllers`, instanciar `WebhookRepository` → `WebhookService` → `WebhookController`/`AdminWebhookController`, seguindo o padrão manual de injeção dos demais módulos. |
| 10 | `src/server.ts` | Modelo para `src/worker.ts` *(novo)*: `bootstrap()`, handlers de `SIGINT`/`SIGTERM` com `prisma.$disconnect()` e `logger.fatal({ err }, 'bootstrap_failed')`. |
| 11 | `src/config/database.ts` | O worker importa `prisma` deste módulo. Como é outro processo Node, a avaliação do módulo cria **outra instância** de `PrismaClient` com a mesma `DATABASE_URL` (Bruno [09:30]). |
| 12 | `src/config/env.ts` | Reaproveitado sem mudança (`DATABASE_URL`, `LOG_LEVEL`, `NODE_ENV`). Intervalos e limites ficam em `src/modules/webhooks/webhook.constants.ts` *(novo)*, porque a reunião não pediu parametrização por ambiente. |
| 13 | `src/shared/logger/index.ts` | Acrescentar os `redactPaths` de secret e assinatura (§9.1). O worker usa `logger.child({ component: 'webhook-worker' })`. |
| 14 | `src/shared/http/response.ts` | `paginated()` na listagem E2. |
| 15 | `src/middlewares/request-logger.middleware.ts` | Sem alteração; fornece o `requestId` usado na correlação (§9.3). |
| 16 | `prisma/schema.prisma` | Novos models e enum (§4) e relação `webhooks Webhook[]` em `Customer`; nova migration em `prisma/migrations/`. |
| 17 | `package.json` | Scripts `"worker": "node --env-file=.env dist/worker.js"` e `"worker:dev": "tsx watch --env-file=.env src/worker.ts"` (entry-point *novo*), espelhando `start`/`dev` (Larissa [09:11]). `tsconfig.build.json` já compila todo o `src/`. |
| 18 | `tests/setup.ts` | O `beforeEach` precisa apagar `webhookDeadLetter`, `webhookDelivery`, `webhookOutbox` e `webhook` **antes** de `customer.deleteMany()` (FK para `customers`). |
| 19 | `tests/helpers/factories.ts` | Novo helper `createTestWebhook()`, no estilo de `createTestCustomer()`; `createTestUser({ role: 'ADMIN' })` já existe para os testes de replay. |

Esqueleto da mudança em `changeStatus`:

```ts
// src/modules/orders/order.service.ts — dentro de prisma.$transaction(async (tx) => { ... })
await tx.order.update({ where: { id }, data: { status: to } });
await tx.orderStatusHistory.create({ data: { orderId: id, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null } });

await publishWebhookEvent(tx, order, from, to); // NOVO — falhou? rollback de tudo

const refreshed = await tx.order.findUnique({ /* inalterado */ });
```

```ts
// src/modules/webhooks/webhook.publisher.ts (novo)
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Pick<Order, 'id' | 'orderNumber' | 'customerId' | 'totalCents'>,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  const hooks = await tx.webhook.findMany({ where: { customerId: order.customerId, active: true } });
  const targets = hooks.filter((h) => (h.events as OrderStatus[]).includes(toStatus));
  if (targets.length === 0) return;

  const occurredAt = new Date().toISOString();
  const rows = targets.map((h) => {
    const eventId = uuidv4();
    return {
      eventId,
      webhookId: h.id,
      orderId: order.id,
      eventType: 'order.status_changed',
      payload: {
        event_id: eventId,
        event_type: 'order.status_changed',
        timestamp: occurredAt,
        order_id: order.id,
        order_number: order.orderNumber,
        from_status: fromStatus,
        to_status: toStatus,
        customer_id: order.customerId,
        total_cents: order.totalCents,
      },
    };
  });

  try {
    await tx.webhookOutbox.createMany({ data: rows });
  } catch (err) {
    logger.error({ err, orderId: order.id }, 'webhook_outbox_insert_failed');
    throw new WebhookOutboxInsertFailedError();
  }
  logger.info({ orderId: order.id, eventIds: rows.map((r) => r.eventId) }, 'webhook_event_enqueued');
}
```

Arquivos novos do módulo: `src/modules/webhooks/webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, `webhook.publisher.ts`, `webhook.processor.ts`, `webhook.signer.ts`, `webhook.constants.ts`, além de `src/worker.ts` e `tests/webhooks.test.ts` — todos *(novos)*.

## 12. Critérios de aceite técnicos

| ID | Critério | Como verificar |
| --- | --- | --- |
| CAT-01 | Mudança de status com webhook inscrito cria exatamente 1 linha `PENDING` na outbox por webhook inscrito | Teste de integração (supertest + Prisma), no estilo de `tests/orders.test.ts` |
| CAT-02 | Status não assinado ou webhook inativo → nenhuma linha na outbox | Teste de integração |
| CAT-03 | Falha forçada no insert da outbox → `orders.status` e `order_status_history` inalterados | Teste com mock de `tx.webhookOutbox.createMany` |
| CAT-04 | Transição inválida (409) não gera evento | Reaproveitar o cenário `PENDING -> SHIPPED` de `tests/orders.test.ts` |
| CAT-05 | Worker entrega em ≤ 2 s + latência do cliente, com os 5 headers e assinatura verificável | Teste do processor contra servidor HTTP local |
| CAT-06 | Respostas não-2xx/timeouts geram `nextAttemptAt` = +1m, +5m, +30m, +2h, +12h; a 6ª falha vai para a DLQ | Teste unitário com relógio controlado |
| CAT-07 | `X-Event-Id` idêntico entre tentativas e após replay | Teste do processor |
| CAT-08 | Payload > 64 KB → DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem envio | Teste unitário |
| CAT-09 | URL `http://` → 400 com detalhe `WEBHOOK_INVALID_URL` | Teste de API |
| CAT-10 | Replay por `OPERATOR` → 403; por `ADMIN` → 202 + log de auditoria + outbox `PENDING` | Teste de API |
| CAT-11 | Após rotação, envios carregam assinatura válida para a secret nova e a antiga até +24 h; depois só a nova | Teste unitário do signer |
| CAT-12 | `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, mais recentes primeiro | Teste de API |
| CAT-13 | `secret` ausente das respostas de E2/E3 e dos logs | Teste de API + inspeção de log |
| CAT-14 | Worker reiniciado com linhas `PROCESSING` → reprocessadas | Teste do processor |
| CAT-15 | Suíte existente (`npm test`) continua verde | CI |
| CAT-16 | Revisão de segurança da Sofia aprovada | Checklist de release |

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Transação do `changeStatus` mais lenta com muitos webhooks por cliente | Baixa | Médio | `createMany` em um único `INSERT`; índice `[customerId, active]` |
| Crescimento contínuo da outbox (arquivamento fora do escopo) | Alta | Médio | Índices compostos por `status`; métrica de volume; abrir item para o job de arquivamento (RFC Q-05) |
| Inversão de ordem por pedido em retries | Média | Baixo | Clientes não exigem ordem global (Marcos [09:14]); `from_status`/`to_status` + `timestamp` no payload permitem reconciliar; decisão em RFC Q-09 |
| Duplicatas após crash do worker | Média | Baixo | At-least-once + `X-Event-Id` documentados (Marcos [09:26]) |
| Secret vazada via log/resposta | Média | Alto | Redaction no Pino; secret só em E1/E5; rotação com grace; revisão da Sofia |
| Worker parado sem ninguém perceber | Média | Alto | Alerta em `webhook_outbox_oldest_pending_age_s` > 60 s |
| Cliente lento consumindo o ciclo do worker | Média | Médio | Timeout de 10 s + paralelismo por `webhookId` |
| Rajada de eventos sobrecarregando o cliente | Média | Médio | Fora do escopo; métrica por cliente para decidir (RFC Q-01) |
| SSRF: URL cadastrada apontando para rede interna | Média | Alto | Não discutido na reunião; levar à revisão da Sofia (RFC Q-12; CAT-16) |
