# FDD — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Documento** | Feature Design Document — especificação de implementação |
| **Autor** | Larissa — Tech Lead (`[09:50]`: "Eu vou abrir o doc de design da feature") |
| **Status** | **Em revisão** — depende das mesmas duas revisões pendentes da [RFC](RFC.md) |
| **Data** | 2026-08-17 |
| **Base** | [RFC](RFC.md) · [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) a [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md) |
| **Público** | desenvolvedor que vai implementar o módulo `src/modules/webhooks/` |

> **Este documento não redecide o que virou ADR.** Onde uma decisão está fechada, ele referencia e desce ao detalhe de execução. Onde a fonte é omissa, ele **marca a lacuna em vez de preenchê-la em silêncio**.

### Convenção de rastreabilidade

Toda afirmação estruturante deste documento carrega uma marca de origem:

| Marca | Significado |
|---|---|
| **F** | **Fonte direta** — item do índice da transcrição (`DEC`/`RF`/`RNF`/`RES`/`TEC`/`COD`/`ABE`/`ADI`) ou arquivo real do repositório |
| **D** | **Derivado** — consequência necessária de uma fonte (aritmética, convenção do código existente, ou implicação lógica de uma decisão) |
| **P** | **Proposta do FDD** — ninguém decidiu; é escolha deste documento, **endereçada a um aprovador nomeado** e **não é requisito enquanto não for aprovada** |

Tudo que está marcado **D** ou **P** está consolidado no [Registro de derivações e de aprovações pendentes](#registro-de-derivações-e-de-aprovações-pendentes), no fim do documento: cada **D** com a cadeia que o liga a um item de fonte, cada **P** com o aprovador e o fórum onde a decisão cabe. **Nenhum item deste documento fica sem um dos dois.** Se você discorda de algum, é lá que a lista está inteira, e não espalhada pelo texto.

---

## 1. Contexto e motivação técnica

O sistema é um Order Management System em produção — Node.js ≥ 20, TypeScript ESM, Express 4.21, Prisma 5.22 sobre MySQL, Pino 9.5, Zod 3.23 (`package.json`, **F**). Ele **não possui nenhum mecanismo de notificação externa**: a varredura registrada em `CONTEXT.md` §16 procurou `webhook`, `event`, `queue`, `outbox`, `worker`, `retry`, `hmac`, `redis`, `kafka` e correlatos em `src/`, `prisma/` e `package.json` e não encontrou uma ocorrência (**F**). O `docker-compose.yml` sobe apenas o MySQL.

Três clientes B2B pediram notificação de mudança de status de pedido (`TEC-29`, **F**), e a Atlas Comercial sinalizou migração para concorrente se a entrega atrasar (`RES-04`, **F**).

O ponto de acoplamento é uma transação já carregada. Em `src/modules/orders/order.service.ts`, o método `changeStatus` (linha 126) roda tudo dentro de um único `prisma.$transaction(async (tx) => { ... })`: valida a transição com `canTransition`, debita ou repõe estoque, executa `tx.order.update` (linha 158) e `tx.orderStatusHistory.create` (linha 159), e relê o pedido para retorno (linha 169). O registro do evento entra **entre a linha 167 e a linha 169** (**D**, a partir de `COD-13` e `CONTEXT.md` §17.1).

A restrição que define o espaço de solução é de integridade, e é absoluta: "Não pode ter caso de status mudar e evento não sair" (`RF-11`, Bruno `[09:40]`, **F**). Somada à stack fechada no MySQL existente (`RES-01`) e à proibição de dependência nova (`DEC-11` / `RES-03`), ela elimina broker, fila e cliente HTTP de terceiros antes de qualquer discussão de desenho.

### O que este documento resolve que a RFC deixou aberto

A RFC declarou que **dois campos estruturantes não têm origem em fonte nenhuma e a arquitetura os exige**: uma contagem de tentativas e um instante de próxima tentativa na outbox. Sem eles, o worker sabe *o que* está pendente mas não sabe que uma linha aguarda o intervalo de 12 horas de `DEC-05`, e a política de [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) não é executável. Este FDD dá forma a esses campos (`attemptCount`, `nextAttemptAt`) e assume a autoria deles: são **P**, e estão na seção 5.

O mesmo vale para `ABE-04` — o histórico de entregas que `RF-06` exige e que nenhum documento projetou. Sem a tabela, `RF-06` não é implementável; com ela, o FDD passa a ser a origem de uma estrutura que a reunião não discutiu.

---

## 2. Objetivos técnicos

| # | Objetivo | Critério de verificação | Origem |
|---|---|---|---|
| OT-01 | Registrar a promessa de notificação atomicamente com a mudança de status | rollback da transação de `changeStatus` não deixa linha na `webhook_outbox` | `RF-11` / `RNF-04` **F** |
| OT-02 | Entregar por HTTPS com assinatura HMAC-SHA256 verificável pelo cliente | `X-Signature` reproduzível com a secret do endpoint sobre o corpo exato | `DEC-08` / `RF-09` **F** |
| OT-03 | Sobreviver a indisponibilidade de cliente por até ~15 horas | 6 chamadas HTTP distribuídas em 14h36min | `DEC-05` / `RNF-02` **F** |
| OT-04 | Não perder evento cuja entrega falhou definitivamente | toda linha esgotada tem registro correspondente em `webhook_dead_letter` | `DEC-06` **F** |
| OT-05 | Permitir ao cliente deduplicar reenvios | `X-Event-Id` estável entre retentativas **e através do replay** | `DEC-10` / `ADR-005` **F**, extensão ao replay **D** |
| OT-06 | Não introduzir infraestrutura nem dependência nova | `package.json` inalterado; `docker-compose.yml` sem serviço novo além do worker | `RES-01` / `RES-03` **F** |
| OT-07 | Não alterar o comportamento observável de quem não usa webhooks | apenas `changeStatus` ganha dependência de escrita; nenhum outro endpoint muda | **D** de `CONTEXT.md` §17 |

---

## 3. Escopo e exclusões

### No escopo

Módulo `src/modules/webhooks/` com os cinco arquivos do molde (`webhook.routes.ts`, `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.schemas.ts`) mais `webhook.worker.ts` como sexto (`DEC-17` / `COD-11`, **F**; nome do sexto arquivo resolvido fora da reunião — `ABE-02`). Entry point `src/worker.ts` (`COD-10`, **F**). Sete endpoints HTTP (seção 7). Quatro tabelas novas (seção 5). Disparo a partir de `OrderService.changeStatus` (`COD-13`, **F**).

### Fora de escopo — decidido adiar na reunião

Estes têm dono, timestamp e decisão explícita de adiamento. Não são lacunas:

| Item | Conteúdo | Origem |
|---|---|---|
| `ADI-01` | Notificação por email ao cliente em falha repetida | Larissa `[09:37]` **F** |
| `ADI-02` / `ABE-01` | Rate limiting de envios outbound | Diego / Larissa `[09:39]` **F** |
| `ADI-03` | Dashboard visual para clientes | Larissa `[09:40]` **F** |
| `ADI-04` | Arquivamento das linhas entregues após ~30 dias | Diego `[09:08]` **F** |
| `ADI-05` | Múltiplos workers com particionamento ou lock pessimista | Diego `[09:13]` **F** |
| `ADI-06` | Endurecimento da autorização do CRUD | Sofia `[09:37]` **F** |

### Fora de escopo — **não decidido por ninguém**

Categoria diferente da anterior, e a distinção é deliberada: nos itens acima alguém adiou; nestes **a pergunta nunca foi feita**. Implementá-los seria decidir escopo de produto dentro de um documento de implementação; omiti-los sem rótulo seria esconder que a pergunta existe.

| Item | Conteúdo | Consequência de não decidir |
|---|---|---|
| Disparo em `OrderService.create` | `CONTEXT.md` §17.2 identifica o método `create` (linha 50) como segundo lugar onde o pedido nasce em `PENDING` e grava o primeiro `orderStatusHistory`, com padrão de acoplamento idêntico. A reunião tratou só de `changeStatus` (`COD-13`). | **Um cliente que assine `PENDING` não recebe nada quando o pedido nasce.** É comportamento visível ao cliente decidido por omissão. Ver [RFC, questão 7](RFC.md#exigem-decisão-mas-não-travam-o-início). |
| Destino das linhas pendentes quando a assinatura é removida | `RF-03` cria `DELETE /webhooks/:id`. Nenhuma fonte diz o que acontece com linhas de `webhook_outbox` ainda pendentes daquela assinatura. | Ver seção 13, item aberto. O FDD **não** propõe `onDelete: Cascade` aqui — apagaria evidência de entrega. |

---

## 4. Fronteira de processos

A propriedade central do desenho é que a promessa de entrega nasce dentro da transação da API e é cumprida por um processo que não compartilha ciclo de vida com ela (`DEC-03` / `RES-05`, **F**).

```
   processo da API — src/server.ts            │    processo do worker — src/worker.ts
   npm run dev | npm start                    │    npm run worker  (TEC-20 — script inexistente hoje)
                                              │
   buildApp({ prisma })                       │    webhook.worker.ts
   └─ OrderService.changeStatus (linha 126)   │    ┌─ loop de polling, 2s ──────────────┐
      ┌─ prisma.$transaction ──────────────┐  │    │ 1. reivindica lote  → PROCESSING   │
      │  tx.order.update           (158)   │  │    │ 2. POST https + HMAC-SHA256        │
      │  tx.orderStatusHistory.create(159) │  │    │ 3. grava WebhookDelivery           │
      │  publishWebhookEvent(tx, …) (COD-14)│ │    │ 4. DELIVERED | reagenda | DLQ      │
      └────────────────────────────────────┘  │    └────────────────────────────────────┘
         tudo ou nada  (RF-11 / RNF-04)       │    PrismaClient próprio (DEC-14)
                                              │    limite de erro próprio (seção 9.5)
   ══════════════════════ MySQL — as quatro tabelas da seção 5 ═══════════════════════
```

`errorMiddleware` é um `ErrorRequestHandler` do Express e **não alcança nada do lado direito** (`ADR-006`, ponto em aberto). A seção 9.5 fecha essa lacuna — é a única das três divergências escaladas pela RFC que este documento resolve, porque é lacuna de cobertura de implementação e não escolha entre alternativas de arquitetura.

---

## 5. Modelo de dados

> **Proposta do FDD.** Os ADRs deliberadamente não projetaram schema e a RFC parou nos artefatos. Este bloco é a primeira estrutura concreta do pacote e **não foi aplicado em `prisma/schema.prisma`** — a entrega é documental (seção 11.2).

Convenções seguidas, todas lidas de `prisma/schema.prisma` (**F**): id `@id @default(uuid()) @db.Char(36)`, `createdAt @default(now())`, `updatedAt @updatedAt`, tabela em snake_case via `@@map`, enums em SCREAMING_CASE, **sem soft delete em modelo nenhum**.

### 5.1 Enum de estado da outbox

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}
```

`TEC-01` lista os quatro valores em português — "pendente, processando, falhou, entregue" (Diego `[09:08]`, **F**). A tradução para SCREAMING_CASE em inglês segue `OrderStatus` e `UserRole` (**D**).

**A semântica de `PENDING` e `FAILED` é decisão deste documento** (**P**), e ela governa a query mais quente do sistema:

| Estado | Significado | Elegível para o polling? |
|---|---|---|
| `PENDING` | aguardando **primeiro envio ou retentativa** — a distinção fica em `nextAttemptAt` | sim, quando `nextAttemptAt <= NOW()` |
| `PROCESSING` | reivindicada por um ciclo do worker, chamada HTTP em andamento | não (ver 6.5, linhas órfãs) |
| `DELIVERED` | resposta 2xx recebida | não — terminal |
| `FAILED` | seis chamadas esgotadas, linha enviada à `webhook_dead_letter` | não — terminal |

Ressalva de honestidade: isso lê o "falhou" de `TEC-01` como **"morreu"**, e não como "a última tentativa deu erro". É interpretação de uma palavra solta numa fala do Diego, não uma decisão dele. A alternativa — `FAILED` como estado de espera entre tentativas — obrigaria a query de polling a varrer dois status, sem ganho.

### 5.2 `WebhookSubscription` — configuração do endpoint

```prisma
model WebhookSubscription {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  subscribedStatuses      Json
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer          @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_subscriptions")
}
```

| Campo | Papel | Origem |
|---|---|---|
| `url`, `secret`, `customerId`, `active` | os quatro campos que Bruno enumerou | `TEC-13` **F** |
| `subscribedStatuses` | lista de status assinados; é o filtro aplicado na inserção | `RF-05` / `TEC-15` / `DEC-16` **F** |
| `previousSecret`, `previousSecretExpiresAt` | sustentam o grace period de 24h da rotação | **D** de `TEC-11` — ver 5.6 |
| tipo `Json` em `subscribedStatuses` | `Customer.address Json` é precedente real no schema | **D** |
| `@db.VarChar(2048)` na url, `128` na secret | dimensionamento não tem fonte | **P** |
| tabela `webhook_subscriptions` | nome não tem fonte; plural segue `users`/`customers`/`products` | **P** |

**Prisma exige a relação inversa**: `Customer` precisa ganhar `webhooks WebhookSubscription[]`. É mudança em `prisma/schema.prisma` e está na checklist da seção 11.2.

### 5.3 `WebhookOutbox` — a promessa de entrega

```prisma
model WebhookOutbox {
  id             String              @id @default(uuid()) @db.Char(36)
  eventId        String              @db.Char(36)
  subscriptionId String              @db.Char(36)
  orderId        String              @db.Char(36)
  status         WebhookOutboxStatus @default(PENDING)
  payload        Json
  attemptCount   Int                 @default(0)
  nextAttemptAt  DateTime            @default(now())
  lastError      String?             @db.VarChar(500)
  requestId      String?             @db.Char(36)
  replayOfId     String?             @db.Char(36)
  deliveredAt    DateTime?
  createdAt      DateTime            @default(now())
  updatedAt      DateTime            @updatedAt

  subscription WebhookSubscription @relation(fields: [subscriptionId], references: [id])
  deliveries   WebhookDelivery[]
  deadLetter   WebhookDeadLetter?

  @@index([status, nextAttemptAt, createdAt])
  @@index([eventId])
  @@index([orderId])
  @@map("webhook_outbox")
}
```

| Campo | Papel | Origem |
|---|---|---|
| `id` UUID | PK da linha | `DEC-19` **F** |
| `eventId` | identificador **estável voltado ao cliente**, viaja em `X-Event-Id` | `TEC-04` / `TEC-05` **F** |
| `payload` | snapshot renderizado na inserção, não no envio | `DEC-15` **F** |
| `status` | ver 5.1 | `TEC-01` **F** |
| **`attemptCount`** | chamadas HTTP já realizadas — índice na tabela de backoff | **P** — campo que a RFC declarou sem origem |
| **`nextAttemptAt`** | instante a partir do qual a linha volta a ser elegível | **P** — idem |
| `lastError` | último motivo de falha, para diagnóstico sem abrir a DLQ; `VarChar(500)` espelha `OrderStatusHistory.reason` | **P** |
| `requestId` | correlação com o `X-Request-Id` da chamada HTTP que mudou o status | **P** — ver seção 10.3 |
| `replayOfId` | aponta para a linha original quando esta nasceu de um replay | **D** de `TEC-28` — ver 5.6 |
| `@@index([status, nextAttemptAt, createdAt])` | índice composto que serve a query de polling inteira | **D** de `TEC-01` + `TEC-02` |
| `@@index([eventId])` **não único** | o replay cria linha nova com o mesmo `eventId` | **D** — ver 5.6 |

`TEC-01` pediu índice em status e em `created_at`. O composto acima cobre os dois na ordem em que a query os usa, e evita um segundo índice que o MySQL não escolheria (**D**).

### 5.4 `WebhookDelivery` — histórico de entregas (`ABE-04`)

Esta tabela **não existe em nenhuma fonte**. `ABE-04` registra que Marcos especificou o comportamento de `RF-06` — "os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" (`[09:34]`) — e que nenhuma estrutura foi projetada. Sem ela, `RF-06` não é implementável.

```prisma
model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  subscriptionId String   @db.Char(36)
  eventId        String   @db.Char(36)
  attemptNumber  Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text
  failureReason  String?  @db.VarChar(64)
  durationMs     Int
  requestId      String?  @db.Char(36)
  attemptedAt    DateTime @default(now())

  outbox       WebhookOutbox       @relation(fields: [outboxId], references: [id])
  subscription WebhookSubscription @relation(fields: [subscriptionId], references: [id])

  @@index([subscriptionId, attemptedAt])
  @@index([outboxId])
  @@index([eventId])
  @@map("webhook_deliveries")
}
```

Uma linha **por chamada HTTP** — até seis por evento, por [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) (**D**).

| Pedido de Marcos (`RF-06`) | Campo | Nota |
|---|---|---|
| sucesso/falha | `success` + `responseStatus` | **D** |
| payload | *lido via relação `outbox.payload`* | **D** — o snapshot já está na outbox; duplicá-lo uma terceira vez não tem justificativa |
| response | `responseBody` | **P** no dimensionamento — ver abaixo |
| tempo de resposta | `durationMs` | **D** — nome idêntico ao campo de `requestLogger` em `src/middlewares/request-logger.middleware.ts`, para não criar sinônimo |

**Teto no `responseBody`** (**P**): guardar corpo de resposta de terceiro sem limite é linha de tabela crescendo com dado que não controlamos. O corpo é truncado antes de persistir, e o teto entra como constante do módulo. Dois avisos que vão para a revisão de `RNF-10`: esse corpo pode conter dado sensível do cliente, e ele **não** é coberto pelo `redact` do Pino porque não passa pelo logger.

### 5.5 `WebhookDeadLetter` — destino terminal

```prisma
model WebhookDeadLetter {
  id             String    @id @default(uuid()) @db.Char(36)
  outboxId       String    @unique @db.Char(36)
  eventId        String    @db.Char(36)
  subscriptionId String    @db.Char(36)
  payload        Json
  failureReason  String    @db.VarChar(64)
  failureDetail  String?   @db.VarChar(500)
  attemptCount   Int
  replayedAt     DateTime?
  replayedById   String?   @db.Char(36)
  createdAt      DateTime  @default(now())

  outbox WebhookOutbox @relation(fields: [outboxId], references: [id])

  @@index([subscriptionId])
  @@index([eventId])
  @@map("webhook_dead_letter")
}
```

Nome da tabela literal de `TEC-12` (singular, ao contrário das outras três — é assim que Diego a nomeou, e a fonte ganha da consistência estética). Os três campos que ele pediu estão lá: `payload`, `failureReason` e `createdAt` (**F**). O payload é duplicado em relação à outbox, e isso é trade-off aceito explicitamente em `ALT-06` / [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md).

`replayedAt` / `replayedById` (**P**): `RF-10` pede que o endpoint de replay **logue** quem executou, e o log satisfaz a fonte literal (seção 10.1). Persistir também é escolha deste documento — log rotaciona, e uma trilha de auditoria que some depois de 30 dias não é trilha de auditoria.

### 5.6 A identidade do evento através do replay

`TEC-28` diz que o replay "recoloca na outbox como pendente" (**F**). Nenhuma fonte diz o que acontece com o `event_id` nesse recolocar, e a resposta decide se `ADR-005` funciona ou não.

O cenário: um cliente processou o evento e demorou mais de 10 segundos: o worker cortou por timeout (`RNF-06`), retentou seis vezes, mandou para a DLQ. Um administrador dá replay. **Se o replay gerar `eventId` novo**, o cliente — que já tem aquele evento processado — não reconhece o reenvio, e a dedup por `X-Event-Id`, que é a única defesa que oferecemos contra duplicata, falha exatamente no caso que ela existe para cobrir. `ADR-005` afirma que o `event_id` "não muda entre retentativas"; o replay é mais uma tentativa, só que atravessando duas tabelas.

Regras, portanto (**D** de `ADR-005` + `TEC-28`):

1. **`eventId` é preservado** por todo o ciclo: `webhook_outbox` → `webhook_dead_letter` → replay → nova linha de `webhook_outbox`.
2. O índice em `eventId` é **não único** nas três tabelas, porque o replay cria linha nova com o mesmo valor; `replayOfId` liga a nova à original.
3. **A linha original da outbox não é apagada** ao cair na DLQ: vai para `FAILED` e fica. Justificativa em fonte: o projeto não tem soft delete nem padrão de deleção em modelo nenhum (`CONTEXT.md` §12), e `ADI-04` adiou explicitamente o arquivamento — o que só faz sentido se as linhas se acumulam.

Consequência que precisa estar no material do cliente: **um mesmo `X-Event-Id` pode chegar dias depois**, se um administrador der replay. É extensão da garantia at-least-once de `DEC-10`, não uma garantia nova, mas é uma cara dela que a reunião não discutiu.

---

## 6. Fluxos detalhados

### 6.1 Criação do evento na outbox

Ponto de entrada: `publishWebhookEvent(tx, order, fromStatus, toStatus)` — função nova que recebe o `tx` client da transação corrente e é invocada pelo `OrderService` (`COD-14`, Bruno `[09:41]`, **F**). Chamada entre `tx.orderStatusHistory.create` (linha 159) e a releitura do pedido (linha 169) de `src/modules/orders/order.service.ts`.

```
publishWebhookEvent(tx, order, fromStatus, toStatus, requestId)
│
├─ 1. tx.webhookSubscription.findMany({ where: { customerId: order.customerId, active: true } })
│         └─ filtro em memória: subscribedStatuses contém toStatus         (RF-05 / TEC-15)
│
├─ 2. se nenhuma assinatura sobrou → RETORNA SEM INSERIR                   (DEC-16)
│         "Se nenhum webhook do customer quer aquele status, nem insere."
│
├─ 3. monta o payload snapshot, uma vez                                    (DEC-15 / TEC-17)
│
├─ 4. verifica o tamanho: Buffer.byteLength(JSON.stringify(payload),'utf8')
│         └─ > WEBHOOK_PAYLOAD_MAX_BYTES → lança WebhookPayloadTooLargeError
│            e a transação inteira sofre rollback                          (DEC-18 — erro, não trunca)
│
└─ 5. uma linha de webhook_outbox POR ASSINATURA que passou no filtro
          eventId = randomUUID()   status = PENDING
          attemptCount = 0         nextAttemptAt = now()
```

Quatro detalhes que a fonte não diz e este documento decide:

- **Uma linha por assinatura, e um `eventId` por linha** (**P**). `TEC-04` diz que o `event_id` é "único por evento", e `TEC-08` justifica o `X-Webhook-Id` dizendo que um cliente pode ter vários cadastros. Se um cliente com dois endpoints recebesse o mesmo `eventId` nos dois, a dedup por `X-Event-Id` do lado dele descartaria a segunda entrega como duplicata — quebrando `TEC-08` no exato caso que ele existe para atender. A alternativa (um `eventId` por mudança de status, compartilhado) é mais intuitiva e está errada.
- **O filtro roda dentro da transação** (**D** de `DEC-16` + `RF-11`): a leitura das assinaturas passa a fazer parte do caminho crítico de `changeStatus`. Consequência já registrada em [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) — mudar assinatura não reprocessa evento passado.
- **O teto de 64KB é medido em bytes UTF-8 do JSON serializado** (**D** de `DEC-18`/`RNF-05`; `65536`). `DEC-18` decidiu erro em vez de truncamento, então a consequência é derrubar uma mudança de status legítima — trade-off deliberado, registrado como risco em [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md).
- **`requestId` vem de `req.id`** (**P**), propagado pelo controller até o service. Ver 10.3.

Payload do evento — campos exatamente como Diego os enumerou em `TEC-17`/`TEC-18`/`TEC-19` (**F**), e **sem os itens do pedido** (`ALT-08`, **F**):

```json
{
  "event_id": "9f1c2e64-3a7b-4c51-9f0d-2b8e6a1d7c34",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-17T14:32:07.412Z",
  "order_id": "3c9a1f28-7e64-4b12-8d5a-1f0e9c3b7a26",
  "order_number": "ORD-2026-000481",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "b41d7e93-2c08-4a6f-9e13-5d7c2a8b4e01",
  "total_cents": 148900
}
```

Chaves em `snake_case`, ao contrário do `camelCase` que a API REST usa (`src/shared/http/response.ts`, `order.schemas.ts`): é o formato literal de `TEC-17`, e o payload do webhook é contrato externo, não a API interna (**F** sobre **D**).

### 6.2 Processamento pelo worker

`src/worker.ts` é o entry point, análogo a `src/server.ts` (`COD-10` / `TEC-20`, **F**); `webhook.worker.ts` tem a lógica (`ABE-02`, resolvido fora da reunião). O worker instancia o **próprio `PrismaClient`** via `createPrismaClient()` de `src/config/database.ts` — não importa o singleton `prisma`, que é do processo da API (`DEC-14` / `COD-12`, **F**).

Um ciclo, a cada `WEBHOOK_POLL_INTERVAL_MS` (`DEC-02`, **F**):

```
┌─ ciclo ────────────────────────────────────────────────────────────────────┐
│ 1. RECUPERAÇÃO  devolve linhas órfãs em PROCESSING para PENDING  (6.5)     │
│                                                                            │
│ 2. SELEÇÃO      WHERE status = 'PENDING' AND nextAttemptAt <= NOW()        │
│                 ORDER BY createdAt ASC                    (TEC-02, TEC-27) │
│                 LIMIT WEBHOOK_POLL_BATCH_SIZE                   (TEC-02)   │
│                                                                            │
│ 3. REIVINDICAÇÃO  UPDATE ... SET status='PROCESSING' WHERE id IN (...)     │
│                   AND status='PENDING'   ← guarda contra worker duplicado  │
│                                                                            │
│ 4. para cada linha, EM SÉRIE (nunca em paralelo)                (ADR-007)  │
│      ├─ carrega a assinatura; se inativa → 6.6                             │
│      ├─ assina o corpo    (seção 8)                                        │
│      ├─ POST com timeout de WEBHOOK_DELIVERY_TIMEOUT_MS         (RNF-06)   │
│      ├─ grava WebhookDelivery — sempre, sucesso ou falha                   │
│      └─ 2xx → DELIVERED   |   resto → 6.3                                  │
└────────────────────────────────────────────────────────────────────────────┘
```

**Serialização estrita do lote** (**D** de `ADR-007`): a garantia de ordem por `order_id` deriva de "um único worker processa em ordem de `created_at`" (`TEC-27`). Processar o lote com `Promise.all` reintroduziria concorrência dentro do worker único e quebraria a garantia sem que nada no código sinalizasse. É o erro mais fácil de cometer aqui, e ele é invisível em teste de caminho feliz.

**Sucesso é 2xx** (**F**, diagrama da [RFC](RFC.md)). Redirecionamentos 3xx contam como falha e **não são seguidos** (**P**): seguir redirect enviaria payload assinado para um host que o cliente não cadastrou, contornando a validação de `https` de `DEC-09`.

### 6.3 Retry e backoff

`DEC-05` fixou "5 tentativas, backoff 1m/5m/30m/2h/12h" (**F**). `TEC-24` registrou que nunca foi dito se as cinco incluem o envio inicial; a aritmética resolve e está em [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md): **cinco retentativas além do envio inicial, seis chamadas HTTP no total**.

Tabela de execução (**D**):

| `attemptCount` após a falha | Chamada que acabou de falhar | Intervalo aplicado | `nextAttemptAt` | Acumulado desde a 1ª falha |
|---|---|---|---|---|
| 1 | envio inicial | 1 min | `now + 1m` | 1min |
| 2 | 1ª retentativa | 5 min | `now + 5m` | 6min |
| 3 | 2ª retentativa | 30 min | `now + 30m` | 36min |
| 4 | 3ª retentativa | 2 h | `now + 2h` | 2h36 |
| 5 | 4ª retentativa | 12 h | `now + 12h` | 14h36 |
| 6 | 5ª retentativa | — | — | → DLQ (6.4) |

```
BACKOFF_INTERVALS_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]
                        1m      5m       30m         2h          12h
```

Constante do módulo, **não** variável de ambiente (**D**): é lista de cinco posições fixada por `DEC-05`, e parametrizá-la em string de env convidaria a divergir da decisão sem rastro.

Ao falhar, na mesma transação (**D**): `attemptCount += 1`, `lastError`, `status` volta para `PENDING`, `nextAttemptAt = now + BACKOFF_INTERVALS_MS[attemptCount - 1]`, e insere a linha de `WebhookDelivery`.

### 6.4 Dead letter queue

Quando a sexta chamada falha (`attemptCount` chegaria a 6), em uma transação (**D** de `DEC-06` + `TEC-12`):

```
tx.webhookOutbox.update  → status = FAILED
tx.webhookDeadLetter.create → { outboxId, eventId, subscriptionId, payload,
                                failureReason, failureDetail, attemptCount: 6 }
logger.warn({...}, 'webhook_dead_lettered')
```

`failureReason` usa o vocabulário controlado da seção 9.6 — o motivo da **última** chamada, com `WEBHOOK_ATTEMPTS_EXHAUSTED` como qualificador registrado em `failureDetail` quando o motivo varia entre tentativas.

**Replay** (`RF-08` / `DEC-07` / `TEC-22`, **F**), `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, `requireRole('ADMIN')` (`DEC-12`, **F**):

```
1. carrega a linha da DLQ; 404 se não existe
2. INSERT nova linha em webhook_outbox:
      eventId       = o mesmo da original          ← 5.6, decisão central
      payload       = o mesmo (snapshot, DEC-15)
      status        = PENDING     attemptCount = 0     nextAttemptAt = now()
      replayOfId    = outboxId da original
3. UPDATE webhook_dead_letter → replayedAt = now(), replayedById = req.user.id
4. logger.info({ userId, deadLetterId, eventId }, 'webhook_replay_requested')   (RF-10)
```

O ciclo recomeça inteiro — seis chamadas novas (`TEC-28`, **F**). Nada limita quantas vezes um mesmo evento volta da DLQ; [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) registra isso como consequência aceita, e este FDD não inventa um limite.

### 6.5 Linhas órfãs em `PROCESSING`

Se o worker morre entre reivindicar a linha e receber a resposta, ela fica em `PROCESSING` para sempre: nenhum polling futuro a pega, nenhuma retentativa acontece, nenhum alarme dispara. **Não é cenário exótico — acontece em todo deploy do worker**, e o resultado contradiz frontalmente a premissa de `RF-11` ("não pode ter caso de status mudar e evento não sair").

Nenhuma fonte trata disso. Duas regras (**P**):

1. **Shutdown gracioso**, no molde real de `src/server.ts`: handlers de `SIGINT`/`SIGTERM` param de aceitar novo lote, esperam a chamada em voo terminar, devolvem para `PENDING` o que não completou, e chamam `prisma.$disconnect()`.
2. **Lease para morte abrupta**: no passo 1 de cada ciclo, linha em `PROCESSING` com `updatedAt` mais velho que `WEBHOOK_PROCESSING_LEASE_MS` volta a `PENDING` sem incrementar `attemptCount`.

O que autoriza a regra 2 sem inventar garantia nova: reivindicar de volta uma linha que talvez tenha sido entregue pode causar entrega dupla — e **duplicata já é o contrato aceito** por `DEC-10` / [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md). A recuperação é licenciada por uma decisão existente.

### 6.6 Assinatura inativa ou removida no meio do caminho

`RF-02` permite desativar (`active = false`) e `RF-03` remover. Nenhuma fonte diz o que acontece com linhas pendentes daquela assinatura.

Este documento especifica apenas o caso **`active = false`** (**P**): o worker descarta a linha marcando `DELIVERED` não é aceitável, e `FAILED` sem DLQ perderia rastro — a linha vai para `FAILED` **com** registro em `webhook_dead_letter` e `failureReason = WEBHOOK_SUBSCRIPTION_INACTIVE`, preservando a evidência.

O caso **`DELETE`** fica em aberto (seção 13): a FK impede apagar a assinatura enquanto houver outbox ou delivery apontando para ela, e resolver isso com `onDelete: Cascade` apagaria o histórico de entregas que `RF-06` promete ao cliente. É decisão de produto, não de implementação.

---

## 7. Contratos públicos

Prefixo `/api/v1`, conforme `src/app.ts` linha 67 (`app.use('/api/v1', buildApiRouter(controllers))`, **F**). Todas as rotas passam por `authenticate` via `router.use(authenticate)`, no molde de `src/modules/orders/order.routes.ts` linha 14 (`DEC-20`, **F**). Recursos individuais voltam **sem envelope**; listas usam `paginated<T>` de `src/shared/http/response.ts`; erros usam `{ error: { code, message, details? } }` produzido por `errorMiddleware` (**F**, `CONTEXT.md` §7–§8).

`customerId` vai no **body** do `POST` e na **query** do `GET`, com o path reservado ao `:id` do próprio recurso — resolução de `ABE-03` pela convenção real de `order.schemas.ts` (`createOrderSchema` declara `customerId` no body; `listOrdersQuerySchema`, na query). A reunião fechou apenas que **não vem do JWT** (`RES-07`, **F**); a forma exata segue carecendo de aval.

| RF | Método e caminho | Autorização | Proveniência do caminho |
|---|---|---|---|
| RF-01 | `POST /api/v1/webhooks` | autenticado | derivado de `order.routes.ts` **D** |
| RF-04/05 | `GET /api/v1/webhooks?customerId=` | autenticado | derivado **D** |
| RF-02 | `PATCH /api/v1/webhooks/:id` | autenticado | derivado **D** |
| RF-03 | `DELETE /api/v1/webhooks/:id` | autenticado | derivado **D** |
| RF-06 | `GET /api/v1/webhooks/:id/deliveries` | autenticado | **literal** `TEC-21` **F** |
| RF-07 | `POST /api/v1/webhooks/:id/rotate-secret` | autenticado | **proposto — sem origem** **P** |
| RF-08 | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | `requireRole('ADMIN')` | **literal** `TEC-22` **F** |

### 7.1 `POST /api/v1/webhooks` — cadastrar (RF-01)

**Request**
```http
POST /api/v1/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json
```
```json
{
  "customerId": "b41d7e93-2c08-4a6f-9e13-5d7c2a8b4e01",
  "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**`201 Created`** — a `secret` é gerada pelo sistema e devolvida **uma única vez** (`TEC-14` / `RF-01`, **F**):
```json
{
  "id": "7d2f8c15-4b93-4e07-a6d1-3c5f2b9e8a44",
  "customerId": "b41d7e93-2c08-4a6f-9e13-5d7c2a8b4e01",
  "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f3a1c7e2b8d4a6f0e5c3b1d9a7f2e4c",
  "createdAt": "2026-08-17T14:20:11.003Z",
  "updatedAt": "2026-08-17T14:20:11.003Z"
}
```

| Código | Quando | Corpo |
|---|---|---|
| `201` | criado | acima |
| `400` | `url` não é `https`, `customerId` não é uuid, `subscribedStatuses` com valor fora de `OrderStatus` | `VALIDATION_ERROR` — ver 9.6 sobre o conflito com `WEBHOOK_INVALID_URL` |
| `401` | sem `Authorization` ou token inválido | `UNAUTHORIZED` (`authenticate`) |
| `404` | `customerId` não existe | `NOT_FOUND` |

`subscribedStatuses` valida com `z.nativeEnum(OrderStatus)`, no padrão de `updateOrderStatusSchema` (**D**). O prefixo `whsec_` no formato da secret é **P**.

### 7.2 `GET /api/v1/webhooks?customerId=` — listar (RF-04 / RF-05)

**Request**: `GET /api/v1/webhooks?customerId=b41d7e93-...&page=1&pageSize=20`

**`200 OK`** — envelope `paginated<T>` real. **A `secret` nunca aparece aqui** (**D** de `TEC-14`: devolvida na criação):
```json
{
  "data": [
    {
      "id": "7d2f8c15-4b93-4e07-a6d1-3c5f2b9e8a44",
      "customerId": "b41d7e93-2c08-4a6f-9e13-5d7c2a8b4e01",
      "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotatedAt": null,
      "createdAt": "2026-08-17T14:20:11.003Z",
      "updatedAt": "2026-08-17T14:20:11.003Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

`page`/`pageSize` com `coerce.number()`, defaults `1`/`20` e `max(100)`, idênticos a `listOrdersQuerySchema` (**D**). Status: `200`, `400` (query inválida), `401`.

### 7.3 `PATCH /api/v1/webhooks/:id` — editar (RF-02)

**Request** — todos os campos opcionais, no molde de um PATCH parcial:
```json
{ "url": "https://api.atlascomercial.com.br/v2/webhook", "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

**`200 OK`**: mesmo corpo de 7.2, sem `secret`. Demais: `400` validação, `401`, `404` `WEBHOOK_NOT_FOUND`.

`customerId` **não é editável** (**P**): mover uma assinatura de cliente mudaria o dono do histórico de entregas já gravado.

### 7.4 `DELETE /api/v1/webhooks/:id` — remover (RF-03)

**`204 No Content`**, sem corpo. `401`; `404` `WEBHOOK_NOT_FOUND`; **`409`** quando há linha de outbox pendente ou histórico de entrega referenciando a assinatura — consequência da FK, e o caso está em aberto (6.6 e seção 13). O `409` é **P**.

### 7.5 `GET /api/v1/webhooks/:id/deliveries` — histórico (RF-06)

Caminho literal de `TEC-21` (**F**). Marcos pediu "os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" (`[09:34]`, **F**).

**Request**: `GET /api/v1/webhooks/7d2f8c15-.../deliveries?page=1&pageSize=20`

**`200 OK`**
```json
{
  "data": [
    {
      "id": "e1a7c3d9-5f24-4b81-9c60-8d2e4a7f1b53",
      "eventId": "9f1c2e64-3a7b-4c51-9f0d-2b8e6a1d7c34",
      "attemptNumber": 2,
      "success": false,
      "responseStatus": 503,
      "responseBody": "{\"error\":\"upstream unavailable\"}",
      "failureReason": "WEBHOOK_DELIVERY_HTTP_ERROR",
      "durationMs": 1284,
      "attemptedAt": "2026-08-17T14:33:12.887Z",
      "payload": {
        "event_id": "9f1c2e64-3a7b-4c51-9f0d-2b8e6a1d7c34",
        "event_type": "order.status_changed",
        "timestamp": "2026-08-17T14:32:07.412Z",
        "order_id": "3c9a1f28-7e64-4b12-8d5a-1f0e9c3b7a26",
        "order_number": "ORD-2026-000481",
        "from_status": "PAID",
        "to_status": "PROCESSING",
        "customer_id": "b41d7e93-2c08-4a6f-9e13-5d7c2a8b4e01",
        "total_cents": 148900
      }
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 47, "totalPages": 3 }
}
```

Como "últimos 100" e paginação convivem (**D**): a janela do endpoint é limitada às **100 entregas mais recentes** da assinatura, ordenadas por `attemptedAt DESC`, e a paginação opera dentro dessa janela — `pagination.total` nunca passa de 100. Atende `RF-06` literalmente e reusa `paginated<T>`, em vez de criar o único formato de lista fora do padrão na API inteira. `payload` vem por relação com a outbox, não duplicado (5.4).

Status: `200`, `400`, `401`, `404` `WEBHOOK_NOT_FOUND`.

### 7.6 `POST /api/v1/webhooks/:id/rotate-secret` — rotação (RF-07)

**O caminho é proposta desta RFC/FDD e não tem origem na transcrição** (**P**). Sofia especificou apenas "endpoint pro cliente conseguir pedir nova secret pela API" (`RF-07`, `[09:21]`). O **comportamento**, ao contrário do caminho, é fonte: `TEC-11` e `DEC-08` dão o grace period de 24 horas.

**Request**: sem corpo.

**`200 OK`** — a secret nova em claro, uma única vez:
```json
{
  "id": "7d2f8c15-4b93-4e07-a6d1-3c5f2b9e8a44",
  "secret": "whsec_2e7b9d4f1a6c3e8b0d5a2f7c9e4b1d63",
  "previousSecretExpiresAt": "2026-08-18T14:20:11.003Z",
  "rotatedAt": "2026-08-17T14:20:11.003Z"
}
```

Efeito no dado: `previousSecret ← secret`, `previousSecretExpiresAt ← now + WEBHOOK_SECRET_GRACE_PERIOD_HOURS`, `secret ← nova` (**D** de `TEC-11`). Durante a janela, os envios carregam **duas assinaturas** — seção 8.2.

Status: `200`, `401`, `404` `WEBHOOK_NOT_FOUND`, e **`409` `WEBHOOK_ROTATION_IN_PROGRESS`** quando já existe uma rotação com grace period ativo (**P**): rotacionar duas vezes em 24h descartaria silenciosamente a secret que o cliente ainda está migrando.

`TEC-11` descreve só a migração planejada. **Revogação imediata em incidente não foi decidida** ([ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md), ponto em aberto) e este endpoint não a oferece.

### 7.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay (RF-08)

Caminho literal de `TEC-22` (**F**), `requireRole('ADMIN')` sobre `authenticate` (`DEC-12` / `COD-09`, **F**).

**`202 Accepted`** — o replay enfileira, não entrega:
```json
{
  "deadLetterId": "a3f7c291-6d84-4e15-b072-9c8f3a1e5d47",
  "eventId": "9f1c2e64-3a7b-4c51-9f0d-2b8e6a1d7c34",
  "outboxId": "5b8e1d47-2f93-4a60-8c15-7e4d9b2a6f38",
  "status": "PENDING",
  "replayedAt": "2026-08-17T15:02:44.119Z"
}
```

`outboxId` é a **linha nova**; `eventId` é o **mesmo da original** (5.6). Status: `202`; `401`; **`403` `FORBIDDEN`** para role `OPERATOR` (`requireRole` lança `ForbiddenError`, **F**); `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

`202` em vez de `200` é **D**: `TEC-28` diz que o replay "recoloca na outbox como pendente", ou seja, aceita o pedido sem completar a entrega.

### 7.8 Contrato de saída — o POST que o cliente recebe

Este é o contrato que mais importa ao cliente e o único que não é da nossa API.

```http
POST /integracoes/oms/webhook HTTP/1.1
Host: api.atlascomercial.com.br
Content-Type: application/json
X-Event-Id: 9f1c2e64-3a7b-4c51-9f0d-2b8e6a1d7c34
X-Webhook-Id: 7d2f8c15-4b93-4e07-a6d1-3c5f2b9e8a44
X-Timestamp: 2026-08-17T14:32:09.550Z
X-Signature: sha256=4f7c1e9a...
```

| Header | Conteúdo | Origem |
|---|---|---|
| `X-Event-Id` | UUID do evento, estável entre retentativas **e através do replay** | `TEC-04`/`TEC-05` **F**; extensão ao replay **D** |
| `X-Signature` | HMAC-SHA256 do corpo — seção 8 | `TEC-06` **F**; formato **P** |
| `X-Timestamp` | instante do envio, ISO 8601 | `TEC-07` **F** |
| `X-Webhook-Id` | id da assinatura, para cliente com vários cadastros | `TEC-08` **F** |
| `Content-Type` | `application/json` | `TEC-09` **F** |

Corpo: exatamente o snapshot de 6.1. **Nenhum header novo é acrescentado** — em particular, o `requestId` de correlação interna **não** vai ao cliente (10.3): [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md) congelou o contrato em cinco headers, e acrescentar um sexto seria redecidir um ADR.

Expectativa de resposta: qualquer `2xx` em até 10 segundos é sucesso. Corpo da resposta é ignorado para efeito de decisão, mas armazenado truncado (5.4).

---

## 8. Assinatura HMAC

### 8.1 Cálculo

HMAC-SHA256 sobre **o corpo do request** (`DEC-08` / `TEC-10`, **F**), com a secret da assinatura, usando `node:crypto` — builtin, dependência zero (**D** de `RES-03`):

```
assinatura = HMAC_SHA256(secret, corpo_exato_enviado)
```

O corpo assinado é **o mesmo array de bytes enviado** (**D**): o JSON é serializado uma vez, assinado, e essa string vai no `body`. Serializar duas vezes abre a porta para divergência de ordenação de chaves que faria a verificação do cliente falhar de forma intermitente e praticamente indiagnosticável.

Codificação: **hex minúsculo, com prefixo de algoritmo** — `sha256=<hex>` (**P**). Nenhuma fonte define hex, base64 ou prefixo. O prefixo deixa a porta aberta para trocar de algoritmo sem quebrar parser de cliente.

### 8.2 Duas assinaturas durante o grace period

`TEC-11`: "Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele" (Sofia `[09:21]`, **F**).

A direção da confiança é: **nós assinamos, o cliente verifica**. "A antiga fica válida" só significa alguma coisa se um cliente que ainda verifica com a secret antiga conseguir validar o que enviamos. Isso força uma escolha que ninguém fez:

| Opção | Consequência |
|---|---|
| assinar só com a nova | a secret antiga não é "válida" em nada, e as 24 horas não servem para nada |
| assinar só com a antiga por 24h | a rotação não produz efeito nenhum durante um dia |
| **enviar as duas assinaturas** | o cliente valida com qualquer uma e migra quando quiser — é o que `TEC-11` descreve |

Decisão deste documento (**P**), a terceira: durante a janela, um header com dois valores, nova primeiro.

```http
X-Signature: sha256=<nova>                      ← fora da janela
X-Signature: sha256=<nova>, sha256=<antiga>     ← durante as 24h
```

Header único com vírgula em vez de header repetido, porque header repetido é onde implementações de cliente divergem em silêncio (**P**).

Três ressalvas que vão nominalmente para a revisão de `RNF-10`:

1. Isto **estende a semântica de `X-Signature`**, que [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) e [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md) registraram como valor único. É a derivação de maior alcance deste FDD.
2. Stripe e GitHub foram citados na reunião (`ALT-07`) **apenas** sobre at-least-once, nunca sobre formato de assinatura. Não me apoio neles como fonte.
3. Durante 24 horas existem duas secrets válidas, e uma delas pode ser justamente a comprometida que motivou a rotação — consequência já registrada em [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md).

### 8.3 Armazenamento da secret em repouso

Questão em aberto bloqueante na [RFC](RFC.md). `TEC-13` lista a tabela como "url + secret + customer_id + estado ativo" sem qualificar formato de armazenamento.

**A secret é armazenada em claro** em `WebhookSubscription.secret`, marcada como **pendente da revisão de `RNF-10`** (**P**).

O motivo de não seguir o padrão do próprio projeto precisa estar explícito, porque é a primeira coisa que um revisor vai estranhar: `User.passwordHash` guarda hash bcrypt porque a senha só precisa ser *comparada*. **A secret do webhook precisa ser reproduzida em claro a cada envio** para calcular o HMAC — hash é tecnicamente impossível aqui, não é uma escolha de conveniência.

Este documento **não** propõe cifragem com chave nova em variável de ambiente: seria inventar mecanismo de segurança dentro exatamente da área que Sofia reservou para revisar com calma (`RNF-10`: "HMAC e geração de secret eu quero olhar com calma"). A decisão é dela.

Vai junto o gap real: `redactPaths` em `src/shared/logger/index.ts` cobre `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, e **nenhum padrão que cubra `secret`**. Correção exigida na checklist 11.2.

---

## 9. Estratégias de resiliência

### 9.1 Orçamento de latência — o alvo não fecha

`RNF-01` pede menos de 10 segundos end-to-end (`[09:02]`, Marcos, **F**). A soma dos valores decididos (**D**):

| Etapa | Pior caso | Origem |
|---|---|---|
| commit da transação → próximo ciclo de polling | 2 000 ms | `DEC-02` **F** |
| seleção, reivindicação, montagem e assinatura | dezenas de ms | **D** |
| chamada HTTP ao cliente | 10 000 ms | `RNF-06` **F** |
| **total no caminho sem retry** | **> 12 s** | **excede `RNF-01`** |

Isso **não é bifurcação de fluxo** — é alvo contra mecanismo, e por isso o fluxo da seção 6 é especificado com os valores decididos, sem ramo alternativo. O conflito está escalado como [questão 4 da RFC](RFC.md#exigem-decisão-mas-não-travam-o-início), com a recomendação (relaxar o alvo, reformulando `RNF-01` como meta de caminho feliz) **explicitamente não decidida por ninguém**. Este documento não decide no lugar.

No caminho com retry a distância é de outra ordem: um evento entregue na sexta chamada chega **14h36 depois** do fato.

### 9.2 Timeout

`WEBHOOK_DELIVERY_TIMEOUT_MS`, default `10000` (`RNF-06` / `TEC-25`, **F**), implementado com `AbortSignal.timeout()` sobre `fetch` (**D** — seção 12). O timeout cobre a resposta completa, não só os headers (**P**).

Consequência já registrada em [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md) e que vale repetir para quem for implementar: **um cliente que processe em mais de 10 segundos recebe o evento de novo sem ter havido falha nenhuma**. Duplicata não é caso excepcional aqui, é consequência garantida do timeout.

### 9.3 Retry e DLQ

Seções 6.3 e 6.4. Cinco retentativas além do envio inicial; backoff fixo; destino terminal em tabela própria.

### 9.4 Ordem sob retry — premissa assumida

Combinando `DEC-04` com `DEC-05`: se o evento `PAID` de um pedido falha e é reagendado para daqui a 1 minuto, e o `SHIPPED` do **mesmo** pedido é criado e entregue nesse intervalo, o cliente recebe `SHIPPED` antes de `PAID`. A garantia de [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md) vale para o caminho feliz.

**Premissa assumida por este documento** (**P**, rotulada como tal): os fluxos da seção 6 são especificados na perna **"aceitar e documentar"** — a recomendação não decidida da [RFC](RFC.md#exigem-decisão-mas-não-travam-o-início), consistente com a postura de `DEC-04` de declarar limitação em vez de construir mecanismo. Nenhum bloqueio por `order_id` existe no fluxo especificado.

**O que muda se a decisão for a outra.** Bloquear a entrega de eventos subsequentes de um `order_id` enquanto houver evento anterior em retry exige: (i) a query de 6.2 ganha uma condição de exclusão dos `order_id` que tenham linha `PENDING` mais antiga com `attemptCount > 0` — o que a tira do índice composto de 5.3 e a transforma em subconsulta correlacionada; (ii) um `order_id` com evento preso em retry de 12 horas passa a segurar **todos** os eventos seguintes daquele pedido pelo mesmo período; (iii) surge risco de fila por pedido que o desenho de polling simples não comporta. É reescrita de 6.2 e 5.3, não um ajuste.

### 9.5 Limite de erro do worker

`COD-08` afirma que o middleware de erro pega os erros do módulo "sem precisar mudar nada". **É verdade no caminho HTTP e falso no worker**: `errorMiddleware` é um `ErrorRequestHandler` do Express, e o worker roda fora dele por `DEC-03`. Não é trade-off entre alternativas, é lacuna de cobertura — e por isso este documento a fecha, em vez de escalar.

O molde é o `src/server.ts` real (**D**):

| Camada | No `src/server.ts` (existe) | No `src/worker.ts` (a construir) |
|---|---|---|
| falha de boot | `bootstrap().catch(err => { logger.fatal({ err }, 'bootstrap_failed'); process.exit(1); })` | idêntico, evento `worker_bootstrap_failed` |
| encerramento | `SIGINT`/`SIGTERM` → `server.close()` + `prisma.$disconnect()` | `SIGINT`/`SIGTERM` → para o loop, devolve linhas em voo (6.5), `prisma.$disconnect()` |
| erro por unidade de trabalho | `errorMiddleware` por request | **try/catch por linha da outbox** — a falha de uma entrega nunca derruba o ciclo nem o processo |
| erro por ciclo | — | try/catch em volta do ciclo inteiro: falha de banco loga `webhook_poll_failed` e o próximo ciclo tenta de novo |

Duas regras que valem escrever porque são o modo de falha real deste desenho (**P**):

- **`AppError` continua sendo a hierarquia** (`RES-03`), mas no worker ela é *logada*, nunca serializada em resposta — não há resposta. `statusCode` é ignorado; `errorCode` vira campo estruturado do log.
- **`unhandledRejection` e `uncaughtException` derrubam o processo deliberadamente**, depois de `logger.fatal`. Um worker vivo em estado indefinido é pior que um worker morto: linhas ficam presas em `PROCESSING`, e é justamente aí que o lease de 6.5 e a supervisão em aberto de [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) importam.

### 9.6 Matriz de erros

Duas tabelas, deliberadamente separadas. Misturá-las seria o erro fácil aqui: os motivos de falha de entrega **não têm `statusCode`, não passam por `errorMiddleware` e não são erros da nossa API**.

`TEC-16` nomeia três códigos e fecha com "etc." (Bruno `[09:28]`, **F**). A regra aplicada: **só entra código que corresponda a uma regra decidida em fonte** — nada de código de completude.

#### 9.6.1 Erros HTTP da API de webhooks

Todas as classes estendem `AppError` (`src/shared/errors/app-error.ts`) via as especializações de `http-errors.ts`, no molde de `InvalidStatusTransitionError` e `InsufficientStockError` (`COD-04`/`COD-05`/`COD-06`, **F**), e são exportadas pelo barrel `src/shared/errors/index.ts`.

| Código | HTTP | Classe | Dispara quando | Origem |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError extends NotFoundError` | assinatura inexistente em `:id` | `TEC-16` **F** |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError extends BadRequestError` | guarda de serviço sobre a url — **ver ressalva** | `TEC-16` **F** |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `BadRequestError` | **sem caminho de disparo no desenho atual — ver ressalva** | `TEC-16` **F** |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `WebhookPayloadTooLargeError extends UnprocessableEntityError` | snapshot acima de 64KB durante `changeStatus` | `DEC-18`/`RNF-05` **D** |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `NotFoundError` | `:id` de replay inexistente | `RF-08` **D** |
| `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | rotação pedida com grace period ativo | **P** (7.6) |
| `WEBHOOK_SUBSCRIPTION_IN_USE` | 409 | `ConflictError` | `DELETE` com outbox pendente ou histórico | **P** (7.4) |

`ConflictError` já aceita código customizado — `src/modules/orders/order.service.ts` linha 187 usa `'INVALID_ORDER_STATE_FOR_DELETE'` exatamente assim (**F**). Nenhuma alteração em `error.middleware.ts` é necessária.

> **Ressalva 1 — `WEBHOOK_INVALID_URL` versus `DEC-09`.** Sofia decidiu que a recusa de `http` é "só uma validação no schema Zod" (`DEC-09`/`TEC-26`, **F**). Mas validação Zod é convertida por `validate.middleware.ts` em `ValidationError`, e `errorMiddleware` responde **`VALIDATION_ERROR`**, não `WEBHOOK_INVALID_URL`. As duas fontes são incompatíveis no caminho HTTP normal. Resolução deste documento: a regra `https` fica no Zod (`DEC-09` é explícita e é da autora do requisito), o cliente vê `VALIDATION_ERROR` com `details[].path = "url"`, e `WEBHOOK_INVALID_URL` fica como guarda de serviço para chamadas que não passem pela rota. **Fica registrado como divergência entre `DEC-09` e `TEC-16` que o FDD não tem autoridade para fechar** (seção 13).
>
> **Ressalva 2 — `WEBHOOK_SECRET_REQUIRED` não tem gatilho.** Bruno nomeou o código às `[09:28]`; às `[09:31]` Marcos estabeleceu que "secret é gerada pela gente e devolvida na criação" (`TEC-14`). **O cliente nunca fornece secret, logo nunca pode omiti-la.** O código foi nomeado três minutos antes da decisão que o tornou inalcançável. Mantido na matriz por ser fonte literal, marcado como candidato a remoção na revisão de design.

#### 9.6.2 Motivos de falha de entrega (worker)

Vocabulário controlado de `WebhookDelivery.failureReason` e `WebhookDeadLetter.failureReason`. Sem `statusCode`, sem resposta HTTP, sem `errorMiddleware`. `TEC-12` exige um "motivo da falha" e não oferece vocabulário; este é **P** na forma, **F**/**D** nas causas.

| Código | Causa | Origem da causa |
|---|---|---|
| `WEBHOOK_DELIVERY_TIMEOUT` | sem resposta completa em 10s | `RNF-06`/`TEC-25` **F** |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | resposta fora da faixa 2xx (inclui 3xx, ver 6.2) | **D** |
| `WEBHOOK_DELIVERY_CONNECTION_ERROR` | DNS, TLS, conexão recusada — sem resposta HTTP | **D** |
| `WEBHOOK_ATTEMPTS_EXHAUSTED` | qualificador de DLQ: seis chamadas consumidas | `DEC-05`/`DEC-06` **F** |
| `WEBHOOK_SUBSCRIPTION_INACTIVE` | assinatura desativada com evento em trânsito | **P** (6.6) |

---

## 10. Observabilidade

A transcrição dá exatamente três coisas nesta área: o logger Pino já integrado (`COD-07`), o `requestLogger` com `X-Request-Id` (`CONTEXT.md` §9), e `RF-10` (logar quem executou o replay). **Métricas e tracing não têm uma linha de origem.** As três subseções abaixo têm, portanto, três níveis de confiança diferentes, e estão rotuladas.

### 10.1 Logs — origem sólida

Reuso do singleton `logger` de `src/shared/logger/index.ts`, sem dependência nova (`COD-07`, **F**). O padrão de evento existe no código e é seguido literalmente: `logger.<nível>({ campos }, 'nome_do_evento')`, nome em snake_case — é assim que `http_request`, `server_started`, `shutdown_initiated` e `bootstrap_failed` são emitidos hoje (**F**).

| Evento | Nível | Campos | Emissor |
|---|---|---|---|
| `webhook_event_enqueued` | `info` | `eventId`, `outboxId`, `subscriptionId`, `orderId`, `toStatus`, `requestId` | API, dentro de `publishWebhookEvent` |
| `webhook_poll_cycle` | `debug` | `claimed`, `durationMs` | worker, por ciclo |
| `webhook_delivery_attempt` | `info` | `eventId`, `outboxId`, `subscriptionId`, `attemptNumber`, `requestId` | worker, antes do POST |
| `webhook_delivery_succeeded` | `info` | idem + `responseStatus`, `durationMs` | worker |
| `webhook_delivery_failed` | `warn` | idem + `failureReason`, `responseStatus?`, `nextAttemptAt` | worker |
| `webhook_dead_lettered` | `error` | `eventId`, `outboxId`, `deadLetterId`, `attemptCount`, `failureReason` | worker |
| `webhook_orphan_reclaimed` | `warn` | `outboxId`, `eventId`, `heldForMs` | worker, passo 1 do ciclo (6.5) |
| `webhook_replay_requested` | `info` | **`userId`**, `deadLetterId`, `eventId`, `outboxId`, `requestId` | API, endpoint de replay |
| `webhook_poll_failed` | `error` | `err` | worker, limite de erro do ciclo (9.5) |
| `worker_bootstrap_failed` | `fatal` | `err` | `src/worker.ts` |

`webhook_replay_requested` com `userId` é `RF-10` literal (Sofia `[09:36]`: "o endpoint de admin tem que logar quem fez o replay, pra auditoria") — **F**. Os demais nomes de evento são **P** na nomenclatura, **D** no fato de existirem.

O worker herda `base: { service: 'order-management-api', env }` do logger existente. **Isso é um defeito de observabilidade conhecido** (**P**): os logs dos dois processos ficam indistinguíveis por `service`. Corrigir exige tocar `src/shared/logger/index.ts` — está na checklist 11.2, não aplicado.

**Nunca logar**: `secret`, `previousSecret`, o header `X-Signature` calculado, e o corpo de resposta do cliente. O `redact` atual **não** protege nenhum deles (8.3).

### 10.2 Métricas — proposta sem origem

> **Ninguém decidiu métrica alguma nesta feature.** Não há `prom-client`, `@opentelemetry/*` nem qualquer biblioteca de métrica em `package.json` (**F**), e `DEC-11`/`RES-03` proíbem dependência nova. **Exportador e coletor não foram decididos por ninguém, e este documento não os escolhe.**

O que ele faz é definir **as grandezas e como obtê-las com o que existe hoje** — cada uma com a query ou o evento de log que a produz. Quem for instrumentar depois tem o alvo pronto; quem for operar amanhã já consegue medir por SQL.

| Grandeza | Por que importa | Como obter hoje |
|---|---|---|
| Profundidade da outbox | detecta worker morto — o sintoma nº 1 do risco principal de [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) | `COUNT(*) WHERE status='PENDING' AND nextAttemptAt <= NOW()` |
| Idade do evento pendente mais antigo | mede o sintoma que `RNF-01` descreve ("que não fique pendurado") melhor que a latência média | `NOW() - MIN(createdAt)` sobre o mesmo filtro |
| Latência de entrega ponta a ponta | confronta o desenho com `RNF-01` | `attemptedAt - outbox.createdAt` na primeira `WebhookDelivery` bem-sucedida |
| Taxa de sucesso por assinatura | identifica o cliente instável que gera 6× o volume (`ABE-01`) | `success` agrupado por `subscriptionId` em `webhook_deliveries` |
| Volume de DLQ por janela | dispara ação humana — não há alerta automático (`ADI-01` fora de escopo) | `COUNT(*)` em `webhook_dead_letter` por `createdAt` |
| Distribuição de `attemptNumber` | mostra se o backoff está resolvendo ou só adiando | `GROUP BY attemptNumber` em `webhook_deliveries` |
| Linhas órfãs recuperadas | mede instabilidade do processo do worker | contagem de `webhook_orphan_reclaimed` no log |
| Duração do ciclo de polling | detecta saturação do lote | `durationMs` de `webhook_poll_cycle` |

**Sem alarme não há observabilidade**, e alarme depende de supervisão do processo, que [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) deixou em aberto e que `docker-compose.yml` não tem onde declarar. As grandezas acima são inúteis sem alguém olhando: isso é ponto aberto, não entrega.

### 10.3 Tracing — proposta sem origem

> **Não existe tracing no sistema.** Nenhum `traceparent`, nenhum contexto propagado, nenhuma biblioteca. O único primitivo de correlação é o `X-Request-Id` de `src/middlewares/request-logger.middleware.ts` (**F**), que nasce em `req.id` e **morre na fronteira do Express** — o worker roda em outro processo (`DEC-03`).

Proposta (**P**): propagar o `requestId` da chamada que mudou o status para dentro da linha da outbox, e dali para cada linha de entrega.

```
PATCH /orders/:id/status
   requestLogger gera X-Request-Id → req.id
        │
        ├─ log 'http_request'      { requestId, method, path, statusCode, durationMs, userId }
        └─ changeStatus → publishWebhookEvent(tx, …, req.id)
                  └─ webhook_outbox.requestId  ←── atravessa a fronteira de processo
                            │
                            └─ worker: todo log de entrega carrega o mesmo requestId
                                       webhook_deliveries.requestId
```

Com isso, `grep` por um `requestId` junta a chamada HTTP que mudou o status às até seis tentativas de entrega que ela originou, horas depois, em outro processo. É o mais próximo de tracing distribuído que se obtém sem dependência nova.

Duas escolhas explícitas:

- **Nome `requestId`, não `correlationId`** (**D**): o vocabulário já existe no código (`req.id`, header `X-Request-Id`, campo `requestId` no log de `requestLogger`). Criar sinônimo é dívida de glossário.
- **O `requestId` não vai ao cliente.** Nenhum header novo no contrato de saída (7.8) — [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md) congelou cinco headers, e o FDD não redecide ADR.

Limite honesto: eventos criados fora de um request HTTP (um replay é um request; uma futura rotina não seria) ficam com `requestId` nulo. Por isso a coluna é opcional.

---

## 11. Integração com o sistema existente

### 11.1 Pontos de acoplamento

Dez arquivos reais. Cada caminho e cada símbolo abaixo foi lido do repositório.

**1. `src/modules/orders/order.service.ts` — método `changeStatus` (linha 126)**
O acoplamento principal. A chamada `publishWebhookEvent(tx, order, fromStatus, toStatus, requestId)` (`COD-14`, **F**) entra **entre a linha 167** (fim de `tx.orderStatusHistory.create`) **e a linha 169** (`const refreshed = await tx.order.findUnique`), dentro do `prisma.$transaction(async (tx) => {...})` aberto na linha 131. Recebe o `tx` — nunca `this.prisma` —, e é isso que faz `RF-11` valer: exceção lançada ali derruba `orders`, `order_status_history` e `stock_quantity` junto. O método passa a ter uma dependência de escrita a mais no caminho crítico; é o **único comportamento existente que muda** nesta feature.

**2. `src/modules/orders/order.service.ts` — método `create` (linha 50)**
Segundo ponto de disparo **potencial**, não prescrito. O pedido nasce em `PENDING` e grava o primeiro `orderStatusHistory` com `reason: 'order created'` (linha 111) dentro de outra transação — mesmo padrão de acoplamento. **Ninguém decidiu se a criação gera evento** (seção 3). Citado aqui para que quem implementar saiba que o lugar existe e que a omissão é deliberada.

**3. `src/routes/index.ts` — `buildApiRouter` (linha 21)**
Duas linhas novas, e a segunda inaugura um namespace:
```typescript
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooksAdmin));
```
O tipo `Controllers` (linha 13) ganha as duas entradas. **`/admin` não existe hoje na API** — `buildApiRouter` monta apenas `/auth`, `/users`, `/customers`, `/products` e `/orders` (**F**). O caminho literal de `TEC-22` cria um nível de rota que o sistema nunca teve, e **nenhuma convenção existente cobre como ele deve crescer**. Manter o caminho é obrigação (é fonte); registrar que ele abre precedente é honestidade.

**4. `src/app.ts` — `buildControllers` (linha 26) e `buildApp` (linha 55)**
Injeção manual no molde exato das linhas 42–44 (`OrderRepository` → `OrderService(orderRepository, prisma)` → `OrderController`). O `WebhookService` recebe `(webhookRepository, prisma)` porque precisa do client para transações, como o `OrderService`. `AppDependencies` (linha 22) **não muda**: o cliente HTTP é o `fetch` global e vive no worker, não na API.

**5. `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole`**
`router.use(authenticate)` no topo do router de webhooks, no molde de `order.routes.ts` linha 14. No router admin, `requireRole('ADMIN')` encadeado depois (`DEC-12`/`COD-09`, **F**). `requireRole` lança `ForbiddenError` → 403 (7.7). Nenhuma alteração no arquivo. Consequência aceita de `DEC-20`: **qualquer usuário autenticado opera webhook de qualquer customer**, já que `customerId` vem do body ou da query e não do token — `ADI-06` é a saída, sem gatilho definido.

**6. `src/middlewares/validate.middleware.ts` — `validate`**
`validate({ body, query, params })` por rota. Carrega a regra `https` de `DEC-09` no schema Zod — e é exatamente por passar por aqui que a resposta sai como `VALIDATION_ERROR` (ressalva 1 da seção 9.6).

**7. `src/shared/errors/` — `app-error.ts`, `http-errors.ts`, `index.ts`**
As classes de 9.6.1 estendem `NotFoundError`, `BadRequestError`, `ConflictError` e `UnprocessableEntityError`, no molde de `InvalidStatusTransitionError extends ConflictError` (linha 45) e `InsufficientStockError extends UnprocessableEntityError` (linha 55). Exportadas pelo barrel. **`src/middlewares/error.middleware.ts` não muda** (`COD-08`) — ele já serializa qualquer `AppError` em `{ error: { code, message, details? } }`.

**8. `src/shared/logger/index.ts` — `logger`**
Singleton Pino reusado pelos dois processos (`COD-07`, **F**). Duas correções exigidas, ambas não aplicadas: `'*.secret'` e `'*.previousSecret'` em `redactPaths` (8.3), e distinção de `service` entre API e worker (10.1).

**9. `src/config/database.ts` — `createPrismaClient()`**
O worker chama `createPrismaClient()` e **não** importa o singleton `prisma` exportado na linha 10 — `PrismaClient` é por processo (`DEC-14`/`COD-12`, **F**). Mesma `DATABASE_URL` (`COD-16`/`RES-02`). Consequência dimensional já registrada em [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md): dois pools de conexão contra o mesmo MySQL, e o dimensionamento nunca foi revisto.

**10. `src/shared/http/response.ts` — `paginated<T>`**
Usado em 7.2 e 7.5. Sem envelope para recurso individual, conforme `CONTEXT.md` §8.

### 11.2 Mudanças exigidas em arquivos que este documento não pode alterar

> **Nenhuma das linhas abaixo foi aplicada.** A entrega é puramente documental: `src/`, `prisma/`, `tests/` e as configurações (`package.json`) estão fora de alcance por regra do desafio. Esta é a lista que o desenvolvedor executa no primeiro dia.

| # | Arquivo | Mudança | Origem |
|---|---|---|---|
| 1 | `prisma/schema.prisma` | quatro models e o enum da seção 5 | **P** |
| 2 | `prisma/schema.prisma` | relação inversa `webhooks WebhookSubscription[]` em `Customer` — exigência do Prisma | **D** |
| 3 | `prisma/migrations/` | migration gerada por `npm run db:migrate` | **D** |
| 4 | `src/modules/orders/order.service.ts` | chamada a `publishWebhookEvent` entre as linhas 167 e 169 | `COD-13`/`COD-14` **F** |
| 5 | `src/modules/orders/order.controller.ts` | propagar `req.id` até `changeStatus` | **P** (10.3) |
| 6 | `src/routes/index.ts` | dois `router.use` e duas entradas em `Controllers` | **D** |
| 7 | `src/app.ts` | instanciação em `buildControllers` | **D** |
| 8 | `src/config/env.ts` | as variáveis da seção 12.2 no `envSchema` | **D** |
| 9 | `src/shared/logger/index.ts` | `'*.secret'` e `'*.previousSecret'` em `redactPaths`; `service` distinto no worker | **P** |
| 10 | `src/shared/errors/index.ts` | export das classes `Webhook*Error` | **D** |
| 11 | **`package.json`** | **script `worker` — `TEC-20` o decidiu e ele não existe hoje** | `TEC-20` **F** |
| 12 | `tests/setup.ts` | quatro `deleteMany()` novos no `beforeEach`, **antes** de `customer.deleteMany()` (ordem inversa de FK) | **D** |
| 13 | `tests/helpers/factories.ts` | `createTestWebhookSubscription` no padrão existente | `CONTEXT.md` §17.10 **D** |
| 14 | `docker-compose.yml` | serviço do worker — hoje só há `mysql`, e não há onde declarar o processo | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) em aberto |

O item 11 merece destaque: `TEC-20` registra Larissa decidindo "criar um src/worker.ts e um script 'npm run worker'" (`[09:11]`), e `package.json` hoje tem apenas `dev`, `build`, `start`, `db:migrate`, `db:reset`, `db:seed`, `test`, `test:watch`, `lint` e `format` (**F**). O script proposto, simétrico ao par `dev`/`start` existente:

```jsonc
"worker":     "tsx watch --env-file=.env src/worker.ts",   // simétrico a "dev"
"worker:start": "node --env-file=.env dist/worker.js"      // simétrico a "start"
```

O segundo nome é **P** — `TEC-20` só nomeou um script.

---

## 12. Dependências e compatibilidade

### 12.1 Nenhuma dependência nova

`DEC-11` / `RES-03`, e Bruno sobre o logger: "já tá no projeto inteiro. Não vamos botar nada novo" (`COD-07`, **F**). O `package.json` fica intacto exceto pelos scripts (11.2, item 11).

| Necessidade | Resolvida por | Verificação |
|---|---|---|
| cliente HTTP com timeout | **`fetch` global + `AbortSignal.timeout()`** | `engines.node >= 20` em `package.json`; não há `axios`, `undici` nem `node-fetch` nas dependências (**F**) |
| HMAC-SHA256 | `node:crypto` (`createHmac`) | builtin |
| geração de UUID | `crypto.randomUUID()` ou `uuid@11.0.3`, já dependência usada em `request-logger.middleware.ts` (**F**) | — |
| validação | `zod@3.23.8` | já usado |
| acesso a dados | `@prisma/client@5.22.0` | já usado |
| logging | `pino@9.5.0` | já usado |

Observação factual sem consequência: **`pino-http@10.3.0` está declarado e não é usado em lugar nenhum** — `requestLogger` é escrito à mão. Registrado para que ninguém conclua que há um padrão de logging HTTP a seguir que na verdade não está em uso.

### 12.2 Variáveis de ambiente

Entram no `envSchema` Zod de `src/config/env.ts`, no padrão existente (`z.coerce.number()`, defaults, `loadEnv()` abortando com `process.exit(1)` — **F**). **O default É a decisão**: tornar o valor configurável não reabre nada.

| Variável | Default | Significado do default |
|---|---|---|
| `WEBHOOK_POLL_INTERVAL_MS` | `2000` | `DEC-02` **F** |
| `WEBHOOK_DELIVERY_TIMEOUT_MS` | `10000` | `RNF-06`/`TEC-25` **F** |
| `WEBHOOK_MAX_ATTEMPTS` | `5` | `DEC-05` — retentativas **além** do envio inicial (`TEC-24`) **F** |
| `WEBHOOK_PAYLOAD_MAX_BYTES` | `65536` | `DEC-18`/`RNF-05`, 64KB **F** |
| `WEBHOOK_SECRET_GRACE_PERIOD_HOURS` | `24` | `TEC-11` **F** |
| `WEBHOOK_POLL_BATCH_SIZE` | `20` | **P** — `TEC-02` diz "batch pequeno" e não dá número |
| `WEBHOOK_PROCESSING_LEASE_MS` | `60000` | **P** — 6.5; nenhuma fonte |

A progressão de backoff **não** é variável de ambiente (6.3). `DATABASE_URL` é a mesma nos dois processos (`COD-16`/`RES-02`, **F**).

### 12.3 Compatibilidade

- **Nenhuma quebra de contrato existente.** Nenhum endpoint atual muda forma de request ou response (**D**).
- **`changeStatus` ganha um modo de falha novo**: payload acima de 64KB derruba a mudança de status (`WEBHOOK_PAYLOAD_TOO_LARGE`). É o único caminho em que um cliente sem webhook nenhum poderia ser afetado — e não pode, porque `DEC-16` não insere linha quando não há assinatura, e sem inserção não há medição de tamanho.
- **Migration é aditiva**: quatro tabelas novas, uma relação inversa em `Customer`, nenhuma coluna alterada em tabela existente (**D**).
- **A API sobe sem o worker.** Eventos acumulam em `PENDING` e nada quebra visivelmente — que é exatamente o risco silencioso de [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).

---

## 13. Critérios de aceite técnicos

Padrão do projeto: Vitest 2.1.4, testes de integração contra banco real via `supertest`, app montado por `buildApp({ prisma })`, sem mock de repositório; `fileParallelism: false`, `pool: 'forks'`, `singleFork: true` (`vitest.config.ts`, `CONTEXT.md` §14, **F**). Arquivos prescritos, **não criados**: `tests/webhooks.test.ts` e `tests/webhook-worker.test.ts`.

> **O `singleFork` tem consequência direta no teste do worker**: o loop de polling **não** roda vivo durante os testes. O que se testa é a função de processamento de um ciclo, invocada diretamente, com as linhas semeadas no banco. Ninguém deve tentar testar o `setInterval`.

| # | Critério | Como verificar | Origem |
|---|---|---|---|
| CA-01 | Rollback de `changeStatus` não deixa linha na outbox | forçar erro após a inserção; `webhook_outbox` vazia e status do pedido inalterado | `RF-11`/`RNF-04` **F** |
| CA-02 | Status sem assinante não gera linha | cliente sem webhook para `PAID`; transição `PENDING→PAID`; zero linhas | `DEC-16` **F** |
| CA-03 | Cliente com dois endpoints assinando o mesmo status gera duas linhas com `eventId` distintos | contar linhas e comparar `eventId` | **P** (6.1) |
| CA-04 | Payload tem exatamente os nove campos de `TEC-17`, sem `items` | comparação de chaves | `TEC-17`/`ALT-08` **F** |
| CA-05 | Payload acima de 64KB derruba a transação | `WEBHOOK_PAYLOAD_TOO_LARGE`, 422, status do pedido inalterado | `DEC-18` **F** |
| CA-06 | `X-Signature` reproduzível | recalcular HMAC-SHA256 com a secret sobre o corpo recebido | `DEC-08` **F** |
| CA-07 | Durante o grace period vão duas assinaturas, a antiga válida | rotacionar, entregar, validar com a secret antiga | `TEC-11` **F** / formato **P** |
| CA-08 | `url` `http` é recusada | `POST` com `http://` → 400 | `DEC-09`/`TEC-26` **F** |
| CA-09 | Secret aparece na criação e na rotação, **nunca** em `GET`/`PATCH` | inspecionar os quatro corpos | `TEC-14` **D** |
| CA-10 | Falha agenda a retentativa no intervalo correto | endpoint devolvendo 500; conferir `attemptCount` e `nextAttemptAt` na tabela de 6.3 | `DEC-05`/`TEC-23` **F** |
| CA-11 | Linha com `nextAttemptAt` futuro não é selecionada | semear e rodar um ciclo; nenhuma chamada | **P** |
| CA-12 | **Seis** chamadas HTTP antes da DLQ | contar requisições no endpoint falso | `TEC-24` **D** |
| CA-13 | Timeout de 10s conta como falha | endpoint que dorme 11s; `WEBHOOK_DELIVERY_TIMEOUT` | `RNF-06` **F** |
| CA-14 | Esgotamento cria linha em `webhook_dead_letter` com payload, motivo e timestamp | inspecionar a linha | `DEC-06`/`TEC-12` **F** |
| CA-15 | Replay preserva o `eventId` e cria linha nova | comparar `eventId` e `replayOfId` | **D** (5.6) |
| CA-16 | Replay exige `ADMIN` | token `OPERATOR` → 403 | `DEC-12` **F** |
| CA-17 | Replay loga `userId` | capturar `webhook_replay_requested` | `RF-10` **F** |
| CA-18 | Linha órfã em `PROCESSING` é recuperada | semear com `updatedAt` antigo; um ciclo devolve a `PENDING` sem incrementar `attemptCount` | **P** (6.5) |
| CA-19 | Lote é processado em série, em ordem de `createdAt` | três linhas do mesmo `order_id`; conferir a ordem de chegada | `TEC-27`/`ADR-007` **D** |
| CA-20 | `deliveries` devolve envelope paginado, no máximo 100, mais recente primeiro | semear 120 entregas | `RF-06` **F** / forma **D** |
| CA-21 | Toda tentativa gera uma `WebhookDelivery`, com sucesso ou falha | contar linhas após um ciclo de falhas | **D** (5.4) |
| CA-22 | Erros saem no formato `{ error: { code, message } }` com código `WEBHOOK_*` | inspecionar respostas de 404 e 409 | `COD-08` **F** |

### Itens que este documento não pode fechar

1. **`DEC-09` × `TEC-16`** — `WEBHOOK_INVALID_URL` não dispara no caminho HTTP normal (ressalva 1 da seção 9.6).
2. **`WEBHOOK_SECRET_REQUIRED` sem gatilho** — nomeado dois minutos antes de `TEC-14` torná-lo inalcançável (ressalva 2).
3. **`DELETE` de assinatura com histórico** — FK versus preservação da evidência de `RF-06` (6.6 e 7.4).
4. **Disparo em `OrderService.create`** — nunca decidido (seção 3).
5. **Armazenamento da secret em repouso** — decisão da revisão de `RNF-10` (8.3).
6. **Formato do `X-Signature` durante a rotação** — proposta de maior alcance deste FDD, endereçada a Sofia (8.2, itens `A-2` e `A-3` do registro).
7. **Supervisão do worker** — sem lugar onde declarar o processo ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)).
8. **`RNF-01`, ordem sob retry** — escalados pela [RFC](RFC.md) e não decididos (9.1, 9.4).

---

## 14. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação | Detalhe |
|---|---|---|---|---|
| Worker cai e ninguém percebe: a API segue aceitando pedidos, a outbox acumula, clientes param de receber em silêncio | Média | Alto | Profundidade e idade da outbox (10.2) são o sinal; **falta o alarme e a supervisão**, sem dono | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| Linhas presas em `PROCESSING` a cada deploy do worker | **Alta** | Alto | Shutdown gracioso + lease (6.5); log `webhook_orphan_reclaimed` como sinal | **P** — sem fonte |
| `eventId` perdido no replay quebraria a dedup do cliente | Média | Alto | `eventId` preservado por todo o ciclo, com CA-15 travando a regressão | 5.6 |
| Duplicata como consequência **garantida**: cliente que processa em mais de 10s recebe de novo sem falha real | Alta | Médio | Documentar no portal antes do go-live e destacar `X-Event-Id` | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md) |
| Lote processado em paralelo por engano quebra a ordem sem sinal nenhum | Média | Médio | Serialização estrita em 6.2 + CA-19; **é invisível em teste de caminho feliz** | [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md) |
| Secret em claro no banco e ausente do `redact` do Pino | Média | Alto | 8.3 e item 9 da checklist; **bloqueado pela revisão de `RNF-10`** | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| Formato de duas assinaturas rejeitado na revisão de segurança, já com clientes integrados | Média | Médio | Escalar 8.2 **antes** da sprint 3, não junto com o código | **P** |
| Autorização frouxa: qualquer autenticado opera webhook de qualquer customer | Alta | Alto | Aceito como temporário por Sofia (`DEC-20`); `ADI-06` sem gatilho | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Payload de 64KB derruba mudança de status legítima | Baixa | Alto | Trade-off deliberado; medir tamanho (10.2) antes de virar incidente | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Replay manual não escala: cliente fora por 15h gera uma linha de DLQ por evento | Média | Médio | Nenhuma operação em lote decidida | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) |
| Duas tabelas crescendo sem limite (`ADI-04` fora de escopo) | Alta | Médio | Nenhuma; é escopo adiado com consciência | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |

---

## Registro de derivações e de aprovações pendentes

Tudo que este documento acrescentou além da fonte literal está aqui, em um lugar só, e **cada item tem um dos dois destinos abaixo — nenhum fica solto**:

- **`D` — derivação explícita.** O item é consequência necessária de um item que tem origem na transcrição ou no código. A coluna **Cadeia de derivação** mostra o caminho completo: o item de fonte, o passo lógico e o resultado. Uma derivação não precisa de aprovação para ser rastreável — precisa de que a cadeia esteja escrita e possa ser conferida. Se a cadeia estiver errada, o item cai.
- **`A` — aprovação pendente.** O item é **proposta**, não informação de origem. Entra no pacote como **questão endereçada a uma pessoa nomeada, em um fórum que a própria reunião criou**: a **revisão de segurança de Sofia** (`RNF-10` `[09:46]` / `RES-06` `[09:49]`) ou a **sessão de revisão de design com Bruno e Diego** que Larissa assumiu abrir (`[09:50]`, nota de leitura (b)-2). Até a aprovação, o item **não é requisito** e não deve ser tratado como tal por quem implementa.

### D — derivações explícitas (13)

| # | Item | Seção | Cadeia de derivação |
|---|---|---|---|
| D-1 | `attemptCount` e `nextAttemptAt` na outbox | 5.3 | `DEC-05` `[09:17]` fixa 5 tentativas com intervalos 1m/5m/30m/2h/12h **+** `TEC-03` `[09:09]` põe o worker em polling sem estado em memória ⇒ contagem e instante da próxima tentativa **têm** de estar na linha, ou o intervalo decidido não é executável |
| D-2 | `PENDING` cobre "aguardando retentativa"; `FAILED` é terminal | 5.1 | `TEC-01` `[09:08]` nomeia exatamente quatro estados (pendente, processando, falhou, entregue) **+** `DEC-05` cria a espera entre tentativas ⇒ a espera tem de caber em um dos quatro; `FAILED` é o que `DEC-06` `[09:18]` encaminha à DLQ, logo é o terminal |
| D-3 | Um `eventId` por assinatura, não por mudança de status | 6.1 | `DEC-16` `[09:34]` filtra na inserção (uma linha por assinatura interessada) **+** `TEC-04` `[09:25]` gera o UUID "quando o evento entra na outbox", único por evento ⇒ uma linha ⇒ um `eventId`. Compartilhá-lo faria a dedup de `DEC-10` descartar a entrega do segundo endpoint |
| D-4 | Recuperação de linhas órfãs em `PROCESSING` | 6.5 | `TEC-01` cria o estado "processando" **+** `DEC-03`/`RES-05` `[09:11]` põem o worker em processo separado, que reinicia ⇒ um estado do qual só se sai por sucesso ou falha precisa de regra de saída no reinício. *O valor do lease é `A-7`* |
| D-5 | Teto no `responseBody` armazenado | 5.4 | `DEC-18` `[09:24]` estabelece o princípio: dado de tamanho não controlado ganha teto e falha com erro, não trunca em silêncio. `responseBody` é o único outro campo de tamanho não controlado do desenho, e vem de terceiro ⇒ mesmo princípio, mesma fonte |
| D-6 | Janela de 100 + envelope paginado em `RF-06` | 7.5 | os 100 são literais de `RF-06` `[09:34]` **+** o envelope vem do código: `paginated<T>` em `src/shared/http/response.ts`, usado por toda listagem, sob `RES-03`/`DEC-11` `[09:30]` (reuso de padrões) |
| D-7 | 3xx tratado como falha, sem seguir redirect | 6.2 | `DEC-09`/`RNF-07` `[09:23]` tornam a URL https cadastrada a **única** autorizada **+** `DEC-08` `[09:22]` assina o corpo para aquele endpoint ⇒ seguir redirect entregaria payload assinado a host que nunca passou pela validação |
| D-8 | `WEBHOOK_ROTATION_IN_PROGRESS` e `WEBHOOK_SUBSCRIPTION_IN_USE` | 9.6.1 | `TEC-16` `[09:28]` fixa o prefixo e fecha com "etc." **+** os dois caminhos são criados pela própria fonte: rotação com grace period (`TEC-11` `[09:21]`) e `DELETE` (`RF-03`) sobre assinatura com histórico (`RF-06`) **+** `ConflictError` já aceita código próprio (`src/modules/orders/order.service.ts` linha 187) |
| D-9 | `WEBHOOK_SUBSCRIPTION_INACTIVE` como motivo de DLQ | 6.6 | `TEC-13` `[09:21]` cria o campo "estado ativo" **+** `DEC-06` `[09:18]` exige "motivo da falha" na DLQ ⇒ desativar assinatura com evento em trânsito é um motivo que a fonte criou e não nomeou |
| D-10 | `customerId` não editável no `PATCH` | 7.3 | `RES-07` `[09:32]` põe o `customerId` fora do JWT, no request **+** `RF-06` `[09:34]` prende o histórico à assinatura ⇒ permitir a troca faria o histórico de um cliente mudar de dono por um `PATCH` |
| D-11 | Nomes de tabela e dimensionamentos de coluna | 5.2, 5.4 | `TEC-01` e `TEC-12` já nomeiam `webhook_outbox` e `webhook_dead_letter` em snake_case **+** `prisma/schema.prisma` mapeia todo model com `@@map` em snake_case plural ⇒ os outros dois nomes seguem a mesma regra |
| D-12 | Nomes dos eventos de log | 10.1 | código: `src/middlewares/request-logger.middleware.ts` emite `http_request` em snake_case **+** `COD-07` `[09:29]` e `RES-03` mandam reusar o Pino existente sem inventar padrão novo |
| D-13 | Script `worker:start` | 11.2 | `TEC-20` `[09:11]` decide o script `worker` **+** `package.json` já pareia `dev` (tsx watch) com `start` (node dist) ⇒ o par simétrico para o worker é derivação da convenção existente, não invenção |

### A — propostas pendentes de aprovação (8)

Cada linha é uma **questão endereçada**, não uma decisão registrada.

| # | Proposta | Seção | Aprovador e fórum | O que quebra se for rejeitada |
|---|---|---|---|---|
| A-1 | Tabela `webhook_deliveries` inteira | 5.4 | Bruno e Diego, na sessão de revisão de design (`[09:50]`) | é a **resposta proposta a `ABE-04`**, questão que a própria reunião deixou aberta; sem ela `RF-06` não é implementável, e uma resposta diferente muda a seção 7.5 |
| A-2 | Duas assinaturas em `X-Signature` durante o grace period | 8.2 | **Sofia**, revisão de segurança (`RNF-10`) | é a **derivação de maior alcance deste documento** e a mais exposta: sem ela, o grace period de 24h de `TEC-11` fica sem significado operacional; com ela, o contrato de saída muda para todos os clientes |
| A-3 | Formato `sha256=<hex>`, header único com vírgula | 8.2 | **Sofia**, revisão de segurança | o contrato de saída muda. Decidir junto com `A-2`, não depois |
| A-4 | Secret armazenada em claro, sem cifragem | 8.3 | **Sofia**, revisão de segurança — ela nomeou "geração de secret" como foco (`[09:46]`) | é exatamente o objeto da revisão que `RES-06` tornou bloqueante; nenhuma outra pessoa tem autoridade para fechar |
| A-5 | Coluna `requestId` e sua propagação entre processos | 5.3, 10.3 | Bruno e Diego, revisão de design — toca `order.controller.ts` | sem ela não há correlação entre a requisição da API e a entrega feita pelo worker |
| A-6 | Métricas como grandezas + query, sem exportador | 10.2 | Larissa e Diego, revisão de design | o alvo de `RNF-01` continua sem instrumentação, e as métricas do [PRD §4.2](PRD.md) ficam sem fonte de dado |
| A-7 | `WEBHOOK_POLL_BATCH_SIZE = 20` e `WEBHOOK_PROCESSING_LEASE_MS = 60000` | 12.2 | Diego, revisão de design | `TEC-02` `[09:08]` diz "batch pequeno" e não dá número; os dois valores são tuning e nada mais depende deles |
| A-8 | Prefixo `whsec_` na secret | 7.1 | **Sofia**, revisão de segurança | nada estrutural; é convenção de formato do valor que ela vai revisar de qualquer forma |

**Premissa de fluxo declarada separadamente**: a seção 6 é construída sobre a perna "aceitar e documentar" da quebra de ordem sob retry — recomendação da [RFC](RFC.md) **não decidida por ninguém** (9.4). Ela não está nas tabelas acima porque não é proposta deste documento: é uma escolha entre duas saídas que a RFC já escalou, e o dono é Larissa ([PRD §13, questão 5](PRD.md#13-questões-abertas-de-produto)).

---

## Fontes

**Índice da transcrição** (`.scratch/fontes/transcricao-index.md`) — **enumeração literal: cada ID abaixo é citado no corpo deste documento**, e nenhum ID citado ficou de fora (97 IDs):

DEC-02, DEC-03, DEC-04, DEC-05, DEC-06, DEC-07, DEC-08, DEC-09, DEC-10, DEC-11, DEC-12, DEC-14, DEC-15, DEC-16, DEC-17, DEC-18, DEC-19, DEC-20 · RF-01, RF-02, RF-03, RF-04, RF-05, RF-06, RF-07, RF-08, RF-09, RF-10, RF-11 · RNF-01, RNF-02, RNF-04, RNF-05, RNF-06, RNF-07, RNF-10 · RES-01, RES-02, RES-03, RES-04, RES-05, RES-06, RES-07 · ALT-06, ALT-07, ALT-08 · ADI-01, ADI-02, ADI-03, ADI-04, ADI-05, ADI-06 · ABE-01, ABE-02, ABE-03, ABE-04 · COD-04, COD-05, COD-06, COD-07, COD-08, COD-09, COD-10, COD-11, COD-12, COD-13, COD-14, COD-16 · TEC-01, TEC-02, TEC-03, TEC-04, TEC-05, TEC-06, TEC-07, TEC-08, TEC-09, TEC-10, TEC-11, TEC-12, TEC-13, TEC-14, TEC-15, TEC-16, TEC-17, TEC-18, TEC-19, TEC-20, TEC-21, TEC-22, TEC-23, TEC-24, TEC-25, TEC-26, TEC-27, TEC-28, TEC-29 · notas de leitura (a), (b) e (c).

**Documentos do pacote**: [RFC](RFC.md) · [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) · [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) · [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) · [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) · [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md) · [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) · [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md).

**Arquivos reais lidos para este documento**: `src/modules/orders/order.service.ts` (`changeStatus` 126, `create` 50, `delete` 181) · `src/modules/orders/order.schemas.ts` · `src/modules/orders/order.routes.ts` (`buildOrderRouter`) · `src/routes/index.ts` (`buildApiRouter`, `Controllers`) · `src/app.ts` (`buildApp`, `buildControllers`, `AppDependencies`) · `src/server.ts` (`bootstrap`, shutdown) · `src/config/database.ts` (`createPrismaClient`) · `src/config/env.ts` (`envSchema`, `loadEnv`) · `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`, `AuthUser`) · `src/middlewares/error.middleware.ts` (`errorMiddleware`) · `src/middlewares/validate.middleware.ts` (`validate`) · `src/middlewares/request-logger.middleware.ts` (`requestLogger`) · `src/shared/errors/app-error.ts` (`AppError`) · `src/shared/errors/http-errors.ts` · `src/shared/logger/index.ts` (`redactPaths`) · `src/shared/http/response.ts` (`paginated`) · `prisma/schema.prisma` · `tests/setup.ts` · `package.json` · `docker-compose.yml` · `CONTEXT.md` §7–§9, §12, §14, §16, §17.
