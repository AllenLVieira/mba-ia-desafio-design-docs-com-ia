# ADR-002 — Worker em processo separado com polling

## Status

**Accepted** — worker como processo separado decidido por Diego (DEC-03, `[09:11]`); polling de 2 segundos fechado por Larissa (DEC-02, `[09:10]`).

> **Nota de contexto — revisão pendente.** A sessão de revisão do documento de design com Bruno e Diego, prometida por Larissa (`[09:50]`), ainda não ocorreu e não teve data definida.

## Contexto

A outbox decidida em [ADR-001](ADR-001-outbox-transacional-no-mysql.md) registra a intenção de entrega, mas não entrega. Alguém precisa ler as linhas pendentes e fazer a chamada HTTP.

O alvo de latência vem do produto: "Pra eles, qualquer coisa abaixo de 10 segundos já é 'tempo real'. O importante é que não fique pendurado" (RNF-01, Marcos `[09:02]`).

Duas restrições de plataforma moldam a solução:

- **O MySQL não notifica processos externos.** "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL" (ALT-03, Diego `[09:09]`).
- **O ciclo de vida da API não pode governar o do worker.** "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker" (DEC-03 / RES-05, Diego `[09:11]`).

## Decisão

**Um worker em processo Node separado, que consome a outbox por polling a cada 2 segundos.**

1. **Processo separado da API** (DEC-03, RES-05). Entry point novo `src/worker.ts`, análogo ao `src/server.ts` já existente, iniciado por um script `npm run worker` (TEC-20 / COD-10, Larissa `[09:11]`).

2. **Dois artefatos distintos, nomes propositalmente próximos:**

   | Arquivo | Papel |
   |---|---|
   | `src/worker.ts` | *Entry point do processo* — equivalente a `src/server.ts` para a API |
   | `src/modules/webhooks/webhook.worker.ts` | *Lógica de processamento* — vive dentro do módulo, equivalente a um `*.service.ts` |

   Bruno propôs a estrutura deixando o nome do segundo arquivo em aberto entre `webhook.worker.ts` e `webhook.processor.ts` (ABE-02, `[09:28]`); a escolha por `webhook.worker.ts` foi feita fora da reunião, na sessão de fechamento dos ADRs. **A proximidade dos nomes é conhecida e aceita** — `src/worker.ts` orquestra o processo, `webhook.worker.ts` contém a regra.

3. **Polling em loop, intervalo de 2 segundos** (DEC-02 / TEC-03, `[09:09]`–`[09:10]`): "A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca." A latência mínima de 2 segundos no pior caso foi explicitamente aceita por Larissa.

4. **Leitura em batch pequeno, ordenada por `created_at`** (TEC-02, Diego `[09:08]`): "Worker lê só os pendentes em batch pequeno, processa, marca como entregue."

5. **`PrismaClient` próprio do worker** (DEC-14 / COD-12, Bruno `[09:30]`): "PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node." Mesma `DATABASE_URL` e mesma stack Prisma do projeto (COD-16, RES-02).

6. **Módulo em `src/modules/webhooks`**, seguindo o padrão dos demais módulos (DEC-17, Bruno `[09:28]`) — detalhado em [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

## Alternativas Consideradas

**Worker dentro da mesma instância da API.** Descartada por acoplamento de ciclo de vida: reiniciar a API mataria o worker (DEC-03 / RES-05, Diego `[09:11]`).

**Trigger no banco notificando o worker** (ALT-03, Diego `[09:09]`). Descartada por limitação do MySQL: não há `NOTIFY`/`LISTEN`. Os workarounds cogitados — escrever em arquivo ou bater num endpoint — foram rejeitados na própria fala que os levantou: "fica esquisito."

**Fila com push (Redis Streams e equivalentes).** Descartada em [ADR-001](ADR-001-outbox-transacional-no-mysql.md) por RES-01/ALT-02; sem ela, polling é o único mecanismo de descoberta disponível.

## Consequências

### Positivas

- **Isolamento de falha nos dois sentidos.** Deploy ou reinício da API não interrompe a entrega de eventos; um worker travado numa chamada HTTP lenta não consome recurso de request da API.
- **Nenhuma dependência nova.** O worker usa Prisma e MySQL, exatamente a stack que o time já opera (RES-02).
- **Escala de leitura previsível.** Batch pequeno e ordenação por `created_at` mantêm a query de polling barata e indexada (TEC-01).

### Negativas

- **O orçamento de latência não fecha no pior caso.** Somando os valores decididos: até 2s de espera no polling (DEC-02) mais até 10s de timeout na chamada ao cliente (RNF-06, decidido em [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)) **excede o alvo de 10 segundos end-to-end** de RNF-01. O alvo só é atendido no caminho feliz, com cliente respondendo rápido. Esse conflito entre RNF-01 e a soma DEC-02 + RNF-06 não foi levantado na reunião.
- **Polling paga custo mesmo ocioso.** Uma query a cada 2 segundos, 24h por dia, independente de haver evento pendente — cerca de 43 mil queries por dia por worker sem nenhum evento no sistema.
- **Trade-off explícito: simplicidade operacional contra eficiência e latência.** Push notificaria em milissegundos e não consumiria banco à toa; custaria infraestrutura nova, que RES-01 proíbe. A reunião trocou latência e eficiência por não operar mais um serviço.
- **Duas instâncias de `PrismaClient` no mesmo banco.** Cada processo abre seu próprio pool de conexões (COD-12). O dimensionamento total de conexões do MySQL passa a depender de quantos processos rodam, e não foi revisto na reunião.
- **O worker é ponto único de falha silencioso.** Se ele cai, nada quebra visivelmente: a API continua aceitando pedidos, a outbox continua acumulando, e os clientes simplesmente param de receber eventos. **Supervisão, restart e alarme do processo não foram tratados na transcrição.**

## Pontos em aberto

- **Supervisão do processo do worker.** Nenhum item do índice trata de como o worker é mantido vivo, reiniciado ou monitorado. `docker-compose.yml` provisiona apenas o MySQL, sem serviço de aplicação — logo não há hoje onde declarar o worker.
- **Conflito RNF-01 × (DEC-02 + RNF-06).** Precisa de decisão explícita: relaxar o alvo de 10s, reduzir o intervalo de polling, ou reduzir o timeout de entrega.

## Fontes

**Índice da transcrição:** DEC-02, DEC-03, DEC-14, DEC-17 · RNF-01, RNF-06 · RES-02, RES-05 · ALT-02, ALT-03 · ABE-02 · COD-10, COD-12, COD-16 · TEC-01, TEC-02, TEC-03, TEC-20, TEC-25.

**Arquivos reais:** `src/server.ts`, `src/app.ts`, `package.json`, `docker-compose.yml`, `.env.example` (`DATABASE_URL`).

**Relacionados:** [ADR-001](ADR-001-outbox-transacional-no-mysql.md) (o que o worker consome), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) (política de falha), [ADR-007](ADR-007-ordering-por-order-id-sob-single-worker.md) (por que só um worker).
