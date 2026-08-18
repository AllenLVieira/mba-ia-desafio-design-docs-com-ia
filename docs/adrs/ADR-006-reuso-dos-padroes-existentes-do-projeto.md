# ADR-006 — Reuso dos padrões existentes do projeto

## Status

**Accepted** — decisão fechada por Larissa (DEC-11, `[09:30]`): "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."

> **Nota de contexto — revisão pendente.** A sessão de revisão do documento de design com Bruno e Diego (`[09:50]`) ainda não ocorreu.

## Contexto

O time é pequeno (RES-01, Diego `[09:07]`) e o prazo é de três sprints, com a revisão de segurança de Sofia incluída no fim (DEC-13, `[09:47]`), sob pressão comercial da Atlas Comercial (RES-04).

A codebase tem convenção estabelecida e uniforme. Bruno descreveu (COD-11, `[09:27]`): "A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual." E sobre erros (COD-04, `[09:29]`): "a gente já tem um padrão. Tem classe AppError, classes específicas tipo InsufficientStockError, InvalidStatusTransitionError."

O módulo de webhooks precisa expor os endpoints de configuração pedidos pelo produto — `POST` para cadastro com url, secret gerada pelo sistema, lista de status e customer_id (RF-01); `PATCH` para edição (RF-02); `DELETE` para remoção (RF-03); `GET` para listar os webhooks de um customer (RF-04), com o filtro de status por endpoint (RF-05) — além dos endpoints decididos em [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) e [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md).

## Decisão

**O módulo de webhooks reusa a infraestrutura transversal existente sem introduzir nenhuma dependência nova** (DEC-11 / RES-03). Bruno, sobre o logger (COD-07, `[09:29]`): "já tá no projeto inteiro. Não vamos botar nada novo."

### Mapa de reuso

| Padrão existente | Arquivo (símbolo) | Uso no módulo de webhooks |
|---|---|---|
| Erro de domínio base | `src/shared/errors/app-error.ts` (`AppError`) | classe base dos erros do módulo (COD-04) |
| Especializações HTTP | `src/shared/errors/http-errors.ts` (`NotFoundError`, `ConflictError`, `ValidationError`) | erros como `WebhookNotFoundError` estendem estas classes, como `InsufficientStockError` e `InvalidStatusTransitionError` já fazem (COD-05, COD-06) |
| Barrel de erros | `src/shared/errors/index.ts` | ponto de export das novas classes |
| Middleware de erro | `src/middlewares/error.middleware.ts` (`errorMiddleware`) | **sem alteração** — já trata `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` (COD-08) |
| Autenticação | `src/middlewares/auth.middleware.ts` (`authenticate`) | `router.use(authenticate)` no router de webhooks, como em `order.routes.ts` |
| Autorização por role | `src/middlewares/auth.middleware.ts` (`requireRole`) | `requireRole('ADMIN')` no endpoint de replay da DLQ (DEC-12, COD-09) |
| Validação por rota | `src/middlewares/validate.middleware.ts` (`validate`) | `validate({ body, query, params })` por rota, incluindo a validação de `https` de DEC-09 |
| Schemas Zod | `src/modules/orders/order.schemas.ts` | molde: schema + tipo inferido por `z.infer<>` |
| Logging | `src/shared/logger/index.ts` (`logger`) | singleton Pino existente, sem dependência nova (COD-07) |
| Molde de módulo | `src/modules/orders/` (`order.routes.ts`, `order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.schemas.ts`) | os cinco arquivos equivalentes em `src/modules/webhooks/` (COD-11, DEC-17) |
| Registro de rotas | `src/routes/index.ts` (`buildApiRouter`) | `router.use('/webhooks', buildWebhookRouter(...))` |
| Composição / injeção | `src/app.ts` (`buildControllers`, `buildApp`) | instanciação de repository, service e controller |
| Cliente Prisma | `src/config/database.ts` | origem do `PrismaClient`; o worker instancia o seu próprio (DEC-14, COD-12) |
| Convenções de schema | `prisma/schema.prisma` | `@id @default(uuid()) @db.Char(36)`, `@@map("snake_case")`, timestamps (DEC-19) |
| Configuração validada | `src/config/env.ts` (`envSchema`) | novas variáveis do módulo validadas no boot |
| Isolamento de testes | `tests/setup.ts`, `tests/helpers/factories.ts` | novas tabelas na lista de truncamento; factories no padrão existente |

### Decisões específicas cobertas por este ADR

1. **Módulo em `src/modules/webhooks`**, com os mesmos cinco arquivos dos demais módulos (DEC-17 / COD-11). O arquivo de lógica do worker, `webhook.worker.ts`, é o sexto — ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md).

2. **Códigos de erro com prefixo de domínio** `WEBHOOK_`, replicando o esquema de `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` (COD-15 / TEC-16, Bruno `[09:29]`): `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, entre outros.

3. **Identificadores UUID** nas novas tabelas, seguindo o resto do projeto (DEC-19, Larissa `[09:51]`: "Tudo é uuid.").

4. **Autorização em dois níveis**: o CRUD de configuração de webhook fica acessível a qualquer role autenticada por enquanto (DEC-20, Sofia `[09:37]`); o replay da DLQ exige `ADMIN` (DEC-12).

5. **`customer_id` segue a convenção de rotas existente.** A reunião fechou que ele **não vem do JWT** — "o JWT atual é do usuário operador, não do cliente" (Bruno `[09:32]`), e Larissa encerrou com "o customer_id é passado no body ou no path. Não vem do JWT" (RES-07, `[09:32]`), sem escolher entre as duas formas (ABE-03). A escolha foi feita fora da reunião, na sessão de fechamento dos ADRs, aplicando a convenção real do código: em `src/modules/orders/order.schemas.ts`, `createOrderSchema` declara `customerId: z.string().uuid()` **no body**, e `listOrdersQuerySchema` declara `customerId` opcional **na query string**; em `order.routes.ts`, o path param `/:id` é usado apenas para o id do próprio recurso. Logo: `customerId` no **body** do `POST /webhooks`, na **query** do `GET /webhooks`, e o path reservado ao `:id` do webhook.

## Alternativas Consideradas

**Introduzir bibliotecas ou padrões novos para o módulo** (cliente HTTP com retry embutido, framework de job, logger próprio do worker). Descartada por DEC-11 e reforçada por COD-07: "Não vamos botar nada novo." O contexto de time pequeno e três sprints (RES-01, DEC-13) foi o argumento.

**`customer_id` implícito no JWT** (Marcos, `[09:31]`: "Customer_id implícito do JWT"). Revertida na própria reunião: o JWT existente carrega `{ id, email, role }` de um usuário operador com role `ADMIN` ou `OPERATOR` (`src/middlewares/auth.middleware.ts`, tipo `AuthUser`), não de um cliente. Registrada como contradição na nota de leitura (a) do índice.

**`customer_id` no path** (`POST /customers/:customerId/webhooks`). Descartada por divergir do padrão de rotas de `order.routes.ts`, onde path param identifica o próprio recurso.

## Consequências

### Positivas

- **Nenhuma superfície operacional nova.** Sem dependência adicional no `package.json`, o custo de deploy e manutenção do módulo é o mesmo dos existentes (COD-07).
- **Erros do módulo funcionam sem tocar no middleware.** `errorMiddleware` já converte `AppError` no formato `{ error: { code, message, details } }` (COD-08); as classes novas herdam esse comportamento de graça.
- **Curva de leitura zero para o time.** Quem já lê `src/modules/orders/` lê `src/modules/webhooks/` sem contexto adicional (COD-11).
- **Autorização reusada e já testada.** `requireRole` está em produção; o replay da DLQ não introduz caminho de autorização novo (COD-09).
- **A convenção de rota mantém a API coerente.** Um consumidor que já usa `GET /orders?customerId=...` encontra a mesma forma em `GET /webhooks?customerId=...`.

### Negativas

- **O reuso de `errorMiddleware` cobre só metade do sistema.** COD-08 afirma que o middleware pega os erros do módulo "sem precisar mudar nada" — verdade no caminho HTTP. Mas o worker roda fora do Express, em processo separado (DEC-03), e `errorMiddleware` é um `ErrorRequestHandler`: **nada do que ele faz alcança o worker.** O tratamento de erro do processo de entrega não foi tratado na transcrição.
- **Trade-off explícito: velocidade contra adequação.** Seguir o molde de `orders` entrega rápido dentro de três sprints, mas o molde foi desenhado para CRUD síncrono sobre HTTP. O worker não tem controller, não tem router e não tem request — a estrutura de cinco arquivos não descreve metade do módulo, e o sexto arquivo (`webhook.worker.ts`) fica fora do padrão que se decidiu seguir.
- **`DEC-20` deixa o CRUD com autorização frouxa.** Qualquer usuário autenticado — `ADMIN` ou `OPERATOR` — pode cadastrar, editar e remover webhooks de **qualquer** customer, já que o `customerId` vem do body ou da query e não do token. Sofia aceitou como estado temporário: "Por enquanto sim. Mais pra frente a gente pode endurecer" (`[09:37]`).
- **Erros do Prisma podem vazar como `CONFLICT` genérico.** `errorMiddleware` mapeia `P2002` para 409 `CONFLICT`; uma violação de unicidade em tabela de webhook produzirá esse código em vez de um `WEBHOOK_*` específico, a menos que o service trate antes.
- **A convenção de rota escolhida não foi validada pelos participantes.** ABE-03 ficou aberto na reunião; a resolução aqui é consistente com o código, mas não tem endosso de Larissa, Bruno ou Marcos.

## Pontos em aberto

- **ADI-06 — endurecimento da autorização do CRUD.** Sofia adiou explicitamente (`[09:37]`). Enquanto isso, vale o descrito na consequência acima.
- **Tratamento de erro dentro do worker.** Não tratado na transcrição; `errorMiddleware` não se aplica.
- **Confirmação de ABE-03 com os participantes.** A convenção adotada precisa de aval na sessão de revisão de design.

## Fontes

**Índice da transcrição:** DEC-11, DEC-12, DEC-13, DEC-17, DEC-19, DEC-20 · RF-01, RF-02, RF-03, RF-04, RF-05 · RES-01, RES-03, RES-04, RES-07 · ADI-06 · ABE-03 · COD-04, COD-05, COD-06, COD-07, COD-08, COD-09, COD-11, COD-12, COD-15 · TEC-13, TEC-16 · notas de leitura (a) e (c)-1.

**Arquivos reais:** `src/shared/errors/app-error.ts` (`AppError`), `src/shared/errors/http-errors.ts` (`NotFoundError`, `ConflictError`, `ValidationError`), `src/shared/errors/index.ts`, `src/middlewares/error.middleware.ts` (`errorMiddleware`), `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`, `AuthUser`), `src/middlewares/validate.middleware.ts` (`validate`), `src/shared/logger/index.ts` (`logger`), `src/modules/orders/order.routes.ts` (`buildOrderRouter`), `src/modules/orders/order.schemas.ts` (`createOrderSchema`, `listOrdersQuerySchema`), `src/modules/orders/order.controller.ts`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.repository.ts`, `src/routes/index.ts` (`buildApiRouter`), `src/app.ts` (`buildControllers`, `buildApp`), `src/config/database.ts`, `src/config/env.ts` (`envSchema`), `prisma/schema.prisma`, `tests/setup.ts`, `tests/helpers/factories.ts`.

**Relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) (o arquivo que foge do molde), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) (`requireRole` no replay), [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) (validação Zod de `https`).
