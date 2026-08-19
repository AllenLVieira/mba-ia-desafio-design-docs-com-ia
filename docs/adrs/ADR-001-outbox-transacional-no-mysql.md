# ADR-001 — Outbox transacional no MySQL

## Status

**Accepted** — decisão fechada pela Tech Lead na reunião técnica do Sistema de Webhooks de Notificação de Pedidos (DEC-01, Larissa `[09:08]`).

> **Nota de contexto — revisão pendente.** Larissa registrou que abriria o documento de design e marcaria uma sessão com Bruno e Diego para revisar antes de começar a codar (`[09:50]`, nota de leitura (b)-2). A data dessa sessão não foi definida na reunião. O status `Accepted` reflete que a decisão foi tomada e fechada, não que a revisão já ocorreu.

## Contexto

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — fizeram pedido formal de notificação de mudanças de status de pedido (TEC-29, Marcos `[09:00]`).

O sistema não possui hoje **nenhum** mecanismo de notificação externa. A varredura registrada em `CONTEXT.md` §16 buscou em `src/`, `prisma/` e `package.json` os termos `webhook`, `event`, `queue`, `outbox`, `worker`, `job`, `retry`, `hmac`, `notify`, `bull`, `redis`, `amqp`, `kafka`, `sns`, `sqs` — zero ocorrências. Não há broker, fila ou cache provisionado: `docker-compose.yml` sobe apenas o MySQL.

A mudança de status já roda hoje dentro de uma transação carregada: atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` dos produtos do pedido (COD-01/02/03, Bruno `[09:04]`).

O requisito de integridade é absoluto: "se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível" (RNF-04, Diego `[09:06]`). Bruno reforçou: "Não pode ter caso de status mudar e evento não sair" (RF-11, `[09:40]`).

## Decisão

**Adotar o padrão outbox sobre o MySQL existente**, com a intenção de disparo gravada na mesma transação SQL da mudança de status.

1. **Tabela `webhook_outbox`** no banco já existente. Chave primária UUID, seguindo a convenção do projeto (DEC-19, Larissa `[09:51]`: "UUID, segue o padrão do resto do projeto. Tudo é uuid."). Campo de status com os valores pendente, processando, falhou e entregue, indexado, mais índice em `created_at` (TEC-01, Diego `[09:08]`).

2. **Ponto de acoplamento**: `src/modules/orders/order.service.ts`, método `changeStatus` (COD-13, Bruno `[09:40]`). A inserção acontece dentro da `prisma.$transaction()` já aberta ali. Se a inserção na outbox falhar, a transação inteira sofre rollback (RF-11).

3. **Interface de escrita**: função nova `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `tx` client da transação corrente e é invocada pelo `OrderService` (COD-14, Bruno `[09:41]`).

4. **Payload como snapshot na inserção**, não renderizado na hora do envio (DEC-15, Larissa `[09:52]`): "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou."

5. **Filtragem na inserção, não no despacho** (DEC-16, Bruno `[09:34]`): se nenhum webhook do customer assinou aquele status, o evento não é inserido. A lista de status por endpoint é o filtro (RF-05 / TEC-15, Marcos `[09:33]`).

6. **Campos do payload** (TEC-17/18/19, Diego `[09:43]`): `event_id`, `event_type` fixo em `"order.status_changed"`, `timestamp` em ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`. Os itens do pedido **não** são enviados (ALT-08).

7. **Limite de 64KB por payload**, com erro caso ultrapasse — sem truncamento (DEC-18 / RNF-05, Larissa e Diego `[09:24]`).

## Alternativas Consideradas

**Despacho síncrono dentro da transação de mudança de status** (ALT-01, Bruno `[09:04]`). Descartada: "A transação de mudança de status hoje já é pesada… Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos." Além do acoplamento de latência, a indisponibilidade do cliente causaria rollback indevido de uma mudança de status legítima.

**Redis Streams ou fila equivalente** (ALT-02, Diego `[09:07]`). Descartada como overengineering: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve." Formalizado como restrição em RES-01.

**Renderizar o payload no momento do envio** em vez de na inserção. Descartada em favor do snapshot (DEC-15): o evento deve refletir o estado no instante da mudança, não o estado no instante da entrega.

## Consequências

### Positivas

- **Atomicidade real.** Não existe estado em que o status mudou e o evento não foi registrado, nem o inverso (RNF-04). A garantia vem do próprio motor transacional do MySQL, sem coordenação distribuída.
- **Zero infraestrutura nova.** Nenhum serviço a provisionar, monitorar ou operar além do MySQL que o time já roda (RES-01).
- **Snapshot é auditável.** O payload guardado é evidência do que foi prometido ao cliente, independente de mutações posteriores no pedido.
- **Filtrar na inserção economiza linhas.** Eventos que ninguém assinou nunca entram na tabela (DEC-16).

### Negativas

- **A tabela `webhook_outbox` vira ponto de contenção de escrita** numa transação que já atualiza três tabelas (COD-01/02/03). Cada mudança de status passa a custar uma escrita adicional, e uma falha nessa escrita agora derruba a mudança de status — que antes não dependia dela.
- **Trade-off explícito: consistência acima de disponibilidade da escrita.** RF-11 escolhe deliberadamente que uma falha ao registrar a *promessa* de notificação impeça a *operação de negócio*. A alternativa (registrar best-effort, fora da transação) preservaria a mudança de status ao custo de eventos perdidos. A reunião escolheu o primeiro lado.
- **O snapshot duplica dados.** O payload congelado convive com a fonte de verdade em `orders`, e divergirá dela por design. Um cliente que compare o webhook recebido com um `GET /orders/:id` posterior verá diferença — e isso é o comportamento pretendido, não um bug.
- **Filtrar na inserção acopla escrita a configuração.** A transação de `changeStatus` passa a depender da leitura das assinaturas de webhook do customer. Mudar a assinatura de um cliente não reprocessa eventos passados: um endpoint cadastrado depois da mudança de status nunca receberá aquele evento, porque ele não chegou a existir.
- **O limite de 64KB falha em vez de degradar.** Um pedido que gere payload acima do teto derruba a transação de mudança de status (DEC-18 decidiu erro, não truncamento). Diego avaliou o risco como remoto — "Nenhum evento nosso vai chegar perto disso" (RNF-05) — mas a consequência, se acontecer, é bloqueio de operação de negócio.

## Pontos em aberto

- **ADI-04 — arquivamento das linhas entregues.** Diego mencionou arquivar após ~30 dias, mas colocou explicitamente fora do escopo desta feature (`[09:08]`). Sem isso, a `webhook_outbox` cresce indefinidamente.
- **Segundo ponto de disparo não decidido.** `CONTEXT.md` §17.2 identifica `src/modules/orders/order.service.ts`, método `create`, como um segundo lugar onde o pedido nasce em `PENDING` e grava o primeiro registro de histórico, com o mesmo padrão de acoplamento. A reunião tratou apenas de `changeStatus` (COD-13); se `order.created` (ou `order.status_changed` para `PENDING`) deve ou não gerar evento nunca foi discutido.
- **ABE-04 — histórico de entregas.** O schema que sustenta `GET /webhooks/:id/deliveries` (RF-06) não foi projetado na reunião; registrado em ADR-003, onde o registro de tentativas e falhas é decidido.

## Fontes

**Índice da transcrição** — enumeração literal dos IDs citados no corpo deste ADR: DEC-01, DEC-15, DEC-16, DEC-18, DEC-19 · RF-05, RF-06, RF-11 · RNF-04, RNF-05 · RES-01 · ALT-01, ALT-02, ALT-08 · ADI-04 · ABE-04 · COD-01, COD-13, COD-14 · TEC-01, TEC-15, TEC-17, TEC-29.

**Arquivos reais:** `src/modules/orders/order.service.ts` (`changeStatus`), `prisma/schema.prisma`, `docker-compose.yml`, `CONTEXT.md` §16 e §17.

**Relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) (quem consome a outbox), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) (o que acontece quando a entrega falha), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) (convenções de schema e módulo).
