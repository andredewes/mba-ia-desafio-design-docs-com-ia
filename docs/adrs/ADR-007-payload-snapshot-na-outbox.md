# ADR-007 — Snapshot do payload renderizado na inserção da outbox

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00–09:53 (discussão final, [09:51]–[09:52])
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)

## Contexto

No fechamento da reunião, Bruno levantou a última dúvida de modelagem: "o evento da outbox guarda o payload renderizado já, ou guarda só `order_id` e renderiza na hora do envio?" ([09:51]).

A pergunta é relevante porque existe uma janela entre a inserção na outbox e a entrega efetiva, que pode chegar a ~15 horas em cenários de retry ([ADR-003](./ADR-003-retry-backoff-e-dlq.md)). Nessa janela, o pedido pode ter mudado de status novamente, ter valores alterados ou ter sido cancelado.

## Decisão

O evento guarda o **payload já renderizado no momento da inserção**, como um snapshot imutável do estado do pedido quando a mudança de status ocorreu ([09:52] Larissa; "Concordo, snapshot na inserção", Diego; "Beleza, snapshot. Decidido", Bruno).

O conteúdo do payload foi definido em [09:43] por Diego: `event_id`, `event_type` (`order.status_changed`), `timestamp` em ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`. **Os itens do pedido não são enviados**, para não inflar o payload; o cliente que precisar de detalhe consulta `GET /orders/:id` depois ([09:43] Diego, [09:44] Bruno).

Como consequência direta, o payload persistido é o mesmo em todas as tentativas de entrega e no replay de DLQ: o conteúdo assinado por HMAC ([ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md)) nunca muda entre retentativas.

## Alternativas consideradas

### A1 — Guardar apenas `order_id` e renderizar o payload na hora do envio

- **Trade-off:** economiza espaço na tabela e mantém o payload sempre "atual", mas produz o caso incoerente de um evento que anuncia a transição `PAID → PROCESSING` carregando um pedido já `SHIPPED` ou `CANCELLED`, além de exigir uma consulta extra ao banco por tentativa de entrega.
- **Motivo do descarte:** "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" ([09:52] Larissa).

## Consequências

### Positivas

- O evento entregue é sempre coerente com o fato que o originou, independentemente de quanto tempo depois a entrega aconteça.
- Entrega e retentativa não dependem de nova leitura do pedido: menos carga no banco no caminho do worker.
- Assinatura HMAC estável entre tentativas, o que simplifica debug de problemas de validação de assinatura no cliente.
- A linha da outbox (e, se falhar, a da DLQ) contém tudo o que é necessário para reproduzir a entrega, o que sustenta o replay manual descrito em [ADR-003](./ADR-003-retry-backoff-e-dlq.md).

### Negativas

- Duplicação de dados: o payload replica informações que já existem nas tabelas `orders` e `order_status_history` (`prisma/schema.prisma`), aumentando o tamanho da outbox.
- Mudança no formato do payload não se aplica retroativamente a eventos já enfileirados; eventos antigos serão entregues no formato antigo.
- O payload persistido carrega dados de pedido do cliente, o que amplia a superfície de dados sensíveis em repouso e reforça a necessidade do limite de 64KB por evento definido em [09:24].

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](./ADR-001-outbox-transacional-no-mysql.md)
- [ADR-003 — Retry com backoff exponencial e DLQ em tabela dedicada](./ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint](./ADR-004-hmac-sha256-secret-por-endpoint.md)
