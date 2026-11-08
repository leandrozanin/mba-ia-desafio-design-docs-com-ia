# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 h

## Status

Aceito (sujeito à revisão de segurança da Sofia antes do deploy — [09:46] Sofia)
- **Decisores:** Sofia (Segurança), Larissa
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-e-filtro-na-insercao.md)

## Contexto

Os eventos carregam dados de pedidos e saem para endpoints fora da nossa infraestrutura. O cliente precisa validar que a requisição veio de nós e que ninguém adulterou o payload no meio do caminho ([09:19] Sofia). A plataforma já teve cliente que vazou secret em log de aplicação ([09:22] Diego).

## Decisão

1. Cada requisição é assinada com **HMAC-SHA256 sobre o corpo do request**, e a assinatura vai no header **`X-Signature`** ([09:20] Sofia, [09:22] Sofia).
2. A secret é **única por endpoint de webhook** — nunca uma secret global da plataforma ([09:21] Sofia). Ela é **gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
3. A configuração do webhook armazena `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno, [09:21] Sofia).
4. A secret é **rotacionável via API**: ao rotacionar, a secret antiga continua válida por **24 horas em paralelo**; depois disso deixa de valer ([09:21] Sofia, [09:22] Sofia).
5. A URL do webhook **deve ser `https`**; `http` é recusado por validação do schema Zod ([09:23] Sofia).

## Alternativas Consideradas

### A. Secret global da plataforma
- **Por que foi descartada:** o vazamento de uma única secret comprometeria todos os clientes — "se vaza uma, vaza tudo" ([09:21] Sofia).

### B. Rotação com troca imediata (sem convivência)
- **Por que não foi adotada:** o cliente precisa de tempo para migrar os sistemas dele; por isso a antiga fica válida por 24 h ([09:21] Sofia).

### C. Outro algoritmo de assinatura
- **Por que não:** HMAC-SHA256 é o padrão de mercado e "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).

## Consequências

### Positivas
- Integridade e autenticidade verificáveis pelo cliente com bibliotecas padrão.
- Raio de impacto de um vazamento limitado a um único endpoint.
- Rotação sem downtime para o cliente.

### Negativas / trade-offs
- A plataforma precisa guardar a secret de forma recuperável (para assinar), o que torna a tabela `webhook_endpoints` sensível. A forma de proteção em repouso **não foi decidida na reunião** e fica para a revisão de segurança (RFC OQ-06).
- Durante as 24 h de convivência, o envio precisa permitir validação com qualquer uma das duas secrets; o formato do header nesse período é proposta do FDD e está listado no RFC (OQ-04).
- Assinar apenas o corpo não protege contra replay; o `X-Timestamp` ([09:44] Diego) é enviado para que o cliente "consiga detectar replay attack se quiser", mas não faz parte do conteúdo assinado decidido.
