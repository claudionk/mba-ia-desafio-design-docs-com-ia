# ADR-001 — Padrão Transactional Outbox no MySQL existente

- **Status:** Aceito
- **Data:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Feature:** Sistema de Webhooks de Notificação de Pedidos
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [RFC](../RFC.md)

## Contexto

Clientes B2B precisam ser notificados quando o status de um pedido muda. O ponto de origem desse evento é o método `changeStatus` de `src/modules/orders/order.service.ts`, que já executa, dentro de um único `prisma.$transaction`, três operações: validação da transição (`canTransition` em `src/modules/orders/order.status.ts`), débito/reposição de estoque (`debitStock` / `replenishStock`), `update` em `orders` e `insert` em `order_status_history`.

Duas forças entram em conflito:

1. **Não acoplar a mudança de status à disponibilidade do cliente.** Uma chamada HTTP dentro da transação faria um cliente lento travar mudanças de status de outros pedidos, e um cliente fora do ar não pode provocar rollback da mudança de status (Bruno, [09:04]).
2. **Não perder eventos.** Se o status mudou, o evento tem que existir; se a transação deu rollback, o evento não pode existir (Diego, [09:06]; Bruno, [09:40]).

O time é pequeno e a stack atual é apenas Node.js + MySQL via Prisma (`prisma/schema.prisma`, `docker-compose.yml` sobe só o `mysql:8.0`).

## Decisão

Adotar o **padrão Transactional Outbox** sobre o **MySQL já existente**:

- Criar a tabela `webhook_outbox` no mesmo banco.
- Dentro da **mesma transação** do `changeStatus`, inserir uma linha por webhook interessado no evento, por meio de uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` corrente (Bruno, [09:41]).
- Se a inserção na outbox falhar, a transação inteira faz rollback: não existe status alterado sem evento (Bruno, [09:40]).
- A tabela tem índice por `status` (pendente, processando, falhou, entregue) e por `created_at` (Diego, [09:08]).
- O despacho HTTP é feito por um processo separado (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- Arquivamento de linhas entregues (≈30 dias) fica **fora do escopo** desta feature (Diego, [09:08]).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Disparo síncrono no `OrderService.changeStatus`** | Acopla a latência e a disponibilidade do cliente à transação de status, que já é pesada (orders + history + estoque). Cliente lento trava outras mudanças; cliente offline não pode causar rollback (Bruno, [09:04]; Diego, [09:06]: "Síncrono está fora de questão"). |
| **Redis Streams (ou fila dedicada)** | Exige subir e operar infraestrutura nova (Redis Cluster) para um time pequeno; foi considerado *overengineering*. Além disso, publicar em um broker fora da transação SQL reintroduz o problema de dual-write (Larissa, [09:07]; Diego, [09:07]). |

## Consequências

**Positivas**

- Atomicidade: evento e mudança de status são gravados ou descartados juntos — sem inconsistência possível (Diego, [09:06]).
- Nenhuma infraestrutura nova: mesmo MySQL, mesmo Prisma, mesma `DATABASE_URL`.
- A outbox serve também como trilha de auditoria do que foi emitido.

**Negativas**

- A transação do `changeStatus` fica um pouco maior (um `INSERT` por webhook inscrito).
- A tabela cresce continuamente; sem o job de arquivamento (fora do escopo), o volume precisa ser observado.
- Latência mínima de entrega passa a depender do intervalo de polling do worker (ver ADR-002).

**Trade-off explícito:** troca-se latência mínima (entrega imediata) e escalabilidade de um broker dedicado por **consistência transacional e simplicidade operacional** com a infraestrutura que já existe.
