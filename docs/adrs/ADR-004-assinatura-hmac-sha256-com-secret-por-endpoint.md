# ADR-004 — Assinatura HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h

- **Status:** Aceito
- **Data:** 2026-10-29
- **Decisores:** Sofia (Eng. de Segurança), Larissa (Tech Lead), Diego (Eng. Sênior), Bruno (Eng. Pleno), Marcos (PM — secret gerada pela plataforma e documentação no portal, [09:31] e [09:26])
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [FDD](../FDD.md)

## Contexto

Os webhooks enviam dados de pedidos para endpoints **fora da nossa infraestrutura** (o fluxo é apenas outbound — Marcos, [09:02]; Sofia, [09:03]). O cliente precisa conseguir verificar que a requisição veio de nós e que o payload não foi adulterado (Sofia, [09:19]). O time já viu um cliente vazar uma secret em log de aplicação (Diego, [09:22]).

## Decisão

1. **HMAC-SHA256 sobre o corpo do request**, enviado no header `X-Signature`; o cliente recalcula e compara (Sofia, [09:20] e [09:22]).
2. **Secret única por endpoint de webhook**, nunca global da plataforma ("se vaza uma, vaza tudo" — Sofia, [09:21]).
3. A configuração do webhook persiste `url`, `secret`, `customer_id` e estado ativo (Bruno, [09:21]; Sofia, [09:21]).
4. A secret é **gerada pela plataforma** e devolvida na criação do webhook (Marcos, [09:31]).
5. **Rotação via API**: o cliente pede uma nova secret; a antiga continua válida por **24 horas** em paralelo e depois é descartada (Sofia, [09:21]).
6. **TLS obrigatório**: só URLs `https://` são aceitas, validadas no schema Zod — validação de schema, não decisão arquitetural (Sofia, [09:23]).
7. O código de assinatura e de geração de secret passa por **revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy** (Sofia, [09:46]).

Como a assinatura precisa ser recalculada a cada envio, a secret é armazenada de forma recuperável (não pode ser *hash* de mão única). A geração usa o módulo `crypto` nativo do Node (o projeto exige Node ≥ 20 em `package.json`), sem nova dependência.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Secret global da plataforma** | Um vazamento comprometeria todos os clientes (Sofia, [09:21]). |
| **Outro algoritmo de HMAC (ex.: SHA-1)** | SHA-256 é o padrão de mercado e todo cliente sério tem biblioteca para ele (Sofia, [09:20]). |
| **Rotação com corte imediato** | Não daria tempo para o cliente migrar seus sistemas; por isso o grace period de 24 h (Sofia, [09:21]). |
| **Aceitar `http://`** | Exporia dados de pedido em trânsito; recusado com erro de validação (Sofia, [09:23]). |

## Consequências

**Positivas**

- Autenticidade e integridade verificáveis pelo cliente com bibliotecas padrão.
- Raio de impacto de vazamento limitado a um único endpoint; rotação permite reação rápida.

**Negativas**

- O cliente precisa implementar a verificação corretamente (documentação no portal — Marcos, [09:26]).
- A secret em claro no banco vira ativo sensível: não pode aparecer em logs (o `redact` de `src/shared/logger/index.ts` precisa incluir o campo) nem em respostas de listagem.
- O comportamento da assinatura **durante** o grace period (assinar com as duas secrets?) e se o `X-Timestamp` entra na assinatura não foram definidos na reunião — ficam como questões em aberto no RFC, para a revisão da Sofia.

**Trade-off explícito:** mais complexidade de gestão de segredos (uma secret por endpoint, rotação com janela dupla) em troca de **isolamento de vazamentos** e de um mecanismo de verificação que é padrão de mercado.
