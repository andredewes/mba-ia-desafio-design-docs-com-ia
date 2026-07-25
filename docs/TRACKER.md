# Tracker de Rastreabilidade

Mapeamento de cada item registrado no pacote de documentação à sua origem, seja a transcrição da
reunião técnica (`TRANSCRICAO.md`) ou o código-fonte da aplicação.

**Regra de integridade:** nenhum requisito, decisão, restrição ou contrato aparece nos documentos
sem uma linha correspondente aqui. Se a coluna *Localização* não pôde ser preenchida, o item foi
removido do documento de origem.

**Convenções da coluna Fonte:**

- `TRANSCRICAO` → localização no formato `[hh:mm] Nome`, referente a `TRANSCRICAO.md`.
- `CODIGO` → caminho de arquivo real do repositório.

**Resumo de cobertura**

| Métrica | Valor |
| --- | --- |
| Total de itens rastreados | 223 |
| Itens com fonte `TRANSCRICAO` | 186 (83%) |
| Itens com fonte `CODIGO` | 37 (17%) |
| Documentos cobertos | PRD, RFC, FDD, ADR-001 a ADR-007 |

---

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Demanda formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | `docs/PRD.md` | Restrição | Feature é exclusivamente outbound: a plataforma envia, o cliente recebe | TRANSCRICAO | `[09:02] Marcos` |
| PRD-CTX-03 | `docs/PRD.md` | Contexto | Confirmação de que outbound webhook simplifica o desenho | TRANSCRICAO | `[09:03] Sofia` |
| PRD-CTX-04 | `docs/PRD.md` | Contexto | Aplicação não possui hoje nenhum mecanismo de notificação, evento ou fila | CODIGO | `src/routes/index.ts` |
| PRD-MOT-01 | `docs/PRD.md` | Problema | Clientes fazem polling em `GET /orders`, tornando a integração lenta e cara | TRANSCRICAO | `[09:00] Marcos` |
| PRD-MOT-02 | `docs/PRD.md` | Requisito Não Funcional | Abaixo de 10 segundos já é considerado "tempo real" pelos clientes | TRANSCRICAO | `[09:02] Marcos` |
| PRD-MOT-03 | `docs/PRD.md` | Risco de negócio | Atlas pode migrar para o concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-MOT-04 | `docs/PRD.md` | Contexto | `GET /orders` é hoje o único caminho de consulta disponível ao cliente | CODIGO | `src/modules/orders/order.routes.ts` |
| PRD-UC-01 | `docs/PRD.md` | Cenário de uso | Cliente escuta apenas `SHIPPED` e `DELIVERED` e atualiza expedição sem consultar a API | TRANSCRICAO | `[09:33] Marcos` |
| PRD-UC-02 | `docs/PRD.md` | Cenário de uso | Cliente em manutenção planejada de duas horas recebe os eventos após retry | TRANSCRICAO | `[09:16] Diego` |
| PRD-UC-03 | `docs/PRD.md` | Cenário de uso | Cliente consulta histórico de entregas para investigar suspeita de falha | TRANSCRICAO | `[09:34] Marcos` |
| PRD-UC-04 | `docs/PRD.md` | Cenário de uso | Cliente rotaciona secret suspeita e migra sistemas durante o grace period | TRANSCRICAO | `[09:21] Sofia` |
| PRD-UC-05 | `docs/PRD.md` | Cenário de uso | Suporte com role ADMIN reprocessa evento em dead-letter | TRANSCRICAO | `[09:36] Sofia` |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo/Métrica | Entrega em menos de 10 segundos, com polling de 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo/Métrica | Cinco intervalos de retry cobrem cerca de 15 horas; contagem de envios pendente | TRANSCRICAO | `[09:17] Diego` |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo/Métrica | Medir adoção entre os três clientes solicitantes, sem meta inventada | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo/Métrica | Zero divergência entre transições de status e eventos publicados | TRANSCRICAO | `[09:40] Bruno` |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo/Métrica | 100% das entregas com assinatura verificável | TRANSCRICAO | `[09:20] Sofia` |
| PRD-OBJ-06 | `docs/PRD.md` | Objetivo/Métrica | Entrega em 3 sprints, incluindo revisão de segurança | TRANSCRICAO | `[09:46] Larissa` |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook com URL, filtro de status e secret gerada pela plataforma | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Listagem dos webhooks de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Edição de webhook via PATCH | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Remoção de webhook via DELETE | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Filtro por status assinado, aplicado na inserção do evento | TRANSCRICAO | `[09:34] Bruno` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Rotação de secret via API com grace period de 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Histórico de entregas com sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Registro do evento na mesma transação da mudança de status, com rollback em caso de falha | TRANSCRICAO | `[09:40] Bruno` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Entrega assíncrona via HTTP POST por processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Retry nos intervalos 1m/5m/30m/2h/12h; contagem final depende de confirmação | TRANSCRICAO | `[09:17] Diego` |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Dead-letter queue com payload, motivo da falha e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Replay manual de dead-letter restrito a role ADMIN, com auditoria | TRANSCRICAO | `[09:36] Sofia` |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Assinatura HMAC-SHA256 do corpo enviada em `X-Signature` | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Headers `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type` | TRANSCRICAO | `[09:44] Diego` |
| PRD-FR-15 | `docs/PRD.md` | Requisito Funcional | Formato do payload sem os itens do pedido | TRANSCRICAO | `[09:43] Diego` |
| PRD-FR-16 | `docs/PRD.md` | Requisito Funcional | Payload como snapshot do estado do pedido na inserção | TRANSCRICAO | `[09:52] Larissa` |
| PRD-FR-17 | `docs/PRD.md` | Requisito Funcional | URL obrigatoriamente `https`; `http` recusado por validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-FR-18 | `docs/PRD.md` | Requisito Funcional | `customer_id` informado no body ou path, não extraído do JWT | TRANSCRICAO | `[09:32] Larissa` |
| PRD-FR-18b | `docs/PRD.md` | Restrição | JWT atual representa o usuário operador, não o cliente | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Polling de 2s limita o início do processamento; objetivo de entrega é abaixo de 10s | TRANSCRICAO | `[09:09] Diego` |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10 segundos por tentativa de entrega | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Limite de 64KB por payload, com erro em vez de truncamento | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-03b | `docs/PRD.md` | Requisito Não Funcional | Limite de payload classificado como RNF, não como decisão arquitetural | TRANSCRICAO | `[09:24] Larissa` |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Garantia at-least-once com deduplicação no cliente | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-05 | `docs/PRD.md` | Restrição | Ordenação garantida por pedido apenas enquanto houver um único worker | TRANSCRICAO | `[09:13] Larissa` |
| PRD-NFR-05b | `docs/PRD.md` | Restrição | Clientes nunca pediram garantia de ordenação global | TRANSCRICAO | `[09:14] Marcos` |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Worker fora da instância da API para sobreviver a restart | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-07 | `docs/PRD.md` | Restrição | Nenhuma infraestrutura nova; usar o MySQL existente | TRANSCRICAO | `[09:07] Diego` |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Reuso dos padrões do projeto e prefixo `WEBHOOK_` nos códigos de erro | TRANSCRICAO | `[09:30] Larissa` |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Uma secret por endpoint, nunca uma secret global | TRANSCRICAO | `[09:21] Sofia` |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | CRUD de configuração aceita qualquer role autenticada nesta fase | TRANSCRICAO | `[09:37] Sofia` |
| PRD-NFR-11 | `docs/PRD.md` | Requisito Não Funcional | Auditoria do replay: registrar quem executou | TRANSCRICAO | `[09:36] Sofia` |
| PRD-NFR-12 | `docs/PRD.md` | Restrição | Dois dias úteis de revisão de segurança antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| PRD-NFR-13 | `docs/PRD.md` | Requisito Não Funcional | Identificadores em UUID, seguindo o padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| PRD-NFR-13b | `docs/PRD.md` | Restrição | Todos os modelos existentes usam `@default(uuid()) @db.Char(36)` | CODIGO | `prisma/schema.prisma` |
| PRD-OUT-01 | `docs/PRD.md` | Fora de escopo | Notificação por e-mail quando o webhook do cliente falha | TRANSCRICAO | `[09:37] Larissa` |
| PRD-OUT-01b | `docs/PRD.md` | Fora de escopo | E-mail registrado como item "futuro" pelo PM | TRANSCRICAO | `[09:38] Marcos` |
| PRD-OUT-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual para o cliente; painel é projeto do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-OUT-03 | `docs/PRD.md` | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | `[09:39] Larissa` |
| PRD-OUT-04 | `docs/PRD.md` | Fora de escopo | Webhooks inbound | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OUT-05 | `docs/PRD.md` | Fora de escopo | Arquivamento das linhas entregues da outbox | TRANSCRICAO | `[09:08] Diego` |
| PRD-OUT-06 | `docs/PRD.md` | Fora de escopo | Escala horizontal do worker com particionamento ou lock pessimista | TRANSCRICAO | `[09:13] Diego` |
| PRD-OUT-07 | `docs/PRD.md` | Fora de escopo | Garantia de ordenação global de eventos | TRANSCRICAO | `[09:13] Larissa` |
| PRD-OUT-08 | `docs/PRD.md` | Fora de escopo | Eventos de outros domínios; nesta fase só `order.status_changed` | TRANSCRICAO | `[09:43] Diego` |
| PRD-RISK-01 | `docs/PRD.md` | Risco | Perda do cliente Atlas por atraso na entrega | TRANSCRICAO | `[09:00] Marcos` |
| PRD-RISK-02 | `docs/PRD.md` | Risco | Degradação da transação de mudança de status, já pesada hoje | TRANSCRICAO | `[09:04] Bruno` |
| PRD-RISK-03 | `docs/PRD.md` | Risco | Cliente sem deduplicação processa evento duplicado | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RISK-04 | `docs/PRD.md` | Risco | Vazamento de secret no lado do cliente, com precedente real | TRANSCRICAO | `[09:22] Diego` |
| PRD-RISK-05 | `docs/PRD.md` | Risco | Cliente indisponível por período longo; janela de 15h aceita pelo PM | TRANSCRICAO | `[09:17] Marcos` |
| PRD-RISK-06 | `docs/PRD.md` | Risco | Rajada de mudanças de status bombardeando o endpoint do cliente | TRANSCRICAO | `[09:38] Diego` |
| PRD-RISK-07 | `docs/PRD.md` | Risco | Crescimento indefinido da outbox degradando o worker | TRANSCRICAO | `[09:07] Bruno` |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Publicação do evento depende da transação de `changeStatus` | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Status assináveis vêm da máquina de estados existente | CODIGO | `src/modules/orders/order.status.ts` |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Autorização do replay reaproveita `requireRole` | TRANSCRICAO | `[09:36] Larissa` |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Novo processo de worker em produção (`npm run worker`) | TRANSCRICAO | `[09:11] Larissa` |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | `[09:50] Larissa` |
| PRD-DEP-06 | `docs/PRD.md` | Dependência | Documentação da integração no portal do desenvolvedor | TRANSCRICAO | `[09:40] Marcos` |
| PRD-AC-01 | `docs/PRD.md` | Critério de Aceitação | Cadastro devolve a secret gerada na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-AC-02 | `docs/PRD.md` | Critério de Aceitação | URL `http` recusada com erro de validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-AC-03 | `docs/PRD.md` | Critério de Aceitação | Status fora do filtro não gera entrega | TRANSCRICAO | `[09:34] Bruno` |
| PRD-AC-04 | `docs/PRD.md` | Critério de Aceitação | Entrega em menos de 10 segundos após a mudança de status | TRANSCRICAO | `[09:02] Marcos` |
| PRD-AC-05 | `docs/PRD.md` | Critério de Aceitação | Requisição entregue com os quatro headers acordados | TRANSCRICAO | `[09:44] Sofia` |
| PRD-AC-06 | `docs/PRD.md` | Critério de Aceitação | Assinatura verificável pelo cliente com HMAC-SHA256 | TRANSCRICAO | `[09:20] Sofia` |
| PRD-AC-07 | `docs/PRD.md` | Critério de Aceitação | Secret anterior válida por 24 horas; protocolo de assinatura requer aprovação | TRANSCRICAO | `[09:21] Sofia` |
| PRD-AC-08 | `docs/PRD.md` | Critério de Aceitação | Retry respeita os cinco intervalos; contagem de envios segue decisão pendente | TRANSCRICAO | `[09:17] Larissa` |
| PRD-AC-09 | `docs/PRD.md` | Critério de Aceitação | Evento esgotado aparece na dead-letter com payload e motivo | TRANSCRICAO | `[09:18] Diego` |
| PRD-AC-10 | `docs/PRD.md` | Critério de Aceitação | Replay negado para role diferente de ADMIN | TRANSCRICAO | `[09:36] Sofia` |
| PRD-AC-13 | `docs/PRD.md` | Critério de Aceitação | Falha na gravação do evento impede a mudança de status | TRANSCRICAO | `[09:41] Diego` |
| PRD-AC-14 | `docs/PRD.md` | Critério de Aceitação | Restart da API não interrompe a entrega de eventos | TRANSCRICAO | `[09:11] Diego` |
| PRD-AC-15 | `docs/PRD.md` | Critério de Aceitação | Evento reflete o estado do pedido no instante da transição | TRANSCRICAO | `[09:52] Diego` |
| PRD-TEST-01 | `docs/PRD.md` | Estratégia de teste | Suíte existente de pedidos deve permanecer verde | CODIGO | `tests/orders.test.ts` |
| PRD-TEST-02 | `docs/PRD.md` | Estratégia de teste | Reset de banco entre testes estendido para as novas tabelas | CODIGO | `tests/setup.ts` |
| PRD-TEST-03 | `docs/PRD.md` | Estratégia de teste | Revisão de segurança de HMAC e geração de secret antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| RFC-META-01 | `docs/RFC.md` | Metadado | Revisores do RFC são os participantes da reunião | TRANSCRICAO | `[09:00] Larissa` |
| RFC-META-02 | `docs/RFC.md` | Metadado | Prazo estimado de três sprints incluindo revisão de segurança | TRANSCRICAO | `[09:47] Larissa` |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Pergunta inicial: síncrono no service ou fila/outbox | TRANSCRICAO | `[09:03] Larissa` |
| RFC-CTX-02 | `docs/RFC.md` | Contexto | Ciclo de vida do pedido controlado por máquina de estados | CODIGO | `src/modules/orders/order.status.ts` |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Outbox transacional no MySQL como núcleo da proposta | TRANSCRICAO | `[09:06] Diego` |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Worker separado em polling de 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| RFC-PROP-03 | `docs/RFC.md` | Decisão | Cinco intervalos de backoff e DLQ dedicada; contagem de envios pendente | TRANSCRICAO | `[09:17] Diego` |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | `[09:22] Sofia` |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | Garantia at-least-once com `X-Event-Id` | TRANSCRICAO | `[09:26] Larissa` |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Módulo `src/modules/webhooks` seguindo o padrão dos demais | TRANSCRICAO | `[09:27] Bruno` |
| RFC-PROP-07 | `docs/RFC.md` | Decisão | Filtro de assinantes aplicado na inserção da outbox | TRANSCRICAO | `[09:34] Diego` |
| RFC-PROP-08 | `docs/RFC.md` | Restrição | Endpoints versionados sob o prefixo `/api/v1` | CODIGO | `src/app.ts` |
| RFC-ALT-01 | `docs/RFC.md` | Trade-off | Disparo HTTP síncrono descartado por travar a transação de status | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-01b | `docs/RFC.md` | Trade-off | Rollback da mudança de status por falha do cliente é inaceitável | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-02 | `docs/RFC.md` | Trade-off | Redis Streams descartado por exigir infraestrutura nova | TRANSCRICAO | `[09:07] Larissa` |
| RFC-ALT-02b | `docs/RFC.md` | Trade-off | Redis Cluster considerado overengineering para um time pequeno | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md` | Trade-off | Trigger de banco descartada: MySQL não tem NOTIFY/LISTEN | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-04 | `docs/RFC.md` | Trade-off | Política de 3 tentativas descartada por cobrir só ~30 minutos | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-04b | `docs/RFC.md` | Trade-off | Proposta original de 3 tentativas como opção mais agressiva | TRANSCRICAO | `[09:16] Bruno` |
| RFC-ALT-05 | `docs/RFC.md` | Trade-off | Exactly-once descartado por exigir coordenação dos dois lados | TRANSCRICAO | `[09:25] Diego` |
| RFC-ALT-06 | `docs/RFC.md` | Trade-off | DLQ como flag na própria outbox descartada em favor de tabela dedicada | TRANSCRICAO | `[09:17] Larissa` |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída: observar e implementar se virar problema | TRANSCRICAO | `[09:38] Diego` |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em aberto | Notificação por e-mail sobre webhook com falha adiada para próxima fase | TRANSCRICAO | `[09:37] Marcos` |
| RFC-OPEN-03 | `docs/RFC.md` | Questão em aberto | Estratégia de escala com múltiplos workers não decidida | TRANSCRICAO | `[09:12] Diego` |
| RFC-OPEN-04 | `docs/RFC.md` | Questão em aberto | Arquivamento das linhas entregues da outbox sem definição | TRANSCRICAO | `[09:08] Diego` |
| RFC-OPEN-05 | `docs/RFC.md` | Questão em aberto | Endurecimento futuro das permissões do CRUD de configuração | TRANSCRICAO | `[09:37] Sofia` |
| RFC-OPEN-06 | `docs/RFC.md` | Questão em aberto | Cinco envios totais ou cinco retentativas após o envio inicial | TRANSCRICAO | `[09:48] Larissa` |
| RFC-OPEN-07 | `docs/RFC.md` | Questão em aberto | Protocolo de assinatura durante o grace period não foi definido | TRANSCRICAO | `[09:21] Sofia` |
| RFC-OPEN-08 | `docs/RFC.md` | Questão em aberto | Tratamento de eventos pendentes ao remover um endpoint | TRANSCRICAO | `[09:33] Bruno` |
| RFC-OPEN-09 | `docs/RFC.md` | Questão em aberto | Recuperação de eventos em `PROCESSING` após crash | TRANSCRICAO | `[09:24] Diego` |
| RFC-OPEN-10 | `docs/RFC.md` | Questão em aberto | Granularidade do `event_id` no fan-out para múltiplos endpoints | TRANSCRICAO | `[09:25] Diego` |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | `changeStatus` passa a chamar `publishWebhookEvent(tx, ...)` | TRANSCRICAO | `[09:41] Bruno` |
| RFC-IMP-02 | `docs/RFC.md` | Impacto | Middleware de erro não precisa de alteração | TRANSCRICAO | `[09:29] Bruno` |
| RFC-IMP-03 | `docs/RFC.md` | Impacto | Novo processo a implantar e monitorar | TRANSCRICAO | `[09:11] Larissa` |
| RFC-NEXT-01 | `docs/RFC.md` | Processo | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | `[09:50] Larissa` |
| FDD-CTX-01 | `docs/FDD.md` | Contexto | `changeStatus` executa validação, estoque, update e histórico numa transação | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-CTX-02 | `docs/FDD.md` | Contexto | Transições válidas definidas por `canTransition` | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-OT-01 | `docs/FDD.md` | Objetivo técnico | Atomicidade entre mudança de status e registro do evento | TRANSCRICAO | `[09:06] Diego` |
| FDD-OT-02 | `docs/FDD.md` | Objetivo técnico | Entrega abaixo de 10 segundos com polling de 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| FDD-OT-03 | `docs/FDD.md` | Objetivo técnico | Tolerar indisponibilidade de até ~15 horas | TRANSCRICAO | `[09:17] Diego` |
| FDD-MODEL-01 | `docs/FDD.md` | Decisão | Tabela `webhook_outbox` com payload, status e tentativas | TRANSCRICAO | `[09:06] Diego` |
| FDD-MODEL-02 | `docs/FDD.md` | Decisão | Índices em status do evento e em `created_at` | TRANSCRICAO | `[09:08] Diego` |
| FDD-MODEL-03 | `docs/FDD.md` | Decisão | Tabela de configuração com url, secret, customer_id e estado ativo | TRANSCRICAO | `[09:21] Bruno` |
| FDD-MODEL-04 | `docs/FDD.md` | Decisão | Tabela `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| FDD-MODEL-05 | `docs/FDD.md` | Decisão | Tabela de entregas para suportar o histórico consultável | TRANSCRICAO | `[09:34] Marcos` |
| FDD-MODEL-06 | `docs/FDD.md` | Restrição | Convenções `@db.Char(36)` e `@@map` em snake_case | CODIGO | `prisma/schema.prisma` |
| FDD-FLOW-01 | `docs/FDD.md` | Fluxo | Inserção do evento dentro da transação, com rollback em caso de falha | TRANSCRICAO | `[09:40] Bruno` |
| FDD-FLOW-02 | `docs/FDD.md` | Fluxo | Nenhuma inserção quando não há assinante do status | TRANSCRICAO | `[09:34] Bruno` |
| FDD-FLOW-03 | `docs/FDD.md` | Fluxo | `event_id` UUID gerado na entrada do evento na outbox | TRANSCRICAO | `[09:25] Diego` |
| FDD-FLOW-04 | `docs/FDD.md` | Fluxo | Worker lê pendentes mais antigos em batch pequeno a cada 2s | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLOW-05 | `docs/FDD.md` | Fluxo | Ordem de processamento por `created_at` com worker único | TRANSCRICAO | `[09:12] Diego` |
| FDD-FLOW-06 | `docs/FDD.md` | Fluxo | Timeout de 10 segundos trata cliente lento como falha | TRANSCRICAO | `[09:42] Diego` |
| FDD-FLOW-07 | `docs/FDD.md` | Fluxo | Cinco intervalos de backoff preservados; contagem de envios pendente | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLOW-08 | `docs/FDD.md` | Fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| FDD-PAYLOAD-01 | `docs/FDD.md` | Contrato | Campos do payload do evento `order.status_changed` | TRANSCRICAO | `[09:43] Diego` |
| FDD-PAYLOAD-02 | `docs/FDD.md` | Contrato | Itens do pedido não são enviados; cliente consulta `GET /orders/:id` | TRANSCRICAO | `[09:44] Bruno` |
| FDD-PAYLOAD-03 | `docs/FDD.md` | Contrato | Payload é snapshot renderizado na inserção | TRANSCRICAO | `[09:52] Larissa` |
| FDD-HEADER-01 | `docs/FDD.md` | Contrato | `X-Event-Id`, `X-Signature`, `X-Timestamp` e `Content-Type` | TRANSCRICAO | `[09:44] Diego` |
| FDD-HEADER-02 | `docs/FDD.md` | Contrato | `X-Webhook-Id` para clientes com múltiplos endpoints | TRANSCRICAO | `[09:44] Sofia` |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | `POST /api/v1/webhooks` devolve a secret gerada na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | `GET /api/v1/webhooks` lista webhooks de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | `PATCH /api/v1/webhooks/:id` edita o webhook | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | `DELETE /api/v1/webhooks/:id` remove o webhook | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | `POST /api/v1/webhooks/:id/secret/rotate` rotaciona a secret | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | `GET /api/v1/webhooks/:id/deliveries` retorna o histórico | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` restrito a ADMIN | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Envelope de erro `{ error: { code, message, details } }` | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | Resposta paginada com `data` + `pagination` | CODIGO | `src/shared/http/response.ts` |
| FDD-CONTRATO-10 | `docs/FDD.md` | Contrato | `DELETE` responde 204 sem corpo, como nos demais módulos | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-ERRO-01 | `docs/FDD.md` | Restrição | Prefixo `WEBHOOK_` em todos os códigos de erro do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERRO-02 | `docs/FDD.md` | Contrato | Códigos `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERRO-03 | `docs/FDD.md` | Contrato | Erros derivam das classes existentes de erro HTTP | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-04 | `docs/FDD.md` | Contrato | `AppError` expõe `statusCode`, `errorCode` e `details` | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-ERRO-05 | `docs/FDD.md` | Contrato | `WEBHOOK_PAYLOAD_TOO_LARGE` para payload acima de 64KB | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-06 | `docs/FDD.md` | Contrato | `WEBHOOK_DELIVERY_TIMEOUT` para estouro do timeout de 10s | TRANSCRICAO | `[09:42] Diego` |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Evento com falha sai do caminho quente pelo backoff exponencial | TRANSCRICAO | `[09:15] Diego` |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Assinatura e payload estáveis entre tentativas | TRANSCRICAO | `[09:52] Diego` |
| FDD-RES-03 | `docs/FDD.md` | Resiliência | Shutdown gracioso no padrão de `bootstrap`/`shutdown` da API | CODIGO | `src/server.ts` |
| FDD-RES-04 | `docs/FDD.md` | Resiliência | Sem fallback por e-mail nesta fase; fallback é a DLQ com replay | TRANSCRICAO | `[09:37] Larissa` |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logging estruturado com o Pino já existente, sem biblioteca nova | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Redaction de campos sensíveis configurada no logger | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Log de auditoria obrigatório no replay de dead-letter | TRANSCRICAO | `[09:36] Sofia` |
| FDD-OBS-04 | `docs/FDD.md` | Observabilidade | Correlação por `requestId` propagado no header `X-Request-Id` | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-OBS-05 | `docs/FDD.md` | Observabilidade | Métrica de idade do evento pendente valida o SLA de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| FDD-OBS-06 | `docs/FDD.md` | Observabilidade | Volume de dead-letter embasa a decisão futura sobre e-mail | TRANSCRICAO | `[09:37] Marcos` |
| FDD-INT-01 | `docs/FDD.md` | Integração | `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada no `$transaction` | TRANSCRICAO | `[09:41] Bruno` |
| FDD-INT-01b | `docs/FDD.md` | Integração | Ponto exato da alteração no método `changeStatus` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Reuso de `AppError` e das classes de erro específicas | CODIGO | `src/shared/errors/index.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | `authenticate` e `requireRole('ADMIN')` aplicados às rotas | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Middleware de erro absorve os erros `WEBHOOK_*` sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Validação via `validate({ params, body, query })` com schemas Zod | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Schemas seguem o formato dos schemas de pedidos | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Registro do módulo em `buildControllers` e no router da API | CODIGO | `src/routes/index.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Nova entrada de processo do worker espelhando o server existente | TRANSCRICAO | `[09:11] Larissa` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Worker usa `PrismaClient` próprio, mesma `DATABASE_URL` | TRANSCRICAO | `[09:30] Bruno` |
| FDD-INT-09b | `docs/FDD.md` | Integração | `createPrismaClient()` disponível para instanciar o client | CODIGO | `src/config/database.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Variáveis do worker validadas pelo schema Zod de ambiente | CODIGO | `src/config/env.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Script `npm run worker` espelhando `dev` e `start` | CODIGO | `package.json` |
| FDD-INT-12 | `docs/FDD.md` | Integração | Lógica de processamento em `webhook.worker.ts` dentro do módulo | TRANSCRICAO | `[09:28] Bruno` |
| FDD-DEP-01 | `docs/FDD.md` | Dependência | MySQL 8.0 já provisionado no ambiente de desenvolvimento | CODIGO | `docker-compose.yml` |
| FDD-DEP-02 | `docs/FDD.md` | Dependência | Sem broker ou infraestrutura adicional | TRANSCRICAO | `[09:07] Diego` |
| FDD-DEP-03 | `docs/FDD.md` | Dependência | HMAC-SHA256 disponível sem nova dependência | TRANSCRICAO | `[09:20] Sofia` |
| FDD-DEP-04 | `docs/FDD.md` | Dependência | Deploy passa a exigir dois processos: API e worker | TRANSCRICAO | `[09:11] Diego` |
| FDD-CAT-01 | `docs/FDD.md` | Critério de aceite técnico | Rollback não deixa linha órfã na outbox | TRANSCRICAO | `[09:40] Bruno` |
| FDD-CAT-02 | `docs/FDD.md` | Critério de aceite técnico | Sem assinante, nenhuma linha é inserida | TRANSCRICAO | `[09:34] Diego` |
| FDD-CAT-03 | `docs/FDD.md` | Critério de aceite técnico | Grace period de 24h, com protocolo sujeito à aprovação de Segurança | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CAT-04 | `docs/FDD.md` | Critério de aceite técnico | `X-Event-Id` idêntico entre tentativas e replay | TRANSCRICAO | `[09:25] Diego` |
| FDD-CAT-05 | `docs/FDD.md` | Critério de aceite técnico | Suíte de pedidos permanece verde após a alteração | CODIGO | `tests/orders.test.ts` |
| FDD-TEST-01 | `docs/FDD.md` | Estratégia de teste | Fábricas de dados de teste estendidas para webhooks | CODIGO | `tests/helpers/factories.ts` |
| ADR-001 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Outbox transacional no MySQL, sem infraestrutura nova | TRANSCRICAO | `[09:08] Larissa` |
| ADR-001-ALT | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Trade-off | Redis Streams descartado por custo de infraestrutura | TRANSCRICAO | `[09:07] Diego` |
| ADR-001-CONS | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Consequência | Arquivamento das linhas entregues fica fora do escopo | TRANSCRICAO | `[09:08] Diego` |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | `[09:10] Larissa` |
| ADR-002-ALT | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Trade-off | Trigger de banco descartada por falta de listener no MySQL | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-CONS | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Consequência | Ordenação garantida apenas por pedido e com worker único | TRANSCRICAO | `[09:13] Larissa` |
| ADR-003 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | Cinco intervalos de backoff e DLQ dedicada; contagem de envios pendente | TRANSCRICAO | `[09:17] Larissa` |
| ADR-003-ALT | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Trade-off | Retry indefinido descartado por deixar evento pendurado | TRANSCRICAO | `[09:15] Diego` |
| ADR-003-CONS | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Consequência | Janela de até 15 horas aceita pelo PM | TRANSCRICAO | `[09:17] Marcos` |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | HMAC-SHA256, secret por endpoint e rotação com grace de 24h | TRANSCRICAO | `[09:22] Sofia` |
| ADR-004-ALT | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Trade-off | Secret global descartada por raio de vazamento total | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-CONS | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Consequência | Revisão de segurança de dois dias antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| ADR-005 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | Entrega at-least-once com deduplicação por `X-Event-Id` | TRANSCRICAO | `[09:26] Larissa` |
| ADR-005-ALT | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Trade-off | Exactly-once descartado pela complexidade de coordenação | TRANSCRICAO | `[09:25] Diego` |
| ADR-005-CONS | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Consequência | Responsabilidade de deduplicação transferida ao cliente | TRANSCRICAO | `[09:25] Sofia` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | `[09:30] Larissa` |
| ADR-006-COD | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Restrição | Estrutura modular controller/service/repository/routes/schemas | CODIGO | `src/modules/orders/order.routes.ts` |
| ADR-006-COD2 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Restrição | Composição de dependências em `buildControllers` | CODIGO | `src/app.ts` |
| ADR-006-ALT | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Trade-off | Injetar o repositório inteiro no `OrderService` foi descartado | TRANSCRICAO | `[09:41] Diego` |
| ADR-007 | `docs/adrs/ADR-007-payload-snapshot-na-outbox.md` | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | `[09:52] Larissa` |
| ADR-007-ALT | `docs/adrs/ADR-007-payload-snapshot-na-outbox.md` | Trade-off | Guardar apenas `order_id` e renderizar no envio foi descartado | TRANSCRICAO | `[09:51] Bruno` |
| ADR-007-CONS | `docs/adrs/ADR-007-payload-snapshot-na-outbox.md` | Consequência | Duplicação de dados já presentes nas tabelas de pedidos | CODIGO | `prisma/schema.prisma` |

---

## Itens da reunião conscientemente **não** transformados em requisito

Registrados aqui para deixar explícito que foram avaliados e descartados, e não esquecidos.

| Item | Situação | Localização |
| --- | --- | --- |
| Notificação por e-mail ao cliente após falhas consecutivas | Descartado desta fase | `[09:37] Larissa` |
| Dashboard visual para o cliente | Descartado; projeto do time de frontend | `[09:40] Larissa` |
| Rate limiting de saída | Adiado; observar e decidir depois | `[09:39] Larissa` |
| Webhooks inbound | Fora de escopo desde o início | `[09:02] Marcos` |
| Trigger de banco para notificação reativa | Alternativa descartada | `[09:09] Diego` |
| Redis Streams como fila | Alternativa descartada | `[09:07] Diego` |
| Política de 3 tentativas | Alternativa descartada | `[09:16] Diego` |
| Garantia exactly-once | Alternativa descartada | `[09:25] Diego` |
| Múltiplos workers com particionamento por `order_id` ou lock pessimista | Adiado | `[09:13] Diego` |
| Arquivamento de linhas entregues após ~30 dias | Fora do escopo da feature | `[09:08] Diego` |
| Endurecimento de permissões no CRUD de configuração | Adiado | `[09:37] Sofia` |
