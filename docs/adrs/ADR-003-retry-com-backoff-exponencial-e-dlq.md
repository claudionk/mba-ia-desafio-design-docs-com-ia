# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e Dead Letter Queue em tabela

- **Status:** Aceito
- **Data:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior), Bruno (Eng. Pleno), Marcos (PM), Sofia (Segurança — permissão de replay)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar lento ou fora do ar. Já houve cliente com indisponibilidade de duas horas em manutenção planejada (Diego, [09:16]). Precisamos insistir o suficiente para cobrir janelas longas de indisponibilidade, sem deixar eventos "pendurados para sempre" quando o cliente sumiu (Diego, [09:15]).

## Decisão

1. **Falha** = timeout de 10 s sem resposta (Diego, [09:42]) ou resposta que não indique sucesso.
2. **Backoff exponencial com teto de 5 tentativas**, nos intervalos **1 min → 5 min → 30 min → 2 h → 12 h**, totalizando ≈ 14h36min (≈ "quase 15 horas") entre a primeira falha e a última tentativa (Diego, [09:15] e [09:17]; Larissa, [09:17]).
3. Esgotadas as tentativas, o evento vira **falha permanente** e é movido para uma **tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp (Diego, [09:18]).
4. **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente (Diego, [09:18]).
5. O replay exige **role `ADMIN`**, reaproveitando o `requireRole` de `src/middlewares/auth.middleware.ts`, e **registra quem executou** para auditoria (Sofia, [09:36]; Larissa, [09:36]).

> Interpretação adotada: os cinco intervalos são aguardados **após** a primeira falha, ou seja, 1 envio inicial + 5 retentativas. Essa leitura é a única compatível com "quase 15 horas entre primeira falha e última tentativa". Está registrada como ponto a confirmar no [RFC](../RFC.md#questões-em-aberto).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Retry indefinido com backoff** | Evento fica pendurado para sempre se o cliente sumiu (Diego, [09:15]). |
| **3 tentativas (mais agressivo)** | Cobriria só ~30 minutos; uma indisponibilidade matinal ou manutenção de 2 h mataria o evento (Bruno, [09:16]; Diego, [09:16]). |
| **Marcar como `failed` na própria outbox (sem tabela de DLQ)** | Polui a leitura da outbox principal; a tabela separada fica mais limpa e serve de evidência para debug e reprocessamento (Larissa, [09:17]; Diego, [09:18]). |

## Consequências

**Positivas**

- Cobre indisponibilidades de até ~15 h sem intervenção humana — aceitável para o negócio (Marcos, [09:17]).
- A DLQ preserva payload e motivo, permitindo diagnóstico e replay controlado.
- Replay restrito a `ADMIN` e auditado.

**Negativas**

- Um evento pode chegar com até ~15 h de atraso; o cliente precisa tolerar isso.
- Replay é manual: sem alerta automático ao cliente (e-mail foi adiado para a próxima fase — Larissa, [09:37]).
- Retries ampliam a chance de entrega duplicada (ver [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)).

**Trade-off explícito:** troca-se a garantia de "nunca desistir" (retry infinito) por **eventos com ciclo de vida finito e observável**, ao custo de exigir ação manual (replay) em falhas longas.
