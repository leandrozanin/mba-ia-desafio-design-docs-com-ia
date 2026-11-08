# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|-------|-------|
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Em revisão; fluxos e contratos propostos para implementação, com decisões pendentes no RFC §5 |
| **Base** | Decisões dos [ADRs 001–007](adrs/) · proposta do [RFC](RFC.md) · requisitos do [PRD](PRD.md) |
| **Código de referência** | OMS em `src/` (Express 4 + Prisma 5.22 + MySQL 8, ESM, Node ≥ 20) |

> Convenção deste documento: tudo que vem da reunião ou do código está referenciado no [Tracker](TRACKER.md). Onde o FDD precisou tomar uma decisão de implementação que **não** foi discutida, o texto marca como **(proposta de implementação)** e o ponto aparece nas questões em aberto do RFC quando afeta contrato ou comportamento.
>
> **Arquivos novos x existentes:** tudo sob `src/modules/webhooks/` e o entry-point `src/worker.ts` são **arquivos a criar** por esta feature. Todos os demais caminhos citados (por exemplo `src/modules/orders/order.service.ts`, `src/shared/errors/http-errors.ts`) já existem no repositório e são os pontos de integração da seção 10.

---

## 1. Contexto e motivação técnica

- Clientes B2B fazem polling em `GET /orders` para descobrir mudanças de status; querem notificação em menos de 10 s ([09:00] Marcos, [09:02] Marcos).
- Toda mudança de status passa por `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que roda num `this.prisma.$transaction(async (tx) => { ... })`: valida a transição (`canTransition` em `src/modules/orders/order.status.ts`), debita/repõe estoque, faz `tx.order.update` e `tx.orderStatusHistory.create` ([09:04] Bruno).
- O sistema não tem eventos, filas nem chamadas HTTP de saída. Chamar o cliente dentro da transação é inaceitável ([09:04] Bruno, [09:06] Diego), e o evento não pode se perder se a transação commitar ([09:40] Bruno).
- Por isso: outbox transacional no MySQL + worker separado em polling ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

| ID | Objetivo |
|----|----------|
| OBJ-01 | Gravar o evento na `webhook_outbox` na **mesma transação** de `changeStatus`; falha na outbox ⇒ rollback da mudança de status. |
| OBJ-02 | Primeira tentativa de entrega em **< 10 s** após o commit em operação normal (polling de 2 s + envio). |
| OBJ-03 | Entrega **at-least-once** com `event_id` estável entre tentativas e replays. |
| OBJ-04 | Retry exponencial 1m/5m/30m/2h/12h e DLQ persistida com replay manual por `ADMIN`. |
| OBJ-05 | Toda requisição assinada com HMAC-SHA256 com a secret do endpoint; suporte a rotação com 24 h de convivência. |
| OBJ-06 | Zero infraestrutura nova e zero mudanças no `error.middleware.ts`; reuso de `AppError`, Pino, `validate()`, `authenticate`/`requireRole`. |

## 3. Escopo e exclusões

**Dentro do escopo**
- Módulo `src/modules/webhooks/` (controller, service, repository, routes, schemas, publisher, signer, worker).
- Entry-point `src/worker.ts` e script `npm run worker`.
- Quatro tabelas Prisma: `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`.
- Endpoints de configuração, rotação de secret, histórico de entregas e replay de DLQ.
- Alteração de `changeStatus` para publicar o evento.

**Fora do escopo (decidido na reunião)**

| Item | Fonte |
|------|-------|
| Webhooks inbound (clientes enviando para nós) | [09:02] Marcos, [09:03] Sofia |
| E-mail de alerta ao cliente quando o webhook falha repetidamente — próxima fase | [09:37] Larissa |
| Rate limiting de envio por cliente — observar e decidir depois | [09:39] Diego, [09:39] Larissa |
| Dashboard/painel visual — projeto do time de frontend | [09:40] Larissa |
| Arquivamento de eventos entregues (~30 dias) | [09:08] Diego |
| Múltiplos workers / ordenação global | [09:12] Diego, [09:13] Larissa |
| Itens do pedido no payload | [09:43] Diego |

## 4. Modelo de dados (Prisma / MySQL)

Novos modelos em `prisma/schema.prisma`, seguindo as convenções existentes: ids `String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa), campos em camelCase, `@@map` para nomes de tabela em snake_case, `createdAt`/`updatedAt`.

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  DELIVERED    // entregue
  FAILED       // falhou (falha permanente; cópia na DLQ)
}

enum WebhookDeliveryResult {
  SUCCESS
  FAILURE
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // lista de OrderStatus, ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean   @default(true)
  deletedAt               DateTime? // proposta de exclusão lógica (RFC OQ-11)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer             @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)
  payload       String              @db.MediumText   // corpo exato que será assinado e enviado
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)       // falhas já registradas
  nextAttemptAt DateTime            @default(now())
  lockedUntil   DateTime?
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    WebhookEndpoint   @relation(fields: [webhookId], references: [id]) // sem cascade; exclusão lógica pendente (RFC OQ-11)
  deliveries WebhookDelivery[]

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([webhookId, orderId, createdAt])
  @@index([eventId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String                @id @default(uuid()) @db.Char(36)
  webhookId      String                @db.Char(36)
  outboxId       String                @db.Char(36)
  eventId        String                @db.Char(36)
  attempt        Int
  result         WebhookDeliveryResult
  requestBody    String                @db.MediumText
  responseStatus Int?
  responseBody   String?               @db.Text
  durationMs     Int
  errorCode      String?               @db.VarChar(64)
  createdAt      DateTime              @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id]) // sem cascade; exclusão lógica pendente (RFC OQ-11)
  outbox  WebhookOutbox   @relation(fields: [outboxId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id                 String    @id @default(uuid()) @db.Char(36)
  outboxId           String    @db.Char(36)
  eventId            String    @db.Char(36)
  webhookId          String    @db.Char(36)
  orderId            String    @db.Char(36)
  payload            String    @db.MediumText
  reason             String    @db.VarChar(64)   // código WEBHOOK_*
  lastResponseStatus Int?
  lastError          String?   @db.VarChar(500)
  attempts           Int
  failedAt           DateTime  @default(now())
  replayedAt         DateTime?
  replayedById       String?   @db.Char(36)

  @@index([failedAt])
  @@index([outboxId])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:

- **Uma linha de outbox por (evento × endpoint interessado).** O `eventId` é gerado uma vez por mudança de status e repetido nas linhas de cada endpoint do mesmo customer; `X-Webhook-Id` diferencia os cadastros ([09:25] Diego, [09:44] Sofia). Retry, histórico e DLQ ficam naturalmente por endpoint. **(proposta de implementação)**
- **`payload` como texto, não `Json`.** A assinatura HMAC é sobre o corpo exato ([09:22] Sofia); a coluna `JSON` do MySQL normaliza ordem de chaves e espaços, o que poderia alterar os bytes assinados. O snapshot ([ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)) é guardado já serializado. **(proposta de implementação)**
- **`events` como `Json`.** O conector MySQL do Prisma não suporta listas escalares; o conteúdo é validado por Zod com `z.nativeEnum(OrderStatus)`, mesmo padrão de `src/modules/orders/order.schemas.ts`.
- `Customer` (em `prisma/schema.prisma`) ganha a relação inversa `webhooks WebhookEndpoint[]`.
- Índices em status e data de criação, conforme a reunião ([09:08] Diego); o índice `(webhookId, orderId, createdAt)` sustenta a regra de ordenação do fluxo F2.

## 5. Fluxos detalhados

### F1 — Criação do evento na outbox (dentro de `changeStatus`)

```
OrderService.changeStatus(id, input, userId)
└─ prisma.$transaction(async (tx) => {
     order = tx.order.findUnique(...)          // existente
     valida transição / estoque                 // existente
     tx.order.update(...)                       // existente
     tx.orderStatusHistory.create(...)          // existente
     await publishWebhookEvent(tx, order, from, to)   // NOVO
     refreshed = tx.order.findUnique(...)       // existente
   })
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` em `src/modules/webhooks/webhook.publisher.ts` ([09:41] Bruno):

1. `endpoints = tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`.
2. Filtra os que têm `toStatus` em `events` ([09:33] Marcos, [09:34] Bruno). Se nenhum, **retorna sem inserir nada** ([09:34] Bruno).
3. Gera `eventId = randomUUID()` (`node:crypto`) uma única vez ([09:25] Diego).
4. Renderiza o snapshot ([09:43] Diego, [09:52] Larissa):

   ```json
   {
     "event_id": "5b0b6c1e-8f3a-4a52-9a43-0f6d2a7c9e11",
     "event_type": "order.status_changed",
     "timestamp": "2026-11-12T14:03:27.512Z",
     "order_id": "c2f7f0a4-3b8e-4a8e-9d2b-6a1f0e5c4b33",
     "order_number": "ORD-000123",
     "from_status": "PROCESSING",
     "to_status": "SHIPPED",
     "customer_id": "9e1d4b7a-2c6f-4f0e-8a3b-5d7c1e2f9a08",
     "total_cents": 18000
   }
   ```

   `timestamp` é o instante da mudança de status; `order_number`, `customer_id` e `total_cents` vêm do registro `order` já carregado no início da transação (`orderNumber`, `customerId`, `totalCents` do model `Order`).
5. Serializa uma vez (JSON.stringify) e mede Buffer.byteLength. Se passar de 64 KB ([09:24] Diego, [09:24] Larissa), **não envia**. Para cada endpoint interessado, cria uma linha da outbox em FAILED e uma linha da DLQ com o mesmo outboxId, eventId e payload, na transação de changeStatus; em seguida, encerra o publisher sem executar o passo 6. Isso mantém o evento rastreável; o replay permanece bloqueado enquanto o payload exceder 64 KB. **Proposta de implementação:** a reunião decidiu rejeitar o envio acima do limite, mas não definiu a persistência ([09:23] Sofia).
6. Apenas se o payload tiver até 64 KB, tx.webhookOutbox.createMany cria uma linha PENDING por endpoint, com attempts = 0 e nextAttemptAt = now().
7. Qualquer erro aqui propaga e o Prisma faz rollback de toda a transação — inclusive de `orders` e `order_status_history` ([09:40] Bruno, [09:41] Diego).

### F2 — Processamento pelo worker

`src/worker.ts` (novo entry-point, no molde de `src/server.ts`) cria um `PrismaClient` via `createPrismaClient()` (`src/config/database.ts`) e inicia `WebhookWorker` (`src/modules/webhooks/webhook.worker.ts`) ([09:11] Larissa, [09:28] Bruno, [09:30] Bruno).

Ciclo, repetido a cada `WEBHOOK_POLL_INTERVAL_MS = 2000` ([09:09] Diego):

1. **Reivindicar um lote** (transação curta, `$queryRaw`):

   ```sql
   SELECT o.id
     FROM webhook_outbox o
    WHERE (   (o.status = 'PENDING'    AND o.nextAttemptAt <= NOW(3))
           OR (o.status = 'PROCESSING' AND o.lockedUntil   <  NOW(3)))  -- recuperação após queda
      AND NOT EXISTS (                                                  -- ordenação por pedido/endpoint
            SELECT 1 FROM webhook_outbox p
             WHERE p.webhookId = o.webhookId
               AND p.orderId   = o.orderId
               AND p.createdAt < o.createdAt
               AND p.status IN ('PENDING','PROCESSING'))
    ORDER BY o.createdAt
    LIMIT :batch
    FOR UPDATE SKIP LOCKED;
   ```

   seguido de `UPDATE ... SET status='PROCESSING', lockedUntil = NOW(3) + INTERVAL 60 SECOND`. Lote pequeno: `WEBHOOK_BATCH_SIZE = 20` (valor inicial a calibrar; a reunião só definiu "batch pequeno", [09:08] Diego).
   - A ordem por `createdAt` dá a ordenação por `order_id` do single-worker ([09:12] Diego). O `NOT EXISTS` impede que um evento posterior do mesmo pedido ultrapasse um anterior que está em backoff — **(proposta de implementação; ver RFC OQ-03)**.
   - `FOR UPDATE SKIP LOCKED` (MySQL 8) e o lease `lockedUntil` não tornam o sistema multi-worker, mas protegem contra execução duplicada acidental e permitem retomar linhas presas em `PROCESSING` se o worker cair.
   - A barreira de ordenação cobre linhas anteriores PENDING/PROCESSING. Quando a anterior chega à DLQ (FAILED), a posterior pode avançar; se a anterior for reproduzida depois, poderá chegar fora da ordem cronológica. Isso é uma limitação proposta e precisa de decisão (RFC OQ-12); AC-T13 não promete ordenação através de DLQ/replay.
2. **Para cada linha, em sequência:**
   1. Carrega o `WebhookEndpoint`. Se não estiver ativo ⇒ marca `FAILED` e grava DLQ com `reason = WEBHOOK_INACTIVE` (sem retry).
   2. Se o endpoint não tiver `secret` ⇒ DLQ com `reason = WEBHOOK_SECRET_REQUIRED` (sem retry; indica dado corrompido).
   3. Monta a requisição (contrato C8) e assina o `payload` armazenado ([09:20] Sofia).
   4. `fetch(url, { method: 'POST', body: payload, headers, redirect: 'manual', signal: AbortSignal.timeout(10_000) })` — timeout de 10 s ([09:42] Diego). `redirect: 'manual'` faz um 3xx contar como falha **(proposta de implementação)**.
   5. Grava `webhook_deliveries` (tentativa, resultado, corpo enviado, status e corpo de resposta truncado em 4 KB, duração) — base do histórico pedido pelo produto ([09:34] Marcos).
   6. **2xx** ⇒ `status = DELIVERED`, `deliveredAt = now()` ([09:08] Diego). **Falha** ⇒ fluxo F3.
3. Um erro inesperado ao processar uma linha é logado e a linha volta a `PENDING` com o mesmo cálculo de backoff — no worker o `catch` não relança, para não derrubar o processo (contexto de background).
4. **Encerramento:** em `SIGINT`/`SIGTERM`, o worker termina a linha corrente, não inicia novo ciclo e chama `prisma.$disconnect()`, como `src/server.ts` faz com o servidor HTTP.

### F3 — Retry com backoff exponencial

São falhas: timeout (`WEBHOOK_DELIVERY_TIMEOUT`), erro de rede/TLS/DNS (`WEBHOOK_DELIVERY_NETWORK_ERROR`) e status fora de 2xx, incluindo 3xx (`WEBHOOK_DELIVERY_HTTP_ERROR`).

```ts
// src/modules/webhooks/webhook.constants.ts
export const WEBHOOK_RETRY_DELAYS_MS = [
  60_000,          // 1 min
  5 * 60_000,      // 5 min
  30 * 60_000,     // 30 min
  2 * 3_600_000,   // 2 h
  12 * 3_600_000,  // 12 h
] as const;        // [09:17] Diego / [09:17] Larissa
```

Ao falhar:

```
attempts = attempts + 1
se attempts <= WEBHOOK_RETRY_DELAYS_MS.length (5):
    status = PENDING
    nextAttemptAt = now + WEBHOOK_RETRY_DELAYS_MS[attempts - 1]
    lastError = <código + resumo>
senão:
    → F4 (DLQ)
```

Linha do tempo de um evento que nunca é aceito: t0 (envio 1) → +1m (2) → +5m (3) → +30m (4) → +2h (5) → +12h (6) ⇒ DLQ. São 6 chamadas e ~14h36min entre a primeira falha e a última tentativa, coerente com "quase 15 horas" ([09:17] Diego). A contagem exata ("5 tentativas", [09:48] Larissa) está pendente de confirmação (RFC OQ-05); muda apenas o limite do `if`.

### F4 — Dead Letter Queue

Na mesma transação que atualiza a outbox:

1. `webhook_outbox.status = FAILED`, `lockedUntil = null`.
2. `webhook_dead_letter.create({ outboxId, eventId, webhookId, orderId, payload, reason: 'WEBHOOK_MAX_ATTEMPTS_EXCEEDED', lastResponseStatus, lastError, attempts })` — payload, motivo e timestamp ([09:18] Diego).
3. Log `webhook_moved_to_dead_letter` (nível `warn`).

### F5 — Replay manual da DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18] Diego), protegido por `authenticate` + `requireRole('ADMIN')` ([09:36] Sofia, [09:36] Larissa):

1. Carrega o registro; inexistente ⇒ `WEBHOOK_DEAD_LETTER_NOT_FOUND`; já reprocessado (`replayedAt` preenchido) ⇒ `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
2. Endpoint removido ou inativo ⇒ WEBHOOK_INACTIVE. Se a DLQ registra WEBHOOK_PAYLOAD_TOO_LARGE, o replay é recusado com 422 WEBHOOK_PAYLOAD_TOO_LARGE: o snapshot imutável continua acima de 64 KB e não pode ser enviado.
3. Numa transação: a linha original da outbox volta para `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`, **mantendo o mesmo `eventId` e o mesmo `payload`** ([09:18] Diego "recoloca na outbox como pendente"; [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)). O registro da DLQ recebe `replayedAt = now()` e `replayedById = req.user.id`.
4. Log de auditoria `webhook_dead_letter_replayed` com `deadLetterId`, `eventId`, `webhookId` e `replayedBy` ([09:36] Sofia).
5. Se falhar de novo, o ciclo F3/F4 se repete e gera um **novo** registro na DLQ (o anterior fica como histórico).

### F6 — Rotação de secret

`POST /api/v1/webhooks/:id/secret/rotate` ([09:21] Sofia):

1. `previousSecret = secret`, `previousSecretExpiresAt = now + 24h`, `secret = novaSecret()` ([09:21] Sofia, [09:22] Sofia).
2. Se uma rotação ocorrer enquanto ainda há `previousSecret` válida, a mais antiga é descartada (só duas secrets convivem). **(proposta de implementação)**
3. Durante a convivência o worker envia `X-Signature` com as duas assinaturas (ver C8). Depois de `previousSecretExpiresAt`, só a nova.

Geração de secret: `randomBytes(32).toString('hex')` (`node:crypto`), retornada **apenas** nas respostas de criação e rotação ([09:31] Marcos); nunca em listagens nem em logs.

## 6. Contratos públicos

Todos os endpoints de configuração ficam sob o prefixo existente `/api/v1` (`src/app.ts`), exigem `Authorization: Bearer <JWT>` (`authenticate`) e usam o envelope de erro de `src/middlewares/error.middleware.ts`:

**Autorização por customer (pendência bloqueante, RFC OQ-09).** authenticate valida identidade/role, mas não prova que o usuário pode administrar o customerId informado. O modelo atual User não tem relação com Customer (prisma/schema.prisma). A reunião definiu autenticação normal e customerId no body/path ([09:32] Bruno/Larissa), mas não definiu essa autorização. Proposta: exigir checagem explícita de escopo por customer em CRUD e deliveries; decidir a forma de representar o vínculo antes da implementação. Autenticação genérica não equivale a autorização por customer.

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

Qualquer role autenticada pode usar o CRUD ([09:37] Sofia); apenas o replay exige `ADMIN` ([09:36] Sofia). O `customer_id` não vem do JWT, porque o JWT é do usuário operador ([09:32] Bruno, [09:32] Larissa); abaixo ele vai no body (criação) e na query (listagem) — **proposta pendente (RFC OQ-02)**.

### C1 — `POST /api/v1/webhooks` — cadastrar webhook

Request ([09:31] Marcos):

```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "9e1d4b7a-2c6f-4f0e-8a3b-5d7c1e2f9a08",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` — a secret aparece **só aqui** (e na rotação):

```json
{
  "id": "3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90",
  "customerId": "9e1d4b7a-2c6f-4f0e-8a3b-5d7c1e2f9a08",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "4f9c2e7a1b3d5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8",
  "createdAt": "2026-11-12T13:00:00.000Z",
  "updatedAt": "2026-11-12T13:00:00.000Z"
}
```

| Status | Quando |
|--------|--------|
| 201 | Criado |
| 400 `VALIDATION_ERROR` | URL não-`https` (`details[].message = "WEBHOOK_INVALID_URL"`), `events` vazio ou com status inválido (`"WEBHOOK_INVALID_EVENTS"`), `customerId` não-UUID |
| 401 `UNAUTHORIZED` | Sem token ou token inválido |
| 404 `WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` inexistente |

Schema Zod (`webhook.schemas.ts`):

```ts
export const createWebhookSchema = z.object({
  customerId: z.string().uuid(),
  url: z.string().url().max(2048)
    .refine((u) => u.startsWith('https://'), { message: 'WEBHOOK_INVALID_URL' }), // [09:23] Sofia
// A validação HTTPS não define destinos de rede permitidos; ver RFC OQ-10.
  events: z.array(z.nativeEnum(OrderStatus), { invalid_type_error: 'WEBHOOK_INVALID_EVENTS' })
    .min(1, 'WEBHOOK_INVALID_EVENTS'),
});
```

### C2 — `GET /api/v1/webhooks?customerId=…` — listar webhooks de um customer

Request ([09:33] Bruno):

```http
GET /api/v1/webhooks?customerId=9e1d4b7a-2c6f-4f0e-8a3b-5d7c1e2f9a08
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK` (sem `secret`):

```json
{
  "data": [
    {
      "id": "3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90",
      "customerId": "9e1d4b7a-2c6f-4f0e-8a3b-5d7c1e2f9a08",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotationPending": false,
      "createdAt": "2026-11-12T13:00:00.000Z",
      "updatedAt": "2026-11-12T13:00:00.000Z"
    }
  ]
}
```

| Status | Quando |
|--------|--------|
| 200 | Lista (vazia se não houver) |
| 400 `VALIDATION_ERROR` | `customerId` ausente/não-UUID |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_CUSTOMER_NOT_FOUND` | Customer inexistente |

### C3 — `PATCH /api/v1/webhooks/:id` — editar webhook

Request ([09:33] Bruno) — todos os campos opcionais, pelo menos um obrigatório:

```http
PATCH /api/v1/webhooks/3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

Response `200 OK`: mesmo formato de um item de C2 (sem `secret`).

| Status | Quando |
|--------|--------|
| 200 | Atualizado (afeta apenas eventos futuros — [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)) |
| 400 `VALIDATION_ERROR` | Corpo vazio, URL não-`https`, `events` inválido |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente |

### C4 — `DELETE /api/v1/webhooks/:id` — remover webhook

```http
DELETE /api/v1/webhooks/3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90
Authorization: Bearer eyJhbGciOi...
```

Response 204 No Content (sem corpo), como DELETE /orders/:id em src/modules/orders/order.controller.ts. **Proposta de implementação (RFC OQ-11):** exclusão lógica (active = false, deletedAt = now). Não gera novos eventos para esse cadastro; linhas pendentes permanecem na outbox e o worker as move à DLQ com WEBHOOK_INACTIVE, preservando payload e histórico. Replay de endpoint excluído devolve 422 WEBHOOK_INACTIVE. A equipe deve revisar essa semântica antes de fechar o contrato.

| Status | Quando |
|--------|--------|
| 204 | Removido |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente |

### C5 — `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret

Request ([09:21] Sofia): sem corpo.

```http
POST /api/v1/webhooks/3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90/secret/rotate
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK`:

```json
{
  "id": "3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90",
  "secret": "a1b2c3d4e5f60718293a4b5c6d7e8f904f9c2e7a1b3d5f60718293a4b5c6d7e8",
  "previousSecretExpiresAt": "2026-11-13T13:00:00.000Z"
}
```

| Status | Quando |
|--------|--------|
| 200 | Nova secret emitida; antiga válida até `previousSecretExpiresAt` (24 h) |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente |

### C6 — `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Request ([09:34] Marcos): `page` e `pageSize` (padrão 100, máximo 100 — "os últimos 100 webhooks"), usando `paginated()` de `src/shared/http/response.ts`.

```http
GET /api/v1/webhooks/3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90/deliveries?pageSize=100
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK` (mais recentes primeiro):

```json
{
  "data": [
    {
      "id": "b7e2...",
      "eventId": "5b0b6c1e-8f3a-4a52-9a43-0f6d2a7c9e11",
      "attempt": 2,
      "result": "SUCCESS",
      "requestBody": "{\"event_id\":\"5b0b6c1e-...\",\"event_type\":\"order.status_changed\",...}",
      "responseStatus": 200,
      "responseBody": "ok",
      "durationMs": 184,
      "errorCode": null,
      "createdAt": "2026-11-12T14:04:29.130Z"
    },
    {
      "id": "a91c...",
      "eventId": "5b0b6c1e-8f3a-4a52-9a43-0f6d2a7c9e11",
      "attempt": 1,
      "result": "FAILURE",
      "requestBody": "{\"event_id\":\"5b0b6c1e-...\",...}",
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10001,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "createdAt": "2026-11-12T14:03:29.020Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

| Status | Quando |
|--------|--------|
| 200 | Histórico |
| 400 `VALIDATION_ERROR` | `pageSize` > 100 ou `id` não-UUID |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente |

### C7 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ (ADMIN)

Request ([09:18] Diego, [09:36] Sofia): sem corpo.

```http
POST /api/v1/admin/webhooks/dead-letter/e4f5a6b7-1c2d-4e3f-9a8b-7c6d5e4f3a21/replay
Authorization: Bearer <JWT com role ADMIN>
```

Response `202 Accepted` — o evento volta para a fila; a entrega é assíncrona:

```json
{
  "deadLetterId": "e4f5a6b7-1c2d-4e3f-9a8b-7c6d5e4f3a21",
  "outboxId": "0c9d8e7f-6a5b-4c3d-2e1f-0a9b8c7d6e5f",
  "eventId": "5b0b6c1e-8f3a-4a52-9a43-0f6d2a7c9e11",
  "status": "PENDING",
  "replayedAt": "2026-11-13T09:15:00.000Z",
  "replayedById": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d"
}
```

| Status | Quando |
|--------|--------|
| 202 | Recolocado na outbox |
| 401 `UNAUTHORIZED` | Sem token |
| 403 `FORBIDDEN` | Role diferente de `ADMIN` (`requireRole`) |
| 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND` | Registro inexistente |
| 409 `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Já reprocessado |
| 422 `WEBHOOK_INACTIVE` | Endpoint removido ou inativo |
| 422 WEBHOOK_PAYLOAD_TOO_LARGE | Payload preservado na DLQ excede o limite de envio |

> A listagem/consulta da DLQ por API não foi discutida na reunião; até lá, o `id` é obtido nos logs `webhook_moved_to_dead_letter` ou por consulta à tabela (RFC OQ-08).

### C8 — Requisição enviada ao cliente (outbound)

```http
POST https://hooks.atlascomercial.com.br/oms
Content-Type: application/json
X-Event-Id: 5b0b6c1e-8f3a-4a52-9a43-0f6d2a7c9e11
X-Webhook-Id: 3d6f1a2b-7c4e-4b9a-8e21-0a5f6c7d8e90
X-Timestamp: 2026-11-12T14:03:29.010Z
X-Signature: sha256=9f2c6a1e...e07b

{"event_id":"5b0b6c1e-8f3a-4a52-9a43-0f6d2a7c9e11","event_type":"order.status_changed","timestamp":"2026-11-12T14:03:27.512Z","order_id":"c2f7f0a4-3b8e-4a8e-9d2b-6a1f0e5c4b33","order_number":"ORD-000123","from_status":"PROCESSING","to_status":"SHIPPED","customer_id":"9e1d4b7a-2c6f-4f0e-8a3b-5d7c1e2f9a08","total_cents":18000}
```

| Header | Valor | Fonte |
|--------|-------|-------|
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento, igual em todas as tentativas e no replay | [09:25] Diego |
| `X-Webhook-Id` | id do cadastro que originou o envio | [09:44] Sofia |
| `X-Timestamp` | ISO 8601 do envio (muda a cada tentativa), para o cliente detectar replay se quiser | [09:44] Diego |
| `X-Signature` | `sha256=<hex(HMAC_SHA256(secret, corpo))>`; durante a convivência pós-rotação: `sha256=<nova>,sha256=<antiga>` | [09:20] Sofia, [09:22] Sofia; formato: proposta (RFC OQ-04) |

**Semântica para o cliente:** qualquer `2xx` em até 10 s confirma a entrega; qualquer outra coisa provoca retry. O mesmo `X-Event-Id` pode chegar mais de uma vez e deve ser deduplicado ([09:24] Diego).

Assinatura (`src/modules/webhooks/webhook.signer.ts`):

```ts
import { createHmac } from 'node:crypto';

export function sign(secret: string, body: string): string {
  return `sha256=${createHmac('sha256', secret).update(body, 'utf8').digest('hex')}`;
}
```

Verificação recomendada no cliente (para o portal do desenvolvedor, [09:26] Marcos): recalcular sobre o corpo bruto recebido e comparar com `crypto.timingSafeEqual` contra cada valor de `X-Signature`.

## 7. Matriz de erros

Erros HTTP estendem `AppError` e são adicionados em `src/shared/errors/http-errors.ts` (exportados por `src/shared/errors/index.ts`), como `InsufficientStockError` ([09:28] Bruno). Erros de entrega não chegam à API: são gravados em `webhook_deliveries.errorCode` / `webhook_dead_letter.reason`.

| Código | HTTP | Classe / origem | Quando | Retry? |
|--------|------|-----------------|--------|--------|
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError extends AppError` | `:id` de webhook inexistente (C3–C6) | — |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError extends AppError` | `customerId` inexistente (C1, C2) | — |
| `WEBHOOK_INVALID_URL` | 400 | Mensagem do refine Zod; sai como `VALIDATION_ERROR` com `details[{ path: "url", message: "WEBHOOK_INVALID_URL" }]` via `validate()` | URL não-`https` ou inválida ([09:23] Sofia) | — |
| `WEBHOOK_INVALID_EVENTS` | 400 | Idem, `details[{ path: "events", ... }]` | Lista vazia ou status fora de `OrderStatus` | — |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError extends AppError` | Replay de id inexistente (C7) | — |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `new ConflictError(msg, 'WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED')` | Replay repetido (C7) | — |
| `WEBHOOK_INACTIVE` | 422 | `new UnprocessableEntityError(msg, 'WEBHOOK_INACTIVE')` | Replay para endpoint inativo/removido (C7); no worker vira motivo de DLQ | Não |
| `WEBHOOK_SECRET_REQUIRED` | — | Worker | Endpoint sem secret ao assinar ([09:28] Bruno) | Não → DLQ |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 no replay | Publisher / replay | Snapshot > 64 KB ([09:24] Diego) | Não → DLQ |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | Worker | Sem resposta em 10 s ([09:42] Diego) | Sim |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | Worker | DNS, conexão recusada, falha TLS | Sim |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | — | Worker | Resposta fora de 2xx (inclui 3xx) | Sim |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | Worker | Esgotou os 5 reenvios | → DLQ |

Erros reaproveitados sem alteração: `UNAUTHORIZED` (401) e `FORBIDDEN` (403) de `src/middlewares/auth.middleware.ts`, `VALIDATION_ERROR` (400) de `src/middlewares/validate.middleware.ts` e `INTERNAL_SERVER_ERROR` (500) de `src/middlewares/error.middleware.ts`.

## 8. Estratégias de resiliência

| Aspecto | Estratégia | Fonte |
|---------|-----------|-------|
| Isolamento da transação de pedidos | Nenhuma rede dentro de `changeStatus`; só `INSERT` na outbox | [09:04] Bruno, [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Atomicidade | Outbox na mesma transação; falha ⇒ rollback | [09:40] Bruno |
| Timeout | `AbortSignal.timeout(10_000)` por chamada | [09:42] Diego |
| Retries | 5 reenvios: 1m/5m/30m/2h/12h | [09:17] Larissa |
| Fallback | DLQ persistida + replay manual ADMIN; sem e-mail nesta fase | [09:18] Diego, [09:37] Larissa |
| Idempotência | `event_id` estável; cliente deduplica | [09:25] Diego |
| Queda do worker | Lease `lockedUntil` (60 s) devolve linhas `PROCESSING` presas ao ciclo; o reenvio resultante é coberto pelo at-least-once | proposta de implementação |
| Reinício da API | Não afeta entregas (processo separado) | [09:11] Diego |
| Ordenação | Single-worker + `ORDER BY createdAt` + bloqueio por `(webhookId, orderId)` | [09:12] Diego; RFC OQ-03 |
| Cliente lento | Timeout limita cada chamada a 10 s; atraso acumulado monitorado pela idade do pendente mais antigo | [09:42] Diego |
| Payload anômalo | > 64 KB não é enviado; vai para DLQ | [09:24] Diego |
| Shutdown | `SIGINT`/`SIGTERM` terminam a linha corrente e desconectam o Prisma | padrão de `src/server.ts` |

Parâmetros (novas variáveis em `src/config/env.ts`, com default): `WEBHOOK_POLL_INTERVAL_MS=2000`, `WEBHOOK_BATCH_SIZE=20`, `WEBHOOK_HTTP_TIMEOUT_MS=10000`, `WEBHOOK_MAX_PAYLOAD_BYTES=65536`.

## 9. Observabilidade

O projeto já tem Pino com logs estruturados e `requestId` por requisição (`src/middlewares/request-logger.middleware.ts`), mas **não** tem biblioteca de métricas nem de tracing. A decisão de não adicionar ferramentas novas ([09:29] Bruno, [09:30] Larissa) leva à abordagem abaixo.

### Logs (Pino, `src/shared/logger/index.ts`)

| Evento (msg) | Nível | Campos |
|--------------|-------|--------|
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `batchSize`, `signal` |
| `webhook_worker_cycle` | debug | `claimed`, `durationMs` |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `orderId`, `attempt`, `responseStatus`, `durationMs` |
| `webhook_delivery_failed` | warn | idem + `errorCode`, `nextAttemptAt` |
| `webhook_moved_to_dead_letter` | warn | `deadLetterId`, `eventId`, `webhookId`, `reason`, `attempts` |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, `replayedBy` (auditoria, [09:36] Sofia) |
| `webhook_secret_rotated` | info | `webhookId`, `userId`, `previousSecretExpiresAt` |
| `webhook_worker_error` | error | `err`, `outboxId` |

O `redact` do logger ganha `'*.secret'` e `'*.previousSecret'`, ao lado de `'*.token'` e `'*.password'` que já existem.

### Métricas (derivadas das tabelas e dos logs)

| Métrica | Como medir | Alvo / alerta |
|---------|-----------|---------------|
| Latência até a 1ª tentativa | `webhook_deliveries.createdAt (attempt=1) − webhook_outbox.createdAt` | p95 < 10 s ([09:02] Marcos) |
| Idade do pendente mais antigo | `NOW() − MIN(createdAt) WHERE status='PENDING' AND attempts=0` | alerta se > 10 s |
| Backlog por status | `COUNT(*) GROUP BY status` na outbox | tendência de crescimento |
| Taxa de sucesso por endpoint | `webhook_deliveries` por `webhookId` e `result` | queda abrupta |
| Entradas na DLQ | `COUNT(*)` em `webhook_dead_letter` por dia e `reason` | qualquer `WEBHOOK_PAYLOAD_TOO_LARGE`/`WEBHOOK_SECRET_REQUIRED` |
| Replays | `COUNT(*) WHERE replayedAt IS NOT NULL` | acompanhamento |

As consultas podem alimentar um job/endpoint de métricas no futuro; expor em Prometheus ou similar fica fora desta fase.

### Tracing (correlação)

Sem tracing distribuído no projeto, o rastreio de ponta a ponta é feito por identificadores: `orderId` (`order_status_history`) → `eventId` (outbox, deliveries, DLQ, logs e header `X-Event-Id` recebido pelo cliente) → `webhookId` (`X-Webhook-Id`). Dado um `X-Event-Id` informado por um cliente, o suporte encontra todas as tentativas, respostas e um eventual replay. Adotar OpenTelemetry não foi discutido e fica fora do escopo.

## 10. Integração com o sistema existente

| Arquivo existente | Como o módulo de webhooks se integra |
|-------------------|--------------------------------------|
| `src/modules/orders/order.service.ts` | Em `changeStatus`, logo após `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, adicionar `await publishWebhookEvent(tx, order, from, to);`. O `order` carregado no início da transação já tem `orderNumber`, `customerId` e `totalCents`. O tipo `TxClient = Prisma.TransactionClient` já declarado no arquivo é o tipo do primeiro parâmetro ([09:40] Bruno, [09:41] Bruno). O construtor de `OrderService` não muda (não injeta repositório, [09:41] Diego). `create()` não publica evento (ver PRD, fora de escopo). |
| `src/modules/orders/order.status.ts` | Fonte da máquina de estados; os status aceitos em `events` são os mesmos valores de `OrderStatus` usados em `transitions`. O publisher não revalida transições — confia que `changeStatus` já chamou `canTransition`. |
| `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` | Novas classes `WebhookNotFoundError`, `WebhookCustomerNotFoundError` e `WebhookDeadLetterNotFoundError` estendendo `AppError` (de `src/shared/errors/app-error.ts`) com códigos `WEBHOOK_*`; conflitos e 422 usam `ConflictError`/`UnprocessableEntityError` passando o código, como `InvalidStatusTransitionError` faz. Exportadas pelo `index.ts`. `NotFoundError` não é reaproveitada porque fixa o código `NOT_FOUND`. |
| `src/middlewares/error.middleware.ts` | **Sem alteração.** O branch `err instanceof AppError` já serializa `{ error: { code, message, details } }` para todas as novas classes ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas do módulo (`router.use(authenticate)`, como `order.routes.ts`); `requireRole('ADMIN')` na rota de replay ([09:36] Larissa). `req.user.id` alimenta `replayedById`. |
| prisma/schema.prisma (modelos User e Customer) | O modelo User tem role, mas não contém vínculo com Customer; autorização por customer exige decisão/modelagem adicional (RFC OQ-09). |
| `src/middlewares/validate.middleware.ts` | Rotas usam `validate({ body, params, query })` com os schemas de `webhook.schemas.ts`. Como o middleware converte toda falha Zod em `ValidationError` (`VALIDATION_ERROR`), os códigos `WEBHOOK_INVALID_URL`/`WEBHOOK_INVALID_EVENTS` vão como mensagem em `details`. |
| `src/routes/index.ts` | O tipo `Controllers` ganha `webhooks: WebhookController`; `buildApiRouter` registra `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. |
| `src/app.ts` | `buildControllers(prisma)` instancia `WebhookRepository`, `WebhookService` e `WebhookController`, como faz para orders. |
| `src/server.ts` → `src/worker.ts` (novo) | O worker copia o padrão de bootstrap: `logger.info(..., 'webhook_worker_started')`, handlers de `SIGINT`/`SIGTERM` com `prisma.$disconnect()`, `bootstrap().catch(... logger.fatal ...)`. |
| `src/config/database.ts` | O worker chama `createPrismaClient()` para ter sua própria instância ([09:30] Bruno); a API continua usando o `prisma` exportado. |
| `src/config/env.ts` | `envSchema` ganha as quatro variáveis `WEBHOOK_*` com `z.coerce.number().int().positive().default(...)`. |
| `src/shared/logger/index.ts` | Reuso do `logger`; inclusão de `'*.secret'` e `'*.previousSecret'` em `redactPaths`. |
| `src/shared/http/response.ts` | `paginated()` no histórico de entregas (C6). |
| `prisma/schema.prisma` | Novos enums e modelos da seção 4; relação `webhooks` em `Customer`; migration gerada com `npm run db:migrate`. |
| `package.json` | Novo script `"worker": "tsx --env-file=.env src/worker.ts"` (dev) — alinhado a `"dev"`; em produção, `node --env-file=.env dist/worker.js`. Nenhuma dependência nova (`fetch` nativo do Node ≥ 20, `node:crypto`). |
| `tests/setup.ts` / `tests/helpers/factories.ts` | O `beforeEach` passa a limpar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhookEndpoint` antes de `order`/`customer`; factory nova `createTestWebhook()`. |

### Estrutura do módulo

```
src/modules/webhooks/
├── webhook.controller.ts      # handlers HTTP (padrão try/next(err) de order.controller.ts)
├── webhook.routes.ts          # buildWebhookRouter, buildWebhookAdminRouter
├── webhook.schemas.ts         # Zod: create, update, params, query
├── webhook.service.ts         # CRUD, rotação de secret, deliveries, replay
├── webhook.repository.ts      # acesso Prisma às 4 tabelas
├── webhook.publisher.ts       # publishWebhookEvent(tx, order, from, to)
├── webhook.signer.ts          # HMAC-SHA256
├── webhook.worker.ts          # loop de polling, entrega, retry, DLQ
└── webhook.constants.ts       # delays de retry, nomes de headers, event_type
src/worker.ts                  # entry-point do processo
```

## 11. Dependências e compatibilidade

- **Runtime:** Node ≥ 20 (`engines` em `package.json`) — necessário para `fetch` e `AbortSignal.timeout` nativos.
- **Banco:** MySQL 8.0 (`docker-compose.yml`), necessário para `FOR UPDATE SKIP LOCKED`.
- **ORM:** Prisma 5.22.0; migration aditiva (só cria tabelas/enums e a relação inversa em `Customer`), sem alterar tabelas existentes. Rollback = remover as tabelas novas.
- **API:** nenhum endpoint existente muda de contrato. `PATCH /api/v1/orders/:id/status` passa a poder falhar se a inserção na outbox falhar (desejado).
- **Operação:** novo processo a manter no ar (`npm run worker`), exatamente uma instância.
- **Externas:** portal do desenvolvedor documentando assinatura e deduplicação ([09:26] Marcos, [09:40] Marcos); revisão de segurança da Sofia antes do deploy ([09:46] Sofia).

## 12. Critérios de aceite técnicos

| ID | Critério |
|----|----------|
| AC-T01 | Mudança de status para um status assinado cria exatamente uma linha `PENDING` por endpoint ativo interessado, com o mesmo `eventId`. |
| AC-T02 | Mudança para um status que nenhum endpoint escuta não cria linha na outbox. |
| AC-T03 | Se a inserção na outbox lançar erro, `orders.status` e `order_status_history` permanecem inalterados (teste de integração com falha forçada). |
| AC-T04 | Com o worker rodando e endpoint respondendo 200, a entrega ocorre em < 10 s e a linha fica `DELIVERED`. |
| AC-T05 | A requisição contém `Content-Type`, `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`; a assinatura confere com HMAC-SHA256 do corpo recebido usando a secret do endpoint. |
| AC-T06 | Endpoint que não responde em 10 s gera `WEBHOOK_DELIVERY_TIMEOUT` e reagendamento em +1 min. |
| AC-T07 | Falhas sucessivas seguem 1m/5m/30m/2h/12h; após o último reenvio a linha fica `FAILED` e existe registro em `webhook_dead_letter` com payload, motivo e timestamp. |
| AC-T08 | Replay por `ADMIN` devolve 202, recoloca a linha em `PENDING` com o mesmo `eventId`, grava `replayedById` e loga `webhook_dead_letter_replayed`; por `OPERATOR` devolve 403. Payload acima de 64 KB devolve 422 `WEBHOOK_PAYLOAD_TOO_LARGE`. |
| AC-T09 | Cadastro com URL `http://` devolve 400 `VALIDATION_ERROR` com `WEBHOOK_INVALID_URL` em `details`. |
| AC-T10 | A secret só aparece nas respostas de criação e rotação; não aparece em listagens nem em logs. |
| AC-T11 | Após rotação, envios nas 24 h seguintes trazem assinaturas válidas para a secret nova e para a antiga; depois, só para a nova. |
| AC-T12 | `GET /webhooks/:id/deliveries` devolve no máximo 100 itens, mais recentes primeiro, com resultado, payload, resposta e duração. |
| AC-T13 | Enquanto o evento anterior do mesmo pedido/endpoint estiver PENDING ou PROCESSING (incluindo backoff), o posterior não é reivindicado; após DLQ/replay, a ordem cronológica não é garantida até decisão de OQ-12. |
| AC-T14 | Reiniciar a API não interrompe entregas; matar o worker no meio de um envio faz a linha voltar a ser processada após o lease. |
| AC-T15 | Suíte `npm test` (Vitest + Supertest, MySQL real) cobre AC-T01–AC-T13; o worker é testado contra um servidor HTTP local, com o endpoint inserido direto via Prisma (o schema exige `https`). |

## 13. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|----|-------|---------------|---------|-----------|
| RISK-01 | Cliente não deduplica e processa o mesmo evento duas vezes (at-least-once) | Média | Médio | `X-Event-Id` estável; documentação destacada no portal ([09:26] Marcos) |
| RISK-02 | Outbox e histórico crescem sem arquivamento nesta fase | Alta | Baixo/Médio | Índices em status e `createdAt`; métrica de volume; arquivamento planejado para depois ([09:08] Diego) |
| RISK-03 | Single-worker vira gargalo quando vários endpoints ficam lentos (até 10 s cada) | Média | Alto (fere o SLA de 10 s) | Alerta de idade do pendente > 10 s; escalar exige particionar por `order_id` ([09:13] Diego) |
| RISK-04 | Inversão de ordem por pedido durante retries | Média | Médio | Bloqueio por `(webhookId, orderId)` no claim (F2); confirmar política (RFC OQ-03) |
| RISK-05 | Secret vazada pelo cliente ([09:22] Diego) | Média | Alto | Secret por endpoint, rotação self-service com 24 h de convivência |
| RISK-06 | Exposição das secrets armazenadas (recuperáveis para assinar) | Baixa | Alto | `redact` no logger, secret nunca listada; proteção em repouso definida na revisão de segurança (RFC OQ-06) |
| RISK-07 | Worker cai entre o envio e a marcação | Baixa | Baixo | Lease de 60 s + at-least-once; o cliente deduplica |
| RISK-08 | Acesso cruzado a webhooks/deliveries de outro customer por falta de vínculo de autorização no modelo atual | Média | Alto | Definir e aplicar autorização por customer antes de expor CRUD/histórico (RFC OQ-09); base: prisma/schema.prisma |
| RISK-09 | Worker alcançar destinos internos ou não autorizados por URL configurável | Média | Alto | Fechar política de egress, DNS e redirecionamentos com Segurança antes do deploy (RFC OQ-10); proposta, não decisão da reunião |
| RISK-10 | Rajada de mudanças de status "bombardeia" um cliente | Média | Médio | Observar volume por endpoint; decisão de rate limiting adiada ([09:39] Larissa) |
| RISK-11 | Atraso no prazo de fim de novembro da Atlas | Média | Alto | Três sprints com revisão de segurança incluída; escopo enxuto ([09:46] Larissa) |
