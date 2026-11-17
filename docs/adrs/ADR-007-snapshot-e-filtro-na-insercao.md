# ADR-007: Payload renderizado (snapshot) e filtro por status no momento da inserção na outbox

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (o snapshot foi decidido depois da saída de Marcos e Sofia)
- **Decisores:** Bruno (Pedidos), Diego (Plataforma), Larissa (Tech Lead)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [FDD, seção 5.1](../FDD.md#51-criação-do-evento-na-outbox)

## Contexto

Cada webhook escolhe os status que quer ouvir, por exemplo só `SHIPPED` e `DELIVERED` ([09:33] Marcos). Isso abre duas perguntas sobre a outbox: em que momento aplicar esse filtro, e o que gravar na linha, se o payload pronto ou só a referência ao pedido.

## Decisão

1. **Filtrar na inserção.** Se nenhum webhook do cliente quer aquele status, nada é inserido na outbox ([09:34] Bruno, com concordância de Diego).
2. **Gravar o payload renderizado (snapshot) na inserção.** O evento reflete o estado do pedido no momento da mudança de status, mesmo que o pedido mude depois ([09:52] Larissa, [09:52] Diego, fechado por Bruno em [09:52]).

O conteúdo do snapshot é o payload enxuto definido em [09:43] por Diego: `event_id`, `event_type` (`order.status_changed`), `timestamp` em ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`, sem os itens. Quem precisar de detalhes consulta `GET /orders/:id` ([09:43] Diego).

## Alternativas consideradas

**Filtrar na hora do envio.** Levantada por Diego em [09:34]. Preterida porque gravaria linhas que nunca seriam enviadas ([09:34] Bruno).

**Guardar só o `order_id` e renderizar no envio.** Levantada por Bruno em [09:51]. Descartada porque, num retry horas depois, o payload mostraria o estado atual do pedido e não o da mudança, um "caso esquisito" ([09:52] Larissa).

## Consequências

**Positivas**

- A outbox só contém trabalho real.
- O conteúdo enviado é estável entre tentativas, o que combina com a deduplicação por `event_id` ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)): duas entregas do mesmo evento têm o mesmo corpo.
- O worker não precisa consultar pedidos, só a outbox.

**Negativas**

- Mudar o filtro de um webhook (`PATCH`) só vale para mudanças de status futuras; eventos já gravados seguem como estavam.
- A transação de `changeStatus` passa a ler os webhooks ativos do cliente e a montar o JSON, um custo extra dentro da transação que Bruno já considerava pesada ([09:04] Bruno).
- O payload fica duplicado em cada linha da outbox (uma por webhook interessado).

**Trade-off explícito:** paga-se um pouco mais dentro da transação de mudança de status para ter uma outbox enxuta e eventos que não mudam entre tentativas.
