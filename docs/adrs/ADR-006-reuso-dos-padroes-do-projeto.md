# ADR-006 — Reuso dos padrões existentes do projeto (módulo, AppError, Pino, middlewares)

## Status

Aceito
- **Decisores:** Larissa, Bruno, Diego
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)

## Contexto

O OMS já tem padrões consolidados, verificáveis no código:

- Cada domínio é um módulo em `src/modules/<dominio>` com controller, service, repository, routes e schemas — por exemplo `src/modules/orders/order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts` e `order.schemas.ts` ([09:27] Bruno).
- Erros de domínio estendem `AppError` (`src/shared/errors/app-error.ts`) com código textual em `UPPER_SNAKE_CASE`, como `InsufficientStockError` (`INSUFFICIENT_STOCK`) e `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) em `src/shared/errors/http-errors.ts` ([09:28] Bruno).
- O middleware central `src/middlewares/error.middleware.ts` já converte `AppError`, `ZodError` e erros conhecidos do Prisma (`P2002`, `P2025`) no envelope `{ error: { code, message, details? } }` ([09:29] Bruno).
- O logger é Pino (`src/shared/logger/index.ts`), usado no projeto inteiro ([09:29] Bruno).
- Entradas são validadas por schemas Zod aplicados por `validate()` (`src/middlewares/validate.middleware.ts`), e a autorização usa `authenticate`/`requireRole` (`src/middlewares/auth.middleware.ts`) ([09:36] Larissa).

## Decisão

"Reuso máximo do que já existe" ([09:30] Larissa):

1. O webhook vira um módulo `src/modules/webhooks/` com a mesma estrutura dos demais ([09:27] Bruno, [09:30] Larissa).
2. Erros novos estendem `AppError` e usam o prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa).
3. Nenhum logger novo: Pino ([09:29] Bruno). Nenhuma mudança no middleware de erro — ele já trata as subclasses de `AppError` ([09:29] Bruno).
4. Validação por schemas Zod no padrão dos módulos existentes, incluindo a regra de URL `https` ([09:23] Sofia).
5. O endpoint de replay de DLQ reaproveita `requireRole('ADMIN')` ([09:36] Larissa), já usado em `src/modules/users/user.routes.ts`.
6. O worker usa o mesmo banco e a mesma stack, com `PrismaClient` próprio (ADR-002, [09:30] Bruno).

## Alternativas Consideradas

### A. Hierarquia de erros e logger próprios para o módulo de webhooks
- **Por que foi descartada:** duplicaria o tratamento já centralizado em `error.middleware.ts` e fragmentaria os logs; contraria a decisão de reuso máximo ([09:30] Larissa). Bruno foi explícito sobre o logger: "Não vamos botar nada novo" ([09:29] Bruno).

### B. Reaproveitar códigos genéricos (`NOT_FOUND`, `CONFLICT`) sem prefixo
- **Por que não foi adotada:** o time quer identificar a origem do erro pelo código; daí o prefixo `WEBHOOK_` para tudo do módulo ([09:29] Larissa).

## Consequências

### Positivas
- Curva de aprendizado zero para quem já trabalha no OMS; revisão de código mais simples.
- Contrato de erro idêntico em todos os módulos para quem consome a API.
- `error.middleware.ts` não precisa ser alterado.

### Negativas / trade-offs
- Herdamos as limitações atuais. Exemplo: o `validate()` converte toda falha de schema Zod em `VALIDATION_ERROR`, então a recusa de URL `http` aparece como `VALIDATION_ERROR` com detalhe no campo `url` (mensagem `WEBHOOK_INVALID_URL`), e não como código de topo `WEBHOOK_INVALID_URL` (ver matriz de erros do FDD).
- O projeto não tem biblioteca de métricas nem de tracing; a observabilidade do worker parte de logs Pino estruturados e de consultas às próprias tabelas (FDD, seção Observabilidade).
