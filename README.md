# Da Reunião ao Documento — Webhooks de Notificação de Pedidos

> O enunciado original do desafio está em [`spec.md`](spec.md) e no repositório base [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript, Express, Prisma e MySQL) decidiu, numa reunião de ~55 minutos, construir um sistema de **webhooks outbound** para avisar clientes B2B quando o status de um pedido muda. A única coisa registrada foi a transcrição literal da call (`TRANSCRICAO.md`). A tarefa foi transformar essa conversa, junto com a leitura do código existente, num pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker que liga cada afirmação à sua origem.

O ponto difícil não era escrever, era **filtrar**: separar o que foi decidido do que foi descartado ou adiado (e-mail de alerta, dashboard, rate limiting), achar as ambiguidades que a reunião deixou passar e ancorar a integração em arquivos reais do código — sem inventar requisitos. O código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|------------|-------|
| **Claude Code** (CLI, modelo Claude Opus) | Agente principal: leu o repositório inteiro e a transcrição, extraiu decisões/requisitos com prompts dirigidos, redigiu todos os documentos e fez as rodadas de revisão crítica. |
| **Codex** | Revisão final dos critérios de aceite, da coerência entre os documentos e das referências do tracker; correção dos fluxos de payload acima de 64 KB e replay. |
| **Script Python de verificação** (gerado pela IA, executado localmente) | Checagem automática anti-alucinação: confere que todo `[hh:mm] Nome` citado existe na transcrição com aquele falante, que todo caminho de código citado existe no repositório e calcula as proporções do tracker. |

## Workflow adotado

1. **Contextualização.** Leitura do enunciado e do código: `order.service.ts` (transação de `changeStatus`), `order.status.ts` (máquina de estados), `shared/errors/*`, `middlewares/*` (erro, auth, validação, log), `app.ts`/`server.ts`/`routes/index.ts`, `prisma/schema.prisma`, `package.json`, `tests/`.
2. **Extração dirigida da transcrição.** Em vez de "resuma a reunião", a extração foi feita por categoria — decisões fechadas, alternativas descartadas (com o motivo), adiamentos, requisitos funcionais, restrições/NFRs e ganchos com o código —, sempre com o timestamp e o falante.
3. **ADRs primeiro** (7): as seis decisões principais listadas no enunciado mais uma decisão secundária com alternativas reais (snapshot do payload + filtro na inserção).
4. **RFC** em cima dos ADRs: proposta, alternativas descartadas e questões em aberto — mantido em nível de arquitetura, apontando para o FDD.
5. **FDD**: modelo de dados, fluxos (outbox, worker, retry, DLQ, replay, rotação), contratos HTTP com exemplos, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção obrigatória de integração com o código.
6. **PRD** por último entre os grandes, como consolidação de produto.
7. **Tracker** montado varrendo os documentos prontos, item a item.
8. **Verificação automática + revisão crítica** contra a checklist do enunciado; correções; nova verificação.

## Prompts customizados

**Prompt 1 — extração dirigida (filtra o que entra e o que não entra):**

```text
Leia TRANSCRICAO.md inteira. NÃO resuma. Produza cinco listas, cada item com
[hh:mm] Nome exatamente como aparece na transcrição:
1. DECISÕES FECHADAS — só o que alguém declarou decidido ("decidido", "anota",
   "beleza, decisão") ou que teve concordância explícita.
2. ALTERNATIVAS DESCARTADAS — a alternativa, quem a levantou e o motivo do descarte.
3. ADIADOS / FORA DE ESCOPO — o que foi empurrado para "próxima fase",
   "observar depois" ou "projeto separado".
4. REQUISITOS FUNCIONAIS e NÃO FUNCIONAIS mencionados, com valores numéricos.
5. AMBIGUIDADES — pontos em que duas falas não fecham entre si ou em que a
   reunião não decidiu algo que a implementação vai precisar decidir.
Se um item não tiver timestamp de origem, não o inclua.
```

**Prompt 2 — ancoragem no código para o FDD:**

```text
Para cada ponto de integração do módulo de webhooks, abra o arquivo real e cite:
caminho, função/classe e o que muda. Arquivos obrigatórios a examinar:
src/modules/orders/order.service.ts (changeStatus e o tipo TxClient),
src/shared/errors/app-error.ts e http-errors.ts, src/middlewares/error.middleware.ts,
src/middlewares/validate.middleware.ts, src/middlewares/auth.middleware.ts,
src/routes/index.ts, src/app.ts, src/server.ts, src/config/database.ts,
src/config/env.ts, prisma/schema.prisma.
Regras: (a) não cite arquivo que não existe — arquivos novos devem ser marcados
"a criar"; (b) se o comportamento atual do código limitar uma decisão da reunião,
diga qual é a limitação em vez de escondê-la; (c) decisões de implementação que a
reunião não tomou devem ser marcadas "(proposta de implementação)".
```

**Prompt 3 — revisão adversarial (usado nas rodadas de correção):**

```text
Aja como a Sofia (segurança) e o Diego (plataforma) revisando o FDD. Procure:
SQL que não faz o que o texto diz, contradições entre ADR/RFC/FDD/PRD, números que
não batem com a transcrição, requisitos que a reunião descartou aparecendo como
escopo, e qualquer afirmação sem fonte. Liste cada problema com o trecho exato.
```

## Iterações e ajustes

Foram **5 iterações principais** (rascunho dos ADRs → RFC/FDD/PRD → verificação automática e revisão adversarial → correções de rastreabilidade/contratos → revisão dos riscos e ajustes finais). Os ajustes concretos:

1. **Ambiguidade de "5 tentativas".** O primeiro rascunho do ADR-003 tratava "5 tentativas" como 5 envios. Na revisão ficou claro que isso não fecha: 5 envios só têm 4 intervalos (≈2h36min), mas Diego listou 5 intervalos e disse "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). Adotei 1 envio + 5 reenvios (6 chamadas, ~14h36min) e registrei a divergência com o resumo de [09:48] Larissa como **OQ-05** no RFC, em vez de escolher silenciosamente.
2. **Bug de precedência no SQL do worker.** A revisão adversarial pegou que `WHERE (pendente) OR (processando expirado) AND NOT EXISTS(...)` aplicava a regra de ordenação só ao segundo ramo — eventos pendentes poderiam ultrapassar eventos anteriores do mesmo pedido. Parênteses corrigidos no FDD (F2).
3. **Payload em coluna `JSON` quebraria a assinatura.** O rascunho do modelo de dados guardava o snapshot como `Json`. Como o MySQL normaliza ordem de chaves e espaços em colunas `JSON`, os bytes enviados poderiam deixar de ser exatamente os que foram assinados. Mudei para texto (`MediumText`) e expliquei o motivo no FDD.
4. **Código de erro que o código atual não consegue produzir.** A primeira versão da matriz dizia que URL `http` devolvia `400 WEBHOOK_INVALID_URL`. Lendo `src/middlewares/validate.middleware.ts`, toda falha Zod vira `VALIDATION_ERROR`; o FDD passou a documentar que `WEBHOOK_INVALID_URL` sai em `details[].message`, e o ADR-006 registra a limitação como consequência negativa.
5. **Lacuna de ordenação durante retries.** A reunião garantiu ordenação por pedido "enquanto for single-worker", mas não percebeu que o backoff permite que o evento seguinte do mesmo pedido seja entregue antes do anterior. O FDD propõe bloquear por `(webhookId, orderId)` e o RFC registra a decisão como **OQ-03**.
6. **Formato do entregável.** O repositório base trazia `docs/adrs/README.md`; ele foi removido para que a pasta contenha apenas arquivos `ADR-NNN-titulo.md` (o índice ficou neste README e no RFC).
7. **Verificação automática do tracker.** O script apontou uma referência mal formada no RFC (o timestamp 09:17 seguido da palavra "e" em vez do nome do falante) e citações de arquivos que ainda não existem (`src/worker.ts`, `src/modules/webhooks/*`). A referência foi corrigida e os arquivos novos passaram a ser marcados como "a criar". Após a revisão final, o tracker tem 212 linhas: 176 (83,0%) da transcrição e 36 do código.
8. **Corte de escopo.** Itens que pareciam boas práticas mas não tinham origem na reunião (por exemplo, criptografia da secret em repouso e endpoint de listagem da DLQ) não viraram requisito: foram para as questões em aberto (OQ-06, OQ-08) com o dono sugerido.
9. **Revisão de segurança e semântica.** A revisão final identificou que o JWT atual não vincula User a Customer, que HTTPS sozinho não limita destinos de saída, que DELETE em cascata apagaria outbox/histórico, e que replay após DLQ pode quebrar ordenação. Esses pontos foram registrados como questões para revisão, não como decisões da reunião. A meta de prazo também foi corrigida: fim de novembro aparece como prazo solicitado pela Atlas, não como compromisso dos três clientes.
10. **Payload acima do limite.** O FDD chegou a prever uma linha de erro na outbox e outra pendente para o mesmo endpoint. O fluxo foi corrigido para persistir apenas a falha na DLQ e bloquear o replay enquanto o snapshot continuar acima de 64 KB.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md) — por que e o quê: problema, público, métricas, escopo e requisitos.
2. [`docs/RFC.md`](docs/RFC.md) — a proposta técnica, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — uma decisão por arquivo:
   - [ADR-001 — Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 — Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
   - [ADR-003 — Retry com backoff e DLQ](docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)
   - [ADR-004 — HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
   - [ADR-005 — At-least-once com X-Event-Id](docs/adrs/ADR-005-at-least-once-com-x-event-id.md)
   - [ADR-006 — Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
   - [ADR-007 — Payload snapshot e filtro na inserção](docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)
4. [`docs/FDD.md`](docs/FDD.md) — como construir: dados, fluxos, contratos, erros, resiliência, observabilidade e integração com o código.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — de onde veio cada item.

```
.
├── README.md            (este arquivo)
├── TRANSCRICAO.md       (inalterado)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/ADR-001 … ADR-007
├── src/ prisma/ tests/  (inalterados)
```
