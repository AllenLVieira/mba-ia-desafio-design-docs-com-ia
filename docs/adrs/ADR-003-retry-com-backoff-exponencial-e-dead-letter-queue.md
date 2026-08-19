# ADR-003 — Retry com backoff exponencial e dead letter queue

## Status

**Accepted** — retry fechado por Larissa (DEC-05, `[09:17]`); DLQ em tabela separada proposta e aceita (DEC-06, Diego `[09:18]`); replay manual via endpoint admin (DEC-07, `[09:18]`).

> **Nota de contexto — revisão pendente.** A sessão de revisão do documento de design (`[09:50]`) ainda não ocorreu.
>
> **Nota de contexto — semântica de "5 tentativas".** DEC-05 fecha "5 tentativas" sem dizer se o envio inicial conta. A desambiguação está registrada na seção Decisão, item 2, com a aritmética que a sustenta.

## Contexto

Clientes B2B ficam indisponíveis, e não brevemente. Diego trouxe o caso concreto: "Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada" (ALT-04, `[09:16]`).

O extremo oposto também foi rejeitado: "Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu" (ALT-05, Diego `[09:15]`).

O worker de [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) precisa de uma política que cubra manutenção planejada sem reter eventos para sempre, e de um destino final para o que não entregou.

## Decisão

**Backoff exponencial com número fixo de retentativas, seguido de dead letter queue em tabela própria e replay manual por administrador.**

1. **Definição de falha**: resposta de erro do cliente **ou** timeout de 10 segundos (RNF-06 / TEC-25, Diego `[09:42]`): "Cliente lento que não responde em 10s a gente trata como falha e marca pra retry."

2. **Cinco retentativas além do envio inicial — seis chamadas HTTP no total.** Intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas (DEC-05 / TEC-23, `[09:17]`).

   TEC-24 registra a ambiguidade: nunca foi dito se as "5 tentativas" incluem o envio inicial. **A aritmética das fontes resolve.** A soma dos cinco intervalos citados em TEC-23 é 1m + 5m + 30m + 2h + 12h = **14h36min**, e Diego descreve o resultado como "quase 15 horas entre primeira falha e última tentativa" (RNF-02, `[09:17]`). Isso só fecha se os cinco intervalos forem consumidos *depois* da primeira falha. Na leitura alternativa — 5 tentativas no total, isto é, envio inicial mais 4 retentativas — apenas quatro intervalos seriam usados, totalizando 2h36min, o que contradiz a fala citada. Reforça a leitura escolhida o próprio texto de DEC-05: "5 tentativas **de retry**".

3. **Esgotadas as seis chamadas, o evento vai para `webhook_dead_letter`** — tabela separada, não marcação na outbox (DEC-06, Diego `[09:18]`). Campos: payload, motivo da falha e timestamp (TEC-12).

4. **Reprocessamento manual via endpoint admin** `POST /admin/webhooks/dead-letter/:id/replay` (DEC-07 / RF-08 / TEC-22, `[09:18]`). O replay "recoloca na outbox como pendente" (TEC-28), reiniciando o ciclo de entrega.

5. **Autorização e auditoria do replay**: role `ADMIN` obrigatória, reaproveitando o `requireRole` existente (DEC-12 / COD-09, Larissa `[09:36]`), e registro de quem executou o replay (RF-10, Sofia `[09:36]`: "o endpoint de admin tem que logar quem fez o replay, pra auditoria").

## Alternativas Consideradas

**3 retentativas** (ALT-04, Bruno `[09:16]`: "3 não é melhor? Mais agressivo."). Descartada por Diego com o argumento de cobertura: três tentativas em 30 minutos não sobrevivem a uma janela de manutenção planejada de duas horas. Larissa fechou em 5 (`[09:17]`).

**Retry indefinido com backoff** (ALT-05, Diego `[09:15]`). Descartada: evento pendurado para sempre quando o cliente desaparece.

**Marcar eventos falhos na própria `webhook_outbox`** (ALT-06, Diego `[09:18]`). Descartada em favor da tabela separada: "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento."

**Replay automático da DLQ.** Não foi cogitado; Diego propôs manual e não houve contraproposta (DEC-07).

## Consequências

### Positivas

- **A janela de 14h36min cobre o caso real que motivou a decisão.** Uma manutenção planejada de duas horas é absorvida com folga, e ainda sobra a tentativa de 12h para incidentes longos (RNF-02).
- **A outbox permanece legível.** Separar a DLQ mantém a query de polling do worker operando só sobre linhas vivas (ALT-06 / TEC-01).
- **A DLQ é evidência.** Payload, motivo e timestamp preservados permitem diagnosticar por que a entrega falhou sem depender de log rotacionado (TEC-12).
- **Replay reusa autorização existente.** `requireRole('ADMIN')` já está no código; nada novo a construir para proteger o endpoint (COD-09).

### Negativas

- **A latência de pior caso é quatro ordens de grandeza acima do alvo.** RNF-01 pede menos de 10 segundos; um evento entregue na sexta tentativa chega quase 15 horas depois do fato. O alvo de "tempo real" descreve o caminho feliz, não o contrato de entrega. A reunião não formalizou essa distinção.
- **O último intervalo é longo demais para diagnóstico.** Entre a quinta e a sexta tentativa passam 12 horas. Um cliente que corrigiu o próprio endpoint às 10h da manhã pode só receber o evento retido às 22h — sem forma de acelerar, já que o replay só existe *depois* da DLQ, não durante o retry.
- **Trade-off explícito: cobertura de indisponibilidade contra prontidão de entrega.** ALT-04 (3 tentativas, ~30min) entregaria mais rápido ou mataria mais cedo, dando sinal claro ao cliente; a escolha por 5 privilegia não perder o evento, ao custo de mantê-lo em trânsito por mais de meio dia.
- **O replay é manual e não escala.** Se um cliente ficar fora por mais de 15 horas, todo evento acumulado no período cai na DLQ individualmente, e cada um exige uma chamada `POST .../:id/replay` separada (TEC-22). Não há operação em lote decidida.
- **Payload duplicado entre outbox e DLQ.** TEC-12 guarda o payload de novo na dead letter, somado ao snapshot já retido pela outbox ([ADR-001](ADR-001-outbox-transacional-no-mysql.md), DEC-15) — e ADI-04 deixou o arquivamento fora de escopo nas duas tabelas.
- **Seis chamadas por evento, sem freio.** Um cliente com muitos pedidos e endpoint instável gera até 6× o volume normal de requisições saindo da plataforma, e o rate limiting ficou sem decisão (ABE-01 / ADI-02).
- **O replay reinicia o ciclo completo.** TEC-28 recoloca como pendente; nada foi decidido sobre limitar quantas vezes um mesmo evento pode voltar da DLQ, o que permite laço manual indefinido.

## Pontos em aberto

- **ABE-01 / ADI-02 — rate limiting de envios.** Diego pediu para registrar como ponto em aberto (`[09:39]`), Larissa reclassificou como "observar e decidir depois". Não há decisão de implementar nem de descartar.
- **ABE-04 — schema do histórico de entregas.** Marcos especificou o comportamento de `GET /webhooks/:id/deliveries` (RF-06, `[09:34]`): últimos 100 envios com sucesso/falha, payload, response e tempo de resposta. Nenhuma tabela ou estrutura foi projetada na reunião, e é este ADR que decide o que gera esses registros (as até seis tentativas por evento). Sem esse schema, RF-06 não é implementável.
- **ADI-01 — notificação por email em falha repetida.** Fora de escopo desta fase (Larissa `[09:37]`): "Talvez próxima fase, depois que a gente medir o impacto." Sem ela, o cliente só descobre que caiu na DLQ se consultar RF-06.
- **Mecanismo de agendamento do retry.** Como o worker sabe que chegou a hora da próxima tentativa não foi tratado na transcrição — TEC-01 só menciona índices em status e `created_at`. Este ADR registra a política, não o schema.

## Fontes

**Índice da transcrição** — enumeração literal dos IDs citados no corpo deste ADR: DEC-05, DEC-06, DEC-07, DEC-12, DEC-15 · RF-06, RF-08, RF-10 · RNF-01, RNF-02, RNF-06 · ALT-04, ALT-05, ALT-06 · ADI-01, ADI-02, ADI-04 · ABE-01, ABE-04 · COD-09 · TEC-01, TEC-12, TEC-22, TEC-23, TEC-24, TEC-25, TEC-28.

**Arquivos reais:** `src/middlewares/auth.middleware.ts` (`requireRole`), `src/routes/index.ts` (`buildApiRouter`), `src/shared/logger/index.ts` (`logger`).

**Relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) (quem executa as tentativas), [ADR-005](ADR-005-entrega-at-least-once-com-event-id.md) (por que retentar produz duplicata), [ADR-007](ADR-007-ordering-por-order-id-sob-single-worker.md) (efeito do retry sobre a ordem).
