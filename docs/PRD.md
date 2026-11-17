# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Autora | Camilla Correia |
| Product Manager | Marcos |
| Status | Aprovado em reunião técnica, em detalhamento |
| Data | 17/11/2026 |
| Documentos | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O OMS vai avisar os clientes B2B, por webhook, sempre que o status de um pedido deles mudar. O cliente cadastra uma URL `https`, escolhe de quais status quer saber e passa a receber uma requisição assinada a cada mudança, sem precisar consultar a API.

A demanda veio de um pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos). Hoje o OMS não tem nenhum mecanismo de notificação externa; a única forma de saber se um pedido mudou é consultar `GET /orders`.

## 2. Problema e motivação

Os três clientes acompanham seus pedidos consultando `GET /orders` de tempos em tempos, e isso está deixando a integração lenta e cara para eles ([09:00] Marcos). A mudança só é percebida na próxima consulta, e o cliente fica tendo que atualizar manualmente ([09:02] Marcos).

A motivação é também comercial: a Atlas sinalizou que pode migrar para um concorrente se a funcionalidade não for entregue no prazo ([09:00] Marcos).

## 3. Público-alvo e cenários de uso

**Público principal:** sistemas dos clientes B2B que compram pela plataforma, começando por Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos). O cadastro é feito pela API do OMS, autenticado com o JWT do sistema, por usuários que representam o cliente ([09:32] Marcos).

**Público secundário:** administradores da plataforma, que reprocessam notificações que falharam de vez ([09:36] Sofia).

| ID | Cenário |
|---|---|
| PRD-CEN-01 | Um cliente cadastra um webhook só para os status `SHIPPED` e `DELIVERED`. Quando um pedido dele é despachado, o sistema do cliente recebe a notificação em segundos e, se precisar dos itens do pedido, consulta o pedido pela API ([09:33] Marcos, [09:43] Diego) |
| PRD-CEN-02 | O endpoint de um cliente fica duas horas fora do ar em manutenção planejada, como já aconteceu com um cliente da plataforma. As notificações desse período são reenviadas automaticamente quando ele volta ([09:16] Diego) |
| PRD-CEN-03 | Um cliente recebe a mesma notificação duas vezes e descarta a repetida pelo identificador do evento ([09:25] Diego) |
| PRD-CEN-04 | Um cliente suspeita que a secret vazou, pede uma nova pela API e tem 24 horas para trocar nos sistemas dele ([09:21] Sofia, [09:22] Diego) |
| PRD-CEN-05 | Um cliente diz que não recebeu uma notificação e consulta o histórico de entregas do webhook para ver o que aconteceu ([09:34] Marcos) |
| PRD-CEN-06 | Uma notificação esgota as tentativas; depois que o cliente corrige o endpoint, um administrador manda reenviar ([09:18] Diego) |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| PRD-OBJ-01 | Notificar em "tempo real" na percepção do cliente | Tempo entre a mudança de status e a entrega ao cliente, com endpoint saudável | Abaixo de 10 segundos em todas as entregas que acertam na primeira tentativa | [09:02] Marcos, [09:09] Diego |
| PRD-OBJ-02 | Atender os clientes que pediram a funcionalidade | Clientes com webhook ativo em produção | 3 de 3 (Atlas, MaxDistribuição, Nova Cargo); meta proposta por este PRD a partir do pedido dos três | [09:00] Marcos |
| PRD-OBJ-03 | Entregar a tempo de reter a Atlas | Funcionalidade em produção | Três sprints com revisão de segurança; a data (fim de novembro ou fim do trimestre) depende da confirmação de Marcos com a Atlas, ver [RFC, questão 10](RFC.md#5-questões-em-aberto) | [09:45] Marcos, [09:47] Larissa |
| PRD-OBJ-04 | Não perder notificações | Mudanças de status com webhook interessado que não foram entregues nem ficaram guardadas como falha definitiva, excluídos os webhooks pausados ou removidos pelo próprio cliente | 0 | [09:06] Diego, [09:18] Diego |

Medir a redução de consultas a `GET /orders` por esses clientes seria um indicador natural do PRD-OBJ-01, mas a reunião não definiu meta para isso; fica como acompanhamento, sem alvo.

## 5. Escopo

### 5.1 Incluso

- Notificação de saída a cada mudança de status de pedido, só da plataforma para o cliente ([09:02] Marcos).
- Cadastro, listagem, edição e remoção de webhooks por cliente, com filtro de status ([09:31] Marcos, [09:33] Bruno).
- Assinatura de cada notificação, com secret por webhook e rotação ([09:22] Sofia).
- Reenvio automático em caso de falha e fila de falhas definitivas com reprocessamento manual ([09:17] Larissa, [09:18] Diego).
- Histórico das últimas entregas por webhook ([09:34] Marcos).

### 5.2 Fora de escopo

| Item | Situação | Origem |
|---|---|---|
| Aviso por e-mail ao cliente quando o webhook falha repetidamente | Adiado para a próxima fase, depois de medir o impacto | [09:37] Larissa, [09:38] Marcos |
| Dashboard visual para o cliente acompanhar os webhooks | Fora de escopo; é projeto separado do time de frontend | [09:40] Larissa |
| Limite de taxa de envio por cliente (rate limiting) | Não entra agora; observar e decidir depois | [09:39] Diego, [09:39] Larissa |
| Garantia de ordem global entre pedidos | Não oferecida; os clientes não pediram | [09:13] Larissa, [09:14] Marcos |
| Entrega exatamente uma vez (exactly-once) | Descartada; a garantia é pelo menos uma vez | [09:25] Diego |
| Webhooks enviados pelo cliente para a plataforma | Não existe esse fluxo | [09:02] Marcos |
| Arquivamento de notificações antigas | Fora desta funcionalidade | [09:08] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | Notificar o cliente quando o status de um pedido dele muda | [09:00] Marcos |
| PRD-FR-02 | Permitir cadastrar um webhook informando a URL e os status de interesse; a plataforma gera a secret e a devolve na criação | [09:31] Marcos |
| PRD-FR-03 | O cliente dono do webhook é informado na requisição (corpo ou caminho), e não deduzido do login | [09:32] Larissa |
| PRD-FR-04 | Permitir listar os webhooks de um cliente, editar e remover um webhook | [09:33] Bruno |
| PRD-FR-05 | Cada webhook recebe só as mudanças para os status que escolheu | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-06 | Cada notificação leva identificador do evento, tipo do evento (mudança de status), horário, pedido (id e número), status anterior e novo, cliente e valor total, sem os itens | [09:43] Diego |
| PRD-FR-07 | Cada notificação é assinada para o cliente conferir origem e integridade | [09:19] Sofia, [09:20] Sofia |
| PRD-FR-08 | O cliente pode pedir uma secret nova; a anterior continua aceita por 24 horas | [09:21] Sofia |
| PRD-FR-09 | Notificações que falham são reenviadas automaticamente em intervalos crescentes | [09:15] Diego, [09:17] Larissa |
| PRD-FR-10 | Notificações que esgotam as tentativas ficam guardadas com o motivo da falha | [09:18] Diego |
| PRD-FR-11 | Administradores podem mandar reenviar uma notificação guardada, e cada reenvio fica registrado com quem o pediu | [09:18] Diego, [09:36] Sofia |
| PRD-FR-12 | O cliente consulta os últimos 100 envios de um webhook, com resultado, conteúdo enviado, resposta recebida e tempo de resposta | [09:34] Marcos |
| PRD-FR-13 | Cada notificação traz identificador do evento e do webhook em cabeçalhos, para deduplicação e para saber qual cadastro gerou o envio | [09:25] Diego, [09:44] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-NFR-01 | Entrega em menos de 10 segundos com o endpoint do cliente saudável | [09:02] Marcos |
| PRD-NFR-02 | Toda mudança de status para a qual algum webhook ativo do cliente pediu aviso gera notificação registrada junto com a mudança; se o registro falhar, a mudança de status não acontece | [09:40] Bruno |
| PRD-NFR-03 | A mudança de status de um pedido não pode esperar nem depender da resposta do cliente | [09:04] Bruno |
| PRD-NFR-04 | Garantia de entrega pelo menos uma vez; o cliente deve tratar repetições | [09:24] Diego |
| PRD-NFR-05 | Mudanças de um mesmo pedido chegam na ordem em que aconteceram, enquanto houver um único processo de envio; não há garantia de ordem entre pedidos diferentes. É uma limitação conhecida, e o comportamento durante reenvios está em confirmação ([RFC, questão 6](RFC.md#5-questões-em-aberto)) | [09:12] Diego, [09:13] Larissa |
| PRD-NFR-06 | Só URLs `https` são aceitas | [09:23] Sofia |
| PRD-NFR-07 | Uma secret por webhook, nunca compartilhada entre clientes | [09:21] Sofia |
| PRD-NFR-08 | Notificações acima de 64 KB não são enviadas e são tratadas como erro | [09:24] Larissa |
| PRD-NFR-09 | O cliente tem até 10 segundos para responder cada notificação | [09:42] Diego |
| PRD-NFR-10 | Reenvios cobrem até cerca de 15 horas de indisponibilidade do cliente | [09:17] Diego |
| PRD-NFR-11 | Reprocessamento manual restrito a administradores | [09:36] Sofia |
| PRD-NFR-12 | Sem infraestrutura nova: a solução usa o banco e a stack atuais | [09:07] Diego, [09:30] Larissa |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Registrar a notificação junto com a mudança de status e enviar depois, por um processo separado | Até 2 segundos a mais de espera, em troca de nunca perder notificação nem travar pedidos | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| Reenviar até cerca de 15 horas e depois exigir ação de um administrador | Notificações muito atrasadas dependem de intervenção humana | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Assinar com uma secret por webhook, rotacionável | Mais gestão de secrets do lado da plataforma | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| Garantir entrega pelo menos uma vez | O cliente precisa descartar repetições | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Construir com os padrões que o produto já usa | Sem ferramentas novas de monitoramento nesta fase | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Fotografar o pedido no momento da mudança | Editar o filtro de um webhook só vale para mudanças futuras | [ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md) |

## 9. Dependências

| Dependência | Tipo | Origem |
|---|---|---|
| Revisão de segurança de pelo menos dois dias úteis antes do deploy, com foco em assinatura e geração de secret; agendamento ainda pendente | Interna, Sofia | [09:46] Sofia, [09:49] Sofia |
| Documentação no portal do desenvolvedor, com destaque para a deduplicação | Interna, Marcos | [09:26] Marcos, [09:40] Marcos |
| Confirmação de prazo com a Atlas | Externa, Marcos | [09:47] Marcos |
| Clientes preparados para verificar assinatura e deduplicar eventos | Externa, clientes | [09:20] Sofia, [09:24] Diego |
| Fluxo atual de mudança de status de pedidos, onde a notificação é registrada | Técnica, código existente | [09:40] Bruno |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RISK-01 | Atraso além do prazo e perda da Atlas para um concorrente | Média: a própria estimativa de três sprints não tem folga | Alto | Escopo fechado (sem e-mail, dashboard ou rate limiting), revisão de segurança reservada na estimativa, prazo a confirmar com a Atlas ([09:47] Marcos) |
| PRD-RISK-02 | Cliente processa a mesma notificação duas vezes | Média | Médio | Identificador do evento em toda notificação e orientação em destaque no portal ([09:26] Marcos) |
| PRD-RISK-03 | Vazamento de secret por um cliente | Média: já aconteceu antes ([09:22] Diego) | Alto | Secret por webhook e rotação com 24 horas de convivência |
| PRD-RISK-04 | Cliente com muitos pedidos mudando ao mesmo tempo recebe uma rajada de notificações | Média | Médio | Monitorar o volume por cliente e decidir sobre limite de taxa com dados ([09:39] Larissa) |
| PRD-RISK-05 | Cliente fora do ar por mais de 15 horas perde a janela de reenvio automático | Baixa | Médio | Notificação fica guardada e pode ser reenviada por um administrador ([09:18] Diego) |
| PRD-RISK-06 | Qualquer usuário autenticado consegue gerenciar webhooks de qualquer cliente | Média | Médio | Aceito por enquanto, com endurecimento previsto mais adiante ([09:37] Sofia) |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-AC-01 | Um cliente cadastra um webhook com URL `https` e recebe a secret na resposta |
| PRD-AC-02 | Um cadastro com URL `http` é recusado |
| PRD-AC-03 | Ao mudar o status de um pedido, cada webhook ativo do cliente que escolheu aquele status recebe uma notificação em menos de 10 segundos, quando o endpoint do cliente está respondendo |
| PRD-AC-04 | Webhooks que não escolheram aquele status não recebem nada |
| PRD-AC-05 | A assinatura de toda notificação confere com a secret do webhook |
| PRD-AC-06 | Com o endpoint do cliente fora do ar, a notificação é reenviada nos intervalos combinados e, esgotadas as tentativas, fica guardada com o motivo |
| PRD-AC-07 | Um administrador consegue reenviar uma notificação guardada; um usuário sem papel de administrador não consegue |
| PRD-AC-08 | Depois de pedir uma secret nova, o cliente consegue validar com qualquer uma das duas por 24 horas |
| PRD-AC-09 | O histórico mostra os últimos 100 envios do webhook, com resultado e tempo de resposta |
| PRD-AC-10 | Se o registro da notificação falhar, a mudança de status do pedido também não acontece |

## 12. Estratégia de testes e validação

- **Testes automatizados** na mesma base de testes que o produto já tem: API de gestão, registro da notificação junto com a mudança de status (inclusive o caso de falha), assinatura, reenvio e reprocessamento. O detalhamento técnico está na seção 13 do [FDD](FDD.md#13-critérios-de-aceite-técnicos).
- **Teste ponta a ponta** com um receptor HTTP de teste fazendo o papel do cliente, previsto na estimativa de Larissa junto com a integração em pedidos ([09:46] Larissa).
- **Revisão de segurança** de Sofia, com pelo menos dois dias úteis, antes do deploy ([09:46] Sofia).
- **Validação com clientes:** Marcos confirma o prazo com a Atlas ([09:47] Marcos) e atualiza os clientes ([09:49] Marcos). A meta de 3 de 3 clientes ativos (PRD-OBJ-02), proposta por este PRD, fecha a validação.
- **Acompanhamento pós-lançamento:** tempo de entrega (PRD-OBJ-01), volume de falhas guardadas e volume de notificações por cliente, este último para a decisão adiada sobre limite de taxa ([09:39] Larissa).
