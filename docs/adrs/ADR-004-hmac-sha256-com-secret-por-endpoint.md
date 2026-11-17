# ADR-004: Assinatura HMAC-SHA256 com secret por endpoint e rotação com 24 h de convivência

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), com contribuições de Bruno e Diego
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [FDD, seção 6.2](../FDD.md#62-contrato-de-saída-requisição-enviada-ao-cliente)

## Contexto

Os webhooks levam dados de pedidos para endpoints fora da infraestrutura da empresa. O cliente precisa conseguir verificar que a requisição veio mesmo da plataforma e que ninguém alterou o payload no caminho ([09:19] Sofia). O fluxo é só de saída: os clientes recebem, não enviam ([09:02] Marcos, [09:03] Sofia). A equipe já viu um cliente vazar uma secret no log da própria aplicação ([09:22] Diego).

## Decisão

1. **HMAC-SHA256 sobre o corpo da requisição.** A plataforma assina o corpo com uma secret compartilhada e envia a assinatura no header `X-Signature`; o cliente recalcula e compara ([09:20] Sofia, fechado em [09:22] Sofia).
2. **Uma secret por endpoint de webhook**, nunca uma secret global da plataforma, para que um vazamento atinja um único cadastro ([09:21] Sofia).
3. **A secret é gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos). A configuração guarda URL, secret, `customer_id` e estado ativo ([09:21] Bruno, confirmado por Sofia).
4. **Rotação pela API.** O cliente pode pedir uma secret nova; a antiga continua válida por 24 horas em paralelo e depois deixa de valer ([09:21] Sofia).
5. **HTTPS obrigatório.** URL com `http://` é recusada com erro de validação no schema Zod. Sofia classificou isso como validação, não como decisão arquitetural ([09:23] Sofia); fica registrado aqui por ser parte da mesma proteção.

Como a assinatura é sobre o corpo, o header `X-Timestamp` ([09:44] Diego) não entra no cálculo. A proteção prática contra replay é a deduplicação por `X-Event-Id` ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)): uma requisição capturada e reenviada por terceiros chega com um `event_id` já processado. O campo `timestamp` do payload é o horário da mudança de status e é igual em todas as tentativas, então não serve como janela de frescor. Assinar também o `X-Timestamp` mudaria a decisão de Sofia de assinar só o corpo e ficou como questão em aberto no [RFC](../RFC.md#5-questões-em-aberto). Uma segunda rotação só é aceita depois que a janela de 24 horas da anterior acaba, para que a secret original nunca expire antes do prazo. O FDD especifica o formato do header e o comportamento durante a janela.

## Alternativas consideradas

**Secret global da plataforma.** Descartada porque, se vazar, vaza para todos os clientes ([09:21] Sofia).

**Rotação com troca imediata (sem janela).** Plausível e mais simples de implementar, mas descartada na própria definição de Sofia: o cliente precisa de tempo para migrar os sistemas dele, daí as 24 horas de convivência ([09:21] Sofia).

## Consequências

**Positivas**

- O cliente valida origem e integridade com bibliotecas padrão de mercado ([09:20] Sofia).
- O vazamento de uma secret fica contido a um endpoint e pode ser remediado pelo próprio cliente com a rotação.

**Negativas**

- O HMAC exige o valor original da secret na hora de assinar, então ela não pode ser guardada como hash irreversível, como as senhas em `users.passwordHash`. A forma de armazenamento (criptografia em repouso ou não) não foi discutida e fica para a revisão de segurança que Sofia pediu antes do deploy ([09:46] Sofia).
- Durante as 24 horas de convivência, o worker precisa assinar com as duas secrets, o que aumenta o tamanho do header e a lógica de assinatura.
- `X-Timestamp` fora da assinatura é informativo: pode ser alterado no caminho sem invalidar a assinatura. Contra replay, o cliente conta com a deduplicação por `X-Event-Id`.

**Trade-off explícito:** a plataforma assume a complexidade de gerar, guardar e rotacionar uma secret por endpoint para que o impacto de um vazamento fique restrito a um único cadastro.
