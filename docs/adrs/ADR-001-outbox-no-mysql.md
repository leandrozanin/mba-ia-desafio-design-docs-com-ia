# ADR-001 — Padrão Outbox no MySQL existente

## Status

Aceito
- **Data:** reunião técnica de quinta-feira (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Consultados:** Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-e-filtro-na-insercao.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status dos pedidos muda, com atraso menor que 10 segundos ([09:00] Marcos, [09:02] Marcos). Hoje eles fazem polling em `GET /orders`.

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e debita/repõe `stockQuantity` dos produtos. Bruno observou que essa transação "já é pesada" e que uma chamada HTTP dentro dela faria um cliente lento travar mudanças de status de outros pedidos; também não faz sentido dar rollback na mudança de status porque o cliente está fora do ar ([09:04] Bruno).

O requisito de consistência é forte: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).

## Decisão

Registrar cada evento de webhook numa tabela **`webhook_outbox` no mesmo MySQL** da aplicação, **dentro da mesma transação** de `changeStatus` que grava `orders` e `order_status_history` ([09:06] Diego, [09:08] Larissa). Se a inserção na outbox falhar, a transação inteira sofre rollback ([09:40] Bruno, [09:41] Diego).

A outbox tem coluna de status (pendente, processando, falhou, entregue) e índices em status e `created_at` ([09:08] Diego). O id é UUID, seguindo o restante do schema ([09:51] Larissa).

A inserção é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` da transação corrente, em vez de injetar um repositório inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

## Alternativas Consideradas

### A. Disparo síncrono dentro de `changeStatus`
- **Por que foi descartada:** acopla a latência do cliente externo à transação de pedidos; um cliente lento trava mudanças de status de outros pedidos, e uma falha do cliente não pode desfazer a mudança de status ([09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão").

### B. Redis Streams (ou broker equivalente)
- **Por que foi descartada:** exigiria subir e operar infraestrutura nova; para um time pequeno, "subir Redis Cluster pra isso é overengineering" ([09:07] Larissa, [09:07] Diego). Também não resolveria sozinho a atomicidade entre o commit do pedido e a publicação do evento — a outbox resolve isso por construção ([09:06] Diego).

## Consequências

### Positivas
- Atomicidade: se a transação do pedido commitou, o evento existe; se deu rollback, o evento some junto ([09:06] Diego).
- Nenhuma infraestrutura nova: mesmo MySQL 8 (`docker-compose.yml`), mesmo Prisma (`prisma/schema.prisma`).
- O tempo de resposta de `PATCH /orders/:id/status` passa a depender apenas de `INSERT`s adicionais, nunca da rede do cliente.

### Negativas / trade-offs
- A tabela cresce continuamente; o arquivamento de linhas entregues (≈30 dias) foi explicitamente deixado **fora do escopo** desta feature ([09:08] Diego). Até lá, o crescimento precisa ser monitorado.
- A latência mínima de entrega passa a ser limitada pelo intervalo de leitura do worker (2 s, ver ADR-002), aceito pelo produto ([09:10] Larissa).
- A transação de `changeStatus` fica um pouco maior (uma leitura de webhooks e um `INSERT` por endpoint interessado). Trade-off explícito: preferimos uma transação um pouco mais longa a qualquer possibilidade de o status mudar sem o evento.
