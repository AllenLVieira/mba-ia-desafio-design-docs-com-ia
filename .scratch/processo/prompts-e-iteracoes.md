# Log de processo — prompts e iterações

Registro contemporâneo da produção do pacote de design docs. Alimenta a seção
"Prompts customizados" e "Iterações e ajustes" do README final.

---

## Fase 0 — Base de fatos

### Ajuste #1 — correção do setup antes de produzir qualquer coisa

A skill de setup (`/setup-matt-pocock-skills`) gerou `docs/agents/domain.md` apontando
os ADRs para `docs/adr/`. O enunciado do desafio exige `docs/adrs/` — e a pasta é
critério de aceite avaliado. Se isso não tivesse sido pego antes da fase de ADRs, as
skills teriam escrito 5–8 arquivos numa pasta que não é avaliada.

Corrigido: duas ocorrências de `docs/adr/` → `docs/adrs/` em `docs/agents/domain.md`.

**Correção relacionada:** `docs/adrs/README.md` (vindo do repositório base) instruía
nomear ADRs como `0001-titulo-da-decisao.md`, enquanto o desafio exige
`ADR-NNN-titulo-em-kebab-case.md`. Alinhado ao formato do enunciado, com as seções
MADR obrigatórias explicitadas — duas fontes de instrução contraditórias dentro do
próprio repositório teriam desviado a skill de ADRs.

### Decisão de fluxo

O fluxo principal das skills de engenharia (`/to-spec` → `/to-tickets` → `/implement`
→ `/tdd`) foi descartado: ele existe para shippar código, e este desafio é puramente
documental — o enunciado proíbe tocar em `src/`, `prisma/`, `tests/`. Aproveitada
apenas a cabeça do fluxo (`/grill-with-docs` + `/domain-modeling`), que produz ADRs em
formato MADR — exatamente o artefato central exigido.

### Estratégia contra alucinação

Antes de escrever qualquer documento, duas varreduras de extração gravam em disco uma
base de fatos ancorada nas fontes. Todos os documentos posteriores leem essa base, não
a transcrição bruta nem o código inteiro. Consequência prática: o TRACKER deixa de ser
um trabalho de arqueologia no fim e vira uma junção contra um índice já pronto — e
qualquer linha que não fecha com o índice é, por construção, alucinação.

---

## Prompt customizado #1 — extração indexada da transcrição

Objetivo: transformar 55 minutos de reunião num índice rastreável, com citação
literal e timestamp por item. O ponto crítico do prompt é forçar a separação entre
DESCARTADO, ADIADO e EM ABERTO — três destinos diferentes nos documentos finais
(fora de escopo do PRD, alternativas do RFC, questões em aberto do RFC) que uma
leitura descuidada funde num só.

```
Você vai construir um ÍNDICE DE FONTES exaustivo do arquivo TRANSCRICAO.md.

CONTEXTO: este índice é a base de rastreabilidade de um pacote de design docs
(PRD, RFC, FDD, ADRs) sobre um "Sistema de Webhooks de Notificação de Pedidos"
discutido nessa reunião. Toda afirmação que entrar nos documentos finais precisará
ser rastreada a uma linha deste índice. Se o índice estiver incompleto ou impreciso,
os documentos finais vão conter alucinação. Precisão literal importa mais que
elegância.

TAREFA:
1. Leia o arquivo INTEIRO. Ele tem ~55 minutos de reunião no formato
   `[hh:mm] Nome: fala`. Não pule trechos, não amostre.
2. Produza o arquivo `.scratch/fontes/transcricao-index.md`.

ESTRUTURA DO ARQUIVO DE SAÍDA:

## Participantes
Tabela: Nome | Papel (só o papel que a própria transcrição revela — se o papel não
for explicitado nem inequívoco pela fala, escreva "não explicitado") | Primeira
aparição `[hh:mm]`.

## Índice de itens
Uma seção por balde, nesta ordem. Em cada seção, uma tabela com colunas:
`ID | Resumo (1 linha) | Falante | Timestamp | Citação literal`

Baldes e prefixos de ID:
- `DEC-NN` — Decisão fechada (o grupo convergiu e seguiu adiante)
- `RF-NN`  — Requisito funcional explícito
- `RNF-NN` — Requisito não funcional (performance, segurança, escala, latência, volume)
- `RES-NN` — Restrição (limite imposto: stack, prazo, infra, orçamento, compatibilidade)
- `ALT-NN` — Alternativa colocada na mesa e DESCARTADA (registre também o
             trade-off/motivo do descarte, na coluna Resumo)
- `ADI-NN` — Item adiado explicitamente para fase futura / v2 / depois
- `ABE-NN` — Questão levantada e deixada EM ABERTO (ninguém decidiu, ficou de
             verificar/pensar)
- `COD-NN` — Gancho com o código existente (menção a arquivo, módulo, classe,
             método, tabela ou padrão do sistema atual)
- `TEC-NN` — Detalhe técnico secundário (nome de header, valor de timeout, formato
             de payload, nome de coluna, código de erro)

REGRAS INEGOCIÁVEIS:
- A coluna `Citação literal` é copy-paste exato do arquivo, entre aspas. Nunca
  parafraseie nela. Se precisar cortar, use `...` no meio, jamais nas pontas.
- Timestamp no formato exato `[hh:mm]` como aparece no arquivo. Nome do falante
  exatamente como grafado.
- Não infira, não complete, não "melhore". Se a fala é vaga, o resumo registra a vagueza.
- A fronteira DESCARTADO (`ALT`) vs ADIADO (`ADI`) vs EM ABERTO (`ABE`) é o ponto mais
  importante deste trabalho — é o que separa "fora de escopo" de "questão em aberto"
  nos documentos finais. Se a transcrição não deixa claro em qual balde um item cai,
  coloque-o no balde mais provável E marque o ID com sufixo `(?)`, adicionando uma
  nota explicando a ambiguidade e citando as duas falas relevantes.
- Um mesmo trecho de fala pode gerar itens em mais de um balde. Duplicar é aceitável;
  omitir não é.
- Seja exaustivo. É esperado algo na faixa de 60 a 120 itens no total. Se você
  produzir 25, você amostrou em vez de varrer — volte e releia.

## Notas de leitura
Ao final, liste em prosa curta: (a) contradições ou reversões dentro da própria
reunião (alguém decidiu X às 10:00 e mudou para Y às 40:00 — registre ambos os
timestamps); (b) itens que alguém prometeu confirmar depois e nunca voltou;
(c) qualquer coisa que soe como decisão mas nunca teve concordância explícita.

NÃO modifique nenhum outro arquivo do repositório. Escreva apenas o arquivo de saída.
```

## Prompt customizado #2 — mapa do código existente

Objetivo: produzir o `CONTEXT.md` que sustenta a seção obrigatória "Integração com o
sistema existente" do FDD (mínimo 4 caminhos de arquivo reais). O prompt ataca o modo
de falha específico deste desafio — citar arquivo, método ou coluna que não existe —
exigindo leitura em vez de memória, e pedindo confirmação por busca de que a lacuna
que a feature preenche é real.

```
Você vai produzir o arquivo CONTEXT.md — o mapa do código existente deste repositório.

CONTEXTO: este repositório é um Order Management System (Node.js + TypeScript +
Prisma + MySQL) em produção. Vai ser escrito um pacote de design docs para uma feature
nova de Webhooks de Notificação de Pedidos, que NÃO existe ainda no código. O
CONTEXT.md que você produzir é a única fonte de verdade sobre o código para quem
escrever esses documentos depois — em particular para uma seção obrigatória chamada
"Integração com o sistema existente", que precisa nomear caminhos de arquivo REAIS e
descrever como o módulo novo se pluga em cada um.

Se você citar um arquivo, símbolo ou coluna que não existe, o documento final fica
errado e isso é o defeito mais grave possível aqui. Cada caminho e cada nome de
símbolo que você escrever precisa ter sido lido por você, não lembrado.

RESTRIÇÃO ABSOLUTA: é proibido modificar qualquer arquivo em `src/`, `prisma/`,
`tests/` ou qualquer config. Você só lê. O único arquivo que você escreve é
CONTEXT.md na raiz.

O QUE INVESTIGAR (leia os arquivos de verdade, não deduza pelos nomes):
- Estrutura geral: src/app.ts, src/server.ts, src/routes/index.ts, src/config/
- Anatomia de um módulo: pegue src/modules/orders/ inteiro (controller, service,
  repository, routes, schemas, status) e descreva o padrão em camadas que os módulos
  seguem — porque o módulo de webhooks vai ter que imitá-lo
- Ciclo de vida do pedido: src/modules/orders/order.status.ts e o método de mudança
  de status em src/modules/orders/order.service.ts. Este é o gancho principal da
  feature — descreva a máquina de estados (estados reais, transições permitidas), a
  assinatura exata do método, o que acontece dentro da transação, e onde exatamente
  um evento de webhook precisaria ser gravado
- Controle transacional: como transações Prisma são abertas, o que entra dentro delas
  (estoque, auditoria), qual o padrão de uso do client
- Auditoria de mudança de status: onde é gravada, qual modelo/tabela, quais campos
- Tratamento de erro: src/shared/errors/ completo — hierarquia de classes, como cada
  uma mapeia para status HTTP, e src/middlewares/error.middleware.ts. A feature nova
  vai reusar essas classes
- Formato de resposta HTTP: src/shared/http/response.ts
- Logging: src/shared/logger/ e src/middlewares/request-logger.middleware.ts
- Autenticação/autorização: src/middlewares/auth.middleware.ts e src/modules/auth/
- Validação: src/middlewares/validate.middleware.ts e o padrão dos *.schemas.ts
- Banco: prisma/schema.prisma — todos os modelos, com os campos que importam, enums,
  e as convenções (nomes de tabela, tipos de id, timestamps, soft delete se houver)
- Config/ambiente: src/config/env.ts e .env.example
- Testes: o que há em tests/, com que ferramenta (vitest.config.ts), e qual o padrão
- Scripts e execução: package.json, docker-compose.yml

CONFIRMAÇÃO EXPLÍCITA NECESSÁRIA: verifique com busca (não com suposição) que NÃO
existe hoje nenhum mecanismo de notificação externa, evento, fila, publisher, worker
ou webhook. Busque por termos como webhook, event, queue, publish, subscribe, outbox,
worker, job, cron, retry, hmac. Registre no documento o resultado dessa busca —
inclusive qualquer coisa que chegue perto e mereça ser mencionada.

FORMATO DO CONTEXT.md:
Markdown, organizado em seções. Regras de escrita:
- Todo caminho de arquivo em crase, relativo à raiz
- Ao descrever comportamento, cite o símbolo real (nome de método, classe, enum,
  campo) — nunca uma descrição genérica
- Quando um trecho for central (a máquina de estados, a assinatura do método de
  mudança de status, a hierarquia de erros), inclua o código real em bloco, curto,
  com o caminho do arquivo acima
- Termine com uma seção "Pontos de extensão para a feature de webhooks": uma lista
  dos lugares concretos do código onde a feature vai encostar, cada um com caminho de
  arquivo e uma frase sobre a natureza do acoplamento. Esta seção é matéria-prima
  direta para o FDD — quanto mais concreta, melhor. Descreva apenas o que o código
  sustenta; não proponha o desenho da feature.
```

---

### Ajuste #2 — timestamps em faixa no índice da transcrição

A varredura produziu 112 itens, mas 4 deles vieram com timestamp em faixa
(`[09:17-18]`, `[09:12-13]`, `[09:08-09]`, `[09:44-45]`) em vez do formato `[hh:mm]`
exigido. A IA fez isso quando o item resumia uma troca que atravessava dois minutos —
comportamento razoável, formato inválido: o critério de aceite do TRACKER exige
timestamp válido no formato `[hh:mm] Nome` em ≥70% das linhas, e uma faixa não é
timestamp válido.

Corrigido buscando na transcrição a fala que contém a citação literal de cada um e
usando o timestamp dessa fala:
`[09:17-18]` → `[09:18]`, `[09:12-13]` → `[09:13]`, `[09:08-09]` → `[09:08]`,
`[09:44-45]` → `[09:44]`.

Lição para os prompts seguintes: exigir formato não basta, é preciso proibir
explicitamente a variante plausível que a IA vai inventar quando o dado real não
couber no formato.

### Verificação da Fase 0

Antes de aceitar as duas varreduras, foram checadas por amostragem as afirmações de
maior risco, em vez de confiar no relatório dos agentes:

- Assinatura de `changeStatus` — confere com `src/modules/orders/order.service.ts:126`
- Modelo `OrderStatusHistory` — existe em `prisma/schema.prisma:116`
- 4 citações literais do índice (`overengineering`, `webhook_dead_letter`,
  `Email tá fora de escopo`, `não notifica processo externo`) — batem palavra por
  palavra com a transcrição, com falante e timestamp corretos
- 5 participantes na transcrição (Diego 42 falas, Larissa 41, Bruno 29, Marcos 21,
  Sofia 18) — confere com a tabela de participantes do índice

---

## Fase 1 — Fechamento das decisões em ADRs

### Prompt customizado #3 — grilling até os ADRs

Objetivo: produzir 5–8 ADRs em `docs/adrs/` sem que nenhuma linha deles seja invenção.
O prompt ataca dois modos de falha ao mesmo tempo. O primeiro é a alucinação, contida
pela regra de rastreabilidade dupla — índice ou caminho de arquivo real, nada mais. O
segundo é mais sutil: uma IA que encontra ambiguidade na fonte tende a **escolher
silenciosamente** a leitura mais plausível e seguir escrevendo, e o resultado passa
despercebido porque parece decidido. Por isso o prompt não pede "resolva as
ambiguidades" — ele nomeia as três (ABE-02, ABE-03, TEC-24) e exige que a sessão
**comece** perguntando sobre elas. A escolha do `/grill-with-docs` como veículo é
consequência disso: é a skill que interroga antes de produzir, em vez de produzir e
pedir revisão depois.

```
Preciso fechar as decisões arquiteturais do Sistema de Webhooks de Notificação de
Pedidos e registrá-las como ADRs em docs/adrs/, formato
ADR-NNN-titulo-em-kebab-case.md.

Fontes (leia antes de começar, são a única base factual permitida):
- .scratch/fontes/transcricao-index.md — 112 itens da reunião, com citação
  literal, falante e timestamp
- CONTEXT.md — mapa do código existente, com os pontos de extensão reais

Restrições: entrega puramente documental, é proibido alterar src/, prisma/ e
tests/. Nada pode entrar num ADR sem origem rastreável a uma linha do índice ou
a um caminho de arquivo real.

Alvo: 5 a 8 ADRs cobrindo pelo menos 5 destas 6 decisões — outbox no MySQL,
retry com backoff e DLQ, HMAC-SHA256 com secret por endpoint, at-least-once com
X-Event-Id, worker em processo separado com polling, reuso dos padrões
existentes. Pelo menos um ADR deve citar arquivos reais do código.

Comece me perguntando sobre as três ambiguidades marcadas no índice
(ABE-02, ABE-03 e a semântica de "5 tentativas" em TEC-24) — a transcrição não
as resolveu e eu preciso decidir.
```

**Resultado:** 7 ADRs, cobrindo 6 das 6 decisões-alvo. Duas rodadas de perguntas, 11
decisões fechadas antes de qualquer arquivo ser escrito.

### Ajuste #3 — terceira fonte de instrução contraditória no repositório

O Ajuste #1 corrigiu `docs/agents/domain.md` (caminho `docs/adr/` → `docs/adrs/`) e
`docs/adrs/README.md` (formato de nome de arquivo). A fase de ADRs revelou que os dois
arquivos **ainda se contradiziam**, agora nos títulos de seção: o README prescrevia
Status / Contexto / Decisão / Alternativas Consideradas / Consequências em português;
o `domain.md` prescrevia Status / Context / Decision / Alternatives Considered /
Consequences em inglês.

Diferente das duas primeiras contradições, esta não teria quebrado critério de aceite —
teria produzido 7 ADRs internamente consistentes num idioma, com o arquivo que descreve
o layout de documentação apontando para outro. Corrigido no sentido do README (é o
arquivo específico do diretório de destino), e o `domain.md` ganhou as duas seções
extras que os ADRs passaram a usar: **Pontos em aberto** e **Fontes**.

Lição: contradição entre arquivos de instrução do próprio repositório não é evento
único a ser corrigido, é classe de defeito a ser varrida a cada fase que consome esses
arquivos.

### Ajuste #4 — política para o que a fonte não diz

O grilling expôs um conflito entre duas exigências do desafio. A rastreabilidade proíbe
afirmação sem origem; mas um ADR sobre retry não consegue descrever agendamento sem
falar de estado persistido, e a única fonte sobre o schema da outbox (TEC-01) menciona
apenas índices em status e `created_at`. Três saídas foram postas na mesa: inventar os
campos com marca `[derivado]`, descrever e listar em Pontos em aberto, ou não descrever
mecanismo nenhum.

Escolhida a terceira: **os ADRs registram política e trade-off, não schema.** Onde a
ausência deixa buraco real, ele vai para Pontos em aberto nomeando o ABE/ADI
correspondente. Única derivação permitida é cálculo aritmético sobre valores citados.

Essa política é o que permitiu desambiguar TEC-24 sem inventar nada: a soma dos cinco
intervalos de backoff de TEC-23 dá 14h36min, e Diego descreve "quase 15 horas entre
primeira falha e última tentativa" (RNF-02). Só fecha se os cinco intervalos vierem
**depois** da primeira falha — logo, 5 retentativas além do envio inicial, 6 chamadas
HTTP no total. A leitura alternativa (5 no total) usaria quatro intervalos e daria
2h36min, contradizendo a fonte. Decisão derivada da própria transcrição, não de
preferência.

### Verificação da Fase 1 — três conflitos que a reunião não levantou

Escrever as consequências de cada ADR com trade-off explícito, em vez de listar
benefícios, fez emergir três contradições entre decisões que foram tomadas em momentos
diferentes da reunião e nunca confrontadas entre si. Nenhuma foi resolvida por conta
própria; todas estão registradas como Pontos em aberto, matéria-prima direta para a
seção de questões em aberto do RFC:

1. **O orçamento de latência não fecha** (ADR-002). RNF-01 pede menos de 10s
   end-to-end; DEC-02 (polling de 2s) somado a RNF-06 (timeout de 10s) já excede isso
   no pior caso. O alvo descreve o caminho feliz.
2. **O retry quebra a ordem por `order_id` mesmo com worker único** (ADR-007). Se o
   evento `PAID` entra em retry de 1 minuto e o `SHIPPED` do mesmo pedido é entregue
   nesse intervalo, o cliente recebe fora de ordem. DEC-04 e DEC-05 foram decididas com
   4 minutos de diferença e nunca confrontadas.
3. **`errorMiddleware` não alcança o worker** (ADR-006). COD-08 afirma que o middleware
   pega os erros do módulo sem alteração — verdade no caminho HTTP, mas o worker roda
   fora do Express por DEC-03, e o tratamento de erro do processo de entrega ficou sem
   decisão.

Lição: a seção de consequências negativas é onde o design é de fato auditado. Exigir
trade-off explícito não é formalidade do MADR — é o mecanismo que força o confronto
entre decisões tomadas isoladamente.

---

## Fase 2 — RFC

### Prompt customizado #4 — consolidação técnica em RFC

Objetivo: escrever `docs/RFC.md` consolidando a proposta técnica sobre as decisões já
registradas. O risco central desta fase é diferente do das anteriores: não é
alucinação, é **redecisão** — uma sessão nova que lê a transcrição e os ADRs tende a
reabrir o que já está fechado, produzindo um RFC que contradiz os ADRs em detalhes
menores. Daí a instrução explícita de referenciar, não redecidir.

O segundo cuidado é de continuidade: três ambiguidades foram resolvidas *fora* da
reunião e três conflitos foram descobertos *durante a escrita* dos ADRs. Nada disso
está na transcrição, e a sessão nova começa fria. O prompt carrega os seis itens
inline, porque uma sessão que não os conheça vai ou reabrir as ambiguidades já
fechadas, ou repetir os conflitos como se fossem novidade.

```
Preciso escrever a RFC do Sistema de Webhooks de Notificação de Pedidos em
docs/RFC.md (o arquivo existe como stub).

A RFC consolida a proposta técnica em cima das decisões já registradas, das
alternativas descartadas e das questões em aberto da reunião. Ela NÃO redecide
nada que já virou ADR — referencia.

Fontes (leia todas antes de começar, são a única base factual permitida):
- docs/adrs/ADR-001-outbox-transacional-no-mysql.md
- docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md
- docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md
- docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md
- docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md
- docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md
- docs/adrs/ADR-007-ordering-por-order-id-sob-single-worker.md
- .scratch/fontes/transcricao-index.md — 112 itens da reunião, com citação
  literal, falante e timestamp
- CONTEXT.md — mapa do código existente, com os pontos de extensão reais

Restrições: entrega puramente documental, é proibido alterar src/, prisma/ e
tests/. Nada entra na RFC sem origem rastreável a um ADR, a uma linha do índice
da transcrição, ou a um caminho de arquivo real. Documento em pt-BR.

A RFC deve cobrir, no mínimo: proposta técnica consolidada (como as 7 decisões
se compõem num sistema só, ponta a ponta), contrato de API dos endpoints
(RF-01 a RF-08), alternativas descartadas com o motivo, e as questões que
seguem em aberto.

Contexto que você precisa saber e não está nos arquivos — três ambiguidades da
reunião foram resolvidas fora dela, na sessão que produziu os ADRs, e estão
marcadas como tal nos documentos:
- ABE-02: arquivo de lógica do worker é webhook.worker.ts (convive com
  src/worker.ts, que é o entry point do processo)
- ABE-03: customer_id vai no body do POST e na query do GET, seguindo a
  convenção de src/modules/orders/order.schemas.ts; path fica para o :id do
  próprio recurso
- TEC-24: "5 tentativas" = 5 retentativas além do envio inicial, 6 chamadas
  HTTP no total

E três conflitos que a escrita dos ADRs fez emergir, que a reunião nunca
levantou e que estão registrados como Pontos em aberto:
1. o alvo de latência de RNF-01 (<10s) não fecha com polling de 2s + timeout
   de 10s (ADR-002)
2. o retry quebra a ordem por order_id mesmo com worker único (ADR-007)
3. errorMiddleware não alcança o worker, que roda fora do Express (ADR-006)

Comece me perguntando: (a) se a RFC pode propor schema de tabelas e campos —
os ADRs deliberadamente não propuseram, e ABE-04 (histórico de entregas), o
agendamento do retry e o armazenamento da secret continuam sem estrutura
definida; (b) se ela deve propor solução para os três conflitos acima ou apenas
escalá-los para decisão; (c) se entra plano de entrega nas três sprints de
DEC-13, com a revisão de segurança bloqueante de RNF-10 no caminho crítico.
```

---

## Fase 3 — FDD

### Prompt customizado #5 — o documento mais técnico do pacote

Objetivo: escrever `docs/FDD.md` em nível acionável, sem que a descida ao detalhe vire
porta de entrada para invenção. O risco desta fase é diferente das anteriores e maior
que ambas. Na Fase 1 o risco era alucinação; na Fase 2, redecisão. Aqui é **invenção
plausível**: um FDD precisa de schema, de payload de exemplo, de código de erro, de
nome de métrica — e nada disso existe na transcrição. Uma IA produz tudo isso com
fluência e o resultado *parece* especificação, não parece invenção. É o único documento
do pacote em que a alucinação sai bem escrita.

O prompt carrega inline os seis itens de contexto que não estão em arquivo nenhum (as
três ambiguidades resolvidas fora da reunião, os três conflitos escalados), como o #4
já fazia, e acrescenta dois que só existem depois da RFC: a afirmação mais tensionada
do pacote (dois campos estruturantes sem origem que a política de ADR-003 exige) e o
fato de que o caminho do endpoint de rotação é proposta, não decisão.

```
de novo — o FDD é o documento mais técnico do pacote e é onde a invenção tem mais
espaço para entrar sem ser notada.

Preciso escrever o FDD do Sistema de Webhooks de Notificação de Pedidos em
docs/FDD.md (o arquivo existe como stub).

O FDD é o documento de implementação: um dev pega ele e começa a codar. Ele NÃO
redecide o que virou ADR e NÃO repete a altura da RFC — desce ao detalhe que a
RFC deliberadamente não desceu.

Fontes (leia todas antes de começar, são a única base factual permitida):
- docs/RFC.md — a proposta consolidada, com a superfície de API, os artefatos
  persistentes e as 12 questões em aberto
- docs/adrs/ADR-001 a ADR-007 (todos os sete)
- .scratch/fontes/transcricao-index.md — 112 itens da reunião, com citação
  literal, falante e timestamp
- CONTEXT.md — mapa do código existente, com os pontos de extensão reais

Restrições: entrega puramente documental, é proibido alterar src/, prisma/ e
tests/. Nada entra no FDD sem origem rastreável a um ADR, à RFC, a uma linha do
índice da transcrição, ou a um caminho de arquivo real. Documento em pt-BR.

O FDD deve cobrir, no mínimo: contexto e motivação técnica, objetivos técnicos,
escopo e exclusões, fluxos detalhados (criação do evento na outbox,
processamento pelo worker, retry, DLQ), contratos públicos (endpoints HTTP com
payload de request e response e status codes — no mínimo 4), matriz de erros
com códigos WEBHOOK_*, estratégias de resiliência, observabilidade (métricas,
logs e tracing), dependências e compatibilidade, critérios de aceite técnicos,
riscos e mitigação. E uma seção obrigatória "Integração com o sistema
existente" que nomeie no mínimo 4 caminhos de arquivo reais e descreva o
acoplamento de cada um.

Contexto que você precisa saber e não está nos arquivos:
- Três ambiguidades da reunião foram resolvidas fora dela e estão marcadas como
  tal: ABE-02 (arquivo de lógica é webhook.worker.ts, convivendo com
  src/worker.ts como entry point), ABE-03 (customer_id no body do POST e na
  query do GET, seguindo order.schemas.ts), TEC-24 ("5 tentativas" = 5
  retentativas além do envio inicial, 6 chamadas HTTP no total).
- A RFC declara que dois campos estruturantes — contagem de tentativas e
  instante da próxima tentativa na outbox — não têm origem em nenhuma fonte e
  mesmo assim a política de ADR-003 os exige. É a afirmação mais tensionada do
  pacote inteiro; o FDD é quem tem que dar forma a eles.
- A RFC escalou três conflitos com recomendação explicitamente NÃO decidida:
  o alvo de latência de RNF-01, a quebra de ordem sob retry, e errorMiddleware
  não alcançar o worker. Nenhum foi decidido por ninguém.
- O caminho do endpoint de rotação de secret (RF-07) é proposta da RFC, sem
  nenhuma origem na transcrição. Sofia só disse "endpoint pro cliente conseguir
  pedir nova secret pela API".

Comece me perguntando:

(a) Observabilidade é a seção com maior risco de invenção do pacote. A
transcrição dá exatamente três coisas: o logger Pino já integrado (COD-07), o
requestLogger com X-Request-Id de CONTEXT.md §9, e RF-10 (logar quem executou o
replay). Métricas e tracing não têm uma linha de origem. O FDD deve derivar a
observabilidade dos padrões que já existem no código e marcar métricas/tracing
como proposta sem origem, ou pode nomear um stack concreto?

(b) O FDD precisa propor schema completo — models Prisma nas convenções de
prisma/schema.prisma — para os quatro artefatos, incluindo a tabela de
histórico de entregas que ABE-04 deixou sem estrutura e sem a qual RF-06 não é
implementável? Ou o schema para de onde a RFC parou?

(c) Os fluxos detalhados esbarram nos três conflitos escalados. O FDD constrói
em cima da recomendação da RFC (documentando como premissa assumida),
especifica os dois caminhos em cada bifurcação, ou deixa o fluxo aberto no
ponto do conflito?
```

**Resultado:** três rodadas de perguntas, 17 decisões fechadas antes de qualquer linha
ser escrita. O FDD saiu com 1.109 linhas, 7 contratos de endpoint (o mínimo eram 4),
10 caminhos de arquivo reais na seção de integração (o mínimo eram 4) e um ledger final
de 21 itens marcados como proposta sem origem.

### Ajuste #5 — a política da Fase 1 não sobrevive ao FDD, e isso é correto

O Ajuste #4 fixou que **ADRs registram política e trade-off, não schema**, e que a única
derivação permitida era cálculo aritmético sobre valores citados. Essa política levou a
RFC a declarar, com todas as letras, que dois campos estruturantes não têm origem em
fonte nenhuma e que a arquitetura os exige mesmo assim.

O FDD não pode herdar essa política sem deixar de ser FDD. Um documento de implementação
que se recusa a propor schema não é acionável — que é o único critério que o define. A
política nova, portanto: **onde a fonte é omissa e a implementação exige, o FDD propõe e
assume a autoria da proposta**, com marca visível.

O mecanismo é uma convenção de três marcas aplicada em todas as tabelas do documento —
**F** (fonte direta), **D** (derivado por aritmética, convenção do código ou implicação
lógica) e **P** (proposta sem origem) — mais um ledger final que reúne os 21 itens **P**
em um lugar só, cada um com a coluna "o que quebra se for rejeitado". A diferença
prática entre isso e "escrever bonito e torcer" é que o revisor consegue atacar as 21
linhas do ledger em vez de reler 1.109.

### Verificação da Fase 3 — cinco defeitos que só apareceram ao descer ao detalhe

Assim como a Fase 1 usou as consequências negativas para auditar o design, esta fase
usou o **schema concreto** como instrumento: campos e queries reais forçam perguntas que
prosa não força. Cinco achados, nenhum deles visível nos documentos anteriores:

1. **`WEBHOOK_SECRET_REQUIRED` não tem gatilho possível.** Bruno nomeia o código às
   `[09:29]` (TEC-16); às `[09:31]` Marcos decide que "secret é gerada pela gente e
   devolvida na criação" (TEC-14). O cliente nunca fornece secret, logo nunca pode
   omiti-la. O código foi batizado dois minutos antes da decisão que o tornou
   inalcançável — e atravessou ADR-006 e a RFC sem que ninguém notasse.
2. **`DEC-09` e `TEC-16` são incompatíveis no caminho HTTP.** Sofia decidiu que a recusa
   de `http` é "só uma validação no schema Zod"; mas Zod passa por
   `validate.middleware.ts` e sai do `errorMiddleware` como `VALIDATION_ERROR`, nunca
   como `WEBHOOK_INVALID_URL`. Duas fontes literais, uma resposta HTTP.
3. **O grace period de 24h não significa nada sem uma decisão que ninguém tomou.** Nós
   assinamos, o cliente verifica: "a secret antiga fica válida" só tem sentido se o
   envio carregar algo que a secret antiga valide. A transcrição descreve o efeito
   pretendido sem descrever o mecanismo, e as três saídas possíveis (assinar com a nova,
   com a antiga, ou com as duas) têm consequências completamente diferentes.
4. **O replay perde a identidade do evento.** TEC-28 diz "recoloca na outbox como
   pendente" e não diz o que acontece com o `event_id`. Se ele muda, a dedup por
   `X-Event-Id` — a única defesa que ADR-005 oferece ao cliente — falha exatamente no
   caso que ela existe para cobrir.
5. **Toda linha reivindicada morre presa a cada deploy do worker.** O status
   `processando` de TEC-01 não tem regra de saída para o caso de o processo morrer no
   meio. Não é cenário exótico: é garantido em todo restart. E o resultado — evento que
   nunca sai — contradiz frontalmente o RF-11 que motivou a arquitetura inteira.

O achado 5 tem um desdobramento que vale registrar como método: a solução (devolver a
linha a `pendente` e arriscar entrega dupla) parecia exigir uma garantia nova, até
ficar claro que **duplicata já é o contrato aceito por DEC-10**. A recuperação é
licenciada por uma decisão existente, não é liberdade inventada. Procurar *qual decisão
já tomada autoriza o que você quer fazer* é mais barato que decidir de novo — e deixa
rastro.

### Ajuste #6 — a contradição de instruções é classe de defeito, não evento

O Ajuste #3 já tinha registrado a lição, e ela se confirmou pela quarta vez: `AGENTS.md`
na raiz ainda apontava `docs/adr/` (singular), depois de `docs/agents/domain.md` e
`docs/adrs/README.md` terem sido corrigidos nas fases anteriores. Corrigido nesta fase.

O padrão agora é claro o suficiente para virar procedimento: **a cada fase, varrer os
arquivos de instrução do repositório antes de consumi-los**, em vez de corrigir o que
aparece. Uma busca por `docs/adr/` teria pego os três de uma vez na Fase 0.

---

## Fase 4 — PRD

### Prompt customizado #6 — o risco inverso do FDD

Objetivo: escrever `docs/PRD.md` em altura de produto. O risco desta fase é o **inverso**
do da Fase 3. O FDD inventa detalhe porque precisa de detalhe e a fonte não tem; o PRD
**rouba** detalhe — com quatro documentos técnicos prontos e uma transcrição rica em
decisões de engenharia, o caminho de menor resistência é resumir a RFC, reciclar o
schema do FDD e entregar um documento técnico com capa de produto. Um PRD assim não
responde "por que" nem "o quê": responde "como", de novo, mais curto.

Há um segundo risco, e ele é de invenção pura: **métricas**. A reunião não tem uma única
métrica de produto — nenhuma meta de adoção, nenhum KPI, nenhum baseline numérico. Só
existem números técnicos (10s, ~15h, 64KB, três sprints, três clientes, dois dias úteis
de revisão). Qualquer meta de produto que apareça no documento é invenção, e é o tipo de
invenção que passa despercebida porque *todo PRD tem métricas* — a ausência é que chama
atenção, não a presença.

O prompt carrega inline sete itens de contexto que não estão em arquivo nenhum. Quatro
são novos em relação ao #5, e três deles são achados que só existem porque as fases
anteriores foram escritas: o estado atual que ficou fora de todos os documentos, a
divergência de prazo que o índice fundiu numa linha só, e a categoria de escopo que o
FDD criou e que não pode ser achatada.

*(O prompt chegou com o bloco de abertura duplicado em colagem; reproduzido abaixo já
consolidado, com a lista de fontes da segunda cópia, que é a completa.)*

```
Vamos agora montar o PRD — o PRD é o último dos documentos grandes e o risco dele
é o inverso do FDD: em vez de inventar detalhe, ele tende a roubar detalhe do FDD
e virar um resumo técnico com capa de produto.

Preciso escrever o PRD do Sistema de Webhooks de Notificação de Pedidos em
docs/PRD.md (o arquivo existe como stub).

O PRD opera em altura de produto e negócio: responde "por quê e o quê". Ele NÃO
repete a arquitetura da RFC, NÃO redecide o que virou ADR e NÃO importa nada da
especificação de implementação do FDD. Onde precisar apontar para o "como", ele
referencia o documento certo com link.

Fontes (leia todas antes de começar, são a única base factual permitida):
- .scratch/fontes/transcricao-index.md — 112 itens da reunião, com citação
  literal, falante e timestamp
- docs/RFC.md — proposta consolidada, alternativas e 12 questões em aberto
- docs/FDD.md — especificação de implementação, com um ledger final de 21 itens
  marcados como proposta sem origem
- docs/adrs/ADR-001 a ADR-007 (todos os sete)
- CONTEXT.md — mapa do código existente

Restrições: entrega puramente documental, é proibido alterar src/, prisma/,
tests/ e package.json. Nada entra no PRD sem origem rastreável a uma linha do
índice da transcrição, a um documento do pacote, ou a um caminho de arquivo
real. Documento em pt-BR.

O PRD deve cobrir, no mínimo: resumo e contexto da feature, problema e
motivação, público-alvo e cenários de uso, objetivos e métricas de sucesso,
escopo (incluso e fora de escopo), requisitos funcionais, requisitos não
funcionais, decisões e trade-offs principais, dependências, riscos e mitigação,
critérios de aceitação, estratégia de testes e validação.

Alvos numéricos do critério de aceite: no mínimo 8 requisitos funcionais
discutidos na reunião (o índice tem RF-01 a RF-11); pelo menos 1 objetivo com
métrica e meta quantitativa; "Fora de escopo" com pelo menos 2 itens
explicitamente descartados ou adiados na reunião; "Riscos" com pelo menos 2
riscos trazendo probabilidade, impacto e mitigação.

Contexto que você precisa saber e não está nos arquivos:

- MÉTRICAS SÃO O MAIOR VETOR DE INVENÇÃO DESTE DOCUMENTO. A reunião não tem
  nenhuma métrica de sucesso de produto: não há meta de adoção, nem KPI, nem
  baseline numérico. Existem apenas números técnicos (RNF-01 10s, RNF-02 ~15h,
  RNF-05 64KB, DEC-13 três sprints, TEC-29 três clientes). Qualquer meta de
  produto que você escrever é proposta sua e precisa estar marcada como tal.

- O ESTADO ATUAL ESTÁ EM UMA FALA SÓ, e nenhum documento do pacote a usou porque
  todos operam acima dela. [09:00] Marcos: os clientes "ficam batendo no
  GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá
  deixando a integração lenta e cara pra eles". É o baseline do problema e a
  origem da expressão "tempo real" de RNF-01.

- O PRAZO TEM DUAS FALAS DIFERENTES e o índice as fundiu numa linha só. RES-04
  resume "fim de novembro" mas cita [09:00], onde Marcos diz "fim do trimestre".
  "Fim de novembro" é literal de [09:45] Marcos. Use [09:45] como referência do
  prazo; e registre que ABE-05 deixou a confirmação com os clientes em aberto —
  Marcos prometeu confirmar e o resultado nunca voltou. O prazo não é
  compromisso confirmado.

- O ALVO DE LATÊNCIA DE RNF-01 É PROMESSA DE PRODUTO QUE O DESENHO NÃO CUMPRE.
  Marcos disse "qualquer coisa abaixo de 10 segundos já é tempo real" [09:02];
  RFC e FDD registram que polling de 2s somado a timeout de 10s excede o alvo já
  no caminho sem retry, e que na sexta tentativa o evento chega ~14h36 depois.
  A RFC escalou isso com recomendação explicitamente NÃO decidida. O PRD não
  pode afirmar o alvo sem a ressalva, nem decidir o conflito sozinho.

- O FDD CRIOU UMA CATEGORIA DE ESCOPO QUE NÃO EXISTE NA TRANSCRIÇÃO e ela precisa
  sobreviver: "fora de escopo — não decidido por ninguém", separada dos ADI-01 a
  ADI-06 que alguém adiou explicitamente. Hoje ela tem dois itens: se a criação
  do pedido (OrderService.create) gera evento, e o que acontece com o histórico
  quando uma assinatura é removida. Achatar as duas categorias numa só apaga a
  diferença entre "decidimos deixar pra depois" e "ninguém perguntou".

- NADA DO LEDGER DE 21 ITENS DO FDD VIRA REQUISITO DO PRD. São propostas de
  implementação sem origem; requisito de produto sai de RF/RNF do índice.

- SOFIA NÃO CONCORDOU COM DEC-10. A nota de leitura (c)-4 registra que ela
  levantou "isso joga responsabilidade pro cliente" sobre a entrega at-least-once
  e nunca retirou a objeção; Larissa fechou como decisão. É trade-off de produto
  com custo para o cliente, não detalhe técnico, e o PRD é o lugar dele.

- TEC-24 foi desambiguado fora da reunião: "5 tentativas" = 5 retentativas além
  do envio inicial, 6 chamadas HTTP no total.

Comece me perguntando:

(a) Métricas de sucesso. Sem nenhuma métrica de produto na fonte, o PRD deve
derivar objetivos mensuráveis apenas dos números técnicos que existem (latência,
janela de retry, três clientes, três sprints) e marcar como proposta qualquer
meta que eu queira além disso? Ou deve propor um conjunto de métricas de produto
de verdade — adoção, redução de polling no GET /orders, taxa de entrega — todas
rotuladas como proposta sem origem? E o PRD carrega a convenção de marcas F/D/P
do FDD, ou ela pesa demais num documento de produto e fica só nas métricas e no
escopo?

(b) RNF-01. O PRD registra "menos de 10 segundos" como Marcos falou, com a
ressalva de que o desenho aprovado não cumpre isso no pior caso e link para a
questão em aberto da RFC? Ou reformula o requisito como meta de caminho feliz —
que é a recomendação NÃO decidida da RFC, e adotá-la aqui seria decidir um
conflito escalado dentro de um PRD?

(c) Público-alvo. A reunião tem uma distinção que nenhum documento explorou: o
JWT do sistema é de usuário operador, não de cliente (RES-07 e nota de leitura
(a)), então quem configura o webhook pela API e quem recebe o POST são dois
públicos diferentes. O PRD deve tratar os dois separadamente, com cenários de
uso para cada um? E os cenários saem só do que tem origem — os três clientes de
TEC-29, a janela de manutenção de duas horas que Diego citou em ALT-04, e o
risco de churn da Atlas em RES-04 — ou posso propor cenários adicionais
marcados?
```

**Resultado:** quatro decisões fechadas antes da primeira linha — a pergunta (a) foi
quebrada em duas na resposta, porque "de onde saem as métricas" e "o documento carrega
F/D/P" são escolhas independentes. As quatro: métricas em modelo **híbrido** (espinha
derivada dos números que existem + bloco de métricas de produto integralmente rotulado
como proposta); marcas **F/D/P restritas às seções de métricas e escopo**; `RNF-01`
registrado **literal, com ressalva e link**, sem decidir o conflito; **dois públicos
separados**, com cenários de origem mais cenários propostos marcados.

O PRD saiu com 444 linhas: 11 requisitos funcionais (o mínimo eram 8), 10 requisitos não
funcionais, 8 objetivos com origem rastreável e 6 métricas propostas, 8 itens de fora de
escopo decidido mais 2 na categoria não decidida, 11 riscos com probabilidade/impacto/
mitigação (o mínimo eram 2), 16 critérios de aceitação e 10 questões abertas de produto
com dono nomeado.

### Ajuste #7 — a marca F/D/P encolhe no PRD, e isso é a mesma lição do Ajuste #5

O Ajuste #5 registrou que a política da Fase 1 (ADRs não propõem schema) não sobrevive
ao FDD sem que o FDD deixe de ser FDD. Aqui a lição se repete espelhada: **a convenção
de três marcas do FDD não sobrevive ao PRD sem que o PRD deixe de ser PRD.**

Marcar toda afirmação estruturante funciona num documento cujo leitor é o dev que vai
implementar — ele precisa saber, linha a linha, o que é fonte e o que é escolha. Num
documento de produto o mesmo aparato produz exatamente o defeito que esta fase existe
para evitar: um texto que *parece* especificação técnica. A marca vira ruído e, pior,
sinaliza ao leitor que ele está lendo o documento errado.

A política nova: **a marca fica onde o risco de invenção se concentra**, não onde a
afirmação é estruturante. No PRD isso são duas seções — objetivos/métricas (onde a fonte
é literalmente vazia) e escopo (onde a diferença entre "adiado" e "nunca perguntado" é a
informação). No resto, a origem vem em prosa: ID do índice, falante, timestamp, ou link.

O mecanismo que substitui a marca no resto do documento é uma coluna: **"Onde está o
como"** na tabela de requisitos funcionais. Cada RF aponta para o ADR ou a seção do FDD
que carrega o mecanismo, em vez de resumi-lo. É o antídoto direto contra o risco desta
fase — o PRD não tem como roubar detalhe do FDD se, no lugar do detalhe, houver um link.

### Verificação da Fase 4 — o que só aparece quando se sobe de altura

As fases anteriores auditaram o design descendo: consequências negativas (Fase 1) e
schema concreto (Fase 3) forçam perguntas que prosa não força. Esta fase fez o oposto —
**subir de altura também é instrumento**, e revelou coisas que quatro documentos técnicos
tinham deixado passar justamente por operarem abaixo delas.

1. **O baseline do problema estava numa fala só, e nenhum dos quatro documentos a usou.**
   `[09:00]` Marcos: os clientes "ficam batendo no `GET /orders` de tempos em tempos… tá
   deixando a integração lenta e cara pra eles". RFC, FDD e os sete ADRs abrem no *pedido*
   dos três clientes (`TEC-29`) e vão direto para a arquitetura; nenhum registra o que o
   cliente faz hoje. É a origem da expressão "tempo real" de `RNF-01` e a única referência
   possível para a métrica principal de produto. Um pacote inteiro de design descreveu a
   solução sem descrever o estado que ela substitui.

2. **O prazo com risco de churn nunca foi acordado com quem o impôs.** Duas falas
   diferentes circulam — "fim do trimestre" (`[09:00]`) e "fim de novembro" (`[09:45]`) —
   e a linha `RES-04` do índice as funde, resumindo a segunda enquanto cita a primeira.
   Somado a `ABE-05` (Marcos prometeu confirmar com os três clientes e o resultado nunca
   voltou), o resultado é que **a data que justifica a pressão do projeto é nossa, não
   deles**. Nenhum documento técnico tinha motivo para notar isso.

3. **São dois públicos, e essa é a causa raiz de um risco que os ADRs trataram como
   detalhe de autorização.** `RES-07` e a nota de leitura (a) registram que o JWT é de
   usuário operador, não de cliente. Os documentos técnicos usaram isso para decidir onde
   o `customer_id` trafega. Em altura de produto o mesmo fato diz outra coisa: **o cliente
   final não tem autoatendimento** (não há login de cliente, e `ADI-03` tirou o painel do
   escopo), e é por isso — não por descuido — que `DEC-20` deixa qualquer operador
   autenticado operar webhook de qualquer cliente.

4. **A métrica principal de produto tem pré-requisito com prazo.** A redução do polling em
   `GET /orders` é o único indicador que mede o problema descrito no achado 1 — e o
   baseline não existe. A fala de `[09:00]` é qualitativa. **Medir o volume atual só é
   possível antes do go-live**; depois, a referência se perde para sempre. É o único
   achado do pacote inteiro cuja janela de ação fecha sozinha.

5. **Dois critérios de aceitação não são verificáveis por software, e ambos bloqueiam o
   go-live.** As 22 verificações do FDD são todas executáveis por teste. Em altura de
   produto aparecem duas que não são: a revisão de segurança de Sofia (`RES-06` a torna
   condição para subir) e a publicação do material que explica ao cliente que ele precisa
   deduplicar. Sem a segunda, a garantia at-least-once de `DEC-10` é uma promessa que a
   outra ponta não sabe que precisa cumprir.

6. **`ALT-08` tensiona o objetivo da feature, e ninguém tinha feito essa ligação.** O
   payload não leva os itens do pedido; quem precisa deles chama `GET /orders/:id` depois.
   Nos documentos técnicos isso é um trade-off de tamanho de mensagem. Em altura de
   produto é uma ressalva sobre o objetivo: **a feature reduz o polling, mas não elimina
   toda consulta** para o perfil de cliente que precisa da composição do pedido.

O achado 6 tem o mesmo formato do que o Ajuste #5 registrou como método na Fase 3: o fato
já estava escrito em três documentos, e o que faltava era ligá-lo a outro fato já escrito.
Nenhuma informação nova entrou — mudou a pergunta que se faz sobre a informação existente.
Vale como procedimento: **antes de propor, procurar qual par de fatos já registrados
ninguém confrontou.**

---

## Fase 5 — TRACKER

### Prompt customizado #7 — o instrumento que precisa se auditar antes de auditar

Objetivo: escrever `docs/TRACKER.md`, a referência cruzada entre cada item dos documentos
e sua origem. O risco desta fase não é alucinação — é **alucinação legitimada**. O tracker
é o instrumento que existe para pegar invenção; se ele próprio for gerado por
plausibilidade, ele carimba como verificado exatamente aquilo que deveria denunciar. E é
o documento mais fácil de gerar errado com aparência perfeita: uma tabela de 200 linhas
com timestamps bem formatados é indistinguível, à vista, de uma tabela correta.

Três fatos moldaram o prompt, e nenhum deles estava em arquivo:

1. **A tarefa é colher e conferir, não derivar.** As fases anteriores deixaram a origem
   inline nos documentos — o PRD tem coluna `Origem` com falante e timestamp em todas as
   tabelas de `RF` e `RNF`, RFC e FDD fecham com seção "Fontes", os ADRs citam IDs do
   índice no corpo. Uma IA a quem se pede "monte a rastreabilidade" reconstrói o
   mapeamento de memória em vez de colher o que já está escrito e checar na transcrição.
2. **O índice não é autoridade.** É artefato derivado da Fase 0 e já tinha um defeito
   conhecido — `RES-04` resume a fala de `[09:45]` citando `[09:00]`, achado 2 da Fase 4.
3. **Os alvos percentuais criam incentivo perverso.** ≥70% das linhas com
   `Fonte = TRANSCRICAO` num pacote que contém, deliberadamente, 21 itens de ledger no
   FDD, 6 métricas propostas no PRD e 3 desambiguações feitas fora da reunião. O caminho
   de menor resistência para bater a meta é dar timestamp a eles. O prompt nomeia cada um.

```
Agora o TRACKER — e este é o documento em que a alucinação é mais cara do pacote
inteiro, porque ele é justamente o instrumento que existe para pegar alucinação.
Um tracker inventado não erra sozinho: ele carimba os outros cinco documentos
como verificados.

Preciso escrever docs/TRACKER.md (hoje é stub de duas linhas).

O TRACKER é transversal: responde "de onde veio cada coisa". Ele NÃO resume
documento, NÃO redecide nada e NÃO acrescenta uma única afirmação nova. Toda
linha dele é uma afirmação que já está escrita em outro arquivo do pacote.

A tarefa é COLHER E CONFERIR, não derivar. Os documentos já carregam a origem
inline: o PRD tem coluna "Origem" com falante e timestamp em todas as tabelas de
RF e RNF; RFC e FDD têm seção "Fontes" no fim listando IDs do índice e caminhos
reais; os ADRs citam IDs do índice no corpo. Colha o que está escrito, depois
confira contra a fonte primária. É proibido reconstruir a origem de cabeça: se
uma afirmação do documento não trouxer origem, isso é um achado, não um convite
para você inferir uma.

Fontes (leia todas antes de começar):
- docs/PRD.md, docs/RFC.md, docs/FDD.md — os itens a rastrear
- docs/adrs/ADR-001 a ADR-007 (todos os sete)
- .scratch/fontes/transcricao-index.md — 112 itens (20 DEC, 11 RF, 10 RNF,
  7 RES, 8 ALT, 6 ADI, 5 ABE, 16 COD, 29 TEC) com citação literal, falante e
  timestamp
- TRANSCRICAO.md — a fonte primária, e a ÚNICA autoridade sobre timestamp e
  falante
- CONTEXT.md e o repositório real — a autoridade sobre caminho de arquivo

Restrições: entrega puramente documental, é proibido alterar src/, prisma/,
tests/, package.json e TRANSCRICAO.md. Documento em pt-BR.

Formato obrigatório da tabela, exatamente estas seis colunas:
ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização

- ID: identificador único do item (ex.: PRD-FR-01, RFC-ALT-02,
  FDD-CONTRATO-03, ADR-002)
- Documento: o arquivo onde o item aparece
- Tipo: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição,
  Trade-off, entre outros
- Conteúdo: uma linha
- Fonte: TRANSCRICAO ou CODIGO
- Localização: para TRANSCRICAO, timestamp + falante no formato [hh:mm] Nome;
  para CODIGO, caminho de arquivo real

Alvos numéricos do critério de aceite: ≥80% dos itens identificáveis dos
documentos com linha correspondente; ≥70% das linhas com Fonte = TRANSCRICAO e
timestamp em formato válido; ≥5 linhas com Fonte = CODIGO e caminho real.

Regras de verificação, não negociáveis:

1. Todo timestamp que você escrever tem que ser confirmado em TRANSCRICAO.md
   localizando a fala que contém a citação. Não copie timestamp do índice sem
   conferir — o índice é artefato derivado e já tem defeito conhecido (ver
   contexto abaixo).
2. Timestamp em faixa ([09:17-18]) é PROIBIDO. Foi exatamente o que uma
   varredura anterior produziu quando o item resumia uma troca que atravessava
   dois minutos. Se o item resume uma troca, use o timestamp da fala que contém
   a citação literal que sustenta o item.
3. Falante inventado é proibido. Se a afirmação nasce de uma troca entre duas
   pessoas, escolha a fala que a sustenta e nomeie quem a disse.
4. Todo caminho de arquivo em coluna Localização tem que existir no repositório
   — verifique. Se algum documento do pacote citar caminho inexistente ou
   número de linha que não bate mais, isso é achado.

Contexto que você precisa saber e não está nos arquivos:

- O ÍNDICE NÃO É AUTORIDADE. Ele tem pelo menos um defeito já identificado:
  a linha RES-04 resume a fala de [09:45] Marcos ("fim de novembro") mas cita
  [09:00], onde ele disse "fim do trimestre". Trate isso como amostra de uma
  classe, não como caso isolado: confira contra TRANSCRICAO.md.

- O PACOTE TEM ITENS DELIBERADAMENTE SEM ORIGEM, E ELES NÃO PODEM GANHAR UMA.
  Os alvos de 70% e 80% criam incentivo direto para dar timestamp a coisa que
  não tem fala nenhuma. Os itens sem origem são: os 21 do ledger "Premissas
  assumidas e derivações sem origem" do FDD; as 6 métricas de produto do PRD
  §4.2 e os cenários adicionais do §3.3; o caminho do endpoint de rotação de
  secret (RF-07), que é proposta da RFC. Nenhum deles recebe [hh:mm].

- TRÊS ITENS FORAM DESAMBIGUADOS FORA DA REUNIÃO e a origem deles é o processo,
  não a fala: ABE-02 (arquivo de lógica webhook.worker.ts com src/worker.ts
  como entry point), ABE-03 (customer_id no body do POST e na query do GET) e
  TEC-24 ("5 tentativas" = 5 retentativas além do envio inicial, 6 chamadas
  HTTP). Eles têm fala de origem para a PERGUNTA e nenhuma para a RESPOSTA.

- TRÊS CONFLITOS ESTÃO ESCALADOS E NÃO DECIDIDOS POR NINGUÉM: o alvo de
  latência de RNF-01, a quebra de ordem sob retry e o errorMiddleware não
  alcançar o worker. Todos têm origem rastreável; nenhum tem resolução. Se
  alguma linha do tracker descrever qualquer um deles como decidido, é defeito
  do tracker ou do documento que ele está rastreando — reporte, não concilie.

- SOFIA NUNCA RETIROU A OBJEÇÃO A DEC-10 (nota de leitura (c)-4 do índice).
  Se aparecer no pacote como consenso, é achado.

- O TRACKER É O ÚLTIMO PORTÃO ANTES DO README. É a única passada que lê os
  cinco documentos lado a lado. Divergência entre documentos, número que mudou
  de um doc para outro, item que sumiu — tudo isso aparece agora ou não aparece
  mais.

Comece me perguntando:

(a) Itens sem origem. O formato obrigatório só admite Fonte = TRANSCRICAO ou
CODIGO, e o pacote tem ~30 itens estruturantes que não são nem um nem outro (o
ledger de 21 do FDD, as 6 métricas do PRD, os cenários propostos, o endpoint de
rotação). Eles ficam fora da tabela principal, numa segunda tabela "itens sem
origem" — mantendo o percentual honesto, mas deixando o tracker sem cobrir
justamente o que mais precisa de vigilância? Entram na tabela principal com um
terceiro valor de Fonte (PROPOSTA), quebrando o formato exigido? Ou entram com
Fonte apontando o documento que assumiu a autoria? Recomende uma e diga o que se
perde nas outras duas.

(b) ID e denominador. Os IDs devem reaproveitar os identificadores que os
documentos já têm (PRD-RF-07 sendo literalmente o RF-07 do PRD, FDD-P-05 sendo
o item 5 do ledger), o que torna o tracker conferível linha a linha contra o
documento? Ou devem seguir a numeração sequencial por tipo que o enunciado
exemplifica (PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03), que é uniforme mas não
junta com nada? E: qual é a unidade que conta como "item identificável"? Sem
declarar a regra de contagem e o número absoluto, o alvo de 80% é auto-atestado
e não significa nada — o tracker deve publicar seu próprio denominador?

(c) Varredura reversa e o que fazer com defeito encontrado. O índice tem 112
itens e boa parte não entrou em documento nenhum de propósito (ALT descartadas,
ADI adiados, TEC secundários). O tracker deve trazer também a direção inversa —
uma tabela de cobertura da fonte, mostrando o que da reunião não chegou a
nenhum documento e se a omissão foi deliberada? É o que transforma o tracker de
"lista do que escrevi" em "prova de que não perdi nada". E quando a varredura
encontrar divergência real (timestamp que não bate, caminho inexistente, número
que mudou entre documentos), você corrige o documento na hora ou só reporta e
espera minha decisão? Minha inclinação é só reportar — corrigir PRD ou FDD
dentro de uma passada de tracker é como um teste consertar o código que ele
está testando.
```

**Resultado:** as três perguntas foram respondidas pela recomendação embutida em cada
uma — tabela dedicada para os itens sem origem, IDs herdados dos documentos com
denominador publicado, varredura reversa com achados apenas reportados. Diferente das
Fases 1 a 4, esta não teve rodada de perguntas: as opções estavam colocadas com
recomendação explícita e a execução seguiu direto.

O tracker saiu com **234 linhas na tabela principal** (208 com `Fonte = TRANSCRICAO`,
88,9%; 26 com `Fonte = CODIGO`, cobrindo 14 arquivos), **41 itens na tabela de sem
origem**, cobertura de **100% dos 275 itens próprios** (83% contando os 56 restatements),
**zero órfãos** na varredura reversa dos 112 itens do índice, e **seis achados** — todos
reportados, nenhum corrigido.

### Ajuste #8 — a regra que faltava era sobre o próprio material de trabalho

As fases anteriores produziram regras sobre o que **escrever** (Ajuste #4: ADR não propõe
schema; Ajuste #5: onde a fonte é omissa o FDD propõe e assina; Ajuste #7: a marca fica
onde o risco se concentra). Esta produziu a primeira regra sobre o que **ler**:

> **Artefato derivado não é autoridade sobre a fonte que ele deriva.**

O índice da transcrição foi construído na Fase 0 e usado como base factual das Fases 1 a
4 sem que ninguém voltasse à transcrição para checá-lo. Ele é excelente como roteiro — os
112 itens, as citações literais, as notas de leitura sobre contradições — e é ele que
tornou as quatro fases possíveis. Mas **os três achados de timestamp desta fase nascem
todos nele, e nenhum nos documentos**: `[09:29]` no lugar de `[09:28]` em cinco linhas,
`[09:47]` no lugar de `[09:49]` em `ABE-05`, e a fusão de duas datas em `RES-04`. Em
todos, a citação literal está certa e o carimbo de origem escorregou.

A consequência prática: conferir 61 pares `[hh:mm] Nome` contra `TRANSCRICAO.md` custa
uma varredura, e é a única razão pela qual as linhas do tracker estão certas onde as do
índice não estão. O Ajuste #2 já tinha ensinado metade disso — que a IA inventa a variante
plausível quando o dado real não cabe no formato. A outra metade é que **ela também herda
o erro plausível quando o dado errado cabe perfeitamente**.

### Verificação da Fase 5 — seis achados, nenhum corrigido

O tracker é a única passada que lê os cinco documentos lado a lado, e a política escolhida
foi reportar sem editar: uma auditoria que conserta o que está auditando deixa de ser
auditoria.

1. **`[09:29]` no lugar de `[09:28]` em cinco itens do índice** (`COD-04`, `COD-05`,
   `COD-06`, `COD-15`, `TEC-16`). A fala de Bruno sobre `AppError` e os códigos `WEBHOOK_*`
   é `[09:28]`; `[09:29]` é a fala seguinte, sobre Pino e o middleware — que é a origem
   correta de `COD-07` e `COD-08`. Propagou para o FDD §9.6 e para o ADR-006. Efeito
   colateral fino: a ressalva 2 do FDD argumenta que `WEBHOOK_SECRET_REQUIRED` foi nomeado
   "dois minutos antes" de `TEC-14` torná-lo inalcançável — com o timestamp certo, são três.

2. **`ABE-05` cita `[09:47]` para uma frase de `[09:49]`.** "Tá bom. Eu atualizo os clientes
   hoje à tarde" é `[09:49]`; em `[09:47]` Marcos diz outra coisa que sustenta o mesmo fato.
   Propagou para o PRD §2.2 e para a dependência `D-04`.

3. **`RES-04` funde duas datas** — já detectado na Fase 4, agora confirmado contra a
   transcrição. É o único dos três em que o documento consumidor (o PRD) **já tinha
   separado as duas datas por conta própria**; a varredura confirma a leitura do PRD e
   condena a linha do índice.

4. **As seções "Fontes" declaram faixas contíguas que o corpo não cita.** A busca literal
   por identificador encontra 9 IDs no PRD, 10 na RFC e 14 no FDD que estão declarados na
   bibliografia e não aparecem no texto — "`COD-01 a COD-16`" é mais barato de escrever do
   que a enumeração real. Na maioria dos casos o conteúdo está lá sem o rótulo: o FDD trata
   de HTTPS obrigatório e de ordem sob single-worker longamente, só não usa `RNF-07` e
   `RNF-08`. É over-declaração de bibliografia, não invenção no corpo — mas é uma afirmação
   verificável que não se sustenta.

5. **Um timestamp em faixa sobreviveu no ADR-002** (`[09:09]`–`[09:10]`), escrito na Fase 1,
   antes de o Ajuste #2 virar regra de prompt. As duas falas existem separadamente. É a
   prova de que o Ajuste #2 corrigiu o índice e não voltou para checar o que já tinha sido
   escrito com ele.

6. **Nenhum caminho inexistente citado como existente, e nenhum número de linha errado.**
   Os 31 caminhos citados foram verificados: 27 existem, e os 4 que não existem
   (`src/worker.ts`, `webhook.worker.ts`, os dois arquivos de teste) são nomeados pelos
   próprios documentos como arquivos a criar. Os 17 números de linha do FDD conferem um a
   um, incluindo o mais específico do pacote — a chamada a `publishWebhookEvent` "entre a
   linha 167 e a linha 169", que é exatamente onde o bloco de `tx.orderStatusHistory.create`
   fecha. As três afirmações negativas mais fáceis de errar também conferem: `package.json`
   não tem script `worker`, `redactPaths` não cobre `secret`, `buildApiRouter` não monta
   `/admin`.

O achado 5 tem a mesma forma do Ajuste #8 e vale como procedimento próprio: **correção
aplicada à fonte não se propaga sozinha para o que já foi derivado dela.** Quando uma
regra nova é criada no meio do processo, o que foi escrito antes dela continua sem a
regra — e só uma varredura posterior encontra.

---

<!-- próxima fase: README do processo -->

---

# Fase 7 — Correção dos achados e destino dos itens propostos

Fase não planejada, aberta depois de uma leitura crítica externa do pacote fechado. Ela
derrubou as duas posturas que a Fase 5 tinha adotado por convicção metodológica.

## O que a leitura externa apontou

1. **Os erros conhecidos permaneceram na entrega.** O tracker registrava cinco achados e não
   corrigia nenhum. "Registrar o defeito é excelente prática de auditoria, mas não satisfaz o
   critério final de que nenhuma informação contradiga ou represente incorretamente a fonte."
2. **Os 41 itens sem origem continuavam sem origem.** Separá-los com honestidade não os torna
   rastreáveis. "Propostas técnicas são necessárias para tornar um FDD acionável, mas deveriam
   aparecer como questões sujeitas à aprovação ou ser justificadas como derivações explícitas.
   A marca **P** evita alucinação silenciosa, porém não transforma esses itens em informação
   originada na fonte."

Os dois pontos atacam o mesmo erro de raciocínio, em dois lugares: **confundir declarar um
problema com resolvê-lo**. O argumento "auditoria não edita o que audita" é verdadeiro sobre
*como auditar* e irrelevante sobre *o que entregar* — a entrega não é a auditoria, é o pacote,
e o critério fala do pacote.

## De onde o defeito veio: duas linhas do prompt #7

Antes do prompt novo, vale nomear o que no prompt anterior produziu as duas falhas. Não foi
falta de rigor da sessão — foi o prompt que pediu o resultado errado, nos dois casos.

1. **Pergunta (c), última frase:** *"E quando a varredura encontrar divergência real, você
   corrige o documento na hora ou só reporta e espera minha decisão? Minha inclinação é só
   reportar — corrigir PRD ou FDD dentro de uma passada de tracker é como um teste consertar o
   código que ele está testando."*
   A pergunta vinha com a resposta anexada. A analogia é boa e a conclusão não segue dela: um
   teste não conserta o código, mas o time **conserta o código antes de entregar**. Faltou a
   fase seguinte, e nada no prompt a exigia.

2. **Bloco de contexto, segundo item:** *"O PACOTE TEM ITENS DELIBERADAMENTE SEM ORIGEM, E ELES
   NÃO PODEM GANHAR UMA."*
   A regra está certa e é **incompleta**. Ela proíbe o destino errado (inventar `[hh:mm]`) e
   não exige nenhum destino certo. O resultado foi previsível: a sessão fez exatamente o que
   foi pedido — separou os 41 numa tabela própria — e parou ali, porque ali era o fim da
   instrução.

A lição de prompt, então, tem duas partes. **Não anexe sua inclinação à pergunta que você está
fazendo** — a sessão a devolve validada, e você troca uma decisão por um eco. E **toda proibição
precisa de uma obrigação ao lado**: "não faça X" sem "faça Y" produz um vazio, e o vazio passa
despercebido porque nada no texto o denuncia.

## O prompt da Fase 7

Escrito para atacar as duas omissões de frente. É o prompt que, colocado na Fase 5, teria
evitado esta fase inteira.

```
Passada de correção sobre o pacote fechado. Duas coisas estão erradas nele, e as
duas são do mesmo tipo: eu confundi DECLARAR um problema com RESOLVÊ-LO.

Problema 1 — os defeitos conhecidos ficaram na entrega.
O TRACKER §5 lista cinco achados (A-01 a A-05) e não corrige nenhum, com o
argumento de que "auditoria não edita o que audita". Esse argumento é verdadeiro
sobre COMO AUDITAR e irrelevante sobre O QUE ENTREGAR. O critério de aceite não
diz "nenhuma divergência não declarada"; diz que nenhuma informação registrada
pode contradizer ou representar incorretamente a fonte. Um timestamp que aponta
para a fala errada representa a fonte incorretamente esteja ou não confessado
três seções abaixo.

Problema 2 — os 41 itens "sem origem" continuam sem origem.
Separá-los com honestidade não os torna rastreáveis, e a regra do desafio é que
TODA informação registrada seja rastreável. A marca P impede alucinação
silenciosa e não transforma proposta em informação originada na fonte.

Tarefa 1: corrigir A-01 a A-05, e o rastro tem que sobreviver à correção.
- Corrija na origem e em TODA propagação. Para cada achado, localize antes as
  ocorrências: as do próprio defeito e as legítimas que compartilham o mesmo
  valor. A-01 é [09:29] -> [09:28] em cinco itens, mas COD-07 e COD-08 estão
  corretos em [09:29] e NÃO podem ser tocados. A-02 é [09:47] -> [09:49] em
  ABE-05, mas DEC-13 está correto em [09:47]. Corrigir demais é o mesmo defeito
  na direção oposta.
- TRANSCRICAO.md não se toca. A fonte não se corrige.
- Reescreva cada achado do TRACKER §5 preservando o QUE ESTAVA, o QUE FICOU e os
  ARQUIVOS ALTERADOS. Apagar o achado junto com o defeito destrói a evidência de
  que a auditoria funcionou.
- A-04 foi levantado sobre PRD, RFC e FDD. Rode a mesma varredura nos 7 ADRs
  antes de declarar o achado fechado, e nos DOIS sentidos: ID declarado na seção
  Fontes e não citado no corpo, E ID citado no corpo e ausente da seção Fontes.
  O segundo caso é o mais grave, porque some com a origem em vez de inflá-la.
- Corrigir A-03 direito significa SEPARAR a linha do índice em duas, o que muda
  a contagem de itens da reunião. Se mudar, varra todo lugar do repositório que
  cita a contagem antiga.
- Ao fim, reverifique do zero sobre os arquivos corrigidos e publique o
  resultado da reverificação, não a intenção da correção.

Tarefa 2: dar destino aos 41 itens, não apenas rótulo.
Cada um passa a ser exatamente uma de duas coisas, e nenhum fica solto:

(D) DERIVAÇÃO EXPLÍCITA — consequência necessária de um item que TEM origem.
Escreva a CADEIA: item de fonte (com ID e timestamp, ou caminho de arquivo) ->
passo lógico -> resultado. Uma derivação com a cadeia escrita é rastreável: a
origem é indireta e conferível, e se a cadeia estiver errada o item cai junto.
Cadeia de uma linha vaga ("decorre da arquitetura") não vale — se você não
consegue nomear o item de origem, não é derivação.

(A) PROPOSTA PENDENTE DE APROVAÇÃO — escolha do documento, e nada mais. Entra
como PERGUNTA ENDEREÇADA, com: aprovador nomeado, fórum onde a decisão cabe,
prazo, e o que se perde se for rejeitada. E o documento tem que dizer, no ponto
em que a proposta aparece, que enquanto não aprovada ela NÃO É requisito, meta
nem compromisso.

Regra de corte, e ela é conservadora: para cada item, pergunte "se um
participante da reunião lesse a cadeia, ele diria 'isso decorre do que eu
decidi' ou 'isso é escolha de vocês'?". Só a primeira resposta vira D. Na
dúvida, vai para A. Se a tabela de derivações sair maior que a de aprovações,
você foi generoso demais — reveja.

Restrição sobre os fóruns: NÃO invente instância de aprovação. Use apenas as que
a reunião criou, e nomeie a fala que as criou. Se um item não couber em nenhuma
delas, isso é o achado, não um convite para inventar um comitê.

Comece me perguntando:

(a) Onde mora a lista reclassificada. O FDD tem o ledger, o TRACKER tem a seção
3, e os itens do PRD estão espalhados em três seções. Uma lista canônica com as
outras apontando para ela, ou cada documento carrega a sua e o TRACKER
consolida? Recomende uma e diga o que se duplica na outra.

(b) O que acontece com a numeração existente. FDD-P-01 a FDD-P-21 estão citados
em outros documentos. A reclassificação renumera (e obriga a varrer as
referências) ou preserva os IDs e acrescenta a classificação como coluna?

(c) O README e o registro de processo afirmam hoje, com todas as letras, que
nenhum achado foi corrigido — é uma seção inteira defendendo a escolha. Reescrevo
como se a escolha nunca tivesse existido, ou registro a mudança de posição com o
argumento que a derrubou? Minha inclinação é a segunda, e quero que você discorde
se achar que ela infla a narrativa.
```

**Decisões que as três perguntas fecharam:** (a) lista canônica por documento, com o TRACKER §3
consolidando as duas categorias e o FDD/PRD carregando a sua parte no ponto de uso — nenhuma
proposta fica visível só no consolidado; (b) **IDs preservados** (`FDD-P-01` continua
`FDD-P-01`), com a classificação entrando como estrutura de tabela, porque renumerar 41 itens
citados em cinco documentos troca um defeito de rastreabilidade por outro; (c) registrar a
mudança de posição com o argumento que a derrubou — a seção "Seis achados encontrados e nenhum
corrigido" do README virou "Seis achados encontrados — e a Fase 5 parou aí", seguida da nova
"Fase 7 — o que eu errei ao só reportar".

## O que a fase fez

### 1. Correção dos cinco achados, com o rastro preservado

| Achado | Correção | Arquivos |
|---|---|---|
| A-01 | `[09:29]` → `[09:28]` em `COD-04`, `COD-05`, `COD-06`, `COD-15`, `TEC-16`; FDD §9.6 e ressalva 2 ("dois minutos" → "três minutos"); ADR-006 em duas citações | índice, `FDD.md`, `ADR-006` |
| A-02 | `ABE-05` e nota (b)-1 → `[09:49]`, com a fala de `[09:47]` nomeada ao lado; PRD §2.2 e `D-04` idem | índice, `PRD.md` |
| A-03 | `RES-04` separado em `RES-04` (pressão comercial, `[09:00]`) e `RES-08` (data-alvo, `[09:45]`) | índice, `PRD.md` |
| A-04 | Enumeração literal de IDs nas seções "Fontes" — **e a varredura estendida aos 7 ADRs**, que o achado original não cobria | `PRD.md`, `RFC.md`, `FDD.md`, 7 ADRs |
| A-05 | Faixa `[09:09]`–`[09:10]` separada nas duas falas, com o papel de cada uma | `ADR-002` |

`TRANSCRICAO.md` não foi tocado. A seção 5 do tracker passou a registrar, por achado, **o que
estava, o que ficou e os arquivos alterados** — o rastro da auditoria sobrevive à correção.

**Duas coisas que só apareceram ao corrigir:**

- **A-04 era maior do que o achado dizia.** Aplicada aos ADRs, a varredura encontrou 8 IDs
  declarados e não citados **e 10 IDs citados no corpo e ausentes da lista de fontes**. O
  segundo caso é pior que o primeiro: infla-declaração é bibliografia frouxa, sub-declaração
  **some com a origem**. A correção deixou de ser edição manual e virou geração da lista a
  partir do corpo, com verificação nos dois sentidos nos dez documentos.
- **Corrigir A-03 direito mexe num denominador publicado.** Separar a linha significou 113
  itens no índice em vez de 112 — número que aparece na varredura reversa do tracker, no
  README e na descrição do artefato. Correção de rastreabilidade que muda contagem obriga a
  varrer todo lugar que cita a contagem.

### 2. Destino dos 41 itens acrescentados

A regra nova: **todo item que não é citação de fala nem leitura de arquivo pertence a uma de
duas categorias, e nenhum fica solto.**

- **Derivação explícita (14).** Consequência necessária de um item que tem origem, com a
  **cadeia escrita**: item de fonte → passo lógico → resultado. Uma derivação com cadeia é
  rastreável — a origem é indireta e conferível, e se a cadeia estiver errada o item cai.
- **Proposta pendente de aprovação (27).** Escolha do documento, e nada mais. Entra como
  **pergunta endereçada a pessoa nomeada, em fórum que a própria reunião criou** — a revisão
  de segurança de Sofia (`RNF-10` `[09:46]`, bloqueante por `RES-06`) ou a sessão de revisão
  de design que Larissa assumiu abrir (`[09:50]`). Enquanto não aprovada, **não é requisito**.

O critério de corte entre as duas foi deliberadamente conservador: na dúvida, o item foi para
"aprovação pendente". A pergunta aplicada a cada um foi *"se um participante da reunião lesse
a cadeia, ele diria 'isso decorre do que eu decidi' ou 'isso é escolha de vocês'?"*. Só a
primeira resposta vira derivação — e é por isso que a tabela de aprovações é quase o dobro da
de derivações.

Onde os itens foram parar: `FDD.md` (registro final, reescrito), `PRD.md` §13.1 (nova, 16
propostas com aprovador e prazo), `RFC.md` (RF-07 com a cadeia do caminho, e as 3
desambiguações como confirmações 13 a 15), `TRACKER.md` §3 (reescrita nas duas categorias).

## A lição

**Um pacote de documentação não é avaliado pela qualidade da sua consciência sobre si mesmo.**
Saber onde está o defeito é pré-requisito de corrigi-lo, não substituto. O tracker fez o
trabalho difícil — encontrar; a Fase 5 parou no passo fácil. E a mesma forma vale para a marca
**P**: separar o que não tem origem é o começo do trabalho de rastreabilidade, não o fim dele.


