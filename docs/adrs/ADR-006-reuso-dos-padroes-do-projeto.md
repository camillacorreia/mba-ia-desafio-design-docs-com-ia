# ADR-006: Webhooks como módulo comum do projeto, reaproveitando os padrões existentes

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [FDD, seção 11](../FDD.md#11-integração-com-o-sistema-existente)

## Contexto

O código já tem convenções estáveis ([09:27] Bruno):

- **Módulos:** cada domínio é um módulo em `src/modules/<domínio>/` com controller, service, repository, routes e schemas (por exemplo `src/modules/orders/`).
- **Erros:** uma hierarquia com `AppError` (`src/shared/errors/app-error.ts`) e classes específicas em `src/shared/errors/http-errors.ts`, como `InsufficientStockError` e `InvalidStatusTransitionError`, com códigos em caixa alta (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`).
- **Tratamento de erro:** o middleware central `src/middlewares/error.middleware.ts` já converte `AppError`, `ZodError` e erros conhecidos do Prisma no envelope `{ error: { code, message, details } }`.
- **Log:** Pino em `src/shared/logger/index.ts`, usado no projeto inteiro.
- **Validação e autorização:** schemas Zod aplicados pelo middleware `validate` (`src/middlewares/validate.middleware.ts`) e autorização por papel com `requireRole` (`src/middlewares/auth.middleware.ts`).

A feature poderia criar estruturas próprias (um logger dedicado, um tratamento de erro próprio, uma pasta fora de `src/modules`), mas o time é pequeno ([09:07] Diego) e a revisão fica mais fácil quando o código novo se parece com o existente.

## Decisão

Reuso máximo do que existe ([09:30] Larissa):

1. O módulo fica em `src/modules/webhooks/`, com a mesma estrutura dos outros ([09:27] Bruno).
2. Erros novos estendem `AppError` ou as classes de `http-errors.ts`, com códigos prefixados por `WEBHOOK_` ([09:29] Larissa). Bruno citou como exemplos `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno); a matriz final de erros fica no FDD.
3. Nenhum logger novo: Pino ([09:29] Bruno).
4. O middleware de erro não muda: ele já trata qualquer `AppError` ([09:29] Bruno).
5. Schemas Zod no padrão dos módulos existentes, inclusive para a regra de URL `https` ([09:23] Sofia).
6. `requireRole('ADMIN')` no endpoint de replay da DLQ ([09:36] Larissa).
7. Identificadores UUID, como em todos os models de `prisma/schema.prisma` ([09:51] Larissa).
8. A integração com pedidos é uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client da transação corrente, chamada por `OrderService.changeStatus`, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

## Alternativas consideradas

**Bibliotecas e estruturas novas para a feature** (logger próprio, tratamento de erro específico, organização fora de `src/modules`). Descartada na fala de Bruno, "não vamos botar nada novo", e na decisão de Larissa de manter o webhook "como módulo igual aos outros" ([09:29] Bruno, [09:30] Larissa).

**Injetar um repository de webhook no `OrderService`.** Considerada por Bruno em [09:41]; preterida pela função que recebe o `tx`, que acopla menos o domínio de pedidos ao de webhooks ([09:41] Diego).

## Consequências

**Positivas**

- Erros do módulo saem no mesmo envelope JSON e com o mesmo formato de log, sem nenhuma mudança no middleware.
- Quem conhece um módulo do projeto entende o de webhooks.
- Revisão mais rápida, o que importa para o prazo de três sprints ([09:46] Larissa).

**Negativas**

- O módulo herda as limitações dos padrões atuais. Por exemplo, o projeto não tem métricas nem tracing distribuído, e esta decisão não adiciona nenhum (ver a seção de observabilidade do FDD).
- O `OrderService` passa a depender de uma função do módulo de webhooks, um acoplamento novo entre domínios, ainda que estreito.

**Trade-off explícito:** prioriza-se consistência e velocidade de entrega em vez de ferramentas especializadas que o projeto ainda não tem.
