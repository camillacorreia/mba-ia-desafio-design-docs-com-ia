# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Autora | Camilla Correia |
| Status | Pronto para implementação, pendente das confirmações listadas no [RFC](RFC.md#5-questões-em-aberto) |
| Data | 17/11/2026 |
| Base | [RFC](RFC.md), [ADR-001 a ADR-007](adrs/), transcrição de quinta-feira 09:00 ([TRANSCRICAO.md](../TRANSCRICAO.md)) |

Este documento diz como construir. O porquê de cada decisão está nos ADRs, e a proposta em nível de arquitetura está no RFC. Nomes de tabelas, colunas, rotas e códigos que a reunião não fixou são escolhas deste FDD; quando é o caso, o texto diz "escolha deste FDD" e o [Tracker](TRACKER.md) aponta a fala ou o arquivo que motivou a escolha.

## 1. Contexto e motivação técnica

A mudança de status de pedido vive em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126 a 179). O método abre uma transação interativa do Prisma, valida a transição com `canTransition` (`src/modules/orders/order.status.ts`), debita ou repõe estoque conforme `shouldDebitStock` e `shouldReplenishStock`, atualiza `orders.status` e insere uma linha em `order_status_history`. Não existe nenhum mecanismo de evento, fila ou chamada externa na aplicação.

Os clientes B2B querem ser avisados dessas mudanças em menos de 10 segundos ([09:02] Marcos). Colocar a chamada HTTP dentro da transação travaria a mudança de status ([09:04] Bruno), então a entrega passa a ser assíncrona: a transação grava o evento numa outbox ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) e um worker separado faz a entrega ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

| ID | Objetivo | Origem |
|---|---|---|
| FDD-OBJ-01 | Toda mudança de status que case com o filtro de pelo menos um webhook ativo do cliente gera evento na outbox na mesma transação; sem evento, sem mudança de status | [09:40] Bruno |
| FDD-OBJ-02 | O evento chega ao cliente em menos de 10 segundos quando o endpoint do cliente está saudável | [09:02] Marcos, [09:09] Diego |
| FDD-OBJ-03 | Nenhuma chamada HTTP a cliente acontece dentro do processo da API | [09:06] Diego, [09:11] Diego |
| FDD-OBJ-04 | O cliente consegue verificar origem e integridade de cada requisição | [09:19] Sofia |
| FDD-OBJ-05 | Nenhum evento enfileirado se perde em silêncio: ou é entregue, ou fica pendente enquanto o webhook estiver pausado, ou vai para a DLQ com o motivo. A exceção é a remoção do webhook pelo cliente, que descarta os eventos dele junto. Mudanças de status ocorridas com o webhook pausado não geram evento, porque o filtro é aplicado na inserção | [09:18] Diego |
| FDD-OBJ-06 | Nenhuma dependência nova de runtime | [09:29] Bruno, [09:30] Larissa |

## 3. Escopo e exclusões

**Dentro do escopo:** tabelas de configuração, outbox, histórico de entregas e DLQ; função `publishWebhookEvent` chamada por `changeStatus`; API de gestão (criar, listar, editar, remover, rotacionar secret, histórico de entregas); endpoint admin de replay; worker em `src/worker.ts`; assinatura HMAC; retry e DLQ; testes.

**Fora do escopo:** os itens de produto descartados ou adiados na reunião (e-mail de aviso, dashboard, rate limiting, arquivamento e outros) estão no [PRD, seção 5.2](PRD.md#52-fora-de-escopo). As exclusões técnicas próprias deste FDD são:

| Item | Situação | Origem |
|---|---|---|
| Múltiplos workers em paralelo | Futuro (particionamento ou lock) | [09:13] Diego |
| Evento na criação do pedido | A reunião só tratou mudança de status em `changeStatus`; `OrderService.create` não publica evento | [09:40] Bruno |
| Listagem da DLQ por API | Não discutida; o `id` para replay vem do log do worker ou do banco | [09:18] Diego |

## 4. Modelo de dados

Quatro models novos em `prisma/schema.prisma`, com migration gerada por `npm run db:migrate`. Todos os ids são UUID em `Char(36)`, como o resto do schema ([09:51] Larissa). Os nomes de tabela `webhook_outbox` e `webhook_dead_letter` vêm da reunião ([09:06] Diego, [09:18] Diego); `webhooks` e `webhook_deliveries` são escolhas deste FDD.

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  FAILED       // falhou: tentativas esgotadas, cópia na DLQ
  DELIVERED    // entregue
}

model Webhook {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  statuses                Json      // lista de OrderStatus que o webhook quer receber
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)
  payload       Json                // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastErrorCode String?             @db.VarChar(64)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    Webhook           @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

  @@unique([eventId, webhookId])
  @@index([status])
  @@index([createdAt])
  @@index([status, nextAttemptAt])
  @@index([webhookId, orderId, createdAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  outboxId       String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text
  durationMs     Int
  errorCode      String?  @db.VarChar(64)
  createdAt      DateTime @default(now())

  webhook Webhook       @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  outbox  WebhookOutbox @relation(fields: [outboxId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  outboxId     String    @db.Char(36)
  eventId      String    @db.Char(36)
  webhookId    String    @db.Char(36)
  payload      Json
  reason       String    @db.VarChar(64)
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  webhook Webhook @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas:

- A configuração guarda URL, secret, cliente e estado ativo, como combinado ([09:21] Bruno, [09:21] Sofia). O `statuses` guarda o filtro por status ([09:33] Marcos).
- Os índices em `status` e `createdAt` da outbox vêm da reunião ([09:08] Diego). `nextAttemptAt` é necessário para o backoff, que a reunião definiu sem dizer onde guardar o próximo horário; o índice composto `(status, nextAttemptAt)` atende a consulta do worker, e `(webhookId, orderId, createdAt)` atende a regra de ordem (seção 5.2). As três são escolhas deste FDD.
- Uma linha de outbox por par evento e webhook. O `eventId` é o mesmo para todos os webhooks que recebem a mesma mudança de status ("único por evento", [09:25] Diego), e o `X-Webhook-Id` diz ao cliente qual cadastro gerou aquele envio ([09:44] Sofia).
- A secret fica em texto, porque o HMAC precisa do valor original para assinar. Criptografia em repouso não foi discutida na reunião; este FDD sugere incluí-la na revisão de segurança que Sofia pediu para o HMAC e a geração de secret ([09:46] Sofia).
- `Customer` ganha a relação `webhooks Webhook[]`. Com `onDelete: Cascade`, remover um cliente remove os webhooks dele e todo o histórico associado.
- Como no resto de `prisma/schema.prisma`, os campos não usam `@map`: as colunas no MySQL ficam em camelCase (`nextAttemptAt`, `webhookId`), e só as tabelas são mapeadas para snake_case com `@@map`. A consulta SQL da seção 5.2 segue essa convenção.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

1. `PATCH /api/v1/orders/:id/status` chega a `OrderController.changeStatus` (`src/modules/orders/order.controller.ts`), que chama `OrderService.changeStatus(id, input, userId)`.
2. Dentro de `this.prisma.$transaction(async (tx) => ...)`, o fluxo atual segue igual até gravar `orderStatusHistory`. Só transições válidas chegam aqui: `from === to` e `!canTransition(from, to)` já lançam erro antes.
3. Logo depois de `tx.orderStatusHistory.create(...)`, o serviço chama `publishWebhookEvent(tx, order, from, to)`, exportada por `src/modules/webhooks/webhook.publisher.ts` ([09:41] Bruno).
4. `publishWebhookEvent` busca, com o mesmo `tx`, os webhooks com `customerId = order.customerId` e `active = true`, e filtra em memória os que têm `to` em `statuses` ([09:34] Bruno). Se nenhum casar, retorna sem gravar nada.
5. Gera um `eventId` (UUID v4, com o pacote `uuid` que já está nas dependências) e renderiza o payload uma única vez (seção 6.2), com `timestamp` igual ao horário da mudança ([09:52] Larissa).
6. Grava uma linha em `webhook_outbox` por webhook interessado, com `status = PENDING`, `attempts = 0` e `nextAttemptAt = now()`, usando `tx.webhookOutbox.createMany`.
7. Qualquer erro nesse ponto propaga e desfaz a transação inteira: o status não muda e a API responde erro ([09:40] Bruno). O publisher registra `webhook_outbox_write_failed` com o erro original e o `orderId` e relança como `WebhookOutboxWriteError` (500, `WEBHOOK_OUTBOX_WRITE_FAILED`). O log vem do publisher porque o `errorMiddleware` responde erros `AppError` sem registrá-los.

```ts
// src/modules/orders/order.service.ts, dentro de changeStatus
await tx.order.update({ where: { id }, data: { status: to } });
await tx.orderStatusHistory.create({ data: { orderId: id, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null } });
await publishWebhookEvent(tx, order, from, to); // novo
```

### 5.2 Processamento pelo worker

O loop roda em `src/modules/webhooks/webhook.processor.ts`, iniciado por `src/worker.ts`. A reunião deixou o nome do arquivo entre `webhook.worker.ts` e `webhook.processor.ts` ([09:28] Bruno); este FDD fica com `webhook.processor.ts`, para não confundir com o entry point.

1. **Partida.** `src/worker.ts` valida o ambiente com `env` (`src/config/env.ts`), cria o próprio client com `createPrismaClient()` (`src/config/database.ts`) ([09:30] Bruno) e devolve para `PENDING` qualquer linha que tenha ficado em `PROCESSING` por queda anterior. Como a entrega é at-least-once, reenviar é seguro ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
2. **Ciclo.** A cada `WEBHOOK_POLL_INTERVAL_MS` (2000 ms, [09:09] Diego), busca até `WEBHOOK_BATCH_SIZE` linhas ([09:08] Diego fala em "batch pequeno"; o valor inicial 10 é escolha deste FDD):

   ```sql
   SELECT o.* FROM `webhook_outbox` o
   JOIN `webhooks` w ON w.`id` = o.`webhookId` AND w.`active` = TRUE
   WHERE o.`status` = 'PENDING' AND o.`nextAttemptAt` <= ?   -- new Date() do processo, em UTC
     AND NOT EXISTS (
       SELECT 1 FROM `webhook_outbox` prev
       WHERE prev.`webhookId` = o.`webhookId` AND prev.`orderId` = o.`orderId`
         AND prev.`createdAt` < o.`createdAt`
         AND prev.`status` IN ('PENDING', 'PROCESSING')
     )
   ORDER BY o.`createdAt`
   LIMIT ?;
   ```

   O horário de corte vai como parâmetro (`new Date()`), e não como `NOW()` do MySQL, porque o Prisma grava `DateTime` em UTC e o `NOW()` usa o fuso da sessão. O `JOIN` com `active = TRUE` deixa de fora eventos de webhooks pausados, que ficam `PENDING` sem ocupar o lote e voltam a ser elegíveis quando o webhook é reativado. O `NOT EXISTS` é a regra de ordem por pedido: um evento só sai depois que o anterior do mesmo pedido, para o mesmo webhook, foi entregue ou foi para a DLQ ([09:12] Diego). Essa regra é proposta deste FDD e está em confirmação no RFC. Ela também garante que o lote nunca tem dois eventos do mesmo par `(webhookId, orderId)`. Como Prisma não expressa esse filtro, a consulta usa `$queryRaw`.
3. **Reserva.** Marca as linhas selecionadas como `PROCESSING`. Com um worker só, não há disputa; a reserva serve para a recuperação do passo 1.
4. **Envio**, em paralelo e sem bloquear o ciclo. Cada linha reservada vira um envio assíncrono, e o próximo ciclo começa 2 segundos depois mesmo que haja envios em andamento. O limite de lote passa a ser de envios simultâneos: cada ciclo busca no máximo `WEBHOOK_BATCH_SIZE` menos o número de envios ainda em andamento. Como linhas em `PROCESSING` bloqueiam, pelo `NOT EXISTS`, os eventos seguintes do mesmo par `(webhookId, orderId)`, o paralelismo não quebra a ordem por pedido, e um cliente lento não atrasa os envios dos outros. Para cada linha:
   1. Serializa o payload (`JSON.stringify`). Se passar de `WEBHOOK_MAX_PAYLOAD_BYTES` (64 KB, [09:24] Larissa), não envia: vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry.
   2. Calcula `X-Signature` (seção 6.2) e faz o `POST` com `fetch` nativo do Node 20 (`engines.node >= 20` em `package.json`), `AbortSignal.timeout(WEBHOOK_HTTP_TIMEOUT_MS)` (10 s, [09:42] Diego) e `redirect: 'manual'`.
   3. Grava uma linha em `webhook_deliveries` com tentativa, sucesso, status HTTP, corpo da resposta (até 64 KB), duração e código de erro ([09:34] Marcos).
   4. Resposta 2xx: `status = DELIVERED`, `deliveredAt = now()`, `attempts + 1`. Qualquer outro resultado segue para o retry (5.3).

### 5.3 Retry

A tabela de backoff segue [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md), com a leitura de um envio inicial mais 5 retentativas ([09:17] Diego):

| Envio que falhou | Próximo envio em | Acumulado desde a primeira falha |
|---|---|---|
| 1º (inicial) | 1 min | 1 min |
| 2º | 5 min | 6 min |
| 3º | 30 min | 36 min |
| 4º | 2 h | 2 h 36 min |
| 5º | 12 h | 14 h 36 min |
| 6º | sem novo envio: DLQ | |

Falha é: status HTTP fora de 2xx (inclusive 3xx, já que redirecionamentos não são seguidos), timeout de 10 segundos ([09:42] Diego) ou erro de rede. A reunião não diferenciou 4xx de 5xx, então todos entram no retry. Ao falhar, o worker incrementa `attempts`, grava `lastErrorCode` e define `status = PENDING` e `nextAttemptAt = now() + intervalo`. Os intervalos ficam numa constante `WEBHOOK_RETRY_SCHEDULE_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]`.

### 5.4 DLQ e replay

**Entrada na DLQ.** Quando o 6º envio falha, ou numa falha sem retry (`WEBHOOK_PAYLOAD_TOO_LARGE`), o worker, numa única transação, marca a outbox como `FAILED` e insere em `webhook_dead_letter` o payload, o motivo e `failedAt` ([09:18] Diego). O motivo é `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` quando as tentativas acabaram, e nesse caso o erro da última tentativa fica em `webhook_deliveries`; ou `WEBHOOK_PAYLOAD_TOO_LARGE`, quando não houve envio e, portanto, nenhuma linha em `webhook_deliveries`.

**Replay** por `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, só para `ADMIN` ([09:36] Larissa):

1. Busca a linha da DLQ. Se não existir: `WEBHOOK_DEAD_LETTER_NOT_FOUND`. Se `replayedAt` já estiver preenchido: `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
2. Na mesma transação, volta a linha original da outbox para `PENDING`, com `attempts = 0` e `nextAttemptAt = now()` ([09:18] Diego: "recoloca na outbox como pendente"), e preenche `replayedAt` e `replayedById` na DLQ.
3. Registra em log `webhook_dead_letter_replayed` com `deadLetterId`, `eventId` e o `userId` de quem pediu ([09:36] Sofia).

O replay reusa a mesma linha de outbox e o mesmo `eventId`, então o cliente que já tinha recebido o evento consegue deduplicar. Zerar as tentativas dá ao evento um ciclo completo de retry. A reunião não falou da contagem no replay; isso é escolha deste FDD. Um evento reprocessado chega depois de eventos mais novos do mesmo pedido que já foram entregues, então o replay é a exceção à ordem por pedido; o cliente se orienta pelo `timestamp` do payload.

### 5.5 Rotação de secret

1. `POST /api/v1/webhooks/:id/rotate-secret` recusa com 409 `WEBHOOK_ROTATION_IN_PROGRESS` se `previousSecretExpiresAt` ainda estiver no futuro. Sem essa trava, uma segunda rotação dentro de 24 horas sobrescreveria a secret original antes do prazo prometido em [09:21] Sofia.
2. Gera uma secret nova (32 bytes de `crypto.randomBytes`, em hexadecimal).
3. Copia a secret atual para `previousSecret`, define `previousSecretExpiresAt = now() + 24h` ([09:21] Sofia) e grava a nova em `secret`.
4. Devolve a secret nova uma única vez, na resposta. Nenhum outro endpoint devolve secret.
5. Enquanto `previousSecretExpiresAt` estiver no futuro, o worker assina com as duas (seção 6.2). Depois disso, a anterior é ignorada e limpa na próxima rotação.

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (`src/app.ts`), exigem `Authorization: Bearer <JWT>` (`authenticate`) e respondem erro no envelope do middleware central: `{ "error": { "code", "message", "details"? } }`. Os corpos da API de gestão usam camelCase, como o resto da API (por exemplo `customerId` em `src/modules/orders/order.schemas.ts`); o payload enviado ao cliente usa snake_case, como definido na reunião ([09:43] Diego).

### 6.1 API de gestão

O `customerId` vai no corpo (criação) ou no path (listagem), e não vem do JWT ([09:32] Larissa). O CRUD aceita qualquer papel autenticado ([09:37] Sofia); só o replay exige `ADMIN`.

| # | Método e rota | Papel | Sucesso | Erros |
|---|---|---|---|---|
| 1 | `POST /api/v1/webhooks` | autenticado | 201 | 400, 401, 404 |
| 2 | `GET /api/v1/customers/:customerId/webhooks` | autenticado | 200 | 400, 401 |
| 3 | `PATCH /api/v1/webhooks/:id` | autenticado | 200 | 400, 401, 404 |
| 4 | `DELETE /api/v1/webhooks/:id` | autenticado | 204 | 400, 401, 404 |
| 5 | `POST /api/v1/webhooks/:id/rotate-secret` | autenticado | 200 | 400, 401, 404, 409 |
| 6 | `GET /api/v1/webhooks/:id/deliveries` | autenticado | 200 | 400, 401, 404 |
| 7 | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | `ADMIN` | 202 | 400, 401, 403, 404, 409 |

Os caminhos 1, 3 e 4 seguem o padrão REST dos módulos existentes; o 2 coloca o cliente no path, uma das duas opções dadas por Larissa ([09:32] Larissa); os de 6 e 7 vêm da reunião ([09:34] Marcos, [09:18] Diego); o de 5 é escolha deste FDD, já que a reunião só disse "endpoint pro cliente conseguir pedir nova secret pela API" ([09:21] Sofia).

**1. Criar webhook** ([09:31] Marcos)

```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "3f2b8c1e-7a4d-4e8b-9c1a-2d5e6f7a8b9c",
  "url": "https://hooks.atlas.example.com/oms",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

```http
HTTP/1.1 201 Created

{
  "id": "9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
  "customerId": "3f2b8c1e-7a4d-4e8b-9c1a-2d5e6f7a8b9c",
  "url": "https://hooks.atlas.example.com/oms",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "5e8f2a9c...c41d",
  "createdAt": "2026-11-17T12:00:00.000Z",
  "updatedAt": "2026-11-17T12:00:00.000Z"
}
```

Validação Zod: `customerId` UUID; `url` URL válida com protocolo `https:` ([09:23] Sofia); `statuses` lista não vazia e sem repetição de `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` e `CANCELLED`. `PENDING` fica de fora porque nenhuma transição de `src/modules/orders/order.status.ts` leva a ele, então um webhook só com `PENDING` nunca dispararia. Cliente inexistente: 404 `WEBHOOK_CUSTOMER_NOT_FOUND`. A `secret` só aparece aqui e na rotação.

**2. Listar webhooks de um cliente** ([09:33] Bruno)

```http
GET /api/v1/customers/3f2b8c1e-7a4d-4e8b-9c1a-2d5e6f7a8b9c/webhooks
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
      "customerId": "3f2b8c1e-7a4d-4e8b-9c1a-2d5e6f7a8b9c",
      "url": "https://hooks.atlas.example.com/oms",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-17T12:00:00.000Z",
      "updatedAt": "2026-11-17T12:00:00.000Z"
    }
  ]
}
```

`customerId` no path, validado como UUID. Sem paginação: o volume esperado é de poucos webhooks por cliente.

**3. Editar webhook** ([09:33] Bruno)

```http
PATCH /api/v1/webhooks/9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "statuses": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

```http
HTTP/1.1 200 OK

{
  "id": "9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
  "customerId": "3f2b8c1e-7a4d-4e8b-9c1a-2d5e6f7a8b9c",
  "url": "https://hooks.atlas.example.com/oms",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-11-17T12:00:00.000Z",
  "updatedAt": "2026-11-17T12:30:00.000Z"
}
```

Campos editáveis: `url`, `statuses` e `active`, todos opcionais, com pelo menos um presente. O filtro novo só vale para mudanças de status futuras ([ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md)). `active: false` pausa o webhook sem perder histórico: eventos já enfileirados ficam pendentes até a reativação, e mudanças de status ocorridas durante a pausa não geram evento.

**4. Remover webhook** ([09:33] Bruno)

```http
DELETE /api/v1/webhooks/9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 204 No Content
```

Remove a configuração e, em cascata, a outbox pendente, o histórico de entregas e a DLQ desse webhook. Os eventos pendentes são descartados por decisão do próprio cliente, a única situação em que um evento some sem entrega nem DLQ. Quem quiser preservar eventos e histórico deve usar `PATCH` com `active: false`.

**5. Rotacionar secret** ([09:21] Sofia)

```http
POST /api/v1/webhooks/9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f/rotate-secret
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 200 OK

{
  "id": "9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
  "secret": "a71c09e4...9f02",
  "previousSecretExpiresAt": "2026-11-18T12:30:00.000Z"
}
```

**6. Histórico de entregas** ([09:34] Marcos)

```http
GET /api/v1/webhooks/9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f/deliveries
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "c4d5e6f7-0a1b-4c2d-8e3f-1a2b3c4d5e6f",
      "eventId": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "payload": { "event_id": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b", "event_type": "order.status_changed", "...": "..." },
      "createdAt": "2026-11-17T13:01:05.000Z"
    },
    {
      "id": "b3c4d5e6-f7a8-4b9c-8d0e-1f2a3b4c5d6e",
      "eventId": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b",
      "attempt": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "payload": { "event_id": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b", "event_type": "order.status_changed", "...": "..." },
      "createdAt": "2026-11-17T13:00:05.000Z"
    }
  ]
}
```

Devolve os 100 envios mais recentes daquele webhook, do mais novo para o mais antigo, com sucesso ou falha, payload, resposta e tempo de resposta ([09:34] Marcos). Cada tentativa é um envio: Marcos pediu "os últimos 100 webhooks que vocês mandaram", e cada retry é uma requisição mandada ao cliente. Com retries, 100 envios podem corresponder a menos de 100 eventos; contar por envio é escolha deste FDD. O `payload` vem da linha de outbox correspondente.

**7. Replay de item da DLQ** ([09:18] Diego, [09:36] Larissa)

```http
POST /api/v1/admin/webhooks/dead-letter/7a8b9c0d-1e2f-4a3b-8c4d-5e6f7a8b9c0d/replay
Authorization: Bearer eyJhbGciOi... (papel ADMIN)
```

```http
HTTP/1.1 202 Accepted

{
  "deadLetterId": "7a8b9c0d-1e2f-4a3b-8c4d-5e6f7a8b9c0d",
  "outboxId": "0d1e2f3a-4b5c-4d6e-8f7a-8b9c0d1e2f3a",
  "eventId": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b",
  "status": "PENDING"
}
```

202 porque a entrega acontece depois, no worker. Usuário sem papel `ADMIN` recebe 403 `FORBIDDEN`, gerado pelo `requireRole` existente.

### 6.2 Contrato de saída (requisição enviada ao cliente)

```http
POST https://hooks.atlas.example.com/oms
Content-Type: application/json
X-Event-Id: e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b
X-Webhook-Id: 9b1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f
X-Timestamp: 2026-11-17T13:00:05.120Z
X-Signature: sha256=8c1f4e...b20a

{
  "event_id": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-17T13:00:03.512Z",
  "order_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3f2b8c1e-7a4d-4e8b-9c1a-2d5e6f7a8b9c",
  "total_cents": 18500
}
```

| Header | Conteúdo | Origem |
|---|---|---|
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento, igual ao `event_id` do corpo; chave de deduplicação | [09:25] Diego |
| `X-Webhook-Id` | id do cadastro que gerou o envio | [09:44] Sofia |
| `X-Timestamp` | horário do envio desta tentativa, ISO 8601 em UTC. Informativo: fica fora da assinatura (ver [RFC, questão 8](RFC.md#5-questões-em-aberto)) | [09:44] Diego |
| `X-Signature` | `sha256=` seguido do HMAC-SHA256 em hexadecimal dos bytes exatos do corpo, com a secret do webhook | [09:20] Sofia, [09:22] Sofia |

**Corpo.** É o snapshot gravado na outbox ([09:52] Larissa), serializado uma vez por envio e assinado nesses mesmos bytes. Os campos são os definidos por Diego em [09:43]; `timestamp` é o horário da mudança de status, e não o do envio. Itens do pedido não vão no payload: o cliente consulta `GET /api/v1/orders/:id` se precisar ([09:43] Diego). Entre os "campos básicos da order", a reunião só nomeou `total_cents`, e é só esse que entra.

**Assinatura durante a rotação.** Enquanto a secret anterior estiver na janela de 24 horas, `X-Signature` leva as duas assinaturas separadas por vírgula, a da secret nova primeiro: `sha256=<hex_nova>,sha256=<hex_anterior>`. O cliente aceita a requisição se qualquer uma bater. O formato é escolha deste FDD para implementar a convivência pedida em [09:21] Sofia.

**Verificação no cliente**, para o portal do desenvolvedor ([09:26] Marcos):

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

export function isValidSignature(rawBody: Buffer, header: string, secret: string): boolean {
  const expected = createHmac('sha256', secret).update(rawBody).digest(); // 32 bytes
  return header.split(',').some((part) => {
    const hex = part.trim().replace(/^sha256=/, '');
    if (!/^[0-9a-f]{64}$/i.test(hex)) return false;
    return timingSafeEqual(Buffer.from(hex, 'hex'), expected);
  });
}
```

**Resposta esperada do cliente.** Qualquer 2xx em até 10 segundos confirma a entrega. Sem 2xx, o envio entra no retry da seção 5.3. Como pode haver duplicatas, o cliente deduplica por `X-Event-Id` ([09:25] Diego), o que também o protege de uma requisição capturada e reenviada por terceiros.

## 7. Matriz de erros

Códigos com prefixo `WEBHOOK_` ([09:29] Larissa), em classes que estendem `AppError` ou as classes de `src/shared/errors/http-errors.ts` ([09:28] Bruno). Dos exemplos citados por Bruno, `WEBHOOK_NOT_FOUND` e `WEBHOOK_INVALID_URL` entram; `WEBHOOK_SECRET_REQUIRED` não tem uso, porque a secret é sempre gerada pela plataforma ([09:31] Marcos) e o cliente nunca a envia. Os erros de API chegam ao cliente pelo `errorMiddleware` sem nenhuma mudança nele; os de entrega ficam em `webhook_deliveries.errorCode`, `webhook_outbox.lastErrorCode` e `webhook_dead_letter.reason`.

| Código | Onde | HTTP | Quando | Comportamento |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | API | 404 | `:id` de webhook inexistente | Classe `WebhookNotFoundError extends AppError`. Não usa `NotFoundError`, cujo código é fixo em `NOT_FOUND` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | API | 404 | `customerId` inexistente na criação | `WebhookCustomerNotFoundError extends AppError` |
| `WEBHOOK_INVALID_URL` | API | 400 | URL sem `https:` ou malformada ([09:23] Sofia) | A regra fica no schema Zod. Como o `validate` transforma todo `ZodError` em `VALIDATION_ERROR`, o código vai em `details[].message` com `path: "url"` |
| `WEBHOOK_EMPTY_UPDATE` | API | 400 | `PATCH` sem nenhum campo | Também por refinement do Zod, em `details` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | API | 404 | Replay de `:id` inexistente na DLQ | `extends AppError` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | API | 409 | Replay de item já reprocessado | `extends ConflictError` |
| `WEBHOOK_ROTATION_IN_PROGRESS` | API | 409 | Nova rotação antes de terminar a janela de 24 h da anterior | `extends ConflictError` |
| `WEBHOOK_OUTBOX_WRITE_FAILED` | API (`PATCH /orders/:id/status`) | 500 | Falha ao gravar a outbox dentro de `changeStatus` | Transação desfeita, status não muda ([09:40] Bruno) |
| `WEBHOOK_DELIVERY_TIMEOUT` | Worker | n/a | Sem resposta em 10 s ([09:42] Diego) | Retry |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Worker | n/a | Resposta fora de 2xx, inclusive redirecionamento | Retry |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | Worker | n/a | DNS, conexão recusada, erro de TLS | Retry |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Worker | n/a | Corpo acima de 64 KB ([09:24] Larissa) | DLQ direto, sem envio e sem retry |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | Worker | n/a | 6º envio falhou | DLQ, com o último erro registrado em `webhook_deliveries` |

Erros genéricos continuam com os códigos atuais: `UNAUTHORIZED` (401) do `authenticate`, `FORBIDDEN` (403) do `requireRole` e `VALIDATION_ERROR` (400) do `validate`.

**Por que o limite de 64 KB é checado no worker e não na inserção.** Se a checagem lançasse erro dentro de `changeStatus`, ela desfaria a mudança de status do pedido. Checar no worker mantém a decisão "erro, não truncar" ([09:23] Sofia, [09:24] Larissa) sem deixar o pedido refém do tamanho do payload. Na prática, com o payload enxuto de [09:43], o limite não deve ser atingido ([09:24] Diego).

## 8. Estratégias de resiliência

| Situação | Estratégia |
|---|---|
| Cliente lento | Timeout de 10 s por chamada, contado como falha ([09:42] Diego) |
| Cliente fora do ar | Backoff de 1 min, 5 min, 30 min, 2 h e 12 h e depois DLQ ([09:17] Diego) |
| Falha permanente | DLQ em tabela separada e replay manual por `ADMIN` ([09:18] Diego) |
| Worker cai no meio do envio | Na partida, linhas em `PROCESSING` voltam a `PENDING`; o cliente pode receber de novo e deduplica por `X-Event-Id` ([09:24] Diego) |
| API reinicia | Não afeta a entrega, porque o worker é outro processo ([09:11] Diego) |
| Falha ao gravar a outbox | Rollback da mudança de status; nada fica pela metade ([09:40] Bruno) |
| Banco indisponível para o worker | O ciclo falha, registra `webhook_worker_poll_failed` e tenta de novo no próximo intervalo; nenhum evento é perdido, porque continua na outbox |
| Webhook pausado pelo cliente | Eventos já enfileirados ficam pendentes e não são enviados até a reativação; mudanças de status durante a pausa não geram evento |
| Encerramento do worker | `SIGINT`/`SIGTERM` param novos ciclos, esperam os envios em andamento terminarem (no máximo 10 s) e chamam `prisma.$disconnect()`. O tratamento de sinais e o `$disconnect` seguem `src/server.ts`; a espera pelo lote é novidade do worker |

Não há fallback por outro canal: o e-mail foi adiado ([09:37] Larissa). Também não há rate limiting de saída ([09:39] Larissa).

## 9. Observabilidade

O projeto hoje tem só logs estruturados com Pino (`src/shared/logger/index.ts`) e o `X-Request-Id` gerado em `src/middlewares/request-logger.middleware.ts`. Não tem biblioteca de métricas nem de tracing, e a reunião decidiu não adicionar ferramentas novas ([09:29] Bruno, [09:30] Larissa). A observabilidade da feature se apoia nisso.

**Logs (Pino).** Todos com `eventId`, `webhookId`, `orderId` e `outboxId`, quando existirem.

| Mensagem | Processo | Campos extras |
|---|---|---|
| `webhook_event_enqueued` | API | `toStatus`, `webhookCount` |
| `webhook_delivery_succeeded` | Worker | `attempt`, `responseStatus`, `durationMs` |
| `webhook_delivery_failed` | Worker | `attempt`, `errorCode`, `responseStatus`, `nextAttemptAt` |
| `webhook_dead_lettered` | Worker | `deadLetterId`, `reason`, `attempts` |
| `webhook_dead_letter_replayed` | API | `deadLetterId`, `userId` (auditoria, [09:36] Sofia) |
| `webhook_worker_started` / `webhook_worker_stopped` | Worker | `pollIntervalMs`, `batchSize`, `recoveredProcessing` |
| `webhook_worker_poll_failed` | Worker | `err` |
| `webhook_worker_cycle` | Worker | `picked`, `delivered`, `failed`, `deadLettered`, `oldestPendingAgeMs` |
| `webhook_outbox_write_failed` | API | `err`, `orderId` |

`secret`, `previousSecret` e `X-Signature` nunca entram em log. Para garantir isso mesmo em caso de engano, entram nos `redactPaths` de `src/shared/logger/index.ts`, ao lado dos atuais `*.password` e `*.token`: `secret`, `previousSecret`, `*.secret`, `*.previousSecret`, `*.*.secret` e `*.*.previousSecret`. No Pino, `*.secret` cobre só um nível de aninhamento, daí os caminhos de topo e de dois níveis. O motivo é o histórico de vazamento de secret por log citado em [09:22] Diego.

**Métricas.** Sem stack de métricas, os indicadores saem das tabelas e dos logs. O worker escreve, a cada ciclo com trabalho, o log `webhook_worker_cycle` da tabela acima. As consultas de referência para acompanhar a feature são:

| Indicador | Como medir | Uso |
|---|---|---|
| Latência de entrega | `deliveredAt - createdAt` em `webhook_outbox`, só linhas entregues com `attempts = 1`: máximo e fração abaixo de 10 s | Verificar a meta de menos de 10 s ([09:02] Marcos) |
| Atraso da fila | idade do `PENDING` mais antigo cujo `nextAttemptAt` já passou (comparando com `new Date()` em UTC) | Detectar worker parado ou lento |
| Taxa de sucesso na primeira tentativa | `webhook_deliveries` com `attempt = 1` | Saúde dos endpoints dos clientes |
| Volume na DLQ | linhas novas em `webhook_dead_letter` por dia | Acionar replay e contato com o cliente |
| Rajadas por cliente | envios por `webhookId` por minuto | Dado para a decisão adiada de rate limiting ([09:39] Larissa) |

**Tracing.** Não há tracing distribuído no projeto, e esta feature não introduz um. A correlação ponta a ponta usa o `eventId`: ele está na outbox, em cada linha de `webhook_deliveries`, em todos os logs do worker e no header `X-Event-Id` que o cliente recebe. Para ir do pedido ao envio, o elo é `orderId`, presente em `order_status_history` e na outbox.

## 10. Segurança

- HMAC-SHA256 sobre o corpo, com secret por endpoint ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- Secret gerada com `crypto.randomBytes(32)`, devolvida só na criação e na rotação, nunca listada nem logada.
- Só `https` no cadastro ([09:23] Sofia); o worker não segue redirecionamentos, para que um 3xx não leve a requisição para outro destino.
- Replay restrito a `ADMIN` e auditado ([09:36] Sofia).
- Pontos para a revisão de segurança de dois dias úteis pedida por Sofia ([09:46] Sofia): armazenamento da secret, geração da secret, implementação do HMAC e o acesso de qualquer papel autenticado ao CRUD de webhooks de qualquer cliente ([09:37] Sofia).

## 11. Integração com o sistema existente

| Arquivo existente | Como o módulo de webhooks se integra |
|---|---|
| `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `publishWebhookEvent(tx, order, from, to)` depois de `tx.orderStatusHistory.create`, dentro do mesmo `$transaction` ([09:40] Bruno, [09:41] Bruno). O primeiro parâmetro é tipado como `Prisma.TransactionClient`, o mesmo tipo que o arquivo usa no alias local `TxClient` (não exportado). O objeto `order` carregado no início da transação já tem `orderNumber`, `customerId` e `totalCents`, então o publisher não faz leitura extra de pedido. `create` e `delete` não mudam |
| `src/modules/orders/order.status.ts` | Não muda. Como `canTransition` é avaliado antes, só transições válidas chegam ao publisher. A lista de valores aceitos em `statuses` é o enum `OrderStatus` de `prisma/schema.prisma`, o mesmo usado em `updateOrderStatusSchema` |
| `prisma/schema.prisma` | Ganha o enum `WebhookOutboxStatus`, os models `Webhook`, `WebhookOutbox`, `WebhookDelivery` e `WebhookDeadLetter` e a relação `webhooks` em `Customer`. A migration nova entra em `prisma/migrations/` |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | As classes `Webhook*Error` entram em `http-errors.ts`, no mesmo lugar de `InsufficientStockError` e `InvalidStatusTransitionError` ([09:28] Bruno), e são exportadas por `src/shared/errors/index.ts`. Estendem `AppError` quando o código precisa ser próprio (404 `WEBHOOK_NOT_FOUND`, já que `NotFoundError` fixa `NOT_FOUND`) ou `ConflictError` (409) |
| `src/middlewares/error.middleware.ts` | Nenhuma mudança: o `instanceof AppError` já serializa os erros novos no envelope atual ([09:29] Bruno) |
| `src/middlewares/validate.middleware.ts` | Os schemas de `src/modules/webhooks/webhook.schemas.ts` são aplicados com `validate({ body, params, query })`, igual a `src/modules/orders/order.routes.ts` |
| `src/middlewares/auth.middleware.ts` | `router.use(authenticate)` nas rotas de webhook; `requireRole('ADMIN')` na rota de replay ([09:36] Larissa) |
| `src/routes/index.ts` | `buildApiRouter` ganha `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`, `router.use('/customers/:customerId/webhooks', buildCustomerWebhookRouter(controllers.webhooks))` (com `Router({ mergeParams: true })`) e `router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks))`; o tipo `Controllers` ganha `webhooks`. O router de clientes existente não muda: `/:id/webhooks` não casa com nenhuma rota dele |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController` com o mesmo `prisma`, no mesmo padrão dos outros módulos |
| `src/server.ts` | Serve de modelo para o novo `src/worker.ts`: bootstrap, log de início, tratamento de `SIGINT`/`SIGTERM` e `prisma.$disconnect()` ([09:11] Larissa) |
| `src/config/database.ts` | O worker usa `createPrismaClient()` para ter a própria instância de `PrismaClient` com a mesma `DATABASE_URL` ([09:30] Bruno) |
| `src/config/env.ts` | O schema ganha `WEBHOOK_POLL_INTERVAL_MS` (2000), `WEBHOOK_HTTP_TIMEOUT_MS` (10000), `WEBHOOK_BATCH_SIZE` (10) e `WEBHOOK_MAX_PAYLOAD_BYTES` (65536), com esses valores como default |
| `src/shared/logger/index.ts` | Logger reaproveitado nos dois processos; `redactPaths` ganha `secret`, `previousSecret`, `*.secret`, `*.previousSecret`, `*.*.secret` e `*.*.previousSecret` (seção 9) |
| `package.json` | Scripts `"worker": "tsx --env-file=.env src/worker.ts"` ([09:11] Larissa) e `"start:worker": "node --env-file=.env dist/worker.js"`. O `tsconfig.build.json` já compila tudo em `src/**/*.ts`, então `dist/worker.js` sai no build atual |
| `tests/setup.ts` e `tests/helpers/factories.ts` | O `beforeEach` passa a limpar as tabelas novas antes de `customer`; `factories.ts` ganha `createTestWebhook` |

Estrutura do módulo novo, no padrão de `src/modules/orders/` ([09:27] Bruno):

```
src/modules/webhooks/
├── webhook.controller.ts
├── webhook.service.ts        # CRUD, rotação, histórico, replay
├── webhook.repository.ts
├── webhook.routes.ts         # buildWebhookRouter, buildCustomerWebhookRouter, buildWebhookAdminRouter
├── webhook.schemas.ts        # Zod: criação, edição, params
├── webhook.publisher.ts      # publishWebhookEvent(tx, order, fromStatus, toStatus)
├── webhook.processor.ts      # loop do worker, envio, retry, DLQ
└── webhook.signature.ts      # HMAC e montagem do header X-Signature
src/worker.ts                 # entry point do processo separado
```

## 12. Dependências e compatibilidade

- **Sem dependências novas de runtime.** HTTP com `fetch` nativo (Node 20, exigido em `package.json`); HMAC e geração de secret com `node:crypto`; UUID com o pacote `uuid` já instalado.
- **Banco.** MySQL 8.0 (`docker-compose.yml`) e Prisma 5.22.0. A consulta do worker usa `NOT EXISTS`, suportado pela versão.
- **Compatibilidade da API.** Rotas novas apenas. O contrato de `PATCH /api/v1/orders/:id/status` não muda, exceto pelo novo erro 500 `WEBHOOK_OUTBOX_WRITE_FAILED` quando a outbox não pode ser gravada.
- **Implantação.** Um processo novo (`npm run start:worker`), com exatamente uma instância ([09:12] Diego). A migration precisa rodar antes de subir API e worker.
- **Variáveis de ambiente.** As quatro novas têm default, então ambientes existentes continuam válidos sem mudança no `.env`.

## 13. Critérios de aceite técnicos

1. Mudar o status de um pedido cujo cliente tem um webhook ativo com aquele status no filtro cria exatamente uma linha `PENDING` na outbox, na mesma transação.
2. Se a gravação da outbox falhar, a mudança de status é desfeita: `orders.status` e `order_status_history` ficam como antes.
3. Mudança de status sem webhook interessado não grava nada na outbox.
4. Com o endpoint do cliente respondendo 200, a requisição chega em menos de 10 s, com os cinco headers da seção 6.2, e a assinatura confere com a secret do webhook.
5. Com o endpoint respondendo 500, os envios seguem os intervalos da seção 5.3 e, depois do 6º, a linha vai para `FAILED` e aparece em `webhook_dead_letter` com `reason = WEBHOOK_MAX_ATTEMPTS_EXCEEDED`.
6. Endpoint que demora mais de 10 s gera `WEBHOOK_DELIVERY_TIMEOUT` e entra no retry.
7. Cadastro com URL `http://` é recusado com 400 e `WEBHOOK_INVALID_URL` em `details`.
8. Replay por usuário `OPERATOR` recebe 403; por `ADMIN`, recebe 202, a linha volta para `PENDING` e o log de auditoria registra o `userId`.
9. Depois de rotacionar, os envios das 24 horas seguintes têm duas assinaturas, e os posteriores só uma. Uma segunda rotação dentro das 24 horas recebe 409 `WEBHOOK_ROTATION_IN_PROGRESS`.
10. Se a regra de ordem do RFC (questão 6) for confirmada: eventos do mesmo pedido para o mesmo webhook são entregues na ordem em que foram criados, mesmo com o primeiro em retry. Um endpoint que leva 10 s para responder não atrasa a entrega para outros webhooks.
11. Eventos de um webhook pausado (`active: false`) não são enviados nem ocupam o lote, e voltam a ser enviados depois da reativação.
12. Matar o worker durante um envio e subir de novo reenvia o evento com o mesmo `X-Event-Id`.
13. `GET /webhooks/:id/deliveries` devolve no máximo 100 envios, do mais recente para o mais antigo.
14. `npm test`, `npm run lint` e `npm run build` passam.

Os testes seguem a stack atual (`vitest` e `supertest`, com `tests/helpers/factories.ts`). Para o worker, um servidor HTTP local criado com `node:http` dentro do teste faz o papel do cliente e registra as requisições recebidas; o relógio do backoff é injetado no processador para que os intervalos possam ser simulados sem esperar.

## 14. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| A transação de `changeStatus` fica mais lenta com a leitura de webhooks e a gravação da outbox | Média | Médio | Uma consulta indexada por `(customerId, active)` e um único `createMany` |
| Subirem duas instâncias do worker | Baixa | Alto: envios em dobro e ordem quebrada | Restrição explícita no deploy; log `webhook_worker_started` permite detectar duas instâncias |
| Endpoint fora do ar segura os eventos seguintes do mesmo pedido | Média | Médio: atraso de até cerca de 15 h para aquele pedido | Comportamento documentado; o evento travado vai para a DLQ e libera os seguintes |
| Crescimento contínuo da outbox e de `webhook_deliveries` | Alta | Baixo no curto prazo | Índices adequados; arquivamento registrado como trabalho futuro ([09:08] Diego) |
| Secret em texto no banco | Média | Alto | Acesso restrito ao banco, redact no log e revisão de Sofia antes do deploy ([09:46] Sofia) |
| Qualquer usuário autenticado gerencia webhooks de qualquer cliente | Média | Médio | Aceito "por enquanto" ([09:37] Sofia); questão aberta no RFC |
| Interpretação errada das "5 tentativas" | Baixa | Baixo | Constante única `WEBHOOK_RETRY_SCHEDULE_MS`, trivial de ajustar depois da confirmação |
