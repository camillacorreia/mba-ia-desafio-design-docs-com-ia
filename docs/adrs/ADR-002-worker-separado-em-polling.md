# ADR-002: Worker em processo separado, lendo a outbox por polling a cada 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), com aceite de Marcos (PM) sobre a latência
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Com a outbox decidida ([ADR-001](ADR-001-outbox-no-mysql.md)), falta definir quem lê a tabela e como esse leitor fica sabendo que há eventos novos. Os clientes consideram "tempo real" qualquer coisa abaixo de 10 segundos ([09:02] Marcos). Hoje a aplicação tem um único entry point, `src/server.ts`, que sobe o Express e trata `SIGINT`/`SIGTERM` com `prisma.$disconnect()`.

## Decisão

1. **Polling em loop a cada 2 segundos.** O worker busca os eventos pendentes mais antigos, processa e marca ([09:09] Diego). A latência adicional fica em até 2 segundos no pior caso, o que foi aceito ([09:10] Marcos, [09:10] Larissa).
2. **Processo separado da API.** Se a API reinicia, o worker não pode cair junto ([09:11] Diego). Ele ganha um entry point próprio, `src/worker.ts`, e um script `npm run worker`, no mesmo modelo de `src/server.ts` ([09:11] Larissa). O FDD acrescenta `npm run start:worker` para rodar a versão compilada, como o `start` da API. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
3. **Mesmo banco, outro PrismaClient.** O worker usa a mesma `DATABASE_URL`, mas instancia o próprio `PrismaClient`, porque o client é por processo ([09:30] Bruno). A fábrica `createPrismaClient()` de `src/config/database.ts` já serve para isso.
4. **Um único worker.** Com uma instância só, os eventos saem em ordem de criação na outbox, e o cliente recebe as mudanças de um mesmo pedido em ordem ([09:12] Diego). A garantia de ordem é por `order_id` e só vale enquanto houver um worker; não há ordem global ([09:13] Larissa).

A reunião não discutiu a interação entre essa ordem e o retry ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)): um evento esperando o intervalo de 30 minutos pode ser ultrapassado por um evento posterior do mesmo pedido. Para manter a promessa de ordem por `order_id`, o FDD propõe que o worker não envie um evento enquanto houver um evento mais antigo do mesmo pedido, para o mesmo webhook, ainda não entregue nem enviado à DLQ. Essa regra é uma derivação desta decisão e está listada como questão em aberto no [RFC](../RFC.md#5-questões-em-aberto).

## Alternativas consideradas

**Trigger do banco para reagir ao evento.** Levantada por Bruno em [09:09]. Descartada porque o MySQL não tem um mecanismo como o `NOTIFY/LISTEN` do Postgres: a trigger executa SQL, mas não avisa um processo externo. Avisar o worker exigiria improvisos como escrever em arquivo ou chamar um endpoint ([09:09] Diego).

**Rodar o worker dentro do processo da API.** Descartada porque um restart da API derrubaria o processamento junto ([09:11] Diego).

**Vários workers em paralelo.** Não foi adotada agora. Escalar exigiria particionar por `order_id` ou usar lock pessimista, e isso ficou como "problema do futuro" ([09:13] Diego).

## Consequências

**Positivas**

- Atende o requisito de menos de 10 segundos com folga ([09:09] Diego).
- Nenhum componente novo além de um segundo processo Node da mesma base de código.
- Ordem por pedido preservada enquanto houver um worker, com uma regra simples na consulta do worker: um evento só sai depois que o anterior do mesmo pedido e webhook saiu do ciclo de retry.

**Negativas**

- Consulta periódica ao banco mesmo quando não há eventos.
- A garantia de ordem depende de nunca subir duas instâncias do worker. Isso vira uma restrição de deploy que precisa estar documentada, porque duas instâncias também processariam o mesmo evento em dobro.
- Throughput limitado a um processo. Quando não bastar, será preciso revisitar esta decisão (particionamento ou lock).
- O replay manual de um item da DLQ ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) reenvia um evento antigo depois de eventos mais novos do mesmo pedido já entregues. Nesse caso a ordem quebra por desenho, e o cliente se orienta pelo `timestamp` do payload.
- Com a regra de ordem por pedido aplicada ao retry, um endpoint indisponível segura os eventos seguintes daquele pedido até o mais antigo ser entregue ou ir para a DLQ (até cerca de 15 horas).

**Trade-off explícito:** aceita-se até 2 segundos de latência extra e um teto de vazão de um processo em troca de não depender de mecanismos que o MySQL não oferece e de conseguir ordem por pedido sem coordenação entre processos.
