# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|-------|-------|
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks de notificação de mudança de status de pedidos |
| **Responsável de produto** | Marcos (PM) |
| **Responsável técnico** | Larissa (Tech Lead) |
| **Status** | Aprovado na reunião técnica; implementação em 3 sprints |
| **Relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

O OMS passa a **avisar ativamente** os clientes B2B quando o status de um pedido deles muda, por meio de **webhooks**: uma chamada HTTPS para uma URL cadastrada pelo cliente, com os dados essenciais da mudança e uma assinatura que prova a origem. Hoje o OMS não tem nenhum mecanismo de notificação externa; os clientes precisam perguntar repetidamente pelo `GET /orders` ([09:00] Marcos).

A feature cobre o cadastro e a manutenção dos webhooks via API, a entrega confiável dos eventos (com novas tentativas automáticas), o histórico de entregas para o cliente e uma ferramenta administrativa para reprocessar falhas.

## 2. Problema e motivação

- **Integração lenta e cara para o cliente.** Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` de tempos em tempos para descobrir mudanças ([09:00] Marcos).
- **Risco comercial.** Os três fizeram pedido formal; a Atlas indicou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos).
- **Expectativa de "tempo real".** Para os clientes, qualquer atraso abaixo de 10 segundos é tempo real; o essencial é não depender de atualização manual ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Quem é | O que precisa |
|---------|--------|---------------|
| **Cliente B2B integrador** | Times técnicos de Atlas Comercial, MaxDistribuição e Nova Cargo | Receber os eventos dos pedidos deles, validar a origem e evitar processar o mesmo evento duas vezes ([09:00] Marcos, [09:19] Sofia, [09:25] Diego) |
| **Usuário da API que representa o cliente** | Usuário do OMS autenticado com JWT que atua em nome do cliente | Cadastrar, editar, remover e listar webhooks; rotacionar a secret; consultar o histórico ([09:32] Marcos) |
| **Administrador do OMS** | Usuário com role `ADMIN` | Reprocessar eventos que falharam definitivamente, com registro de auditoria ([09:36] Sofia) |

**Cenários**

- **CEN-01 — Pedido despachado.** O pedido da Nova Cargo vai para `SHIPPED`; em poucos segundos o sistema da Nova Cargo recebe o evento e atualiza o rastreio, sem consultar a API.
- **CEN-02 — Filtrar o que interessa.** A MaxDistribuição só quer `SHIPPED` e `DELIVERED`; mudanças para `PAID` ou `PROCESSING` não geram chamadas para ela ([09:33] Marcos).
- **CEN-03 — Cliente em manutenção.** O endpoint da Atlas fica 2 h fora do ar; o OMS reenvia automaticamente e o evento é entregue quando o endpoint volta ([09:16] Diego).
- **CEN-04 — Secret vazada.** O cliente percebe que a secret apareceu em um log, pede uma nova pela API e tem 24 h para atualizar seus sistemas sem perder eventos ([09:21] Sofia, [09:22] Diego).
- **CEN-05 — Investigação de falha.** O cliente consulta os últimos envios (sucesso/falha, payload, resposta, tempo de resposta) para entender por que não recebeu algo ([09:34] Marcos).
- **CEN-06 — Reprocessamento.** Depois que o cliente corrige o endpoint, um ADMIN reprocessa os eventos que esgotaram as tentativas ([09:18] Diego).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
|----|----------|---------|------|
| OBJ-01 | Notificar mudanças de status em tempo real | Tempo entre mudança e primeira tentativa; p95 como indicador proposto | **< 10 segundos** (limite discutido em [09:02] Marcos; escolha de p95 é proposta de medição) |
| OBJ-02 | Nunca perder um evento de mudança de status | Mudanças de status commitadas sem evento correspondente na outbox (para webhooks interessados) | **0 casos** ([09:40] Bruno) |
| OBJ-03 | Atender aos clientes que fizeram o pedido formal | Clientes com webhook ativo em produção | **Meta proposta: 3 de 3** (Atlas, MaxDistribuição, Nova Cargo), com data-alvo a confirmar. A transcrição registra fim de novembro como prazo solicitado pela Atlas ([09:00] Marcos, [09:45] Marcos) |
| OBJ-04 | Tolerar indisponibilidades do cliente | Janela coberta por retentativas automáticas antes da DLQ | **~15 horas** ([09:17] Diego, [09:17] Marcos) |
| OBJ-05 | Substituir o polling | Volume de chamadas `GET /orders` dos três clientes após a adoção | Tendência de queda (linha de base a medir antes do lançamento; meta numérica não definida na reunião) |

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de webhooks por customer, via API autenticada.
- Filtro, por webhook, dos status de pedido que ele quer receber.
- Envio automático de evento a cada mudança de status de pedido que corresponda ao filtro.
- Assinatura de cada envio e secret exclusiva por webhook, com rotação.
- Retentativas automáticas e fila de falhas definitivas (DLQ) com reprocessamento manual por ADMIN.
- Histórico das entregas por webhook.

### 5.2 Fora de escopo

| Item | Situação | Fonte |
|------|----------|-------|
| **E-mail de alerta ao cliente** quando o webhook falha repetidamente | **Adiado** para a próxima fase, após medir o impacto | [09:37] Marcos, [09:37] Larissa |
| **Dashboard/painel visual** para o cliente ver seus webhooks | **Descartado** desta feature — projeto separado do time de frontend; só endpoints | [09:39] Marcos, [09:40] Larissa |
| **Rate limiting de envio** por cliente | **Adiado** — observar e decidir depois | [09:38] Diego, [09:39] Larissa |
| **Webhooks de entrada** (clientes enviando para o OMS) | **Descartado** — só saída | [09:02] Marcos, [09:03] Sofia |
| **Garantia de ordenação global** e múltiplos workers | **Adiado** — só ordenação por pedido, com um worker | [09:13] Larissa, [09:14] Marcos |
| **Garantia exactly-once** | **Descartado** — entrega at-least-once | [09:25] Diego |
| **Arquivamento** de eventos entregues (~30 dias) | **Adiado** — fora do escopo da feature | [09:08] Diego |
| Itens do pedido no payload | **Descartado** — cliente consulta `GET /orders/:id` | [09:43] Diego |
| Evento na criação do pedido | Não discutido; a feature trata **mudanças** de status ([09:00] Marcos). Ver RFC | — |

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
|----|-----------|-------|
| FR-01 | O usuário autenticado cadastra um webhook informando o customer, a URL e a lista de status que deseja receber; a secret é **gerada pela plataforma** e devolvida na criação. | [09:31] Marcos |
| FR-02 | O usuário lista os webhooks de um customer. | [09:33] Bruno |
| FR-03 | O usuário edita um webhook (URL, status de interesse, ativo/inativo). | [09:33] Bruno |
| FR-04 | O usuário remove um webhook. | [09:33] Bruno |
| FR-05 | Cada webhook define quais status de pedido quer receber; mudanças para outros status não geram envio para ele. | [09:33] Marcos, [09:34] Bruno |
| FR-06 | Toda mudança de status de pedido gera um evento `order.status_changed` para cada webhook ativo interessado do customer do pedido. | [09:00] Marcos, [09:43] Diego |
| FR-07 | O evento contém `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`, sem itens, refletindo o estado do momento da mudança. | [09:43] Diego, [09:52] Larissa |
| FR-08 | Cada envio é assinado com HMAC-SHA256 sobre o corpo, com a secret do webhook, no header `X-Signature`. | [09:20] Sofia, [09:22] Sofia |
| FR-09 | Cada envio traz os headers `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `Content-Type: application/json`. | [09:44] Diego, [09:44] Sofia |
| FR-10 | Envios que falham são retentados automaticamente com intervalos crescentes de 1 min, 5 min, 30 min, 2 h e 12 h. | [09:17] Larissa |
| FR-11 | Eventos que esgotam as tentativas vão para uma fila de falhas (DLQ) com o payload, o motivo e o momento da falha. | [09:18] Diego |
| FR-12 | Um ADMIN reprocessa manualmente um evento da DLQ; o sistema registra quem fez o reprocessamento. | [09:18] Diego, [09:36] Sofia |
| FR-13 | O usuário solicita uma nova secret para um webhook; a anterior continua válida por 24 h. | [09:21] Sofia |
| FR-14 | O usuário consulta o histórico das últimas 100 entregas de um webhook, com sucesso/falha, payload, resposta e tempo de resposta. | [09:34] Marcos |
| FR-15 | O cadastro só aceita URLs `https`; `http` é recusado com erro de validação. | [09:23] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Fonte |
|----|-----------|-------|
| NFR-01 | Latência: a primeira tentativa de entrega ocorre em menos de 10 s após a mudança de status (leitura da fila a cada 2 s). | [09:02] Marcos, [09:10] Larissa |
| NFR-02 | Consistência: a mudança de status e o registro do evento são atômicos — nunca há status alterado sem evento, nem evento de mudança que não aconteceu. | [09:06] Diego, [09:40] Bruno |
| NFR-03 | Isolamento: a lentidão ou indisponibilidade de um cliente não afeta a mudança de status de nenhum pedido. | [09:04] Bruno |
| NFR-04 | Entrega at-least-once; o cliente deduplica pelo `X-Event-Id`. | [09:24] Diego, [09:25] Diego |
| NFR-05 | Ordenação garantida apenas por pedido, enquanto houver um único worker. | [09:12] Diego, [09:13] Larissa |
| NFR-06 | Timeout de 10 s por chamada ao cliente; acima disso conta como falha. | [09:42] Diego |
| NFR-07 | Payload limitado a 64 KB; acima disso o evento não é enviado e é tratado como erro. | [09:23] Sofia, [09:24] Larissa |
| NFR-08 | Secret exclusiva por webhook; nunca uma secret global. | [09:21] Sofia |
| NFR-09 | O processamento de entregas roda em processo separado da API; reiniciar a API não interrompe entregas. | [09:11] Diego |
| NFR-10 | Sem infraestrutura nova: mesmo MySQL e mesma stack (Prisma, Pino, padrões de erro e de módulos). | [09:07] Diego, [09:30] Larissa |
| NFR-11 | Identificadores UUID, como no restante do projeto. | [09:51] Larissa |
| NFR-12 | Auditoria: todo reprocessamento de DLQ registra o usuário que o executou. | [09:36] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---------|------------------|-----|
| Outbox na mesma transação da mudança de status, no MySQL existente | Tabela que cresce e precisará de arquivamento futuro, em troca de atomicidade e zero infra nova | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado em polling de 2 s, instância única | Latência mínima de ~2 s e ordenação só por pedido, em troca de simplicidade | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas com backoff até ~15 h, depois DLQ com replay manual | Evento pode atrasar até ~15 h; reprocessamento exige ação de ADMIN | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256, secret por webhook, rotação com 24 h | A plataforma guarda secrets recuperáveis (precisam ser protegidas) | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Herda as limitações atuais (ex.: sem métricas/tracing nativos) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload snapshot e filtro na inserção | Mudança de filtro só vale para eventos futuros | [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Fonte |
|----|-------------|-------|
| DEP-01 | Revisão de segurança da Sofia (HMAC e geração de secret), com pelo menos 2 dias úteis antes do deploy | [09:46] Sofia |
| DEP-02 | Portal do desenvolvedor atualizado por Marcos: como integrar via API, como validar a assinatura e como deduplicar | [09:26] Marcos, [09:40] Marcos |
| DEP-03 | Usuários do OMS que representam cada cliente, para autenticar as chamadas de configuração | [09:32] Marcos |
| DEP-04 | Novo processo (worker) operado em produção junto com a API | [09:11] Diego |
| DEP-05 | Fluxo atual de mudança de status de pedidos (`PATCH /orders/:id/status`) como gatilho dos eventos | `src/modules/orders/order.routes.ts` |
| DEP-06 | Confirmação do prazo com a Atlas | [09:47] Marcos |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|----|-------|---------------|---------|-----------|
| RSK-01 | A entrega não atender ao prazo solicitado pela Atlas e ela migrar para o concorrente | Média | Alto | Manter o escopo enxuto e confirmar com a Atlas a data de entrega; a reunião estimou 3 sprints, mas não registrou confirmação de uma data final ([09:00] Marcos, [09:46] Larissa, [09:47] Marcos) |
| RSK-02 | Cliente processar evento duplicado por não deduplicar pelo `X-Event-Id` | Média | Médio | Comportamento at-least-once destacado no portal do desenvolvedor ([09:26] Marcos) |
| RSK-03 | Vazamento de secret pelo cliente (já aconteceu antes) | Média | Alto | Secret por webhook limita o impacto; rotação self-service com 24 h de convivência ([09:21] Sofia, [09:22] Diego) |
| RSK-04 | Atraso acima de 10 s quando vários endpoints de clientes ficam lentos ao mesmo tempo (um único worker, timeout de 10 s) | Média | Alto | Monitorar a idade do evento pendente mais antigo; escalar exige rever a ordenação ([09:12] Diego, [09:42] Diego) |
| RSK-05 | Cliente receber rajadas de chamadas quando muitos pedidos mudam juntos | Média | Médio | Observar volume por cliente após o lançamento; rate limiting decidido depois ([09:38] Diego, [09:39] Larissa) |
| RSK-06 | Cliente fora do ar por mais de ~15 h perder eventos automáticos | Baixa | Médio | DLQ preserva os eventos; ADMIN reprocessa após a correção ([09:17] Marcos, [09:18] Diego) |
| RSK-07 | Acesso cruzado a webhooks ou histórico de outro customer por falta de vínculo de autorização no modelo atual | Média | Alto | Definir autorização por customer antes de expor CRUD/histórico; User atual não tem vínculo com Customer (prisma/schema.prisma; RFC OQ-09) |
| RSK-08 | Requisições a destinos internos ou não autorizados por URL configurável | Média | Alto | Revisão de segurança deve definir política de egress e validação de destino além de HTTPS (URL configurável: [09:31] Marcos; RFC OQ-10) |

## 11. Critérios de aceitação

| ID | Critério |
|----|----------|
| CA-01 | Um usuário autenticado cadastra um webhook `https` com lista de status e recebe a secret na resposta; um cadastro com `http` é recusado. |
| CA-02 | Ao mudar o status de um pedido para um status assinado, o endpoint do cliente recebe o evento em menos de 10 s, com os headers definidos e assinatura válida para a secret do webhook. |
| CA-03 | Mudanças para status não assinados não geram chamadas. |
| CA-04 | Se o endpoint está fora do ar, o evento é reenviado nos intervalos definidos e, se continuar falhando, aparece na DLQ com payload e motivo. |
| CA-05 | Um ADMIN reprocessa um evento da DLQ e ele é reenviado com o mesmo `X-Event-Id`; um usuário sem role ADMIN recebe acesso negado. |
| CA-06 | O usuário rotaciona a secret e, durante 24 h, o cliente consegue validar os envios com a secret antiga ou com a nova. |
| CA-07 | O usuário consulta o histórico e vê as últimas 100 entregas com sucesso/falha, payload, resposta e tempo de resposta. |
| CA-08 | Se a mudança de status falhar, nenhum evento é enviado; se for concluída, o evento sempre é registrado. |
| CA-09 | O usuário lista, edita e remove webhooks de um customer. |

## 12. Estratégia de testes e validação

- **Testes automatizados** no padrão atual do repositório (`tests/*.test.ts` com Vitest + Supertest contra MySQL real, `vitest.config.ts`): CRUD e validações da API, criação atômica do evento junto com a mudança de status (incluindo rollback), filtro por status e regras de autorização (ADMIN no replay).
- **Testes do worker** contra um servidor HTTP local que simula o cliente: sucesso, timeout de 10 s, respostas de erro, sequência de retentativas, ida para a DLQ e replay. Os intervalos de retentativa são verificados pelo agendamento calculado, sem esperar horas.
- **Verificação da assinatura** com um verificador independente (o mesmo código de exemplo publicado no portal), inclusive durante as 24 h após rotação.
- **Revisão de segurança** da Sofia sobre HMAC e geração de secret, com dois dias úteis reservados antes do deploy ([09:46] Sofia).
- **Validação com os clientes:** Marcos confirma o prazo com a Atlas e documenta a integração no portal ([09:40] Marcos, [09:47] Marcos); a métrica OBJ-01 é acompanhada desde o primeiro dia em produção.
