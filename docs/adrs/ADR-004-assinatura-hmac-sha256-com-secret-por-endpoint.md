# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint

## Status

**Accepted** — decisão fechada por Sofia, Engenheira de Segurança (DEC-08, `[09:22]`).

> **Nota de contexto — revisão pendente.** Além da sessão de revisão de design (`[09:50]`), esta decisão tem uma segunda revisão pendente e **bloqueante**: Sofia reservou no mínimo dois dias úteis para revisar o código de segurança antes do deploy, com foco em HMAC e geração de secret (RNF-10 / RES-06, `[09:46]` e `[09:49]`).

## Contexto

O cliente recebe um POST de origem alegada na nossa plataforma. Sem prova criptográfica, qualquer um que descubra a URL do endpoint pode forjar mudanças de status de pedido.

Sofia estabeleceu o mecanismo: "A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature. Cliente verifica do lado dele" (RF-09, `[09:20]`).

E o raio de comprometimento: "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo" (RNF-09, `[09:21]`).

## Decisão

**HMAC-SHA256 sobre o corpo do request, com secret única por endpoint cadastrado, rotacionável com grace period de 24 horas, sobre TLS obrigatório** (DEC-08, Sofia `[09:22]`).

1. **Algoritmo: HMAC-SHA256**, especificamente (TEC-10, Sofia `[09:20]`): "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso."

2. **Escopo da assinatura: o corpo do request** (DEC-08). Transmitida no header `X-Signature` (TEC-06). O contrato completo de headers está em [ADR-005](ADR-005-entrega-at-least-once-com-event-id.md).

3. **Uma secret por endpoint de webhook**, não uma secret global da plataforma (RNF-09). O vazamento de uma secret compromete um endpoint de um cliente, não a base inteira.

4. **Secret gerada pelo sistema**, não fornecida pelo cliente, e devolvida na resposta de criação do webhook (TEC-14 / RF-01, Marcos `[09:31]`: "secret é gerada pela gente e devolvida na criação").

5. **Rotação por endpoint da API, com grace period de 24 horas** (RF-07 / TEC-11 / DEC-08, Sofia `[09:21]`): "Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre."

6. **TLS obrigatório**: a URL cadastrada deve ser `https`. URL `http` é recusada com erro de validação no schema Zod (DEC-09 / RNF-07 / TEC-26, Sofia `[09:23]`). Sofia classificou este ponto como validação, não decisão arquitetural — está registrado aqui por pertencer ao mesmo modelo de ameaça.

## Alternativas Consideradas

**Secret global da plataforma** (RNF-09, Sofia `[09:21]`). Descartada pelo raio de comprometimento: "Senão se vaza uma, vaza tudo."

**Outro algoritmo de HMAC que não SHA-256** (TEC-10, Sofia `[09:20]`). Descartado por interoperabilidade: SHA-256 é o que as bibliotecas dos clientes já suportam.

**Aceitar endpoints `http`** (DEC-09). Descartado: sem TLS, o payload trafega em claro e a assinatura protege integridade mas não confidencialidade.

**Não consideradas na reunião:** assinatura assimétrica (o cliente verificaria com chave pública, sem segredo compartilhado) e mTLS. Nenhuma das duas foi levantada por qualquer participante.

## Consequências

### Positivas

- **Comprometimento isolado por endpoint.** Uma secret vazada afeta um cadastro, e a rotação é a resposta pronta (RNF-09 + RF-07).
- **Barreira de adoção baixa.** HMAC-SHA256 tem biblioteca em toda linguagem relevante; o cliente não precisa de infraestrutura de chaves (TEC-10).
- **Rotação sem downtime do cliente.** As 24 horas de sobreposição permitem migrar sistemas sem janela de indisponibilidade nem coordenação com a nossa equipe (TEC-11).
- **Geração pelo sistema elimina secret fraca.** O cliente não escolhe a entropia (TEC-14).

### Negativas

- **A revisão de segurança é bloqueante e cara em cronograma.** RNF-10 exige no mínimo dois dias úteis de Sofia antes do deploy, e RES-06 torna isso condição para subir — dentro de um prazo total de três sprints (DEC-13) e sob pressão comercial de RES-04. **Trade-off explícito:** construir HMAC próprio em vez de terceirizar autenticação custa dois dias úteis de revisão especializada no caminho crítico da entrega.
- **Duas secrets válidas simultaneamente durante 24 horas.** É exatamente o que RF-07 pede, e é também uma janela em que uma secret comprometida continua aceita. Não foi decidido se a rotação pode ser forçada com revogação imediata em caso de incidente — o texto de TEC-11 só descreve o caminho de migração planejada.
- **A secret devolvida na criação transita e é exibida uma vez.** TEC-14 estabelece o retorno em claro na resposta de criação. Nada foi decidido sobre não poder recuperá-la depois, nem sobre redação em log — o logger existente (`src/shared/logger/index.ts`) redige `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, mas **nenhum padrão que cubra `secret`**.
- **Assinar apenas o corpo deixa os headers fora da proteção.** `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id` (TEC-05/07/08) trafegam sem cobertura da assinatura. O `X-Timestamp` é oferecido ao cliente para detectar replay (TEC-07), mas, não estando assinado, pode ser alterado por quem intercepte — a proteção efetiva contra replay vem do TLS obrigatório (DEC-09), não do header.
- **Verificação é responsabilidade inteiramente do cliente.** Se ele não validar a assinatura, o mecanismo não protege nada, e não temos como saber se ele valida.

## Pontos em aberto

- **Armazenamento da secret em repouso.** A transcrição não trata de como a secret é guardada no banco. TEC-13 lista a tabela de configuração como "url + secret + customer_id + estado ativo", sem qualificar o formato de armazenamento. Item candidato à revisão de segurança de RNF-10, onde Sofia declarou querer olhar "HMAC e geração de secret" com calma.
- **Redação de `secret` no logger.** A lista de redação em `src/shared/logger/index.ts` não cobre o campo; não foi discutido.
- **Revogação imediata em incidente.** TEC-11 define a rotação planejada; o caminho de emergência (matar a secret antiga na hora) não foi decidido.
- **Agendamento da revisão de segurança.** Sofia pediu para ser agendada (`[09:49]`); ninguém confirmou data ou responsável (nota de leitura (b)-3).

## Fontes

**Índice da transcrição** — enumeração literal dos IDs citados no corpo deste ADR: DEC-08, DEC-09, DEC-13 · RF-01, RF-07, RF-09 · RNF-07, RNF-09, RNF-10 · RES-04, RES-06 · TEC-05, TEC-06, TEC-07, TEC-10, TEC-11, TEC-13, TEC-14, TEC-26.

**Arquivos reais:** `src/middlewares/validate.middleware.ts` (`validate`), `src/modules/orders/order.schemas.ts` (padrão de schema Zod), `src/shared/logger/index.ts` (`logger`, lista de `redact`), `src/config/env.ts` (`envSchema`).

**Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-event-id.md) (contrato completo de headers), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) (validação Zod e códigos de erro `WEBHOOK_*`).
