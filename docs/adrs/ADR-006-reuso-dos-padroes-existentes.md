# ADR-006 — Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00–09:53
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno — Pedidos)
- **Consultados:** Diego (Eng. Sênior — Plataforma), Sofia (Eng. Segurança)

## Contexto

O bloco de [09:27] a [09:30] da reunião tratou de estrutura de código e padrões. A codebase já tem convenções estabelecidas e consistentes entre os cinco módulos existentes (`auth`, `users`, `customers`, `products`, `orders`), e Bruno propôs que o módulo de webhooks siga exatamente o mesmo desenho em vez de introduzir estrutura nova ([09:27]).

Este ADR é o único do conjunto que trata diretamente de convenções de código, e por isso referencia arquivos reais do repositório.

## Decisão

O módulo de webhooks **não introduz padrão, biblioteca ou infraestrutura nova**. Reaproveita o que já existe:

| Padrão existente | Onde está no código | Como é reaproveitado |
| --- | --- | --- |
| Estrutura modular (controller, service, repository, routes, schemas) | `src/modules/orders/`, `src/modules/customers/` | Nova pasta `src/modules/webhooks/` com os mesmos cinco arquivos ([09:27] Bruno) |
| Registro de rotas versionadas | `src/routes/index.ts`, `src/app.ts` | Novo `buildWebhookRouter` registrado sob `/api/v1`, e novo controller adicionado ao tipo `Controllers` e a `buildControllers` |
| Classe base de erro e códigos de erro | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | Novas classes de erro estendendo `AppError` / `NotFoundError` / `ValidationError`, no mesmo estilo de `InsufficientStockError` e `InvalidStatusTransitionError`, com códigos prefixados `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa) |
| Tratamento centralizado de erro | `src/middlewares/error.middleware.ts` | Nenhuma alteração necessária: o middleware já trata `AppError`, `ZodError` e erros do Prisma ([09:29] Bruno) |
| Validação de entrada | `src/middlewares/validate.middleware.ts` + schemas Zod por módulo | `webhook.schemas.ts` no mesmo formato de `order.schemas.ts`, incluindo a validação de URL `https` ([09:23] Sofia) |
| Autenticação e autorização | `src/middlewares/auth.middleware.ts` | `authenticate` no CRUD de configuração; `requireRole('ADMIN')` no endpoint de replay de DLQ ([09:36] Larissa/Sofia) |
| Logging estruturado | `src/shared/logger/index.ts` (Pino), `src/middlewares/request-logger.middleware.ts` | Mesmo logger, sem biblioteca nova ([09:29] Bruno) |
| Entrada de processo | `src/server.ts` | Nova entrada de worker no mesmo formato, com script `npm run worker` em `package.json` ([09:11] Larissa) |
| Acesso a dados | `src/config/database.ts`, `prisma/schema.prisma` | Mesmo Prisma, mesma `DATABASE_URL`, IDs em UUID `@db.Char(36)` e nomes de tabela em snake_case via `@@map` ([09:51] Larissa) |
| Resposta paginada | `src/shared/http/response.ts` | `paginated()` reutilizado na listagem de webhooks e no histórico de entregas |

Regras derivadas:

- **Todos os códigos de erro do módulo usam o prefixo `WEBHOOK_`** ([09:29] Larissa). Exemplos citados na reunião: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno).
- **O endpoint de replay de DLQ exige role `ADMIN`** e registra em log quem executou a operação, para auditoria ([09:36] Sofia). O restante do CRUD de configuração aceita qualquer role autenticada por enquanto ([09:37] Sofia).
- **O worker cria seu próprio `PrismaClient`**, porque o client é por processo, apontando para o mesmo banco ([09:30] Bruno).
- **A integração com o domínio de pedidos é feita por uma função que recebe o client transacional**, `publishWebhookEvent(tx, order, fromStatus, toStatus)`, chamada de dentro do `$transaction` em `src/modules/orders/order.service.ts`, em vez de injetar o repositório de webhooks inteiro no `OrderService` ([09:41] Bruno e Diego).

## Alternativas consideradas

### A1 — Tratar webhooks como serviço/pacote separado, com estrutura e ferramental próprios

- **Trade-off:** isolamento maior e evolução independente, ao custo de duplicar autenticação, logging, tratamento de erro e configuração, além de fragmentar a base para um time pequeno.
- **Motivo do descarte:** a diretriz fechada foi "reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros" ([09:30] Larissa).

### A2 — Injetar o `WebhookRepository` completo no `OrderService`

- **Trade-off:** encaixaria no estilo de injeção de dependência já usado em `buildControllers` (`src/app.ts`), mas acopla o serviço de pedidos ao módulo de webhooks e obriga a propagar o `tx` por dentro do repositório.
- **Motivo do descarte:** "Boa, função pura recebendo o tx. Não precisa injetar repository inteiro" ([09:41] Diego).

## Consequências

### Positivas

- Curva de aprendizado nula para quem já trabalha na codebase.
- Nenhuma dependência nova em `package.json` para o núcleo do módulo.
- O middleware de erro centralizado já cobre os erros novos sem alteração ([09:29] Bruno).
- Consistência de contrato de erro para os clientes B2B: mesmo envelope `{ error: { code, message, details } }` produzido por `src/middlewares/error.middleware.ts`.

### Negativas

- O módulo de webhooks herda também as limitações dos padrões atuais (por exemplo, ausência de rate limiting e de mecanismo de outbox genérico).
- Há acoplamento explícito, ainda que mínimo, entre `src/modules/orders/order.service.ts` e o módulo de webhooks; a alteração no `changeStatus` é um ponto sensível coberto por testes existentes em `tests/orders.test.ts`.
- Manter tudo no mesmo processo de deploy da API significa que uma mudança no módulo de webhooks exige redeploy da API (embora o worker seja processo separado, conforme [ADR-002](./ADR-002-worker-separado-com-polling.md)).

## Decisões relacionadas

- [ADR-002 — Worker em processo separado com polling de 2 segundos](./ADR-002-worker-separado-com-polling.md)
- [ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint](./ADR-004-hmac-sha256-secret-por-endpoint.md)
