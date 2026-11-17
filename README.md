# Da Reunião ao Documento: design docs do Sistema de Webhooks

Entrega do desafio "Da Reunião ao Documento: Design Docs Gerados por IA". O enunciado original está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Um OMS em Node.js, TypeScript, Prisma e MySQL vai ganhar webhooks de notificação de pedidos. A decisão técnica saiu de uma reunião de 53 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, e o único registro é a transcrição. A tarefa foi transformar essa transcrição, junto com o código do OMS, num pacote de design docs (PRD, RFC, FDD, ADRs e um tracker de rastreabilidade) que um time conseguisse usar para começar a implementar.

O que mais pesou foi a regra de que nada pode ser inventado. A reunião tem decisões fechadas, propostas que ninguém fechou, ideias descartadas, itens adiados e até correções no meio da conversa, como o `customer_id` que "vem do JWT" e, um minuto depois, não vem mais. O trabalho foi separar essas categorias, levar cada uma ao documento certo e conseguir apontar, para cada linha escrita, a fala ou o arquivo de onde ela saiu.

## Ferramentas de IA utilizadas

- **Claude Code (modelo Claude Opus 5.5), no terminal:** ferramenta principal. Leu a transcrição e o código direto do repositório, escreveu os documentos e aplicou as correções.
- **Subagentes do Claude Code com prompts dedicados:** um agente de extração (modelo Claude Sonnet 5.5) montou o inventário da transcrição antes da escrita, e dois agentes revisores leram os documentos prontos contra a transcrição e o código. Eles não escreveram nada, só apontaram problemas. Separar quem escreve de quem revisa foi o principal recurso contra alucinação.
- **Scripts Python gerados na sessão:** montaram o Tracker a partir de uma lista de itens e validaram cada linha (o `[hh:mm] Nome` existe na transcrição com aquele falante? o caminho de código existe?).

## Workflow adotado

1. **Contexto.** Leitura integral da transcrição e dos arquivos que a feature toca: `order.service.ts`, `order.status.ts`, a hierarquia de erros, os middlewares, o logger, `env.ts`, `database.ts`, `server.ts`, `app.ts`, `routes/index.ts`, `schema.prisma` e os testes.
2. **Inventário dirigido.** Em paralelo à leitura, um subagente de extração classificou a transcrição em decisões fechadas, alternativas descartadas, requisitos, restrições, itens fora de escopo, questões em aberto, ganchos com o código e contradições internas, sempre com timestamp (prompt 1 abaixo).
3. **ADRs primeiro.** As sete decisões viraram ADRs no formato MADR: as seis principais da reunião mais uma secundária, snapshot e filtro na inserção, que tinha alternativas reais discutidas. Cada ADR cita a fala que fechou a decisão e, quando é o caso, o arquivo do código.
4. **RFC** em cima dos ADRs, curto, com as alternativas descartadas e as questões em aberto da reunião.
5. **FDD**, com modelo de dados, fluxos, contratos, matriz de erros e a seção de integração com o código existente.
6. **PRD** por último entre os grandes, consolidando o que já estava decidido na altura de produto.
7. **Revisão adversarial** dos quatro documentos por um agente que não escreveu nenhum deles (prompt 2), seguida das correções.
8. **Tracker** gerado por script e validado automaticamente.
9. **Revisão final** do sentido de cada linha do Tracker e da consistência depois das correções (prompt 3).
10. **README** por último, com o processo já completo.

A divisão de altura entre os documentos foi definida antes de escrever: o PRD responde por que e o quê, sem caminho de arquivo nem nome de framework; o RFC propõe e lista o que está em aberto; os ADRs explicam cada decisão isolada; o FDD diz como construir. Sempre que um nome de tabela, rota ou código de erro não foi dito na reunião, o FDD escreve "escolha deste FDD", e o Tracker aponta a fala ou o arquivo que motivou a escolha.

## Prompts customizados

**Prompt 1: extração dirigida da transcrição.** Escrito para impedir o resumo genérico e forçar a separação entre o que entra e o que não entra. A seção 8 rendeu as descobertas mais úteis.

```text
Você é um analista técnico extraindo um inventário rastreável de uma transcrição de reunião.
Não escreva documentos, não resuma de forma genérica e não use conhecimento externo: só o que
está no texto.

Leia por inteiro TRANSCRICAO.md (reunião sobre um Sistema de Webhooks de Notificação de Pedidos
para um OMS em Node.js/TypeScript + Prisma/MySQL). Não leia nem altere nenhum outro arquivo.

Produza, em português, um inventário em tabelas markdown com estas categorias, nesta ordem:

1. DECISÕES FECHADAS: decisão, quem fechou, timestamp exato no formato [hh:mm] Nome da fala que
   fecha a decisão (e, se diferente, a da fala que propôs).
2. ALTERNATIVAS DESCARTADAS: alternativa, motivo do descarte com as palavras da reunião, [hh:mm] Nome.
3. REQUISITOS FUNCIONAIS EXPLÍCITOS: o que o sistema deve fazer, [hh:mm] Nome.
4. REQUISITOS NÃO FUNCIONAIS E RESTRIÇÕES (números, limites, prazos, segurança), [hh:mm] Nome.
5. FORA DE ESCOPO / ADIADO / "OBSERVAR DEPOIS": item, classificação exata (descartado, adiado
   para próxima fase, observar e decidir depois, outro projeto), [hh:mm] Nome.
6. QUESTÕES EM ABERTO OU LIMITAÇÕES CONHECIDAS que a reunião não resolveu.
7. GANCHOS COM O CÓDIGO: todo arquivo, classe, função, tabela, padrão ou comando do código
   existente que foi citado na fala, [hh:mm] Nome.
8. CORREÇÕES E CONTRADIÇÕES DENTRO DA REUNIÃO: pontos em que alguém afirmou algo e depois foi
   corrigido, ou em que dois trechos não batem (números, prazos, contagens). Para cada um, cite
   os dois timestamps e diga qual versão prevalece e por quê. Seja minucioso aqui: confira as
   contas (por exemplo, somas de intervalos versus número de tentativas) e datas.
9. DECISÕES TOMADAS DEPOIS QUE PARTE DOS PARTICIPANTES SAIU DA CALL: quais, e quem estava presente.

Regras: cada linha precisa de um timestamp que exista no arquivo; se não achar timestamp, não
inclua a linha. Não transforme sugestão em decisão: se alguém propôs e ninguém fechou, classifique
como proposta. Não invente nomes de endpoint, campos ou valores que não foram ditos. Devolva só
o inventário.
```

**Prompt 2: revisão adversarial.** Dado a um agente que não participou da escrita, com a instrução explícita de não elogiar e de conferir todas as citações, não uma amostra.

```text
Você é um revisor técnico adversarial. Sua função é achar erros, não elogiar. Não edite nenhum
arquivo.

Fontes de verdade: TRANSCRICAO.md e o código em src/, prisma/, tests/, package.json,
tsconfig*.json, docker-compose.yml. Documentos sob revisão: docs/PRD.md, docs/RFC.md,
docs/FDD.md e docs/adrs/ADR-00*.md.

Verifique, item por item, e reporte só problemas concretos:
1. Citações erradas: toda referência [hh:mm] Nome precisa existir na transcrição com aquele nome
   naquele minuto, e a fala precisa sustentar a afirmação em que foi usada. Confira TODAS.
2. Invenção: requisitos, números, decisões ou restrições sem origem na transcrição nem no código
   e que não estejam marcados explicitamente como "escolha deste FDD".
3. Escopo invertido: algo descartado, adiado ou "observar depois" aparecendo como requisito.
4. Contradições entre documentos e entre documentos e transcrição/código.
5. Código: todo caminho, classe, função, linha ou comportamento citado precisa existir e estar
   descrito corretamente.
6. Fronteira entre documentos: RFC descendo a detalhe de FDD, trechos duplicados, PRD com jargão.
7. Requisitos do desafio (lista dos mínimos de cada documento).
8. Erros técnicos de desenho: algo no FDD que não funcionaria como descrito (SQL, Prisma, Node,
   HMAC, HTTP) ou que contradiz uma decisão fechada.
9. Links internos quebrados.

Formato: lista numerada do mais grave para o menos grave, com arquivo, trecho citado, categoria,
problema e correção sugerida. Sem achados vagos ou de estilo.
```

**Prompt 3: revisão semântica do Tracker e da consistência pós-correção.** O script de validação só prova que a fala existe; este prompt confere se a fala sustenta o item.

```text
Para CADA linha de docs/TRACKER.md, confira: (a) a fala ou o arquivo apontado em "Localização"
sustenta de fato o "Conteúdo (resumo)"; (b) o item existe no documento indicado. Já foi verificado
por script que todos os timestamps e caminhos existem, então não reporte isso; reporte só linhas
em que a fala/arquivo NÃO sustenta o conteúdo (fala de outro assunto, minuto vizinho errado,
falante errado) e sugira o [hh:mm] Nome correto.

Depois, reporte contradições entre os documentos nos pontos que acabaram de ser alterados e
qualquer resquício da versão anterior (query ?customerId=, webhook.errors.ts, p95,
"100 tentativas", next_attempt_at, NOW(), WEBHOOK_SECRET_REQUIRED usado como erro).
```

## Iterações e ajustes

Foram quatro ciclos principais de geração, revisão e correção.

**Iteração 1: o inventário contra a primeira versão dos ADRs.** A comparação mostrou três buracos. O ADR-002 prometia ordem por pedido sem ressalva, mas com retry um evento esperando 30 minutos é ultrapassado por um evento posterior do mesmo pedido; isso virou uma regra proposta no FDD e uma questão em aberto no RFC. Como o `customer_id` não vem do JWT, qualquer usuário autenticado gerencia webhooks de qualquer cliente, o que entrou como risco e questão em aberto. E os blocos da estimativa de Larissa já somavam três sprints antes do "mais um pouco" e da revisão de Sofia, então o prazo deixou de aparecer como fato no PRD. A ambiguidade das "5 tentativas" (cinco intervalos que só somam as "quase 15 horas" se houver um envio inicial mais cinco retentativas) foi pega nas duas leituras e ficou registrada como interpretação a confirmar.

**Iteração 2: revisão adversarial, com 31 achados.** Os mais sérios eram erros de desenho que pareciam corretos:

- **SQL do worker:** usava colunas em snake_case, mas o `schema.prisma` do projeto não usa `@map` e as colunas são camelCase, então o `$queryRaw` quebraria em runtime.
- **Envio em série:** com um worker e timeout de 10 segundos, um cliente lento seguraria o lote e estouraria a meta de todos os outros, que é o mesmo problema que motivou descartar o envio síncrono. Passou a ser paralelo dentro do lote.
- **Webhooks pausados:** as linhas deles ficavam no topo da fila para sempre e travavam o worker.
- **Rotação dupla:** duas rotações seguidas matavam a secret original antes das 24 horas prometidas por Sofia, o que criou o erro `WEBHOOK_ROTATION_IN_PROGRESS`.
- **Exemplo de HMAC:** o código de verificação do cliente podia lançar `RangeError` com header malformado.

Também havia invenção sutil: o "p95" da meta e a "meta de 3 de 3 clientes" estavam atribuídos a falas que não diziam isso, a listagem por query `?customerId=` estava atribuída a Larissa (que falou em "body ou path") e o PRD inventava detalhes, como a indisponibilidade de duas horas ser "da Nova Cargo". Por último, o PRD tinha jargão e caminho de arquivo, e o RFC repetia detalhes do FDD. Tudo isso foi corrigido ou marcado como escolha do documento.

**Iteração 3: Tracker gerado e validado por script.** O script pegou o que a leitura humana deixa passar: cada `[hh:mm] Nome` foi casado com as falas reais da transcrição e cada caminho com o sistema de arquivos. A cobertura foi ampliada até incluir critérios de aceite, decisões resumidas no PRD, alternativas de cada ADR e todos os códigos de erro.

**Iteração 4: revisão semântica final.** Das 217 linhas do Tracker naquele momento, 12 apontavam para uma fala que existia mas não sustentava o item.

- **Atribuições trocadas:** o `X-Webhook-Id` estava creditado ao "boa" de Diego em [09:45], quando quem propôs foi Sofia em [09:44]. A revisão de segurança aparecia como cobrindo o armazenamento da secret, mas Sofia só pediu para revisar "HMAC e geração de secret"; o armazenamento virou sugestão do FDD, marcada como escolha de projeto.
- **Linhas com várias fontes:** as que juntavam duas ou três decisões com uma fala só foram desmembradas.
- **Contradições nos documentos:** a pausa de um webhook dava a entender que guardava os eventos, quando o filtro na inserção faz as mudanças da pausa nunca virarem evento. O envio paralelo ainda esperava o lote inteiro, o que deixava um cliente lento atrasar o ciclo seguinte; o ciclo passou a não bloquear. Também sobravam resquícios da versão anterior, como um `p95` na tabela de métricas e o `WEBHOOK_SECRET_REQUIRED` listado como erro em uso no ADR-006.

O Tracker final tem 224 linhas, com 87% das fontes na transcrição e 29 no código.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [docs/PRD.md](docs/PRD.md): o problema, os clientes, as metas, o que entra e o que fica de fora.
2. [docs/RFC.md](docs/RFC.md): a proposta em nível de arquitetura, as alternativas descartadas e as dez questões em aberto.
3. [docs/adrs/](docs/adrs/): sete decisões, uma por arquivo, com índice em [docs/adrs/README.md](docs/adrs/README.md).
4. [docs/FDD.md](docs/FDD.md): como construir, com modelo de dados, fluxos, contratos HTTP, matriz de erros `WEBHOOK_*`, observabilidade e a seção 11, de integração com o código existente.
5. [docs/TRACKER.md](docs/TRACKER.md): a origem de cada item, para conferir qualquer afirmação dos documentos.

```
.
├── README.md                 este arquivo
├── TRANSCRICAO.md            fonte, não alterada
└── docs/
    ├── PRD.md
    ├── RFC.md
    ├── FDD.md
    ├── TRACKER.md
    └── adrs/
        ├── README.md
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-worker-separado-em-polling.md
        ├── ADR-003-retry-com-backoff-e-dlq.md
        ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-005-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-do-projeto.md
        └── ADR-007-snapshot-e-filtro-na-insercao.md
```

O código da aplicação (`src/`, `prisma/`, `tests/` e configurações) não foi alterado.
