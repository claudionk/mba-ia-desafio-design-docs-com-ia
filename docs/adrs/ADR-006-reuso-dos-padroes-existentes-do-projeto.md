# ADR-006 — Reuso dos padrões existentes do projeto (módulo, erros, logger, validação, auth)

- **Status:** Aceito
- **Data:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Segurança — validação `https` no schema Zod e uso do `requireRole` no replay, [09:23] e [09:36])
- **Relacionados:** [FDD — Integração com o sistema existente](../FDD.md#11-integração-com-o-sistema-existente)

## Contexto

A codebase do OMS já tem convenções claras e consistentes:

- **Módulos por domínio** em `src/modules/<dominio>/` com `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts` (ex.: `src/modules/orders/`, `src/modules/customers/`).
- **Erros de domínio** derivados de `AppError` (`src/shared/errors/app-error.ts`), com `statusCode` + `errorCode` em `UPPER_SNAKE_CASE`, como `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) e `InsufficientStockError` (`INSUFFICIENT_STOCK`) em `src/shared/errors/http-errors.ts`.
- **Middleware de erro centralizado** (`src/middlewares/error.middleware.ts`) que já trata `AppError`, `ZodError` e erros conhecidos do Prisma (`P2002`, `P2025`).
- **Validação com Zod** via `validate({ body, query, params })` (`src/middlewares/validate.middleware.ts`).
- **Autenticação/autorização** com `authenticate` e `requireRole(...)` (`src/middlewares/auth.middleware.ts`).
- **Logger Pino** compartilhado (`src/shared/logger/index.ts`) com `redact` de campos sensíveis.
- **Composição manual de dependências** em `buildControllers` (`src/app.ts`) e montagem de rotas em `buildApiRouter` (`src/routes/index.ts`).

## Decisão

**Reuso máximo do que já existe**; o webhook é "um módulo igual aos outros" (Larissa, [09:30]):

1. Novo módulo `src/modules/webhooks/` *(novo)* com controller, service, repository, routes e schemas (Bruno, [09:27]), mais o processor do worker (Bruno, [09:28]).
2. Erros do módulo estendem `AppError` e usam **prefixo `WEBHOOK_`** nos códigos — ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` (Bruno, [09:28]; Larissa, [09:29]).
3. Logs com o **Pino já existente**; nenhuma lib de logging nova (Bruno, [09:29]).
4. O **error middleware não muda**: os erros do módulo chegam como `AppError` e já são serializados no envelope `{ error: { code, message, details } }` (Bruno, [09:29]).
5. Autorização do replay de DLQ com o **`requireRole('ADMIN')` existente** (Larissa, [09:36]).
6. Validação de entrada com schemas **Zod** no padrão de `*.schemas.ts` (Larissa, [09:30]; Sofia, [09:23]).
7. Ponto de integração com pedidos é uma **função pura** `publishWebhookEvent(tx, ...)` chamada pelo `OrderService`, em vez de injetar um repository inteiro (Bruno e Diego, [09:41]).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Estrutura própria para o módulo (ex.: pasta `src/webhooks/` fora de `modules`, hierarquia de erros nova)** | Quebraria a consistência da codebase e duplicaria infraestrutura de erros/validação que já funciona; o time optou explicitamente por seguir o padrão (Bruno, [09:27]; Larissa, [09:30]). |
| **Injetar `WebhookRepository` no `OrderService`** | Aumenta o acoplamento entre módulos; uma função que recebe o `tx` é suficiente (Diego, [09:41]). |
| **Introduzir libs novas (logger, validação, HTTP client)** | "Não vamos botar nada novo" (Bruno, [09:29]); Node ≥ 20 (`package.json`) já oferece `fetch` e `crypto`. |

## Consequências

**Positivas**

- Curva de aprendizado mínima; revisão de código mais fácil.
- Respostas de erro do módulo idênticas às do resto da API.
- Nenhuma dependência nova a auditar (relevante para a revisão de segurança).

**Negativas**

- Herda limitações dos padrões atuais. Exemplo concreto: `validate.middleware.ts` converte qualquer `ZodError` em `ValidationError` com código `VALIDATION_ERROR`; portanto, a validação de `https` feita no schema Zod chega ao cliente como `VALIDATION_ERROR` (com detalhe no campo `url`), e não como um código `WEBHOOK_*` de topo. `NotFoundError` também fixa o código `NOT_FOUND`, então `WEBHOOK_NOT_FOUND` exige uma subclasse própria de `AppError`.
- O worker roda fora do Express e não passa pelo error middleware: precisa tratar e logar seus próprios erros.

**Trade-off explícito:** consistência e velocidade de entrega acima de flexibilidade para desenhar o módulo "do zero".
