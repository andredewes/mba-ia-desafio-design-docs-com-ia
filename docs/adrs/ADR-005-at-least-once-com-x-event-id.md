# ADR-005 — Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00–09:53
- **Decisores:** Diego (Eng. Sênior — Plataforma), Larissa (Tech Lead)
- **Consultados:** Bruno (Eng. Pleno — Pedidos), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

A combinação de outbox ([ADR-001](./ADR-001-outbox-transacional-no-mysql.md)) com retry automático ([ADR-003](./ADR-003-retry-backoff-e-dlq.md)) torna a entrega duplicada inevitável: uma resposta HTTP perdida, um timeout de rede ou um crash do worker após a entrega e antes da marcação levam à retentativa de um evento que o cliente já processou.

Diego levantou o ponto de forma direta: "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado" ([09:24]).

A pergunta seguinte foi como o cliente distingue uma entrega nova de uma repetida ([09:25] Bruno).

## Decisão

A garantia de entrega é **at-least-once**, com deduplicação delegada ao cliente por meio de um identificador único de evento.

- Cada evento recebe um **UUID gerado no momento em que entra na outbox**, único por evento ([09:25] Diego).
- Esse identificador viaja no header **`X-Event-Id`** em todas as tentativas de entrega, inclusive nas retentativas e no replay de DLQ.
- O cliente deduplica pelo `X-Event-Id` do lado dele ([09:25] Diego).
- A semântica at-least-once e a necessidade de deduplicação serão **documentadas com destaque no portal do desenvolvedor** ([09:26] Marcos).

## Alternativas consideradas

### A1 — Garantia exactly-once

- **Trade-off:** eliminaria a necessidade de deduplicação no cliente, mas exigiria coordenação entre os dois lados (protocolo de confirmação, controle de estado por evento em ambas as pontas), aumentando muito a complexidade da feature e do contrato de integração.
- **Motivo do descarte:** "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" ([09:25] Diego). O precedente de mercado foi citado: Stripe e GitHub adotam o mesmo modelo ([09:25] Diego).

### A2 — At-most-once (entregar uma vez e desistir na falha)

- **Trade-off:** o cliente nunca receberia duplicata, mas eventos seriam silenciosamente perdidos em qualquer falha transitória — exatamente o problema que motivou o retry.
- **Motivo do descarte:** incompatível com a política de retry decidida em [ADR-003](./ADR-003-retry-backoff-e-dlq.md) e com a expectativa do cliente de não perder mudança de status.

## Consequências

### Positivas

- Nenhum evento é perdido por falha transitória de rede ou do cliente.
- Contrato simples, alinhado ao que os clientes B2B já encontram em Stripe e GitHub.
- O `X-Event-Id` também serve de correlação para suporte e debug: um id único liga a linha da outbox, os registros de tentativa e a requisição recebida pelo cliente.

### Negativas

- **Transfere responsabilidade para o cliente**, ponto levantado por Sofia ([09:25]): quem não implementar deduplicação pode processar o mesmo evento mais de uma vez.
- Exige documentação explícita e bem visível no portal do desenvolvedor, senão vira ticket de suporte recorrente.
- Combinada com a limitação de ordenação descrita em [ADR-002](./ADR-002-worker-separado-com-polling.md), a integração do cliente precisa ser tolerante tanto a duplicatas quanto a eventos fora de ordem em cenários de retry.

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](./ADR-001-outbox-transacional-no-mysql.md)
- [ADR-003 — Retry com backoff exponencial e DLQ em tabela dedicada](./ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint](./ADR-004-hmac-sha256-secret-por-endpoint.md)
