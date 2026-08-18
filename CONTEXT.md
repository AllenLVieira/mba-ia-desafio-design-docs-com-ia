# CONTEXT.md — Mapa do Código: Order Management System

> Fonte de verdade para quem escreve design docs da feature de Webhooks de Notificação de Pedidos.
> Cada caminho de arquivo, nome de símbolo e campo de banco descritos aqui foram lidos diretamente do código.

---

## 1. Visão Geral da Aplicação

**Stack**: Node.js ≥ 20, TypeScript (ESM, `"type": "module"`), Express 4.21, Prisma 5.22 (MySQL), pino 9.5, Zod 3.23, jsonwebtoken 9.

**Ponto de entrada**: `src/server.ts` — chama `bootstrap()`, que constrói o app via `buildApp({ prisma })` e inicia o servidor em `env.PORT`. Captura `SIGINT`/`SIGTERM` para graceful shutdown (fecha o servidor HTTP e chama `prisma.$disconnect()`).

**Composição do app** (`src/app.ts`):
- `buildApp(deps: AppDependencies): Express` — monta Express com `express.json({ limit: '1mb' })`, `requestLogger`, rota `/health`, router principal em `/api/v1`, handler 404 (`NotFoundError`), e `errorMiddleware`.
- `buildControllers(prisma)` — instancia todos os repositórios, serviços e controladores com injeção de dependência manual, retorna o objeto `Controllers`.

**Roteamento** (`src/routes/index.ts`):

```typescript
export function buildApiRouter(controllers: Controllers): Router {
  router.use('/auth',      buildAuthRouter(controllers.auth));
  router.use('/users',     buildUserRouter(controllers.users));
  router.use('/customers', buildCustomerRouter(controllers.customers));
  router.use('/products',  buildProductRouter(controllers.products));
  router.use('/orders',    buildOrderRouter(controllers.orders));
  return router;
}
```

Para adicionar o módulo de webhooks basta acrescentar um `router.use('/webhooks', buildWebhookRouter(...))` aqui e registrar o controller em `buildControllers`.

---

## 2. Anatomia de um Módulo (padrão seguido por todos os módulos)

O módulo `src/modules/orders/` é o modelo a ser imitado. Arquivos e responsabilidades:

| Arquivo | Responsabilidade |
|---|---|
| `order.routes.ts` | `buildOrderRouter(controller)` — instala `authenticate`, aplica `validate` por rota, delega para o controller |
| `order.controller.ts` | Classe com métodos `RequestHandler` (arrow functions), try/catch com `next(err)` |
| `order.service.ts` | Regras de negócio, transações Prisma, lança erros de domínio |
| `order.repository.ts` | Acesso a dados via `prisma: PrismaClient`, sem lógica de negócio |
| `order.schemas.ts` | Schemas Zod + tipos TypeScript inferidos (`z.infer<>`) |
| `order.status.ts` | Funções puras para a máquina de estados |

**Padrão de rota** (extraído de `src/modules/orders/order.routes.ts`):
```typescript
export function buildOrderRouter(controller: OrderController): Router {
  const router = Router();
  router.use(authenticate);                    // autenticação em todas as rotas
  router.get('/',      validate({ query: listOrdersQuerySchema }),    controller.list);
  router.get('/:id',   validate({ params: orderIdParamSchema }),      controller.getById);
  router.post('/',     validate({ body: createOrderSchema }),         controller.create);
  router.patch('/:id/status',
    validate({ params: orderIdParamSchema, body: updateOrderStatusSchema }),
    controller.changeStatus);
  router.delete('/:id', validate({ params: orderIdParamSchema }),     controller.delete);
  return router;
}
```

**Padrão de controller** (extraído de `src/modules/orders/order.controller.ts`):
```typescript
changeStatus: RequestHandler = async (req, res, next) => {
  try {
    if (!req.user) throw new UnauthorizedError();
    const updated = await this.orders.changeStatus(req.params.id!, req.body, req.user.id);
    res.status(200).json(updated);
  } catch (err) {
    next(err);
  }
};
```

---

## 3. Ciclo de Vida do Pedido — Máquina de Estados

**Arquivo**: `src/modules/orders/order.status.ts`

```typescript
const transitions: Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>> = {
  [OrderStatus.PENDING]:    [OrderStatus.PAID, OrderStatus.CANCELLED],
  [OrderStatus.PAID]:       [OrderStatus.PROCESSING, OrderStatus.CANCELLED],
  [OrderStatus.PROCESSING]: [OrderStatus.SHIPPED, OrderStatus.CANCELLED],
  [OrderStatus.SHIPPED]:    [OrderStatus.DELIVERED],
  [OrderStatus.DELIVERED]:  [],
  [OrderStatus.CANCELLED]:  [],
};
```

Estados terminais (sem transições de saída): `DELIVERED`, `CANCELLED`.

**Funções exportadas**:
- `canTransition(from: OrderStatus, to: OrderStatus): boolean`
- `allowedTransitions(from: OrderStatus): ReadonlyArray<OrderStatus>`
- `isTerminal(status: OrderStatus): boolean`
- `shouldDebitStock(from, to): boolean` — verdadeiro apenas para `PENDING → PAID`
- `shouldReplenishStock(from, to): boolean` — verdadeiro para `PAID → CANCELLED` e `PROCESSING → CANCELLED`

**Diagrama resumido**:
```
PENDING ──► PAID ──► PROCESSING ──► SHIPPED ──► DELIVERED (terminal)
   │           │           │
   └───────────┴───────────┴──────────────────► CANCELLED (terminal)
```

---

## 4. Método de Mudança de Status — Assinatura e Corpo da Transação

**Arquivo**: `src/modules/orders/order.service.ts`

```typescript
async changeStatus(
  id: string,
  input: UpdateOrderStatusInput,   // { toStatus: OrderStatus; reason?: string }
  userId: string,
): Promise<OrderWithRelations>
```

Tudo ocorre dentro de uma única transação `prisma.$transaction(async (tx) => { ... })`:

1. `tx.order.findUnique({ where: { id }, include: { items: true } })` — busca pedido com itens
2. Valida `from !== to` (lança `ConflictError`) e `canTransition(from, to)` (lança `InvalidStatusTransitionError`)
3. Se `shouldDebitStock(from, to)`: chama `debitStock(tx, order.items)` — verifica estoque e decrementa `product.stockQuantity` em cada produto
4. Se `shouldReplenishStock(from, to)`: chama `replenishStock(tx, order.items)` — incrementa `product.stockQuantity`
5. `tx.order.update({ where: { id }, data: { status: to } })`
6. `tx.orderStatusHistory.create({ data: { orderId, fromStatus, toStatus, changedById, reason } })`
7. `tx.order.findUnique` com include completo (itens + produto, histórico, cliente) — retorno da função

**Um registro de webhook precisaria ser gravado dentro desta transação** (passo 6½, entre a criação do histórico e o return), ou em uma tabela de outbox dentro da mesma transação.

---

## 5. Controle Transacional

O padrão no código é `this.prisma.$transaction(async (tx) => { ... })` onde `tx` é tipado como `Prisma.TransactionClient` (alias local: `type TxClient = Prisma.TransactionClient`).

O cliente `prisma: PrismaClient` chega ao serviço via construtor:
```typescript
constructor(
  private readonly orders: OrderRepository,
  private readonly prisma: PrismaClient,
) {}
```

O `OrderRepository` recebe também `prisma: PrismaClient` e usa fora de transações. Dentro de transações, o service passa `tx` diretamente para as operações.

O método `list` do `OrderRepository` usa `prisma.$transaction([...])` no formato de array (transação interativa) para `findMany` + `count` em paralelo — padrão diferente do callback usado no service.

---

## 6. Auditoria de Mudança de Status

**Modelo Prisma**: `OrderStatusHistory` (tabela: `order_status_history`)

```prisma
model OrderStatusHistory {
  id          String       @id @default(uuid()) @db.Char(36)
  orderId     String       @db.Char(36)
  fromStatus  OrderStatus?                        // null na criação do pedido
  toStatus    OrderStatus
  changedAt   DateTime     @default(now())
  changedById String       @db.Char(36)
  reason      String?      @db.VarChar(500)

  order     Order @relation(fields: [orderId], references: [id], onDelete: Cascade)
  changedBy User  @relation("StatusChangedBy", fields: [changedById], references: [id])

  @@index([orderId])
  @@index([changedAt])
  @@map("order_status_history")
}
```

Gravada dentro da mesma transação da mudança de status. Na criação do pedido, `fromStatus` é `null` e `reason` é `'order created'`.

---

## 7. Tratamento de Erro

### Hierarquia de classes

**Arquivo**: `src/shared/errors/app-error.ts`

```typescript
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly errorCode: string;
  public readonly details: ErrorDetails;   // Record<string, unknown> | unknown[] | undefined
  constructor(message, statusCode, errorCode, details?) { ... }
}
```

**Arquivo**: `src/shared/errors/http-errors.ts`

| Classe | HTTP | `errorCode` padrão | Observações |
|---|---|---|---|
| `BadRequestError` | 400 | `BAD_REQUEST` | |
| `ValidationError` | 400 | `VALIDATION_ERROR` | `details`: array `{ path, message }[]` |
| `UnauthorizedError` | 401 | `UNAUTHORIZED` | |
| `ForbiddenError` | 403 | `FORBIDDEN` | |
| `NotFoundError` | 404 | `NOT_FOUND` | mensagem: `"${resource} not found"` |
| `ConflictError` | 409 | `CONFLICT` | aceita code e details customizados |
| `UnprocessableEntityError` | 422 | `UNPROCESSABLE_ENTITY` | |
| `InvalidStatusTransitionError` | 409 | `INVALID_STATUS_TRANSITION` | extends `ConflictError` |
| `InsufficientStockError` | 422 | `INSUFFICIENT_STOCK` | extends `UnprocessableEntityError` |

**Arquivo de barrel**: `src/shared/errors/index.ts` — re-exporta todas as classes acima.

### Middleware de erro

**Arquivo**: `src/middlewares/error.middleware.ts` — `errorMiddleware: ErrorRequestHandler`

Formato de resposta de erro:
```json
{ "error": { "code": "STRING", "message": "string", "details": ... } }
```

Casos tratados: `AppError` (usa campos da instância), `ZodError` (400 VALIDATION_ERROR), `Prisma.PrismaClientKnownRequestError` (P2002 → 409 CONFLICT, P2025 → 404 NOT_FOUND). Qualquer outro erro: log pino `error`, resposta 500 `INTERNAL_SERVER_ERROR`.

---

## 8. Formato de Resposta HTTP

**Arquivo**: `src/shared/http/response.ts`

Recursos individuais: retornados diretamente como JSON (sem envelope).

Respostas paginadas:
```typescript
export type PaginatedResponse<T> = {
  data: T[];
  pagination: { page: number; pageSize: number; total: number; totalPages: number };
};
// construída por:
paginated<T>(data, page, pageSize, total): PaginatedResponse<T>
```

Erros: `{ error: { code, message, details? } }` (gerenciado por `errorMiddleware`).

---

## 9. Logging

**Arquivo**: `src/shared/logger/index.ts`

Logger pino. Nível controlado por `env.LOG_LEVEL`. Redacts automáticos: `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`. Campos base em todo log: `service: 'order-management-api'`, `env`. Em `development`, usa `pino-pretty`. Exportado como singleton `logger: Logger`.

**Arquivo**: `src/middlewares/request-logger.middleware.ts` — `requestLogger: RequestHandler`

- Gera/herda `X-Request-Id` (uuid v4 ou valor do header `x-request-id`), salva em `req.id`, devolve no header de resposta.
- Loga evento `'http_request'` no `res.finish` com campos: `{ requestId, method, path, statusCode, durationMs, userId }`.

---

## 10. Autenticação e Autorização

**Arquivo**: `src/middlewares/auth.middleware.ts`

```typescript
export type AuthUser = {
  id: string;
  email: string;
  role: 'ADMIN' | 'OPERATOR';
};
// Declarado em express-serve-static-core: Request.user?: AuthUser; Request.id?: string
```

- `authenticate: RequestHandler` — valida header `Authorization: Bearer <token>`, chama `jwt.verify(token, env.JWT_SECRET)`, define `req.user = { id: payload.sub, email, role }`. Lança `UnauthorizedError` em caso de ausência ou token inválido.
- `requireRole(...roles): RequestHandler` — verifica `req.user.role ∈ roles`, lança `ForbiddenError` se não. Pode ser encadeado após `authenticate`.

**Arquivo**: `src/modules/auth/auth.service.ts`

- `login(input: LoginInput): Promise<LoginResult>` — busca usuário por e-mail, `bcrypt.compare`, assina JWT com `{ sub: userId, email, role }` e `expiresIn: env.JWT_EXPIRES_IN`.
- Retorna `LoginResult = { user: PublicUser, tokens: { accessToken, expiresIn, tokenType: 'Bearer' } }`.

**JWT**: asssinado com `env.JWT_SECRET` (min 16 chars), expiração `env.JWT_EXPIRES_IN` (default `8h`). Não há refresh token.

---

## 11. Validação

**Arquivo**: `src/middlewares/validate.middleware.ts` — `validate(schemas: { body?, query?, params? }): RequestHandler`

Aplica schemas Zod em `req.body`, `req.query` e/ou `req.params`. Erros Zod são convertidos para `ValidationError` com array de `{ path, message }`. Aplicado por rota (não globalmente).

**Padrão de schemas** (arquivo `src/modules/orders/order.schemas.ts`):
```typescript
export const updateOrderStatusSchema = z.object({
  toStatus: z.nativeEnum(OrderStatus),
  reason: z.string().max(500).optional(),
});
export type UpdateOrderStatusInput = z.infer<typeof updateOrderStatusSchema>;
```

---

## 12. Banco de Dados (Prisma Schema)

**Arquivo**: `prisma/schema.prisma`

Provider: MySQL. `shadowDatabaseUrl` configurada para migrations.

**Convenções universais**:
- IDs: `@id @default(uuid()) @db.Char(36)`
- Timestamps: `createdAt @default(now())`, `updatedAt @updatedAt` (quando aplicável)
- Nomes de tabela: snake_case com `@@map(...)`
- **Não há soft delete** (nenhum campo `deletedAt` em nenhum modelo)

**Enums**:
```prisma
enum UserRole    { ADMIN  OPERATOR }
enum OrderStatus { PENDING  PAID  PROCESSING  SHIPPED  DELIVERED  CANCELLED }
```

**Modelos e campos relevantes**:

| Modelo | Tabela | Campos principais |
|---|---|---|
| `User` | `users` | `id`, `email` (unique), `passwordHash`, `name`, `role (UserRole)` |
| `Customer` | `customers` | `id`, `email` (unique), `name`, `phone`, `document` (indexed), `address (Json)` |
| `Product` | `products` | `id`, `sku` (unique), `name`, `priceCents (Int)`, `stockQuantity (Int)`, `active (Bool, indexed)` |
| `Order` | `orders` | `id`, `orderNumber` (unique), `customerId`, `status (OrderStatus)`, `subtotalCents`, `discountCents`, `totalCents`, `notes?`, `createdById`, `createdAt` (indexed), `updatedAt` |
| `OrderItem` | `order_items` | `id`, `orderId` (indexed), `productId` (indexed), `quantity`, `unitPriceCents`, `totalCents`; `onDelete: Cascade` de `Order` |
| `OrderStatusHistory` | `order_status_history` | `id`, `orderId` (indexed), `fromStatus? (OrderStatus)`, `toStatus`, `changedAt` (indexed), `changedById`, `reason? (max 500)`; `onDelete: Cascade` de `Order` |
| `OrderNumberSequence` | `order_number_sequence` | `id (Int, @default(1))`, `nextValue (Int, @default(1))` |

---

## 13. Configuração e Variáveis de Ambiente

**Arquivo**: `src/config/env.ts` — schema Zod; `loadEnv()` lê `process.env`, aborta com `process.exit(1)` em caso de erro.

| Variável | Tipo/Validação | Default |
|---|---|---|
| `NODE_ENV` | `'development' \| 'test' \| 'production'` | `'development'` |
| `PORT` | `coerce.number().int().positive()` | `3000` |
| `LOG_LEVEL` | `'fatal'\|'error'\|'warn'\|'info'\|'debug'\|'trace'` | `'info'` |
| `DATABASE_URL` | `string().min(1)` — obrigatória | — |
| `JWT_SECRET` | `string().min(16)` — obrigatória | — |
| `JWT_EXPIRES_IN` | `string()` | `'8h'` |

Variáveis presentes apenas no `.env.example` (não validadas pelo app, usadas pelo Docker/Prisma): `SHADOW_DATABASE_URL`, `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`.

---

## 14. Testes

**Framework**: Vitest 2.1.4 (`vitest.config.ts`).

Configuração:
- `environment: 'node'`
- `include: ['tests/**/*.test.ts']`
- `setupFiles: ['tests/setup.ts']`
- `fileParallelism: false`, `pool: 'forks'`, `singleFork: true` (execução serial em fork único)
- `testTimeout/hookTimeout: 30000ms`

**Setup** (`tests/setup.ts`): `beforeAll` conecta prisma; `beforeEach` trunca todas as tabelas na ordem inversa de FK (`orderStatusHistory` → `orderItem` → `order` → `orderNumberSequence` → `product` → `customer` → `user`); `afterAll` desconecta.

**Padrão**: testes de integração contra banco real via `supertest`. Sem mocks de repositório — o app inteiro é montado via `buildApp({ prisma })` e chamado por HTTP.

**Factories** (`tests/helpers/factories.ts`): `createTestUser`, `createTestCustomer`, `createTestProduct`, `bootstrapAuthenticatedUser` (cria usuário + faz login, retorna `{ user, token }`), `loginAndGetToken`, `getTestApp` (singleton do app de teste).

**Arquivos de teste**: `tests/auth.test.ts`, `tests/orders.test.ts`.

---

## 15. Scripts e Execução

| Script | Comando |
|---|---|
| `dev` | `tsx watch --env-file=.env src/server.ts` |
| `build` | `tsc -p tsconfig.build.json` |
| `start` | `node --env-file=.env dist/server.js` |
| `db:migrate` | `prisma migrate dev` |
| `db:reset` | `prisma migrate reset --force` |
| `db:seed` | `tsx --env-file=.env prisma/seed.ts` |
| `test` | `vitest run` |
| `test:watch` | `vitest` |

**Docker** (`docker-compose.yml`): um único serviço `mysql` (mysql:8.0, porta 3306), com healthcheck via `mysqladmin ping`. Volume `oms_mysql_data`. Nenhum serviço de fila, cache ou worker.

---

## 16. Busca por Mecanismos de Notificação Existentes

**Termos pesquisados** em `src/`, `prisma/` e `package.json`:
`webhook`, `event`, `queue`, `publish`, `subscribe`, `outbox`, `worker`, `job`, `cron`, `retry`, `hmac`, `emit`, `notify`, `notification`, `bull`, `redis`, `amqp`, `rabbitmq`, `kafka`, `sns`, `sqs`.

**Resultado**: **zero ocorrências** em qualquer arquivo do projeto.

O `package.json` não declara nenhuma dependência de fila de mensagens, broker, cache, ou qualquer biblioteca de notificação externa. O `docker-compose.yml` não provisionou nenhum serviço além do MySQL.

**Conclusão**: não existe hoje qualquer mecanismo de notificação externa, evento, fila, publisher, worker, outbox ou webhook no sistema. A feature de Webhooks de Notificação de Pedidos parte do zero.

---

## 17. Pontos de Extensão para a Feature de Webhooks

Lista dos lugares concretos do código onde a feature nova vai encostar, com natureza do acoplamento:

1. **`src/modules/orders/order.service.ts` — método `changeStatus` (linha 126)**
   Acoplamento principal. O registro de intenção de disparo do webhook (ex.: gravação em tabela outbox) precisa acontecer dentro da transação `prisma.$transaction()` já aberta neste método, logo após a criação do `orderStatusHistory`, para garantir atomicidade entre a mudança de status e a promessa de notificação.

2. **`src/modules/orders/order.service.ts` — método `create` (linha 50)**
   Segundo ponto de disparo: o pedido nasce com status `PENDING` e grava o primeiro registro de histórico dentro de outra transação. Um webhook de `order.created` ou `order.status.changed` (para PENDING) teria o mesmo padrão de acoplamento.

3. **`src/modules/orders/order.status.ts`**
   As funções `canTransition`, `isTerminal`, `shouldDebitStock` e `shouldReplenishStock` são puras e reutilizáveis. O módulo de webhooks pode importar `isTerminal` para decidir se é necessário registrar uma última notificação antes de encerrar tentativas de retry.

4. **`src/routes/index.ts` — função `buildApiRouter`**
   Ponto de registro do router de webhooks. Um `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` precisa ser adicionado aqui.

5. **`src/app.ts` — funções `buildControllers` e `buildApp`**
   `buildControllers` precisa instanciar `WebhookRepository`, `WebhookService` e `WebhookController` e passá-los para `buildApiRouter`. Dependências adicionais (ex.: cliente HTTP para dispatch) entram aqui como parâmetros de `AppDependencies`.

6. **`prisma/schema.prisma`**
   Novos modelos (`WebhookSubscription`, tabela de outbox ou tabela de delivery log) precisam ser adicionados ao schema. Seguir as convenções: `@id @default(uuid()) @db.Char(36)`, `@@map("snake_case")`, timestamps `createdAt`/`updatedAt`.

7. **`src/config/env.ts`**
   Novas variáveis de ambiente necessárias para o módulo (ex.: chave para assinatura HMAC dos payloads, timeout de entrega, número máximo de tentativas) devem ser adicionadas ao `envSchema` Zod para serem validadas na inicialização.

8. **`src/shared/errors/http-errors.ts`**
   Erros de domínio do módulo de webhooks (ex.: `WebhookNotFoundError`, `DuplicateSubscriptionError`) devem estender `AppError` ou as classes existentes (`NotFoundError`, `ConflictError`) seguindo a hierarquia já estabelecida.

9. **`tests/setup.ts` — `beforeEach`**
   As novas tabelas do módulo de webhooks precisam ser incluídas na lista de truncamento para manter o isolamento entre testes.

10. **`tests/helpers/factories.ts`**
    Factories para `createTestWebhookSubscription` e para criar pedidos em estados específicos (necessárias para testar o disparo em cada transição) devem seguir o padrão do arquivo existente.
