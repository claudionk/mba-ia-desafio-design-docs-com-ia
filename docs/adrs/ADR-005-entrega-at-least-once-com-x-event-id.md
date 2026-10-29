# ADR-005 — Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data:** 2026-10-29
- **Decisores:** Diego (Eng. Sênior), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Com outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e retries ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), existem cenários em que o cliente recebe o mesmo evento mais de uma vez — por exemplo, o cliente processa a requisição mas a resposta não chega dentro do timeout de 10 s, ou o worker cai depois do envio e antes de marcar o evento como entregue. O cliente precisa de um meio para identificar duplicatas (Bruno, [09:25]).

## Decisão

1. A garantia de entrega é **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado (Diego, [09:24]).
2. Cada evento recebe um **UUID gerado no momento em que entra na outbox**, enviado no header **`X-Event-Id`** (e no campo `event_id` do payload). O cliente deduplica por esse identificador (Diego, [09:25] e [09:43]).
3. O `X-Event-Id` é **estável entre retentativas e replays** do mesmo evento — é isso que torna a deduplicação possível.
4. O comportamento é documentado com destaque no portal de desenvolvedor (Marcos, [09:26]).

Os IDs seguem o padrão do projeto: UUID em todas as chaves (Larissa, [09:51]), como em `prisma/schema.prisma` (`@default(uuid()) @db.Char(36)`).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Exactly-once** | Exigiria coordenação dos dois lados (protocolo de confirmação/estado compartilhado com o cliente) e fica muito mais complexo; at-least-once com `event_id` resolve ~99% dos casos e é o que Stripe e GitHub fazem (Diego, [09:25]). |
| **At-most-once (sem retry)** | Incompatível com a política de retry decidida em [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md); perderia eventos sempre que o cliente estivesse indisponível. |

## Consequências

**Positivas**

- Nenhum evento confirmado na outbox é perdido por falha transitória.
- Implementação simples do nosso lado; padrão conhecido pelos clientes.

**Negativas**

- Transfere ao cliente a responsabilidade de deduplicar (Sofia, [09:25]).
- Clientes que ignorarem o `X-Event-Id` podem processar o mesmo evento duas vezes.

**Trade-off explícito:** simplicidade e robustez do nosso lado em troca de **exigir idempotência do consumidor**, mitigada por documentação no portal.
