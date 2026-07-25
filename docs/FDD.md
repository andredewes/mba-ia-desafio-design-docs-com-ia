# FDD ΓÇö Sistema de Webhooks de Notifica├º├úo de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Sistema de Webhooks de Notifica├º├úo de Pedidos |
| **Autor** | Larissa (Tech Lead), com Bruno (time de Pedidos) e Diego (time de Plataforma) |
| **Status** | Pronto para implementa├º├úo |
| **Base** | [RFC](./RFC.md) e [ADRs 001ΓÇô007](./adrs/) |
| **Rastreabilidade** | [TRACKER.md](./TRACKER.md) |
| **Estimativa** | 3 sprints, incluindo revis├úo de seguran├ºa ([09:46] Larissa) |

> Este documento ├⌐ o "como construir". As decis├╡es e seus porqu├¬s est├úo nos [ADRs](./adrs/); a
> proposta em n├¡vel de arquitetura est├í no [RFC](./RFC.md); o problema de produto est├í no
> [PRD](./PRD.md).

---

## 1. Contexto e motiva├º├úo t├⌐cnica

O OMS atual n├úo possui nenhum mecanismo de notifica├º├úo externa, evento ou fila. A mudan├ºa de status
de um pedido acontece integralmente dentro de `OrderService.changeStatus`
(`src/modules/orders/order.service.ts`), em uma ├║nica transa├º├úo Prisma que:

1. l├¬ o pedido com seus itens;
2. valida a transi├º├úo contra a m├íquina de estados de `src/modules/orders/order.status.ts`
   (`canTransition`);
3. debita ou rep├╡e estoque (`shouldDebitStock` / `shouldReplenishStock`);
4. atualiza `orders.status`;
5. insere uma linha em `order_status_history`.

Qualquer notifica├º├úo externa precisa se ancorar exatamente nesse ponto. Como uma chamada HTTP
s├¡ncrona dentro dessa transa├º├úo travaria mudan├ºas de status de outros pedidos quando um cliente
estivesse lento ([09:04] Bruno), a feature ├⌐ constru├¡da sobre o padr├úo **Outbox**: o evento ├⌐
gravado no banco na mesma transa├º├úo e entregue depois, de forma ass├¡ncrona, por um worker separado
([ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md),
[ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)).

---

## 2. Objetivos t├⌐cnicos

| # | Objetivo |
| --- | --- |
| OT-1 | Garantir atomicidade entre a mudan├ºa de status e o registro do evento, sem dual-write ([09:06] Diego) |
| OT-2 | Entregar o evento ao cliente em menos de 10 segundos no caminho feliz, com lat├¬ncia de polling de 2 segundos ([09:02] Marcos, [09:09] Diego) |
| OT-3 | Tolerar indisponibilidade do cliente por at├⌐ ~15 horas via retry com backoff ([09:17] Diego) |
| OT-4 | N├úo perder eventos: falha permanente vai para DLQ persistida e reprocess├ível ([09:18] Diego) |
| OT-5 | Permitir ao cliente verificar autenticidade e integridade do payload via HMAC-SHA256 ([09:20] Sofia) |
| OT-6 | N├úo introduzir infraestrutura, biblioteca de fila ou padr├úo de c├│digo novo no projeto ([09:07] Diego, [09:30] Larissa) |
| OT-7 | Isolar o ciclo de vida do processamento de eventos do ciclo de vida da API ([09:11] Diego) |

---

## 3. Escopo e exclus├╡es

### 3.1 No escopo

- Modelagem das tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e
  `webhook_dead_letter` + migration Prisma.
- M├│dulo `src/modules/webhooks/` (controller, service, repository, routes, schemas) ([09:27] Bruno).
- Nova entrada de processo para o worker e script `npm run worker` ([09:11] Larissa).
- Publica├º├úo do evento dentro da transa├º├úo de `changeStatus` via `publishWebhookEvent(tx, ...)`
  ([09:41] Bruno).
- Assinatura HMAC-SHA256, gera├º├úo e rota├º├úo de secret com grace period de 24h ([09:21] Sofia).
- CRUD de configura├º├úo, hist├│rico de entregas e replay administrativo de DLQ.

### 3.2 Fora do escopo

| Exclus├úo | Origem |
| --- | --- |
| Webhooks inbound (cliente enviando para n├│s) | [09:02] Marcos |
| Notifica├º├úo por e-mail quando o webhook do cliente falha | [09:37] Larissa |
| Rate limiting de sa├¡da por cliente | [09:39] Larissa |
| Dashboard visual para o cliente | [09:40] Larissa |
| Arquivamento/expurgo das linhas entregues da outbox | [09:08] Diego |
| Escala horizontal do worker (particionamento por `order_id`, lock pessimista) | [09:13] Diego |
| Eventos de outros dom├¡nios que n├úo `order.status_changed` | [09:43] Diego |

---

## 4. Modelo de dados

Todas as tabelas seguem as conven├º├╡es j├í presentes em `prisma/schema.prisma`: id em UUID
(`@default(uuid()) @db.Char(36)`) ([09:51] Larissa) e nome de tabela em snake_case via `@@map`.

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  FAILED
  DELIVERED
}

model WebhookEndpoint {
  id                     String    @id @default(uuid()) @db.Char(36)
  customerId             String    @db.Char(36)
  url                    String    @db.VarChar(500)
  secret                 String    @db.VarChar(128)
  previousSecret         String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  eventStatuses          Json
  active                 Boolean   @default(true)
  createdAt              DateTime  @default(now())
  updatedAt              DateTime  @updatedAt

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id                 String   @id @default(uuid()) @db.Char(36)
  outboxId           String   @db.Char(36)
  webhookId          String   @db.Char(36)
  attempt            Int
  success            Boolean
  responseStatusCode Int?
  responseBody       String?  @db.Text
  errorCode          String?  @db.VarChar(64)
  durationMs         Int
  attemptedAt        DateTime @default(now())

  @@index([webhookId, attemptedAt])
  @@index([outboxId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @db.Char(36)
  webhookId    String   @db.Char(36)
  eventType    String   @db.VarChar(64)
  payload      Json
  failureReason String  @db.VarChar(500)
  attempts     Int
  replayedAt   DateTime?
  replayedById String?  @db.Char(36)
  createdAt    DateTime @default(now())

  @@index([webhookId])
  @@index([createdAt])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:

- `WebhookOutbox.id` ├⌐ tamb├⌐m o `event_id` enviado ao cliente. Assim, retry e replay preservam o
  mesmo `X-Event-Id` sem criar um segundo identificador para o mesmo evento ([09:25] Diego).
- Os ├¡ndices em status e `created_at` da outbox atendem diretamente ao requisito levantado por Diego
  em [09:08] (ler s├│ os pendentes mais antigos em batch pequeno). O ├¡ndice composto
  `[status, nextAttemptAt]` ├⌐ a forma pr├ítica de suportar tamb├⌐m o backoff.
- `payload` guarda o **snapshot renderizado** ([ADR-007](./adrs/ADR-007-payload-snapshot-na-outbox.md)).
- `webhook_deliveries` existe para sustentar o requisito de hist├│rico com sucesso/falha, payload,
  response e tempo de resposta ([09:34] Marcos).
- `previousSecret` + `previousSecretExpiresAt` implementam o grace period de 24h ([09:21] Sofia).

---

## 5. Fluxos detalhados

### 5.1 Cria├º├úo do evento na outbox (dentro da transa├º├úo)

Ponto de integra├º├úo: `OrderService.changeStatus`, em `src/modules/orders/order.service.ts`.

```mermaid
sequenceDiagram
    participant C as OrderController
    participant S as OrderService.changeStatus
    participant TX as Prisma $transaction
    participant W as publishWebhookEvent(tx, ...)
    participant DB as MySQL

    C->>S: changeStatus(id, { toStatus }, userId)
    S->>TX: in├¡cio da transa├º├úo
    TX->>DB: findUnique(order + items)
    S->>S: canTransition(from, to) / shouldDebitStock / shouldReplenishStock
    TX->>DB: update orders.status
    TX->>DB: insert order_status_history
    S->>W: publishWebhookEvent(tx, order, from, to)
    W->>DB: select webhook_endpoints (customerId, active = true)
    alt nenhum endpoint ativo assina o status "to"
        W-->>S: no-op (nenhuma linha inserida)
    else h├í assinantes
        W->>W: renderiza payload snapshot + gera eventId (uuid)
        W->>W: valida tamanho do payload (<= 64KB)
        W->>DB: insert webhook_outbox (1 linha por endpoint assinante)
    end
    TX-->>S: commit
    S-->>C: order atualizado
```

Regras:

1. A inser├º├úo acontece **dentro da mesma transa├º├úo** que atualiza `orders` e `order_status_history`.
   Falha ao inserir na outbox provoca rollback da mudan├ºa de status inteira ([09:40] Bruno,
   [09:41] Diego).
2. A assinatura de eventos ├⌐ filtrada **na inser├º├úo**: se nenhum webhook ativo do cliente escuta
   aquele status, nada ├⌐ inserido ([09:34] Bruno e Diego).
3. Uma linha de outbox ├⌐ criada **por endpoint assinante**. O `id` UUID da pr├│pria linha ├⌐ tamb├⌐m o
  `event_id` enviado em `X-Event-Id`; n├úo existe um segundo identificador de evento. Isso aplica a
  decis├úo de gerar o UUID quando o evento entra na outbox ([09:25] Diego) sem duplicar identidade
  no modelo.
4. O payload excedendo 64KB gera erro `WEBHOOK_PAYLOAD_TOO_LARGE`; o evento n├úo ├⌐ enviado
   ([09:23] Sofia, [09:24] Diego/Larissa).
5. Assinatura da fun├º├úo: `publishWebhookEvent(tx, order, fromStatus, toStatus)` ΓÇö fun├º├úo que recebe o
   client transacional, em vez de injetar o reposit├│rio inteiro no `OrderService` ([09:41] Bruno e
   Diego).

### 5.2 Processamento pelo worker

```mermaid
sequenceDiagram
    participant L as Loop (2s)
    participant P as WebhookProcessor
    participant DB as MySQL
    participant CL as Endpoint do cliente

    loop a cada 2 segundos
        L->>DB: select webhook_outbox where status=PENDING and nextAttemptAt <= now() order by createdAt asc limit N
        DB-->>P: batch de eventos
        loop para cada evento
            P->>DB: update status = PROCESSING
            P->>P: assina payload (HMAC-SHA256) e monta headers
            P->>CL: POST url (timeout 10s)
            alt 2xx
                CL-->>P: 200
                P->>DB: insert webhook_deliveries (success = true)
                P->>DB: update status = DELIVERED
            else falha / timeout / n├úo-2xx
                CL-->>P: erro
                P->>DB: insert webhook_deliveries (success = false)
                alt attempts + 1 < 5
                    P->>DB: update status = PENDING, attempts++, nextAttemptAt = now() + backoff
                else esgotou
                    P->>DB: insert webhook_dead_letter
                    P->>DB: update status = FAILED
                end
            end
        end
    end
```

Detalhes:

- Intervalo de polling: **2 segundos** ([09:09] Diego).
- Ordena├º├úo por `created_at` ascendente, o que d├í ordena├º├úo por pedido enquanto houver um ├║nico
  worker ([09:12] Diego).
- Batch pequeno por ciclo ([09:08] Diego).
- Timeout HTTP: **10 segundos**; sem resposta nesse prazo ├⌐ falha ([09:42] Diego).
- Respostas fora da faixa 2xx s├úo tratadas como falha e entram no ciclo de retry.
- O processo tem sua pr├│pria inst├óncia de `PrismaClient`, apontando para a mesma `DATABASE_URL`
  ([09:30] Bruno).

**Requisi├º├úo enviada ao cliente:**

```http
POST /webhooks/oms HTTP/1.1
Host: erp.atlascomercial.com.br
Content-Type: application/json
X-Event-Id: 6f9c3a9e-9a2f-4a1e-9f0e-6c9d1a2b3c4d
X-Webhook-Id: 2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33
X-Signature: sha256=9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
X-Timestamp: 2025-11-18T14:22:31.482Z

{
  "event_id": "6f9c3a9e-9a2f-4a1e-9f0e-6c9d1a2b3c4d",
  "event_type": "order.status_changed",
  "timestamp": "2025-11-18T14:22:29.311Z",
  "order_id": "0f0a6d1a-4b7e-4f2a-9a91-2f7cbe4b1a01",
  "order_number": "ORD-000123",
  "customer_id": "b3b0f0c2-2d2a-4bd1-9c2f-1a3f5d6e7a88",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "total_cents": 158900
}
```

Campos do payload conforme definido em [09:43] por Diego. **Os itens do pedido n├úo s├úo enviados**
para n├úo inflar o payload; o cliente que precisar de detalhe consulta `GET /api/v1/orders/:id`
([09:43] Diego, [09:44] Bruno).

Headers, conforme [09:44] Diego e [09:44] Sofia:

| Header | Conte├║do |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento, gerado na inser├º├úo na outbox; chave de deduplica├º├úo do cliente |
| `X-Signature` | `sha256=<hex>` do HMAC-SHA256 do corpo bruto da requisi├º├úo, com a secret do endpoint |
| `X-Timestamp` | Timestamp ISO 8601 do envio, para o cliente detectar replay attack se quiser |
| `X-Webhook-Id` | Id do cadastro de webhook, para clientes com m├║ltiplos endpoints |

O formato exato da assinatura durante o grace period n├úo foi definido na reuni├úo. Antes da
implementa├º├úo, Sofia deve validar se o worker assina com a secret anterior, com a nova, ou publica
duas assinaturas em headers distintos. At├⌐ essa defini├º├úo, o contrato garante apenas que a secret
anterior permanece utiliz├ível por 24 horas ([09:21] Sofia); n├úo prescreve um formato inventado para
`X-Signature`.

### 5.3 Retry e backoff

| Envio | Espera desde a falha anterior | Tempo acumulado aproximado |
| --- | --- | --- |
| Inicial | ΓÇö | 0 |
| Retentativa 1 | 1 minuto | 1 min |
| Retentativa 2 | 5 minutos | 6 min |
| Retentativa 3 | 30 minutos | 36 min |
| Retentativa 4 | 2 horas | 2h36 |
| Retentativa 5 | 12 horas | ~14h36 |
| Fim (DLQ) | Ap├│s falha da retentativa 5 | ~14h36 |

A implementa├º├úo trata a sequ├¬ncia decidida em [09:17] como **cinco retentativas ap├│s o envio
inicial**, pois s├│ assim o intervalo final de 12 horas ocorre antes da ├║ltima tentativa e a janela
total chega a quase 15 horas, como explicado por Diego. O resumo de [09:48] usa "total 5
tentativas"; essa diverg├¬ncia terminol├│gica deve ser confirmada pelos revisores do RFC antes de
codificar, sem remover nenhum dos cinco intervalos registrados na reuni├úo.

`nextAttemptAt` ├⌐ calculado como `now() + backoff[attempts]`, e o worker s├│ considera eleg├¡veis os
eventos com `nextAttemptAt <= now()`. Cada tentativa gera uma linha em `webhook_deliveries` com
status code, corpo da resposta, c├│digo de erro interno e dura├º├úo em milissegundos.

### 5.4 DLQ e replay

Esgotadas as 5 tentativas, o evento ├⌐ copiado para `webhook_dead_letter` com payload, motivo da
falha, n├║mero de tentativas e timestamp ([09:18] Diego), e a linha da outbox ├⌐ marcada como `FAILED`.

O replay ├⌐ **manual**, via `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, que reativa como
pendente a linha original da outbox, preservando seu `id`/`event_id` ([09:18] Diego). O endpoint
exige role `ADMIN` ([09:36] Sofia) e
registra em log quem executou a opera├º├úo, para auditoria ([09:36] Sofia). A linha da DLQ ├⌐ marcada
com `replayedAt` e `replayedById` para evitar replay duplicado.

---

## 6. Contratos p├║blicos

Prefixo comum: `/api/v1`, conforme `src/app.ts` (`app.use('/api/v1', buildApiRouter(controllers))`).
Todos os endpoints exigem `Authorization: Bearer <jwt>`, validado por `authenticate`
(`src/middlewares/auth.middleware.ts`). O `customer_id` **n├úo** ├⌐ extra├¡do do JWT: o token atual
representa o usu├írio operador, n├úo o cliente, ent├úo o cliente ├⌐ informado no body ou no path
([09:32] Bruno, [09:32] Larissa).

O envelope de erro ├⌐ o produzido por `src/middlewares/error.middleware.ts`:
`{ "error": { "code": "...", "message": "...", "details": ... } }`.

### 6.1 `POST /api/v1/webhooks` ΓÇö cadastrar webhook

Cadastra um endpoint de webhook para um cliente. A secret ├⌐ **gerada pela plataforma** e devolvida
na resposta da cria├º├úo ([09:31] Marcos).

**Request**

```json
{
  "customerId": "b3b0f0c2-2d2a-4bd1-9c2f-1a3f5d6e7a88",
  "url": "https://erp.atlascomercial.com.br/webhooks/oms",
  "eventStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`**

```json
{
  "id": "2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33",
  "customerId": "b3b0f0c2-2d2a-4bd1-9c2f-1a3f5d6e7a88",
  "url": "https://erp.atlascomercial.com.br/webhooks/oms",
  "eventStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f2c1d8b7a6e5f4c3b2a1908f7e6d5c4",
  "createdAt": "2025-11-18T14:02:11.004Z"
}
```

| Status | Sem├óntica |
| --- | --- |
| `201` | Webhook criado. **A secret aparece apenas nesta resposta.** |
| `400` | `WEBHOOK_INVALID_URL` (URL n├úo ├⌐ `https`) ou `VALIDATION_ERROR` (schema Zod) |
| `401` | Token ausente ou inv├ílido |
| `404` | `NOT_FOUND` ΓÇö cliente inexistente |
| `422` | `WEBHOOK_INVALID_EVENT_FILTER` ΓÇö status fora do enum `OrderStatus` |

### 6.2 `GET /api/v1/webhooks?customerId=<uuid>` ΓÇö listar webhooks de um cliente

Listagem paginada, no mesmo formato de `src/shared/http/response.ts` (`paginated`)
([09:33] Bruno).

**Request**

```http
GET /api/v1/webhooks?customerId=b3b0f0c2-2d2a-4bd1-9c2f-1a3f5d6e7a88&page=1&pageSize=20 HTTP/1.1
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33",
      "customerId": "b3b0f0c2-2d2a-4bd1-9c2f-1a3f5d6e7a88",
      "url": "https://erp.atlascomercial.com.br/webhooks/oms",
      "eventStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2025-11-18T14:02:11.004Z",
      "updatedAt": "2025-11-18T14:02:11.004Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Sem├óntica |
| --- | --- |
| `200` | Lista retornada. **A secret nunca ├⌐ retornada nesta rota.** |
| `400` | `VALIDATION_ERROR` ΓÇö `customerId` ausente ou n├úo-UUID |
| `401` | Token ausente ou inv├ílido |

### 6.3 `PATCH /api/v1/webhooks/:id` ΓÇö editar webhook

Permite alterar URL, filtro de eventos e estado ativo ([09:33] Bruno).

**Request**

```json
{
  "url": "https://erp.atlascomercial.com.br/webhooks/oms-v2",
  "eventStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `200 OK`**

```json
{
  "id": "2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33",
  "customerId": "b3b0f0c2-2d2a-4bd1-9c2f-1a3f5d6e7a88",
  "url": "https://erp.atlascomercial.com.br/webhooks/oms-v2",
  "eventStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2025-11-18T15:10:44.220Z"
}
```

| Status | Sem├óntica |
| --- | --- |
| `200` | Webhook atualizado |
| `400` | `WEBHOOK_INVALID_URL` ou `VALIDATION_ERROR` |
| `401` | Token ausente ou inv├ílido |
| `404` | `WEBHOOK_NOT_FOUND` |
| `422` | `WEBHOOK_INVALID_EVENT_FILTER` |

### 6.4 `DELETE /api/v1/webhooks/:id` ΓÇö remover webhook

**Request**

```http
DELETE /api/v1/webhooks/2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33 HTTP/1.1
Authorization: Bearer <jwt>
```

**Response `204 No Content`** (sem corpo), no mesmo padr├úo de `OrderController.delete`
(`src/modules/orders/order.controller.ts`).

| Status | Sem├óntica |
| --- | --- |
| `204` | Webhook removido da sele├º├úo de novos eventos. O tratamento de eventos j├í enfileirados depende da decis├úo aberta QI-1. |
| `401` | Token ausente ou inv├ílido |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.5 `POST /api/v1/webhooks/:id/secret/rotate` ΓÇö rotacionar secret

Gera uma nova secret e mant├⌐m a anterior v├ílida por 24 horas ([09:21] Sofia).

**Request** (sem corpo)

```http
POST /api/v1/webhooks/2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33/secret/rotate HTTP/1.1
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "id": "2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33",
  "secret": "whsec_3a7f10c2e94b5d6a8f1c0b2d3e4f5a67",
  "previousSecretExpiresAt": "2025-11-19T15:22:03.981Z"
}
```

| Status | Sem├óntica |
| --- | --- |
| `200` | Nova secret gerada. A anterior continua aceita at├⌐ `previousSecretExpiresAt`. |
| `401` | Token ausente ou inv├ílido |
| `404` | `WEBHOOK_NOT_FOUND` |
| `409` | `WEBHOOK_ROTATION_IN_PROGRESS` ΓÇö j├í existe uma rota├º├úo em grace period |

### 6.6 `GET /api/v1/webhooks/:id/deliveries` ΓÇö hist├│rico de entregas

Retorna as ├║ltimas entregas do webhook com sucesso/falha, payload, resposta e tempo de resposta
([09:34] Marcos). Paginado, com `pageSize` padr├úo 20 e m├íximo 100, no mesmo padr├úo de
`listOrdersQuerySchema` (`src/modules/orders/order.schemas.ts`).

**Request**

```http
GET /api/v1/webhooks/2b1f5c62-5c0f-4a4a-9a92-6f8e2b7c1d33/deliveries?page=1&pageSize=20 HTTP/1.1
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "d1c2b3a4-0011-4a2b-9c8d-7e6f5a4b3c2d",
      "eventId": "a7e2f3b1-8c9d-4e0f-9a1b-2c3d4e5f6071",
      "attempt": 2,
      "success": false,
      "responseStatusCode": 503,
      "responseBody": "{\"error\":\"maintenance\"}",
      "errorCode": "WEBHOOK_DELIVERY_FAILED",
      "durationMs": 812,
      "attemptedAt": "2025-11-18T14:23:31.902Z",
      "payload": {
        "event_id": "6f9c3a9e-9a2f-4a1e-9f0e-6c9d1a2b3c4d",
        "event_type": "order.status_changed",
        "order_number": "ORD-000123",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED"
      }
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Sem├óntica |
| --- | --- |
| `200` | Hist├│rico retornado |
| `400` | `VALIDATION_ERROR` ΓÇö pagina├º├úo inv├ílida |
| `401` | Token ausente ou inv├ílido |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` ΓÇö replay de DLQ

Recoloca o evento morto na outbox como pendente ([09:18] Diego). Exige role `ADMIN`, aplicada com o
`requireRole` j├í existente em `src/middlewares/auth.middleware.ts` ([09:36] Larissa), e registra em
log quem executou a opera├º├úo ([09:36] Sofia).

**Request** (sem corpo)

```http
POST /api/v1/admin/webhooks/dead-letter/9e8d7c6b-5a4f-4e3d-2c1b-0a9f8e7d6c5b/replay HTTP/1.1
Authorization: Bearer <jwt-com-role-ADMIN>
```

**Response `202 Accepted`**

```json
{
  "deadLetterId": "9e8d7c6b-5a4f-4e3d-2c1b-0a9f8e7d6c5b",
  "outboxId": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f",
  "status": "PENDING",
  "replayedAt": "2025-11-19T09:12:44.512Z",
  "replayedBy": "3f2e1d0c-9b8a-4756-8493-2a1b0c9d8e7f"
}
```

| Status | Sem├óntica |
| --- | --- |
| `202` | Evento recolocado na outbox; a entrega ocorrer├í no pr├│ximo ciclo do worker |
| `401` | Token ausente ou inv├ílido |
| `403` | `FORBIDDEN` ΓÇö usu├írio n├úo tem role `ADMIN` |
| `404` | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| `409` | `WEBHOOK_ALREADY_REPLAYED` |

---

## 7. Matriz de erros

Todos os erros do m├│dulo usam o prefixo `WEBHOOK_` ([09:29] Larissa) e s├úo implementados como
classes que estendem as existentes em `src/shared/errors/http-errors.ts`, no mesmo estilo de
`InsufficientStockError` e `InvalidStatusTransitionError` ([09:28] Bruno).

| C├│digo | HTTP | Classe sugerida | Quando ocorre | Onde |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError extends NotFoundError` | Webhook inexistente em GET/PATCH/DELETE/rotate/deliveries | API |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError extends ValidationError` | URL n├úo ├⌐ `https` ou ├⌐ malformada ([09:23] Sofia) | API (schema Zod) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `WebhookSecretRequiredError extends ValidationError` | Opera├º├úo que exige secret sem secret dispon├¡vel ([09:28] Bruno) | API |
| `WEBHOOK_INVALID_EVENT_FILTER` | 422 | `extends UnprocessableEntityError` | `eventStatuses` cont├⌐m valor fora de `OrderStatus` ([09:33] Marcos) | API |
| `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | `extends ConflictError` | Rota├º├úo solicitada durante grace period ativo ([09:21] Sofia) | API |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `extends NotFoundError` | Replay de id inexistente na DLQ ([09:35] Diego) | API admin |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `extends ConflictError` | Replay de evento j├í reprocessado ([09:18] Diego) | API admin |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `extends UnprocessableEntityError` | Payload renderizado excede 64KB ([09:23] Sofia, [09:24] Diego) | Publica├º├úo na outbox |
| `WEBHOOK_DELIVERY_TIMEOUT` | ΓÇö (interno) | `extends AppError` | Cliente n├úo respondeu em 10s ([09:42] Diego) | Worker |
| `WEBHOOK_DELIVERY_FAILED` | ΓÇö (interno) | `extends AppError` | Resposta fora da faixa 2xx ou erro de rede | Worker |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | ΓÇö (interno) | `extends AppError` | 5┬¬ retentativa falhou; evento movido para DLQ ([09:17] Larissa) | Worker |

Os c├│digos marcados como internos n├úo chegam ao cliente HTTP: s├úo gravados em
`webhook_deliveries.errorCode`, em `webhook_dead_letter.failureReason` e no log estruturado. O
middleware `src/middlewares/error.middleware.ts` j├í serializa qualquer `AppError` no envelope padr├úo
sem nenhuma altera├º├úo ([09:29] Bruno).

---

## 8. Estrat├⌐gias de resili├¬ncia

| Aspecto | Estrat├⌐gia |
| --- | --- |
| **Timeout** | 10 segundos por tentativa de entrega ([09:42] Diego). Estouro ├⌐ falha e entra no backoff. |
| **Retry** | Envio inicial seguido de 5 retentativas em 1m / 5m / 30m / 2h / 12h; interpreta├º├úo pendente de confirma├º├úo devido ao resumo de [09:48] ([09:17] Diego). |
| **Isolamento de falha** | Falha de um endpoint n├úo bloqueia os demais: cada linha da outbox ├⌐ independente e um evento com falha sai do caminho quente (`nextAttemptAt` no futuro). |
| **DLQ** | Ap├│s esgotar as tentativas, o evento vai para `webhook_dead_letter` com payload e motivo ([09:18] Diego), preservando a possibilidade de replay manual. |
| **Atomicidade** | Publica├º├úo dentro da transa├º├úo de `changeStatus`; falha de inser├º├úo causa rollback ([09:40] Bruno). |
| **Idempot├¬ncia** | `X-Event-Id` est├ível entre tentativas e replays, para deduplica├º├úo no cliente ([09:25] Diego). |
| **Payload est├ível** | Snapshot na inser├º├úo: retentativas enviam o mesmo corpo e o mesmo `X-Event-Id`. A assinatura ├⌐ calculada no envio conforme a secret v├ílida naquele momento ([09:52] Larissa). |
| **Limite de tamanho** | 64KB por payload; acima disso o evento erra em vez de ser truncado ([09:23] Sofia, [09:24] Diego). |
| **Crash do worker** | A garantia at-least-once exige recuperar eventos deixados em `PROCESSING`, mas lease, timeout de recupera├º├úo e campos necess├írios n├úo foram definidos na reuni├úo. Essa lacuna ├⌐ a decis├úo aberta QI-2. |
| **Shutdown gracioso** | O worker trata `SIGINT`/`SIGTERM` finalizando o ciclo corrente e chamando `prisma.$disconnect()`, no mesmo padr├úo de `src/server.ts`. |
| **Sem fallback alternativo de canal** | Notifica├º├úo por e-mail em caso de falha est├í fora desta fase ([09:37] Larissa); o fallback operacional ├⌐ a DLQ com replay manual. |

---

## 9. Observabilidade

Toda a instrumenta├º├úo usa o Pino j├í configurado em `src/shared/logger/index.ts`. Nenhuma biblioteca
nova ├⌐ introduzida ([09:29] Bruno).

### 9.1 Logs

Eventos de log estruturados, seguindo o estilo `snake_case` j├í usado no projeto
(`http_request`, `server_started`, `shutdown_initiated`):

| Evento | N├¡vel | Campos |
| --- | --- | --- |
| `webhook_event_published` | `info` | `eventId`, `webhookId`, `orderId`, `fromStatus`, `toStatus` |
| `webhook_delivery_attempt` | `info` | `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs`, `success` |
| `webhook_delivery_failed` | `warn` | `eventId`, `webhookId`, `attempt`, `errorCode`, `nextAttemptAt` |
| `webhook_dead_lettered` | `error` | `eventId`, `webhookId`, `attempts`, `failureReason` |
| `webhook_dead_letter_replayed` | `warn` | `deadLetterId`, `outboxId`, `replayedById` ΓÇö **log de auditoria obrigat├│rio** ([09:36] Sofia) |
| `webhook_secret_rotated` | `warn` | `webhookId`, `previousSecretExpiresAt` |
| `worker_started` / `worker_shutdown` | `info` | `pollIntervalMs`, `signal` |

A lista de `redactPaths` de `src/shared/logger/index.ts` deve ser estendida com os caminhos das
secrets (`*.secret`, `*.previousSecret`) para que nunca apare├ºam em log ΓÇö a preocupa├º├úo vem do caso
real de secret vazada em log de aplica├º├úo de cliente ([09:22] Diego).

### 9.2 M├⌐tricas

| M├⌐trica | Tipo | Uso |
| --- | --- | --- |
| `webhook_outbox_pending_total` | gauge | Backlog da outbox; crescimento sustentado indica worker parado ou lento |
| `webhook_outbox_oldest_pending_age_seconds` | gauge | Mede diretamente o SLA de "abaixo de 10 segundos" ([09:02] Marcos) |
| `webhook_delivery_total{result}` | counter | Volume de entregas por resultado (sucesso/falha/timeout) |
| `webhook_delivery_duration_ms` | histograma | Lat├¬ncia de resposta dos clientes; base para avaliar o timeout de 10s |
| `webhook_dead_letter_total` | counter | Eventos irrecuper├íveis; base para a decis├úo futura de notifica├º├úo por e-mail ([09:37] Marcos) |
| `webhook_retry_total{attempt}` | counter | Distribui├º├úo de tentativas; valida a pol├¡tica de 5 tentativas ([09:16] Diego) |

### 9.3 Tracing e correla├º├úo

N├úo h├í APM instalado no projeto hoje, ent├úo a correla├º├úo ├⌐ feita por identificadores propagados nos
logs, aproveitando o `requestId` j├í gerado por `src/middlewares/request-logger.middleware.ts`
(header `X-Request-Id`):

- O `requestId` da requisi├º├úo `PATCH /orders/:id/status` ├⌐ persistido junto ao evento da outbox e
  reemitido em todos os logs de entrega, ligando a mudan├ºa de status ├á entrega final.
- O `eventId` (`X-Event-Id`) funciona como trace id de ponta a ponta: aparece na outbox, em cada
  linha de `webhook_deliveries`, na DLQ, nos logs e na requisi├º├úo recebida pelo cliente.

Caso um APM seja adotado no futuro, `requestId` e `eventId` s├úo os candidatos naturais ├á correla├º├úo
de traces; a reuni├úo n├úo decidiu ferramenta nem formato de tracing.

---

## 10. Integra├º├úo com o sistema existente

Esta se├º├úo nomeia os arquivos reais do reposit├│rio afetados pela feature.

### 10.1 `src/modules/orders/order.service.ts`

Ponto de integra├º├úo cr├¡tico. O m├⌐todo `changeStatus` executa hoje um `prisma.$transaction` que
valida a transi├º├úo via `canTransition`, ajusta estoque, atualiza `orders.status` e insere em
`order_status_history`. A feature acrescenta **uma chamada dentro da mesma transa├º├úo**, logo ap├│s a
cria├º├úo do hist├│rico e antes da leitura final do pedido:

```ts
await tx.orderStatusHistory.create({ /* ...c├│digo atual... */ });

await publishWebhookEvent(tx, order, from, to);
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o `Prisma.TransactionClient` (o alias
`TxClient` j├í declarado no topo do arquivo), consulta os endpoints ativos do cliente do pedido,
filtra por status assinado e insere as linhas na outbox. Se essa chamada lan├ºar, o `$transaction`
faz rollback e a mudan├ºa de status n├úo acontece ([09:40] Bruno, [09:41] Diego). A fun├º├úo ├⌐ exportada
pelo m├│dulo de webhooks, evitando injetar o reposit├│rio inteiro no `OrderService` ([09:41] Diego).

### 10.2 `src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts`

As classes de erro do m├│dulo estendem as existentes, exatamente como `InsufficientStockError`
estende `UnprocessableEntityError` e `InvalidStatusTransitionError` estende `ConflictError`. A base
`AppError` j├í carrega `statusCode`, `errorCode` e `details`, que ├⌐ tudo o que a matriz de erros da
se├º├úo 7 precisa. Os novos erros s├úo exportados pelo barrel `src/shared/errors/index.ts`, mantendo o
import ├║nico usado nos demais services ([09:28] Bruno).

### 10.3 `src/middlewares/auth.middleware.ts`

`authenticate` protege todas as rotas do m├│dulo, no mesmo formato de
`src/modules/orders/order.routes.ts` (`router.use(authenticate)`). O endpoint de replay de DLQ usa
`requireRole('ADMIN')`, o mesmo helper j├í existente, e o `req.user.id` disponibilizado pelo
middleware ├⌐ o valor gravado em `webhook_dead_letter.replayedById` e no log de auditoria
([09:36] Larissa e Sofia).

### 10.4 `src/middlewares/error.middleware.ts`

Nenhuma altera├º├úo necess├íria. O middleware j├í converte qualquer `AppError` no envelope
`{ error: { code, message, details } }`, trata `ZodError` como `VALIDATION_ERROR` e mapeia erros
conhecidos do Prisma. Os erros `WEBHOOK_*` s├úo absorvidos automaticamente ([09:29] Bruno).

### 10.5 `src/middlewares/validate.middleware.ts` e schemas Zod

`webhook.schemas.ts` segue o formato de `src/modules/orders/order.schemas.ts`: schemas de params,
body e query aplicados via `validate({ params, body, query })`. A valida├º├úo de TLS obrigat├│rio ├⌐
implementada aqui como um refinamento de schema (`z.string().url().startsWith('https://')`), e n├úo
como componente arquitetural ([09:23] Sofia). O filtro de eventos usa `z.nativeEnum(OrderStatus)`,
reaproveitando o enum do Prisma.

### 10.6 `src/app.ts` e `src/routes/index.ts`

`buildControllers` instancia `WebhookRepository` ΓåÆ `WebhookService` ΓåÆ `WebhookController` no mesmo
padr├úo dos demais m├│dulos, o tipo `Controllers` ganha a chave `webhooks`, e `buildApiRouter` registra
`router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e a rota administrativa sob
`/admin/webhooks`.

### 10.7 `src/server.ts` ΓåÆ nova entrada de worker

O worker ├⌐ uma entrada de processo separada, espelhando a estrutura de `bootstrap()` em
`src/server.ts`: carrega `env`, cria o `PrismaClient`, inicia o loop de polling e trata
`SIGINT`/`SIGTERM` com `prisma.$disconnect()`. A diferen├ºa ├⌐ que n├úo sobe servidor HTTP
([09:11] Larissa, [09:11] Diego). O `PrismaClient` ├⌐ uma inst├óncia pr├│pria, pois o client ├⌐ por
processo ([09:30] Bruno).

### 10.8 `prisma/schema.prisma` e `src/config/database.ts`

Os quatro modelos da se├º├úo 4 s├úo adicionados ao schema, seguindo as conven├º├╡es existentes
(`@db.Char(36)`, `@@map` em snake_case, ├¡ndices expl├¡citos). A defini├º├úo de rela├º├╡es Prisma e de
chaves estrangeiras deve acompanhar a decis├úo de reten├º├úo dos endpoints (QI-1); o trecho acima
mostra apenas os campos m├¡nimos derivados da reuni├úo. O worker usa `createPrismaClient()` de
`src/config/database.ts` para obter sua pr├│pria inst├óncia ([09:30] Bruno).

### 10.9 `src/config/env.ts` e `package.json`

`env.ts` deve validar as configura├º├╡es operacionais do worker. `package.json` ganha o script
`npm run worker`, conforme decidido em [09:11] Larissa. O comando exato de desenvolvimento e
produ├º├úo deve seguir os scripts existentes e ser fechado na implementa├º├úo, sem fixar neste FDD uma
forma que n├úo foi decidida na reuni├úo.

### 10.10 `src/shared/http/response.ts` e `src/shared/logger/index.ts`

`paginated()` ├⌐ reutilizado nas listagens de webhooks e de entregas. O logger Pino ├⌐ o mesmo, com
`redactPaths` estendido para os campos de secret (se├º├úo 9.1).

---

## 11. Depend├¬ncias e compatibilidade

| Item | Situa├º├úo |
| --- | --- |
| MySQL 8.0 (`docker-compose.yml`) | J├í existe; ganha 4 tabelas novas |
| Prisma 5.22 (`@prisma/client`) | J├í existe; nova migration |
| Express 4.21, Zod 3.23, Pino 9.5, `uuid` 11 | J├í existem; usados sem altera├º├úo de vers├úo |
| Cliente HTTP no worker | Deve suportar timeout de 10 segundos ([09:42] Diego). A biblioteca ou API concreta n├úo foi definida na reuni├úo. |
| HMAC-SHA256 | `node:crypto` nativo; nenhuma depend├¬ncia nova ([09:20] Sofia) |
| Compatibilidade de API | Aditiva. Nenhum endpoint existente muda de contrato. |
| Compatibilidade de dados | Novas tabelas s├úo necess├írias; rela├º├╡es e chaves estrangeiras dependem da decis├úo QI-1. |
| Deploy | Passa a exigir **dois processos**: `npm start` (API) e `npm run worker` ([09:11] Diego) |
| Ordem de deploy | Deve garantir que o schema da outbox exista antes de API e worker utilizarem-no; a estrat├⌐gia operacional n├úo foi discutida na reuni├úo. |

---

## 12. Crit├⌐rios de aceite t├⌐cnicos

| # | Crit├⌐rio |
| --- | --- |
| CAT-1 | Rollback da transa├º├úo de `changeStatus` n├úo deixa linha ├│rf├ú na `webhook_outbox`, e falha na inser├º├úo da outbox impede a mudan├ºa de status ([09:40] Bruno) |
| CAT-2 | Mudan├ºa de status sem nenhum endpoint ativo assinando aquele status n├úo insere linha na outbox ([09:34] Bruno) |
| CAT-3 | O worker inicia a primeira tentativa em at├⌐ 2 segundos ap├│s o commit, no caminho feliz; a entrega completa permanece sujeita ao objetivo de produto de menos de 10 segundos ([09:02] Marcos, [09:09] Diego) |
| CAT-4 | Entrega inclui os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`, e `Content-Type: application/json` ([09:44] Diego e Sofia) |
| CAT-5 | `X-Signature` valida como HMAC-SHA256 do corpo bruto com a secret do endpoint ([09:20] Sofia) |
| CAT-6 | Ap├│s rota├º├úo, assinaturas geradas com a secret antiga continuam aceitas por 24 horas e deixam de ser aceitas depois ([09:21] Sofia) |
| CAT-7 | Cadastro com URL `http` ├⌐ rejeitado com `WEBHOOK_INVALID_URL` e status 400 ([09:23] Sofia) |
| CAT-8 | Cliente respondendo com atraso superior a 10 segundos ├⌐ tratado como falha ([09:42] Diego) |
| CAT-9 | Ap├│s a falha do envio inicial e das cinco retentativas, o evento aparece em `webhook_dead_letter` com payload e motivo, e sai da varredura de pendentes ([09:17] e [09:18] Diego) |
| CAT-10 | As cinco retentativas respeitam os intervalos 1m / 5m / 30m / 2h / 12h ([09:17] Diego) |
| CAT-11 | Replay de DLQ com usu├írio `OPERATOR` retorna 403; com `ADMIN` retorna 202 e gera log de auditoria com o id do usu├írio ([09:36] Sofia e Larissa) |
| CAT-12 | Payload acima de 64KB n├úo ├⌐ enviado e gera `WEBHOOK_PAYLOAD_TOO_LARGE` ([09:24] Larissa) |
| CAT-13 | O `X-Event-Id` ├⌐ id├¬ntico em todas as tentativas e no replay do mesmo evento ([09:25] Diego) |
| CAT-14 | O payload entregue reflete o estado do pedido no instante da transi├º├úo, mesmo que o pedido tenha mudado depois ([09:52] Larissa) |
| CAT-15 | Nenhuma secret aparece em log, nem em `GET /webhooks` ([09:22] Diego) |
| CAT-16 | Rein├¡cio da API n├úo interrompe o processamento de eventos ([09:11] Diego) |
| CAT-17 | Su├¡te existente em `tests/orders.test.ts` continua verde ap├│s a altera├º├úo no `changeStatus` |

### 12.1 Estrat├⌐gia de testes

Vitest + Supertest, no mesmo formato de `tests/orders.test.ts`, com o reset de banco de
`tests/setup.ts` estendido para as novas tabelas e as f├íbricas de `tests/helpers/factories.ts`
ampliadas com um `createWebhookEndpoint`.

- **Unit├írios:** c├ílculo de backoff, gera├º├úo e verifica├º├úo de HMAC, valida├º├úo de schema (URL `https`,
  filtro de status), limite de 64KB.
- **Integra├º├úo:** publica├º├úo na outbox dentro da transa├º├úo (inclusive o caso de rollback), CRUD dos
  endpoints, hist├│rico de entregas, replay de DLQ com e sem role `ADMIN`.
- **Worker:** entrega bem-sucedida, timeout de 10s, escalonamento de tentativas at├⌐ a DLQ, e
  estabilidade do `X-Event-Id` entre tentativas ΓÇö com um servidor HTTP local simulando o cliente.

---

## 13. Riscos e mitiga├º├úo

| Risco | Probabilidade | Impacto | Mitiga├º├úo |
| --- | --- | --- | --- |
| A escrita extra na transa├º├úo de `changeStatus` degrada o fluxo mais cr├¡tico do OMS | M├⌐dia | Alto | Payload renderizado em mem├│ria antes da escrita; consulta de assinantes indexada por `customerId`+`active`; batch de inserts ├║nico; medir `changeStatus` antes e depois |
| Cliente lento segura o worker single-threaded e atrasa a fila inteira | M├⌐dia | Alto | Timeout de 10s ([09:42] Diego); evento com falha sai do caminho quente via `nextAttemptAt`; m├⌐trica `webhook_outbox_oldest_pending_age_seconds` como alarme |
| Cliente sem deduplica├º├úo processa evento duplicado | Alta | M├⌐dio | `X-Event-Id` em todas as tentativas ([09:25] Diego) e documenta├º├úo destacada no portal ([09:26] Marcos) |
| Crescimento indefinido da outbox degrada o polling | M├⌐dia | M├⌐dio | ├ìndices em `status`/`createdAt` ([09:08] Diego); arquivamento definido como trabalho posterior ([09:08] Diego) |
| Secret vazada no lado do cliente | Baixa | Alto | Secret por endpoint ([09:21] Sofia); rota├º├úo com grace period; redaction no logger; revis├úo de seguran├ºa de 2 dias antes do deploy ([09:46] Sofia) |
| Eventos presos em `PROCESSING` ap├│s crash do worker | Baixa | M├⌐dio | Reelegibilidade por idade em `PROCESSING`; duplicidade coberta pela garantia at-least-once ([09:24] Diego) |
| Rajada de mudan├ºas de status bombardeia o endpoint do cliente | M├⌐dia | M├⌐dio | Sem mitiga├º├úo nesta fase por decis├úo expl├¡cita: observar e implementar rate limiting se virar problema ([09:39] Diego e Larissa) |

---

## 14. Decis├╡es de implementa├º├úo em aberto

| ID | Quest├úo | Por que bloqueia |
| --- | --- | --- |
| QI-1 | Ao remover um webhook, eventos j├í enfileirados s├úo descartados, entregues com a configura├º├úo preservada ou enviados para DLQ? A reuni├úo decidiu o endpoint `DELETE`, mas n├úo o ciclo de vida das pend├¬ncias ([09:33] Bruno). | Define soft delete versus hard delete, rela├º├╡es do Prisma e disponibilidade de URL/secret para eventos antigos. |
| QI-2 | Como recuperar eventos deixados em `PROCESSING` ap├│s crash: lease temporal, retorno imediato a `PENDING` ou outra estrat├⌐gia? | Sem essa decis├úo, o worker pode perder eventos ou duplic├í-los; at-least-once exige recupera├º├úo ([09:24] Diego). |
| QI-3 | Durante as 24 horas de grace period, qual secret assina novas entregas e como o cliente recebe uma ou mais assinaturas? | A reuni├úo decidiu duas secrets v├ílidas em paralelo, mas n├úo o formato do header nem a regra para eventos enfileirados ([09:21] Sofia). |
| QI-4 | "5 tentativas" significa cinco envios totais ou cinco retentativas depois do envio inicial? | A sequ├¬ncia decidida cont├⌐m cinco intervalos e chega a quase 15 horas, enquanto o resumo usa "total 5 tentativas" ([09:17] Diego, [09:48] Larissa). |
| QI-5 | Uma mudan├ºa de status com v├írios endpoints assinantes gera um ├║nico `event_id` compartilhado ou um evento de entrega por endpoint? | A reuni├úo definiu UUID ├║nico por evento e filtro por endpoint, mas n├úo a granularidade do fan-out ([09:25] Diego, [09:34] Bruno). Isso define se `WebhookOutbox.id` pode ser o `event_id` sem uma tabela l├│gica adicional. |
