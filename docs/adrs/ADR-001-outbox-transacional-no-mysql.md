# ADR-001 — Outbox transacional no MySQL para eventos de webhook

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00–09:53
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)
- **Consultados:** Marcos (PM), Sofia (Eng. Segurança)

## Contexto

Os clientes B2B precisam ser notificados quando o status de um pedido muda. A primeira pergunta levantada na reunião foi se o disparo seria síncrono dentro do service de pedidos ou se passaria por alguma forma de fila/outbox ([09:03] Larissa).

O disparo síncrono foi rejeitado por dois motivos concretos levantados na reunião:

1. A transação de mudança de status já é pesada: atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` dos produtos. Acrescentar uma chamada HTTP no meio faria um cliente lento travar a mudança de status de outros pedidos ([09:04] Bruno). Isso é verificável em `src/modules/orders/order.service.ts`, no método `changeStatus`, que executa tudo dentro de um único `prisma.$transaction`.
2. Se o cliente estivesse fora do ar, não haveria resposta razoável: dar rollback na mudança de status por causa de uma notificação externa não é aceitável ([09:04] Bruno).

Diego formalizou a alternativa: o que o time quer não é exatamente uma "fila", e sim o padrão **outbox** ([09:06]).

## Decisão

Adotamos o **padrão Outbox persistido no MySQL já existente**.

Quando o status de um pedido muda, **dentro da mesma transação SQL** que atualiza `orders` e `order_status_history`, uma linha de evento é inserida na tabela `webhook_outbox` ([09:06] Diego). Um worker separado lê essa tabela e executa as chamadas HTTP.

Propriedades garantidas:

- Se a transação principal commitou, o evento está registrado.
- Se a transação principal deu rollback, o evento desaparece junto.
- Não existe estado intermediário em que o status mudou e o evento não foi registrado ([09:40] Bruno, [09:41] Diego).

Detalhes de modelagem decididos na reunião:

- Chave primária em **UUID**, seguindo o padrão do restante do projeto ([09:51] Larissa/Diego) — confirmado em `prisma/schema.prisma`, onde todos os modelos usam `@default(uuid()) @db.Char(36)`.
- Índices em **status do evento** (pendente, processando, falhou, entregue) e em **`created_at`**, para o worker ler apenas os pendentes mais antigos em batch pequeno ([09:08] Diego).
- O evento guarda o **payload já renderizado (snapshot)** no momento da inserção — ver [ADR-007](./ADR-007-payload-snapshot-na-outbox.md).
- O filtro de assinatura é aplicado **na inserção**: se nenhum webhook ativo do customer escuta aquele status, a linha nem é inserida ([09:34] Bruno/Diego).

## Alternativas consideradas

### A1 — Disparo HTTP síncrono dentro de `changeStatus`

- **Trade-off:** implementação trivial e latência mínima, mas acopla a disponibilidade do cliente externo à disponibilidade da operação de mudança de status, e não tem resposta para falha do cliente sem corromper a semântica da transação.
- **Motivo do descarte:** "Síncrono está fora de questão" ([09:06] Diego); risco de travar mudanças de status ([09:04] Bruno).

### A2 — Redis Streams (ou broker equivalente) como fila de eventos

- **Trade-off:** desacoplamento e throughput melhores, ao custo de subir e operar infraestrutura nova, além de reintroduzir o problema de dual-write (commit no MySQL + publish no Redis não são atômicos).
- **Motivo do descarte:** "a gente acabaria precisando subir mais infra" ([09:07] Larissa) e "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" ([09:07] Diego).

## Consequências

### Positivas

- Atomicidade real entre mudança de status e registro do evento, sem dual-write.
- Zero infraestrutura nova: reaproveita o MySQL e o Prisma já em produção (`src/config/database.ts`, `docker-compose.yml`).
- A outbox funciona como trilha de auditoria natural do que foi (ou não) notificado.

### Negativas

- A tabela de outbox cresce com o volume de mudanças de status e vira responsabilidade operacional. O arquivamento de linhas entregues (na ordem de 30 dias) foi mencionado, mas **explicitamente colocado fora do escopo desta feature** ([09:08] Diego).
- A transação de `changeStatus` ganha mais uma escrita, aumentando levemente o tempo de lock.
- Se a inserção na outbox falhar, a mudança de status inteira sofre rollback — é o trade-off aceito conscientemente: "Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).
- O MySQL passa a ser ponto único de falha também para a entrega de notificações.

## Decisões relacionadas

- [ADR-002 — Worker em processo separado com polling de 2 segundos](./ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ em tabela dedicada](./ADR-003-retry-backoff-e-dlq.md)
- [ADR-007 — Snapshot do payload na inserção da outbox](./ADR-007-payload-snapshot-na-outbox.md)
