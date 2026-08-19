# Índice de Fontes — Transcrição
## Reunião Técnica: Sistema de Webhooks de Notificação de Pedidos

---

## Participantes

| Nome | Papel | Primeira aparição |
|------|-------|-------------------|
| Larissa | Tech Lead | `[09:00]` |
| Marcos | Product Manager | `[09:00]` |
| Bruno | Engenheiro Pleno, time de Pedidos | `[09:00]` |
| Diego | Engenheiro Sênior, time de Plataforma | `[09:05]` |
| Sofia | Engenheira de Segurança | `[09:02]` |

---

## Índice de itens

### DEC — Decisões fechadas

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| DEC-01 | Padrão outbox em MySQL como mecanismo de entrega de webhooks (não síncrono, não Redis) | Larissa | `[09:08]` | "Tá decidido então: outbox em MySQL." |
| DEC-02 | Worker em polling com intervalo de 2 segundos | Larissa | `[09:10]` | "Vamos registrar isso como uma decisão. Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos." |
| DEC-03 | Worker deve rodar como processo separado da API | Diego | `[09:11]` | "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker." |
| DEC-04 | Ordering garantida apenas por order_id e somente com single-worker (limitação documentada) | Larissa | `[09:13]` | "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker." |
| DEC-05 | 5 tentativas de retry com backoff exponencial 1m/5m/30m/2h/12h | Larissa | `[09:17]` | "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h." |
| DEC-06 | DLQ persistida em tabela separada webhook_dead_letter (não marcação na própria outbox) | Diego | `[09:18]` | "Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento." |
| DEC-07 | Reprocessamento de DLQ via endpoint admin manual POST /admin/webhooks/dead-letter/:id/replay | Diego | `[09:18]` | "Manual via endpoint admin. Tipo um POST /admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente." |
| DEC-08 | HMAC-SHA256 sobre corpo do request, secret por endpoint, rotação com grace period de 24h | Sofia | `[09:22]` | "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h." |
| DEC-09 | TLS obrigatório: URL de webhook deve ser https; http recusado com erro de validação Zod | Sofia | `[09:23]` | "TLS obrigatório. URL do webhook tem que ser https. Se o cliente cadastrar http, recusamos com erro de validação. Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod." |
| DEC-10 | Garantia at-least-once; deduplicação por X-Event-Id é responsabilidade do cliente | Larissa | `[09:26]` | "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão." |
| DEC-11 | Reuso máximo de infraestrutura existente: AppError, Pino, error middleware, padrão de módulos, Zod, códigos de erro | Larissa | `[09:30]` | "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros." |
| DEC-12 | Endpoint de replay de DLQ exige role ADMIN; reaproveitamento do requireRole existente | Larissa | `[09:36]` | "Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe." |
| DEC-13 | Prazo de três sprints com revisão de segurança da Sofia incluída no fim | Larissa | `[09:47]` | "Combinado. Três sprints com a revisão da Sofia incluída no fim." |
| DEC-14 | PrismaClient instanciado separadamente no worker (mesmo banco, instância nova por ser outro processo) | Bruno | `[09:30]` | "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node." |
| DEC-15 | Payload do evento armazenado como snapshot na inserção na outbox (não renderizado na hora do envio) | Larissa | `[09:52]` | "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito." |
| DEC-16 | Filtragem de eventos feita na inserção na outbox (não no despacho): se nenhum webhook quer o status, não insere | Bruno | `[09:34]` | "Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela." |
| DEC-17 | Módulo de webhooks em src/modules/webhooks seguindo padrão existente; entry point do worker em src/worker.ts | Bruno | `[09:28]` | "Eu colocaria src/worker.ts como entry separada, e a lógica de processamento fica num arquivo dentro do módulo, tipo src/modules/webhooks/webhook.worker.ts ou webhook.processor.ts." |
| DEC-18 | Limite de 64KB no tamanho do payload; caso ultrapasse, retorna erro (não trunca) | Larissa | `[09:24]` | "64KB de limite, erro caso ultrapasse. Anotado, mas não vejo como decisão arquitetural separada, é só requisito não funcional." |
| DEC-19 | UUID como identificador primário na tabela outbox (segue padrão do projeto) | Larissa | `[09:51]` | "UUID, segue o padrão do resto do projeto. Tudo é uuid." |
| DEC-20 | CRUD de configuração de webhook acessível a qualquer role autenticada (por enquanto) | Sofia | `[09:37]` | "Por enquanto sim. Mais pra frente a gente pode endurecer." |

---

### RF — Requisitos funcionais explícitos

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| RF-01 | Endpoint POST para cadastro de webhook (campos: url, secret gerada pelo sistema, lista de status, customer_id) | Marcos | `[09:31]` | "O cliente precisa cadastrar webhook. Endpoint POST. Campos: url, secret é gerada pela gente e devolvida na criação. Lista de status que ele quer receber. Customer_id implícito do JWT." |
| RF-02 | Endpoint PATCH para edição de configuração de webhook | Bruno | `[09:33]` | "Continuando: PATCH pra editar, DELETE pra remover, GET pra listar os webhooks de um customer." |
| RF-03 | Endpoint DELETE para remoção de webhook | Bruno | `[09:33]` | "Continuando: PATCH pra editar, DELETE pra remover, GET pra listar os webhooks de um customer." |
| RF-04 | Endpoint GET para listar webhooks de um customer | Bruno | `[09:33]` | "Continuando: PATCH pra editar, DELETE pra remover, GET pra listar os webhooks de um customer." |
| RF-05 | Filtro de eventos por status por endpoint (lista de status que o webhook quer ouvir) | Marcos | `[09:33]` | "Filtro de eventos é uma lista dos status que o webhook quer ouvir. Tipo 'só quero saber quando vira SHIPPED e DELIVERED', e a gente filtra na hora de inserir na outbox." |
| RF-06 | Endpoint GET /webhooks/:id/deliveries — histórico dos últimos 100 envios (sucesso/falha, payload, response, tempo de resposta) | Marcos | `[09:34]` | "o cliente precisa conseguir ver o histórico de entregas. Tipo 'esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta'. GET /webhooks/:id/deliveries." |
| RF-07 | Endpoint para rotação de secret por cliente (antiga válida por 24h em paralelo) | Sofia | `[09:21]` | "E a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre." |
| RF-08 | Endpoint admin POST /admin/webhooks/dead-letter/:id/replay para reprocessamento manual de DLQ | Diego | `[09:35]` | "Sim. POST /admin/webhooks/dead-letter/:id/replay." |
| RF-09 | Assinatura HMAC do payload em cada envio de webhook | Sofia | `[09:20]` | "A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature. Cliente verifica do lado dele." |
| RF-10 | Endpoint de replay de DLQ deve logar o usuário que executou (auditoria) | Sofia | `[09:36]` | "o endpoint de admin tem que logar quem fez o replay, pra auditoria." |
| RF-11 | Inserção na webhook_outbox dentro da mesma transação SQL da mudança de status (atomicidade) | Bruno | `[09:40]` | "a alteração crítica é dentro do service de orders, no método changeStatus. Hoje a transação faz update na order, insere no history e atualiza estoque. A gente vai inserir na webhook_outbox dentro da mesma transação. Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair." |

---

### RNF — Requisitos não funcionais

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| RNF-01 | Latência end-to-end abaixo de 10 segundos (definição de "tempo real" dos clientes B2B) | Marcos | `[09:02]` | "Pra eles, qualquer coisa abaixo de 10 segundos já é 'tempo real'. O importante é que não fique pendurado e eles tenham que ficar atualizando manualmente." |
| RNF-02 | Janela de retry cobre até ~15 horas de indisponibilidade do cliente (5 tentativas, backoff até 12h) | Diego | `[09:17]` | "Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa." |
| RNF-03 | Garantia at-least-once de entrega | Larissa | `[09:26]` | "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão." |
| RNF-04 | Atomicidade entre mudança de status e registro do evento na outbox (sem inconsistência possível) | Diego | `[09:06]` | "Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível." |
| RNF-05 | Limite de 64KB por payload de evento; falha com erro se excedido | Diego | `[09:24]` | "Acho que 64KB já é um teto generoso. Nenhum evento nosso vai chegar perto disso." |
| RNF-06 | Timeout de 10 segundos por chamada HTTP do worker ao endpoint do cliente | Diego | `[09:42]` | "10 segundos. Cliente lento que não responde em 10s a gente trata como falha e marca pra retry." |
| RNF-07 | TLS obrigatório no endpoint de destino do webhook (https) | Sofia | `[09:23]` | "TLS obrigatório. URL do webhook tem que ser https." |
| RNF-08 | Ordering de eventos garantida por order_id somente enquanto single-worker (não global) | Larissa | `[09:13]` | "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker." |
| RNF-09 | Secret por endpoint (não global); isolamento de comprometimento | Sofia | `[09:21]` | "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo." |
| RNF-10 | Revisão de segurança obrigatória pela Sofia antes do deploy, mínimo 2 dias úteis | Sofia | `[09:46]` | "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma." |

---

### RES — Restrições

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| RES-01 | Stack restrita ao MySQL existente; proibido subir nova infra como Redis Cluster | Diego | `[09:07]` | "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve." |
| RES-02 | Worker deve usar a mesma DATABASE_URL e stack (Prisma) do projeto existente | Bruno | `[09:11]` | "Pode ser, mas vai precisar conectar no mesmo banco e usar o mesmo Prisma client." |
| RES-03 | Reuso obrigatório de padrões existentes: AppError, Pino, error middleware, módulos, Zod | Larissa | `[09:30]` | "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro." |
| RES-04 | Pressão comercial: a Atlas sinalizou migração para o concorrente se a entrega não sair "até fim do trimestre" — a referência de tempo desta fala **não** é a data de entrega (ver RES-08) | Marcos | `[09:00]` | "A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente." |
| RES-05 | Worker obrigatoriamente em processo separado da API (não mesma instância) | Diego | `[09:11]` | "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker." |
| RES-06 | Revisão de segurança pela Sofia é bloqueante para deploy | Sofia | `[09:49]` | "Ok, só não esqueçam de me agendar pra revisão de segurança antes de subir." |
| RES-07 | customer_id NÃO deve vir do JWT (JWT é de usuário operador, não de cliente); deve vir no body ou path | Larissa | `[09:32]` | "Então é endpoint autenticado normal, e o customer_id é passado no body ou no path. Não vem do JWT." |
| RES-08 | Data-alvo de entrega: fim de novembro, pedida pela Atlas — é a única data concreta da reunião, e é diferente do "fim do trimestre" de RES-04 | Marcos | `[09:45]` | "A Atlas quer pra fim de novembro. Larissa, dá em quantos sprints?" |

---

### ALT — Alternativas descartadas

| ID | Resumo (1 linha) + motivo do descarte | Falante | Timestamp | Citação literal |
|----|--------------------------------------|---------|-----------|-----------------|
| ALT-01 | Despacho síncrono de webhook dentro da transação de mudança de status — descartado porque cliente lento travaria mudança de status e indisponibilidade do cliente causaria rollback indevido | Bruno | `[09:04]` | "Síncrono não rola. A transação de mudança de status hoje já é pesada... Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos." |
| ALT-02 | Redis Streams (ou fila equivalente) — descartada como overengineering para time pequeno com MySQL já disponível | Diego | `[09:07]` | "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve." |
| ALT-03 | Database trigger para notificar o worker de novos eventos — descartada porque MySQL não tem NOTIFY/LISTEN nativo; workarounds (arquivo, endpoint) considerados inadequados ("fica esquisito") | Diego | `[09:09]` | "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL. Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito." |
| ALT-04 | 3 tentativas de retry — descartada por ser muito agressiva; não cobre janela de manutenção planejada de até 2 horas | Diego | `[09:16]` | "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada." |
| ALT-05 | Retry indefinido com backoff — descartada porque evento ficaria pendurado para sempre se cliente desaparecer | Diego | `[09:15]` | "Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu." |
| ALT-06 | Marcar eventos falhos na própria tabela outbox (em vez de tabela DLQ separada) — descartada para manter leitura da outbox principal limpa e facilitar debug/reprocessamento | Diego | `[09:18]` | "Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento." |
| ALT-07 | Garantia exactly-once de entrega — descartada porque exige coordenação de ambos os lados; at-least-once com event_id é o padrão de mercado (Stripe, GitHub) | Diego | `[09:25]` | "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos." |
| ALT-08 | Envio dos itens do pedido (order items) no payload do evento — descartado para manter payload enxuto; cliente pode buscar detalhes via GET /orders/:id | Diego | `[09:43]` | "Não manda items pra não inflar. Se o cliente quiser detalhes, ele bate no GET /orders/:id depois." |

---

### ADI — Itens adiados para fase futura

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| ADI-01 | Notificação por email ao cliente quando webhook falha repetidamente — adiado para próxima fase | Larissa | `[09:37]` | "Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." |
| ADI-02 | Rate limiting de envios de webhook para um único cliente — adiado para observar e decidir se virar problema | Diego | `[09:39]` | "Eu acho que não. A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto." |
| ADI-03 | Dashboard visual para clientes visualizarem seus webhooks — fora do escopo desta fase, projeto do time de frontend | Larissa | `[09:40]` | "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend." |
| ADI-04 | Arquivamento/limpeza de linhas entregues na outbox após 30 dias — fora do escopo desta feature | Diego | `[09:08]` | "Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature." |
| ADI-05 | Escalonamento para múltiplos workers com particionamento por order_id ou lock pessimista — problema do futuro | Diego | `[09:13]` | "Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora." |
| ADI-06 | Endurecimento das regras de autorização do CRUD de webhooks (além de "qualquer autenticado") — adiado | Sofia | `[09:37]` | "Por enquanto sim. Mais pra frente a gente pode endurecer." |

---

### ABE — Questões em aberto

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| ABE-01 | Rate limiting de envios outbound: Diego disse para registrar como "ponto em aberto" porém Larissa categorizou como "observar e decidir depois" — sem decisão de implementar nem de descartar definitivamente | Diego / Larissa | `[09:39]` | "Mas vale registrar como ponto em aberto." / "Tá. Fica como 'observar e decidir depois'." |
| ABE-02(?) | Nome exato do arquivo de lógica do worker dentro do módulo: webhook.worker.ts OU webhook.processor.ts — Bruno apresentou as duas opções com "ou", Diego disse "Beleza" sem escolher entre elas | Bruno | `[09:28]` | "a lógica de processamento fica num arquivo dentro do módulo, tipo src/modules/webhooks/webhook.worker.ts ou webhook.processor.ts." |
| ABE-03(?) | Localização do customer_id na requisição: body OU path — Larissa disse "body ou path" sem especificar qual; ninguém confirmou | Larissa | `[09:32]` | "Então é endpoint autenticado normal, e o customer_id é passado no body ou no path. Não vem do JWT." |
| ABE-04 | Schema de armazenamento do histórico de entregas para GET /webhooks/:id/deliveries — Marcos listou campos desejados (sucesso/falha, payload, response, tempo de resposta), mas nenhuma tabela ou estrutura foi projetada na reunião | Marcos | `[09:34]` | "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" |
| ABE-05 | Confirmação do prazo com os clientes (Atlas, MaxDistribuição, Nova Cargo) — Marcos prometeu confirmar "hoje à tarde" mas o resultado não voltou na reunião. A promessa aparece duas vezes: primeiro como intenção em `[09:47]` ("Atlas vai gostar. Eu confirmo prazo com eles"), depois com a citação literal abaixo | Marcos | `[09:49]` | "Tá bom. Eu atualizo os clientes hoje à tarde." |

---

### COD — Ganchos com o código existente

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| COD-01 | Tabela `orders` — atualizada dentro da transação de changeStatus | Bruno | `[09:04]` | "A transação de mudança de status hoje já é pesada — atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido." |
| COD-02 | Tabela `order_status_history` — insert dentro da transação de changeStatus | Bruno | `[09:04]` | "A transação de mudança de status hoje já é pesada — atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido." |
| COD-03 | Campo `stock_quantity` dos produtos do pedido — decrementado dentro da transação de changeStatus | Bruno | `[09:04]` | "A transação de mudança de status hoje já é pesada — atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido." |
| COD-04 | Classe `AppError` — padrão de erros de domínio a ser reutilizado pelo módulo de webhooks | Bruno | `[09:28]` | "a gente já tem um padrão. Tem classe AppError, classes específicas tipo InsufficientStockError, InvalidStatusTransitionError. Todas usam código tipo INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION." |
| COD-05 | Classe `InsufficientStockError` — exemplo de especialização de AppError no projeto | Bruno | `[09:28]` | "classes específicas tipo InsufficientStockError, InvalidStatusTransitionError" |
| COD-06 | Classe `InvalidStatusTransitionError` — exemplo de especialização de AppError no projeto | Bruno | `[09:28]` | "classes específicas tipo InsufficientStockError, InvalidStatusTransitionError" |
| COD-07 | Logger `Pino` — já integrado no projeto inteiro; será reutilizado sem alteração | Bruno | `[09:29]` | "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo." |
| COD-08 | Middleware centralizado de erro — já trata AppError, Zod e Prisma; não precisará de alteração | Bruno | `[09:29]` | "O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada." |
| COD-09 | `requireRole` — middleware existente de autorização por role a ser reutilizado no endpoint de replay | Larissa | `[09:36]` | "Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe." |
| COD-10 | `src/server.ts` — entry point existente da API; analogia para o novo `src/worker.ts` | Larissa | `[09:11]` | "Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script 'npm run worker'." |
| COD-11 | Padrão de módulos em `src/modules/` — cada domínio tem controller, service, repository, routes e schemas; webhooks seguirá igual | Bruno | `[09:27]` | "A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual." |
| COD-12 | `PrismaClient` — pool de conexão compartilhado por processo; worker instancia um novo client (mesmo banco) | Bruno | `[09:30]` | "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node." |
| COD-13 | `OrderService.changeStatus` — método onde a inserção na webhook_outbox será adicionada dentro da transação | Bruno | `[09:40]` | "a alteração crítica é dentro do service de orders, no método changeStatus." |
| COD-14 | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` — nova função a ser criada, recebe o tx client da transação; invocada pelo OrderService | Bruno | `[09:41]` | "Vou propor uma função publishWebhookEvent(tx, order, fromStatus, toStatus) que aceita o tx client da transação atual. Aí o order.service chama isso." |
| COD-15 | Esquema de códigos de erro com prefixo por domínio (ex.: INSUFFICIENT_STOCK) — padrão a replicar com prefixo WEBHOOK_ | Bruno | `[09:28]` | "Quero seguir igual pra webhook. Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc." |
| COD-16 | `DATABASE_URL` — variável de ambiente usada pelo Prisma; a mesma será usada pelo worker | Bruno | `[09:30]` | "Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node." |

---

### TEC — Detalhes técnicos secundários

| ID | Resumo (1 linha) | Falante | Timestamp | Citação literal |
|----|-----------------|---------|-----------|-----------------|
| TEC-01 | Tabela `webhook_outbox` com campo de status (pendente, processando, falhou, entregue) e `created_at`, ambos indexados | Diego | `[09:08]` | "A tabela tem índice no campo de status (pendente, processando, falhou, entregue) e em created_at." |
| TEC-02 | Worker lê eventos pendentes em batch pequeno, em ordem de created_at | Diego | `[09:08]` | "Worker lê só os pendentes em batch pequeno, processa, marca como entregue." |
| TEC-03 | Polling de 2 segundos como implementação do worker de despacho | Diego | `[09:09]` | "Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca." |
| TEC-04 | `event_id` UUID gerado no momento da inserção do evento na outbox | Diego | `[09:25]` | "a gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox. É único por evento." |
| TEC-05 | Header `X-Event-Id` — UUID do evento, enviado em cada request de webhook | Diego | `[09:25]` | "A gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox." |
| TEC-06 | Header `X-Signature` — HMAC-SHA256 do corpo do request | Sofia | `[09:20]` | "manda a assinatura num header tipo X-Signature. Cliente verifica do lado dele." |
| TEC-07 | Header `X-Timestamp` — timestamp do envio (permite cliente detectar replay attack) | Diego | `[09:44]` | "X-Timestamp com o timestamp do envio (pra cliente conseguir detectar replay attack se quiser)" |
| TEC-08 | Header `X-Webhook-Id` — id do endpoint webhook cadastrado, para clientes com múltiplos cadastros | Sofia | `[09:44]` | "Adiciona um X-Webhook-Id também, com o id do endpoint webhook, pra cliente que tem vários conseguir saber qual cadastro caiu naquele envio." |
| TEC-09 | Header `Content-Type: application/json` nos requests de webhook | Diego | `[09:44]` | "Content-Type application/json." |
| TEC-10 | Algoritmo HMAC-SHA256 especificamente (não outro algoritmo HMAC) | Sofia | `[09:20]` | "SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso." |
| TEC-11 | Grace period de 24 horas na rotação de secret (antiga permanece válida para migração do cliente) | Sofia | `[09:21]` | "a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre." |
| TEC-12 | Tabela `webhook_dead_letter` com campos: payload, motivo da falha, timestamp | Diego | `[09:18]` | "Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp." |
| TEC-13 | Tabela de configuração de webhook com campos: url, secret, customer_id, estado ativo | Bruno | `[09:21]` | "Então a tabela de configuração de webhook armazena url + secret + customer_id + estado ativo?" |
| TEC-14 | Secret gerada pelo sistema (não fornecida pelo cliente) e devolvida na resposta de criação | Marcos | `[09:31]` | "secret é gerada pela gente e devolvida na criação." |
| TEC-15 | Filtro de eventos (lista de status por webhook) armazenado por endpoint, aplicado na inserção na outbox | Marcos | `[09:33]` | "Filtro de eventos é uma lista dos status que o webhook quer ouvir... e a gente filtra na hora de inserir na outbox." |
| TEC-16 | Códigos de erro com prefixo `WEBHOOK_`: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED (exemplos) | Bruno | `[09:28]` | "Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc." |
| TEC-17 | Campos do payload do evento: event_id, event_type ("order.status_changed"), timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents | Diego | `[09:43]` | "JSON com event_id, event_type tipo 'order.status_changed', timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, e os campos básicos da order tipo total_cents." |
| TEC-18 | event_type fixo: `"order.status_changed"` | Diego | `[09:43]` | "event_type tipo 'order.status_changed'" |
| TEC-19 | Timestamp do payload em formato ISO 8601 | Diego | `[09:43]` | "timestamp ISO 8601" |
| TEC-20 | Script `npm run worker` para iniciar o worker como processo separado | Larissa | `[09:11]` | "Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script 'npm run worker'." |
| TEC-21 | Caminho do endpoint de histórico de entregas: `GET /webhooks/:id/deliveries` | Marcos | `[09:34]` | "GET /webhooks/:id/deliveries." |
| TEC-22 | Caminho do endpoint admin de replay: `POST /admin/webhooks/dead-letter/:id/replay` | Diego | `[09:18]` | "Tipo um POST /admin/webhooks/dead-letter/:id/replay." |
| TEC-23 | Progressão do backoff: 1min → 5min → 30min → 2h → 12h (total ~15 horas) | Diego | `[09:17]` | "Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa." |
| TEC-24 | 5 tentativas totais de retry (inclui o envio inicial mais 4 retentativas, ou 5 retentativas após a primeira?) | Diego | `[09:15]` | "Eu sugiro 5. Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu. Cinco já dá pra cobrir uma janela de até 12 ou 24 horas." |
| TEC-25 | Timeout de HTTP de 10 segundos por chamada do worker | Diego | `[09:42]` | "10 segundos. Cliente lento que não responde em 10s a gente trata como falha e marca pra retry." |
| TEC-26 | Validação de URL de webhook via schema Zod (rejeitar http, aceitar apenas https) | Sofia | `[09:23]` | "Se o cliente cadastrar http, recusamos com erro de validação. Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod." |
| TEC-27 | Ordering implícita por order_id e por created_at da outbox em single-worker | Diego | `[09:12]` | "Se a gente tem um único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em ordem." |
| TEC-28 | Replay de DLQ re-insere o evento na outbox com status pendente | Diego | `[09:18]` | "Recoloca na outbox como pendente." |
| TEC-29 | Clientes que solicitaram a feature: Atlas Comercial, MaxDistribuição, Nova Cargo | Marcos | `[09:00]` | "a gente recebeu na semana passada um pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo." |

---

## Notas de leitura

### (a) Contradições e reversões

**customer_id via JWT vs. body/path** (`[09:31]` → `[09:32]`): Marcos afirmou inicialmente que o customer_id viria "implícito do JWT" (`[09:31]` Marcos: "Customer_id implícito do JWT."). Bruno imediatamente contestou que o JWT atual é do operador, não do cliente (`[09:32]` Bruno: "Espera, mas o JWT atual é do usuário operador, não do cliente."). Marcos tentou reconciliar dizendo que os usuários representam o cliente (`[09:32]` Marcos: "A gente tem usuários que representam o cliente."), mas Larissa encerrou a discussão com uma nova posição: customer_id no body ou path (`[09:32]` Larissa: "o customer_id é passado no body ou no path. Não vem do JWT."). A proposta inicial de Marcos foi revertida, mas a forma exata (body vs. path) ficou em aberto — ver ABE-03.

**Número de retries (3 vs. 5)** (`[09:16]`): Bruno sugeriu 3 retries (`[09:16]` Bruno: "3 não é melhor? Mais agressivo."), Diego defendeu 5 com argumento de cobertura para manutenção planejada. Larissa fechou em 5 (`[09:17]`). Não é reversão do mesmo falante, mas é uma posição apresentada e descartada que vale registrar — ver ALT-04.

### (b) Promessas de confirmação que não retornaram na reunião

1. **Confirmação do prazo com os clientes** (`[09:49]` Marcos: "Tá bom. Eu atualizo os clientes hoje à tarde."; a mesma promessa já aparecia em `[09:47]`: "Atlas vai gostar. Eu confirmo prazo com eles."): Marcos prometeu confirmar o prazo de três sprints com Atlas Comercial, MaxDistribuição e Nova Cargo, mas o resultado desta confirmação não está na reunião.

2. **Agendamento da sessão de revisão de design** (`[09:50]` Larissa: "Eu vou abrir o doc de design da feature e marcar uma sessão pro Bruno e o Diego revisarem comigo antes da gente começar a codar."): Larissa prometeu abrir o documento e agendar sessão de revisão, sem data ou prazo informados.

3. **Agendamento da revisão de segurança com Sofia** (`[09:49]` Sofia: "Ok, só não esqueçam de me agendar pra revisão de segurança antes de subir."): Sofia fez o pedido, ninguém confirmou data ou responsável pelo agendamento.

### (c) Itens que soam como decisão sem concordância explícita

1. **ABE-03(?) — customer_id no body vs. path**: Larissa declarou "body ou path" como se fosse uma decisão, mas ela própria deixou duas opções em aberto e nenhum outro participante confirmou a escolha. Este item entra nos documentos como decisão parcial (não vem do JWT), com sub-decisão em aberto (body ou path).

2. **ABE-02(?) — Nome do arquivo do worker**: Bruno propôs `webhook.worker.ts ou webhook.processor.ts` com "ou" explícito; Diego disse "Beleza" para a estrutura geral. O nome exato nunca foi escolhido.

3. **TEC-24 — Semântica exata do "5 tentativas"**: Diego diz "Eu sugiro 5" (`[09:15]`) e Larissa fecha "Decidido: 5 tentativas" (`[09:17]`). Não foi especificado se as 5 contam o envio inicial ou são 5 retentativas após a primeira falha. A progressão de backoff tem 5 intervalos (1m/5m/30m/2h/12h), sugerindo 5 retentativas (não incluindo o envio inicial), mas isso não foi dito explicitamente.

4. **DEC-10 / at-least-once e responsabilidade do cliente**: Diego apresentou a posição (`[09:25]`), Sofia registrou preocupação ("Isso joga responsabilidade pro cliente"), Diego respondeu com argumento de mercado (`[09:25]`), e Marcos disse que documentaria no portal (`[09:26]`). Larissa fechou como "Decisão" (`[09:26]`). A concordância da Sofia com a decisão final não foi expressa — ela registrou preocupação mas não disse "ok" explicitamente.

### (d) Correções aplicadas a este índice

Este arquivo é artefato derivado, e a varredura de rastreabilidade registrada em [`docs/TRACKER.md` §5](../../docs/TRACKER.md#5-achados-da-varredura) encontrou três defeitos de carimbo nele. **Os três foram corrigidos aqui, e o registro do que mudou fica abaixo** — a transcrição em si não foi tocada.

| Achado | O que estava | O que ficou |
|---|---|---|
| A-01 | `COD-04`, `COD-05`, `COD-06`, `COD-15` e `TEC-16` citavam `[09:29]` | `[09:28]`, que é onde a fala de Bruno sobre `AppError` e os códigos `WEBHOOK_*` realmente está. `COD-07` e `COD-08` seguem em `[09:29]` |
| A-02 | `ABE-05` e a nota (b)-1 citavam `[09:47]` com a frase "Tá bom. Eu atualizo os clientes hoje à tarde" | `[09:49]`, que é onde a frase está; as duas linhas passaram a nomear também a fala de `[09:47]`, que sustenta o mesmo fato |
| A-03 | `RES-04` resumia "entrega até fim de novembro" citando a fala de `[09:00]`, que diz "fim do trimestre" | duas linhas: `RES-04` (a pressão comercial de `[09:00]`) e `RES-08` (a data-alvo de `[09:45]`). **O índice passou de 112 para 113 itens** |
