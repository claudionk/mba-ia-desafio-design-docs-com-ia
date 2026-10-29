# ADR-007 — Payload enxuto renderizado como snapshot na inserção da outbox, com filtro de eventos na inserção

- **Status:** Aceito
- **Data:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior), Bruno (Eng. Pleno), Marcos (PM — levantou formato do payload e headers, [09:43]–[09:44]), Sofia (Segurança — limite de 64 KB e header `X-Webhook-Id`, [09:23] e [09:44])
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [FDD — Contratos](../FDD.md#6-contratos-públicos)

## Contexto

Três decisões secundárias sobre o conteúdo do evento afetam diretamente o desenho da outbox:

1. **Quando renderizar o payload?** Guardar o payload pronto na inserção ou guardar só `order_id` e montar no envio (Bruno, [09:51])?
2. **O que vai no payload?** Pedido completo com itens ou só o essencial (Marcos, [09:43])?
3. **Onde aplicar o filtro de eventos?** Cada webhook escolhe os status que quer ouvir (Marcos, [09:33]); filtrar na inserção ou no envio (Diego, [09:34])?

## Decisão

1. **Snapshot na inserção**: o payload é renderizado e gravado na `webhook_outbox` dentro da transação do `changeStatus`. Se o pedido mudar depois, o evento continua refletindo o estado do momento da transição (Larissa, [09:52]; Diego e Bruno, [09:52]).
2. **Payload enxuto em JSON**: `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Itens não são enviados**; quem quiser detalhes consulta `GET /orders/:id` (Diego, [09:43]; Bruno, [09:44]).
3. **Limite de 64 KB** por payload; se ultrapassar, o evento **não é enviado** (erro, sem truncamento) — requisito não funcional, não decisão arquitetural (Sofia, [09:23]; Diego, [09:24]; Larissa, [09:24]).
4. **Filtro na inserção**: só se insere linha na outbox para webhooks ativos do cliente que assinam o `to_status`; se nenhum assina, nada é inserido (Bruno, [09:34]; Diego, [09:34]).

Os nomes de status vêm do enum `OrderStatus` de `prisma/schema.prisma` (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) e os campos de origem existem no model `Order` (`orderNumber`, `customerId`, `totalCents`).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Guardar só `order_id` e renderizar no envio** | O evento passaria a refletir o estado do pedido no momento do envio, não da transição — "tem caso esquisito", ainda mais com retries de até ~15 h (Larissa, [09:52]). |
| **Payload completo com itens** | Infla o payload sem necessidade; o cliente pode buscar detalhes via API (Diego, [09:43]). |
| **Truncar payloads acima do limite** | Um evento desse tamanho indica erro; truncar entregaria dado corrompido (Sofia, [09:23]). |
| **Filtrar no momento do envio** | Gravaria linhas inúteis na outbox; filtrar na inserção economiza armazenamento e trabalho do worker (Bruno, [09:34]). |

## Consequências

**Positivas**

- Eventos imutáveis e auditáveis; retries e replays reenviam exatamente o mesmo conteúdo.
- Payload pequeno, rápido de assinar e transmitir.
- Outbox só contém trabalho real.

**Negativas**

- Mais bytes por linha na outbox (JSON armazenado) e mais trabalho dentro da transação do `changeStatus`.
- Mudanças na assinatura de eventos de um webhook valem **apenas para transições futuras**; eventos já enfileirados seguem a regra antiga.
- A verificação de 64 KB **não pode** lançar erro dentro da transação do `changeStatus` (isso desfaria a mudança de status, contrariando [09:04]); ela é aplicada pelo worker, que manda o evento direto para a DLQ (detalhe no FDD).

**Trade-off explícito:** custo de armazenamento e um pouco mais de trabalho na transação em troca de **fidelidade temporal do evento** e de um contrato simples para o cliente.
