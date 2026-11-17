# ADR-005: Entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), com ressalva registrada por Sofia e aceite de Marcos para comunicação aos clientes
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Contexto

Com outbox, worker e retry, existem situações em que o mesmo evento é enviado mais de uma vez: o cliente processa e a resposta se perde, a resposta passa do timeout de 10 segundos ([09:42] Diego), ou o worker cai depois de enviar e antes de marcar o evento como entregue. Era preciso definir a garantia oferecida aos clientes.

## Decisão

A plataforma garante **at-least-once**: o cliente pode receber o mesmo evento duas vezes e precisa estar preparado ([09:24] Diego). Cada evento recebe um UUID gerado no momento em que entra na outbox, enviado no header `X-Event-Id` e único por evento; o cliente deduplica por esse valor ([09:25] Diego). Decisão fechada por Larissa em [09:26].

O `event_id` também vai no corpo do payload ([09:43] Diego), e os identificadores seguem o padrão UUID do restante do projeto ([09:51] Larissa; ver `@default(uuid())` em `prisma/schema.prisma`).

## Alternativas consideradas

**Exactly-once.** Descartado porque exigiria coordenação dos dois lados e ficaria muito mais complexo; at-least-once com `event_id` resolve a grande maioria dos casos e é o modelo usado por Stripe e GitHub ([09:25] Diego).

## Consequências

**Positivas**

- O worker pode reenviar sem medo sempre que estiver em dúvida (timeout, queda, retry), o que simplifica a recuperação.
- Modelo conhecido pelos clientes, por ser o padrão de mercado.

**Negativas**

- Joga responsabilidade para o cliente, que precisa guardar os `event_id` já processados ([09:25] Sofia). Marcos se comprometeu a documentar isso em destaque no portal do desenvolvedor ([09:26] Marcos).
- Cliente que não deduplica pode aplicar o mesmo evento duas vezes.

**Trade-off explícito:** troca-se a garantia mais forte (exactly-once) por um sistema mais simples e robusto, ao custo de exigir idempotência do lado do cliente.
