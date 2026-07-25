# ADR-002 — Worker em processo separado consumindo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00–09:53
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)
- **Consultados:** Marcos (PM)

## Contexto

Definido o outbox ([ADR-001](./ADR-001-outbox-transacional-no-mysql.md)), a pergunta seguinte foi como o worker lê a tabela ([09:08] Larissa).

A restrição de produto é clara: os clientes consideram "tempo real" qualquer coisa **abaixo de 10 segundos**; o inaceitável é o evento ficar pendurado ([09:02] Marcos).

Também foi levantado que o worker não pode viver dentro da instância da API: "se a API reinicia, perde o worker" ([09:11] Diego).

## Decisão

1. **Consumo por polling em loop, a cada 2 segundos.** A cada ciclo o worker busca os eventos pendentes mais antigos, processa e marca o resultado ([09:09] Diego). O tempo até o início do processamento é de até 2 segundos; a entrega completa continua sujeita ao objetivo de menos de 10 segundos ([09:02] Marcos).
2. **Processo separado da API.** Uma nova entrada de execução espelha o padrão existente em `src/server.ts`, com um script `npm run worker` ([09:11] Larissa). A lógica de processamento fica dentro do novo módulo de webhooks ([09:28] Bruno). O nome final dos arquivos será definido na implementação; este documento não os apresenta como caminhos já existentes.
3. **Mesmo banco, instância própria de Prisma.** O worker usa a mesma `DATABASE_URL`, mas cria seu próprio `PrismaClient`, porque o client é por processo ([09:29] Diego, [09:30] Bruno).
4. **Single-worker por enquanto.** Com um único worker, o processamento respeita a ordem de `created_at` da outbox e o cliente recebe os eventos de um pedido na ordem correta ([09:12] Diego).

## Alternativas consideradas

### A1 — Trigger de banco para notificar o worker de forma reativa

- **Proposta:** Bruno perguntou se dava para usar trigger do MySQL em vez de polling ([09:09]).
- **Trade-off:** seria mais reativo e reduziria a latência, mas o MySQL não tem listener nativo equivalente ao `NOTIFY`/`LISTEN` do Postgres; a trigger só executa SQL e não avisa processo externo. Notificar o worker exigiria gambiarras como escrever em arquivo ou chamar um endpoint ([09:09] Diego).
- **Motivo do descarte:** complexidade sem ganho relevante — "polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo" ([09:09] Diego).

### A2 — Worker rodando dentro do mesmo processo da API

- **Trade-off:** um único deploy e um único `PrismaClient`, mas o ciclo de vida do worker fica preso ao da API: restart, deploy ou crash da API derruba o processamento de eventos.
- **Motivo do descarte:** "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker" ([09:11] Diego).

### A3 — Múltiplos workers em paralelo desde o início

- **Trade-off:** mais throughput, mas perde a garantia de ordenação por pedido, exigindo particionamento por `order_id` ou lock pessimista ([09:12]–[09:13] Diego).
- **Motivo do descarte:** "isso é problema do futuro, não agora" ([09:13] Diego).

## Consequências

### Positivas

- Isolamento de falhas: a API não é afetada por lentidão ou travamento do processamento de webhooks, e vice-versa.
- Implementação simples, sem dependência de recursos exóticos do banco.
- Ordenação por pedido garantida de graça enquanto houver um único worker.

### Negativas

- Polling gera consultas constantes ao MySQL mesmo com a outbox vazia.
- Espera de até 2 segundos antes do início do processamento, por construção.
- **Limitação conhecida e documentada:** não há garantia de ordenação global; a garantia é por `order_id` e apenas enquanto o modelo for single-worker ([09:13] Larissa). Marcos confirmou que os clientes nunca pediram ordenação global ([09:14]).
- Escalar horizontalmente exige uma decisão futura de particionamento ou lock pessimista.
- Mais um processo para operar, monitorar e implantar.

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](./ADR-001-outbox-transacional-no-mysql.md)
- [ADR-003 — Retry com backoff exponencial e DLQ em tabela dedicada](./ADR-003-retry-backoff-e-dlq.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./ADR-006-reuso-dos-padroes-existentes.md)
