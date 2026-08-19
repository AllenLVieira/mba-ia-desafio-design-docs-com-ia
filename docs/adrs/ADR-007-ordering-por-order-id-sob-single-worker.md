# ADR-007 — Ordering por order id sob single worker

## Status

**Accepted** — registrado por Larissa como limitação conhecida, não como garantia (DEC-04, `[09:13]`): "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker."

> **Nota de contexto — revisão pendente.** A sessão de revisão do documento de design (`[09:50]`) ainda não ocorreu.

## Contexto

O ciclo de vida do pedido é uma máquina de estados linear com dois estados terminais. Conforme `src/modules/orders/order.status.ts`:

```
PENDING ──► PAID ──► PROCESSING ──► SHIPPED ──► DELIVERED (terminal)
   │           │           │
   └───────────┴───────────┴──────────────────► CANCELLED (terminal)
```

Se os eventos chegarem fora de ordem, o cliente pode registrar um pedido como `PAID` depois de já tê-lo visto `SHIPPED` — regredindo o estado no sistema dele, e produzindo um estado que a própria máquina de estados proíbe.

A propriedade emerge do desenho de [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), não de um mecanismo dedicado. Diego (TEC-27, `[09:12]`): "Se a gente tem um único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em ordem."

## Decisão

**Declarar ordering como garantia limitada e condicional, documentada, em vez de construir mecanismo para sustentá-la** (DEC-04 / RNF-08).

1. **Escopo da garantia**: por `order_id`, não global. Eventos de pedidos diferentes podem chegar ao cliente em qualquer ordem relativa.

2. **Condição da garantia**: apenas enquanto houver **um único worker**. A ordem deriva do processamento sequencial por `created_at` da outbox (TEC-27 + TEC-02), não de particionamento, lock ou número de sequência.

3. **Natureza do registro**: limitação conhecida e documentada. Nenhum mecanismo é construído para impor a condição — não há nada no código que impeça subir um segundo worker.

## Alternativas Consideradas

**Garantia de ordering global.** Nunca foi perseguida; a discussão partiu direto do que o desenho single-worker entrega naturalmente (TEC-27).

**Particionamento por `order_id` ou lock pessimista** (ADI-05, Diego `[09:13]`). Reconhecidos como os caminhos para preservar a ordem com múltiplos workers, e adiados: "Mas isso é problema do futuro, não agora."

**Número de sequência no payload.** Não foi proposto por ninguém. Os campos definidos em TEC-17 não incluem contador nem versão — o cliente só recebe `from_status` e `to_status`.

## Consequências

### Positivas

- **Custo zero de implementação.** A ordem vem de graça do desenho já decidido em [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md); nenhuma coordenação, lock ou partição a construir.
- **Expectativa honesta.** A garantia é declarada pelo que é, com escopo e condição explícitos, em vez de prometida de forma vaga (DEC-04).
- **`from_status` e `to_status` no payload dão ao cliente como se defender.** TEC-17 inclui os dois, então um cliente que receba um evento inconsistente com seu estado local consegue detectá-lo — ainda que sem saber a ordem correta.

### Negativas

- **O retry quebra a ordem mesmo com um único worker.** Esta é a consequência mais séria, e ela decorre de combinar DEC-04 com DEC-05 ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)): se o evento `PAID` de um pedido falha e é reagendado para daqui a 1 minuto, e o evento `SHIPPED` do **mesmo** pedido é criado e entregue nesse intervalo, o cliente recebe `SHIPPED` antes de `PAID`. A garantia "por order_id" vale para o caminho feliz. **A interação entre retry e ordering não foi levantada na reunião.**
- **Escala horizontal do worker fica bloqueada por uma decisão implícita.** Um segundo worker é a resposta natural para vazão insuficiente, e adotá-lo quebra a garantia — silenciosamente, sem erro, sem alarme, sem nada no código que sinalize (ADI-05).
- **Trade-off explícito: simplicidade agora contra teto de vazão depois.** O sistema fica limitado ao throughput de um processo em polling de 2 segundos. Quando esse teto for atingido, a saída exige a decisão adiada em ADI-05, e a migração acontecerá sob pressão de capacidade.
- **O cliente não tem como reordenar de forma confiável.** Sem número de sequência, ele depende do `timestamp` ISO 8601 do payload (TEC-19) e da transição `from_status`/`to_status` (TEC-17) para inferir a ordem — e nada disso é uma ordem total garantida.
- **A condição não é verificável em runtime.** Nada no sistema sabe quantos workers estão rodando.

## Pontos em aberto

- **ADI-05 — múltiplos workers.** Particionamento por `order_id` ou lock pessimista, adiado como "problema do futuro" (`[09:13]`). Sem gatilho definido: nenhum limiar de vazão foi estabelecido para disparar a decisão.
- **Ordem sob retry.** Não tratada na transcrição. Precisa de decisão explícita: aceitar como limitação adicional (e documentá-la ao cliente junto com DEC-04), ou bloquear a entrega de eventos subsequentes do mesmo `order_id` enquanto houver um evento anterior em retry.
- **Nada impede um segundo worker.** Não foi decidido se a condição de single-worker deve ser imposta (lock de instância) ou apenas documentada.

## Fontes

**Índice da transcrição** — enumeração literal dos IDs citados no corpo deste ADR: DEC-04, DEC-05 · RNF-08 · ADI-05 · TEC-02, TEC-17, TEC-19, TEC-27.

**Arquivos reais:** `src/modules/orders/order.status.ts` (`canTransition`, `allowedTransitions`, `isTerminal`, tabela `transitions`), `prisma/schema.prisma` (enum `OrderStatus`).

**Relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) (de onde a ordem emerge), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) (o que a quebra), [ADR-005](ADR-005-entrega-at-least-once-com-event-id.md) (a outra garantia limitada oferecida ao cliente).
