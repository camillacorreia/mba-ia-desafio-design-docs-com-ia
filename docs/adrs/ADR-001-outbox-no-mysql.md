# ADR-001: Padrão Outbox no MySQL, na mesma transação da mudança de status

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (a transcrição não registra o dia)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-e-filtro-na-insercao.md)

## Contexto

Três clientes B2B querem ser avisados quando o status dos pedidos deles muda, em vez de consultar `GET /orders` periodicamente ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que abre uma transação Prisma (`this.prisma.$transaction`) e, dentro dela, valida a transição com `canTransition` (`src/modules/orders/order.status.ts`), debita ou repõe estoque, atualiza `orders` e insere em `order_status_history`. Bruno descreveu essa transação como "já pesada" ([09:04] Bruno).

O problema a resolver é como garantir que toda mudança de status gere a notificação correspondente sem colocar uma chamada HTTP a um sistema de terceiro dentro dessa transação.

## Decisão

Usar o padrão Outbox no MySQL que a aplicação já usa. Dentro da mesma transação de `changeStatus`, além de atualizar `orders` e inserir em `order_status_history`, o serviço insere o evento numa tabela `webhook_outbox` ([09:06] Diego). Um worker separado lê essa tabela e faz as chamadas HTTP (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).

Se a inserção na outbox falhar, a transação inteira faz rollback: não pode existir mudança de status para a qual algum webhook ativo tenha pedido evento sem que o evento fique registrado ([09:40] Bruno, [09:41] Diego). Quando nenhum webhook do cliente quer aquele status, nada é gravado ([ADR-007](ADR-007-snapshot-e-filtro-na-insercao.md)). A tabela tem índice em `status` (pendente, processando, falhou, entregue) e na data de criação (`created_at` na fala de Diego, `createdAt` no padrão do schema), e o worker lê os pendentes em lotes pequenos ([09:08] Diego). Decisão fechada por Larissa em [09:08].

## Alternativas consideradas

**Disparo síncrono dentro de `changeStatus`.** Descartado por Bruno, Larissa e Diego ([09:04] Bruno, [09:06] Diego). Um cliente lento travaria a mudança de status de outros pedidos, e um cliente fora do ar deixaria duas saídas ruins: fazer rollback da mudança de status ou perder a notificação.

**Redis Streams (ou fila equivalente).** Levantado por Larissa em [09:07]. Descartado porque exige subir infraestrutura nova, e Diego classificou subir um Redis Cluster para isso como overengineering para um time pequeno ([09:07] Diego).

## Consequências

**Positivas**

- Atomicidade: se a transação commitou, o evento existe; se fez rollback, o evento some junto ([09:06] Diego). Não há estado intermediário.
- Nenhuma infraestrutura nova: mesmo MySQL, mesmo Prisma, mesmas migrations em `prisma/`.
- A mudança de status deixa de depender da disponibilidade do cliente.

**Negativas**

- A transação de `changeStatus` ganha mais escritas (uma linha por webhook interessado no evento) e a latência dela cresce um pouco.
- A tabela cresce continuamente. O arquivamento de linhas entregues "depois de 30 dias ou assim" foi citado como fora do escopo desta feature ([09:08] Diego), então o crescimento fica sem tratamento até essa tarefa existir.
- A entrega deixa de ser imediata: passa a depender do ciclo do worker (2 segundos no pior caso, [09:10] Larissa).

**Trade-off explícito:** troca-se latência mínima e simplicidade de "chamar e pronto" por consistência garantida entre o status do pedido e o evento, sem pagar o custo operacional de uma fila externa.
