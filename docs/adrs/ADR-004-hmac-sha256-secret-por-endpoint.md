# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00–09:53
- **Decisores:** Sofia (Eng. Segurança), Larissa (Tech Lead)
- **Consultados:** Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)

## Contexto

A feature expõe dados de pedidos para endpoints fora da nossa infraestrutura. Sofia colocou o requisito de segurança de forma explícita: o cliente precisa conseguir validar que a requisição veio realmente de nós e que ninguém adulterou o payload no caminho ([09:19]).

O escopo já havia sido fechado como **outbound apenas** — os clientes recebem, não enviam ([09:02] Marcos, [09:03] Sofia) —, o que elimina a necessidade de autenticação de entrada e reduz o problema a autenticar a origem da mensagem.

Um incidente real reforçou a preocupação com vazamento: o time já teve cliente que vazou secret em log de aplicação ([09:22] Diego).

## Decisão

1. **HMAC-SHA256 sobre o corpo da requisição.** A assinatura vai no header `X-Signature`; o cliente recalcula e compara do lado dele ([09:20] Sofia). O algoritmo é SHA-256 por ser o padrão de mercado, com biblioteca disponível em qualquer stack séria ([09:20] Sofia).
2. **Secret única por endpoint de webhook**, e não uma secret global da plataforma: "se vaza uma, vaza tudo" ([09:21] Sofia). A secret é **gerada pela plataforma** e devolvida ao cliente no momento da criação do webhook ([09:31] Marcos).
3. **Rotação de secret via API, com grace period de 24 horas.** Ao rotacionar, a secret antiga continua válida por 24 horas em paralelo com a nova, dando ao cliente tempo de migrar seus sistemas; depois disso a antiga morre ([09:21] Sofia, confirmado no resumo de [09:48] Larissa).
4. **TLS obrigatório.** A URL cadastrada precisa ser `https`; `http` é recusado com erro de validação. Isso é implementado como validação de schema Zod, não como componente arquitetural ([09:23] Sofia).
5. **Headers de segurança complementares:** `X-Timestamp` com o timestamp do envio, para o cliente detectar replay attack se quiser ([09:44] Diego), e `X-Webhook-Id` com o id do cadastro de webhook, para clientes com vários endpoints identificarem qual configuração originou o envio ([09:44] Sofia).
6. **Tabela de configuração** guarda `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno/Sofia).

## Alternativas consideradas

### A1 — Secret global da plataforma compartilhada por todos os clientes

- **Trade-off:** gestão trivial de uma única chave, mas o raio de explosão de um vazamento é total.
- **Motivo do descarte:** "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo" ([09:21] Sofia).

### A2 — Rotação de secret sem grace period (corte imediato)

- **Trade-off:** modelo mental mais simples e janela de exposição menor, mas quebra a integração do cliente no instante da rotação, já que ele precisa atualizar seus sistemas de forma atômica.
- **Motivo do descarte:** "Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele" ([09:21] Sofia).

## Consequências

### Positivas

- Autenticidade e integridade do payload verificáveis pelo cliente, com padrão amplamente conhecido no mercado.
- Vazamento de uma secret compromete apenas um endpoint de um cliente.
- Rotação sem downtime na integração do cliente.
- Confidencialidade em trânsito garantida por TLS obrigatório.

### Negativas

- Durante as 24 horas de grace period existem duas secrets válidas simultaneamente para o mesmo endpoint, ampliando temporariamente a superfície de ataque.
- A plataforma passa a armazenar material secreto por endpoint, com as obrigações de proteção que isso implica (o logger Pino já aplica redaction de campos sensíveis em `src/shared/logger/index.ts`, e a lista de campos redigidos precisa ser estendida para os campos de secret).
- A validação da assinatura é responsabilidade do cliente; um cliente que não valide não ganha nada com o mecanismo.
- Sofia reservou **dois dias úteis de revisão de segurança** sobre HMAC e geração de secret antes do deploy ([09:46]), o que precisa entrar no cronograma.

## Decisões relacionadas

- [ADR-005 — Entrega at-least-once com `X-Event-Id`](./ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./ADR-006-reuso-dos-padroes-existentes.md)
