# ADR-007 — Payload renderizado (snapshot) e filtro de eventos aplicados na inserção da outbox

## Status

Aceito
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

Duas perguntas de modelagem da outbox ficaram em aberto durante a reunião:

1. A outbox guarda o payload já renderizado ou só o `order_id`, renderizando na hora do envio ([09:51] Bruno)?
2. O filtro de status de cada webhook (ex.: "só quero saber quando vira SHIPPED e DELIVERED", [09:33] Marcos) é aplicado ao inserir na outbox ou ao enviar ([09:34] Diego)?

## Decisão

1. **Snapshot na inserção:** o payload é renderizado e gravado no momento em que o evento entra na outbox. Se o pedido mudar depois, o evento continua refletindo o estado de quando o status mudou ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
2. **Filtro na inserção:** se nenhum webhook do customer quer aquele status, nenhuma linha é inserida ([09:34] Bruno, [09:34] Diego).
3. O payload é enxuto: `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order, como `total_cents` — **sem itens**; detalhes são obtidos via `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).

## Alternativas Consideradas

### A. Guardar apenas `order_id` e renderizar no envio
- **Por que foi descartada:** um retry horas depois enviaria o estado atual do pedido, não o da mudança — "senão tem caso esquisito" ([09:52] Larissa).

### B. Inserir sempre e filtrar no envio
- **Por que foi descartada:** cria linhas que nunca serão enviadas; filtrar antes "economiza linha na tabela" ([09:34] Bruno).

### C. Incluir os itens do pedido no payload
- **Por que foi descartada:** infla o payload sem necessidade; o cliente pode consultar `GET /orders/:id` ([09:43] Diego).

## Consequências

### Positivas
- Retries e replays são reprodutíveis: o mesmo `event_id` sempre carrega o mesmo corpo e, portanto, a mesma assinatura HMAC para uma mesma secret (ADR-004).
- Outbox menor e worker mais simples (o worker não consulta pedidos nem filtros).

### Negativas / trade-offs
- Alterar o filtro de um webhook **não afeta eventos já enfileirados**, e um webhook criado depois de uma mudança de status não recebe eventos anteriores.
- A leitura dos webhooks ativos do customer passa a rodar dentro da transação de `changeStatus` (uma consulta indexada por `customer_id`).
- O limite de 64 KB ([09:24] Diego, [09:24] Larissa) se aplica ao snapshot; com o payload enxuto ele não deve ser atingido na prática.
