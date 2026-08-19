# ADR-005 — Entrega at-least-once com event id

## Status

**Accepted** — decisão fechada por Larissa (DEC-10, `[09:26]`).

> **Nota de contexto — revisão pendente.** A sessão de revisão do documento de design (`[09:50]`) ainda não ocorreu.
>
> **Nota de contexto — concordância parcial.** A nota de leitura (c)-4 do índice registra que Sofia levantou preocupação com esta decisão ("Isso joga responsabilidade pro cliente") e **não expressou concordância explícita** com o fechamento. Diego respondeu com argumento de mercado e Larissa fechou como decisão.

## Contexto

A política de retry de [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) reenvia eventos cuja entrega falhou ou estourou o timeout de 10 segundos. Um cliente que processou o evento mas cuja resposta se perdeu receberá o mesmo evento de novo. A pergunta é o que a plataforma promete sobre isso.

Diego posicionou o custo de evitar duplicata: "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" (ALT-07, `[09:25]`), citando Stripe e GitHub como precedente de mercado.

## Decisão

**Garantia at-least-once. A deduplicação é responsabilidade do cliente, e a plataforma fornece o identificador que a torna possível** (DEC-10 / RNF-03, Larissa `[09:26]`: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.").

1. **`event_id` UUID gerado no momento da inserção do evento na outbox** (TEC-04, Diego `[09:25]`): "É único por evento." O identificador nasce junto com a linha na `webhook_outbox` ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)) e **não muda entre retentativas** — é o que permite ao cliente reconhecer o reenvio.

2. **Transmitido no header `X-Event-Id`** (TEC-05) e também presente no corpo do payload como `event_id` (TEC-17).

3. **Contrato de headers do request de webhook:**

   | Header | Conteúdo | Fonte |
   |---|---|---|
   | `X-Event-Id` | UUID do evento, estável entre retentativas | TEC-04, TEC-05 |
   | `X-Signature` | HMAC-SHA256 do corpo — ver [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) | TEC-06 |
   | `X-Timestamp` | Timestamp do envio, para o cliente detectar replay se quiser | TEC-07 |
   | `X-Webhook-Id` | Id do endpoint cadastrado, para clientes com vários cadastros | TEC-08 |
   | `Content-Type` | `application/json` | TEC-09 |

   `X-Webhook-Id` foi pedido por Sofia (`[09:44]`): "pra cliente que tem vários conseguir saber qual cadastro caiu naquele envio."

4. **Documentação da responsabilidade do cliente.** Marcos assumiu documentar a garantia no portal (`[09:26]`).

## Alternativas Consideradas

**Garantia exactly-once** (ALT-07, Diego `[09:25]`). Descartada por custo de coordenação bilateral: exigiria protocolo de confirmação entre plataforma e cliente, e o benefício sobre at-least-once com `event_id` foi avaliado como marginal ("resolve 99% dos casos").

**Deduplicação do lado da plataforma.** Não foi proposta por ninguém. A alternativa implícita — a plataforma rastrear o que o cliente já confirmou — recai no mesmo problema de coordenação de ALT-07.

## Consequências

### Positivas

- **A plataforma não precisa de estado de confirmação por cliente.** O worker marca como entregue quando recebe resposta de sucesso, e nada mais (TEC-02).
- **Alinhamento com o padrão de mercado.** Clientes que já integram Stripe ou GitHub reconhecem o modelo e frequentemente já têm código de deduplicação (ALT-07).
- **`X-Event-Id` estável torna a dedup trivial do lado do cliente.** Uma chave única sobre o id recebido basta.
- **`X-Webhook-Id` desambigua múltiplos cadastros** sem o cliente precisar inferir pela URL de destino (TEC-08).

### Negativas

- **Duplicata não é caso raro: é consequência garantida do timeout.** Combinando RNF-06 com esta decisão — um cliente que processe o evento em mais de 10 segundos recebe resposta cortada pelo worker, que marca falha e retenta. O evento é processado duas vezes do lado do cliente, sem nenhuma falha real. Quanto mais lento o cliente, mais duplicatas.
- **Trade-off explícito: complexidade nossa transferida para o cliente.** Foi a objeção de Sofia, e ela não a retirou. O custo de integração sobe para cada cliente, e a qualidade da entrega passa a depender de código que não controlamos — um cliente que ignore `X-Event-Id` processará pedidos em duplicidade sem que saibamos.
- **Não temos como verificar se o cliente deduplica.** A garantia é contratual e documental (`[09:26]`), não técnica.
- **A promessa depende de documentação que ainda não existe.** Marcos assumiu documentar no portal; o índice não registra que tenha sido feito.
- **`X-Timestamp` não é proteção efetiva contra replay.** É oferecido "pra cliente conseguir detectar replay attack se quiser" (TEC-07), mas não está coberto pela assinatura, que cobre apenas o corpo (DEC-08) — detalhado nas consequências de [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md).

## Pontos em aberto

- **Concordância de Sofia com a decisão final.** Registrado no índice como item que soa decisão sem concordância explícita (nota de leitura (c)-4). Vale confirmar com ela antes da revisão de segurança de RNF-10.
- **Documentação no portal.** Prometida por Marcos (`[09:26]`), sem prazo nem confirmação de execução.

## Fontes

**Índice da transcrição** — enumeração literal dos IDs citados no corpo deste ADR: DEC-08, DEC-10 · RNF-03, RNF-06, RNF-10 · ALT-07 · TEC-02, TEC-04, TEC-05, TEC-06, TEC-07, TEC-08, TEC-09, TEC-17 · notas de leitura (c)-4.

**Arquivos reais:** `src/middlewares/request-logger.middleware.ts` (precedente de header de correlação `X-Request-Id`), `prisma/schema.prisma` (convenção `uuid` `@db.Char(36)`).

**Relacionados:** [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md) (o que produz o reenvio), [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) (`X-Signature`), [ADR-001](ADR-001-outbox-transacional-no-mysql.md) (onde o `event_id` nasce).
