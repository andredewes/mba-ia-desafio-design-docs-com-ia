# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00–09:53
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)
- **Consultados:** Marcos (PM), Sofia (Eng. Segurança)

## Contexto

Como a entrega é uma chamada HTTP para infraestrutura de terceiros, falha é o caso normal, não a exceção. A pergunta colocada foi direta: "Se o cliente tá offline, o que a gente faz?" ([09:14] Larissa).

Duas subquestões precisavam de resposta: quantas vezes retentar e com qual espaçamento; e o que fazer com o evento quando as tentativas se esgotam ([09:17] Larissa).

## Decisão

1. **Backoff com os cinco intervalos decididos.** A progressão é **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**, totalizando aproximadamente 15 horas entre a primeira falha e a última tentativa ([09:17] Diego). Há uma ambiguidade a resolver: essa sequência implica cinco retentativas após o envio inicial, enquanto o resumo de [09:48] diz "total 5 tentativas". A contagem final de envios deve ser confirmada pelos decisores antes da implementação.
2. **Falha permanente vai para DLQ em tabela separada.** Esgotadas as tentativas, o evento é movido para a tabela `webhook_dead_letter`, que armazena **payload, motivo da falha e timestamp** ([09:18] Diego). A tabela separada mantém a leitura da outbox principal limpa e serve de evidência para debug e reprocessamento.
3. **Reprocessamento manual via endpoint administrativo.** `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Larissa/Diego). O controle de acesso desse endpoint está em [ADR-006](./ADR-006-reuso-dos-padroes-existentes.md) (role `ADMIN` via `requireRole`, com log de auditoria de quem executou).
4. **Timeout de 10 segundos por tentativa.** Cliente que não responde em 10 segundos é tratado como falha e entra no ciclo de retry ([09:42] Sofia/Diego).

## Alternativas consideradas

### A1 — 3 tentativas (política mais agressiva)

- **Proposta:** Bruno sugeriu 3 tentativas por ser mais agressivo ([09:16]).
- **Trade-off:** libera a fila mais rápido e reduz o custo de retentativas, mas mata eventos cedo demais. Com 3 tentativas a janela total cairia para cerca de 30 minutos, e o time já teve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).
- **Motivo do descarte:** cobertura insuficiente para janelas reais de indisponibilidade dos clientes.

### A2 — Retry indefinido com backoff

- **Trade-off:** nenhuma perda de evento, mas eventos ficam pendurados para sempre quando o cliente simplesmente sumiu, e a outbox nunca drena ([09:15] Diego).
- **Motivo do descarte:** "traz o problema de evento ficar pendurado pra sempre se o cliente sumiu" ([09:15] Diego).

### A3 — Marcar o evento como `failed` na própria outbox, sem tabela de DLQ

- **Proposta:** levantada por Larissa em [09:17].
- **Trade-off:** menos uma tabela para modelar, mas mistura eventos vivos com eventos mortos na mesma varredura do worker e polui a leitura da outbox.
- **Motivo do descarte:** "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento" ([09:18] Diego).

## Consequências

### Positivas

- Tolerância a indisponibilidades longas do cliente (até ~15 horas) sem intervenção humana, desde que os cinco intervalos sejam preservados.
- A outbox drena naturalmente: eventos irrecuperáveis saem da tabela quente.
- A DLQ preserva payload e motivo, o que torna o replay manual viável e auditável.

### Negativas

- Um evento pode ser entregue até 15 horas depois do fato, muito acima do SLA de 10 segundos do caminho feliz — aceito explicitamente: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).
- Retentativas longas aumentam a chance de o cliente receber eventos fora de ordem em relação a eventos novos do mesmo pedido.
- A DLQ exige acompanhamento operacional: sem alguém olhando, eventos mortos ficam parados. Notificação proativa do cliente por e-mail foi **descartada desta fase** ([09:37] Larissa).
- Mais uma tabela para modelar, migrar e manter.

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](./ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2 segundos](./ADR-002-worker-separado-com-polling.md)
- [ADR-005 — Entrega at-least-once com `X-Event-Id`](./ADR-005-at-least-once-com-x-event-id.md)
