# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da
transcrição de uma reunião técnica (`TRANSCRICAO.md`) e do código-fonte de um Order Management
System em Node.js + TypeScript.

> O enunciado original do desafio está no repositório base:
> <https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia>

---

## Sobre o desafio

Uma empresa que opera um OMS em produção decidiu construir um sistema de webhooks para notificar
clientes B2B quando o status dos pedidos deles muda. A decisão técnica foi tomada em uma call de
55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança — e nada foi
registrado além da transcrição literal da conversa. O desafio é transformar essa conversa, somada ao
código existente da aplicação, em um pacote de documentação técnica acionável: PRD, RFC, FDD, ADRs e
um tracker de rastreabilidade.

A parte difícil não é gerar texto: é **filtrar**. A reunião contém decisões fechadas, requisitos
explícitos, alternativas descartadas, ideias adiadas para fases futuras e detalhes técnicos
secundários misturados no mesmo fluxo de fala. Coisas como notificação por e-mail, dashboard visual
e rate limiting de saída foram explicitamente cortadas, e precisam aparecer como *fora de escopo*, e
não como requisito. Além disso, todo item registrado precisa ser rastreável a um timestamp da
transcrição ou a um arquivo real do repositório — o que transforma a IA de "geradora de documento"
em "extratora e organizadora de informação verificável".

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **GitHub Copilot em modo agente**, dentro do VS Code | Ferramenta principal. Leu a transcrição e o código diretamente no workspace, mapeou os pontos de integração, produziu os documentos e aplicou as revisões. O modelo específico não é registrado aqui porque houve troca de LLM durante a revisão |
| **Ferramentas do workspace do VS Code** | Leitura dirigida dos arquivos, busca de referências, edição dos Markdown e verificação de diagnósticos |
| **PowerShell** | Auditorias reproduzíveis: validação dos timestamps contra `TRANSCRICAO.md`, existência dos caminhos citados e contagem das linhas do tracker por tipo de fonte |
| **Segunda passagem com outra LLM** | Revisão independente solicitada após a primeira geração; encontrou ambiguidades de retry, identidade do evento, SLA e decisões de implementação sem fonte |

---

## Workflow adotado

A ordem de produção seguiu a lógica de **construir de baixo para cima**: primeiro as decisões, depois
a proposta, depois o detalhe de implementação e, por último, a camada de produto.

```
1. Contextualização    → leitura integral da transcrição + varredura do código
2. Extração dirigida   → tabela crua de decisões / requisitos / descartes / ganchos no código
3. ADRs (001–007)      → uma decisão por arquivo, com alternativa real e trade-off explícito
4. RFC                 → consolida a proposta em nível de arquitetura, linka os ADRs
5. FDD                 → desce ao contrato, à matriz de erros e à integração com o código
6. PRD                 → sobe ao nível de produto, já com tudo decidido
7. TRACKER             → varredura final ligando cada item à origem
8. README              → documentação do processo
9. Revisão final       → checklist de critérios de aceite, item por item
```

Duas regras de trabalho valeram do início ao fim:

1. **Nenhuma frase sem origem.** Se não dava para apontar o timestamp ou o arquivo, a frase saía do
   documento. O tracker foi construído como mecanismo de verificação, não como entregável decorativo.
2. **Cada documento em sua altura.** Sempre que um trecho do RFC começava a virar especificação de
   endpoint, ele era movido para o FDD; sempre que o FDD começava a justificar uma escolha, a
   justificativa ia para o ADR correspondente.

### Passo a passo da interação com a IA

| Etapa | Interação com a IA | Validação complementar |
| --- | --- | --- |
| Contextualização | Leitura integral de `TRANSCRICAO.md` e dos pontos relevantes de `src/`, `prisma/` e `tests/` | Conferência automatizada dos arquivos citados |
| Extração | Separação de decisões fechadas, requisitos, descartes e ganchos no código | Comparação dos timestamps com a transcrição |
| ADRs | Um arquivo por decisão, formato MADR, com alternativa da reunião e trade-off | Verificação de estrutura e links |
| RFC | Consolidação em nível de arquitetura, com alternativas e questões em aberto | Revisão para evitar duplicação do FDD |
| FDD | Contratos, matriz de erros `WEBHOOK_*`, fluxos e integração com arquivos reais | Busca de decisões técnicas sem fonte e de contradições internas |
| PRD | Consolidação em nível de produto | Revisão de metas para remover compromissos não definidos na reunião |
| Tracker | Mapeamento de requisitos, decisões e restrições | Contagem automatizada por tipo de fonte e validação dos timestamps |

---

## Prompts customizados

### Prompt 1 — Extração dirigida e classificação da transcrição

Este foi o prompt mais importante do processo. Pedir "resuma a reunião" produz um texto genérico que
mistura decisão com especulação. O prompt abaixo força a IA a **classificar** cada informação e a
**declarar a incerteza** em vez de preencher lacunas.

```text
Leia TRANSCRICAO.md integralmente. Não gere nenhum documento ainda.

Produza uma tabela com uma linha por informação técnica ou de produto mencionada na
reunião, com as colunas:

  timestamp | falante | conteudo | classificacao | confianca

A coluna "classificacao" deve usar EXATAMENTE um destes valores:
  DECISAO_FECHADA     - foi decidido e confirmado por alguém na call
  REQUISITO_FUNCIONAL - comportamento que o sistema deve ter
  REQUISITO_NAO_FUNC  - restrição de qualidade, limite, prazo, política
  ALTERNATIVA_DESCARTADA - foi proposto e rejeitado, com motivo
  ADIADO_PROXIMA_FASE - reconhecido como válido mas tirado deste escopo
  FORA_DE_ESCOPO      - explicitamente recusado
  QUESTAO_EM_ABERTO   - levantado e não decidido
  GANCHO_NO_CODIGO    - referência a arquivo, classe ou padrão da aplicação
  RUIDO               - conversa social, atraso, despedida

Regras:
- Se a mesma decisão foi discutida em vários momentos, crie uma linha por momento.
- Para ALTERNATIVA_DESCARTADA, o conteúdo DEVE incluir o motivo do descarte dito na call.
- Não infira nada que não esteja dito. Se ficou ambíguo, use confianca = BAIXA e
  explique a ambiguidade.
- Não invente timestamps. Copie exatamente do arquivo.
```

### Prompt 2 — Geração de ADR ancorada em evidência

```text
Escreva o ADR referente à decisão "<DECISÃO>" no formato MADR, em português,
no arquivo docs/adrs/ADR-<NNN>-<kebab-case>.md.

Seções obrigatórias: Status, Contexto, Decisão, Alternativas Consideradas,
Consequências (positivas e negativas), Decisões relacionadas.

Restrições rígidas:
1. Toda afirmação factual deve vir acompanhada da referência [hh:mm] Nome do falante,
   copiada da transcrição. Se você não consegue citar, NÃO escreva a afirmação.
2. A seção "Alternativas Consideradas" deve conter pelo menos uma alternativa REAL
   discutida na reunião, com o trade-off que motivou o descarte nas palavras da call.
   Alternativa genérica de livro-texto não serve.
3. As consequências negativas não podem ser amenizadas. Escreva o custo real da decisão.
   Se o time aceitou conscientemente um custo, cite quem aceitou e quando.
4. Não repita detalhe de implementação (payload, endpoint, status code). Isso é do FDD.
5. Antes de escrever, verifique no código se algum arquivo real sustenta o contexto.
   Se sim, cite o caminho exato. Se você não tem certeza de que o arquivo existe,
   não o cite.
```

### Prompt 3 — Verificação anti-alucinação (rodado ao final)

```text
Modo auditoria. Não reescreva nada ainda, apenas reporte.

Para cada arquivo em docs/ (PRD.md, RFC.md, FDD.md, TRACKER.md e todos os ADRs):

1. Liste TODOS os caminhos de arquivo do código citados no texto e verifique um a um
   se eles existem no repositório. Reporte os inexistentes.
2. Liste TODAS as referências [hh:mm] Nome e verifique se o timestamp existe em
   TRANSCRICAO.md e se o falante daquele timestamp é o citado. Reporte divergências.
3. Aponte qualquer afirmação técnica sem referência a timestamp nem a arquivo.
4. Aponte trechos duplicados entre RFC e FDD, indicando em qual documento cada trecho
   deveria ficar.

Saída: uma tabela por problema encontrado, com arquivo, trecho e tipo do problema.
Se um documento não tiver problemas, diga explicitamente.
```

---

## Iterações e ajustes

Foram **duas rodadas principais**: geração do pacote e revisão independente após a troca de LLM.
A segunda rodada encontrou problemas concretos que passaram pela primeira auditoria de timestamps,
mostrando que referência válida não garante interpretação correta.

### Ajuste 1 — Retry incompatível com a janela de 15 horas

A primeira versão dizia "5 tentativas", mas executava a quinta em 2h36 e usava as 12 horas apenas
como espera antes da DLQ. Isso contrariava a explicação de Diego, que colocou a última tentativa
depois dos cinco intervalos ([09:17]), e o resumo de Larissa, que falou em "total 5 tentativas"
([09:48]). A revisão não inventou uma resolução: preservou 1m/5m/30m/2h/12h e registrou a contagem
de envios como questão aberta para Larissa e Diego.

### Ajuste 2 — Identidade do evento ausente no modelo

O contrato usava `eventId` em headers, logs e histórico, mas o modelo `WebhookOutbox` não explicava
onde ele era persistido. A correção definiu que o próprio UUID `WebhookOutbox.id` é o `event_id`,
mantido entre retry e replay, conforme a decisão de gerar o identificador ao inserir na outbox
([09:25] Diego).

### Ajuste 3 — Polling confundido com SLA de entrega

A primeira versão afirmava entrega em até 2 segundos. Polling de 2 segundos garante apenas quando o
worker começa a processar; a chamada HTTP pode levar até o timeout de 10 segundos ([09:42] Diego).
PRD, FDD e ADR-002 passaram a separar o início do processamento do objetivo de entrega abaixo de
10 segundos ([09:02] Marcos).

### Ajuste 4 — Decisões de implementação sem fonte

O FDD prescrevia duas assinaturas separadas por vírgula durante rotação, descarte de eventos ao
excluir um endpoint e recuperação automática de linhas em `PROCESSING`. A reunião não decidiu
nenhum desses protocolos. Eles foram removidos do contrato fechado e registrados como questões
abertas sobre rotação, remoção e recuperação após crash.

### Ajuste 5 — Meta de adoção não acordada

Três clientes solicitaram a feature, mas apenas a Atlas associou a demanda a um prazo comercial
([09:00] Marcos). A meta "migrar os três até o fim do trimestre" foi removida; o PRD agora mede
adoção entre os solicitantes sem atribuir prazo ou compromisso não decidido.

### Auditoria final

O tracker foi recalculado após os ajustes: são 223 itens rastreáveis, 186 com fonte na transcrição
(83,4%) e 37 no código. As novas questões abertas também têm origem explícita.

---

## Como navegar a entrega

```
.
├── README.md                                      ← este arquivo (processo de produção)
├── TRANSCRICAO.md                                 ← fonte primária (não alterado)
├── docs/
│   ├── PRD.md                                     ← problema, escopo, requisitos, métricas
│   ├── RFC.md                                     ← proposta técnica, alternativas, questões em aberto
│   ├── FDD.md                                     ← contratos, fluxos, erros, integração com o código
│   ├── TRACKER.md                                 ← rastreabilidade item a item
│   └── adrs/
│       ├── README.md                              ← índice dos ADRs
│       ├── ADR-001-outbox-transacional-no-mysql.md
│       ├── ADR-002-worker-separado-com-polling.md
│       ├── ADR-003-retry-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes.md
│       └── ADR-007-payload-snapshot-na-outbox.md
├── src/ · prisma/ · tests/                        ← aplicação existente (não alterada)
```

### Ordem de leitura sugerida

| # | Documento | Por quê |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](./docs/PRD.md) | Entender o problema, quem é impactado e o que está dentro e fora de escopo |
| 2 | [`docs/RFC.md`](./docs/RFC.md) | Ver a proposta técnica em nível de arquitetura, as alternativas descartadas e o que ficou em aberto |
| 3 | [`docs/adrs/`](./docs/adrs/) | Aprofundar cada decisão isoladamente, com seu trade-off explícito |
| 4 | [`docs/FDD.md`](./docs/FDD.md) | Descer ao detalhe de implementação: contratos, erros, fluxos e integração com o código |
| 5 | [`docs/TRACKER.md`](./docs/TRACKER.md) | Auditar a origem de qualquer item dos documentos anteriores |

Quem for **implementar** pode inverter a ordem e começar pelo FDD, usando a seção
"Integração com o sistema existente" como ponto de partida e voltando aos ADRs sempre que precisar
entender o porquê de uma restrição.

---

## Sobre a aplicação existente

O repositório contém um Order Management System funcional em Node.js + TypeScript, com módulos de
autenticação, usuários, clientes, produtos e pedidos, banco MySQL via Prisma, máquina de estados de
pedido, controle transacional de estoque e auditoria de mudanças de status. A entrega deste desafio
é **puramente documental**: nenhum arquivo de `src/`, `prisma/` ou `tests/` foi modificado.

Para subir o ambiente da aplicação:

```bash
docker compose up -d      # MySQL 8.0
npm install
npm run db:migrate
npm run db:seed
npm run dev
```
