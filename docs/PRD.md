# PRD — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Documento** | Product Requirements Document — por que e o quê |
| **Autor** | Marcos — Product Manager (dono dos requisitos de produto da reunião) |
| **Status** | **Em revisão** — depende das mesmas duas revisões pendentes da [RFC](RFC.md), e de três confirmações de produto que nunca voltaram (seção 9) |
| **Data** | 2026-08-17 |
| **Base** | [RFC](RFC.md) · [FDD](FDD.md) · [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) a [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md) |
| **Público** | produto, liderança técnica, segurança e quem for conversar com Atlas Comercial, MaxDistribuição e Nova Cargo |

> **Este documento opera em altura de produto.** Ele responde *por que* e *o quê*. Não repete a arquitetura da [RFC](RFC.md), não redecide o que virou ADR e não importa nada da especificação de implementação do [FDD](FDD.md). Onde o *como* importa, há um link para o documento que o carrega.
>
> **O ledger de 21 propostas sem origem do FDD não gera requisito aqui.** São escolhas de implementação em revisão; requisito de produto sai de `RF`/`RNF` do índice da transcrição.

### Convenção de rastreabilidade — aplicada em duas seções

O FDD marca **F** / **D** / **P** em todo o texto. Aqui a marca aparece **apenas nas seções 4 (objetivos e métricas) e 5 (escopo)** — os dois lugares onde este documento tem risco real de inventar. No resto, a origem vem em prosa: ID do índice, falante e timestamp, ou link para o documento que decidiu.

| Marca | Significado |
|---|---|
| **F** | **Fonte direta** — item do índice da transcrição (`DEC`/`RF`/`RNF`/`RES`/`TEC`/`ALT`/`ADI`/`ABE`) ou arquivo real do repositório |
| **D** | **Derivado** — consequência necessária de uma fonte (aritmética, ou implicação de uma decisão) |
| **P** | **Proposta deste PRD, sem origem** — ninguém decidiu; está aberta a revisão |

---

## 1. Resumo e contexto

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente, na semana anterior à reunião, que a plataforma os avise quando o status de um pedido muda (`TEC-29`, Marcos `[09:00]`). Hoje eles descobrem sozinhos, perguntando.

A feature entrega **notificação ativa de mudança de status de pedido**: o cliente cadastra um endpoint HTTPS pela nossa API, escolhe quais status quer ouvir, e passa a receber um `POST` assinado a cada transição que assinou. Se o endpoint dele estiver fora do ar, a plataforma retenta ao longo de aproximadamente 15 horas; o que não entregar fica guardado e pode ser reprocessado manualmente por um administrador.

O sistema **não tem nada disso hoje**. A varredura registrada em `CONTEXT.md` §16 procurou `webhook`, `event`, `queue`, `outbox`, `worker`, `retry`, `hmac`, `redis` e correlatos em `src/`, `prisma/` e `package.json`: zero ocorrências. A feature parte do zero.

O prazo é externo e tem risco comercial associado: Marcos registrou que a Atlas sugeriu migrar para um concorrente se a entrega não sair (`RES-04`, `[09:00]`). O time fechou três sprints, com a revisão de segurança de Sofia incluída no fim (`DEC-13`, Larissa `[09:47]`).

**O que este documento não decide.** Duas promessas ao cliente estão em disputa aberta e o PRD as registra sem fechá-las: o alvo de latência de 10 segundos (seção 7, `RNF-01`) e a garantia at-least-once, cuja objeção de Sofia nunca foi retirada (seção 8, `DEC-10`).

---

## 2. Problema e motivação

### 2.1 O estado atual: o cliente pergunta em loop

O baseline do problema está em uma única fala da reunião, e **nenhum outro documento do pacote a usou** — RFC, FDD e ADRs operam todos acima dela:

> `[09:00]` **Marcos**: os clientes "ficam batendo no `GET /orders` de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles".

O endpoint citado é real: `router.get('/', validate({ query: listOrdersQuerySchema }), controller.list)` em `src/modules/orders/order.routes.ts`, montado em `/api/v1/orders`. É a única forma que um cliente tem hoje de saber que um pedido mudou.

Três consequências, todas na fala:

1. **Lento** — o cliente descobre a mudança no próximo ciclo de consulta dele, não quando ela acontece.
2. **Caro** — para os dois lados: ele paga chamadas que na maioria das vezes não trazem novidade, e nós servimos todas elas.
3. **Frágil** — quem depende de estado atualizado precisa consultar com frequência alta, o que piora o item 2.

É desse cenário que sai a definição de "tempo real" do produto: *"Pra eles, qualquer coisa abaixo de 10 segundos já é 'tempo real'. O importante é que não fique pendurado e eles tenham que ficar atualizando manualmente"* (`RNF-01`, Marcos `[09:02]`). **O material do requisito é a segunda frase**: parar de atualizar manualmente. O número está na primeira, e é ele que entra em conflito com o desenho (seção 7).

### 2.2 A pressão comercial

O pedido é formal e tem nome: Atlas Comercial, MaxDistribuição e Nova Cargo (`TEC-29`, `[09:00]`). A Atlas anexou uma consequência: *"A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente"* (`RES-04`, Marcos `[09:00]`).

**Duas datas diferentes circulam, e é preciso separar.** A fala de `[09:00]` diz "fim do trimestre". A data concreta — **fim de novembro** — é literal de Marcos em `[09:45]`, e é a que este documento adota como referência de prazo. A linha `RES-04` do índice funde as duas: resume "fim de novembro" citando `[09:00]`.

**E o prazo não é compromisso confirmado.** Marcos assumiu atualizar os clientes: *"Eu atualizo os clientes hoje à tarde"* (`ABE-05`, `[09:47]`), e o resultado nunca voltou. Ninguém dos três clientes confirmou que fim de novembro serve. Ver risco R-02 e dependência D-04.

### 2.3 Por que agora, e por que assim

O time é pequeno (`RES-01`, Diego `[09:07]`) e a stack está fechada no MySQL existente — nenhuma infraestrutura nova, nenhuma dependência nova (`RES-01` / `DEC-11` / `RES-03`). Isso não é preferência de produto, é restrição herdada, e ela molda o que dá para prometer ao cliente. As consequências de produto dessa restrição estão na seção 8.

---

## 3. Público-alvo e cenários de uso

### 3.1 São dois públicos, não um

A reunião contém uma distinção que nenhum documento do pacote explorou, e ela muda como a feature deve ser pensada:

> `[09:31]` **Marcos**: "Customer_id implícito do JWT."
> `[09:32]` **Bruno**: "Espera, mas o JWT atual é do usuário operador, não do cliente."
> `[09:32]` **Larissa**: "o customer_id é passado no body ou no path. Não vem do JWT." (`RES-07`)

O tipo `AuthUser` em `src/middlewares/auth.middleware.ts` confirma: o token carrega `{ id, email, role }` com `role: 'ADMIN' | 'OPERATOR'` — é credencial de **operador da plataforma**, não de cliente. Não existe login de cliente no sistema.

Logo, **quem configura o webhook e quem recebe o `POST` são partes diferentes**:

| | **P1 — Operador da plataforma** | **P2 — Sistema do cliente B2B** | **P3 — Administrador** |
|---|---|---|---|
| **Quem é** | usuário interno com role `OPERATOR` ou `ADMIN`, agindo em nome de um cliente | o software da Atlas, da MaxDistribuição ou da Nova Cargo | usuário interno com role `ADMIN` |
| **Como se autentica** | JWT existente (`authenticate`) | **não se autentica na nossa API** — apenas recebe e verifica | JWT + `requireRole('ADMIN')` (`DEC-12`) |
| **O que faz** | cadastra, edita, lista e remove endpoints; rotaciona a secret; consulta o histórico de entregas | recebe o `POST`, verifica o HMAC, deduplica por `X-Event-Id`, responde `2xx` | reprocessa o que caiu na dead letter (`DEC-07`) |
| **Requisitos** | RF-01 a RF-07 | RF-09 (assinatura), e todo o `RNF` de entrega | RF-08, RF-10 |

**Por que isso importa para produto, e não só para a API.** Três consequências que só ficam visíveis quando os dois públicos são separados:

1. **O cliente final não tem autoatendimento.** Não há login de cliente, e o dashboard visual ficou fora de escopo (`ADI-03`). Toda operação de configuração passa por um operador nosso. Nenhuma fonte trata de como o cliente pede uma alteração.
2. **A autorização é frouxa por consequência direta disso.** Como o `customerId` vem do body ou da query e não do token, *qualquer* operador autenticado opera webhook de *qualquer* cliente. Sofia aceitou como estado temporário — *"Por enquanto sim. Mais pra frente a gente pode endurecer"* (`DEC-20`, `[09:37]`) — e o endurecimento virou `ADI-06`, sem gatilho. Ver risco R-05.
3. **Metade do contrato roda em código que não controlamos.** A verificação da assinatura e a deduplicação acontecem no sistema do cliente. Se P2 não implementar, o mecanismo não protege nada e a garantia de entrega não vale nada — e não temos como saber.

### 3.2 Cenários de uso com origem na reunião

| # | Cenário | Público | Origem |
|---|---|---|---|
| CU-01 | Um dos três clientes que pediram a feature passa a receber notificação em vez de consultar `GET /orders` em loop | P1 configura, P2 recebe | `TEC-29` `[09:00]` + baseline de `[09:00]` |
| CU-02 | O cliente quer só parte do ciclo: *"só quero saber quando vira SHIPPED e DELIVERED"* — e não recebe os demais status | P1 configura o filtro | `RF-05`, Marcos `[09:33]` |
| CU-03 | O endpoint do cliente entra em **manutenção planejada de duas horas** e, ao voltar, recebe os eventos retidos — caso concreto que Diego trouxe de um cliente real | P2 | `ALT-04`, Diego `[09:16]` |
| CU-04 | O cliente sumiu por mais de ~15 horas: os eventos esgotam as tentativas, caem na dead letter, e um administrador reprocessa manualmente depois | P3 | `DEC-07` / `TEC-28` `[09:18]` |
| CU-05 | O cliente suspeita que a secret vazou (ou faz rotação de rotina) e pede uma nova; a antiga continua válida por 24h para ele migrar sem downtime | P1 dispara, P2 migra | `RF-07` / `TEC-11`, Sofia `[09:21]` |
| CU-06 | O cliente tem **vários endpoints cadastrados** e precisa saber qual cadastro recebeu cada envio — daí o `X-Webhook-Id` | P2 | `TEC-08`, Sofia `[09:44]` |
| CU-07 | O cliente reclama que não recebeu um evento e quer evidência: consulta o histórico dos últimos 100 envios com sucesso/falha, payload, response e tempo de resposta | P1 consulta | `RF-06`, Marcos `[09:34]` |
| CU-08 | O cliente precisa dos itens do pedido, que **não vão no payload**, e busca em `GET /orders/:id` depois | P2 | `ALT-08`, Diego `[09:43]` |
| CU-09 | Auditoria: é preciso saber quem executou um reprocessamento e quando | P3 | `RF-10`, Sofia `[09:36]` |
| CU-10 | A Atlas avalia migrar para o concorrente se a entrega não sair no prazo | contexto comercial | `RES-04` `[09:00]` + prazo `[09:45]` |

### 3.3 Cenários adicionais — **propostos, sem origem na reunião**

Nenhum dos quatro abaixo foi levantado por ninguém. Estão aqui porque são jornadas previsíveis dos públicos acima, e a ausência deles na fonte é informação.

| # | Cenário proposto | Por que provavelmente aparece | Estado |
|---|---|---|---|
| CU-P1 | O cliente troca a URL do endpoint (migração de infraestrutura dele) sem perder eventos em trânsito | `RF-02` existe (`PATCH`); ninguém discutiu eventos já enfileirados durante a troca | **P** |
| CU-P2 | Um quarto cliente, que não pediu a feature, quer aderir | `TEC-29` nomeia três; nada trata de expansão | **P** |
| CU-P3 | O cliente quer testar a integração antes de ir para produção (envio de evento de teste) | nenhuma fonte menciona; não há endpoint de teste em escopo | **P** |
| CU-P4 | O cliente descobre sozinho que parou de receber | não há alerta: `ADI-01` (email) está fora de escopo, e a única forma é consultar `RF-06` — ver risco R-08 | **P** |

---

## 4. Objetivos e métricas de sucesso

> **Aviso que sustenta esta seção inteira: a reunião não produziu nenhuma métrica de produto.** Não há meta de adoção, KPI, baseline numérico nem definição de sucesso. Existem apenas **números técnicos**: 10 segundos (`RNF-01`), ~15 horas (`RNF-02`), 64KB (`RNF-05`), três sprints (`DEC-13`), três clientes (`TEC-29`), dois dias úteis de revisão (`RNF-10`). A seção 4.1 é construída sobre esses números. **A seção 4.2 é proposta deste PRD, inteira.**

### 4.1 Objetivos com origem rastreável

| # | Objetivo | Métrica e meta | Origem |
|---|---|---|---|
| OB-01 | O cliente deixa de consultar para descobrir: a plataforma avisa | **Zero necessidade de polling** para saber de mudança de status assinada. Verificação binária por cliente integrado | `RNF-01` `[09:02]` + baseline `[09:00]` **F** |
| OB-02 | A notificação chega em tempo que o cliente considera real | **< 10 segundos** ponta a ponta, da mudança de status ao `POST` recebido | `RNF-01`, Marcos `[09:02]` **F** — **meta em conflito com o desenho; ver seção 7 e [questão 4 da RFC](RFC.md#exigem-decisão-mas-não-travam-o-início)** |
| OB-03 | Nenhum evento se perde por indisponibilidade planejada do cliente | Janela de retentativa **≥ 2 horas** (o caso real que motivou a decisão); o desenho decidido cobre **14h36** | meta: `ALT-04` `[09:16]` **F** · cobertura: `DEC-05`/`RNF-02` **F**, soma dos intervalos **D** |
| OB-04 | Nunca existe pedido com status alterado e evento não registrado | **Zero** ocorrências. *"Não pode ter caso de status mudar e evento não sair"* | `RF-11`, Bruno `[09:40]` / `RNF-04` **F** |
| OB-05 | O evento que não entregou não é descartado em silêncio | **100%** dos eventos com tentativas esgotadas têm registro recuperável e reprocessável | `DEC-06` / `DEC-07` `[09:18]` **F** |
| OB-06 | Os três clientes que pediram passam a usar | **3 de 3** — Atlas Comercial, MaxDistribuição e Nova Cargo com ao menos um endpoint ativo | os três nomes: `TEC-29` **F**; tratá-los como **alvo de adoção**: **P** |
| OB-07 | A entrega cabe no prazo com a revisão de segurança dentro dele | **3 sprints**, com **≥ 2 dias úteis** de revisão de Sofia no fim, mirando **fim de novembro** | `DEC-13` `[09:47]` · `RNF-10` `[09:46]` · prazo `[09:45]` **F** — **não confirmado com os clientes** (`ABE-05`) |
| OB-08 | A feature não aumenta a superfície operacional da plataforma | **Zero** dependência nova em `package.json`; nenhum serviço novo além do próprio worker | `RES-01` / `DEC-11` / `RES-03` **F** |

**OB-02 é o único objetivo desta tabela cuja meta o desenho aprovado não cumpre.** A ressalva completa está na seção 7 e não é repetida aqui, mas nenhuma leitura desta tabela está correta sem ela.

### 4.2 Métricas de produto — **propostas deste PRD, sem origem** (**P**)

Nada abaixo foi decidido por ninguém. Cada linha traz o instrumento de medição, porque uma meta sem instrumento é só um número.

| # | Métrica proposta | Meta proposta | Como medir | Ressalva |
|---|---|---|---|---|
| MP-01 | **Adoção** — clientes com endpoint ativo | 3 de 3 em **30 dias** após go-live | contagem de assinaturas ativas por cliente | o prazo de 30 dias é arbitrário |
| MP-02 | **Redução de polling** — queda no volume de `GET /orders` dos clientes integrados | **≥ 50%** em 30 dias | comparar chamadas por cliente nos 30 dias antes e depois; a plataforma já loga `http_request` com `path` e `durationMs` (`src/middlewares/request-logger.middleware.ts`) | **o baseline não existe.** A fala de `[09:00]` é qualitativa ("lenta e cara"). Sem medir o volume atual antes do go-live, os 50% não têm referência — **medir o baseline é pré-requisito, não consequência** |
| MP-03 | **Taxa de entrega na primeira tentativa** | **≥ 95%** | proporção de eventos entregues sem retentativa | número sem origem; mede a saúde dos endpoints dos clientes tanto quanto a nossa |
| MP-04 | **Volume de dead letter por semana** | tendência **decrescente**, sem alvo absoluto | contagem de registros por janela | é o sinal de que a janela de ~15h não bastou para algum cliente |
| MP-05 | **Latência percebida** — idade do evento pendente mais antigo | **p95 < 10s**, alinhado a OB-02 | mede o que Marcos descreveu ("que não fique pendurado") melhor que a média | herda integralmente o conflito de OB-02 |
| MP-06 | **Churn evitado** — Atlas Comercial retida | binário, avaliado no trimestre seguinte ao go-live | acompanhamento comercial | `RES-04` dá o risco; transformá-lo em métrica é proposta |

As grandezas técnicas que alimentam MP-03, MP-04 e MP-05 estão detalhadas no [FDD §10.2](FDD.md) — que também as marca como proposta, e registra que **não há alarme nem instrumentação decidida**. Métrica sem alguém olhando não é métrica.

---

## 5. Escopo

### 5.1 Incluído nesta fase

| Item | Público | Origem |
|---|---|---|
| Cadastro, edição, remoção e listagem de endpoints de webhook | P1 | `RF-01` a `RF-04` **F** |
| Filtro por status: o cliente escolhe quais transições quer ouvir | P1 / P2 | `RF-05` **F** |
| Entrega do evento por `POST` HTTPS assinado, a cada mudança de status assinada | P2 | `RF-09` / `RNF-07` **F** |
| Retentativa automática com espaçamento crescente e destino final para o que não entregar | — | `DEC-05` / `DEC-06` **F** |
| Reprocessamento manual do que caiu no destino final, restrito a administrador e auditado | P3 | `RF-08` / `RF-10` / `DEC-12` **F** |
| Rotação de secret com 24h de convivência entre a antiga e a nova | P1 / P2 | `RF-07` / `TEC-11` **F** |
| Consulta ao histórico dos últimos 100 envios | P1 | `RF-06` **F** |
| Atomicidade entre mudança de status e registro do evento | — | `RF-11` / `RNF-04` **F** |

**O gatilho é um só: a mudança de status de um pedido** (`OrderService.changeStatus`, `COD-13` **F**). Nenhum outro evento de negócio gera webhook nesta fase.

### 5.2 Fora de escopo — **descartado ou adiado explicitamente na reunião**

Cada item abaixo tem dono, timestamp e uma decisão dita em voz alta. Não são lacunas.

| Item | O que fica de fora | Quem e quando | Marca |
|---|---|---|---|
| `ADI-01` | **Aviso ao cliente quando o webhook falha repetidamente** (email). *"Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto"* | Larissa `[09:37]` | **F** |
| `ADI-03` | **Dashboard visual para o cliente.** *"Só endpoints. Painel é projeto separado do time de frontend"* | Larissa `[09:40]` | **F** |
| `ADI-02` / `ABE-01` | **Limite de vazão de envios** para um cliente. Classificação divergente: Diego pediu "ponto em aberto", Larissa reclassificou como "observar e decidir depois" — não há decisão de fazer nem de descartar | Diego / Larissa `[09:39]` | **F** |
| `ADI-04` | **Limpeza/arquivamento dos registros antigos** após ~30 dias. Sem isso, as tabelas crescem indefinidamente | Diego `[09:08]` | **F** |
| `ADI-05` | **Escalar para mais de um worker.** *"Isso é problema do futuro, não agora"* | Diego `[09:13]` | **F** |
| `ADI-06` | **Endurecer a autorização do CRUD** além de "qualquer usuário autenticado" | Sofia `[09:37]` | **F** |
| `ALT-08` | **Itens do pedido no payload do evento.** Descartado para não inflar o envio: *"Se o cliente quiser detalhes, ele bate no GET /orders/:id depois"* | Diego `[09:43]` | **F** |
| `ALT-07` | **Garantia de entrega sem duplicata (exactly-once).** Descartada por custo de coordenação bilateral — ver seção 8, T-02 | Diego `[09:25]` | **F** |

### 5.3 Fora de escopo — **não decidido por ninguém**

> **Esta categoria não existe na transcrição. Ela foi criada pelo [FDD §3](FDD.md) e é mantida aqui de propósito.** A diferença entre ela e a seção 5.2 é a diferença entre *"decidimos deixar pra depois"* e *"ninguém perguntou"*. Achatar as duas numa lista só apagaria exatamente a informação que importa: nos itens abaixo, **a pergunta nunca foi feita** — e as duas são perguntas de produto, com efeito visível para o cliente.

| Item | A pergunta que ninguém fez | Consequência de não decidir | Marca |
|---|---|---|---|
| **A criação do pedido gera evento?** | `CONTEXT.md` §17.2 identifica `OrderService.create` (linha 50) como um segundo lugar onde o pedido nasce em `PENDING` e grava o primeiro registro de histórico, com o mesmo padrão de acoplamento de `changeStatus`. A reunião tratou só de `changeStatus` (`COD-13`) | **Um cliente que assine `PENDING` não recebe nada quando o pedido nasce** — só quando ele sai de `PENDING`. É comportamento visível ao cliente, decidido por omissão. Ver [RFC, questão 7](RFC.md#exigem-decisão-mas-não-travam-o-início) e [FDD §3](FDD.md) | **F** que a pergunta não existe; a consequência é **D** |
| **O que acontece com o histórico quando uma assinatura é removida?** | `RF-03` cria a remoção do endpoint. Nenhuma fonte diz o que acontece com os eventos ainda em trânsito daquela assinatura, nem com o histórico de entregas que `RF-06` prometeu ao cliente | Apagar junto destrói a evidência prometida por `RF-06`; não apagar impede a remoção. **É decisão de produto, não de implementação** — o FDD registra explicitamente que não a toma ([FDD §6.6 e §13](FDD.md)) | **F** que a pergunta não existe |

---

## 6. Requisitos funcionais

Onze requisitos, todos com fala e timestamp. A coluna **Onde está o "como"** aponta para o documento que especifica o mecanismo — o PRD não o repete.

| ID | Requisito (o quê) | Público | Origem | Onde está o "como" |
|---|---|---|---|---|
| **RF-01** | O cliente precisa ter um endpoint de webhook cadastrado, com URL, lista de status desejados e uma **secret gerada pela plataforma** e devolvida no cadastro | P1 | Marcos `[09:31]` (`TEC-14`) | [FDD §7.1](FDD.md) |
| **RF-02** | A configuração de um endpoint pode ser editada | P1 | Bruno `[09:33]` | [FDD §7.3](FDD.md) |
| **RF-03** | Um endpoint pode ser removido | P1 | Bruno `[09:33]` | [FDD §7.4](FDD.md) — **com pergunta aberta de produto, ver 5.3** |
| **RF-04** | É possível listar os endpoints de um cliente | P1 | Bruno `[09:33]` | [FDD §7.2](FDD.md) |
| **RF-05** | Cada endpoint declara **quais status quer ouvir**, e recebe só esses. *"Tipo 'só quero saber quando vira SHIPPED e DELIVERED'"* | P1 / P2 | Marcos `[09:33]` (`TEC-15`) | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) — o filtro é aplicado no registro do evento (`DEC-16`) |
| **RF-06** | O cliente consegue ver o **histórico dos últimos 100 envios** feitos para ele: sucesso/falha, payload, response e tempo de resposta | P1 | Marcos `[09:34]` | [FDD §7.5](FDD.md) — **a estrutura que sustenta isso não foi projetada na reunião (`ABE-04`); o FDD a propõe e assume a autoria** |
| **RF-07** | A secret é **rotacionável pela API**, e a antiga continua válida por **24 horas** em paralelo, para o cliente migrar os sistemas dele sem downtime | P1 / P2 | Sofia `[09:21]` (`TEC-11`) | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) · [FDD §7.6 e §8.2](FDD.md) — **o caminho do endpoint e o formato durante a janela são proposta, não decisão** |
| **RF-08** | Um administrador consegue **reprocessar manualmente** um evento que esgotou as tentativas | P3 | Diego `[09:35]` (`TEC-22`) · role `ADMIN` por `DEC-12` | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) · [FDD §7.7](FDD.md) |
| **RF-09** | Todo envio é **assinado**, para o cliente provar que veio de nós. *"A gente assina o payload com uma secret compartilhada… Cliente verifica do lado dele"* | P2 | Sofia `[09:20]` | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| **RF-10** | O reprocessamento **registra quem o executou**, para auditoria | P3 | Sofia `[09:36]` | [FDD §10.1](FDD.md) |
| **RF-11** | O registro do evento acontece **na mesma transação** da mudança de status. *"Não pode ter caso de status mudar e evento não sair"* | — | Bruno `[09:40]` | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |

**Nota sobre RF-01 e o `customer_id`.** A primeira formulação de Marcos — *"Customer_id implícito do JWT"* — foi revertida na própria reunião (seção 3.1). Larissa fechou que **não vem do JWT**, deixando "body ou path" em aberto (`ABE-03`); a forma concreta foi escolhida fora da reunião, pela convenção do código, e segue **sem aval** de Larissa, Bruno ou Marcos ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).

---

## 7. Requisitos não funcionais

| ID | Requisito | Valor | Origem |
|---|---|---|---|
| **RNF-01** | **Latência ponta a ponta** — definição de "tempo real" dos clientes | **< 10 segundos** | Marcos `[09:02]` — **ver ressalva abaixo** |
| **RNF-02** | **Tolerância a indisponibilidade do cliente** | janela de retentativa de **~15 horas** (14h36 entre a primeira falha e a última tentativa) | Diego `[09:17]` |
| **RNF-03** | **Garantia de entrega** | **at-least-once** — o mesmo evento pode chegar mais de uma vez; a deduplicação é do cliente | Larissa `[09:26]` — ver T-02 |
| **RNF-04** | **Integridade** | não existe estado em que o status mudou e o evento não foi registrado, nem o inverso | Diego `[09:06]` |
| **RNF-05** | **Tamanho do evento** | teto de **64KB**; acima disso **falha com erro, não trunca** | Diego / Larissa `[09:24]` — ver T-05 |
| **RNF-06** | **Paciência com cliente lento** | **10 segundos** de espera por resposta; depois disso é falha e vira retentativa | Diego `[09:42]` |
| **RNF-07** | **Transporte** | **HTTPS obrigatório**; endpoint `http` é recusado no cadastro | Sofia `[09:23]` |
| **RNF-08** | **Ordem dos eventos** | garantida **por pedido**, não global, e **apenas enquanto houver um único worker** — limitação declarada, não garantia plena | Larissa `[09:13]` — ver T-06 |
| **RNF-09** | **Raio de comprometimento** | **uma secret por endpoint**, nunca uma secret global. *"Senão se vaza uma, vaza tudo"* | Sofia `[09:21]` |
| **RNF-10** | **Revisão de segurança** | **mínimo 2 dias úteis** de Sofia antes do deploy, com foco em HMAC e geração de secret — **bloqueante** (`RES-06`) | Sofia `[09:46]` e `[09:49]` |

### 7.1 Ressalva obrigatória sobre RNF-01

**O alvo de 10 segundos é uma promessa de produto que o desenho aprovado não cumpre.** Isto precisa estar dito antes de o número ser comunicado a qualquer cliente.

Marcos disse *"qualquer coisa abaixo de 10 segundos já é 'tempo real'"* (`[09:02]`). A soma dos valores que a mesma reunião decidiu:

| Etapa | Pior caso | Decidido em |
|---|---|---|
| espera até o worker perceber o evento | 2 s | `DEC-02`, Larissa `[09:10]` |
| espera pela resposta do cliente | 10 s | `RNF-06`, Diego `[09:42]` |
| **total, no caminho sem nenhuma retentativa** | **> 12 s** | **excede o alvo** |

E no caminho com retentativa a distância é de outra ordem: um evento entregue na sexta chamada chega **14h36 depois** do fato — quatro ordens de grandeza acima do alvo.

**Três fatos sobre este conflito:**

1. **Ninguém o levantou na reunião.** Ele emergiu na escrita dos ADRs ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)) e foi escalado pela [RFC como questão 4](RFC.md#exigem-decisão-mas-não-travam-o-início).
2. **A RFC recomenda relaxar o alvo — e registra a recomendação como explicitamente NÃO decidida.** O FDD constrói os fluxos sobre os valores decididos e também não decide ([FDD §9.1](FDD.md)).
3. **Este PRD também não decide.** A decisão é de produto e o dono do requisito é Marcos. As três saídas na mesa: relaxar o alvo (reformulando `RNF-01` como meta de caminho feliz), reduzir o intervalo de espera do worker, ou reduzir o tempo de paciência com o cliente. Cada uma tem custo diferente, e nenhuma é neutra.

**Enquanto isso não for decidido, `RNF-01` fica registrado como o cliente o formulou, com o desvio conhecido ao lado.** O que Marcos descreveu materialmente — *"que não fique pendurado e eles tenham que ficar atualizando manualmente"* — o desenho entrega. O número na primeira metade da frase, não.

---

## 8. Decisões e trade-offs principais

Trade-offs de **produto**: cada linha abaixo tem custo visível para o cliente ou para o negócio. A justificativa técnica de cada decisão está no ADR correspondente e não é repetida.

### T-01 — Notificar depois, não durante (`ALT-01` → `ADR-001`)

**O que se ganha:** a mudança de status nunca depende do endpoint do cliente. Um cliente lento não trava pedidos de outros, e a indisponibilidade dele não reverte uma operação legítima.
**O que se paga:** a entrega não é imediata por construção. É a origem de todo o orçamento de latência da seção 7.1.

### T-02 — At-least-once: a deduplicação é do cliente (`DEC-10` / `ALT-07` → [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md))

**A decisão:** o mesmo evento pode chegar mais de uma vez. Fornecemos um identificador estável (`X-Event-Id`) e o cliente deduplica.
**O que se paga, e é custo do cliente, não nosso:** o custo de integração sobe para cada cliente. Pior — **duplicata não é caso excepcional, é consequência garantida**: um cliente que leve mais de 10 segundos para processar tem a resposta cortada, o evento é retentado, e ele processa duas vezes **sem que tenha havido falha nenhuma**. Quanto mais lento o cliente, mais duplicatas.

> **Esta decisão tem uma objeção que nunca foi retirada, e o PRD é o lugar dela.**
> Diego apresentou a posição (`[09:25]`); **Sofia respondeu que "isso joga responsabilidade pro cliente"**; Diego rebateu com argumento de mercado (Stripe, GitHub); Marcos disse que documentaria no portal; **Larissa fechou como decisão** (`[09:26]`). A nota de leitura (c)-4 do índice registra que **Sofia nunca expressou concordância** e não retirou a objeção.
> É a única alternativa descartada do pacote cujo descarte ainda tem discordância viva, e a objeção é de produto — o custo cai no cliente que estamos tentando reter. **Reabrir isso antes da revisão de segurança é ação, não observação.** Ver dependência D-02 e risco R-04.

### T-03 — Cobrir indisponibilidade longa em vez de falhar rápido (`ALT-04` → [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md))

Bruno propôs três tentativas, mais agressivo. Diego defendeu cinco com um caso concreto: *"Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada"* — três tentativas em 30 minutos não sobrevivem a isso. Larissa fechou em cinco.
*Desambiguação feita fora da reunião:* **cinco retentativas além do envio inicial — seis chamadas HTTP no total** (`TEC-24`, resolvido em [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) pela aritmética dos intervalos).
**O que se paga:** o evento fica em trânsito por até 14h36 em vez de morrer em 30 minutos, e o cliente não tem como acelerar — **entre a quinta e a sexta tentativa passam 12 horas**. Quem corrigiu o próprio endpoint às 10h da manhã pode só receber o evento retido às 22h.

### T-04 — Payload enxuto, sem os itens do pedido (`ALT-08`)

**O que se ganha:** evento pequeno e barato de transmitir.
**O que se paga:** o cliente que precisa da composição do pedido faz uma chamada extra a `GET /orders/:id` — ou seja, **a feature reduz o polling, mas não elimina toda consulta** para esse perfil. Tensão direta com OB-01 e MP-02.

### T-05 — Consistência acima de disponibilidade da escrita (`RF-11` + `DEC-18`)

`RF-11` faz a mudança de status **depender** do registro do evento: se o registro falhar, a operação de negócio inteira é revertida. Somado a `DEC-18` (payload acima de 64KB **falha, não trunca**), existe um caminho — remoto, mas real — em que **a feature de notificação derruba uma mudança de status legítima**. Diego avaliou o risco como remoto (*"Nenhum evento nosso vai chegar perto disso"*), e o trade-off foi assumido deliberadamente.

### T-06 — Ordem declarada como limitação, não construída como garantia (`DEC-04` → [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md))

*"Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker."*
**O que se paga, e o cliente sente:** se o evento `PAID` de um pedido falha e é reagendado, e o `SHIPPED` do **mesmo** pedido é entregue nesse intervalo, o cliente recebe `SHIPPED` antes de `PAID` — e pode regredir o estado no sistema dele. **A interação entre retentativa e ordem não foi levantada na reunião**, está escalada como [questão 5 da RFC](RFC.md#exigem-decisão-mas-não-travam-o-início) e permanece não decidida.

### T-07 — Configuração acessível a qualquer operador autenticado (`DEC-20`)

Sofia aceitou como estado temporário: *"Por enquanto sim. Mais pra frente a gente pode endurecer."* O endurecimento é `ADI-06`, **sem gatilho definido**. Consequência de produto na seção 3.1 e no risco R-05.

### T-08 — Nenhuma infraestrutura nova (`RES-01` / `ALT-02`)

*"a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering."*
**O que se paga:** o teto de vazão e o piso de latência da feature ficam determinados por essa escolha, e a saída (`ADI-05`) foi adiada sem limiar definido — ou seja, **a migração acontecerá sob pressão de capacidade, não por planejamento**.

---

## 9. Dependências

| # | Dependência | Natureza | Estado | Origem |
|---|---|---|---|---|
| **D-01** | **Revisão de segurança de Sofia — bloqueante para deploy**, mínimo 2 dias úteis, foco em HMAC e geração de secret | interna, bloqueante | **sem data e sem responsável pelo agendamento.** Sofia pediu para ser agendada (`[09:49]`) e ninguém assumiu (nota de leitura (b)-3) | `RNF-10` / `RES-06` |
| **D-02** | **Documentação da garantia at-least-once no portal do cliente** — a promessa de T-02 só funciona se o cliente souber que precisa deduplicar | interna, de produto | Marcos assumiu (`[09:26]`); **sem prazo e sem confirmação de execução** ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md)) | `[09:26]` |
| **D-03** | **Sessão de revisão do documento de design** com Bruno e Diego, antes de começar a codar | interna | Larissa assumiu (`[09:50]`); **sem data** (nota (b)-2) | `[09:50]` |
| **D-04** | **Confirmação do prazo com Atlas Comercial, MaxDistribuição e Nova Cargo** | externa | Marcos prometeu confirmar "hoje à tarde" (`[09:47]`); **o resultado nunca voltou** | `ABE-05` |
| **D-05** | **O cliente implementar a verificação da assinatura e a deduplicação por `X-Event-Id`** | externa, fora do nosso controle | não temos como verificar se ele faz — a garantia é contratual, não técnica | `RF-09` / `DEC-10` |
| **D-06** | **Onde o processo de entrega roda, quem o mantém vivo e quem é avisado se ele cair** | interna, de plataforma | **não tratado por ninguém na reunião**; `docker-compose.yml` provisiona apenas o MySQL, sem serviço de aplicação onde declarar o processo | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md), ponto em aberto |
| **D-07** | **Decisão sobre `RNF-01`** (seção 7.1) antes de o alvo de 10s ser comunicado a qualquer cliente | interna, de produto | **não decidida** | [RFC, questão 4](RFC.md#exigem-decisão-mas-não-travam-o-início) |
| **D-08** | **Decisão sobre as duas perguntas nunca feitas** da seção 5.3 (evento na criação do pedido; histórico na remoção da assinatura) | interna, de produto | **não decididas** | [FDD §3](FDD.md) |
| **D-09** | **Capacidade do time** — Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) em três sprints, com time declaradamente pequeno | interna | `DEC-13` fechou o prazo; nada foi dito sobre alocação | `DEC-13` / `RES-01` |
| **D-10** | **O ponto de acoplamento no código existente** — a mudança de status de pedido é o único gatilho | interna, técnica | decidido | `COD-13` |

---

## 10. Riscos e mitigação

| # | Risco | Prob. | Impacto | Mitigação | Origem |
|---|---|---|---|---|---|
| **R-01** | **Churn da Atlas Comercial** se a entrega escorregar. O prazo tem revisão de segurança bloqueante de ≥2 dias úteis **no fim** de três sprints, e ninguém agendou essa revisão | Alta | **Alto** — perda de cliente nomeado | **Agendar a revisão de Sofia agora, não na sprint 3.** Tratar D-01 como item de cronograma, não de qualidade. Reportar progresso ao cliente antes do fim de novembro | `RES-04` / `DEC-13` / `RNF-10` |
| **R-02** | **O prazo nunca foi confirmado com os três clientes.** "Fim de novembro" (`[09:45]`) é nossa data, não acordo deles; e a fala de `[09:00]` fala em "fim do trimestre" — duas referências diferentes circulando | Alta | Médio | Fechar D-04: Marcos confirma e registra a resposta dos três. **Até lá, não tratar fim de novembro como compromisso assumido** em nenhuma comunicação | `ABE-05` / `RES-04` |
| **R-03** | **Prometer "tempo real abaixo de 10 segundos" e não cumprir.** O desenho excede o alvo já no caminho sem retentativa (>12s) e chega a 14h36 no pior caso | Alta | Médio-Alto — quebra de expectativa no exato atributo que motivou o pedido | Resolver D-07 **antes** de qualquer material chegar ao cliente. Comunicar o alvo como caminho feliz, com o comportamento de exceção descrito | seção 7.1 · [RFC q4](RFC.md#exigem-decisão-mas-não-travam-o-início) |
| **R-04** | **O cliente não deduplica e processa o mesmo pedido duas vezes.** Duplicata é garantida, não excepcional (T-02), e a única defesa é código do cliente | Alta | **Alto** — efeito colateral no negócio do cliente, e a objeção de Sofia nunca foi respondida | Fechar D-02 antes do go-live, com `X-Event-Id` em destaque no material de integração. **Reabrir a objeção de Sofia sobre `DEC-10` antes da revisão de segurança** | `DEC-10` · nota (c)-4 |
| **R-05** | **Autorização frouxa entre clientes**: qualquer operador autenticado cadastra, edita e remove webhook de **qualquer** cliente, inclusive apontando a URL de um cliente para outro destino | Alta | **Alto** — exposição de dados entre clientes | Estado aceito como temporário por Sofia (`DEC-20`). **`ADI-06` é a saída e não tem gatilho** — definir o gatilho é ação de produto, não de engenharia | `DEC-20` / `ADI-06` |
| **R-06** | **A entrega para de funcionar em silêncio.** Se o processo de entrega cair, a API segue aceitando pedidos normalmente, nada quebra visivelmente, e os clientes simplesmente param de receber | Média | **Alto** | Fechar D-06 antes do deploy: onde o processo roda, quem o reinicia, quem é avisado. **Hoje não há onde declarar o processo nem alarme decidido** | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| **R-07** | **A feature de notificação derruba uma mudança de status legítima** (T-05: registro na mesma transação + payload acima de 64KB falha em vez de truncar) | Baixa | **Alto** — bloqueio de operação de negócio | Trade-off deliberado da reunião. Medir o tamanho real dos payloads antes de virar incidente | `RF-11` / `DEC-18` |
| **R-08** | **O cliente descobre tarde que parou de receber.** Sem aviso por email (`ADI-01`) e sem painel (`ADI-03`), a única forma é ele consultar o histórico por conta própria (`RF-06`) — e para isso precisa suspeitar antes | Alta | Médio | Nenhuma nesta fase, por escopo. Larissa condicionou `ADI-01` a "depois que a gente medir o impacto" — **MP-04 é o instrumento dessa medição** | `ADI-01` / `ADI-03` |
| **R-09** | **Evento fora de ordem regride o estado no sistema do cliente** (T-06): `SHIPPED` chega antes de `PAID` quando o primeiro é retentado | Média | Médio | Decisão pendente na [RFC q5](RFC.md#exigem-decisão-mas-não-travam-o-início). Se a saída for "aceitar", **isso precisa estar no material do cliente junto com a limitação de `DEC-04`** — hoje não está em lugar nenhum voltado ao cliente | [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md) |
| **R-10** | **Reprocessamento manual não escala.** Um cliente fora por mais de 15 horas gera um registro por evento perdido, cada um exigindo uma ação separada do administrador | Média | Médio | Nenhuma operação em lote foi decidida. Avaliar antes do primeiro incidente real, não durante | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) |
| **R-11** | **Surpresa de escopo por omissão** (5.3): um cliente assina `PENDING` esperando saber quando o pedido nasce, e não recebe nada | Média | Médio | Fechar D-08. Enquanto não fechar, **não oferecer `PENDING` como status assinável sem ressalva** no material do cliente | `CONTEXT.md` §17.2 |

---

## 11. Critérios de aceitação

Critérios de **produto**: cada um é verificável por observação do comportamento, sem abrir o código. A verificação técnica correspondente está no [FDD §13](FDD.md#13-critérios-de-aceite-técnicos) (22 critérios) e não é duplicada aqui.

| # | Critério | Requisito coberto |
|---|---|---|
| **AC-01** | Um cliente com endpoint cadastrado para um status recebe um `POST` quando um pedido dele muda para esse status, **sem ter feito nenhuma consulta** | OB-01, RF-05, RF-09 |
| **AC-02** | Um cliente que assinou apenas parte dos status **não recebe** os demais | RF-05 |
| **AC-03** | O cliente consegue verificar, com a secret que recebeu no cadastro, que o envio veio de nós | RF-09, RNF-09 |
| **AC-04** | Um endpoint `http` é **recusado no cadastro** | RNF-07 |
| **AC-05** | A secret é entregue ao cliente no cadastro e na rotação, e **em nenhum outro momento** | RF-01, RF-07 |
| **AC-06** | Após a rotação, o cliente que **ainda usa a secret antiga continua conseguindo validar** os envios durante 24 horas | RF-07, CU-05 |
| **AC-07** | Um endpoint indisponível por **duas horas** (a janela de manutenção planejada de `ALT-04`) recebe o evento quando volta | OB-03, CU-03 |
| **AC-08** | Um endpoint indisponível por mais tempo que a janela de retentativa não perde o evento: ele fica recuperável e um administrador consegue reprocessá-lo | OB-05, RF-08, CU-04 |
| **AC-09** | O reprocessamento é **negado a quem não é administrador** e **registra quem o executou** | RF-08, RF-10, CU-09 |
| **AC-10** | O cliente consegue consultar os últimos 100 envios feitos para o endpoint dele, com sucesso/falha, payload, response e tempo de resposta | RF-06, CU-07 |
| **AC-11** | Se o registro do evento falhar, **a mudança de status não acontece** — e não existe pedido com status alterado sem evento registrado | OB-04, RF-11 |
| **AC-12** | Um evento acima de 64KB **falha com erro**, sem truncamento silencioso | RNF-05, T-05 |
| **AC-13** | Um cliente com mais de um endpoint cadastrado consegue identificar **qual cadastro** recebeu cada envio | CU-06 |
| **AC-14** | A entrega é feita e verificada **sem nenhuma dependência nova** no projeto e sem serviço novo além do processo de entrega | OB-08 |
| **AC-15** | A revisão de segurança de Sofia ocorreu, com no mínimo 2 dias úteis, **antes do deploy** | OB-07, RNF-10, D-01 |
| **AC-16** | O material do cliente descreve, antes do go-live: a garantia at-least-once e a necessidade de deduplicar por `X-Event-Id`; a limitação de ordem; e o que a plataforma promete de latência (conforme a decisão de D-07) | D-02, T-02, T-06, R-03, R-04 |

**AC-15 e AC-16 são de produto e de processo, não de software.** Nenhum dos dois é verificável por teste automatizado, e ambos são bloqueantes para o go-live — `RES-06` torna a revisão condição para subir, e sem AC-16 a garantia de T-02 é uma promessa que o cliente não sabe que precisa cumprir.

---

## 12. Estratégia de testes e validação

Três camadas com propósitos diferentes. Só a primeira é automatizável, e ela é a que **não** pertence a este documento.

### 12.1 Verificação técnica — especificada em outro lugar

O [FDD §13](FDD.md#13-critérios-de-aceite-técnicos) define 22 critérios de aceite técnicos sobre o padrão de teste real do projeto: Vitest, integração contra banco real via `supertest`, aplicação montada inteira, sem mock de repositório (`CONTEXT.md` §14). O PRD **não repete e não altera** essa lista; os critérios da seção 11 são a camada de produto acima dela.

Uma consequência de produto vale registrar: **o processo de entrega não roda vivo durante os testes** — o que se verifica é um ciclo de processamento invocado diretamente ([FDD §13](FDD.md#13-critérios-de-aceite-técnicos)). Ou seja, **o comportamento contínuo do processo em produção não é coberto por teste automatizado**, e é exatamente onde mora o risco R-06.

### 12.2 Validação com o cliente — antes do go-live

| # | Validação | Por quê | Estado |
|---|---|---|---|
| V-01 | **Revisão de segurança de Sofia**, ≥2 dias úteis, com foco em HMAC e geração de secret | bloqueante por `RES-06`; é a única barreira de qualidade formal que a reunião estabeleceu | **sem data** (D-01) |
| V-02 | **Sessão de revisão de design** com Bruno e Diego antes de codar | Larissa assumiu (`[09:50]`) | **sem data** (D-03) |
| V-03 | **Confirmação do prazo com os três clientes** | R-02 | **pendente** (D-04) |
| V-04 | **Material de integração publicado no portal** cobrindo at-least-once, `X-Event-Id`, limitação de ordem e o alvo de latência decidido | AC-16; sem isso, T-02 é promessa unilateral | **pendente** (D-02) |
| V-05 | **Decisões de produto pendentes fechadas**: `RNF-01` (D-07) e as duas perguntas nunca feitas (D-08) | comunicá-las depois do go-live é comunicar mudança de contrato | **pendentes** |

### 12.3 Validação de produto — **proposta deste PRD, sem origem**

Nada abaixo foi decidido na reunião. É a proposta de como saber se a feature funcionou de verdade, e não apenas se o código passou.

| # | Proposta | Instrumento | Ressalva |
|---|---|---|---|
| VP-01 | **Medir o baseline de polling antes do go-live** — volume de `GET /orders` por cliente nos 30 dias anteriores | log `http_request` existente, com `path` e `userId` | **Pré-requisito de MP-02, não consequência.** Se não for medido antes, a métrica principal de produto fica sem referência para sempre |
| VP-02 | **Piloto com um dos três clientes** antes de liberar para os outros dois | acordo com o cliente escolhido | contradiz parcialmente OB-06 (3 de 3 rápido); é troca de velocidade por risco |
| VP-03 | **Exercício de indisponibilidade combinado com o cliente do piloto** — derrubar o endpoint dele por duas horas e verificar que os eventos chegam depois | reproduz CU-03, que é o caso real que motivou `ALT-04` | exige cooperação do cliente |
| VP-04 | **Verificar que o cliente do piloto realmente deduplica**, forçando um reenvio | única forma de reduzir R-04 de "não temos como saber" para "sabemos de um" | não escala para todos os clientes |
| VP-05 | **Acompanhamento de MP-01 a MP-05 nos 30 dias após o go-live**, com revisão explícita ao fim do período | ver seção 4.2 | sem alarme decidido, o acompanhamento é manual ([FDD §10.2](FDD.md)) |
| VP-06 | **Reavaliar `ADI-01` (aviso de falha ao cliente) ao fim dos 30 dias** | Larissa condicionou a próxima fase a "depois que a gente medir o impacto" (`[09:37]`) — MP-04 é a medição | o gatilho existe na fala; a data é proposta |

---

## 13. Questões abertas de produto

Consolidação do que este documento **deliberadamente não fecha**, com o dono natural de cada decisão.

| # | Questão | Dono natural | Onde está detalhada |
|---|---|---|---|
| 1 | **O alvo de 10 segundos vale como está, ou é reformulado?** | Marcos (dono do requisito) | seção 7.1 · [RFC q4](RFC.md#exigem-decisão-mas-não-travam-o-início) |
| 2 | **A criação do pedido gera evento?** Nunca foi perguntado | Marcos + Bruno | seção 5.3 |
| 3 | **O que acontece com o histórico quando a assinatura é removida?** Nunca foi perguntado | Marcos + Larissa | seção 5.3 |
| 4 | **A objeção de Sofia a `DEC-10` continua de pé?** Nunca foi retirada nem respondida | Sofia + Larissa | T-02 |
| 5 | **Evento fora de ordem: aceitar e documentar ao cliente, ou construir mecanismo?** | Larissa | T-06 · [RFC q5](RFC.md#exigem-decisão-mas-não-travam-o-início) |
| 6 | **Qual o gatilho de `ADI-06`** (endurecer a autorização entre clientes)? | Sofia + Marcos | T-07 · R-05 |
| 7 | **Limite de vazão de envios**: implementar, descartar, ou seguir "observando"? Classificação divergente entre Diego e Larissa | Larissa | `ABE-01` / `ADI-02` |
| 8 | **Fim de novembro é acordo ou é nossa data?** | Marcos | seção 2.2 · R-02 |
| 9 | **Quem agenda a revisão de segurança, e para quando?** Ninguém assumiu | Larissa | D-01 · R-01 |
| 10 | **Onde o processo de entrega roda e quem é avisado se ele cair?** | Diego | D-06 · R-06 |

---

## Fontes

**Índice da transcrição** (`.scratch/fontes/transcricao-index.md`): DEC-01 a DEC-20 · RF-01 a RF-11 · RNF-01 a RNF-10 · RES-01 a RES-07 · ALT-01, ALT-02, ALT-04, ALT-07, ALT-08 · ADI-01 a ADI-06 · ABE-01, ABE-03, ABE-04, ABE-05 · COD-13 · TEC-08, TEC-11, TEC-14, TEC-15, TEC-22, TEC-24, TEC-28, TEC-29 · notas de leitura (a), (b)-1, (b)-2, (b)-3 e (c)-4.

**Falas usadas que o índice não indexa como item próprio:** `[09:00]` Marcos (o baseline do polling em `GET /orders`, e "fim do trimestre") · `[09:45]` Marcos ("fim de novembro") · `[09:26]` Marcos (documentar a garantia no portal).

**Documentos do pacote**: [RFC](RFC.md) · [FDD](FDD.md) · [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) · [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) · [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) · [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) · [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md) · [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) · [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md).

**Arquivos reais**: `src/modules/orders/order.routes.ts` (`GET /orders`, o polling atual) · `src/modules/orders/order.service.ts` (`changeStatus`, `create`) · `src/middlewares/auth.middleware.ts` (`AuthUser`, `authenticate`, `requireRole`) · `src/middlewares/request-logger.middleware.ts` (`http_request`, `X-Request-Id`) · `docker-compose.yml` · `package.json` · `CONTEXT.md` §9, §10, §14, §16, §17.2.
