# Da Reunião ao Documento — processo de produção

Este repositório contém um pacote de design docs — PRD, RFC, FDD, 7 ADRs e um tracker de
rastreabilidade — produzido a partir de duas fontes: a transcrição de uma reunião técnica de
55 minutos (`TRANSCRICAO.md`) e o código de um Order Management System em produção.

Este README documenta **como o pacote foi produzido**. O enunciado original do desafio está
preservado em [`docs/DESAFIO.md`](docs/DESAFIO.md).

---

## Sobre o desafio

O ponto de partida é uma feature já decidida e nunca registrada. Uma empresa que opera um OMS
em produção vai construir um Sistema de Webhooks de Notificação de Pedidos; tech lead, PM,
engenheiros e segurança fecharam a arquitetura numa call, e o único registro que sobrou foi a
gravação. A tarefa é transformar essa call em documentação técnica acionável o suficiente para
o time começar a implementar — usando IA como ferramenta principal de produção, no papel de
maestro: definir o que precisa existir, formular os prompts, revisar criticamente e corrigir.

O que torna o desafio difícil não é gerar documento — LLM gera documento com fluência. É que
**toda afirmação precisa ser rastreável** à transcrição ou ao código, e a transcrição é
deliberadamente cheia de armadilhas: coisas ditas e descartadas, coisas adiadas para uma v2,
coisas levantadas e nunca decididas, e detalhes técnicos soltos que não são requisito. Os três
destinos são diferentes nos documentos finais — descartado vira alternativa no RFC, adiado vira
fora de escopo no PRD, em aberto vira questão aberta — e uma leitura descuidada funde os três.
Somado a isso, o pacote tem cinco documentos que operam em **alturas diferentes**: conteúdo
duplicado entre eles é sinal de que algo está no lugar errado. O trabalho real, portanto, é
menos escrever e mais **conter modos de falha específicos** — e cada fase tem o seu.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code (Opus)** | Ferramenta única de produção. Leu o código e a transcrição direto do repositório, escreveu todos os documentos e fez as varreduras de verificação. |
| **Plugin `mattpocock-skills`** | Três skills usadas: `/grill-with-docs` e `/grilling`, que interrogam antes de produzir, e `/domain-modeling`, que fixa o formato MADR dos ADRs. O resto do plugin foi descartado — ver Workflow. |
| **Sub-agentes de exploração** | Usados na Fase 0 para as duas varreduras de base (transcrição e código), cada uma isolada em seu próprio contexto para varrer o arquivo inteiro sem competir por espaço com o resto da sessão. |
| **`AGENTS.md` + `docs/agents/`** | Configuração de skills do repositório: declara o layout de documentação (contexto único, `CONTEXT.md` na raiz, ADRs em `docs/adrs/` no formato MADR) para que as skills escrevam no lugar certo e no formato certo. |

Quatro itens, e nenhum a mais. Não usei ChatGPT, Gemini, Cursor ou Copilot em nenhum momento —
o diferencial da entrega não está em variedade de ferramenta, está no método de contenção
descrito abaixo.

---

## Workflow adotado

### A decisão de fluxo: o que foi descartado

O plugin `mattpocock-skills` oferece um fluxo de engenharia completo — `/to-spec` →
`/to-tickets` → `/implement` → `/tdd`. Ele foi **descartado inteiro**: existe para shippar
código, e esta entrega é puramente documental (o enunciado proíbe tocar em `src/`, `prisma/`,
`tests/`). Aproveitei só a cabeça do fluxo, `/grill-with-docs` + `/domain-modeling`, que produz
ADRs em formato MADR — exatamente o artefato central exigido.

Foi a primeira decisão do trabalho e ela define o resto: em vez de um pipeline que gera e
depois pede revisão, um formato que **interroga antes de produzir**.

### A estratégia central: base de fatos em disco antes de qualquer documento

Antes de escrever uma linha de documento, duas varreduras gravaram em disco a base factual:

- **`.scratch/fontes/transcricao-index.md`** — 113 itens da reunião, cada um com citação
  literal, falante e timestamp, classificados em nove baldes (20 decisões fechadas, 11
  requisitos funcionais, 10 não funcionais, 8 restrições, 8 alternativas descartadas, 6 itens
  adiados, 5 questões em aberto, 16 ganchos com o código, 29 detalhes técnicos). Foram 112 até
  a Fase 7, que separou em duas uma linha que fundia duas falas de datas diferentes.
- **`CONTEXT.md`** — o mapa do código existente, com a máquina de estados do pedido, a
  assinatura real de `changeStatus`, a hierarquia de erros, e uma seção de pontos de extensão
  concretos onde a feature vai encostar.

Todos os documentos posteriores leem essa base — **não a transcrição bruta nem o código
inteiro**. A consequência prática apareceu no fim: o TRACKER deixou de ser trabalho de
arqueologia e virou uma junção contra um índice pronto, e qualquer linha que não fechasse com o
índice era, por construção, alucinação.

### As sete fases

A ordem de produção segue a lógica de que decisão vem antes de proposta, e proposta antes de
detalhe. **Não é a ordem de leitura** — ver [Como navegar a entrega](#como-navegar-a-entrega).

| Fase | Produto | Risco que a fase precisava conter |
| --- | --- | --- |
| **0** | Base de fatos (índice + `CONTEXT.md`) | Documentar sobre memória em vez de sobre leitura |
| **1** | 7 ADRs | **Alucinação** — afirmação sem origem |
| **2** | RFC | **Redecisão** — sessão nova reabre o que já virou ADR |
| **3** | FDD | **Invenção plausível** — o único documento em que a alucinação sai bem escrita |
| **4** | PRD | **Roubo de altura** — resumir o FDD e entregar documento técnico com capa de produto |
| **5** | TRACKER | **Alucinação legitimada** — o instrumento que carimba os outros cinco como verificados |
| **6** | Este README | **Inflar a jornada** — narrar um processo mais limpo que o registro |
| **7** | Correção dos achados e reclassificação dos itens propostos | **Defeito conhecido que fica** — auditar bem e entregar errado assim mesmo |

A Fase 7 não estava no plano, e a razão de ela existir é a lição mais cara do processo: está
descrita em [Fase 7 — o que eu errei ao só reportar](#fase-7--o-que-eu-errei-ao-só-reportar).

Cada fase teve um prompt escrito para o seu risco, não um pedido genérico de documento. Os sete
prompts, na íntegra, e o registro contemporâneo de cada ajuste estão em
[`.scratch/processo/prompts-e-iteracoes.md`](.scratch/processo/prompts-e-iteracoes.md).

### Como a interação foi organizada

Uma sessão por fase, começando fria. Isso é deliberado — sessão longa acumula contexto e começa
a confundir o que leu com o que escreveu — mas cobra um preço: **as decisões tomadas fora dos
arquivos não sobrevivem à troca de sessão**. Três ambiguidades da reunião foram resolvidas na
Fase 1, e três conflitos de design emergiram durante a escrita dos ADRs; nada disso está na
transcrição. Por isso os prompts das Fases 2 em diante carregam esses itens **inline**, no
corpo do prompt: uma sessão que não os conhecesse ou reabriria o que já estava fechado, ou
apresentaria os conflitos como descoberta nova.

O padrão de abertura de cada fase é o mesmo: o prompt termina com **"comece me perguntando"**,
listando as bifurcações reais. A sessão pergunta, eu decido, e só então ela escreve. Foi assim
que 11 decisões foram fechadas antes do primeiro ADR e 17 antes da primeira linha do FDD.

---

## Prompts customizados

Quatro dos oito, escolhidos por contraste. Juntos eles mostram uma progressão que nenhum par
mostra: o prompt deixa de pedir *documento* e passa a pedir *contenção de um modo de falha
específico* — e o modo de falha muda a cada fase. O quarto quebra o padrão de propósito: é o
único escrito **contra o resultado de um prompt anterior**, e por isso é o que mais ensina.

### #1 — Extração indexada da transcrição (Fase 0)

O ponto crítico é forçar a separação entre **DESCARTADO**, **ADIADO** e **EM ABERTO**: três
destinos diferentes nos documentos finais que uma leitura descuidada funde num só. E, quando a
fonte é ambígua, o prompt não manda escolher — manda **marcar a ambiguidade**.

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

As "Notas de leitura" pareciam um extra na hora de escrever o prompt. Foram o item de maior
retorno do pacote inteiro: é lá que está registrado que **Sofia nunca retirou a objeção à
entrega at-least-once** — Larissa fechou como decisão, e a objeção nunca foi retomada. Sem essa
nota, o pacote registraria um consenso que não houve.

### #3 — Grilling até os ADRs (Fase 1)

Este é o prompt mais curto e o que mais muda o resultado. Ele ataca dois modos de falha ao mesmo
tempo. O primeiro é alucinação, contido pela regra de rastreabilidade dupla — índice ou caminho
de arquivo real, nada mais. O segundo é mais sutil: uma IA que encontra ambiguidade na fonte
tende a **escolher silenciosamente** a leitura mais plausível e seguir escrevendo, e o resultado
passa despercebido porque parece decidido.

Por isso o prompt não pede "resolva as ambiguidades". Ele **nomeia as três** e exige que a
sessão *comece* perguntando sobre elas.

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

Resultado: 7 ADRs cobrindo 6 das 6 decisões-alvo, com duas rodadas de perguntas e 11 decisões
fechadas antes de qualquer arquivo ser escrito.

### #7 — O TRACKER, o instrumento que precisa se auditar antes de auditar (Fase 5)

Aqui o risco não é alucinação, é **alucinação legitimada**. O tracker existe para pegar
invenção; se ele próprio for gerado por plausibilidade, carimba como verificado exatamente
aquilo que deveria denunciar. E é o documento mais fácil de gerar errado com aparência
perfeita: uma tabela de 200 linhas com timestamps bem formatados é indistinguível, à vista, de
uma tabela correta.

O prompt está reproduzido com corte (`[...]`) na parte que apenas repete o formato de tabela
exigido pelo enunciado.

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

[... formato obrigatório das seis colunas e alvos percentuais do critério de
aceite: ≥80% dos itens com linha correspondente, ≥70% com Fonte = TRANSCRICAO e
timestamp válido, ≥5 com Fonte = CODIGO ...]

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
exemplifica? E: qual é a unidade que conta como "item identificável"? Sem
declarar a regra de contagem e o número absoluto, o alvo de 80% é auto-atestado
e não significa nada — o tracker deve publicar seu próprio denominador?

(c) Varredura reversa e o que fazer com defeito encontrado. O índice tem 112
itens e boa parte não entrou em documento nenhum de propósito (ALT descartadas,
ADI adiados, TEC secundários). O tracker deve trazer também a direção inversa —
uma tabela de cobertura da fonte, mostrando o que da reunião não chegou a
nenhum documento e se a omissão foi deliberada? É o que transforma o tracker de
"lista do que escrevi" em "prova de que não perdi nada". E quando a varredura
encontrar divergência real, você corrige o documento na hora ou só reporta e
espera minha decisão? Minha inclinação é só reportar — corrigir PRD ou FDD
dentro de uma passada de tracker é como um teste consertar o código que ele
está testando.
```

O item que mais mudou o resultado é o segundo do bloco de contexto. Os alvos percentuais do
critério de aceite (≥70% com fonte na transcrição) criam um **incentivo perverso** direto num
pacote que contém, de propósito, 21 itens de ledger no FDD, 6 métricas propostas no PRD e 3
desambiguações feitas fora da reunião. O caminho de menor resistência para bater a meta é dar
timestamp a eles. Nomear cada um antes é o que impede isso.

**E duas linhas deste mesmo prompt produziram os dois defeitos que a Fase 7 teve que
consertar** — vale mais registrar isso do que o acerto:

- *"Minha inclinação é só reportar"*, no fim da pergunta (c). Eu não perguntei, eu induzi. A
  sessão adotou a inclinação e o pacote saiu com cinco defeitos conhecidos dentro.
- *"itens deliberadamente sem origem, e eles não podem ganhar uma"*, no bloco de contexto.
  A regra está certa — nenhum deles pode receber `[hh:mm]` — e é **incompleta**: ela proíbe o
  destino errado sem exigir nenhum destino certo. Os 41 itens ficaram numa tabela de "sem
  origem" e o trabalho parou ali.

### #8 — A passada de correção que a Fase 5 não fez (Fase 7)

Este é o prompt que teria evitado a Fase 7 se estivesse na Fase 5 — e, escrito depois, é o que
a executou. Ele ataca as duas omissões acima de frente: **proíbe entregar defeito conhecido** e
**exige destino, não só rótulo**, para todo item sem origem.

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

Duas coisas nesse prompt vieram de erro anterior, não de sabedoria. A regra "corrigir demais é
o mesmo defeito na direção oposta" existe porque a primeira tentativa de A-01 quase levou
`COD-07` junto — o valor `[09:29]` aparece cinco vezes no pacote e só as erradas eram para
mudar. E a pergunta (c) inverte deliberadamente o vício da pergunta (c) do prompt #7: em vez de
declarar minha inclinação e recebê-la de volta validada, declaro e **peço discordância**.

---

## Iterações e ajustes

### Quantas iterações, e a regra de contagem

O enunciado pede o número de iterações principais e sugere esperar de 3 a 5 ciclos de
geração → revisão → ajuste de prompt → nova geração. Dou dois números, com a regra explícita,
porque nenhum sozinho descreve o que aconteceu:

- **7 fases** de produção, cada uma com prompt próprio escrito para o seu modo de falha — a
  sétima não estava no plano e existe porque o pacote fechado tinha defeito conhecido dentro.
- **9 ajustes** numerados no log — momentos em que algo saiu errado e uma regra mudou por causa
  disso. O nono é o maior: **duas linhas do prompt da Fase 5 produziram os dois defeitos que a
  Fase 7 teve que consertar**, e as duas eram do mesmo tipo — proibição sem obrigação ao lado,
  e pergunta com a resposta anexada.

O que os dois números escondem, e que precisa ser dito: **houve pouquíssima regeração**. Nenhum
documento grande foi jogado fora e reescrito. Isso não é porque saiu certo de primeira — é
porque o custo foi deslocado para a frente, para antes da escrita. As sessões perguntaram e eu
decidi: **11 decisões fechadas antes do primeiro ADR** (2 rodadas de perguntas), **17 antes da
primeira linha do FDD** (3 rodadas), 4 antes do PRD. O ciclo não foi "gerar e regerar"; foi
"interrogar antes de gerar".

Isso tem um custo que também vale registrar: cada rodada de perguntas é uma decisão minha, e
decidir custa mais atenção do que revisar. O ganho é que a decisão fica registrada no prompt e
sobrevive à troca de sessão — enquanto a correção pós-geração vive só no documento e some do
processo.

### As cinco lições que os 9 ajustes produziram

**1. Contradição entre arquivos de instrução do próprio repositório é classe de defeito, não
evento.** (Ajustes #1, #3, #6 — quatro ocorrências.) O setup gerou `docs/agents/domain.md`
apontando ADRs para `docs/adr/`, enquanto o desafio exige `docs/adrs/` — e a pasta é critério de
aceite avaliado. Pego antes da Fase 1; se não fosse, as skills teriam escrito 7 arquivos numa
pasta que não é avaliada. Corrigido, apareceu de novo em `docs/adrs/README.md` (formato do nome
do arquivo), depois **de novo** nos títulos de seção — README em português, `domain.md` em
inglês — e uma quarta vez em `AGENTS.md`, na raiz, já na Fase 3. Uma busca por `docs/adr/` na
Fase 0 teria pego três de uma vez. O procedimento que ficou: **varrer os arquivos de instrução
do repositório a cada fase, antes de consumi-los.**

**2. Exigir formato não basta — é preciso proibir a variante plausível.** (Ajuste #2.) A
varredura da transcrição produziu 112 itens, e 4 vieram com timestamp em faixa (`[09:17-18]`)
em vez de `[hh:mm]`. A IA fez isso quando o item resumia uma troca que atravessava dois minutos:
comportamento razoável, formato inválido — e o critério de aceite do tracker exige `[hh:mm]`.
A lição é que a IA inventa a variante plausível quando o dado real não cabe no formato pedido,
então o prompt tem que **nomear a variante e proibi-la**. É por isso que a regra 2 do prompt #7
está escrita do jeito que está.

**3. A política de rastreabilidade não é única — ela muda de altura a cada documento.**
(Ajustes #4, #5, #7.) Na Fase 1 fixei que **ADRs registram política e trade-off, não schema**;
onde a fonte é omissa, vai para "Pontos em aberto". Isso levou a RFC a declarar, com todas as
letras, que dois campos estruturantes da outbox não têm origem em fonte nenhuma e que a
arquitetura os exige mesmo assim. **O FDD não pode herdar essa política sem deixar de ser FDD**
— um documento de implementação que se recusa a propor schema não é acionável, que é o único
critério que o define. A política virou: onde a fonte é omissa e a implementação exige, **o FDD
propõe e assume a autoria**, com marca visível — **F** (fonte direta), **D** (derivado) e **P**
(proposta sem origem), mais um ledger final reunindo os 21 itens **P** num lugar só, cada um com
a coluna "o que quebra se for rejeitado". A diferença entre isso e "escrever bonito e torcer" é
que o revisor ataca 21 linhas em vez de reler 1.109. E então a lição se repetiu espelhada no
PRD: **a marca de três níveis não sobrevive ao PRD sem que o PRD deixe de ser PRD** — num
documento de produto o mesmo aparato produz exatamente o defeito que a fase existia para evitar,
um texto que *parece* especificação técnica. No PRD a marca ficou restrita a duas seções
(métricas e escopo), onde o risco de invenção se concentra.

**4. Artefato derivado não é autoridade sobre a fonte que ele deriva.** (Ajuste #8.) O índice da
transcrição foi construído na Fase 0 e usado como base factual das Fases 1 a 4 **sem que ninguém
voltasse à transcrição para conferi-lo**. Ele é excelente como roteiro e é o que tornou as
quatro fases possíveis. Mas os três achados de timestamp da Fase 5 nascem todos nele, e nenhum
nos documentos: `[09:29]` no lugar de `[09:28]` em cinco linhas, `[09:47]` no lugar de `[09:49]`
em outra, e a fusão de duas datas numa terceira. Em todos, a citação literal está certa e o
carimbo de origem escorregou. O Ajuste #2 tinha ensinado metade disso — a IA inventa a variante
plausível quando o dado não cabe. A outra metade é que **ela também herda o erro plausível
quando o dado errado cabe perfeitamente**. Os três foram corrigidos na Fase 7, com o antes e o
depois de cada um registrado na nota de leitura (d) do próprio índice.

**5. Proibição sem obrigação ao lado produz um vazio que nada denuncia — e pergunta com a
resposta anexada não é pergunta.** (Ajuste #9.) As duas falhas que a Fase 7 consertou nasceram
de duas linhas do prompt da Fase 5, e nenhuma delas é descuido de execução: a sessão fez
exatamente o que foi pedido.

A primeira linha dizia que os itens sem origem "não podem ganhar uma". Correto, e **incompleto**
— proíbe o destino errado (inventar `[hh:mm]`) sem exigir nenhum destino certo. Os 41 itens
foram para uma tabela de "sem origem" e o trabalho parou ali, porque ali acabava a instrução.
O vazio passou despercebido justamente porque nada no texto o apontava: o prompt foi cumprido
à risca.

A segunda fechava uma das perguntas de abertura com *"minha inclinação é só reportar"*. Eu não
perguntei, induzi — e recebi minha própria inclinação de volta, agora com o peso de ter sido
"decidida". O padrão "comece me perguntando" só vale enquanto a pergunta for aberta; anexar a
resposta a converte num eco caro. O prompt #8 inverte isso de propósito: declara a inclinação e
**pede discordância explícita**.

### Quatro achados concretos que a revisão produziu

Os ajustes acima mudaram o método. Estes mudaram o conteúdo — são defeitos de design que a
reunião não levantou e que só apareceram porque escrever força perguntas que discutir não força:

**`WEBHOOK_SECRET_REQUIRED` não tem gatilho possível.** Bruno nomeia o código de erro na
reunião; três minutos depois, Marcos decide que a secret é gerada por nós e devolvida na
criação. O cliente nunca fornece secret, logo nunca pode omiti-la. O código foi batizado antes
da decisão que o tornou inalcançável — e atravessou um ADR e a RFC inteira sem que ninguém
notasse. Só apareceu quando o FDD precisou montar a matriz de erros com gatilho por linha.

**Toda linha reivindicada morre presa a cada deploy do worker.** O status `processando` da
outbox não tem regra de saída para o caso de o processo morrer no meio. Não é cenário exótico:
é garantido em todo restart. E o resultado — evento que nunca sai — contradiz frontalmente o
requisito que motivou a arquitetura inteira. O desdobramento vale como método: a solução
(devolver a linha a `pendente` e arriscar entrega dupla) parecia exigir uma garantia nova, até
ficar claro que **duplicata já era o contrato aceito** por uma decisão anterior. Procurar *qual
decisão já tomada autoriza o que você quer fazer* é mais barato que decidir de novo — e deixa
rastro.

**O baseline do problema estava numa fala só, e nenhum dos quatro documentos técnicos a usou.**
Aos `[09:00]`, Marcos diz que os clientes "ficam batendo no `GET /orders` de tempos em tempos
pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles". RFC, FDD
e os sete ADRs abrem no *pedido* dos clientes e vão direto para a arquitetura; nenhum registra
o que o cliente faz hoje. É a origem da expressão "tempo real" do requisito de latência e a
única referência possível para a métrica principal de produto. Um pacote inteiro descreveu a
solução sem descrever o estado que ela substitui. Só apareceu ao **subir de altura** no PRD —
descer ao detalhe audita o design, subir de altura audita o propósito.

**O prazo que justifica a pressão do projeto nunca foi acordado com quem o impôs.** Duas datas
diferentes circulam na reunião — "fim do trimestre" e "fim de novembro" — e o índice as fundiu
numa linha. Somado a uma questão em aberto onde Marcos promete confirmar com os três clientes e
o resultado nunca volta, a conclusão é que **a data é nossa, não deles**. Nenhum documento
técnico tinha motivo para notar isso.

### Seis achados encontrados — e a Fase 5 parou aí

A Fase 5 é a única passada que lê os cinco documentos lado a lado, e a política que escolhi na
hora foi **reportar sem editar**: uma auditoria que conserta o que está auditando deixa de ser
auditoria. O tracker fechou com seis achados abertos — os três erros de timestamp do índice
(dois deles já propagados para o FDD e o PRD), seções "Fontes" que declaram faixas contíguas de
IDs que o corpo não cita, e um timestamp em faixa que sobreviveu no ADR-002 porque foi escrito
na Fase 1, **antes de o Ajuste #2 virar regra de prompt**.

Esse último tem forma própria e vale como procedimento: **correção aplicada à fonte não se
propaga sozinha para o que já foi derivado dela.** Quando uma regra nova nasce no meio do
processo, tudo que foi escrito antes dela continua sem a regra — e só uma varredura posterior
encontra.

O sexto achado é negativo e é o que sustenta o resto: dos 31 caminhos de arquivo citados no
pacote, 27 existem e os 4 que não existem são nomeados pelos próprios documentos como arquivos
a criar; os 17 números de linha do FDD conferem um a um; e as três afirmações negativas mais
fáceis de errar também conferem.

### Fase 7 — o que eu errei ao só reportar

O argumento "auditoria não edita o que audita" está certo sobre **como auditar** e errado sobre
**o que entregar**. Ele confunde dois papéis: quem audita não deve mexer no que está medindo
*durante* a medição — mas a entrega não é a auditoria, é o pacote. E o critério de consistência
do desafio não diz "nenhuma divergência não declarada": diz que **nenhuma informação registrada
pode contradizer ou representar incorretamente a fonte**. Um timestamp que aponta para a fala
errada representa a fonte incorretamente **esteja ou não confessado três seções abaixo**.
Declarar um defeito é boa prática de auditoria; não é o mesmo que não ter o defeito.

A mesma confusão aparecia no segundo ponto. Os documentos separavam, com honestidade, 41 itens
"sem origem em fonte nenhuma" — e paravam aí. Mas a regra do desafio é que **toda informação
registrada seja rastreável**, e um item que se declara não rastreável continua não rastreável.
A marca **P** impedia alucinação silenciosa e não transformava proposta em informação de origem.

A Fase 7 fez as duas coisas que faltavam:

**1. Corrigi os cinco achados**, e o registro do que mudou ficou no lugar do achado — cada um
com o que estava, o que ficou e os arquivos tocados ([TRACKER §5](docs/TRACKER.md#5-achados-da-varredura)).
`TRANSCRICAO.md` não foi tocado: a fonte não se corrige. Duas coisas apareceram nessa passada
que a Fase 5 não tinha visto. A primeira: o achado A-04 valia também para os **sete ADRs**, que
a varredura original não cobria — e neles o defeito aparecia **nos dois sentidos**, incluindo 10
IDs citados no corpo e **ausentes** da lista de fontes, que é o caso mais grave, porque some com
a origem em vez de inflá-la. A segunda: corrigir A-03 direito significou **separar a linha em
duas**, e o índice passou de 112 para 113 itens — uma correção de rastreabilidade que muda um
denominador publicado obriga a mexer em todo lugar que cita o denominador.

**2. Dei destino aos 41 itens acrescentados.** Cada um passou a ser uma de duas coisas, e
nenhum ficou solto:

- **14 derivações explícitas**, cada uma com a **cadeia** que a liga ao item de fonte de que
  decorre — `attemptCount` decorre de `DEC-05` mais `TEC-03`, o envelope paginado decorre de
  `RF-06` mais `response.ts` mais a regra de reuso, e assim por diante. Derivação com a cadeia
  escrita **é** rastreável: a origem é indireta e conferível, e se a cadeia estiver errada o
  item cai junto.
- **27 propostas pendentes de aprovação**, cada uma endereçada a **uma pessoa nomeada, nos dois
  fóruns que a própria reunião criou**: a revisão de segurança de Sofia (`RNF-10`, bloqueante
  por `RES-06`) e a sessão de revisão de design que Larissa assumiu abrir (`[09:50]`). Elas
  entram como **pergunta com dono e prazo**, não como afirmação — e os documentos dizem, no
  ponto em que cada uma aparece, que **enquanto não aprovada não é requisito, meta nem
  compromisso**.

A lição que fecha o processo: **um pacote de documentação não é avaliado pela qualidade da sua
consciência sobre si mesmo.** Saber onde está o defeito é o pré-requisito de corrigi-lo, não o
substituto. O tracker fez o trabalho difícil — encontrar — e eu parei no passo fácil.

---

## Como navegar a entrega

### Ordem de leitura ≠ ordem de produção

Produzi na ordem **decisão → proposta → detalhe → produto → auditoria**, porque as decisões
formam o esqueleto do "como implementar" e escrever o PRD antes teria produzido um documento
que promete o que a arquitetura ainda não sabia se aguentava. Mas ninguém deve **ler** nessa
ordem: para quem chega de fora, o caminho é o inverso — do "por quê" ao "como".

A divergência é o mesmo argumento de altura que estrutura o pacote inteiro. Documento de altura
alta é fácil de ler e difícil de escrever primeiro; documento de altura baixa é o contrário.

| Ler | Documento | Responde | Tamanho | Produzido |
| --- | --- | --- | --- | --- |
| 1º | [`docs/PRD.md`](docs/PRD.md) | Por quê e o quê? | 473 linhas | 5º |
| 2º | [`docs/RFC.md`](docs/RFC.md) | Como pretendemos resolver, e o que segue em aberto? | 225 linhas | 3º |
| 3º | [`docs/adrs/`](docs/adrs/) | Por que decidimos exatamente assim? | 7 ADRs, 69–92 linhas cada | 2º |
| 4º | [`docs/FDD.md`](docs/FDD.md) | Como construir, em detalhe? | 1123 linhas | 4º |
| 5º | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada coisa? | 640 linhas | 6º |

Atalhos por interesse: quem quer **avaliar rastreabilidade** pode começar direto pelo TRACKER,
que publica o próprio denominador, traz a varredura reversa e registra o antes/depois de cada
correção. Quem vai **implementar** pode ir direto ao FDD e usar o registro final — 13 derivações
com a cadeia até a fonte, 8 propostas com aprovador — como a lista do que precisa ser confirmado
antes de codar; as 8 propostas são as que **não valem como requisito** até a aprovação sair.

### Fontes e material de trabalho

| Arquivo | O que é |
| --- | --- |
| [`TRANSCRICAO.md`](TRANSCRICAO.md) | Fonte primária. A reunião de 55 minutos, 5 participantes. Não alterado. |
| [`CONTEXT.md`](CONTEXT.md) | Mapa do código existente produzido na Fase 0. Sustenta a seção "Integração com o sistema existente" do FDD. |
| [`docs/DESAFIO.md`](docs/DESAFIO.md) | O enunciado original, preservado. É a régua contra a qual os números abaixo se leem. |
| [`.scratch/fontes/transcricao-index.md`](.scratch/fontes/transcricao-index.md) | **Material de trabalho, não entregável.** O índice de 113 itens da transcrição (112 até a Fase 7 separar `RES-04` em dois). Base factual das Fases 1 a 4, e o arquivo que concentra os três defeitos de carimbo corrigidos na Fase 7 — a nota de leitura (d) registra o que mudou. |
| [`.scratch/processo/prompts-e-iteracoes.md`](.scratch/processo/prompts-e-iteracoes.md) | **Material de trabalho, não entregável.** Registro contemporâneo do processo: os 8 prompts na íntegra, os 9 ajustes e as verificações de cada fase, incluindo a Fase 7 e a autópsia das duas linhas do prompt #7 que a tornaram necessária. É a fonte deste README. |

Os dois arquivos em `.scratch/` estão versionados de propósito. Sem eles, este README seria uma
narrativa auto-atestada; com eles, é uma narrativa conferível contra o registro que a produziu.

### O que o pacote entrega, em números

| Documento | Entregue | Mínimo exigido |
| --- | --- | --- |
| PRD | 11 requisitos funcionais, 10 não funcionais, 11 riscos com probabilidade/impacto/mitigação, 16 critérios de aceitação, 8 itens fora de escopo decidido + 2 na categoria "ninguém perguntou", 10 questões abertas com dono + 16 propostas próprias com aprovador e prazo | 8 RF, 2 riscos, 2 fora de escopo |
| RFC | 225 linhas, 8 alternativas descartadas com trade-off, 15 questões em aberto (12 da reunião + 3 desambiguações feitas fora dela, aguardando ratificação), links para os 7 ADRs | 2 alternativas, 2 questões, 2 links |
| FDD | 7 contratos de endpoint, 10 caminhos de arquivo reais na seção de integração, matriz de erros `WEBHOOK_*`, registro final com 13 derivações (cada uma com a cadeia até a fonte) e 8 propostas com aprovador nomeado | 4 endpoints, 4 caminhos |
| ADRs | 7, cobrindo 6 das 6 decisões principais | 5 a 8 arquivos, 5 das 6 decisões |
| TRACKER | 234 linhas na tabela principal (208 com fonte na transcrição, 88,9%; 26 no código, cobrindo 14 arquivos), 41 itens acrescentados pelos documentos — 14 derivações com cadeia e 27 propostas com aprovador —, zero órfãos na varredura reversa dos 113 itens do índice, 6 achados **corrigidos e reverificados** | 80% de cobertura, 70% transcrição, 5 linhas de código |
