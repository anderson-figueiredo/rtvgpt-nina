# Preparação para visita do RTV

Quando o RTV informa no WhatsApp que vai visitar um cliente, a Nina classifica a intenção `visit_preparation` e o Digibee monta um **relatório de briefing** no mesmo chat. O texto precisa ser útil no campo: data da última visita, anotações e registros anteriores, e histórico de pedidos.

Complementa [`README.md`](../README.md), [`interpretacao-nina.md`](interpretacao-nina.md), [`detalhes-tecnicos-integracoes.md`](detalhes-tecnicos-integracoes.md) e [`validacao-rtv-cpf.md`](validacao-rtv-cpf.md). O fluxo ponta a ponta no README está em [Fluxo: preparação para visita](../README.md#fluxo-preparação-para-visita).

## Objetivo

- reconhecer, em `pt-BR`, que o RTV vai realizar uma visita;
- resolver o cliente **somente** na carteira vigente do `rtvId` de servidor;
- consolidar fatos de origem com proveniência e freshness;
- devolver um relatório em texto no WhatsApp, sem IDs inventados pela LLM e sem dado financeiro se o AAL for insuficiente.

## Frases de acionamento

Exemplos de utterance (menção não é identidade):

- `Vou visitar o cliente Agro Tal amanhã`
- `Me prepara para a visita na Cooperativa X hoje`
- `Briefing do cliente Agro Tal para visita na segunda`
- `Quero o dossiê do cliente, visito ele dia 15`

Data ausente não bloqueia: o servidor assume a data civil de **hoje** em `America/Sao_Paulo`.

## Tópicos do catálogo

Uma intenção principal: `visit_preparation`. Tópicos entram em `requestedTopics[]`.

| Tópico | Obrigatório no briefing | Fonte | Freshness |
| --- | --- | --- | --- |
| `last_visit` | Sim, se a origem responder | TOTVS/Datasul (histórico comercial / SFA) | 15 min |
| `visit_notes` | Sim, se a origem responder | TOTVS (anotações da visita e observações autorizadas do cadastro) | 15 min |
| `order_history` | Sim, se a origem responder | TOTVS após integração; Portal só durante captura | 15 min |
| `credit_limit` | Não | Tarken, somente com AAL financeiro | 5 min |
| `overdue_titles` | Não | TOTVS | 5 min |
| `delivery_eta` | Não | LoogAI, pedidos em aberto do cliente autorizado | 15 min |

O fan-out padrão, após ABAC do recurso, busca **sempre** `last_visit`, `visit_notes` e `order_history`. Crédito e logística são enriquecimento: entram se o AAL permitir e a origem responder dentro da janela; timeout **não** impede o relatório.

Cadastro autorizado (nome fantasia, cidade, status) tem freshness de 24 h e é o único bloco sem o qual não há briefing.

## Extração híbrida da data

| Sinal | Método | Resultado |
| --- | --- | --- |
| `hoje`, `amanhã`, `depois de amanhã` | Parser civil + `America/Sao_Paulo` | `civilDate` `YYYY-MM-DD` |
| `segunda`, `terça`, … | Próxima ocorrência à frente da data de negócio | `civilDate` |
| `10/09`, `dia 15` | Parser `pt-BR`; ano = ano civil corrente da TZ de negócio, salvo desambiguação | `civilDate` |
| Ausente | Padrão servidor | hoje na TZ de negócio |

`mentions.plannedVisitDate.civilDate` da LLM só é aceito se coincidir com o parser. Conflito ou data civil ambígua → `clarificationCode=AMBIGUOUS_DATE`. A LLM não escolhe timezone.

## Resolução e autorização

Ordem obrigatória: sessão OIDC + ABAC de conversa → NLU → resolução na carteira → ABAC do recurso → fan-out.

| Resultado | Efeito |
| --- | --- |
| Sem menção de cliente | `MISSING_CUSTOMER`; sem fan-out |
| N candidatos na carteira | `AMBIGUOUS_CUSTOMER`; lista curta (nome fantasia + cidade); sem dossiê |
| 0 na carteira | `FORBIDDEN` genérico; evento de segurança; **nenhuma** consulta a visitas, pedidos, Tarken ou LoogAI |
| 1 candidato + AAL2 | Fan-out de visitas, anotações e pedidos |
| 1 candidato + AAL financeiro insuficiente | Briefing sem bloco de crédito; não chama Tarken; **não** exige step-up para o núcleo do relatório |
| Cliente autorizado irresolúvel após TTL do slot | Handoff Teams; WhatsApp informa limitação genérica |

Consulta cadastral, visitas e pedidos exige AAL2. Limite, score e títulos exigem AAL financeiro conforme [`validacao-rtv-cpf.md`](validacao-rtv-cpf.md). A NLU não reduz esse requisito.

Finalidade: `RTV_CUSTOMER_SERVICE`. Ação: `visit_preparation`.

```text
subject autenticado
AND ação visit_preparation permitida
AND cliente pertence à carteira vigente
AND finalidade autorizada
AND nível de autenticação suficiente para o tópico
```

## Fan-out Digibee

Consultas em paralelo, cada uma com deadline e `operationId` de leitura (cache/reconciliação, não mutação):

| Bloco | Fonte | Recorte |
| --- | --- | --- |
| Cadastro | Lecom (campos) + TOTVS (carteira) | Nome fantasia, cidade, status; CNPJ tokenizado — nunca completo no WhatsApp |
| Última visita | TOTVS/Datasul | Data civil, RTV da visita se autorizado, objetivo, resultado |
| Anotações e registros | TOTVS | Até 5 registros mais recentes; texto minimizado e passado por DLP |
| Histórico de pedidos | TOTVS + Portal | Janela de 180 dias; até 8 pedidos (máximo 15); cabeçalho, valor em minor units, status, principais itens |
| Crédito (opcional) | Tarken + títulos TOTVS | Snapshot; não solicita análise nova nem afirma aprovação |
| Logística (opcional) | LoogAI | Pedidos em aberto / ETA / ocorrência do cliente autorizado |

Esta ação **não** reutiliza o fan-out de `order_query` de um único pedido. O contrato é por cliente, com janela de histórico.

## Insights determinísticos

O Digibee calcula `insights` sobre o consolidado. A composição só verbaliza códigos presentes. Máximo de 6 itens, nesta ordem: alerta → oportunidade → contexto → pauta.

| Código | Condição |
| --- | --- |
| `VISIT_GAP` | `diasDesdeUltimaVisita >= 45` em relação a `plannedVisitDate` |
| `OPEN_ORDERS` | há pedido autorizado ainda aberto |
| `DELIVERY_EXCEPTION` | entrega com ocorrência na origem |
| `CREDIT_NEAR_LIMIT` | uso do limite ≥ 80% **e** bloco Tarken `SUCCESS` |
| `OVERDUE_TITLES` | há título vencido autorizado |
| `VOLUME_DROP` | volume da janela inferior ao período comparável (180 × 180 dias) |
| `MISSING_RECURRING_SKU` | item recorrente na janela anterior ausente na janela atual |
| `TALKING_POINT` | pauta derivada de um ou mais códigos anteriores ou de anotação lastreada |

Uma visita há 29 dias **não** gera `VISIT_GAP`. Timeout de visitas não inventa gap nem data. Timeout de Tarken não gera `CREDIT_NEAR_LIMIT`.

## Sucesso parcial

`PARTIAL_SUCCESS` é permitido.

| Situação | WhatsApp |
| --- | --- |
| Visitas/anotações `TIMEOUT` ou `NOT_FOUND`, pedidos `SUCCESS` | Relatório com pedidos; bloco de última visita omitido e limitação explícita |
| Pedidos indisponíveis, visitas `SUCCESS` | Relatório com última visita e anotações; histórico omitido |
| Crédito ou LoogAI indisponível | Omitir o tópico; núcleo intacto |
| Cadastro autorizado e nenhum núcleo (`last_visit`, `visit_notes`, `order_history`) | Briefing mínimo do cadastro + limitação; sem handoff automático |
| `FORBIDDEN` | Resposta genérica; não revelar existência nem carteira |

## Minimização e DLP

Classe do payload: `COMMERCIAL_CONFIDENTIAL`. Notas podem conter `PERSONAL_IDENTIFIER`; o DLP age **antes** da consolidação visível à composição e ao WhatsApp.

- CNPJ/CPF, telefone e score não entram no relatório;
- cada anotação: no máximo 280 caracteres após sanitização;
- no máximo 5 anotações e 8 pedidos no texto;
- teto do relatório: **2000** caracteres (o limite do canal é maior; o produto recorta com prioridade: crédito/logística → anotações antigas → pedidos antigos; última visita e nome fantasia permanecem se existirem);
- ITSM recebe resumo mínimo, não o dossiê integral.

## Relatório no WhatsApp

Renderer determinístico preferencial. LLM de composição só com `sourceField` por segmento; recusa cai no template. Tom: briefing de campo, blocos curtos, sem jargão de integração.

Estrutura obrigatória:

1. Título: nome fantasia autorizado e data civil da visita planejada.
2. Última visita (ou aviso de ausência/indisponibilidade).
3. Anotações e registros anteriores (ou aviso).
4. Histórico de pedidos da janela (3 a 5 linhas no texto; o consolidado pode ter até 8).
5. Pauta apenas com insights lastreados, se houver.
6. `asOf` da consolidação.

Exemplo de texto (fatos do consolidado de referência):

```text
Preparação de visita — Agro Tal Ltda
Ribeirão Preto · visita em 16/09/2026

Última visita: 28/07/2026 (há 50 dias).
Objetivo: reposição da linha de defensivos.
Resultado: combinado retorno com nova tabela.

Anotações e registros:
• 28/07/2026 — Cliente reclamou atraso da NF. Combinado retorno em 30 dias.
• 03/06/2026 — Interesse em aumentar foliar se prazo for 28 dias.
• 20/05/2026 — Comprador no período da manhã.

Pedidos recentes (180 dias):
• 12345 · 01/09 · R$ 15.230,55 · LIBERADO — Defensivo A
• 11890 · 12/08 · R$ 22.100,00 · FATURADO — Defensivo A
• 11002 · 02/07 · R$ 19.800,00 · FATURADO — Semente C
• 10211 · 18/05 · R$ 29.289,55 · FATURADO — Defensivo A, Foliar B

Pauta:
• Há 50 dias sem visita registrada.
• Pedido 12345 em aberto; confirmar status na visita.
• Foliar B não veio no último pedido; retomar prazo de 28 dias.

Atualizado em 15/09/2026 22:12 (America/Sao_Paulo).
```

## Testes mínimos

1. `Vou visitar o cliente Agro Tal amanhã` com cliente na carteira → `visit_preparation`, `civilDate` de amanhã, fan-out de visitas/anotações/pedidos, texto no WhatsApp com os três blocos nucleares.
2. Mesma utterance com cliente fora da carteira → `FORBIDDEN`, zero TOTVS de visitas/pedidos, zero Tarken.
3. Nome ausente → `MISSING_CUSTOMER`.
4. Dois homônimos → `AMBIGUOUS_CUSTOMER` com cidade; sem dossiê.
5. `amanhã` / `hoje` / `10/09` unívocos; `1,000` não se aplica; data ambígua → `AMBIGUOUS_DATE`.
6. LLM devolve `customerId` ou `rtvId` → descartados.
7. Visitas `TIMEOUT` e pedidos `SUCCESS` → `PARTIAL_SUCCESS`; relatório sem inventar data de visita.
8. Visita há 29 dias → não emite `VISIT_GAP`; visita há 45 dias ou mais → emite.
9. AAL financeiro insuficiente → briefing sem crédito e sem chamada Tarken.
10. Composição sem `sourceField` é rejeitada; fallback no renderer.
11. Anotação com telefone/CPF é sanitizada antes do WhatsApp.
12. Relatório excede 2000 caracteres → recorte pela prioridade documentada, sem perder última visita quando ela existir.

## Critérios para produção

- Tópicos `last_visit`, `visit_notes` e `order_history` no schema `intents-v1`.
- Parser de data civil com testes `pt-BR` e TZ `America/Sao_Paulo`.
- Resolução na carteira idêntica às demais intenções.
- Renderer determinístico do relatório; validador factual se houver LLM.
- DLP de anotações comprovado por fixture.
- Matriz de `PARTIAL_SUCCESS` coberta em contract tests.
- RIPD inclui anotações comerciais no canal WhatsApp.
