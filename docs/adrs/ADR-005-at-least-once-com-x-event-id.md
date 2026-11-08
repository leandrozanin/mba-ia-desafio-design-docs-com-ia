# ADR-005 — Garantia de entrega at-least-once com deduplicação via `X-Event-Id`

## Status

Aceito
- **Decisores:** Diego, Larissa; Marcos (comunicação aos clientes)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)

## Contexto

Com outbox + worker + retry (ADR-001/002/003) existem janelas em que o cliente recebe o evento, mas a confirmação não chega até nós — por exemplo, o cliente processa e demora mais de 10 s para responder, ou o worker cai entre o envio e a marcação como entregue. Nesses casos o evento será reenviado.

## Decisão

1. A entrega é **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado ([09:24] Diego).
2. Cada evento recebe um **`event_id` (UUID) gerado quando o evento entra na outbox**, único por evento, enviado no header **`X-Event-Id`** ([09:25] Diego) e também no corpo, campo `event_id` ([09:43] Diego). O cliente deduplica pelo `event_id` do lado dele ([09:25] Diego).
3. O comportamento será documentado com destaque no portal do desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

### A. Exactly-once
- **Por que foi descartada:** exigiria coordenação dos dois lados e é muito mais complexo; at-least-once com `event_id` "resolve 99% dos casos" e é o padrão de mercado — "Stripe faz assim, GitHub faz assim" ([09:25] Diego).

### B. At-most-once (enviar uma vez, sem retry)
- **Por que não foi adotada:** contradiz a política de retry decidida (ADR-003) e perderia eventos em qualquer indisponibilidade do cliente.

## Consequências

### Positivas
- Implementação simples do nosso lado; nenhum protocolo extra de confirmação.
- Retries e replays de DLQ são seguros desde que o cliente deduplique.

### Negativas / trade-offs
- Transfere responsabilidade ao cliente — "Isso joga responsabilidade pro cliente" ([09:25] Sofia). Mitigação: documentação destacada no portal ([09:26] Marcos).
- O `event_id` precisa ser preservado em retries e no replay da DLQ: o replay recoloca **o mesmo evento**, não cria um novo (ver FDD, fluxo F5).
