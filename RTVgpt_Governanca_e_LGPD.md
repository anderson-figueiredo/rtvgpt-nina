# RTVgpt — Governança, LGPD e Conformidade com a Arquitetura

Este documento consolida governança, riscos e indicadores do RTVgpt em conformidade com a arquitetura de referência do projeto: Digibee como hub obrigatório, aceite assíncrono com inbox/outbox duráveis, `conversationId` como identidade da conversa, Nina com NLU de catálogo fechado e controles de identidade/ABAC no servidor.

Fontes de referência: `README.md`, `docs/detalhes-tecnicos-integracoes.md`, `docs/riscos-integracao.md`, `docs/validacao-rtv-cpf.md` e `docs/interpretacao-nina.md`.

## 1. Princípios de governança obrigatórios

1. Toda operação de negócio passa pelo Digibee; não há fan-out direto da LLM para sistemas de origem.
2. O `200 OK` do webhook confirma somente validação + persistência durável na inbox.
3. `conversationId` é obrigatório; `ticketId` é opcional e tratado como projeção operacional.
4. Identidade, `rtvId`, tenant, carteira e destinatário são derivados no servidor; nunca aceitos da LLM.
5. Outbox durável e `operationId` garantem idempotência e reconciliação entre ITSM, WhatsApp e Teams.
6. A LLM é não confiável: intenção e menções são hipóteses; fatos finais só saem de fontes versionadas.
7. DLP e minimização são aplicados antes de OpenAI, Teams, ITSM e WhatsApp.

## 2. Domínios de dados, ownership e fonte de verdade

| Domínio | Fonte de verdade | Dono do domínio | Regra de governança |
| --- | --- | --- | --- |
| Identidade RTV e vínculo telefone | IAM corporativo + cofre de canal | Segurança/IAM | OIDC + PKCE + MFA, sessão curta e revogação por mudança de vínculo |
| Carteira vigente | TOTVS/Datasul | Comercial + ERP | ABAC exige carteira vigente antes de qualquer consulta de cliente/pedido |
| Cadastro fiscal | Lecom (por campo) | Master Data | Lecom complementa, mas não expande autorização da carteira |
| Visita comercial e anotações | TOTVS/Datasul (SFA) | Comercial | Histórico de visita não é inventado; DLP antes do WhatsApp |
| Crédito e limite | Tarken (crédito) + TOTVS (títulos) | Crédito | Consulta assistida; não representa aprovação automática de pedido |
| Pedido integrado | TOTVS/Datasul | Comercial + ERP | Portal é auxiliar durante captura; pós-integração vale TOTVS |
| ETA/logística | LoogAI | Logística | ETA é independente do status financeiro/comercial |
| Conversa e auditoria | Event store/auditoria imutável | Arquitetura + Segurança | ITSM não é trilha primária de auditoria |

## 3. Segurança e LGPD por desenho

### 3.1 Identidade, sessão e autorização

- Autenticação: OIDC Authorization Code + PKCE + MFA.
- Sessão: escopo por `tenantId + environment + conversationId + subjectId`.
- ABAC: sujeito autenticado + ação permitida + cliente/pedido na carteira + finalidade + nível de autenticação.
- Step-up: obrigatório para crédito, dados financeiros e mutações.
- Prefixo de CPF: apenas sinal antifraude; não autentica nem autoriza.

### 3.2 Minimização e classificação de dados

| Classe | Exemplos | Política |
| --- | --- | --- |
| `PERSONAL_IDENTIFIER` | CPF/CNPJ, telefone, nome | Tokenização/mascaração conforme finalidade |
| `COMMERCIAL_CONFIDENTIAL` | pedido, condições comerciais | Menor conjunto necessário e ACL restritiva |
| `FINANCIAL_PROFILE` | limite, score, títulos | Step-up, destino restrito e mascaramento reforçado |
| `SECURITY_EVIDENCE` | sinais de fraude/injeção | Trilha segregada; não expor em WhatsApp/ITSM público |

### 3.3 Requisitos formais LGPD

- Inventário de tratamento com finalidade e base legal por fluxo.
- RIPD para uso de WhatsApp + provedores LLM.
- Contratos de operador (DPA) com provedores de canal e IA.
- Retenção, descarte e exclusão propagada definidos e testados.
- Controles de transferência internacional e retenção mínima.
- Revisão humana para decisões automatizadas materialmente relevantes.

## 4. Riscos técnicos e controles mandatórios

| Risco | Impacto | Controle arquitetural |
| --- | --- | --- |
| Reentrega/duplicidade de webhook | Duplicação de efeitos | Inbox com unicidade de `messageId`, lease, fencing e CAS |
| Corrida entre sistemas externos | Estado divergente | Outbox por efeito + reconciliação por `operationId` |
| Cliente fora da carteira receber resposta | Vazamento/IDOR | Resolução na carteira + resposta `FORBIDDEN` genérica |
| LLM inventar `customerId`/`rtvId` | Acesso indevido | IDs da LLM descartados; identidade só do servidor |
| Resposta fora de ordem | Regressão de estado | Sequência e versão da conversa; respostas `STALE` não enviadas |
| Falha de ITSM bloquear conversa | Paralisação operacional | ITSM como projeção; conversa segue com `ticketLinkStatus` |
| Valor monetário ambíguo (ex.: `1,000`) | Decisão errada de crédito | Parser `pt-BR` + clarificação obrigatória |
| Janela de 24h WhatsApp | Falha de notificação proativa | Templates aprovados e política de envio fora da janela |

## 5. Indicadores de governança e valor

### 5.1 Indicadores operacionais e de segurança

- Latência de ACK pós-persistência da inbox.
- Backlog e idade máxima de inbox/outbox.
- Taxa de duplicatas rejeitadas e de eventos `STALE`.
- Taxa de `FORBIDDEN`, `STEP_UP_REQUIRED`, `NLU_REJECTED` e `NLU_TIMEOUT`.
- Divergência entre estados ITSM e WhatsApp até reconciliação.
- Handoffs Teams por idade/estado (`QUEUED`, `ASSIGNED`, `REPLIED`, `CLOSED`).

### 5.2 Indicadores de experiência e negócio

- Percentual de interações resolvidas sem handoff humano.
- Tempo até primeira resposta útil por intenção.
- Redução de retrabalho por consultas repetidas de pedido/crédito.
- Taxa de roteamento correto na primeira tentativa.
- Adoção do canal WhatsApp pelos RTVs no piloto.
- Precisão factual da resposta (amostragem com validação por `sourceField`).

## 6. Critérios mínimos de conformidade para produção

1. Inbox/outbox duráveis com testes de concorrência, replay e reconciliação.
2. ABAC aplicado antes do fan-out e reforçado nas fontes quando possível.
3. NLU de catálogo fechado (`intents-v1`) com schema estrito e sem orquestração generativa de sistemas corporativos.
4. Resolução de cliente/pedido somente na carteira vigente do RTV autenticado.
5. Composição factual determinística ou LLM validada por `sourceField`.
6. Callback de handoff Teams autenticado, autorizado, idempotente e não repetível.
7. DLP ativo em todas as fronteiras e auditoria imutável segregada do ITSM.
8. Evidência formal de RIPD, retenção, descarte e exclusão propagada.

## 7. Decisão de governança

Com base na arquitetura vigente, o RTVgpt deve operar como uma camada de orquestração segura e auditável, não como um agente autônomo de decisão. O desenho aprovado combina produtividade (autoatendimento no WhatsApp), controle de risco (ABAC + step-up + DLP) e conformidade LGPD (minimização, rastreabilidade e gestão de ciclo de vida dos dados), preservando o Digibee e os sistemas de origem como autoridade de negócio.

