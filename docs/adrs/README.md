# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada em um arquivo individual, nomeado no
formato `ADR-NNN-titulo-em-kebab-case.md`, seguindo o formato MADR (Status, Contexto,
Decisão, Alternativas Consideradas, Consequências).

## Índice — Sistema de Webhooks de Notificação de Pedidos

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](./ADR-001-outbox-transacional-no-mysql.md) | Outbox transacional no MySQL para eventos de webhook | Aceito |
| [ADR-002](./ADR-002-worker-separado-com-polling.md) | Worker em processo separado consumindo a outbox por polling de 2 segundos | Aceito |
| [ADR-003](./ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada | Aceito |
| [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) | Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | Aceito |
| [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) | Garantia de entrega at-least-once com deduplicação por `X-Event-Id` | Aceito |
| [ADR-006](./ADR-006-reuso-dos-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto no módulo de webhooks | Aceito |
| [ADR-007](./ADR-007-payload-snapshot-na-outbox.md) | Snapshot do payload renderizado na inserção da outbox | Aceito |

Todas as decisões acima têm origem rastreável na reunião registrada em
[`TRANSCRICAO.md`](../../TRANSCRICAO.md) ou no código-fonte da aplicação. O mapeamento
item a item está em [`docs/TRACKER.md`](../TRACKER.md).
