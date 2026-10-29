# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) — redação: Claudio Nakayama, com apoio de IA |
| **Status** | Em revisão |
| **Data** | 2026-10-29 |
| **Revisores** | Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Sofia (Eng. de Segurança), Marcos (Product Manager) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B, via **webhooks HTTP outbound**, sempre que o status de um pedido mudar. A mudança de status passa a gravar, **na mesma transação SQL**, um evento numa tabela **outbox no MySQL existente**. Um **worker em processo separado** lê a outbox **a cada 2 s**, envia o evento **assinado com HMAC-SHA256** (secret por endpoint) e, em caso de falha, faz **retry com backoff exponencial (5 tentativas, ≈ 15 h)** antes de mover o evento para uma **DLQ** reprocessável por um admin. A entrega é **at-least-once**, com deduplicação pelo cliente via `X-Event-Id`. Tudo é construído como um módulo a mais da codebase, sem infraestrutura nova. Estimativa: **3 sprints**, incluindo revisão de segurança.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados quando o status dos seus pedidos muda. Hoje eles fazem polling em `GET /orders`, o que torna a integração lenta e cara. A Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre. Para eles, "tempo real" é **qualquer coisa abaixo de 10 s** (Marcos, [09:00]–[09:02]).

O OMS não possui hoje nenhum mecanismo de eventos, fila ou notificação externa. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma transação que já atualiza `orders`, insere em `order_status_history` e movimenta estoque. Qualquer solução precisa:

- **não** degradar nem tornar essa transação dependente de sistemas externos;
- **não** perder eventos de status que foram de fato confirmados;
- caber em um **time pequeno**, sem nova infraestrutura para operar.

## 3. Proposta técnica

### 3.1 Visão geral

```
 PATCH /orders/:id/status
          │
          ▼
 ┌──────────────────────── transação MySQL ───────────────────────┐
 │ orders.update → order_status_history.insert → estoque           │
 │ publishWebhookEvent(tx, …) → webhook_outbox.insert (1 por hook) │
 └─────────────────────────────────────────────────────────────────┘
          │ commit
          ▼
 ┌──────── npm run worker (processo separado) ────────┐
 │ polling 2 s → assina HMAC → POST https://cliente   │──► 2xx: entregue
 │                                                    │──► falha: retry 1m/5m/30m/2h/12h
 └────────────────────────────────────────────────────┘──► esgotou: webhook_dead_letter
                                                              │
                                    POST /admin/.../replay ◄──┘ (ADMIN)
```

### 3.2 Componentes

| Componente | Responsabilidade | Decisão |
| --- | --- | --- |
| **Outbox (`webhook_outbox`)** | Registro atômico do evento junto com a mudança de status; payload salvo como snapshot e filtrado pelos status assinados. | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md) |
| **Worker** | Processo Node separado (`src/worker.ts`, *novo*), polling de 2 s, single-instance, mesmo banco com PrismaClient próprio. | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| **Política de falha** | Timeout de 10 s; 5 retentativas com backoff 1m/5m/30m/2h/12h; DLQ em tabela separada; replay manual restrito a `ADMIN` e auditado. | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| **Segurança** | HMAC-SHA256 do corpo em `X-Signature`; secret única por endpoint, gerada por nós e rotacionável com grace de 24 h; só `https`. | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| **Semântica de entrega** | At-least-once; `X-Event-Id` (UUID) estável para deduplicação no cliente. | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| **API de gestão** | CRUD de webhooks por cliente, rotação de secret, histórico de entregas e replay de DLQ, como módulo `src/modules/webhooks` *(novo)*. | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |

### 3.3 Princípios

1. **A transação de status é a fonte da verdade**: sem commit, sem evento; com commit, o evento existe.
2. **Nenhuma infraestrutura nova**: MySQL, Prisma, Pino, Zod e Express já existentes.
3. **Ordem garantida apenas por pedido e enquanto houver um único worker**; ordenação global não é objetivo (Marcos, [09:14]).
4. **Contrato simples para o cliente**: payload enxuto, headers padronizados (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`), documentação no portal do desenvolvedor.

Os contratos HTTP, o modelo de dados, a matriz de erros e os fluxos passo a passo estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que motivou o descarte | Origem |
| --- | --- | --- | --- |
| ALT-01 | **Disparo HTTP síncrono dentro do `changeStatus`** | Menor latência e nenhum componente novo, **mas** acopla a transação (já pesada) ao tempo de resposta e à disponibilidade do cliente: um cliente lento trava mudanças de status de outros pedidos, e um cliente fora do ar não pode provocar rollback. | Bruno [09:04]; Diego [09:06] |
| ALT-02 | **Redis Streams / fila dedicada** | Escala melhor e é reativo, **mas** exige subir e operar Redis Cluster — *overengineering* para um time pequeno — e não resolve sozinho a atomicidade com a transação SQL. | Larissa [09:07]; Diego [09:07] |
| ALT-03 | **Trigger do MySQL para acordar o worker** | Seria mais reativo que polling, **mas** MySQL não tem `LISTEN/NOTIFY`; trigger só executa SQL, exigindo gambiarras (arquivo, chamada HTTP). Polling de 2 s já atende < 10 s. | Bruno [09:09]; Diego [09:09] |
| ALT-04 | **Retry indefinido** ou **apenas 3 tentativas** | Indefinido deixa eventos pendurados para sempre; 3 tentativas cobrem só ~30 min e não sobrevivem a uma manutenção de 2 h. | Diego [09:15]–[09:16]; Bruno [09:16] |
| ALT-05 | **Exactly-once** | Exigiria coordenação com o cliente e muito mais complexidade; at-least-once + `event_id` resolve ~99% dos casos (padrão Stripe/GitHub). | Diego [09:25] |
| ALT-06 | **Secret global da plataforma** | Mais simples de gerenciar, mas um vazamento comprometeria todos os clientes. | Sofia [09:21] |

## 5. Questões em aberto

### 5.1 Levantadas na reunião e não decididas / adiadas

| # | Questão | Situação | Origem |
| --- | --- | --- | --- |
| Q-01 | **Rate limiting de saída por cliente** (ex.: 50 mudanças em 1 min = 50 chamadas). | Fora do escopo; **observar em produção e decidir depois**. | Diego [09:38]–[09:39]; Larissa [09:39] |
| Q-02 | **Notificar o cliente (e-mail) quando o webhook falha repetidamente.** | Adiado para a **próxima fase**, após medir impacto. | Marcos [09:37]; Larissa [09:37] |
| Q-03 | **Escalar para múltiplos workers** sem perder ordem por pedido (particionar por `order_id` ou lock pessimista). | "Problema do futuro"; hoje single-worker, limitação documentada. | Diego [09:12]–[09:13]; Larissa [09:13] |
| Q-04 | **Endurecer permissões do CRUD de webhooks** (hoje qualquer role autenticada). | "Mais pra frente a gente pode endurecer." | Sofia [09:37] |
| Q-05 | **Arquivamento de linhas entregues** da outbox (~30 dias). | Fora do escopo desta feature; precisa de dono e prazo. | Diego [09:08] |
| Q-06 | **Leitura de "5 tentativas"**: o documento adota 1 envio inicial + 5 retentativas (1m/5m/30m/2h/12h ≈ 14h36). Confirmar com Diego/Larissa. | A confirmar na revisão do RFC. | Diego [09:17]; Larissa [09:48] |

### 5.2 Identificadas na revisão do desenho (não discutidas na reunião)

Estas questões **não** foram decididas por ninguém. O FDD registra um comportamento padrão provisório, marcado como "a validar".

| # | Questão | Dono sugerido |
| --- | --- | --- |
| Q-07 | **Assinatura durante o grace period de 24 h**: enviar duas assinaturas (secret nova e antiga) no `X-Signature` ou só a nova? Sem as duas, a "validade paralela" da secret antiga não tem efeito prático para o cliente. | Sofia |
| Q-08 | **`X-Timestamp` fora da assinatura**: a decisão foi assinar só o corpo, então o timestamp pode ser alterado por um atacante. Incluir o timestamp no material assinado? | Sofia |
| Q-09 | **Ordem por pedido durante retries**: se o evento `PAID` de um pedido está aguardando retry e o `PROCESSING` do mesmo pedido fica pendente, o worker deve segurar o segundo (head-of-line) ou enviá-lo antes? | Diego / Larissa |
| Q-10 | **Retenção após `DELETE` do webhook**: histórico de entregas, eventos pendentes e DLQ do webhook removido são apagados em cascata ou preservados? | Bruno / Sofia |
| Q-11 | **Criação do pedido**: `OrderService.create` grava o status inicial `PENDING` sem passar por `changeStatus`. Os clientes querem ser avisados de pedido novo? Hoje não há evento. | Marcos |
| Q-12 | **SSRF**: a URL cadastrada pode apontar para a rede interna; bloquear IPs privados/loopback além de exigir `https`? | Sofia |

## 6. Impacto e riscos

**Impacto no sistema existente**

- `OrderService.changeStatus` ganha um passo a mais na transação (inserção na outbox); a latência do `PATCH /orders/:id/status` aumenta marginalmente.
- Três tabelas novas no MySQL (configuração, outbox, DLQ) e uma de histórico de entregas, via migration Prisma.
- Um novo processo para implantar (`npm run worker`).
- Nenhuma mudança de contrato nos endpoints existentes.

**Riscos principais** (detalhamento e mitigação no [PRD](PRD.md#10-riscos-e-mitigação) e no [FDD](FDD.md#13-riscos-e-mitigação))

| Risco | Mitigação proposta |
| --- | --- |
| Prazo: Atlas quer até o fim de novembro; estimativa de 3 sprints. | Escopo enxuto (sem e-mail, dashboard ou rate limiting); revisão de segurança já reservada no cronograma (Larissa [09:46]; Sofia [09:46]). |
| Worker parado acumula eventos na outbox. | Eventos não se perdem (outbox persistente); alertar sobre a idade do evento pendente mais antigo. |
| Rajadas de eventos sobrecarregando o cliente (sem rate limiting). | Observar em produção (Q-01). |
| Cliente não deduplica e processa eventos duas vezes. | Documentação destacada no portal (Marcos [09:26]). |
| Vazamento de secret. | Secret por endpoint, rotação com grace de 24 h, redaction no logger, revisão da Sofia. |

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling de 2 s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com `X-Event-Id`](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Payload enxuto como snapshot na inserção](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md)

## 8. Próximos passos

1. Sessão de revisão deste RFC com Bruno e Diego antes de começar a codar (Larissa [09:50]).
2. Responder Q-06 a Q-12 e atualizar FDD/ADRs.
3. Agendar a revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy (Sofia [09:46], [09:49]).
