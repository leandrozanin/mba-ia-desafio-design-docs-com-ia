# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|-------|-------|
| **RFC** | RFC-001 |
| **Autor** | Larissa (Tech Lead) — redação a partir da reunião técnica ("Eu vou abrir o doc de design da feature", [09:50] Larissa) |
| **Status** | Em revisão |
| **Data** | 2026-11-08 (data desta versão do RFC; a data da reunião não consta na transcrição) |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. TL;DR

Propomos notificar clientes B2B sobre mudanças de status de pedidos por **webhooks outbound** assinados com **HMAC-SHA256**. O evento é gravado numa **outbox no próprio MySQL, na mesma transação** que muda o status do pedido; um **worker em processo separado** lê a outbox **a cada 2 s** e entrega com **retry exponencial (1m/5m/30m/2h/12h)**, movendo falhas permanentes para uma **DLQ** com replay manual por ADMIN. A garantia é **at-least-once**, com deduplicação no cliente via **`X-Event-Id`**. Tudo é construído como um módulo a mais (`src/modules/webhooks`), reusando `AppError`, Pino e os middlewares existentes. Nenhuma infraestrutura nova.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para serem notificados quando o status dos seus pedidos muda. Hoje eles consultam `GET /orders` periodicamente, o que deixa a integração "lenta e cara"; a Atlas sinalizou possível migração para um concorrente se nada for entregue até o fim do trimestre ([09:00] Marcos). Para os clientes, "tempo real" é **menos de 10 segundos** ([09:02] Marcos). O fluxo é só de saída: nós enviamos, eles recebem ([09:02] Marcos, [09:03] Sofia).

O OMS atual não tem nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status está concentrada em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), uma transação Prisma que atualiza `orders`, grava `order_status_history` e mexe no estoque ([09:04] Bruno). Qualquer solução precisa (a) não deixar essa transação dependente da rede de terceiros e (b) nunca permitir que o status mude sem que o evento correspondente seja emitido ([09:40] Bruno).

## 3. Proposta técnica

```
PATCH /orders/:id/status
        │
        ▼
OrderService.changeStatus ── $transaction ────────────────────────────┐
  update orders · insert order_status_history · estoque               │
  publishWebhookEvent(tx, order, from, to)  →  INSERT webhook_outbox  │  (commit atômico)
        └─────────────────────────────────────────────────────────────┘
                                   │
                       (processo separado: npm run worker)
                                   ▼
          WebhookWorker — polling 2 s, lote pequeno, em ordem de created_at
                 │  POST https://cliente  (X-Signature HMAC-SHA256, X-Event-Id, ...)
        ┌────────┴─────────┐
      2xx → entregue    falha/timeout 10 s → reagenda 1m/5m/30m/2h/12h
                                   │ esgotou
                                   ▼
                        webhook_dead_letter ── POST /admin/webhooks/dead-letter/:id/replay (ADMIN)
```

**Componentes**

1. **Outbox transacional** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) — `changeStatus` chama `publishWebhookEvent(tx, …)` com o client da transação corrente. Só são inseridas linhas para webhooks ativos do customer que escutam o status de destino, e o payload é gravado já renderizado ([ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)).
2. **Worker** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)) — novo entry-point `src/worker.ts` (arquivo a criar; `npm run worker`), mesmo banco, `PrismaClient` próprio, instância única. Timeout de 10 s por chamada ([09:42] Diego).
3. **Retry e DLQ** ([ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)) — backoff exponencial em 5 degraus; depois, tabela `webhook_dead_letter` com payload, motivo e timestamp; replay manual auditado.
4. **Segurança** ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)) — HMAC-SHA256 sobre o corpo, secret por endpoint gerada pela plataforma, rotação com 24 h de convivência, URL obrigatoriamente `https`, limite de 64 KB de payload.
5. **Semântica de entrega** ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) — at-least-once; `X-Event-Id` estável entre retries e replays.
6. **API de configuração** — CRUD de webhooks por customer, rotação de secret, histórico de entregas (`GET /webhooks/:id/deliveries`) para usuários autenticados; replay de DLQ apenas para `ADMIN` ([09:33] Bruno, [09:34] Marcos, [09:36] Sofia).
7. **Padrões do projeto** ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)) — módulo `src/modules/webhooks`, erros `WEBHOOK_*` estendendo `AppError`, Pino, `error.middleware.ts` sem alterações.

Contratos, schema das tabelas, matriz de erros, algoritmo do worker e observabilidade estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte | Fonte |
|---|-------------|----------------------------------|-------|
| ALT-01 | **Disparo HTTP síncrono dentro de `changeStatus`** | Amarra a transação de pedidos à latência e à disponibilidade do cliente: um cliente lento trava mudanças de status de outros pedidos, e uma falha do cliente não pode desfazer a mudança de status. | [09:04] Bruno, [09:06] Diego |
| ALT-02 | **Redis Streams / broker dedicado** | Exige subir e operar infraestrutura nova; para um time pequeno é "overengineering". A outbox no MySQL existente entrega a mesma desacoplagem com atomicidade nativa. | [09:07] Larissa, [09:07] Diego |
| ALT-03 | **Trigger do MySQL para acordar o worker** | MySQL não tem `LISTEN/NOTIFY`; um trigger não notifica processos externos sem improvisos. Polling de 2 s já atende os 10 s. | [09:09] Bruno, [09:09] Diego |
| ALT-04 | **Worker dentro do processo da API** | Reinício da API derrubaria o worker. | [09:11] Diego |
| ALT-05 | **Retry indefinido** ou **apenas 3 tentativas** | Indefinido deixa evento pendurado para sempre; 3 tentativas cobrem só ~30 min e não sobrevivem a uma manutenção de 2 h. | [09:15] Diego, [09:16] Bruno, [09:16] Diego |
| ALT-06 | **Exactly-once** | Exige coordenação dos dois lados; at-least-once + `event_id` resolve "99% dos casos" e é o padrão de mercado. | [09:25] Diego |
| ALT-07 | **Secret global da plataforma** | Um vazamento comprometeria todos os clientes. | [09:21] Sofia |

Decisões menores também descartadas (DLQ como status na própria outbox, renderizar o payload no envio, filtrar eventos no envio) estão registradas nos ADR-003 e ADR-007.

## 5. Questões em aberto

| ID | Questão | Situação na reunião | Dono sugerido |
|----|---------|---------------------|---------------|
| OQ-01 | **Rate limiting de saída por cliente** (ex.: 50 pedidos mudando de status em 1 minuto geram 50 chamadas). | Fora do escopo; "observar e decidir depois". | Diego — [09:38] Diego, [09:39] Larissa |
| OQ-02 | **Onde vai o `customer_id`** nos endpoints de configuração: body ou path? Não vem do JWT, porque o JWT é do usuário operador. | Não decidido ("no body ou no path"). O FDD propõe body na criação e query string na listagem, pendente de aceite. | Bruno — [09:32] Bruno, [09:32] Larissa |
| OQ-03 | **Ordenação durante retries.** Com um único worker a ordem por `order_id` vale para a primeira tentativa, mas um evento em backoff pode ser ultrapassado pelo evento seguinte do mesmo pedido. Bloquear eventos posteriores do mesmo pedido/endpoint (head-of-line) ou aceitar a inversão? | Não discutido; a reunião só fixou "ordering por order_id enquanto for single-worker". O FDD propõe bloquear por `(webhook_id, order_id)`. | Diego — [09:12] Diego, [09:13] Larissa |
| OQ-04 | **Formato do `X-Signature` durante as 24 h de convivência de secrets** (uma assinatura por secret válida?). | Decidido que a antiga "fica válida por 24 horas em paralelo", sem detalhar o envio. O FDD propõe enviar as duas assinaturas. | Sofia — [09:21] Sofia |
| OQ-05 | **Contagem de tentativas:** "5 tentativas" com 5 intervalos (1m/5m/30m/2h/12h) e "quase 15 horas" implicam 1 envio inicial + 5 reenvios (6 chamadas). Confirmar. | Ambíguo entre a fala de [09:17] Diego e o resumo de [09:48] Larissa. | Larissa — [09:17] Diego, [09:48] Larissa |
| OQ-06 | **Proteção da secret em repouso** (precisa ser recuperável para assinar). | Não discutido; Sofia reservou dois dias de revisão de HMAC e geração de secret antes do deploy. | Sofia — [09:46] Sofia |
| OQ-07 | **Endurecimento de permissões do CRUD** de webhooks (hoje qualquer role autenticada). | Adiado: "Mais pra frente a gente pode endurecer". | Sofia — [09:37] Sofia |
| OQ-08 | **Como o ADMIN descobre o id da DLQ** para o replay: só o endpoint de replay foi definido; listagem/consulta da DLQ por API não foi discutida. | Não discutido. Até decisão, via logs `webhook_moved_to_dead_letter` e consulta à tabela. | Diego — [09:18] Diego |
| OQ-09 | **Autorização por customer:** como confirmar que o usuário autenticado pode administrar o `customerId` enviado no body/query? O JWT atual identifica usuário e role, mas o modelo `User` não tem associação com `Customer`. | Não decidido; definir vínculo/escopo antes de expor CRUD e histórico. Proposta: checagem explícita de autorização por customer, sem confiar apenas no ID fornecido na chamada. | Larissa/Segurança — `prisma/schema.prisma`, [09:32] Bruno e Larissa |
| OQ-10 | **Política de destino para URLs de webhook:** quais endereços de rede o worker pode chamar e como tratar DNS, endereços privados e redirecionamentos? | Não discutido. HTTPS sozinho não define uma política de saída segura. Proposta para revisão de segurança: limitar destinos a endereços públicos e validar o destino efetivo no envio. | Sofia/Plataforma — URL configurável [09:31] Marcos; payload enviado [09:43] Diego |
| OQ-11 | **Semântica de DELETE/desativação:** cancelar entregas enfileiradas, deixá-las drenar ou movê-las à DLQ? Como preservar histórico após remover o cadastro? | A reunião definiu DELETE, mas não seu efeito sobre outbox/histórico. Proposta: exclusão lógica, sem cascata; decidir o destino das entregas pendentes antes da implementação. | Bruno/Produto — [09:33] Bruno; histórico [09:34] Marcos |
| OQ-12 | **Ordenação após DLQ e replay:** evento posterior pode ser entregue enquanto o anterior está na DLQ; replay posterior pode chegar fora da ordem cronológica. | Não discutido. O FDD limita a garantia enquanto o anterior permanece pendente/em retry e registra a inversão possível após DLQ/replay. Decidir entre bloquear até replay ou aceitar a exceção. | Diego/Larissa — [09:12] Diego, [09:13] Larissa, [09:18] Diego |

Itens explicitamente **fora desta fase** (não são questões em aberto, são adiamentos): e-mail de alerta para cliente com falhas ([09:37] Larissa), dashboard visual ([09:40] Larissa) e arquivamento de eventos entregues após ~30 dias ([09:08] Diego).

## 6. Impacto e riscos

**Impacto no sistema existente**
- `changeStatus` passa a ter uma leitura de webhooks do customer e N inserções na outbox na mesma transação; falha nessa etapa faz rollback da mudança de status (comportamento desejado, [09:40] Bruno).
- Novo processo para operar (`npm run worker`) e quatro tabelas novas no schema Prisma.
- Nenhuma dependência nova de runtime; o envio HTTP usa o `fetch` nativo do Node ≥ 20 (`package.json`, `engines`).

**Riscos principais** (detalhados com probabilidade/impacto no [PRD](PRD.md#10-riscos-e-mitigação) e no [FDD](FDD.md#13-riscos-e-mitigação))
- Clientes que não deduplicam processam eventos duplicados → documentação destacada no portal ([09:26] Marcos).
- Um único worker com timeout de 10 s pode acumular atraso se vários endpoints ficarem lentos ao mesmo tempo → monitorar idade do evento pendente mais antigo; escalar exige rever ordenação (OQ-03).
- CRUD autenticado sem vínculo usuário-customer definido pode permitir acesso cruzado a webhooks e payloads (OQ-09).
- URLs de destino configuráveis exigem política de saída contra SSRF, além da validação HTTPS (OQ-10).
- Exclusão em cascata pode apagar eventos/histórico, e replay após DLQ pode quebrar a ordem cronológica (OQ-11/OQ-12).
- Crescimento da outbox sem arquivamento nesta fase → índices em status/`created_at` e acompanhamento de volume.
- Vazamento de secret no lado do cliente ([09:22] Diego) → secret por endpoint e rotação com convivência de 24 h.

**Prazo:** três sprints, incluindo dois dias úteis de revisão de segurança da Sofia antes do deploy; alvo da Atlas é fim de novembro ([09:45] Marcos, [09:46] Larissa, [09:46] Sofia).

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling de 2 s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Payload snapshot e filtro na inserção](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)
