# Implementation Plan: Interpretação robusta da Nina

## Overview

A arquitetura Digibee + WhatsApp + identidade + ABAC + outbox já está definida. A lacuna é a Nina: hoje ela existe como bot de Teams feito em Microsoft Copilot Studio, sem contrato auditável de interpretação para o WhatsApp.

Este plano entrega um runtime de NLU de catálogo fechado, com Copilot (Azure OpenAI / Microsoft Foundry) e OpenAI como provedores, extração híbrida de linguagem natural, resolução de cliente na carteira do RTV e fan-out Digibee só depois dos guardrails vigentes. O primeiro recorte vertical é a análise de crédito do tipo “cabe um pedido de 1 milhão para Hommerson Agro?”.

Especificação durável: [`interpretacao-nina.md`](interpretacao-nina.md). Origem arquitetural: [`README.md`](../README.md), [`validacao-rtv-cpf.md`](validacao-rtv-cpf.md), [`detalhes-tecnicos-integracoes.md`](detalhes-tecnicos-integracoes.md), [`riscos-integracao.md`](riscos-integracao.md).

## Specification Link

| Documento | Uso neste plano |
| --- | --- |
| [Interpretação da Nina](interpretacao-nina.md) | Contrato NLU, catálogo, resolução, provedores, exemplo Hommerson Agro |
| [Arquitetura de referência](../README.md) | Inbox, identidade, ABAC, composição, outbox |
| [Validação RTV](validacao-rtv-cpf.md) | OIDC/MFA, carteira, step-up, resposta genérica |
| [Detalhes técnicos](detalhes-tecnicos-integracoes.md) | Pipelines Digibee, schemas, consolidação |
| [Riscos](riscos-integracao.md) | P0/P1 existentes; novos riscos R26–R31 |

## Requirements Summary

### Funcionais

- Interpretar utterance em `pt-BR` e devolver uma intenção do catálogo `intents-v1`, tópicos e menções.
- Reconhecer análise de crédito com nome fantasia e valor em linguagem natural (`1milhão`).
- Resolver o cliente **somente** na carteira do RTV autenticado, via Digibee.
- Recusar de forma genérica cliente fora da carteira, injeção, fora de escopo e ID inventado pelo modelo.
- Consultar Tarken e títulos TOTVS só após ABAC do recurso e AAL financeiro.
- Responder com fatos de origem e insights `CREDIT_*`; não aprovar pedido.
- Clarificar homônimos, valor ambíguo e baixa confiança, com slot pendente versionado.
- Manter o bot Copilot no Teams como canal/handoff, sem orquestração generativa contra ERP.

### Não funcionais

- Structured output equivalente nos dois provedores; timeout e recusa tipados.
- DLP e minimização antes de Copilot e OpenAI; treinamento desabilitado; região aprovada.
- Failover só para falha de transporte/`429`/`5xx`, nunca para a intenção “mais permissiva”.
- Contract tests, testes de acesso cruzado e trilha imutável sem utterance completa com PII.
- Latência de NLU observável e separada da latência de fan-out.

### Critérios de aceite

- A utterance de Hommerson Agro, com cliente na carteira, produz `credit_analysis`, valor `100000000` BRL e consultas Digibee autorizadas.
- A mesma utterance com cliente fora da carteira não chama Tarken e não revela existência externa.
- Payload da LLM com `customerId` ou `rtvId` adulterado não altera identidade nem recurso.
- Copilot indisponível falha de forma controlada ou transfere para OpenAI sem duplicar efeito de negócio.
- Composição de crédito sem `sourceField` é rejeitada e cai no renderer determinístico.

## Technical Approach

Não reescrever o hub. Inserir `nina-nlu` entre a ABAC de conversa e o orquestrador já previsto.

```text
evento canônico
  -> sessão + ABAC conversa
  -> extratores determinísticos
  -> adapter NLU (Copilot | OpenAI)
  -> validador de catálogo/guardrail
  -> resolve_customer_mention (Digibee, filtro de carteira)
  -> ABAC recurso + step-up
  -> nina-whatsapp-orchestrator (intenção)
  -> consolidação / renderer / outbox
```

Decisões:

1. **Copilot Studio não é o cérebro do WhatsApp.** Inventariar tópicos atuais só para corpus de exemplos.
2. **Dois contratos LLM.** Nina → adapter (interno) e adapter → provedor (nativo). O provedor `microsoft_copilot` usa Azure OpenAI/Foundry com o mesmo schema; AI prompts do Studio só se o adapter revalidar o JSON.
3. **Catálogo fechado.** Sem tool-calling do modelo contra Tarken/TOTVS. Digibee MCP, se existir, não publica sistemas de origem ao agente.
4. **Menção versus identidade.** A LLM devolve `mentions.customer.raw`; o Digibee resolve na carteira.
5. **Híbrido.** Parser `pt-BR` de dinheiro/pedido/documento + LLM para intenção e nome. Conflito gera clarificação.
6. **Crédito é fato, não aprovação.** `CREDIT_SUFFICIENT_FOR_AMOUNT` / `CREDIT_INSUFFICIENT` nascem de Tarken + valor pedido.

## Phases

### Fase 0 — Inventário do Copilot atual

Objetivo: conhecer o bot Teams sem bloqueá-lo como dependência do runtime novo.

- [ ] Exportar tópicos, frases de gatilho, entidades, actions, knowledge e conectores HTTP/MCP do Copilot Studio.
- [ ] Classificar cada action: canal/handoff versus consulta a sistema de origem.
- [ ] Desligar ou isolar generative orchestration que chame ERP, crédito ou ITSM.
- [ ] Recolher utterances reais (minimizadas) para o corpus `pt-BR` de NLU.
- [ ] Registrar gaps: o que o bot faz hoje e o que o runtime novo precisa cobrir.

Saída: inventário versionado e corpus inicial. O WhatsApp não passa a depender do Studio.

### Fase 1 — Contratos e catálogo

Objetivo: tornar a interpretação testável antes de ligar provedores.

- [ ] Publicar JSON Schema `nina-nlu-request` e `nina-nlu-result` (`additionalProperties: false`).
- [ ] Publicar `intents-v1` com `credit_analysis`, `order_query`, `visit_preparation`, `customer_lookup`, `customer_update`, `order_create`, `clarification_response`, `human_handoff_request`, `out_of_scope`.
- [ ] Definir `requestedTopics[]` e matriz mínima de `credit_analysis`.
- [ ] Versionar limiares de confiança, timeout NLU e política de failover.
- [ ] Contract tests produtor/consumidor; exemplos do Hommerson Agro válidos no schema.
- [ ] OpenAPI 3.1 de `POST /v1/nina/interpret` (interno) e `POST /v1/nina/resolve-customer`.

Saída: schemas no pipeline de contrato; ruptura bloqueia merge.

### Fase 2 — Runtime NLU e provedores

Objetivo: classificar utterance sem fan-out de negócio.

- [ ] Implementar `nina-nlu` no Digibee (ou workload adjacente) consumindo o evento canônico já autorizado.
- [ ] Parser determinístico `pt-BR` de valores, pedidos e documentos.
- [ ] Adapter `microsoft_copilot` (Azure OpenAI/Foundry, structured output, allowlist de modelo).
- [ ] Adapter `openai` equivalente; roteamento e failover versionados.
- [ ] Validador: enum, schema, recusa, incompletude, `UNGROUNDED_ID`.
- [ ] Motor de guardrail: injeção, cross-portfolio, spoof de identidade, out_of_scope.
- [ ] DLP antes do provedor; utterance completa fora de logs comuns.
- [ ] Métricas: rejeição, timeout, provedor, intenção, latência.

Saída: NLU isolada, com fixtures, sem consultar Tarken.

### Fase 3 — Resolução na carteira e guardrails de recurso

Objetivo: ligar a menção ao cliente do RTV sem vazar carteira alheia.

- [ ] Pipeline Digibee `resolve_customer_mention` filtrado por `rtvId` de servidor (TOTVS vigente; Lecom auxiliar).
- [ ] Resultados 0 / 1 / N com limiar de similaridade versionado.
- [ ] Clarificação `AMBIGUOUS_CUSTOMER` só com rótulos da carteira.
- [ ] ABAC do recurso e step-up financeiro reutilizando a policy engine vigente.
- [ ] Testes: cliente de outro RTV, homônimo, `rtvId` no payload da LLM, prefixo de CPF sem sessão.

Saída: evidência de que fora da carteira = `FORBIDDEN` genérico e zero Tarken.

### Fase 4 — Recorte vertical de análise de crédito

Objetivo: a utterance de 1 milhão vira fato consolidado.

- [ ] Orquestrador: `credit_analysis` → Tarken (limite/disponibilidade) + TOTVS (títulos), deadlines e freshness de 5 min.
- [ ] Comparação do valor pedido somente com bloco Tarken `SUCCESS`.
- [ ] Insights `CREDIT_INSUFFICIENT`, `CREDIT_SUFFICIENT_FOR_AMOUNT`, reuso de `CREDIT_NEAR_LIMIT` e `OVERDUE_TITLES`.
- [ ] Renderer determinístico da análise; LLM de composição só com `sourceField`.
- [ ] Slot de clarificação para valor/cliente ausente ou ambíguo.
- [ ] Sem `PARTIAL_SUCCESS` decisório: timeout Tarken não afirma que “cabe” o pedido.

Saída: ponta a ponta WhatsApp → NLU → Digibee → resposta factual.

### Fase 5 — Reuso das intenções já desenhadas

Objetivo: a mesma NLU alimentar `order_query` e `visit_preparation` sem novo cérebro.

- [ ] Mapear tópicos extraídos para a matriz de resultado mínimo já publicada.
- [ ] Resolução de pedido na carteira, análoga à de cliente.
- [ ] `clarification_response` retoma a intenção pendente por `conversationVersion`.
- [ ] `human_handoff_request` reusa o fallback Teams existente.
- [ ] Corpus e testes para multi-tópico (“previsão do pedido 12345 e meu limite”).

Saída: uma interpretação, vários orquestradores já especificados.

### Fase 6 — Operação, canário e produção

Objetivo: evidências P0/P1 do runtime de interpretação.

- [ ] Canário Copilot versus OpenAI nas mesmas fixtures; alerta de divergência.
- [ ] Dashboards e runbooks: NLU, resolução 0/1/N, injeção, step-up, insights de crédito.
- [ ] RIPD e inventário LGPD do novo processamento de utterance.
- [ ] Chaos: timeout de provedor, timeout Tarken, ITSM indisponível (conversa segue).
- [ ] Checklist de produção de [`interpretacao-nina.md`](interpretacao-nina.md) e P0 de [`riscos-integracao.md`](riscos-integracao.md).

Saída: go/no-go com evidências; Copilot Studio permanece canal, não orquestrador.

## Dependencies

| Dependência | Por quê | Bloqueia |
| --- | --- | --- |
| Sessão OIDC + vínculo telefone–RTV | Sem sujeito não há carteira | Fan-out e NLU de crédito |
| Policy engine ABAC vigente | Carteira e AAL financeiro | Fase 3–4 |
| Contrato Tarken / TOTVS no Digibee | Fatos de crédito e títulos | Fase 4 |
| Adapter WhatsApp + inbox | Evento canônico | Todas as fases de runtime |
| Aprovação de região/retenção Copilot e OpenAI | DLP e RIPD | Tráfego real de utterance |
| Inventário Copilot Studio | Corpus e desligar tools perigosas | Fase 0; não bloqueia schemas da Fase 1 |
| Modelos allowlist com structured output | Schema estrito | Fase 2 |

A ausência do código-fonte do bot Copilot **não** bloqueia Fases 1–4. O runtime é novo por desenho.

## Risks & Mitigation

| ID | Risco | Mitigação |
| --- | --- | --- |
| R26 | Copilot Studio chama ERP por generative orchestration | Desligar tools de origem; único efeito = gateway Nina |
| R27 | NLU inventa `customerId` / `rtvId` | IDs da LLM descartados; resolução só na carteira |
| R28 | Busca de nome vaza cliente de outro RTV | Filtro de carteira na origem; 0 e “existe fora” indistinguíveis |
| R29 | Failover escolhe a intenção mais permissiva | Failover só em erro de plataforma; canário alerta divergência |
| R30 | Parser de `1milhão` / `1,000` erra o valor | Parser versionado + clarificação; crédito sem valor não decide |
| R31 | Modelo afirma “pedido aprovado” | Renderer determinístico; validador factual; insight ≠ aprovação |
| R06/R12 | Já mapeados | Reafirmados na NLU: identidade servidor-side e `sourceField` |

Detalhamento em [`riscos-integracao.md`](riscos-integracao.md).

## Success Criteria

1. Utterance natural de crédito é interpretada, autorizada e respondida com fatos Tarken/TOTVS.
2. Cliente fora da carteira do RTV nunca gera análise nem vazamento de existência.
3. Copilot e OpenAI são intercambiáveis no adapter, com o mesmo schema interno.
4. O bot Teams atual não é requisito para o WhatsApp e não orquestra sistemas de origem.
5. Testes da seção “Testes mínimos” de [`interpretacao-nina.md`](interpretacao-nina.md) passam no pipeline.

## Task breakdown for a task database

Se houver base de tarefas no Notion, criar um item por checkbox das Fases 0–6, com esta página como spec, status “To Do” e vínculo ao recorte `credit_analysis`. Ordem sugerida: Fase 1 em paralelo à 0; 2 depende de 1; 3 depende de 2 e da ABAC vigente; 4 depende de 3 e dos contratos Tarken/TOTVS; 5 depois do vertical de crédito; 6 contínuo a partir da Fase 2.
