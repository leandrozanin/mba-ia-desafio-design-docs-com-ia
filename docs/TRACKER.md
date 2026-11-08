# Tracker de Rastreabilidade

Cada linha liga um item registrado nos documentos à sua origem: a transcrição da reunião (`TRANSCRICAO.md`, formato `[hh:mm] Nome`) ou um arquivo do código base. Quando um item tem mais de uma fonte, a tabela registra a principal; as demais aparecem no próprio documento.

Itens marcados nos documentos como **proposta de implementação** (decisões de detalhe que a reunião não tomou) estão ligados ao trecho da reunião ou do código que os motivou e aparecem nas questões em aberto do RFC quando afetam contrato ou comportamento.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| PRD-CTX-01 | docs/PRD.md | Contexto | Atlas, MaxDistribuição e Nova Cargo pediram notificação de mudança de status; hoje fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | Risco de a Atlas migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PUB-01 | docs/PRD.md | Público-alvo | Configuração feita por usuários do OMS que representam o cliente, via API com JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público-alvo | Administradores (role ADMIN) reprocessam a DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-03 | docs/PRD.md | Cenário | Cliente com indisponibilidade de 2 h em manutenção planejada | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário | Cliente que vazou secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Primeira tentativa < 10 s; p95 identificado como indicador proposto | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Zero mudanças de status sem evento correspondente | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Meta proposta de adoção por 3 clientes; só a Atlas citou fim de novembro | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Janela de ~15 h coberta por retentativas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Redução do polling em GET /orders (sem meta numérica definida) | TRANSCRICAO | [09:00] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de alerta ao cliente adiado para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual descartado; só endpoints | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de envio adiado ("observar e decidir depois") | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks inbound descartados; apenas outbound | TRANSCRICAO | [09:03] Sofia |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Ordenação global não é requisito dos clientes | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (~30 dias) fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Itens do pedido fora do payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (POST) com URL e status; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Evento order.status_changed a cada mudança de status | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Campos do payload enxuto, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Assinatura HMAC no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-FR-09b | docs/PRD.md | Requisito Funcional | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retentativas em 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay manual da DLQ por ADMIN com registro de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Rotação de secret com validade paralela de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas (GET /webhooks/:id/deliveries) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Somente URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Mudança de status e evento atômicos | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não afeta mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Ordenação só por pedido e com um worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB, erro se exceder | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Identificadores UUID | TRANSCRICAO | [09:51] Larissa |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança com 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Portal do desenvolvedor documentando integração e deduplicação | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Gatilho é a rota PATCH /orders/:id/status existente | CODIGO | src/modules/orders/order.routes.ts |
| PRD-DEP-06 | docs/PRD.md | Dependência | Prazo a confirmar com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-RSK-01 | docs/PRD.md | Risco | Estouro do prazo; estimativa de 3 sprints com revisão incluída | TRANSCRICAO | [09:46] Larissa |
| PRD-RSK-02 | docs/PRD.md | Risco | Cliente não deduplicar eventos | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-04 | docs/PRD.md | Risco | Atraso com um único worker e clientes lentos | TRANSCRICAO | [09:12] Diego |
| PRD-RSK-05 | docs/PRD.md | Risco | Rajada de chamadas para o cliente | TRANSCRICAO | [09:38] Diego |
| PRD-RSK-06 | docs/PRD.md | Risco | Cliente fora por mais de ~15 h | TRANSCRICAO | [09:17] Marcos |
| PRD-TST-01 | docs/PRD.md | Estratégia de teste | Testes Vitest + Supertest contra MySQL real, no padrão de tests/*.test.ts | CODIGO | vitest.config.ts |
| RFC-META-01 | docs/RFC.md | Metadado | Larissa abre o documento de design; revisão com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | changeStatus é uma transação que atualiza orders, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service de orders | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / infra nova | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do banco para notificar o worker | TRANSCRICAO | [09:09] Bruno |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Worker dentro do processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido / apenas 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-OQ-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída por cliente | TRANSCRICAO | [09:39] Diego |
| RFC-OQ-02 | docs/RFC.md | Questão em aberto | customer_id no body ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-OQ-03 | docs/RFC.md | Questão em aberto | Ordenação durante retries | TRANSCRICAO | [09:12] Diego |
| RFC-OQ-04 | docs/RFC.md | Questão em aberto | Formato do X-Signature durante convivência de secrets | TRANSCRICAO | [09:21] Sofia |
| RFC-OQ-05 | docs/RFC.md | Questão em aberto | 5 tentativas vs. 1 envio + 5 reenvios | TRANSCRICAO | [09:48] Larissa |
| RFC-OQ-06 | docs/RFC.md | Questão em aberto | Proteção da secret em repouso (revisão de segurança) | TRANSCRICAO | [09:46] Sofia |
| RFC-OQ-07 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-OQ-08 | docs/RFC.md | Questão em aberto | Descoberta do id da DLQ (sem listagem definida) | TRANSCRICAO | [09:18] Diego |
| RFC-IMP-01 | docs/RFC.md | Impacto | Envio via fetch nativo; Node ≥ 20 já exigido | CODIGO | package.json |
| RFC-PRZ-01 | docs/RFC.md | Restrição | Prazo de três sprints | TRANSCRICAO | [09:47] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL na mesma transação de changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Outbox não pode falhar sem rollback do status | TRANSCRICAO | [09:40] Bruno |
| ADR-001-IDX | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices em status e created_at; estados pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| ADR-001-FN | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | publishWebhookEvent(tx, order, from, to) recebendo o tx | TRANSCRICAO | [09:41] Bruno |
| ADR-001-COD | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Transação Prisma de changeStatus no OrderService | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Polling a cada 2 s | TRANSCRICAO | [09:09] Diego |
| ADR-002-PROC | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | src/worker.ts e npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-FILE | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Lógica em src/modules/webhooks/webhook.worker.ts | TRANSCRICAO | [09:28] Bruno |
| ADR-002-PRISMA | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-ORD | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Single-worker; ordenação por order_id | TRANSCRICAO | [09:12] Diego |
| ADR-002-ESC | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa adiada | Escalar por partição de order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| ADR-002-SRV | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | Entry-point único atual com tratamento de SIGINT/SIGTERM | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | 3 tentativas | TRANSCRICAO | [09:16] Diego |
| ADR-003-DLQ | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Tabela webhook_dead_letter separada (vs. status failed na outbox) | TRANSCRICAO | [09:18] Diego |
| ADR-003-15H | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | ~15 h de atraso aceitas pelo produto | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALG | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | SHA-256 como algoritmo | TRANSCRICAO | [09:20] Sofia |
| ADR-004-TBL | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Configuração guarda url, secret, customer_id e ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-REV | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Revisão de HMAC e geração de secret pela Sofia | TRANSCRICAO | [09:46] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ID | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | event_id UUID gerado ao entrar na outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar fica com o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks no padrão controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| ADR-006-PFX | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Prefixo WEBHOOK_ para todo o módulo | TRANSCRICAO | [09:29] Larissa |
| ADR-006-LOG | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Sem logger novo; error middleware sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-C1 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Classe base AppError com statusCode/errorCode/details | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-C2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | InsufficientStockError e InvalidStatusTransitionError com códigos UPPER_SNAKE_CASE | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-C3 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Middleware central trata AppError, ZodError e Prisma P2002/P2025 | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-C4 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Logger Pino com redact | CODIGO | src/shared/logger/index.ts |
| ADR-006-C5 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | requireRole('ADMIN') já usado em rota de usuários | CODIGO | src/modules/users/user.routes.ts |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-FLT | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Filtro de status aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-007-ALT | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Alternativa descartada | Renderizar no envio a partir do order_id | TRANSCRICAO | [09:51] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Primeira tentativa < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-DM-01 | docs/FDD.md | Decisão | Ids Char(36) UUID e @@map snake_case como no schema existente | CODIGO | prisma/schema.prisma |
| FDD-DM-02 | docs/FDD.md | Decisão | Linha de outbox por evento × endpoint; X-Webhook-Id distingue cadastros (proposta) | TRANSCRICAO | [09:44] Sofia |
| FDD-DM-03 | docs/FDD.md | Decisão | Payload armazenado como texto para preservar os bytes assinados (proposta) | TRANSCRICAO | [09:22] Sofia |
| FDD-DM-04 | docs/FDD.md | Restrição | events como Json: conector MySQL do Prisma sem listas escalares | CODIGO | prisma/schema.prisma |
| FDD-FLX-01 | docs/FDD.md | Fluxo | publishWebhookEvent chamado após orderStatusHistory.create na transação | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-01b | docs/FDD.md | Fluxo | Snapshot usa orderNumber, customerId e totalCents do model Order | CODIGO | prisma/schema.prisma |
| FDD-FLX-01c | docs/FDD.md | Fluxo | Payload > 64 KB não é enviado e vai para a DLQ (proposta) | TRANSCRICAO | [09:23] Sofia |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos em lote pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-02b | docs/FDD.md | Fluxo | Bloqueio por (webhookId, orderId) para preservar ordem durante retries (proposta) | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-02c | docs/FDD.md | Fluxo | Worker usa createPrismaClient() próprio | CODIGO | src/config/database.ts |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Constantes de retry 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo | DLQ grava payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente, mesmo event_id | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-06 | docs/FDD.md | Fluxo | Rotação: antiga válida 24 h em paralelo | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks (customerId, url, events) → 201 com secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-01b | docs/FDD.md | Contrato | Prefixo /api/v1 das rotas | CODIGO | src/app.ts |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId= | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id → 204, como DELETE de orders | CODIGO | src/modules/orders/order.controller.ts |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/secret/rotate | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries (até 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06b | docs/FDD.md | Contrato | Paginação via paginated() | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Requisição outbound com Content-Type, X-Event-Id, X-Signature, X-Timestamp | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | customer_id não vem do JWT (JWT é do operador) | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Envelope de erro { error: { code, message, details } } | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERR-01 | docs/FDD.md | Erro | Erros WEBHOOK_* estendem AppError e ficam em http-errors.ts | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL sai como VALIDATION_ERROR com details (limitação do validate) | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERR-03 | docs/FDD.md | Erro | 401/403 reaproveitados de authenticate/requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT após 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE acima de 64 KB | TRANSCRICAO | [09:24] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Reinício da API não afeta entregas | TRANSCRICAO | [09:11] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Fallback: DLQ + replay; sem e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Shutdown gracioso no padrão de server.ts | CODIGO | src/server.ts |
| FDD-RES-04 | docs/FDD.md | Resiliência | Variáveis WEBHOOK_* validadas no envSchema | CODIGO | src/config/env.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados Pino; nenhum logger novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | requestId por requisição já existente | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log de auditoria do replay com replayedBy | TRANSCRICAO | [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus estendido com publishWebhookEvent | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Status de interesse são os valores de OrderStatus da máquina de estados | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novas rotas registradas no buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Controllers instanciados em buildControllers | CODIGO | src/app.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Limpeza das novas tabelas no beforeEach dos testes | CODIGO | tests/setup.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Exportação das novas classes de erro | CODIGO | src/shared/errors/index.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | MySQL 8.0 (SKIP LOCKED) | CODIGO | docker-compose.yml |
| FDD-RISK-03 | docs/FDD.md | Risco | Single-worker como gargalo com timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-10 | docs/FDD.md | Risco | Rajada de chamadas ao cliente (rate limiting adiado) | TRANSCRICAO | [09:39] Larissa |
| FDD-RISK-11 | docs/FDD.md | Risco | Prazo de fim de novembro | TRANSCRICAO | [09:45] Marcos |

| PRD-RSK-07 | docs/PRD.md | Risco / Segurança | Falta de autorização por customer pode expor configurações e histórico entre customers | CODIGO | prisma/schema.prisma |
| PRD-RSK-08 | docs/PRD.md | Risco / Segurança | URL configurável exige decisão de política de destino/egress | TRANSCRICAO | [09:31] Marcos |
| RFC-OQ-09 | docs/RFC.md | Questão em aberto / Segurança | Definir autorização por customer; User atual não possui vínculo com Customer | CODIGO | prisma/schema.prisma |
| RFC-OQ-10 | docs/RFC.md | Questão em aberto / Segurança | Política de destino para URLs configuráveis, DNS, rede privada e redirecionamentos | TRANSCRICAO | [09:31] Marcos |
| RFC-OQ-11 | docs/RFC.md | Questão em aberto | Semântica de DELETE para outbox pendente e histórico | TRANSCRICAO | [09:33] Bruno |
| RFC-OQ-12 | docs/RFC.md | Questão em aberto | Ordenação quando o evento anterior está na DLQ e é reproduzido depois | TRANSCRICAO | [09:12] Diego |
| FDD-AUTH-01 | docs/FDD.md | Restrição / Segurança | Autenticação existente não representa autorização por customer | CODIGO | prisma/schema.prisma |
| FDD-SEC-01 | docs/FDD.md | Risco / Proposta | HTTPS não define política de egress para destinos internos | TRANSCRICAO | [09:31] Marcos |
| FDD-DEL-01 | docs/FDD.md | Proposta | Preservar outbox/histórico ao remover endpoint; semântica pendente | TRANSCRICAO | [09:33] Bruno |
| FDD-ORD-01 | docs/FDD.md | Restrição | Ordenação em retry não cobre replay após DLQ | TRANSCRICAO | [09:12] Diego |

| PRD-DEP-03 | docs/PRD.md | Dependência | Usuários do OMS que representam cada cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Operação do novo processo worker junto com a API | TRANSCRICAO | [09:11] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de aceitação | Usuário autenticado cadastra webhook HTTPS e recebe secret; HTTP recusado | TRANSCRICAO | [09:31] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de aceitação | Evento chega em menos de 10 s com headers e assinatura válidos | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-03 | docs/PRD.md | Critério de aceitação | Status fora do filtro não gera chamada | TRANSCRICAO | [09:33] Marcos |
| PRD-CA-04 | docs/PRD.md | Critério de aceitação | Falha aciona retries e DLQ com payload e motivo | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-05 | docs/PRD.md | Critério de aceitação | ADMIN reproduz DLQ com o mesmo event id; não ADMIN recebe bloqueio | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-06 | docs/PRD.md | Critério de aceitação | Rotação mantém secret anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de aceitação | Histórico mostra as últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-08 | docs/PRD.md | Critério de aceitação | Falha de mudança não envia evento; sucesso registra outbox | TRANSCRICAO | [09:40] Bruno |
| PRD-CA-09 | docs/PRD.md | Critério de aceitação | Usuário lista, edita e remove webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Outbox na mesma transação e rollback se inserir falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Entrega at-least-once com event id estável | TRANSCRICAO | [09:25] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Backoff e DLQ com replay ADMIN | TRANSCRICAO | [09:18] Diego |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Assinatura HMAC e rotação de secret | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Sem infraestrutura nova e reuso dos padrões | TRANSCRICAO | [09:30] Larissa |
| FDD-AC-T01 | docs/FDD.md | Critério técnico | Um evento de outbox por endpoint ativo interessado | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-T02 | docs/FDD.md | Critério técnico | Status sem endpoint interessado não cria outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-T03 | docs/FDD.md | Critério técnico | Falha de inserção reverte status e histórico | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-T04 | docs/FDD.md | Critério técnico | Entrega 2xx ocorre abaixo de 10 s | TRANSCRICAO | [09:10] Larissa |
| FDD-AC-T05 | docs/FDD.md | Critério técnico | Request outbound inclui headers e HMAC do corpo | TRANSCRICAO | [09:44] Diego |
| FDD-AC-T06 | docs/FDD.md | Critério técnico | Timeout de 10 s reagenda retry em 1 min | TRANSCRICAO | [09:42] Diego |
| FDD-AC-T07 | docs/FDD.md | Critério técnico | Retries seguem a progressão e falha final vai à DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-AC-T08 | docs/FDD.md | Critério técnico | Replay ADMIN preserva eventId e registra ator | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-T09 | docs/FDD.md | Critério técnico | URL HTTP retorna falha de validação | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-T10 | docs/FDD.md | Critério técnico | Secret só aparece na criação e rotação; não em logs | TRANSCRICAO | [09:29] Bruno |
| FDD-AC-T11 | docs/FDD.md | Critério técnico | Rotacionação convive por 24 h e depois usa só nova | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-T12 | docs/FDD.md | Critério técnico | Histórico retorna até 100 entregas recentes | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-T13 | docs/FDD.md | Critério técnico | Bloqueia posterior durante retry; exceção DLQ/replay fica aberta | TRANSCRICAO | [09:12] Diego |
| FDD-AC-T14 | docs/FDD.md | Critério técnico | Reiniciar API não para worker em processo separado | TRANSCRICAO | [09:11] Diego |
| FDD-AC-T15 | docs/FDD.md | Critério técnico | Suíte baseada no Vitest e setup existente | CODIGO | vitest.config.ts |
| FDD-RISK-01 | docs/FDD.md | Risco | Cliente pode processar evento duplicado sem deduplicar | TRANSCRICAO | [09:25] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Crescimento da outbox sem arquivamento na fase atual | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Inversão por pedido durante retries | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Vazamento da secret no lado do cliente | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Secret recuperável precisa de proteção em repouso | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-07 | docs/FDD.md | Risco | Queda do worker entre envio e marcação | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-08 | docs/FDD.md | Risco | Escopo de customer sem vínculo no User atual | TRANSCRICAO | [09:32] Bruno |
| FDD-RISK-09 | docs/FDD.md | Risco | URL configurável precisa de política de egress | TRANSCRICAO | [09:31] Marcos |
## Resumo de rastreabilidade

| Métrica | Valor |
|---------|-------|
| Linhas no tracker | 212 |
| Linhas com Fonte = TRANSCRICAO | 176 (83,0%) |
| Linhas com Fonte = CODIGO | 36 |
| Documentos cobertos | PRD, RFC, FDD e ADR-001 a ADR-007 |
| Nota sobre cobertura | Cobertura dos itens explicitamente identificados por ID nas tabelas do PRD, RFC e FDD: 106/106 (100%). Proporção da origem das linhas do tracker: 176/212 (83,0%) com fonte na transcrição. A contagem por ID não considera cada afirmação narrativa como item separado; propostas continuam identificadas como propostas. |
