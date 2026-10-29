# ADR-002 — Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM — validou a latência de 2 s, [09:10])
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Com a outbox no MySQL ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e fazer as chamadas HTTP. Os clientes consideram "tempo real" qualquer entrega abaixo de **10 segundos** (Marcos, [09:02]).

Hoje o projeto tem um único entry-point, `src/server.ts`, que sobe o Express (`buildApp` de `src/app.ts`) e trata `SIGINT`/`SIGTERM` desconectando o Prisma. O Prisma Client é criado em `src/config/database.ts` (`createPrismaClient()` / `prisma`).

## Decisão

1. **Polling em loop a cada 2 segundos**: o worker busca, em lotes pequenos, os eventos pendentes mais antigos (ordenados por `created_at`), processa e atualiza o status (Diego, [09:09]; Larissa, [09:10]). No pior caso, a latência de detecção é de 2 s — aceito pelo time e validado pelo PM ("2 segundos serve, perfeito" — Marcos, [09:10]).
2. **Processo separado da API**: novo entry-point `src/worker.ts` *(novo)* com script `npm run worker` em `package.json`, nos moldes de `src/server.ts` (Diego, [09:11]; Larissa, [09:11]).
3. **Mesmo banco, mesma stack, PrismaClient próprio**: o worker usa a mesma `DATABASE_URL` (validada por `src/config/env.ts`), mas instancia seu próprio `PrismaClient`, porque o client é por processo (Bruno, [09:11] e [09:30]).
4. **Single-worker**: roda uma única instância. A ordem de entrega é implícita por `created_at` e, portanto, por `order_id`, apenas enquanto houver um único worker. Ordenação global **não** é garantida e isso é uma limitação conhecida (Diego, [09:12]; Larissa, [09:13]).
5. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.processor.ts` *(novo)*; `src/worker.ts` só faz bootstrap e shutdown (Bruno, [09:28]).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Trigger do MySQL notificando o worker (reativo)** | MySQL não tem `LISTEN/NOTIFY` como o Postgres; uma trigger só executa SQL e não avisa processo externo. Seria preciso improvisar (escrever arquivo, chamar endpoint) — "fica esquisito". Polling de 2 s já atende ao requisito de < 10 s (Bruno, [09:09]; Diego, [09:09]). |
| **Worker dentro do processo da API** | Se a API reinicia, o worker morre junto; acopla ciclo de vida e consumo de recursos (Diego, [09:11]). |
| **Múltiplos workers em paralelo** | Perde a garantia de ordem por pedido; exigiria particionar por `order_id` ou lock pessimista. Adiado: "problema do futuro" (Diego, [09:13]). |

## Consequências

**Positivas**

- Atende o SLA de < 10 s com folga (2 s de polling + até 10 s de timeout HTTP só se o cliente for lento).
- Reinícios/deploys da API não interrompem entregas, e vice-versa.
- Reaproveita `src/config/database.ts`, `src/config/env.ts` e o logger de `src/shared/logger/index.ts` sem novas dependências.

**Negativas**

- Polling gera consultas constantes ao MySQL mesmo sem eventos (mitigado pelos índices em `status`/`created_at`).
- Single-worker é um ponto único de vazão; se o volume crescer, será preciso revisitar (questão em aberto no RFC).
- Um novo processo para implantar e monitorar.

**Trade-off explícito:** aceita-se latência mínima de 2 s e um teto de vazão de um único worker em troca de **simplicidade** (sem broker, sem trigger improvisada) e de **ordenação por pedido** sem coordenação distribuída.
