# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Time envolvido** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Status** | Em revisão técnica |
| **Prazo alvo** | 3 sprints; compromisso comercial com a Atlas para fim de novembro ([09:45] Marcos, [09:46] Larissa) |
| **Documentos relacionados** | [RFC](./RFC.md), [FDD](./FDD.md), [ADRs](./adrs/), [Tracker](./TRACKER.md) |

---

## 1. Resumo e contexto

O OMS passa a notificar clientes B2B, de forma automática e em tempo quase real, sempre que o status
de um pedido muda. A notificação é enviada como uma requisição HTTP (webhook) para um endpoint
cadastrado pelo próprio cliente, assinada criptograficamente para que ele possa validar a origem e a
integridade da mensagem.

O contexto é uma demanda formal de três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova
Cargo** — recebida na semana anterior à reunião de definição técnica ([09:00] Marcos). A plataforma
hoje não possui nenhum mecanismo de notificação externa: a única forma de o cliente saber que algo
mudou é consultar a API de pedidos repetidamente.

A feature é **exclusivamente outbound**: a plataforma envia, o cliente recebe. Não há recebimento de
webhooks de clientes ([09:02] Marcos, [09:03] Sofia).

---

## 2. Problema e motivação

**Problema.** Os clientes B2B integrados ao OMS não têm como saber que o status de um pedido mudou
sem consultar a API. Hoje eles ficam "batendo no `GET /orders` de tempos em tempos pra ver se mudou
alguma coisa", o que torna a integração lenta e cara do lado deles ([09:00] Marcos).

**Motivações:**

| # | Motivação | Origem |
| --- | --- | --- |
| M-1 | Reduzir a latência com que o cliente descobre uma mudança de status; qualquer coisa abaixo de 10 segundos já é considerada "tempo real" por eles | [09:02] Marcos |
| M-2 | Eliminar o custo e a lentidão do polling na integração dos clientes | [09:00] Marcos |
| M-3 | **Risco comercial concreto:** a Atlas Comercial sinalizou que pode migrar para um concorrente se a entrega não acontecer até o fim do trimestre | [09:00] Marcos |
| M-4 | Reduzir a carga de leitura repetida sobre `GET /orders`, hoje o único caminho disponível para o cliente (`src/modules/orders/order.routes.ts`) | `src/modules/orders/order.routes.ts` |

---

## 3. Público-alvo e cenários de uso

### 3.1 Público-alvo

| Perfil | Descrição |
| --- | --- |
| **Clientes B2B integrados via API** | Atlas Comercial, MaxDistribuição e Nova Cargo; consomem a API do OMS a partir dos ERPs deles ([09:00] Marcos) |
| **Usuários da plataforma que representam o cliente** | Autenticam-se com JWT do nosso sistema e configuram os webhooks pela nossa API ([09:32] Marcos) |
| **Operação/Suporte com role `ADMIN`** | Reprocessam eventos que falharam definitivamente ([09:36] Sofia) |

### 3.2 Cenários de uso

**C-1 — Cliente acompanha o ciclo de vida do pedido.** O ERP da Atlas cadastra um endpoint
escutando `SHIPPED` e `DELIVERED`. Quando um pedido é despachado, o worker inicia a tentativa em até
2 segundos e o objetivo de produto é entregar em menos de 10 segundos, sem exigir nova consulta à
API ([09:33] Marcos, [09:02] Marcos, [09:09] Diego).

**C-2 — Cliente fica indisponível temporariamente.** A MaxDistribuição entra em manutenção
planejada de duas horas. As entregas falham, são retentadas com espaçamento crescente e chegam
quando o sistema volta, sem intervenção humana ([09:16] Diego).

**C-3 — Cliente descobre que perdeu eventos.** A Nova Cargo suspeita de falha na integração e
consulta o histórico de entregas do webhook, vendo sucesso/falha, payload, resposta e tempo de
resposta de cada tentativa ([09:34] Marcos).

**C-4 — Rotação de secret.** O cliente suspeita de vazamento da secret e solicita uma nova pela
API; a antiga continua válida por 24 horas enquanto ele migra os sistemas dele ([09:21] Sofia).

**C-5 — Reprocessamento após falha definitiva.** Um evento esgota a política de retry e vai para a
dead-letter queue. O suporte, com role `ADMIN`, dispara o replay manual e o evento volta para a fila
([09:18] Diego, [09:36] Sofia).

---

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| O-1 | Entregar a notificação em tempo percebido como real | Tempo entre o commit da mudança de status e a entrega bem-sucedida | **< 10 segundos** no caminho feliz, com latência de polling de **2 segundos** | [09:02] Marcos, [09:09] Diego |
| O-2 | Não perder eventos por indisponibilidade transitória do cliente | Cobertura da janela de retry | Aplicar os cinco intervalos decididos (**1m/5m/30m/2h/12h**), cobrindo **~15 horas**; a contagem exata de envios aguarda confirmação técnica | [09:17] Diego, [09:48] Larissa |
| O-3 | Eliminar a dependência de polling dos clientes solicitantes | Nº de clientes B2B que adotam webhook em substituição ao polling | Medir adoção entre Atlas, MaxDistribuição e Nova Cargo; a reunião não definiu meta quantitativa nem prazo de migração para os três | [09:00] Marcos |
| O-4 | Nenhuma mudança de status sem evento correspondente | Divergência entre transições em `order_status_history` e eventos publicados na outbox | **0 divergências**, garantido pela atomicidade transacional | [09:40] Bruno |
| O-5 | Nenhum evento entregue sem assinatura verificável | Proporção de entregas com `X-Signature` válido | **100%** | [09:20] Sofia |
| O-6 | Entregar dentro do compromisso comercial | Prazo de desenvolvimento | **3 sprints**, incluindo 2 dias úteis de revisão de segurança | [09:46] Larissa e Sofia |

---

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de endpoints de webhook por cliente ([09:31] Marcos,
  [09:33] Bruno).
- Filtro de eventos por endpoint: o cliente escolhe quais status quer receber ([09:33] Marcos).
- Geração de secret pela plataforma e rotação com grace period de 24 horas ([09:31] Marcos,
  [09:21] Sofia).
- Assinatura HMAC-SHA256 do payload ([09:20] Sofia).
- Publicação transacional do evento junto à mudança de status ([09:06] Diego).
- Worker de entrega com retry, backoff e dead-letter queue ([09:09] e [09:15] Diego).
- Histórico de entregas consultável por endpoint ([09:34] Marcos).
- Replay manual de eventos mortos, restrito a role `ADMIN` ([09:35] Diego, [09:36] Sofia).
- Evento único nesta fase: mudança de status de pedido (`order.status_changed`) ([09:43] Diego).

### 5.2 Fora de escopo

| # | Item | Situação | Origem |
| --- | --- | --- | --- |
| FE-1 | **Notificação por e-mail ao cliente quando o webhook dele falha** (ex.: após 3 falhas seguidas) | Descartado desta fase. "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" | [09:37] Marcos e Larissa |
| FE-2 | **Dashboard visual para o cliente acompanhar os webhooks** | Descartado. "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" | [09:39] Marcos, [09:40] Larissa |
| FE-3 | **Rate limiting de saída** (limitar rajadas de chamadas contra o endpoint do cliente) | Adiado. "A gente observa e implementa se virar problema" | [09:38] Diego, [09:39] Larissa |
| FE-4 | **Webhooks inbound** (cliente enviando eventos para a plataforma) | Fora de escopo desde a definição inicial: "Só saindo da gente pra eles" | [09:02] Marcos |
| FE-5 | **Arquivamento/expurgo das linhas entregues da outbox** (ordem de 30 dias) | Reconhecido como necessário, mas "fora do escopo dessa feature" | [09:08] Diego |
| FE-6 | **Escala horizontal do worker** com garantia de ordenação (particionamento por `order_id` ou lock pessimista) | Adiado. "Isso é problema do futuro, não agora" | [09:13] Diego |
| FE-7 | **Garantia de ordenação global de eventos** | Não será oferecida; a garantia é por pedido e apenas enquanto houver um único worker. Os clientes nunca pediram ordenação global | [09:13] Larissa, [09:14] Marcos |
| FE-8 | **Eventos de outros domínios** (clientes, produtos, usuários) | Nesta fase, apenas mudança de status de pedido | [09:43] Diego |

---

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | O cliente deve poder **cadastrar** um endpoint de webhook informando a URL e a lista de status que deseja receber. A secret é gerada pela plataforma e devolvida na criação | [09:31] Marcos |
| RF-02 | O cliente deve poder **listar** os webhooks cadastrados de um cliente | [09:33] Bruno |
| RF-03 | O cliente deve poder **editar** um webhook existente (URL, filtro de eventos, estado ativo) | [09:33] Bruno |
| RF-04 | O cliente deve poder **remover** um webhook | [09:33] Bruno |
| RF-05 | Cada endpoint deve permitir **filtrar quais status de pedido** deseja receber (ex.: apenas `SHIPPED` e `DELIVERED`); o filtro é aplicado no momento de registrar o evento | [09:33] Marcos, [09:34] Bruno |
| RF-06 | O cliente deve poder **rotacionar a secret** de um endpoint via API, com a secret anterior permanecendo válida por 24 horas | [09:21] Sofia |
| RF-07 | O cliente deve poder **consultar o histórico de entregas** de um webhook, com sucesso/falha, payload, resposta e tempo de resposta | [09:34] Marcos |
| RF-08 | O sistema deve **registrar o evento na mesma transação** da mudança de status; se o registro falhar, a mudança de status sofre rollback | [09:40] Bruno |
| RF-09 | O sistema deve **entregar o evento por HTTP POST** ao endpoint cadastrado, de forma assíncrona, por meio de um processo separado da API | [09:11] Diego |
| RF-10 | O sistema deve **retentar entregas com falha** nos intervalos de 1m, 5m, 30m, 2h e 12h. A contagem de cinco envios totais versus cinco retentativas deve ser confirmada antes da implementação | [09:17] Diego, [09:48] Larissa |
| RF-11 | O sistema deve **mover para uma dead-letter queue** os eventos que esgotarem as tentativas, preservando payload, motivo da falha e timestamp | [09:18] Diego |
| RF-12 | Um usuário com role `ADMIN` deve poder **reprocessar manualmente** um evento da dead-letter queue, recolocando-o na fila; a operação é registrada em log para auditoria | [09:18] e [09:35] Diego, [09:36] Sofia |
| RF-13 | Toda entrega deve ser **assinada com HMAC-SHA256** sobre o corpo da requisição, usando a secret específica daquele endpoint, e enviada no header `X-Signature` | [09:20] e [09:21] Sofia |
| RF-14 | Toda entrega deve incluir os headers **`X-Event-Id`, `X-Timestamp` e `X-Webhook-Id`**, além de `Content-Type: application/json` | [09:44] Diego e Sofia |
| RF-15 | O payload deve conter `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, **sem os itens do pedido** | [09:43] Diego |
| RF-16 | O payload deve ser um **snapshot do estado do pedido no momento da transição**, renderizado na inserção do evento | [09:52] Larissa |
| RF-17 | A URL cadastrada deve ser **obrigatoriamente `https`**; URLs `http` são recusadas com erro de validação | [09:23] Sofia |
| RF-18 | Os endpoints de configuração exigem autenticação; o `customer_id` é informado no corpo ou no caminho da requisição, e **não** extraído do JWT | [09:32] Bruno e Larissa |

---

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RNF-01 | **Latência:** entrega abaixo de 10 segundos no caminho feliz; polling a cada 2 segundos limita o tempo até o início do processamento, não o tempo total da chamada HTTP | [09:02] Marcos, [09:09] Diego |
| RNF-02 | **Timeout de entrega:** 10 segundos por tentativa; cliente que não responde nesse prazo é tratado como falha | [09:42] Diego |
| RNF-03 | **Limite de payload:** 64KB por evento; acima disso o evento erra em vez de ser truncado | [09:23] Sofia, [09:24] Diego e Larissa |
| RNF-04 | **Garantia de entrega at-least-once:** o cliente pode receber o mesmo evento mais de uma vez e deve deduplicar pelo `X-Event-Id` | [09:24] e [09:25] Diego |
| RNF-05 | **Ordenação:** garantida por pedido enquanto houver um único worker; não há garantia de ordenação global | [09:12] e [09:13] Diego, [09:13] Larissa |
| RNF-06 | **Isolamento de processo:** o worker roda fora da instância da API, para que restart da API não interrompa o processamento | [09:11] Diego |
| RNF-07 | **Sem nova infraestrutura:** a solução usa o MySQL já existente, sem broker ou serviço adicional | [09:07] Diego |
| RNF-08 | **Reuso dos padrões do projeto:** `AppError`, Pino, middleware de erro, estrutura de módulos, schemas Zod e o padrão de códigos de erro, com prefixo `WEBHOOK_` | [09:29] e [09:30] Larissa |
| RNF-09 | **Segurança da secret:** uma secret por endpoint, nunca uma secret global | [09:21] Sofia |
| RNF-10 | **Autorização:** o replay de dead-letter exige role `ADMIN`; o restante do CRUD aceita qualquer role autenticada nesta fase | [09:36] e [09:37] Sofia |
| RNF-11 | **Auditoria:** a operação de replay registra quem a executou | [09:36] Sofia |
| RNF-12 | **Revisão de segurança:** dois dias úteis reservados para revisão de HMAC e geração de secret antes do deploy | [09:46] Sofia |
| RNF-13 | **Persistência com UUID:** os identificadores das novas tabelas seguem o padrão UUID do restante do projeto | [09:51] Larissa |

---

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Outbox transacional no MySQL, em vez de disparo síncrono ou broker | Ganha atomicidade e zero infraestrutura nova; aceita crescimento de tabela e uma escrita a mais na transação mais crítica do OMS | [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Worker separado em polling de 2 segundos | Ganha isolamento e simplicidade; aceita latência mínima de 2 segundos e consultas periódicas ao banco | [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) |
| Backoff 1m/5m/30m/2h/12h e DLQ dedicada | Cobre indisponibilidades longas do cliente; aceita que um evento possa ser entregue até ~15 horas depois; a contagem exata de envios está pendente | [ADR-003](./adrs/ADR-003-retry-backoff-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint e rotação com grace de 24h | Limita o raio de um vazamento e evita quebrar a integração na rotação; aceita duas secrets válidas por 24 horas | [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| Entrega at-least-once com `X-Event-Id` | Evita a complexidade de exactly-once; transfere a deduplicação para o cliente, o que exige documentação clara no portal | [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso máximo dos padrões existentes | Velocidade e consistência; herda também as limitações atuais da codebase | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Payload como snapshot na inserção | Evento sempre coerente com o fato que o gerou; aceita duplicação de dados na outbox | [ADR-007](./adrs/ADR-007-payload-snapshot-na-outbox.md) |

---

## 9. Dependências

| Tipo | Dependência | Detalhe |
| --- | --- | --- |
| Técnica | MySQL existente | Novas tabelas e migration, sem alteração de tabelas atuais (`prisma/schema.prisma`) |
| Técnica | Transação de `changeStatus` | A publicação do evento entra no `$transaction` de `src/modules/orders/order.service.ts` ([09:40] Bruno) |
| Técnica | Máquina de estados de pedidos | Os status disponíveis para assinatura são os de `src/modules/orders/order.status.ts` |
| Técnica | Autenticação e autorização existentes | `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts` ([09:36] Larissa) |
| Operacional | Novo processo de worker em produção | `npm run worker`, deployado separadamente da API ([09:11] Larissa e Diego) |
| Pessoas | Revisão de segurança da Sofia | 2 dias úteis antes do deploy ([09:46] Sofia) |
| Pessoas | Revisão do design por Bruno e Diego | Sessão dedicada antes do início da implementação ([09:50] Larissa) |
| Externa | Endpoints HTTPS dos clientes | Os clientes precisam expor endpoints válidos e implementar a verificação de assinatura e a deduplicação |
| Documentação | Portal do desenvolvedor | Marcos documenta a integração e a semântica at-least-once para os clientes ([09:26] e [09:40] Marcos) |

---

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| R-1 | **Perda do cliente Atlas por atraso na entrega.** A Atlas sinalizou possível migração para o concorrente se a feature não sair até o fim do trimestre ([09:00] Marcos) | Média | Alto | Escopo enxuto de 3 sprints com corte explícito de e-mail, dashboard e rate limiting; PM atualiza os clientes sobre o andamento ([09:47] Marcos) |
| R-2 | **Degradação do fluxo de mudança de status** por causa da escrita adicional dentro da transação já pesada de `changeStatus` ([09:04] Bruno) | Média | Alto | Filtro de assinantes antes de inserir, evitando linhas desnecessárias ([09:34] Bruno); payload renderizado em memória; sem chamada HTTP dentro da transação ([09:06] Diego) |
| R-3 | **Cliente não implementa deduplicação** e processa o mesmo evento mais de uma vez, sob garantia at-least-once ([09:25] Sofia) | Alta | Médio | `X-Event-Id` em todas as entregas e documentação destacada no portal do desenvolvedor ([09:26] Marcos) |
| R-4 | **Vazamento de secret do lado do cliente** — já houve caso real de secret vazada em log de aplicação de cliente ([09:22] Diego) | Baixa | Alto | Secret única por endpoint ([09:21] Sofia); rotação com grace period de 24h; revisão de segurança dedicada antes do deploy ([09:46] Sofia) |
| R-5 | **Cliente indisponível por período longo**, gerando acúmulo de eventos e entregas muito atrasadas | Média | Médio | Backoff com cinco intervalos cobrindo ~15 horas ([09:17] Diego); DLQ com replay manual ([09:18] Diego); aceito pelo PM: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos) |
| R-6 | **Rajada de mudanças de status** bombardeia o endpoint do cliente com muitas chamadas em sequência ([09:38] Diego) | Média | Médio | Sem mitigação nesta fase, por decisão explícita: observar em produção e decidir depois ([09:39] Larissa) |
| R-7 | **Crescimento indefinido da outbox** degrada o desempenho do worker | Média | Médio | Índices em status e data de criação, leitura em batch pequeno ([09:08] Diego); arquivamento planejado para depois ([09:08] Diego) |

---

## 11. Critérios de aceitação

| # | Critério |
| --- | --- |
| CA-01 | Um cliente consegue cadastrar um webhook informando URL `https` e a lista de status desejados, e recebe a secret na resposta da criação |
| CA-02 | O cadastro de uma URL `http` é recusado com erro de validação |
| CA-03 | Alterações de status que não constam no filtro do endpoint não geram entrega |
| CA-04 | Uma mudança de status de pedido para um cliente com webhook ativo resulta em entrega ao endpoint em menos de 10 segundos |
| CA-05 | A requisição entregue traz o payload no formato acordado e os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` |
| CA-06 | O cliente consegue validar o `X-Signature` recalculando o HMAC-SHA256 do corpo com a secret dele |
| CA-07 | Após rotacionar a secret, a secret anterior permanece válida por 24 horas; o protocolo de assinatura durante esse período deve ser aprovado por Segurança |
| CA-08 | Um endpoint indisponível é reprocessado nos intervalos de 1m, 5m, 30m, 2h e 12h; a contagem de envios segue a decisão técnica que resolver a ambiguidade registrada no FDD |
| CA-09 | Esgotadas as tentativas, o evento aparece na dead-letter queue com payload e motivo da falha |
| CA-10 | Um usuário `ADMIN` consegue reprocessar um evento da dead-letter queue; um usuário sem essa role recebe 403 |
| CA-11 | O replay gera registro de auditoria identificando quem executou a operação |
| CA-12 | O cliente consegue consultar o histórico de entregas de um webhook com sucesso/falha, payload, resposta e tempo de resposta |
| CA-13 | Se a gravação do evento falhar, a mudança de status não é persistida |
| CA-14 | Reiniciar a API não interrompe a entrega de eventos pendentes |
| CA-15 | O evento entregue reflete o estado do pedido no instante da transição, mesmo que o pedido mude depois |
| CA-16 | Nenhuma secret é exposta em listagens ou em logs |

---

## 12. Estratégia de testes e validação

### 12.1 Testes automatizados

A suíte segue o ferramental já existente no projeto (Vitest + Supertest, com reset de banco em
`tests/setup.ts` e fábricas em `tests/helpers/factories.ts`):

| Nível | Cobertura |
| --- | --- |
| Unitário | Cálculo do backoff, geração e verificação de HMAC-SHA256, validação de schema (URL `https`, filtro de status), limite de 64KB |
| Integração (API) | CRUD de webhooks, rotação de secret, histórico de entregas, replay de dead-letter com e sem role `ADMIN` |
| Integração (domínio) | Publicação do evento dentro da transação de `changeStatus`, incluindo o cenário de rollback, e o caso de nenhum assinante |
| Worker | Entrega bem-sucedida, timeout de 10 segundos, progressão até a dead-letter queue e estabilidade do `X-Event-Id` entre tentativas |
| Regressão | A suíte existente de pedidos (`tests/orders.test.ts`) deve permanecer verde após a alteração no `changeStatus` ([09:40] Bruno) |

### 12.2 Validação manual e de segurança

- **Revisão de segurança dedicada:** dois dias úteis reservados por Sofia para revisar HMAC e
  geração de secret antes do deploy ([09:46] Sofia).
- **Revisão de design:** sessão com Bruno e Diego sobre o documento antes do início da implementação
  ([09:50] Larissa).

### 12.3 Validação em produção

| Indicador | O que valida |
| --- | --- |
| Tempo entre commit da transição e entrega bem-sucedida | O objetivo O-1 (< 10 segundos) |
| Taxa de entregas bem-sucedidas na primeira tentativa | Saúde geral da integração dos clientes |
| Volume de eventos em dead-letter queue | Base para decidir sobre a notificação por e-mail adiada ([09:37] Larissa) |
| Volume de chamadas por cliente por minuto | Base para decidir sobre rate limiting de saída ([09:39] Larissa) |
| Redução do volume de chamadas a `GET /orders` pelos três clientes | O objetivo O-3 (migração de polling para webhook) |
