# ADR-002 — Worker em processo separado, lendo a outbox por polling de 2 s

## Status

Aceito
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Com a outbox (ADR-001), algo precisa ler os eventos pendentes e fazer as chamadas HTTP. O requisito de negócio é atraso abaixo de 10 segundos ([09:02] Marcos). Hoje a aplicação tem um único entry-point, `src/server.ts`, que sobe o Express e trata `SIGINT`/`SIGTERM`.

## Decisão

1. O worker lê a outbox por **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos em lote pequeno, processando e marcando o resultado ([09:08] Diego, [09:09] Diego, [09:10] Larissa).
2. O worker roda como **processo separado da API** ([09:11] Diego), com entry-point novo `src/worker.ts` (a criar) e script `npm run worker` ([09:11] Larissa). A lógica de processamento fica dentro do módulo, no arquivo novo `src/modules/webhooks/webhook.worker.ts` ([09:28] Bruno).
3. O worker usa o mesmo banco (`DATABASE_URL`) e a mesma stack, mas abre **sua própria instância de `PrismaClient`**, porque o client é por processo ([09:11] Bruno, [09:30] Bruno). A criação reusa `createPrismaClient()` de `src/config/database.ts`.
4. **Single-worker**: há uma única instância do worker. Com isso, eventos são processados em ordem de `created_at`, o que dá ordenação implícita **por `order_id`**; não há garantia de ordenação global ([09:12] Diego, [09:13] Larissa).

## Alternativas Consideradas

### A. Trigger do banco para reagir imediatamente
- **Por que foi descartada:** MySQL não tem `LISTEN/NOTIFY` como o Postgres; um trigger só executa SQL e não consegue notificar um processo externo sem improvisos (escrever em arquivo, chamar endpoint) ([09:09] Bruno, [09:09] Diego).

### B. Worker dentro do mesmo processo da API
- **Por que foi descartada:** se a API reinicia, o worker morre junto ([09:11] Diego).

### C. Múltiplos workers em paralelo
- **Por que foi adiada:** perde a garantia de ordenação por pedido. Se for preciso escalar, as opções levantadas foram particionar por `order_id` ou usar lock pessimista — "problema do futuro, não agora" ([09:12] Diego, [09:13] Diego).

## Consequências

### Positivas
- Pior caso de espera ≈ 2 s antes do envio, dentro do limite de 10 s ([09:09] Diego, [09:10] Marcos).
- Isolamento de falhas: reinício ou deploy da API não interrompe entregas, e vice-versa.
- Nenhuma dependência nova: loop em Node + Prisma.

### Negativas / trade-offs
- Polling gera consultas periódicas mesmo sem eventos (custo baixo graças aos índices em status e `created_at`, ADR-001).
- Um único worker é um ponto único de vazão: com timeout de 10 s por chamada ([09:42] Diego), poucos clientes lentos podem atrasar a fila de todos. Registrado como risco no FDD (RISK-03) e no PRD.
- Ordenação é **limitação conhecida**: só por `order_id` e só enquanto houver um worker ([09:13] Larissa). Retries com backoff abrem uma brecha adicional nessa ordenação, tratada como questão em aberto no RFC (OQ-03).
