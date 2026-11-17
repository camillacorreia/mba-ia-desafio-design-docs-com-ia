# ADR-003: Retry com backoff exponencial em 5 tentativas e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Sofia (Segurança) no controle de acesso do replay
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

O cliente pode estar fora do ar ou lento no momento do envio ([09:14] Larissa). A equipe já teve cliente com duas horas de indisponibilidade em manutenção planejada ([09:16] Diego). Era preciso decidir quantas vezes tentar de novo, em que intervalos e o que fazer com o evento quando as tentativas acabam.

## Decisão

1. **Backoff exponencial com 5 tentativas**, nos intervalos 1 min, 5 min, 30 min, 2 h e 12 h ([09:17] Diego, fechado por Larissa em [09:17]).
2. **Esgotadas as tentativas, o evento vai para a DLQ**, numa tabela separada `webhook_dead_letter` com o payload, o motivo da falha e o timestamp ([09:18] Diego).
3. **Reprocessamento manual por endpoint admin**: `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Diego).
4. **O replay exige o papel `ADMIN`** do JWT, reaproveitando o `requireRole` de `src/middlewares/auth.middleware.ts`, e registra em log quem fez o replay, para auditoria ([09:36] Sofia, [09:36] Larissa).
5. **Timeout de 10 segundos** por chamada. Cliente que não responde nesse tempo conta como falha e entra no retry ([09:42] Diego).

**Como contar as 5 tentativas.** A reunião diz "5 tentativas" e lista cinco intervalos, somando "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). A soma dos cinco intervalos é 14 h 36 min, o que só fecha se os cinco intervalos acontecerem depois da primeira falha. Esta decisão adota essa leitura: um envio inicial e mais 5 retentativas, com a DLQ depois da sexta falha. A interpretação está registrada como ponto a confirmar no [RFC](../RFC.md#5-questões-em-aberto).

## Alternativas consideradas

**Retry indefinido com backoff.** Descartado porque o evento ficaria pendurado para sempre se o cliente sumisse ([09:15] Diego).

**3 tentativas.** Proposta por Bruno em [09:16] como mais agressiva. Descartada porque três tentativas se esgotariam em cerca de 30 minutos, menos que a indisponibilidade de duas horas que um cliente já teve ([09:16] Diego).

**Marcar como `failed` na própria outbox em vez de tabela separada.** Levantada por Larissa em [09:17]. Descartada em favor da tabela `webhook_dead_letter`, que deixa a leitura da outbox mais limpa e guarda a evidência para debug e reprocessamento ([09:18] Diego).

## Consequências

**Positivas**

- Cobre indisponibilidades longas do cliente (até cerca de 15 horas) sem intervenção humana.
- A DLQ concentra o que precisa de análise, separada da outbox, que continua guardando o histórico de eventos entregues e falhos até existir o arquivamento ([09:08] Diego).
- O reprocessamento é explícito, auditado e restrito a administradores.

**Negativas**

- Um evento pode chegar ao cliente até cerca de 15 horas depois da mudança de status, já fora da janela de "tempo real".
- A DLQ exige ação humana. Não há aviso automático ao cliente sobre falhas: e-mail foi adiado para uma próxima fase ([09:37] Larissa).
- Como não há listagem da DLQ definida na reunião, o administrador precisa obter o `id` do item por outro meio (logs do worker ou consulta ao banco).

**Trade-off explícito:** entre insistir para sempre e desistir cedo, escolheu-se um teto de cerca de 15 horas, que cobre indisponibilidades reais já vistas, e aceitou-se que o que passar disso dependa de replay manual.
