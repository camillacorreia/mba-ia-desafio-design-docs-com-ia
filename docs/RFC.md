# RFC: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Autora | Camilla Correia, a partir da reunião técnica conduzida por Larissa |
| Status | Em revisão |
| Data | 17/11/2026 |
| Revisores | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenheira de Segurança) |
| Documentos | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo (TL;DR)

Propomos notificar os clientes B2B por webhooks de saída sempre que o status de um pedido mudar. A mudança de status passa a gravar o evento numa tabela de outbox no MySQL, dentro da mesma transação que já atualiza o pedido. Um worker em processo separado lê essa tabela a cada 2 segundos e faz a chamada HTTP ao cliente, assinada com HMAC-SHA256 e uma secret por endpoint. Falhas são retentadas com backoff exponencial e, esgotadas as tentativas, vão para uma DLQ com replay manual por administrador. A entrega é at-least-once, com `X-Event-Id` para o cliente deduplicar. Nenhuma infraestrutura nova é necessária, e o módulo segue os padrões que o projeto já tem.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para serem avisados quando o status dos pedidos muda. Hoje elas consultam `GET /orders` de tempos em tempos, o que deixa a integração lenta e cara, e a Atlas sinalizou que pode migrar para um concorrente se a entrega não sair a tempo ([09:00] Marcos). Para esses clientes, "tempo real" é qualquer coisa abaixo de 10 segundos ([09:02] Marcos).

A aplicação não tem hoje nenhum mecanismo de evento, fila ou notificação externa. A mudança de status vive em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), numa transação que valida a transição, mexe no estoque, atualiza `orders` e grava `order_status_history`. Qualquer solução precisa se encaixar nesse ponto sem deixar a transação refém de um sistema de terceiro ([09:04] Bruno).

## 3. Proposta técnica

A solução tem quatro partes.

**Registro do evento.** Em `changeStatus`, depois de gravar o histórico, uma função do novo módulo de webhooks, que recebe a transação corrente, verifica quais webhooks ativos do cliente querem aquele status e grava, para cada um, uma linha na `webhook_outbox` com o payload já renderizado. Tudo na mesma transação: se a gravação falhar, a mudança de status também é desfeita ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md)).

**Entrega.** Um processo novo (`npm run worker` em desenvolvimento), na mesma base de código e no mesmo banco, consulta a outbox a cada 2 segundos, em lotes pequenos e em ordem de criação. Para cada evento, faz um `POST` na URL do cliente com timeout de 10 segundos ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

**Falha e recuperação.** Resposta fora de 2xx, erro de rede ou timeout reagendam o evento com backoff de 1 min, 5 min, 30 min, 2 h e 12 h. Depois disso, o evento vai para a tabela `webhook_dead_letter`, de onde um administrador pode reenviá-lo por `POST /admin/webhooks/dead-letter/:id/replay` ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).

**Segurança e contrato com o cliente.** Cada webhook tem uma secret própria, gerada pela plataforma e rotacionável com 24 horas de convivência. O corpo é assinado com HMAC-SHA256 e a assinatura vai em `X-Signature`, junto com `X-Event-Id`, `X-Webhook-Id` e `X-Timestamp`. Só URLs `https` são aceitas e o payload tem teto de 64 KB ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).

A gestão dos webhooks é uma API REST autenticada: cadastrar, listar por cliente, editar, remover, rotacionar a secret e consultar o histórico de entregas. Contratos, modelo de dados e fluxos detalhados estão no [FDD](FDD.md).

```
PATCH /orders/:id/status
        │
        ▼
OrderService.changeStatus ── transação ──┬─ orders / order_status_history / estoque
                                         └─ publishWebhookEvent → webhook_outbox
                                                                      │ polling 2 s
                                                                      ▼
                                              worker (processo separado) ── HTTPS + HMAC ──▶ cliente
                                                    │ esgotou o retry
                                                    ▼
                                              webhook_dead_letter ── replay (ADMIN) ──▶ webhook_outbox
```

## 4. Alternativas consideradas

| Alternativa | Por que foi descartada | Origem |
|---|---|---|
| Disparar o HTTP de forma síncrona dentro de `changeStatus` | Cliente lento travaria a mudança de status de outros pedidos, e cliente fora do ar obrigaria a escolher entre rollback do status e perda da notificação | [09:04] Bruno, [09:06] Diego |
| Redis Streams ou outra fila externa | Exige subir infraestrutura nova; para um time pequeno, um Redis Cluster seria overengineering | [09:07] Larissa, [09:07] Diego |
| Trigger no banco para acordar o worker | MySQL não tem `NOTIFY/LISTEN`; a trigger só executa SQL e avisar um processo externo exigiria improvisos | [09:09] Bruno, [09:09] Diego |
| Worker dentro do processo da API | Um restart da API derrubaria o processamento | [09:11] Diego |
| Retry indefinido, ou só 3 tentativas | Indefinido deixa eventos pendurados para sempre; 3 tentativas se esgotam em cerca de 30 minutos, menos que indisponibilidades já vistas | [09:15] Diego, [09:16] Diego |
| Secret global da plataforma | Um vazamento exporia todos os clientes | [09:21] Sofia |
| Garantia exactly-once | Exigiria coordenação dos dois lados e muito mais complexidade | [09:25] Diego |

## 5. Questões em aberto

1. **Rate limiting de saída.** Um cliente com 50 pedidos mudando de status em um minuto recebe 50 chamadas. A decisão foi não implementar agora, observar e decidir depois ([09:38] Diego, [09:39] Larissa). Falta definir o que vamos medir para saber se virou problema.
2. **Escala além de um worker.** A ordem por pedido só vale com um worker. Particionar por `order_id` ou usar lock pessimista ficou para o futuro ([09:13] Diego, [09:13] Larissa).
3. **Aviso ao cliente quando o webhook falha repetidamente.** E-mail após falhas seguidas foi adiado para uma próxima fase, depois de medir o impacto ([09:37] Marcos, [09:37] Larissa).
4. **Permissões do CRUD de configuração.** Por enquanto qualquer papel autenticado pode gerenciar webhooks; Sofia indicou que isso pode ser endurecido mais adiante ([09:37] Sofia).
5. **Arquivamento da outbox.** Linhas entregues seriam arquivadas "depois de 30 dias ou assim", mas isso ficou fora desta feature ([09:08] Diego).
6. **Ordem por pedido durante o retry.** A ordem por `order_id` foi prometida para o cenário de um worker ([09:12] Diego), mas a reunião não tratou o caso em que um evento em backoff é ultrapassado por um evento posterior do mesmo pedido. O FDD propõe segurar os eventos seguintes do mesmo pedido e webhook até o anterior sair do ciclo de retry. Pedimos confirmação de Diego, porque isso atrasa os eventos seguintes quando o cliente está fora do ar.
7. **Escopo de acesso por cliente.** O `customer_id` vem no body ou no path, e não do JWT, porque os usuários atuais são operadores ([09:32] Bruno, [09:32] Larissa). Com qualquer papel autenticado podendo gerenciar webhooks ([09:37] Sofia), um usuário consegue operar os webhooks de qualquer cliente. Isso se conecta ao "endurecer mais pra frente" do item 4.
8. **Proteção contra replay pelo `X-Timestamp`.** Diego incluiu `X-Timestamp` para o cliente "detectar replay attack se quiser" ([09:44] Diego), mas a assinatura cobre só o corpo ([09:22] Sofia), então o header pode ser alterado sem invalidar a assinatura. Por ora a proteção é a deduplicação por `X-Event-Id`. Incluir o timestamp na assinatura é uma decisão para Sofia.
9. **Contagem das tentativas.** A reunião fala em "5 tentativas" com cinco intervalos que somam cerca de 15 horas "entre primeira falha e última tentativa" ([09:17] Diego). O FDD adota um envio inicial mais 5 retentativas, única leitura em que a soma fecha. Pedimos confirmação de Diego e Larissa.
10. **Prazo.** Marcos falou em "fim do trimestre" no início ([09:00] Marcos) e em "fim de novembro" ao discutir prazo ([09:45] Marcos). A estimativa fechada foi de três sprints com a revisão de segurança incluída ([09:47] Larissa), mas os blocos listados por Larissa já somam três sprints antes de "HMAC, schemas, validações, mais um pouco" e dos dois dias úteis de revisão pedidos por Sofia ([09:46] Larissa, [09:46] Sofia). Marcos ficou de confirmar a data com a Atlas ([09:47] Marcos).

## 6. Impacto e riscos

**Impacto no sistema atual.** A única mudança em código existente com efeito em produção é a chamada ao módulo de webhooks dentro de `changeStatus`, que acrescenta leituras e escritas à transação. O resto é aditivo: um módulo novo com suas rotas, um entry point para o worker e migrations novas.

**Impacto operacional.** Passa a existir um segundo processo para implantar e monitorar, que precisa rodar com exatamente uma instância enquanto a ordem por pedido for uma garantia.

Os riscos de produto (prazo da Atlas, duplicatas no cliente, vazamento de secret) estão no [PRD](PRD.md#10-riscos-e-mitigação). Aqui ficam os de arquitetura:

| Risco | Efeito | Mitigação proposta |
|---|---|---|
| Transação de `changeStatus` mais pesada | Mudança de status mais lenta, numa transação que já é pesada ([09:04] Bruno) | Uma consulta indexada aos webhooks do cliente e uma única escrita em lote ([ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md)) |
| Duas instâncias do worker por engano | Envios em dobro e ordem quebrada | Restrição a registrar no procedimento de deploy; a partida do worker fica em log |
| Cliente indisponível por muito tempo | Eventos do mesmo pedido ficam retidos atrás do que está em retry, por até cerca de 15 horas | Comportamento documentado; ao ir para a DLQ, o evento libera os seguintes ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)) |
| Crescimento contínuo da outbox | Consultas do worker mais lentas com o tempo | Índices por status e data; arquivamento fica como trabalho futuro ([09:08] Diego) |
| Revisão de segurança fora do cronograma | Deploy atrasado ou sem revisão | Dois dias úteis reservados na estimativa ([09:47] Larissa); o agendamento ainda está pendente ([09:49] Sofia) |

## 7. Decisões relacionadas

- [ADR-001: Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker em processo separado com polling de 2 segundos](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005: At-least-once com `X-Event-Id`](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007: Snapshot e filtro na inserção](adrs/ADR-007-snapshot-e-filtro-na-insercao.md)
