# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada

## Status

Aceito
- **Decisores:** Larissa, Diego, Bruno; Marcos (aceite de produto)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar. Já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). Precisamos de uma política que não desista cedo demais, mas que também não deixe evento "pendurado pra sempre" quando o cliente sumiu ([09:15] Diego).

## Decisão

1. **Backoff exponencial com 5 reenvios** nos intervalos **1 min → 5 min → 30 min → 2 h → 12 h** ([09:15] Diego, [09:17] Diego, [09:17] Larissa).
   - **Interpretação adotada:** a primeira chamada acontece quando o worker pega o evento; cada um dos 5 intervalos agenda um novo envio após uma falha. Isso totaliza **6 chamadas HTTP** e ~14h36min entre a primeira falha e a última tentativa — coerente com o "total de quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). A redação "total 5 tentativas" do resumo ([09:48] Larissa) é ambígua (5 envios só teriam 4 intervalos, ~2h36min); está registrada como questão em aberto no RFC (OQ-05) para confirmação com a Larissa.
2. É falha passível de retry: timeout de 10 s ([09:42] Diego), erro de rede ou resposta HTTP fora da faixa 2xx.
3. Esgotados os reenvios, o evento é considerado **falha permanente** e movido para uma tabela separada **`webhook_dead_letter`**, com o payload, o motivo da falha e o timestamp ([09:15] Diego, [09:18] Diego).
4. Reprocessamento é **manual**, pelo endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). Exige role `ADMIN` e registra quem fez o replay, para auditoria ([09:36] Sofia, [09:36] Larissa).

## Alternativas Consideradas

### A. Retry indefinido com backoff
- **Por que foi descartada:** o evento pode ficar pendurado para sempre se o cliente desapareceu ([09:15] Diego).

### B. 3 tentativas (mais agressivo)
- **Por que foi descartada:** retentaria três vezes em ~30 minutos e mataria o evento; não cobre uma indisponibilidade de manhã nem uma manutenção de 2 h ([09:16] Bruno, [09:16] Diego).

### C. Marcar como `failed` na própria outbox, sem tabela de DLQ
- **Por que foi descartada:** a tabela separada deixa a leitura da outbox principal mais limpa e serve de evidência para debug e reprocessamento ([09:17] Larissa, [09:18] Diego).

## Consequências

### Positivas
- Cobre indisponibilidades de até ~15 h, aceitas pelo produto: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).
- Falhas permanentes ficam auditáveis e reprocessáveis sem acesso direto ao banco.

### Negativas / trade-offs
- Um evento pode chegar ao cliente com até ~15 h de atraso em cenário de falha prolongada.
- Replay manual gera trabalho operacional; não há aviso automático ao cliente — o e-mail de alerta foi adiado para a próxima fase ([09:37] Larissa).
- Retries reenviam o mesmo evento, reforçando a necessidade de deduplicação no cliente (ADR-005).
