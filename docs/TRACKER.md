# Tracker de rastreabilidade

Cada linha liga um item dos documentos à origem dele: uma fala da reunião (`TRANSCRICAO`, com `[hh:mm] Nome`) ou um arquivo do código (`CODIGO`, com o caminho). Itens que os documentos marcam como escolha própria ("escolha deste FDD", "meta proposta por este PRD") aparecem com o tipo `Escolha de projeto` e apontam para a fala ou o arquivo que motivou a escolha.

Total: 224 linhas, 195 com fonte na transcrição (87%) e 29 com fonte no código. Todas foram conferidas por script: cada `[hh:mm] Nome` existe na transcrição com aquele falante naquele minuto, e cada caminho de código existe no repositório. Depois disso, uma revisão linha a linha conferiu se a fala ou o arquivo sustenta o conteúdo resumido.

Os ids seguem a ordem em que os itens aparecem em cada documento. Nas tabelas numeradas do PRD (cenários, objetivos, requisitos, riscos, critérios), o id é o mesmo do documento; um sufixo `B` indica uma segunda fonte para o mesmo item. No RFC, `RFC-ALT-NN` e `RFC-OQ-NN` seguem a ordem das alternativas e das questões em aberto. No FDD, `FDD-CONTRATO-01` a `07` são os endpoints numerados da seção 6.1.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo por notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Problema | Risco comercial: Atlas pode migrar para concorrente se não houver entrega | TRANSCRICAO | [09:00] Marcos |
| PRD-PUB-01 | docs/PRD.md | Público-alvo | Cadastro feito pela API, com JWT do sistema, por usuários que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público-alvo | Administradores reprocessam notificações com falha definitiva | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-01 | docs/PRD.md | Cenário | Cliente filtra só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-01B | docs/PRD.md | Cenário | Cliente consulta o pedido pela API se precisar dos itens | TRANSCRICAO | [09:43] Diego |
| PRD-CEN-02 | docs/PRD.md | Cenário | Endpoint fora do ar por 2 h em manutenção recebe as notificações depois | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Cliente descarta notificação repetida pelo identificador do evento | TRANSCRICAO | [09:25] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário | Cliente rotaciona secret suspeita de vazamento e tem 24 h para migrar | TRANSCRICAO | [09:21] Sofia |
| PRD-CEN-05 | docs/PRD.md | Cenário | Cliente consulta histórico de entregas para investigar notificação | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-06 | docs/PRD.md | Cenário | Administrador reenvia notificação que esgotou as tentativas | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Entrega abaixo de 10 s (definição de tempo real dos clientes) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | 3 de 3 clientes com webhook ativo (meta proposta pelo PRD a partir do pedido) | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Três sprints com revisão de segurança; data a confirmar com a Atlas | TRANSCRICAO | [09:47] Larissa |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Zero notificações perdidas (entregues ou guardadas como falha) | TRANSCRICAO | [09:06] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificar o cliente quando o status de um pedido muda | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL e status; secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Cliente informado no corpo ou no caminho, não deduzido do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar, editar e remover webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Conteúdo da notificação: event_id, tipo, horário, pedido, status, cliente, total, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Notificação assinada para verificar origem e integridade | TRANSCRICAO | [09:19] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com a anterior aceita por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Reenvio automático em intervalos crescentes | TRANSCRICAO | [09:15] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Notificações que esgotam tentativas guardadas com o motivo | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reenvio manual por administrador, registrado com quem pediu | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico dos últimos 100 envios com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Cabeçalhos com identificador do evento e do webhook | TRANSCRICAO | [09:44] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Entrega em menos de 10 s com endpoint saudável | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Registro da notificação atômico com a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Mudança de status não espera nem depende da resposta do cliente | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Entrega pelo menos uma vez | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Ordem por pedido só com um processo de envio; sem ordem global | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Só URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Uma secret por webhook | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Teto de 64 KB, erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s para o cliente responder | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Reenvios cobrem cerca de 15 h de indisponibilidade | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Reprocessamento manual só para administradores | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-12 | docs/PRD.md | Restrição | Sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente após falhas seguidas, adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual, projeto separado do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Ordem global entre pedidos não oferecida | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Webhooks de entrada não existem | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Arquivamento de notificações antigas | TRANSCRICAO | [09:08] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação de prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Fluxo atual de mudança de status, onde o evento é registrado | CODIGO | src/modules/orders/order.service.ts |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso e perda da Atlas | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente processa notificação duplicada | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret por cliente (já ocorreu) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajada de notificações sem rate limiting | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Cliente fora do ar além da janela de 15 h | TRANSCRICAO | [09:17] Marcos |
| PRD-RISK-06 | docs/PRD.md | Risco | Qualquer usuário autenticado gerencia webhooks de qualquer cliente | TRANSCRICAO | [09:37] Sofia |
| PRD-AC-02 | docs/PRD.md | Critério de aceite | Cadastro com URL http recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de aceite | Reenvio manual só por administrador | TRANSCRICAO | [09:36] Larissa |
| PRD-AC-10 | docs/PRD.md | Critério de aceite | Falha no registro desfaz a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-TEST-01 | docs/PRD.md | Estratégia de teste | Teste ponta a ponta previsto na estimativa junto da integração com pedidos | TRANSCRICAO | [09:46] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência | Clientes verificam assinatura e deduplicam eventos | TRANSCRICAO | [09:24] Diego |
| PRD-AC-01 | docs/PRD.md | Critério de aceite | Cadastro com URL https devolve a secret | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-03 | docs/PRD.md | Critério de aceite | Notificação em menos de 10 s para webhooks interessados | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-04 | docs/PRD.md | Critério de aceite | Webhook sem aquele status no filtro não recebe nada | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-05 | docs/PRD.md | Critério de aceite | Assinatura confere com a secret | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de aceite | Reenvio nos intervalos combinados e guarda da falha | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-08 | docs/PRD.md | Critério de aceite | Duas secrets válidas por 24 h após rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-09 | docs/PRD.md | Critério de aceite | Histórico com os últimos 100 envios | TRANSCRICAO | [09:34] Marcos |
| PRD-DEC-01 | docs/PRD.md | Decisão | Registrar junto com a mudança e enviar depois por processo separado | TRANSCRICAO | [09:48] Larissa |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Reenvio por cerca de 15 h e depois ação de administrador | TRANSCRICAO | [09:17] Marcos |
| PRD-DEC-03 | docs/PRD.md | Decisão | Secret por webhook, rotacionável | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Pelo menos uma vez, cliente descarta repetições | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-05 | docs/PRD.md | Decisão | Construir com os padrões existentes | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão | Fotografar o pedido no momento da mudança | TRANSCRICAO | [09:52] Bruno |
| PRD-TEST-02 | docs/PRD.md | Estratégia de teste | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-03 | docs/PRD.md | Estratégia de teste | Marcos confirma o prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-TEST-05 | docs/PRD.md | Estratégia de teste | Marcos atualiza os clientes sobre a entrega | TRANSCRICAO | [09:49] Marcos |
| PRD-TEST-04 | docs/PRD.md | Estratégia de teste | Acompanhar volume por cliente para decidir rate limiting | TRANSCRICAO | [09:39] Diego |
| RFC-META-01 | docs/RFC.md | Escolha de projeto | Revisores são os cinco participantes; Larissa previu revisão do design doc antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-PROP-01 | docs/RFC.md | Decisão | Registro do evento na outbox dentro da transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-05 | docs/RFC.md | Decisão | DLQ com replay manual | TRANSCRICAO | [09:18] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Headers X-Signature, X-Event-Id, X-Webhook-Id, X-Timestamp | TRANSCRICAO | [09:44] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams: infraestrutura nova, overengineering | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger no banco: MySQL sem NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Worker dentro da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido: evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05B | docs/RFC.md | Alternativa descartada | Só 3 tentativas: esgota em cerca de 30 min | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-OQ-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| RFC-OQ-02 | docs/RFC.md | Questão em aberto | Escala além de um worker (particionar ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-OQ-03 | docs/RFC.md | Questão em aberto | E-mail ao cliente em falhas repetidas, próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OQ-04 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-OQ-05 | docs/RFC.md | Questão em aberto | Arquivamento da outbox após ~30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-OQ-06 | docs/RFC.md | Questão em aberto | Ordem por pedido durante o retry | TRANSCRICAO | [09:12] Diego |
| RFC-OQ-07 | docs/RFC.md | Questão em aberto | Escopo de acesso por cliente, customer_id fora do JWT | TRANSCRICAO | [09:32] Bruno |
| RFC-OQ-08 | docs/RFC.md | Questão em aberto | X-Timestamp fora da assinatura | TRANSCRICAO | [09:44] Diego |
| RFC-OQ-09 | docs/RFC.md | Questão em aberto | Contagem das 5 tentativas versus 5 intervalos | TRANSCRICAO | [09:17] Diego |
| RFC-OQ-10 | docs/RFC.md | Questão em aberto | Prazo: fim do trimestre versus fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-RISK-01 | docs/RFC.md | Risco | Transação de changeStatus mais pesada | TRANSCRICAO | [09:04] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Revisão de segurança ainda sem agendamento | TRANSCRICAO | [09:49] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Única mudança em código existente é a chamada em changeStatus | CODIGO | src/modules/orders/order.service.ts |
| RFC-TLDR-01 | docs/RFC.md | Decisão | Resumo: outbox, worker, HMAC, retry, DLQ, at-least-once, sem infra nova | TRANSCRICAO | [09:48] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Streams / fila externa | TRANSCRICAO | [09:07] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Transação de changeStatus atualiza orders, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-POLL | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Polling a cada 2 s aceito | TRANSCRICAO | [09:10] Larissa |
| ADR-002-PRISMA | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-ALT | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Trigger do banco | TRANSCRICAO | [09:09] Bruno |
| ADR-002-ORD | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Ordem só por order_id e com um worker | TRANSCRICAO | [09:13] Larissa |
| ADR-002-ENT | docs/adrs/ADR-002-worker-separado-em-polling.md | Integração | Entry point nos moldes de src/server.ts | CODIGO | src/server.ts |
| ADR-002-ALT2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Worker dentro do processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT3 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Vários workers em paralelo, adiado | TRANSCRICAO | [09:13] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | DLQ como status failed na outbox | TRANSCRICAO | [09:18] Diego |
| ADR-003-SEC | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay exige ADMIN via requireRole existente | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-003-AUD | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay registra em log quem o executou | TRANSCRICAO | [09:36] Sofia |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | Retry indefinido com backoff | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT3 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | Apenas 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-003-TO | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Até cerca de 15 h de atraso aceitável | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-TLS | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | HTTPS obrigatório validado no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CTX | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Histórico de cliente que vazou secret em log | TRANSCRICAO | [09:22] Diego |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para deduplicação | TRANSCRICAO | [09:26] Larissa |
| ADR-005-TO | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar vai para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-ALT | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-DOC | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Dependência | Marcos documenta a deduplicação no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso de AppError, Pino, error middleware, módulos, Zod e códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Integração | Hierarquia AppError e códigos INSUFFICIENT_STOCK / INVALID_STATUS_TRANSITION | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-MW | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Integração | Middleware central já trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-FN | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | publishWebhookEvent(tx, ...) em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa descartada | Bibliotecas e estruturas novas para a feature | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa descartada | Injetar repository de webhook no OrderService | TRANSCRICAO | [09:41] Bruno |
| ADR-006-ID | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Identificadores UUID como no resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Decisão | Filtro por status na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-SNAP | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Decisão | Payload snapshot gravado na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Alternativa descartada | Filtrar na hora do envio | TRANSCRICAO | [09:34] Diego |
| ADR-007-ALT2 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Alternativa descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-PAY | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Decisão | Payload enxuto sem itens; detalhes via GET /orders/:id | TRANSCRICAO | [09:43] Diego |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Evento na outbox na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Nenhuma chamada HTTP no processo da API | TRANSCRICAO | [09:11] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Nenhuma dependência nova de runtime | TRANSCRICAO | [09:29] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Tabela de configuração com url, secret, customer_id e ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Status da outbox: pendente, processando, falhou, entregue; índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Ids UUID Char(36) como o resto do schema | CODIGO | prisma/schema.prisma |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | Tabela webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-05 | docs/FDD.md | Escolha de projeto | Uma linha de outbox por evento e webhook, event_id compartilhado | TRANSCRICAO | [09:44] Sofia |
| FDD-DADOS-06 | docs/FDD.md | Escolha de projeto | Secret em texto porque o HMAC precisa do valor; sugestão de levar o armazenamento à revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | publishWebhookEvent chamado após orderStatusHistory.create | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Só transições válidas chegam ao publisher (canTransition antes) | CODIGO | src/modules/orders/order.status.ts |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Worker usa createPrismaClient() próprio | CODIGO | src/config/database.ts |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Polling de 2 s, lotes pequenos, mais antigos primeiro | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Escolha de projeto | Regra NOT EXISTS para ordem por pedido e webhook | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Timeout de 10 s por envio | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Tabela de backoff e DLQ após a última retentativa | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo | Rotação: anterior válida 24 h, trava contra segunda rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-10 | docs/FDD.md | Fluxo | Recuperação de linhas PROCESSING segura por at-least-once | TRANSCRICAO | [09:24] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks: url, statuses, customerId; secret na resposta | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /customers/:customerId/webhooks | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries, últimos 100 envios | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay, ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Payload de saída com os campos definidos | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Header X-Timestamp com horário do envio | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Rotas sob /api/v1 com envelope de erro atual | CODIGO | src/app.ts |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Corpos da API em camelCase como customerId de orders | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-CONTRATO-13 | docs/FDD.md | Escolha de projeto | PENDING excluído do filtro: nenhuma transição leva a ele | CODIGO | src/modules/orders/order.status.ts |
| FDD-ERR-01 | docs/FDD.md | Erro | Prefixo WEBHOOK_ em todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND e WEBHOOK_INVALID_URL citados como exemplo | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL em details porque validate gera VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE vai direto para DLQ | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT entra no retry | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_OUTBOX_WRITE_FAILED desfaz a mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND precisa de classe própria: NotFoundError fixa NOT_FOUND | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Restart da API não afeta a entrega | TRANSCRICAO | [09:11] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Sem fallback por e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Encerramento por sinais com prisma.$disconnect como no server | CODIGO | src/server.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs Pino, sem ferramenta nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Redact de secret no logger, motivado por vazamento em log | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | redactPaths atuais do logger | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | X-Request-Id existente no request logger | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Log de auditoria do replay com userId | TRANSCRICAO | [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | order.service.ts: chamada em changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-INT-02 | docs/FDD.md | Integração | Rotas novas em buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Variáveis WEBHOOK_* no schema de ambiente | CODIGO | src/config/env.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Scripts worker e start:worker | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-05 | docs/FDD.md | Integração | Node >= 20 permite fetch nativo | CODIGO | package.json |
| FDD-INT-06 | docs/FDD.md | Integração | Build já compila src/**/*.ts em dist | CODIGO | tsconfig.build.json |
| FDD-INT-07 | docs/FDD.md | Integração | Limpeza das tabelas novas no beforeEach dos testes | CODIGO | tests/setup.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Módulo src/modules/webhooks com a estrutura padrão | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-09 | docs/FDD.md | Integração | requireRole('ADMIN') reaproveitado no replay | TRANSCRICAO | [09:36] Larissa |
| FDD-DEP-01 | docs/FDD.md | Dependência | MySQL 8.0 do ambiente local | CODIGO | docker-compose.yml |
| FDD-RISK-01 | docs/FDD.md | Risco | Duas instâncias do worker quebram ordem e duplicam envios | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Crescimento da outbox e do histórico | TRANSCRICAO | [09:08] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Entrega em menos de 10 s com endpoint saudável | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Cliente verifica origem e integridade | TRANSCRICAO | [09:19] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Nenhum evento perdido em silêncio | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND na criação (cliente vem no corpo) | TRANSCRICAO | [09:32] Larissa |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_EMPTY_UPDATE no PATCH sem campos | TRANSCRICAO | [09:33] Bruno |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND no replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-11 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_ROTATION_IN_PROGRESS protege a janela de 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-13 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR e NETWORK_ERROR entram no retry | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-14 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED leva à DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-15 | docs/FDD.md | Erro | Códigos genéricos UNAUTHORIZED e FORBIDDEN do auth existente | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-SEC-01 | docs/FDD.md | Segurança | Revisão de Sofia antes do deploy, com foco em HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| FDD-SEC-02 | docs/FDD.md | Segurança | Assinatura dupla durante a convivência de 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-01 | docs/FDD.md | Critério de aceite | Mudança de status com webhook interessado cria linha PENDING | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-08 | docs/FDD.md | Critério de aceite | Replay: 403 para OPERATOR, 202 para ADMIN com auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-14 | docs/FDD.md | Critério de aceite | Testes na stack vitest e supertest existente | CODIGO | tests/helpers/factories.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Classes Webhook*Error ao lado das existentes em shared/errors | CODIGO | src/shared/errors/index.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Controllers do módulo instanciados em buildControllers | CODIGO | src/app.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Relação webhooks em Customer e migration nova | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
