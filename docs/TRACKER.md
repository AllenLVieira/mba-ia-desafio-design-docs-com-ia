# TRACKER — Rastreabilidade do pacote de design docs

Referência cruzada entre cada item registrado nos documentos do pacote e sua origem: a transcrição da reunião (`TRANSCRICAO.md`) ou o código da aplicação.

## Como ler

- **Fonte `TRANSCRICAO`** — a **Localização** traz `[hh:mm] Nome` da fala que contém a citação que sustenta o item. **Todo timestamp desta tabela foi conferido diretamente em `TRANSCRICAO.md`**, não copiado do índice de fontes (`.scratch/fontes/transcricao-index.md`), que é artefato derivado e tem defeitos conhecidos — ver seção 5.
- **Fonte `CODIGO`** — a **Localização** traz o caminho real do arquivo, e o símbolo ou a linha quando o item depende deles. Todos os caminhos foram verificados no repositório.
- **Itens sem origem em fonte nenhuma** não recebem timestamp inventado: eles estão na **seção 3**, em tabela própria.
- Quando um item nasce do **confronto entre duas falas** (um conflito que ninguém levantou na reunião), a Localização traz a fala principal e o Conteúdo nomeia a outra.

## Índice

| Seção | O que traz |
|---|---|
| [1. Regra de contagem](#1-regra-de-contagem-e-denominador) | o que conta como item, e o denominador dos percentuais |
| [2. Tabela de rastreabilidade](#2-tabela-de-rastreabilidade) | 234 itens com origem — PRD, RFC, FDD e ADRs |
| [3. Itens sem origem](#3-itens-sem-origem-em-fonte-nenhuma) | 41 itens que os documentos propõem e ninguém decidiu |
| [4. Varredura reversa](#4-varredura-reversa--cobertura-da-transcrição) | os 112 itens da reunião: onde cada um foi parar |
| [5. Achados](#5-achados-da-varredura) | divergências encontradas, reportadas e **não corrigidas** |
| [6. Conformidade](#6-conformidade-com-os-critérios-de-aceite) | os números fechados contra os critérios do desafio |

---

## 1. Regra de contagem e denominador

Sem uma regra de contagem declarada, "80% dos itens identificáveis" é um percentual que se autoatesta. Esta é a regra usada, e o número absoluto que ela produz.

**Conta como item identificável:** toda afirmação que os documentos já identificam com um rótulo próprio — `RF-xx`, `RNF-xx`, `OB-xx`, `MP-xx`, `CU-xx`, `T-xx`, `D-xx`, `R-xx`, `AC-xx`, `V-xx`, `VP-xx`, `OT-xx`, `CA-xx`, questão numerada da RFC, alternativa considerada, endpoint de contrato, código de erro, ponto de integração, model do schema, variável de ambiente, item do ledger do FDD, e cada decisão registrada por um ADR.

**Uma linha por item, no documento onde o item é definido.** Quando um documento apenas repete um item definido em outro (o FDD relistando os `ADI` do PRD, a RFC relistando riscos, o PRD §13 consolidando questões já registradas em `D-xx` e `T-xx`), isso é **restatement** e não gera linha nova — a repetição já é rastreável pela linha original.

| | Quantidade |
|---|---|
| Itens rotulados varridos nos cinco documentos | **331** |
| — dos quais **restatements** (definidos em outro documento do pacote) | 56 |
| **Itens próprios, que exigem linha no tracker** | **275** |
| Registrados na seção 2 (com origem em transcrição ou código) | 234 |
| Registrados na seção 3 (sem origem em fonte nenhuma) | 41 |
| **Cobertura dos itens próprios** | **275 / 275 = 100%** |
| **Cobertura de todos os itens rotulados** (incluindo restatements) | **275 / 331 = 83%** |

Os dois percentuais estão acima do mínimo de 80%. O segundo é o mais conservador possível: trata cada repetição como se fosse item não rastreado.

---

## 2. Tabela de rastreabilidade

### 2.1 PRD — `docs/PRD.md`

#### Cenários de uso

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CU-01 | docs/PRD.md | Cenário de uso | Cliente passa a receber notificação em vez de consultar `GET /orders` em loop | TRANSCRICAO | [09:00] Marcos |
| PRD-CU-02 | docs/PRD.md | Cenário de uso | Cliente assina só parte do ciclo: "só quero saber quando vira SHIPPED e DELIVERED" | TRANSCRICAO | [09:33] Marcos |
| PRD-CU-03 | docs/PRD.md | Cenário de uso | Endpoint em manutenção planejada de duas horas recebe os eventos retidos ao voltar | TRANSCRICAO | [09:16] Diego |
| PRD-CU-04 | docs/PRD.md | Cenário de uso | Cliente fora por mais de ~15h: eventos vão à dead letter e são reprocessados manualmente | TRANSCRICAO | [09:18] Diego |
| PRD-CU-05 | docs/PRD.md | Cenário de uso | Rotação de secret com 24h de convivência entre a antiga e a nova | TRANSCRICAO | [09:21] Sofia |
| PRD-CU-06 | docs/PRD.md | Cenário de uso | Cliente com vários cadastros identifica qual recebeu o envio, pelo `X-Webhook-Id` | TRANSCRICAO | [09:44] Sofia |
| PRD-CU-07 | docs/PRD.md | Cenário de uso | Cliente pede evidência: últimos 100 envios com sucesso/falha, payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-CU-08 | docs/PRD.md | Cenário de uso | Cliente que precisa dos itens do pedido busca em `GET /orders/:id` depois | TRANSCRICAO | [09:43] Diego |
| PRD-CU-09 | docs/PRD.md | Cenário de uso | Auditoria: saber quem executou um reprocessamento e quando | TRANSCRICAO | [09:36] Sofia |
| PRD-CU-10 | docs/PRD.md | Cenário de uso | Atlas Comercial avalia migrar para o concorrente se a entrega não sair no prazo | TRANSCRICAO | [09:00] Marcos |

#### Objetivos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OB-01 | docs/PRD.md | Objetivo | O cliente deixa de consultar para descobrir: a plataforma avisa | TRANSCRICAO | [09:00] Marcos |
| PRD-OB-02 | docs/PRD.md | Objetivo | Notificação em menos de 10 segundos ponta a ponta | TRANSCRICAO | [09:02] Marcos |
| PRD-OB-03 | docs/PRD.md | Objetivo | Cobrir indisponibilidade planejada: meta ≥2h; o desenho decidido cobre 14h36 | TRANSCRICAO | [09:16] Diego |
| PRD-OB-04 | docs/PRD.md | Objetivo | Zero pedidos com status alterado e evento não registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-OB-05 | docs/PRD.md | Objetivo | 100% dos eventos com tentativas esgotadas ficam recuperáveis e reprocessáveis | TRANSCRICAO | [09:18] Diego |
| PRD-OB-06 | docs/PRD.md | Objetivo | Os três clientes que pediram passam a usar (os nomes são fonte; a meta "3 de 3" é proposta) | TRANSCRICAO | [09:00] Marcos |
| PRD-OB-07 | docs/PRD.md | Objetivo | Três sprints, com a revisão de segurança de Sofia incluída no fim | TRANSCRICAO | [09:47] Larissa |
| PRD-OB-08 | docs/PRD.md | Objetivo | Nenhuma dependência nova e nenhum serviço novo além do próprio worker | TRANSCRICAO | [09:30] Larissa |

#### Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastro de endpoint com url, lista de status e secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | A configuração de um endpoint pode ser editada | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Um endpoint pode ser removido | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | É possível listar os endpoints de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Cada endpoint declara quais status quer ouvir e recebe só esses | TRANSCRICAO | [09:33] Marcos |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Histórico dos últimos 100 envios: sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Secret rotacionável pela API, com a antiga válida por 24h em paralelo | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Administrador reprocessa manualmente evento que esgotou as tentativas | TRANSCRICAO | [09:35] Diego |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Todo envio é assinado, para o cliente provar que veio da plataforma | TRANSCRICAO | [09:20] Sofia |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | O reprocessamento registra quem o executou, para auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | O registro do evento acontece na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |

#### Requisitos não funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência ponta a ponta abaixo de 10 segundos — definição de "tempo real" do cliente | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Tolerância a indisponibilidade do cliente: janela de ~15 horas (14h36) | TRANSCRICAO | [09:17] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once; a deduplicação é do cliente | TRANSCRICAO | [09:26] Larissa |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Integridade: não existe status mudado sem evento registrado, nem o inverso | TRANSCRICAO | [09:06] Diego |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Teto de 64KB por evento; acima disso falha com erro, não trunca | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | 10 segundos de espera pela resposta do cliente; depois é falha e vira retentativa | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | HTTPS obrigatório; endpoint `http` é recusado no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Ordem garantida por pedido, não global, e apenas enquanto houver um único worker | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Uma secret por endpoint, nunca uma secret global: "se vaza uma, vaza tudo" | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-10 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança de no mínimo 2 dias úteis antes do deploy, bloqueante | TRANSCRICAO | [09:46] Sofia |

#### Escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-ESC-01 | docs/PRD.md | Item adiado | Fora de escopo: aviso por email ao cliente quando o webhook falha repetidamente | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-02 | docs/PRD.md | Item adiado | Fora de escopo: limite de vazão de envios — classificação divergente, sem decisão | TRANSCRICAO | [09:39] Diego |
| PRD-ESC-03 | docs/PRD.md | Item adiado | Fora de escopo: dashboard visual para o cliente — projeto do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-04 | docs/PRD.md | Item adiado | Fora de escopo: arquivamento das linhas entregues após ~30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-05 | docs/PRD.md | Item adiado | Fora de escopo: escalar para mais de um worker — "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| PRD-ESC-06 | docs/PRD.md | Item adiado | Fora de escopo: endurecer a autorização do CRUD além de "qualquer autenticado" | TRANSCRICAO | [09:37] Sofia |
| PRD-ESC-ND-01 | docs/PRD.md | Escopo não decidido | Se a criação do pedido gera evento — a pergunta nunca foi feita na reunião | CODIGO | src/modules/orders/order.service.ts (`create`, linha 50) |
| PRD-ESC-ND-02 | docs/PRD.md | Escopo não decidido | O que acontece com o histórico quando uma assinatura é removida — nunca perguntado | TRANSCRICAO | [09:33] Bruno |

#### Decisões e trade-offs de produto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-T-01 | docs/PRD.md | Trade-off | Notificar depois, não durante: desacoplamento de latência ao custo de entrega não imediata | TRANSCRICAO | [09:04] Bruno |
| PRD-T-02 | docs/PRD.md | Trade-off | At-least-once: a deduplicação (e o custo dela) é do cliente | TRANSCRICAO | [09:26] Larissa |
| PRD-T-03 | docs/PRD.md | Trade-off | Cobrir indisponibilidade longa em vez de falhar rápido: evento em trânsito por até 14h36 | TRANSCRICAO | [09:16] Diego |
| PRD-T-04 | docs/PRD.md | Trade-off | Payload enxuto sem os itens do pedido: a feature reduz o polling mas não elimina toda consulta | TRANSCRICAO | [09:43] Diego |
| PRD-T-05 | docs/PRD.md | Trade-off | Consistência acima de disponibilidade da escrita: 64KB falha e derruba a mudança de status | TRANSCRICAO | [09:24] Larissa |
| PRD-T-06 | docs/PRD.md | Trade-off | Ordem declarada como limitação, não construída como garantia | TRANSCRICAO | [09:13] Larissa |
| PRD-T-07 | docs/PRD.md | Trade-off | Configuração acessível a qualquer operador autenticado, como estado temporário | TRANSCRICAO | [09:37] Sofia |
| PRD-T-08 | docs/PRD.md | Trade-off | Nenhuma infraestrutura nova: teto de vazão e piso de latência fixados por essa escolha | TRANSCRICAO | [09:07] Diego |

#### Dependências

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-D-01 | docs/PRD.md | Dependência | Revisão de segurança de Sofia, bloqueante, sem data e sem responsável pelo agendamento | TRANSCRICAO | [09:46] Sofia |
| PRD-D-02 | docs/PRD.md | Dependência | Documentar a garantia at-least-once no portal do cliente — assumido por Marcos, sem prazo | TRANSCRICAO | [09:26] Marcos |
| PRD-D-03 | docs/PRD.md | Dependência | Sessão de revisão de design com Bruno e Diego antes de codar — sem data | TRANSCRICAO | [09:50] Larissa |
| PRD-D-04 | docs/PRD.md | Dependência | Confirmação do prazo com os três clientes — prometida, resultado nunca voltou | TRANSCRICAO | [09:49] Marcos |
| PRD-D-05 | docs/PRD.md | Dependência | O cliente implementar a verificação da assinatura e a dedup por `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| PRD-D-06 | docs/PRD.md | Dependência | Onde o worker roda, quem o mantém vivo e quem é avisado se cair — não há serviço de aplicação | CODIGO | docker-compose.yml |
| PRD-D-07 | docs/PRD.md | Dependência | Decidir `RNF-01` antes de comunicar o alvo de 10s (conflito com `DEC-02` e `RNF-06`) | TRANSCRICAO | [09:10] Larissa |
| PRD-D-08 | docs/PRD.md | Dependência | Decidir as duas perguntas nunca feitas do escopo (evento na criação; histórico na remoção) | CODIGO | src/modules/orders/order.service.ts (`create`, linha 50) |
| PRD-D-09 | docs/PRD.md | Dependência | Capacidade do time em três sprints, com time declaradamente pequeno | TRANSCRICAO | [09:47] Larissa |
| PRD-D-10 | docs/PRD.md | Dependência | O ponto de acoplamento: a mudança de status é o único gatilho | TRANSCRICAO | [09:40] Bruno |

#### Riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-R-01 | docs/PRD.md | Risco | Churn da Atlas se a entrega escorregar — revisão bloqueante no fim e nunca agendada | TRANSCRICAO | [09:00] Marcos |
| PRD-R-02 | docs/PRD.md | Risco | O prazo nunca foi confirmado com os três clientes; "fim de novembro" é data nossa | TRANSCRICAO | [09:45] Marcos |
| PRD-R-03 | docs/PRD.md | Risco | Prometer "tempo real abaixo de 10s" e não cumprir: >12s já sem retentativa | TRANSCRICAO | [09:02] Marcos |
| PRD-R-04 | docs/PRD.md | Risco | O cliente não deduplica e processa o mesmo pedido duas vezes — objeção nunca retirada | TRANSCRICAO | [09:25] Sofia |
| PRD-R-05 | docs/PRD.md | Risco | Autorização frouxa: qualquer operador opera webhook de qualquer cliente | TRANSCRICAO | [09:37] Sofia |
| PRD-R-06 | docs/PRD.md | Risco | A entrega para de funcionar em silêncio se o processo do worker cair | TRANSCRICAO | [09:11] Diego |
| PRD-R-07 | docs/PRD.md | Risco | A feature de notificação derruba uma mudança de status legítima (64KB + mesma transação) | TRANSCRICAO | [09:24] Larissa |
| PRD-R-08 | docs/PRD.md | Risco | O cliente descobre tarde que parou de receber: sem email e sem painel | TRANSCRICAO | [09:37] Larissa |
| PRD-R-09 | docs/PRD.md | Risco | Evento fora de ordem regride o estado no sistema do cliente | TRANSCRICAO | [09:13] Larissa |
| PRD-R-10 | docs/PRD.md | Risco | Reprocessamento manual não escala: uma ação por evento perdido | TRANSCRICAO | [09:18] Diego |
| PRD-R-11 | docs/PRD.md | Risco | Surpresa de escopo: quem assina `PENDING` não recebe nada quando o pedido nasce | CODIGO | src/modules/orders/order.service.ts (`create`, linha 50) |

#### Critérios de aceitação de produto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-AC-01 | docs/PRD.md | Critério de aceite | Cliente com endpoint cadastrado recebe `POST` sem ter feito nenhuma consulta | TRANSCRICAO | [09:00] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de aceite | Cliente que assinou parte dos status não recebe os demais | TRANSCRICAO | [09:33] Marcos |
| PRD-AC-03 | docs/PRD.md | Critério de aceite | O cliente consegue verificar, com a secret recebida, que o envio veio da plataforma | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-04 | docs/PRD.md | Critério de aceite | Endpoint `http` é recusado no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-05 | docs/PRD.md | Critério de aceite | A secret é entregue no cadastro e na rotação, e em nenhum outro momento | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-06 | docs/PRD.md | Critério de aceite | Após a rotação, quem ainda usa a secret antiga valida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de aceite | Endpoint indisponível por duas horas recebe o evento quando volta | TRANSCRICAO | [09:16] Diego |
| PRD-AC-08 | docs/PRD.md | Critério de aceite | Além da janela de retentativa, o evento fica recuperável e reprocessável | TRANSCRICAO | [09:18] Diego |
| PRD-AC-09 | docs/PRD.md | Critério de aceite | O reprocessamento é negado a quem não é administrador e registra quem o executou | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-10 | docs/PRD.md | Critério de aceite | O cliente consulta os últimos 100 envios com sucesso/falha, payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-11 | docs/PRD.md | Critério de aceite | Se o registro do evento falhar, a mudança de status não acontece | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-12 | docs/PRD.md | Critério de aceite | Evento acima de 64KB falha com erro, sem truncamento silencioso | TRANSCRICAO | [09:24] Larissa |
| PRD-AC-13 | docs/PRD.md | Critério de aceite | Cliente com mais de um endpoint identifica qual cadastro recebeu cada envio | TRANSCRICAO | [09:44] Sofia |
| PRD-AC-14 | docs/PRD.md | Critério de aceite | A entrega funciona sem nenhuma dependência nova no projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-AC-15 | docs/PRD.md | Critério de aceite | A revisão de segurança ocorreu, com no mínimo 2 dias úteis, antes do deploy | TRANSCRICAO | [09:49] Sofia |
| PRD-AC-16 | docs/PRD.md | Critério de aceite | O material do cliente descreve at-least-once, dedup, limitação de ordem e latência | TRANSCRICAO | [09:26] Marcos |

#### Validações antes do go-live

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-V-01 | docs/PRD.md | Validação | Revisão de segurança de Sofia, ≥2 dias úteis, foco em HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| PRD-V-02 | docs/PRD.md | Validação | Sessão de revisão de design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-V-03 | docs/PRD.md | Validação | Confirmação do prazo com Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:49] Marcos |
| PRD-V-04 | docs/PRD.md | Validação | Material de integração publicado no portal antes do go-live | TRANSCRICAO | [09:26] Marcos |
| PRD-V-05 | docs/PRD.md | Validação | Fechar as decisões de produto pendentes antes de comunicar contrato ao cliente | TRANSCRICAO | [09:02] Marcos |

---

### 2.2 RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Metadados | Revisores do documento: os cinco participantes da reunião | TRANSCRICAO | [09:00] Larissa |
| RFC-META-02 | docs/RFC.md | Metadados | Autoria atribuída a Larissa, que assumiu abrir o doc de design | TRANSCRICAO | [09:50] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Despacho síncrono na transação — cliente lento travaria mudança de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams ou fila equivalente — overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger no banco notificando o worker — MySQL não tem `NOTIFY`/`LISTEN` | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Três retentativas — não sobrevivem a manutenção planejada de duas horas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido — evento pendurado para sempre se o cliente sumiu | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Marcar falha na própria outbox — tabela DLQ separada mantém a leitura limpa | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Exactly-once — exige coordenação bilateral; descarte com discordância viva de Sofia | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-08 | docs/RFC.md | Alternativa descartada | Itens do pedido no payload — descartado para não inflar o evento | TRANSCRICAO | [09:43] Diego |
| RFC-API-01 | docs/RFC.md | Contrato | `POST /api/v1/webhooks` — caminho derivado da convenção de rotas existente | CODIGO | src/modules/orders/order.routes.ts (`buildOrderRouter`, linha 12) |
| RFC-API-02 | docs/RFC.md | Contrato | `PATCH /api/v1/webhooks/:id` — caminho derivado | CODIGO | src/modules/orders/order.routes.ts |
| RFC-API-03 | docs/RFC.md | Contrato | `DELETE /api/v1/webhooks/:id` — caminho derivado | CODIGO | src/modules/orders/order.routes.ts |
| RFC-API-04 | docs/RFC.md | Contrato | `GET /api/v1/webhooks?customerId=` — `customerId` na query, pela convenção real | CODIGO | src/modules/orders/order.schemas.ts (`listOrdersQuerySchema`) |
| RFC-API-05 | docs/RFC.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` — caminho literal da reunião | TRANSCRICAO | [09:34] Marcos |
| RFC-API-07 | docs/RFC.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — caminho literal, exige `ADMIN` | TRANSCRICAO | [09:18] Diego |
| RFC-ART-01 | docs/RFC.md | Modelo de dados | Artefato: configuração de webhook com url, secret, `customer_id` e estado ativo | TRANSCRICAO | [09:21] Bruno |
| RFC-ART-02 | docs/RFC.md | Modelo de dados | Artefato: `webhook_outbox` com status e `created_at` indexados | TRANSCRICAO | [09:08] Diego |
| RFC-ART-03 | docs/RFC.md | Modelo de dados | Artefato: `webhook_dead_letter` com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| RFC-ART-04 | docs/RFC.md | Modelo de dados | Artefato: histórico de entregas — comportamento pedido, estrutura não projetada | TRANSCRICAO | [09:34] Marcos |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Schema do histórico de entregas: sem ele, `RF-06` não é implementável | TRANSCRICAO | [09:34] Marcos |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Agendamento do retry: nenhuma fonte diz como o worker sabe que chegou a hora | TRANSCRICAO | [09:17] Larissa |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Armazenamento da secret em repouso — a fonte lista o campo sem qualificar o formato | TRANSCRICAO | [09:21] Bruno |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | O alvo de latência não fecha: `DEC-02` (2s) + `RNF-06` (10s) excedem `RNF-01` | TRANSCRICAO | [09:10] Larissa |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | O retry quebra a ordem por `order_id`, combinando `DEC-04` com `DEC-05` | TRANSCRICAO | [09:13] Larissa |
| RFC-Q-06 | docs/RFC.md | Questão em aberto | `errorMiddleware` não alcança o worker, que roda fora do Express | TRANSCRICAO | [09:29] Bruno |
| RFC-Q-07 | docs/RFC.md | Questão em aberto | Segundo ponto de disparo: a criação do pedido nunca foi discutida | CODIGO | src/modules/orders/order.service.ts (`create`, linha 50) |
| RFC-Q-08 | docs/RFC.md | Questão em aberto | Supervisão do processo do worker — não há serviço de aplicação onde declará-lo | CODIGO | docker-compose.yml |
| RFC-Q-09 | docs/RFC.md | Questão em aberto | A convenção de `customerId` (body/query) não tem endosso dos participantes | TRANSCRICAO | [09:32] Larissa |
| RFC-Q-10 | docs/RFC.md | Questão em aberto | Concordância de Sofia com a entrega at-least-once nunca foi expressa | TRANSCRICAO | [09:25] Sofia |
| RFC-Q-11 | docs/RFC.md | Questão em aberto | Rate limiting de envios: classificação divergente, sem decisão de fazer ou descartar | TRANSCRICAO | [09:39] Diego |
| RFC-Q-12 | docs/RFC.md | Questão em aberto | Modelo de ameaça não explorado: a assinatura cobre só o corpo, headers ficam de fora | TRANSCRICAO | [09:22] Sofia |

---

### 2.3 FDD — `docs/FDD.md`

#### Objetivos técnicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-OT-01 | docs/FDD.md | Objetivo técnico | Registrar a promessa de notificação atomicamente com a mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-OT-02 | docs/FDD.md | Objetivo técnico | Entregar por HTTPS com assinatura HMAC-SHA256 verificável pelo cliente | TRANSCRICAO | [09:22] Sofia |
| FDD-OT-03 | docs/FDD.md | Objetivo técnico | Sobreviver a indisponibilidade de até ~15h: 6 chamadas em 14h36 | TRANSCRICAO | [09:17] Larissa |
| FDD-OT-04 | docs/FDD.md | Objetivo técnico | Não perder evento cuja entrega falhou definitivamente | TRANSCRICAO | [09:18] Diego |
| FDD-OT-05 | docs/FDD.md | Objetivo técnico | Permitir ao cliente deduplicar reenvios por `X-Event-Id` estável | TRANSCRICAO | [09:26] Larissa |
| FDD-OT-06 | docs/FDD.md | Objetivo técnico | Não introduzir infraestrutura nem dependência nova | TRANSCRICAO | [09:30] Larissa |
| FDD-OT-07 | docs/FDD.md | Objetivo técnico | Não alterar o comportamento observável de quem não usa webhooks | CODIGO | src/modules/orders/order.service.ts (`changeStatus`, linha 126) |

#### Modelo de dados

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-MODELO-01 | docs/FDD.md | Modelo de dados | Enum `WebhookOutboxStatus` com os quatro estados citados na reunião | TRANSCRICAO | [09:08] Diego |
| FDD-MODELO-02 | docs/FDD.md | Modelo de dados | `WebhookSubscription`: url, secret, `customerId`, `active` e a lista de status assinados | TRANSCRICAO | [09:21] Bruno |
| FDD-MODELO-03 | docs/FDD.md | Modelo de dados | `WebhookOutbox`: id UUID, status e `createdAt` indexados, payload em snapshot | TRANSCRICAO | [09:08] Diego |
| FDD-MODELO-04 | docs/FDD.md | Modelo de dados | `WebhookDeadLetter`: payload, motivo da falha e timestamp, em tabela separada | TRANSCRICAO | [09:18] Diego |

#### Fluxos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Criação do evento na outbox via `publishWebhookEvent(tx, …)`, dentro da transação | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Ciclo do worker: seleção, reivindicação, envio em série e marcação | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff 1m/5m/30m/2h/12h e a tabela de execução por tentativa | TRANSCRICAO | [09:17] Larissa |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Dead letter na sexta falha e replay manual que recoloca a linha como pendente | TRANSCRICAO | [09:18] Diego |

#### Contratos públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /api/v1/webhooks` — request, `201` com a secret devolvida uma única vez | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /api/v1/webhooks?customerId=` — envelope paginado, sem a secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id` — campos opcionais, `200` sem a secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id` — `204`, com `409` quando há histórico referenciando | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` — janela de 100 entregas em envelope paginado | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `POST /api/v1/webhooks/:id/rotate-secret` — comportamento com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — `202`, `403` para `OPERATOR` | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Contrato de saída: o `POST` que o cliente recebe, com corpo e cinco headers | TRANSCRICAO | [09:44] Diego |

#### Headers e payload do evento

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-HEADER-01 | docs/FDD.md | Contrato | `X-Event-Id` — UUID do evento, estável entre retentativas | TRANSCRICAO | [09:25] Diego |
| FDD-HEADER-02 | docs/FDD.md | Contrato | `X-Signature` — HMAC-SHA256 do corpo do request | TRANSCRICAO | [09:20] Sofia |
| FDD-HEADER-03 | docs/FDD.md | Contrato | `X-Timestamp` — instante do envio, para o cliente detectar replay | TRANSCRICAO | [09:44] Diego |
| FDD-HEADER-04 | docs/FDD.md | Contrato | `X-Webhook-Id` — id do cadastro, para clientes com vários endpoints | TRANSCRICAO | [09:44] Sofia |
| FDD-HEADER-05 | docs/FDD.md | Contrato | `Content-Type: application/json` | TRANSCRICAO | [09:44] Diego |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Payload com nove campos, `event_type` fixo e timestamp ISO 8601, sem os itens | TRANSCRICAO | [09:43] Diego |
| FDD-HMAC-01 | docs/FDD.md | Decisão | Cálculo do HMAC-SHA256 sobre o corpo exato enviado, com `node:crypto` | TRANSCRICAO | [09:20] Sofia |

#### Matriz de erros

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-ERRO-01 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` (404) — assinatura inexistente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` (400) — guarda de serviço sobre a url | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` (400) — código literal, sem caminho de disparo | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` (422) — snapshot acima de 64KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-05 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404) — `:id` de replay inexistente | TRANSCRICAO | [09:35] Diego |
| FDD-ERRO-W-01 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` — sem resposta completa em 10s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-W-02 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_HTTP_ERROR` — resposta fora da faixa 2xx | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-W-03 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_CONNECTION_ERROR` — DNS, TLS ou conexão recusada | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-W-04 | docs/FDD.md | Erro | `WEBHOOK_ATTEMPTS_EXHAUSTED` — qualificador de DLQ: seis chamadas consumidas | TRANSCRICAO | [09:17] Larissa |

#### Observabilidade, integração e configuração

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-LOG-01 | docs/FDD.md | Observabilidade | `webhook_replay_requested` com `userId` — auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Ponto de integração | `changeStatus`: chamada a `publishWebhookEvent` entre as linhas 167 e 169, no `tx` | CODIGO | src/modules/orders/order.service.ts (linha 126) |
| FDD-INT-02 | docs/FDD.md | Ponto de integração | `create`: segundo ponto de disparo potencial, não prescrito | CODIGO | src/modules/orders/order.service.ts (linha 50) |
| FDD-INT-03 | docs/FDD.md | Ponto de integração | `buildApiRouter` ganha dois `router.use`; `/admin` não existe hoje na API | CODIGO | src/routes/index.ts (linha 21) |
| FDD-INT-04 | docs/FDD.md | Ponto de integração | Injeção manual em `buildControllers`, no molde das linhas 42–44 | CODIGO | src/app.ts (linhas 26 e 55) |
| FDD-INT-05 | docs/FDD.md | Ponto de integração | `authenticate` no router e `requireRole('ADMIN')` no admin — arquivo não muda | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Ponto de integração | `validate` por rota carrega a regra `https` no schema Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Ponto de integração | Classes `Webhook*Error` estendem as especializações existentes de `AppError` | CODIGO | src/shared/errors/http-errors.ts (linhas 45 e 55) |
| FDD-INT-08 | docs/FDD.md | Ponto de integração | Logger Pino reusado pelos dois processos, com duas correções exigidas | CODIGO | src/shared/logger/index.ts |
| FDD-INT-09 | docs/FDD.md | Ponto de integração | Worker chama `createPrismaClient()` e não importa o singleton da linha 10 | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Ponto de integração | Listas usam `paginated<T>`; recurso individual volta sem envelope | CODIGO | src/shared/http/response.ts |
| FDD-CHK-01 | docs/FDD.md | Restrição | Script `worker` decidido na reunião e inexistente hoje em `package.json` | TRANSCRICAO | [09:11] Larissa |
| FDD-CHK-02 | docs/FDD.md | Restrição | `redactPaths` não cobre `secret` nem `previousSecret` — correção exigida | CODIGO | src/shared/logger/index.ts (linha 4) |
| FDD-CHK-03 | docs/FDD.md | Restrição | Quatro tabelas novas entram na lista de truncamento dos testes | CODIGO | tests/setup.ts |
| FDD-CHK-04 | docs/FDD.md | Restrição | Não há serviço de aplicação onde declarar o processo do worker | CODIGO | docker-compose.yml |
| FDD-ENV-01 | docs/FDD.md | Configuração | `WEBHOOK_POLL_INTERVAL_MS` = 2000 | TRANSCRICAO | [09:10] Larissa |
| FDD-ENV-02 | docs/FDD.md | Configuração | `WEBHOOK_DELIVERY_TIMEOUT_MS` = 10000 | TRANSCRICAO | [09:42] Diego |
| FDD-ENV-03 | docs/FDD.md | Configuração | `WEBHOOK_MAX_ATTEMPTS` = 5, retentativas além do envio inicial | TRANSCRICAO | [09:17] Larissa |
| FDD-ENV-04 | docs/FDD.md | Configuração | `WEBHOOK_PAYLOAD_MAX_BYTES` = 65536 | TRANSCRICAO | [09:24] Larissa |
| FDD-ENV-05 | docs/FDD.md | Configuração | `WEBHOOK_SECRET_GRACE_PERIOD_HOURS` = 24 | TRANSCRICAO | [09:21] Sofia |
| FDD-DEP-01 | docs/FDD.md | Restrição | Nenhuma dependência nova: reuso de Prisma, Zod, Pino e builtins | TRANSCRICAO | [09:30] Larissa |
| FDD-DEP-02 | docs/FDD.md | Restrição | Cliente HTTP com timeout resolvido por `fetch` global e `AbortSignal.timeout()` | CODIGO | package.json (`engines.node >= 20`) |
| FDD-DEP-03 | docs/FDD.md | Restrição | `pino-http` está declarado e não é usado — não há padrão de logging HTTP a seguir | CODIGO | package.json (linha 31) |
| FDD-ABERTO-01 | docs/FDD.md | Questão em aberto | `DEC-09` × `TEC-16`: a recusa de `http` sai como `VALIDATION_ERROR`, não `WEBHOOK_INVALID_URL` | TRANSCRICAO | [09:23] Sofia |
| FDD-ABERTO-02 | docs/FDD.md | Questão em aberto | `WEBHOOK_SECRET_REQUIRED` ficou inalcançável quando a secret passou a ser gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |

#### Critérios de aceite técnicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CA-01 | docs/FDD.md | Critério de aceite | Rollback de `changeStatus` não deixa linha na outbox | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de aceite | Status sem assinante não gera linha na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-03 | docs/FDD.md | Critério de aceite | Cliente com dois endpoints no mesmo status gera duas linhas com `eventId` distintos | TRANSCRICAO | [09:44] Sofia |
| FDD-CA-04 | docs/FDD.md | Critério de aceite | Payload tem exatamente os nove campos citados, sem `items` | TRANSCRICAO | [09:43] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de aceite | Payload acima de 64KB derruba a transação com 422 | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-06 | docs/FDD.md | Critério de aceite | `X-Signature` reproduzível com a secret sobre o corpo recebido | TRANSCRICAO | [09:22] Sofia |
| FDD-CA-07 | docs/FDD.md | Critério de aceite | Durante o grace period a secret antiga continua validando os envios | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-08 | docs/FDD.md | Critério de aceite | URL `http` é recusada no cadastro com 400 | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de aceite | Secret aparece na criação e na rotação, nunca em `GET` ou `PATCH` | TRANSCRICAO | [09:31] Marcos |
| FDD-CA-10 | docs/FDD.md | Critério de aceite | Falha agenda a retentativa no intervalo correto da tabela de backoff | TRANSCRICAO | [09:17] Diego |
| FDD-CA-11 | docs/FDD.md | Critério de aceite | Linha com próxima tentativa no futuro não é selecionada pelo ciclo | TRANSCRICAO | [09:17] Larissa |
| FDD-CA-12 | docs/FDD.md | Critério de aceite | Seis chamadas HTTP antes da dead letter | TRANSCRICAO | [09:15] Diego |
| FDD-CA-13 | docs/FDD.md | Critério de aceite | Timeout de 10s conta como falha | TRANSCRICAO | [09:42] Diego |
| FDD-CA-14 | docs/FDD.md | Critério de aceite | Esgotamento cria linha na dead letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-CA-15 | docs/FDD.md | Critério de aceite | Replay preserva o `eventId` e cria linha nova na outbox | TRANSCRICAO | [09:18] Diego |
| FDD-CA-16 | docs/FDD.md | Critério de aceite | Replay exige role `ADMIN`; `OPERATOR` recebe 403 | TRANSCRICAO | [09:36] Larissa |
| FDD-CA-17 | docs/FDD.md | Critério de aceite | Replay loga o `userId` de quem executou | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-18 | docs/FDD.md | Critério de aceite | Linha órfã em `PROCESSING` é recuperada sem incrementar as tentativas | TRANSCRICAO | [09:11] Diego |
| FDD-CA-19 | docs/FDD.md | Critério de aceite | Lote processado em série, em ordem de `createdAt` | TRANSCRICAO | [09:12] Diego |
| FDD-CA-20 | docs/FDD.md | Critério de aceite | `deliveries` devolve no máximo 100, mais recente primeiro, paginado | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-21 | docs/FDD.md | Critério de aceite | Toda tentativa gera um registro de entrega, com sucesso ou falha | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-22 | docs/FDD.md | Critério de aceite | Erros saem no formato `{ error: { code, message } }` com código `WEBHOOK_*` | TRANSCRICAO | [09:29] Bruno |

---

### 2.4 ADRs — `docs/adrs/`

Uma linha por decisão fechada. O identificador junta o ADR que a carrega e o item do índice que a registrou.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001-DEC-01 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Padrão outbox em MySQL como mecanismo de entrega — nem síncrono, nem Redis | TRANSCRICAO | [09:08] Larissa |
| ADR-001-DEC-15 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Payload gravado como snapshot na inserção, não renderizado no envio | TRANSCRICAO | [09:52] Larissa |
| ADR-001-DEC-16 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Filtragem por status aplicada na inserção, não no despacho | TRANSCRICAO | [09:34] Bruno |
| ADR-001-DEC-18 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Limite de 64KB por payload, com erro em vez de truncamento | TRANSCRICAO | [09:24] Larissa |
| ADR-001-DEC-19 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | UUID como identificador primário, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-002-DEC-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em polling com intervalo de 2 segundos, latência mínima aceita | TRANSCRICAO | [09:10] Larissa |
| ADR-002-DEC-03 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker roda como processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-DEC-14 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | `PrismaClient` próprio do worker, mesmo banco e mesma `DATABASE_URL` | TRANSCRICAO | [09:30] Bruno |
| ADR-002-DEC-17 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Módulo em `src/modules/webhooks` e entry point em `src/worker.ts` | TRANSCRICAO | [09:28] Bruno |
| ADR-003-DEC-05 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Decisão | Cinco tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DEC-06 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Decisão | DLQ persistida em tabela separada, não marcação na própria outbox | TRANSCRICAO | [09:18] Diego |
| ADR-003-DEC-07 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Decisão | Reprocessamento manual da DLQ por endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-003-DEC-12 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Decisão | Replay exige role `ADMIN`, reaproveitando o `requireRole` existente | TRANSCRICAO | [09:36] Larissa |
| ADR-004-DEC-08 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-DEC-09 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | TLS obrigatório: `http` recusado por validação Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-005-DEC-10 | docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md | Decisão | Entrega at-least-once, com dedup por `X-Event-Id` do lado do cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-006-DEC-11 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo do que já existe: `AppError`, Pino, middleware, módulos, Zod | TRANSCRICAO | [09:30] Larissa |
| ADR-006-DEC-20 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | CRUD de configuração acessível a qualquer role autenticada, por enquanto | TRANSCRICAO | [09:37] Sofia |
| ADR-007-DEC-04 | docs/adrs/ADR-007-ordering-por-order-id-sob-single-worker.md | Decisão | Ordering garantida só por `order_id` e só sob single-worker, como limitação | TRANSCRICAO | [09:13] Larissa |

---

## 3. Itens sem origem em fonte nenhuma

Os documentos do pacote contêm 41 itens estruturantes que **não têm fala na reunião nem respaldo no código**. Eles não aparecem na tabela da seção 2 porque não há Localização honesta a preencher — e é exatamente por isso que precisam estar em algum lugar. Dar timestamp a qualquer um deles para engordar o percentual da seção 2 seria o defeito que este documento existe para impedir.

### 3.1 Ledger do FDD — 21 propostas de implementação

Reproduzido de [FDD, "Premissas assumidas e derivações sem origem"](FDD.md#premissas-assumidas-e-derivações-sem-origem). O documento que as propõe é o único responsável por elas.

| ID | Documento | Tipo | Conteúdo (resumo) | Autoria | O que quebra se for rejeitado |
|---|---|---|---|---|---|
| FDD-P-01 | docs/FDD.md | Proposta | `attemptCount` e `nextAttemptAt` na outbox | FDD §5.3 | a política de retry deixa de ser executável |
| FDD-P-02 | docs/FDD.md | Proposta | Tabela `webhook_deliveries` inteira | FDD §5.4 | `RF-06` deixa de ser implementável |
| FDD-P-03 | docs/FDD.md | Proposta | `PENDING` cobre "aguardando retentativa"; `FAILED` é terminal | FDD §5.1 | a query de polling muda |
| FDD-P-04 | docs/FDD.md | Proposta | Um `eventId` por assinatura, não por mudança de status | FDD §6.1 | a dedup do cliente descartaria a entrega do segundo endpoint |
| FDD-P-05 | docs/FDD.md | Proposta | Duas assinaturas em `X-Signature` durante o grace period | FDD §8.2 | o grace period de 24h fica sem significado operacional |
| FDD-P-06 | docs/FDD.md | Proposta | Formato `sha256=<hex>`, header único com vírgula | FDD §8.2 | o contrato de saída muda |
| FDD-P-07 | docs/FDD.md | Proposta | Secret armazenada em claro, sem cifragem proposta | FDD §8.3 | decisão da revisão de segurança |
| FDD-P-08 | docs/FDD.md | Proposta | Recuperação de linhas órfãs: shutdown gracioso + lease | FDD §6.5 | eventos somem em todo deploy do worker |
| FDD-P-09 | docs/FDD.md | Proposta | Coluna `requestId` e sua propagação entre processos | FDD §5.3, §10.3 | não há correlação entre API e worker |
| FDD-P-10 | docs/FDD.md | Proposta | Nomes dos eventos de log | FDD §10.1 | nada estrutural |
| FDD-P-11 | docs/FDD.md | Proposta | Métricas como grandezas + query, sem exportador | FDD §10.2 | o alvo continua sem instrumentação |
| FDD-P-12 | docs/FDD.md | Proposta | Teto no `responseBody` armazenado | FDD §5.4 | linha de tabela cresce com dado de terceiro |
| FDD-P-13 | docs/FDD.md | Proposta | Janela de 100 + envelope paginado em `RF-06` | FDD §7.5 | a forma da resposta muda |
| FDD-P-14 | docs/FDD.md | Proposta | 3xx tratado como falha, sem seguir redirect | FDD §6.2 | payload assinado poderia ir a host não cadastrado |
| FDD-P-15 | docs/FDD.md | Proposta | `WEBHOOK_ROTATION_IN_PROGRESS` e `WEBHOOK_SUBSCRIPTION_IN_USE` | FDD §9.6.1 | dois caminhos ficam sem resposta definida |
| FDD-P-16 | docs/FDD.md | Proposta | `WEBHOOK_SUBSCRIPTION_INACTIVE` como motivo de DLQ | FDD §6.6 | evento de assinatura desativada fica sem destino |
| FDD-P-17 | docs/FDD.md | Proposta | `customerId` não editável no `PATCH` | FDD §7.3 | o histórico poderia trocar de dono |
| FDD-P-18 | docs/FDD.md | Proposta | `WEBHOOK_POLL_BATCH_SIZE` = 20 e `WEBHOOK_PROCESSING_LEASE_MS` = 60000 | FDD §12.2 | ajuste de tuning |
| FDD-P-19 | docs/FDD.md | Proposta | Nomes de tabela e dimensionamentos de coluna | FDD §5.2, §5.4 | nada estrutural |
| FDD-P-20 | docs/FDD.md | Proposta | Script `worker:start` | FDD §11.2 | a reunião só nomeou um script |
| FDD-P-21 | docs/FDD.md | Proposta | Prefixo `whsec_` na secret | FDD §7.1 | nada estrutural |

### 3.2 Caminho de endpoint proposto

| ID | Documento | Tipo | Conteúdo (resumo) | Autoria | Observação |
|---|---|---|---|---|---|
| RFC-API-06 | docs/RFC.md · docs/FDD.md §7.6 | Contrato | Caminho `POST /api/v1/webhooks/:id/rotate-secret` | RFC | Sofia especificou apenas "endpoint pro cliente conseguir pedir nova secret pela API" ([09:21]); o caminho é proposta. O **comportamento** tem fonte e está em `FDD-CONTRATO-06` |

### 3.3 Métricas, cenários e validações propostos pelo PRD

A reunião não produziu **nenhuma** métrica de produto: não há meta de adoção, KPI nem baseline. Tudo abaixo é proposta do PRD.

| ID | Documento | Tipo | Conteúdo (resumo) | Autoria | Observação |
|---|---|---|---|---|---|
| PRD-MP-01 | docs/PRD.md §4.2 | Métrica | Adoção: 3 de 3 clientes com endpoint ativo em 30 dias | PRD | o prazo de 30 dias é arbitrário |
| PRD-MP-02 | docs/PRD.md §4.2 | Métrica | Redução de polling em `GET /orders` ≥ 50% em 30 dias | PRD | **o baseline não existe**; medi-lo antes do go-live é pré-requisito |
| PRD-MP-03 | docs/PRD.md §4.2 | Métrica | Taxa de entrega na primeira tentativa ≥ 95% | PRD | número sem origem |
| PRD-MP-04 | docs/PRD.md §4.2 | Métrica | Volume de dead letter por semana, com tendência decrescente | PRD | sem alvo absoluto |
| PRD-MP-05 | docs/PRD.md §4.2 | Métrica | Latência percebida: p95 < 10s | PRD | herda o conflito de `RNF-01` |
| PRD-MP-06 | docs/PRD.md §4.2 | Métrica | Churn evitado: Atlas Comercial retida | PRD | o risco tem fonte; transformá-lo em métrica é proposta |
| PRD-CU-P1 | docs/PRD.md §3.3 | Cenário proposto | Cliente troca a URL sem perder eventos em trânsito | PRD | ninguém discutiu eventos enfileirados durante a troca |
| PRD-CU-P2 | docs/PRD.md §3.3 | Cenário proposto | Um quarto cliente quer aderir | PRD | nada na fonte trata de expansão |
| PRD-CU-P3 | docs/PRD.md §3.3 | Cenário proposto | Cliente quer testar a integração antes de produção | PRD | não há endpoint de teste em escopo |
| PRD-CU-P4 | docs/PRD.md §3.3 | Cenário proposto | Cliente descobre sozinho que parou de receber | PRD | sem email e sem painel, a única via é consultar o histórico |
| PRD-VP-01 | docs/PRD.md §12.3 | Validação proposta | Medir o baseline de polling antes do go-live | PRD | janela que fecha sozinha no go-live |
| PRD-VP-02 | docs/PRD.md §12.3 | Validação proposta | Piloto com um dos três clientes | PRD | troca velocidade por risco |
| PRD-VP-03 | docs/PRD.md §12.3 | Validação proposta | Exercício de indisponibilidade combinado com o cliente do piloto | PRD | exige cooperação do cliente |
| PRD-VP-04 | docs/PRD.md §12.3 | Validação proposta | Verificar que o cliente do piloto realmente deduplica | PRD | não escala para todos |
| PRD-VP-05 | docs/PRD.md §12.3 | Validação proposta | Acompanhamento das métricas nos 30 dias após o go-live | PRD | sem alarme, o acompanhamento é manual |
| PRD-VP-06 | docs/PRD.md §12.3 | Validação proposta | Reavaliar o aviso por email ao fim dos 30 dias | PRD | o gatilho existe na fala; a data é proposta |

### 3.4 Desambiguações feitas fora da reunião

Três itens em que a **pergunta** tem origem na transcrição e a **resposta** não: foram resolvidos no processo de produção dos documentos, e nenhum participante avalizou a escolha.

| ID | Documento | Tipo | Pergunta (com origem) | Resposta (sem origem) |
|---|---|---|---|---|
| RES-ABE-02 | docs/FDD.md §6.2 · ADR-002 | Desambiguação | Nome do arquivo de lógica do worker: `webhook.worker.ts` **ou** `webhook.processor.ts` — Bruno ofereceu as duas ([09:28]) | `webhook.worker.ts`, convivendo com `src/worker.ts` como entry point |
| RES-ABE-03 | docs/FDD.md §7 · ADR-006 | Desambiguação | Onde trafega o `customer_id`: body **ou** path — Larissa não escolheu ([09:32]) | body no `POST`, query no `GET`, pela convenção real de `order.schemas.ts` |
| RES-TEC-24 | docs/FDD.md §6.3 · ADR-003 | Desambiguação | "5 tentativas" inclui o envio inicial? Nunca foi dito ([09:15] / [09:17]) | cinco retentativas além do envio inicial, seis chamadas HTTP no total |

---

## 4. Varredura reversa — cobertura da transcrição

A direção inversa da seção 2: dos **112 itens** indexados na reunião, quais chegaram a algum documento. Sem isto, o tracker provaria apenas que o que foi escrito tem origem — não que o que foi dito não se perdeu.

Método: busca literal pelo identificador de cada item nos cinco documentos do pacote.

| Categoria | Itens | Aparecem em ≥1 documento | Órfãos |
|---|---|---|---|
| `DEC` — decisões fechadas | 20 | 20 | **0** |
| `RF` — requisitos funcionais | 11 | 11 | **0** |
| `RNF` — requisitos não funcionais | 10 | 10 | **0** |
| `RES` — restrições | 7 | 7 | **0** |
| `ALT` — alternativas descartadas | 8 | 8 | **0** |
| `ADI` — itens adiados | 6 | 6 | **0** |
| `ABE` — questões em aberto | 5 | 5 | **0** |
| `COD` — ganchos com o código | 16 | 16 | **0** |
| `TEC` — detalhes técnicos secundários | 29 | 29 | **0** |
| **Total** | **112** | **112** | **0** |

**Nenhum item da reunião ficou de fora do pacote.** Os dois itens com cobertura mais rasa — presentes em um único documento — são `TEC-03` (polling de 2s como implementação, só em ADR-002) e `COD-15` (esquema de códigos de erro com prefixo por domínio, só em ADR-006). Ambos têm o conteúdo repetido sem o rótulo em outros documentos; a cobertura é real, o rótulo é que não viajou.

**Itens deliberadamente registrados como exclusão, não como requisito** — a varredura confirma que nenhum deles virou requisito em documento nenhum: `ADI-01` (email), `ADI-02` (rate limiting), `ADI-03` (dashboard), `ADI-04` (arquivamento), `ADI-05` (múltiplos workers), `ADI-06` (endurecer autorização), `ALT-01` a `ALT-08` (as oito alternativas descartadas). Todos aparecem apenas em seções de escopo, de alternativas ou de consequência.

---

## 5. Achados da varredura

Divergências encontradas ao conferir os documentos contra `TRANSCRICAO.md` e contra o repositório. **Nenhuma foi corrigida por esta passada** — uma auditoria que edita o que está auditando deixa de ser auditoria. A tabela da seção 2 usa os valores corretos; os documentos de origem seguem como estão, à espera de decisão.

### A-01 — Cinco linhas do índice apontam `[09:29]` para uma fala que é `[09:28]`

`COD-04`, `COD-05`, `COD-06`, `COD-15` e `TEC-16` citam a fala de Bruno sobre `AppError`, `InsufficientStockError`, `InvalidStatusTransitionError` e os códigos `WEBHOOK_*`. Em `TRANSCRICAO.md` essa fala está em **`[09:28]`**; `[09:29]` é a fala seguinte, sobre o logger Pino e o middleware de erro (que é a origem correta de `COD-07` e `COD-08`).

**Propagação:** o FDD herdou o timestamp errado em §9.6 ("`TEC-16` nomeia três códigos… Bruno `[09:29]`") e na ressalva 2, que argumenta que o código foi nomeado "dois minutos antes" de `TEC-14` — com o timestamp correto, são **três** minutos. O ADR-006 também cita `COD-04` e `COD-15` como `[09:29]`.
**Severidade:** baixa. O falante, o conteúdo e a conclusão continuam corretos; erra o minuto.

### A-02 — `ABE-05` cita `[09:47]` para uma frase dita em `[09:49]`

A citação literal "Tá bom. Eu atualizo os clientes hoje à tarde" é de **`[09:49]` Marcos**. Em `[09:47]` Marcos diz outra coisa — "Atlas vai gostar. Eu confirmo prazo com eles" —, que sustenta o mesmo fato por outra via.

**Propagação:** PRD §2.2 e a dependência `D-04` citam `[09:47]` com a frase de `[09:49]`; a nota de leitura (b)-1 do índice tem o mesmo defeito.
**Severidade:** baixa. Duas falas diferentes registram a mesma promessa; o tracker usa `[09:49]`, que é onde a citação está.

### A-03 — `RES-04` funde duas datas e cita a fala errada

A linha `RES-04` do índice resume "entrega até **fim de novembro**" mas cita `[09:00]`, onde Marcos diz "fim do **trimestre**". "Fim de novembro" é literal de **`[09:45]` Marcos**.

**Propagação:** o PRD já **detectou e registrou** esta divergência em §2.2 e no risco `R-02`, adotando `[09:45]` como referência de prazo — este achado confirma a leitura do PRD contra a transcrição. RFC e ADR-004/006 citam `RES-04` genericamente, sem afirmar data.
**Severidade:** média no índice, nula nos documentos — o único que precisava da data já separou as duas.

### A-04 — As seções "Fontes" declaram faixas de IDs que o corpo não cita

As três grandes peças fecham com listas em faixa contígua ("`COD-01 a COD-16`", "`TEC-01 a TEC-29`"). A varredura literal por identificador encontra IDs declarados que não aparecem no corpo do respectivo documento:

| Documento | IDs declarados e não citados no corpo | Quais |
|---|---|---|
| docs/PRD.md | 9 | `DEC-03`, `DEC-08`, `DEC-09`, `DEC-14`, `DEC-15`, `DEC-17`, `DEC-19`, `RES-02`, `RES-05` |
| docs/RFC.md | 10 | `DEC-09`, `RF-09`, `RF-10`, `RNF-02`, `RNF-03`, `RNF-05`, `RNF-07`, `RNF-08`, `RNF-09`, `RES-02` |
| docs/FDD.md | 14 | `DEC-13`, `RNF-03`, `RNF-07`, `RNF-08`, `RNF-09`, `RES-06`, `ALT-02`, `ALT-03`, `ALT-04`, `ALT-05`, `COD-02`, `COD-03`, `COD-15`, `TEC-03` |

Em quase todos os casos o **conteúdo** está no documento sem o rótulo — o FDD trata de HTTPS obrigatório (`RNF-07`) e de ordem sob single-worker (`RNF-08`) longamente, só não usa os IDs. É over-declaração de bibliografia, não invenção no corpo.
**Severidade:** baixa. Correção: trocar faixa por enumeração literal nas três seções "Fontes".

### A-05 — Um timestamp em faixa sobreviveu no ADR-002

ADR-002, item 3 da Decisão, cita "(`DEC-02` / `TEC-03`, `[09:09]`–`[09:10]`)". Faixa não é timestamp válido no formato exigido; as duas falas existem separadamente — `[09:09]` Diego (a proposta de polling) e `[09:10]` Larissa (o fechamento).
**Severidade:** baixa, e é o mesmo defeito de formato que já havia sido corrigido no índice.

### A-06 — Nenhum caminho de arquivo inexistente citado como existente

Os cinco documentos citam **31 caminhos** distintos. **27 são de arquivos que existem hoje, e todos os 27 foram verificados um a um no repositório.** Os outros quatro — `src/worker.ts`, `src/modules/webhooks/webhook.worker.ts`, `tests/webhooks.test.ts` e `tests/webhook-worker.test.ts` — não existem, e **os documentos os nomeiam explicitamente como arquivos a criar** (o FDD §13 chega a rotulá-los "arquivos prescritos, não criados"). Não é o defeito que o critério de consistência procura: nenhum documento afirma que um arquivo inexistente já está no repositório.

Os números de linha citados pelo FDD também conferem — `changeStatus` (126), `create` (50), `tx.order.update` (158), `tx.orderStatusHistory.create` (159, bloco fechando em 167), releitura do pedido (169), `delete` (181), `'INVALID_ORDER_STATE_FOR_DELETE'` (187), `buildApiRouter` (21), `Controllers` (13), `buildControllers` (26), `buildApp` (55), `app.use('/api/v1')` (67), `router.use(authenticate)` (14), `InvalidStatusTransitionError` (45), `InsufficientStockError` (55), singleton `prisma` (10).

Confirmadas também as três afirmações negativas mais fáceis de errar: `package.json` **não** tem script `worker`; `redactPaths` em `src/shared/logger/index.ts` **não** cobre `secret`; `buildApiRouter` **não** monta nenhuma rota `/admin`.

### Resumo dos achados

| # | Achado | Onde nasce | Severidade | Ação sugerida |
|---|---|---|---|---|
| A-01 | `[09:29]` no lugar de `[09:28]` em 5 itens | índice de fontes | Baixa | corrigir índice; ajustar "dois minutos" para "três" na ressalva 2 do FDD |
| A-02 | `[09:47]` no lugar de `[09:49]` em `ABE-05` | índice de fontes | Baixa | corrigir índice, PRD §2.2 e `D-04` |
| A-03 | `RES-04` funde duas datas | índice de fontes | Média | separar a linha do índice em duas |
| A-04 | Faixas de ID declaradas e não citadas | PRD, RFC, FDD | Baixa | enumerar literalmente nas seções "Fontes" |
| A-05 | Timestamp em faixa | ADR-002 | Baixa | usar `[09:09]` e `[09:10]` separados |
| A-06 | — | — | — | nenhuma ação: caminhos e linhas conferem |

**Os três primeiros achados nascem no índice, não nos documentos** — e todos os três são do mesmo tipo: a citação literal está certa, o carimbo de origem é que escorregou. É a razão pela qual esta varredura conferiu cada timestamp na transcrição em vez de confiar no índice.

---

## 6. Conformidade com os critérios de aceite

| Critério | Alvo | Resultado |
|---|---|---|
| Formato da tabela | `ID · Documento · Tipo · Conteúdo · Fonte · Localização` | ✅ seis colunas, em todas as tabelas da seção 2 |
| Cobertura dos itens dos documentos | ≥ 80% | ✅ **100%** dos 275 itens próprios; **83%** contando os 56 restatements |
| Linhas com `Fonte = TRANSCRICAO` e timestamp `[hh:mm] Nome` | ≥ 70% | ✅ **208 de 234 = 88,9%** |
| Linhas com `Fonte = CODIGO` e caminho real | ≥ 5 | ✅ **26 linhas**, cobrindo 14 arquivos distintos, todos verificados |
| Timestamps válidos | formato `[hh:mm]`, sem faixas | ✅ 208 de 208; os 61 pares `[hh:mm] Nome` distintos foram conferidos contra `TRANSCRICAO.md` |
| Caminhos de arquivo existentes | todos | ✅ 27 de 27; os outros 4 caminhos citados são declarados como arquivos a criar (A-06) |

**Distribuição das 234 linhas da seção 2**

| Documento | Linhas | `TRANSCRICAO` | `CODIGO` |
|---|---|---|---|
| docs/PRD.md | 97 | 93 | 4 |
| docs/RFC.md | 32 | 26 | 6 |
| docs/FDD.md | 86 | 70 | 16 |
| docs/adrs/ | 19 | 19 | 0 |
| **Total** | **234** | **208** | **26** |

**Falantes citados** — os cinco participantes aparecem como origem: Diego, Larissa, Bruno, Marcos e Sofia.

---

## Fontes deste documento

`TRANSCRICAO.md` (fonte primária de todo timestamp) · `.scratch/fontes/transcricao-index.md` (índice dos 112 itens, usado como roteiro e auditado contra a transcrição) · [PRD](PRD.md) · [RFC](RFC.md) · [FDD](FDD.md) · [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) a [ADR-007](adrs/ADR-007-ordering-por-order-id-sob-single-worker.md) · repositório: os 27 caminhos existentes verificados em `src/`, `prisma/`, `tests/`, `package.json` e `docker-compose.yml`, mais os 4 declarados como arquivos a criar.
