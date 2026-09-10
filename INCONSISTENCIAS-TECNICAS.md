# Parecer de inconsistências técnicas e arquitetura recomendada

## 1. Escopo e critério da análise

Foram analisados:

- `README.md`;
- `docs/detalhes-tecnicos-integracoes.md`;
- `docs/riscos-integracao.md`;
- `docs/validacao-rtv-cpf.md`.

Este parecer avalia a arquitetura documentada, não uma implementação executável. Os achados são classificados como:

- **Inconsistência**: dois trechos ou exemplos definem comportamentos incompatíveis.
- **Risco**: a solução proposta não sustenta, por si só, a garantia declarada.
- **Lacuna**: uma decisão necessária para implementação não foi especificada.

Prioridades:

- **P0 — crítico**: pode causar acesso indevido, perda/duplicação de mensagens ou inviabilizar o fluxo.
- **P1 — alto**: pode produzir dados incorretos, estados divergentes ou integrações inseguras.
- **P2 — médio**: reduz interoperabilidade, operação ou evolução segura.

## 2. Resumo executivo

A visão central — Digibee como hub de integração, Nina como camada conversacional e sistemas corporativos acessados somente por contratos controlados — é adequada. Porém, a documentação ainda não define uma arquitetura segura e implementável para produção.

Os principais bloqueadores são:

1. O webhook deve responder rapidamente ao WhatsApp, mas o fluxo coloca ITSM e Nina antes do `200 OK`.
2. O sistema diz continuar quando o ITSM falha, embora todos os passos seguintes exijam um `ticketId`.
3. A deduplicação descrita não é atômica e não impede corrida entre entregas simultâneas.
4. Atualizar ITSM e enviar ao WhatsApp são duas escritas independentes, sem outbox ou reconciliação.
5. Três dígitos de CPF não autenticam um RTV, e `rtvId` aparece em dados controláveis pela Nina/LLM.
6. O callback do Teams não possui contrato verificável de autenticação, ação e proteção contra replay.
7. O exemplo de Adaptive Card não estabelece um caminho funcional entre o clique no Teams e o Digibee.
8. Há exemplos em que a LLM produz fatos ausentes do payload recebido.
9. Não há ordenação causal por conversa; respostas podem chegar fora de ordem.
10. Contratos, estados e fontes de verdade não estão formalizados.

**Recomendação principal:** adotar uma arquitetura orientada a eventos com inbox/outbox duráveis, identidade corporativa forte, autorização por recurso, contratos versionados e efeitos externos idempotentes. ITSM deve ser uma projeção operacional, não a única fonte de auditoria ou a identidade primária da conversa.

## 3. Inconsistências e soluções

### P0.1 — ACK rápido do WhatsApp versus processamento síncrono

**Tipo:** inconsistência.

**Evidência:** no `README.md`, seção **8 / Fluxo A**, o Digibee deve responder `200` rapidamente, mas antes consulta o Object Store, cria ticket, grava correlação, adiciona comentário e encaminha à Nina. O diagrama também posiciona o `200 OK` após ITSM e Nina.

**Impacto:** lentidão ou indisponibilidade downstream provoca timeout e reentrega do webhook, ampliando duplicações e concorrência.

**Melhor solução:**

1. Validar tamanho, tipo e assinatura sobre os bytes originais.
2. Persistir o evento em uma **inbox durável** com chave única do `messageId`.
3. Retornar `200 OK` após a persistência confirmada.
4. Processar ITSM, Nina e demais integrações assincronamente.
5. Serializar o consumo por `conversationId`.

O `200` deve significar “evento aceito de forma durável”, não “fluxo de negócio concluído”.

### P0.2 — Continuidade sem ITSM versus obrigatoriedade de `ticketId`

**Tipo:** inconsistência.

**Evidência:** o `README.md`, seção **8 / Fluxo A**, afirma que falha na abertura do ticket não bloqueia a conversa. Entretanto, o evento enviado à Nina, o outbound, o fallback e o callback humano pressupõem um `ticketId`. `docs/detalhes-tecnicos-integracoes.md`, seção **2**, também define como saída do inbound um evento com `ticketId`.

**Impacto:** o estado “conversa ativa sem ticket” não cabe nos contratos atuais; fallback, auditoria e resposta tornam-se indefinidos.

**Melhor solução:** separar a identidade interna da conversa da referência externa:

- `conversationId`: obrigatório e criado antes de integrações externas;
- `ticketId`: opcional enquanto `ticketLinkStatus=PENDING|UNAVAILABLE`;
- mensagens e decisões persistidas em trilha durável;
- criação e reconciliação do ticket de forma assíncrona e idempotente;
- replay ordenado dos eventos no ITSM após recuperação.

Operações sensíveis podem exigir auditoria durável antes de prosseguir, mas essa auditoria não deve depender exclusivamente do ITSM.

### P0.3 — Idempotência não atômica

**Tipo:** risco.

**Evidência:** `README.md`, seção **8 / Modelo de correlação**, e `docs/detalhes-tecnicos-integracoes.md`, seção **3**, usam `itsm:msg:{messageId}` como deduplicação, mas descrevem “consultar e depois gravar”. Não há reserva atômica, estado de processamento ou recuperação de reserva abandonada.

**Impacto:** duas entregas concorrentes podem criar tickets ou comentários duplicados; uma falha após marcar a chave pode perder definitivamente o processamento.

**Melhor solução:**

- `put-if-absent`/restrição única antes dos efeitos;
- estados `RECEIVED`, `PROCESSING`, `COMPLETED` e `FAILED_RETRYABLE`;
- compare-and-set na correlação da conversa;
- lease/fencing token para retomada segura;
- `operationId` específico em cada escrita externa;
- reconciliação após timeout, pois timeout não prova que o destino não processou a chamada.

### P0.4 — Dupla escrita ITSM + WhatsApp

**Tipo:** inconsistência e lacuna.

**Evidência:** o `README.md`, seções **8 / Fluxo B** e **Fluxo Conversacional**, diz atualizar o ticket e só depois enviar ao WhatsApp. A seção **Resiliência e observabilidade** afirma que falha do ITSM não impede a entrega.

**Impacto:** uma política bloqueia a mensagem e a outra permite envio sem auditoria. Além disso:

- ITSM pode indicar `AGUARDANDO_USUARIO` mesmo se o WhatsApp falhar;
- retry pode duplicar comentário ou mensagem;
- aceite da API do WhatsApp não significa entrega ao usuário.

**Melhor solução:** criar comando outbound durável e efeitos independentes:

```text
OUTBOUND_ACCEPTED
  -> ITSM_COMMENT_PENDING/RECORDED
  -> WHATSAPP_SEND_PENDING/ACCEPTED
  -> DELIVERY_CONFIRMED/FAILED
  -> RECONCILED
```

Cada efeito usa `operationId` próprio. O estado do ticket só deve representar “aguardando usuário” após o marco de entrega definido pelo negócio. Webhooks de status do WhatsApp devem atualizar `sent`, `delivered`, `read` e `failed`.

### P0.5 — Três dígitos de CPF não validam identidade

**Tipo:** risco de segurança e inconsistência de linguagem.

**Evidência:** `docs/validacao-rtv-cpf.md` diz “garantir” a correspondência do usuário com o `rtvId`, mas reconhece que o prefixo não pode ser fator único. O `README.md`, seção **10 / Autenticação**, transforma o match em condição para acesso.

**Impacto:** existem apenas mil prefixos possíveis. O controle não prova posse do CPF, identidade corporativa, posse legítima do telefone nem resistência a SIM swap ou aparelho compartilhado. Hash simples de CPF também é enumerável.

**Melhor solução:**

- autenticação corporativa OIDC, Authorization Code + PKCE e MFA;
- vínculo servidor-side entre telefone, identidade imutável e `rtvId`;
- sessão curta com `auth_time`, nível de autenticação e versão das permissões;
- step-up MFA para crédito, dados financeiros e mutações;
- prefixo do CPF, se mantido, apenas como sinal antifraude, nunca como autorização;
- HMAC com chave em KMS/HSM quando uma correlação de CPF for indispensável.

### P0.6 — Identidade controlável pela Nina/LLM

**Tipo:** risco de segurança.

**Evidência:** exemplos do `README.md`, seção **10 / Contrato HTTP** e **Exemplos de Payload**, incluem `rtvId` dentro do payload de orquestração e até da saída do LLM.

**Impacto:** modelo generativo, prompt ou chamador pode selecionar a identidade usada na autorização, permitindo IDOR e acesso cruzado.

**Melhor solução:** a LLM retorna apenas intenção e entidades não confiáveis. O backend deriva `subjectId`, `rtvId`, tenant e permissões de claims validados e constrói a chamada ao Digibee. Aplicar ABAC com negação por padrão:

```text
subject autenticado
AND ação permitida
AND cliente pertence à carteira vigente
AND finalidade autorizada
AND nível de autenticação suficiente
```

A autorização deve ocorrer antes do fan-out e, quando possível, ser reforçada na fonte de dados.

### P0.7 — Callback Teams incompleto e potencialmente falsificável

**Tipo:** inconsistência e risco.

**Evidência:** o `README.md`, seção **9 / Callback do humano**, mostra somente `X-Correlation-Id`. O body permite informar `ticketId`, `userId`, `agent.id`, autor e texto. Os botões definem `assign`, `reply` e `close`, mas o callback usa apenas `action=reply_and_update_ticket`, sem carregar a ação humana.

**Impacto:** o contrato não distingue assumir, responder ou fechar e, implementado literalmente, permitiria personificação, replay, alteração de ticket e envio para destinatário escolhido pelo chamador.

**Melhor solução:**

- receber a interação por um Teams bot/app autenticado;
- validar token do Bot Framework/Entra (`iss`, `aud`, tenant, assinatura e validade);
- bot chamar Digibee com identidade de workload e escopo `handoff.callback`;
- resolver servidor-side ticket, conversa, destino e agente;
- usar `handoffEventId`, nonce de uso único, expiração e `handoffVersion`;
- separar `pipelineAction=process_handoff_event` de `handoffAction=assign|reply|close`;
- exigir ownership ou permissão de supervisor e usar compare-and-set.

### P0.8 — Adaptive Card não define um caminho executável

**Tipo:** inconsistência técnica.

**Evidência:** o `README.md`, seção **9 / Digibee -> Microsoft Graph**, publica um card via Graph e pressupõe que `Action.Submit` chame diretamente o callback Digibee. Também representa `attachments[].content` como objeto e não referencia o attachment no corpo da mensagem.

**Impacto:** os botões podem não funcionar e não existe componente responsável por receber e validar a ação.

**Melhor solução:** documentar uma topologia única: Digibee publica via Graph; Teams bot instalado recebe a atividade de ação; o bot valida o contexto e chama o endpoint interno. Usar o modelo de Universal Actions suportado, serializar corretamente o conteúdo do attachment e associá-lo ao `body.content`.

### P0.9 — Ausência de ordenação causal por conversa

**Tipo:** lacuna.

**Evidência:** a correlação usa `channel + userId`, mas não há sequência monotônica, versão da conversa ou política para resposta tardia.

**Impacto:** uma consulta lenta iniciada antes pode responder depois de uma consulta mais nova, regredindo contexto, ticket e experiência do usuário.

**Melhor solução:**

- `conversationSequence` monotônico;
- fila particionada por `conversationId`;
- `eventId`, `causationId` e `inReplyToMessageId` obrigatórios;
- compare-and-set por `conversationVersion`;
- política explícita para resposta `STALE`.

### P1.1 — Máquina de estados não cobre os próprios eventos

**Tipo:** inconsistência.

**Evidência:** `README.md`, seção **8 / Máquina de status**, não permite `ABERTO -> ESCALADO`, embora o fallback de segurança possa ocorrer imediatamente. O texto menciona reabertura sem transição saindo de `RESOLVIDO`. Estados de ticket, handoff e entrega aparecem misturados.

**Melhor solução:** definir três máquinas separadas e versionadas:

- ticket: `PROVISIONING`, `OPEN`, `PROCESSING`, `WAITING_USER`, `RESOLVED`;
- handoff: `QUEUED`, `ASSIGNED`, `REPLIED`, `CLOSED`;
- entrega: `PENDING`, `ACCEPTED`, `DELIVERED`, `READ`, `FAILED`.

Cada transição deve ter evento, precondição, responsável, versão e regra para evento tardio.

### P1.2 — Webhook nativo e evento normalizado foram misturados

**Tipo:** inconsistência.

**Evidência:** `README.md`, seção **8 / WhatsApp -> Digibee**, afirma aceitar o envelope nativo, mas o exemplo contém campos internos `pipeline`, `action` e `message`.

**Melhor solução:** documentar contratos distintos:

1. Meta/BSP → adapter Digibee, incluindo verificação `GET`, assinatura e envelope real;
2. adapter Digibee → Nina, com evento canônico normalizado.

Cloud API e BSP devem ter adapters separados quando headers ou envelopes forem diferentes.

### P1.3 — LLM produz fatos ausentes do payload

**Tipo:** inconsistência.

**Evidência:** no `README.md`, o contrato de composição do briefing contém apenas dados agregados e títulos de insights, mas a saída seguinte inclui números e valores de pedidos, produtos, comprador, anotação, ETA e condição comercial que não estavam na entrada. Isso contradiz “a Nina não inventa insight”.

**Impacto:** informação comercial falsa pode orientar decisões ou ser atribuída ao cliente errado.

**Melhor solução:** preferir renderer determinístico para fatos. Se a LLM permanecer:

- cada afirmação factual deve referenciar um `sourceField`;
- validador deve rejeitar nomes, números, datas e valores ausentes do payload;
- usar schema de saída estrito e fallback para template;
- não enviar histórico conversacional livre na etapa de composição.

### P1.4 — Dados declarados como mascarados aparecem completos

**Tipo:** inconsistência.

**Evidência:** o `README.md` exige CNPJ mascarado e proteção de score/crédito, mas payloads de briefing, desambiguação e consolidado mostram CNPJ e score completos. Respostas com dados financeiros usam `containsSensitiveData=false`.

**Melhor solução:** classificar dados antes do LLM e por destino. Substituir o booleano manual por classes calculadas, por exemplo `PERSONAL_IDENTIFIER`, `COMMERCIAL_CONFIDENTIAL` e `FINANCIAL_PROFILE`. Mascarar antes das fronteiras OpenAI, Teams, ITSM e WhatsApp conforme política de finalidade.

### P1.5 — Falta de minimização e governança LGPD

**Tipo:** lacuna.

**Evidência:** os documentos tratam principalmente de mascaramento de logs, mas não definem finalidade, base legal, retenção, descarte, operadores, transferência internacional, direitos do titular ou revisão de decisões automatizadas.

**Melhor solução:** criar inventário por processamento e destino, executar RIPD, definir retenção por categoria e política de exclusão propagada. OpenAI deve receber somente dados minimizados/tokenizados, com treinamento desabilitado, retenção e região aprovadas contratualmente. Segurança deve registrar eventos em trilha segregada de ITSM e Teams.

### P1.6 — ITSM tratado como auditoria e transcrição integral

**Tipo:** risco.

**Evidência:** toda mensagem e resposta é copiada para o ITSM; exemplos usam telefone no título e `visibility=public`.

**Impacto:** tickets podem ser editáveis, amplamente pesquisáveis e retidos por período incompatível com a finalidade. `public` pode expor conteúdo pessoal ou financeiro.

**Melhor solução:** armazenar no ITSM resumo mínimo, identificadores tokenizados e ACL por fila/finalidade. Manter auditoria imutável separada, com ator autenticado, decisão de autorização, ação, recurso, finalidade, timestamp e resultado.

### P1.7 — Retry genérico em operações de escrita

**Tipo:** risco.

**Evidência:** `README.md`, `docs/detalhes-tecnicos-integracoes.md` e `docs/riscos-integracao.md` recomendam retry para timeout/5xx, mas somente a criação de ticket exemplifica `Idempotency-Key`.

**Impacto:** timeout pode ocorrer após o destino confirmar internamente a operação; repetir pode duplicar comentários, mensagens, handoffs ou pedidos.

**Melhor solução:** definir `operationId` estável por efeito:

- `ticket-create:{inboundEventId}`;
- `ticket-comment:{eventId}`;
- `whatsapp-send:{outboundCommandId}`;
- `teams-handoff:{handoffId}`;
- `order-create:{confirmedDraftId}`.

Sem idempotência suportada pelo destino, consultar e reconciliar antes de repetir.

### P1.8 — Fonte de verdade e consistência temporal indefinidas

**Tipo:** lacuna.

**Evidência:** cliente vem de Lecom e TOTVS; pedidos vêm de Portal e TOTVS; não há precedência por campo, timestamp, versão ou regra de conflito. Consultas paralelas podem combinar snapshots de momentos diferentes.

**Melhor solução:** criar matriz de ownership por entidade/campo e incluir `source`, `sourceUpdatedAt`, `observedAt`, `version` e `staleness`. Definir `asOf` do consolidado e omitir ou sinalizar insights baseados em fonte fora da janela de frescor.

### P1.9 — Semântica de sucesso parcial e fallback ambígua

**Tipo:** inconsistência.

**Evidência:** a tabela de fallback inclui sistema indisponível; outros trechos aceitam `PARTIAL_SUCCESS` quando ainda existem dados utilizáveis.

**Melhor solução:** criar matriz por intenção com campos obrigatórios, opcionais, freshness máxima e comportamento para `NOT_FOUND`, `TIMEOUT`, `FORBIDDEN`, `STALE` e `PARTIAL_SUCCESS`.

### P1.10 — Contratos da OpenAI são, na verdade, contratos internos

**Tipo:** inconsistência de contrato.

**Evidência:** exemplos “Nina -> LLM (OpenAI)” usam `provider`, `operation`, `digibeeOutput` e `schemaName`, que não constituem diretamente requests da Responses API ou Chat Completions.

**Melhor solução:** renomear para “Nina → adapter LLM” e documentar separadamente o mapping para uma API escolhida, incluindo modelo, structured output, timeout, recusa e resposta incompleta.

### P1.11 — Intents e campos divergem entre exemplos

**Tipo:** inconsistência.

**Evidência:**

- `allowedIntents` contém `delivery_eta` e `credit_limit`, mas o exemplo retorna `delivery_eta_and_credit_limit`;
- `ticketId` aparece ora no envelope, ora dentro de `input`;
- o fluxo cita `input_text` e `channel_context`, enquanto o contrato usa `input.text`;
- botões Teams têm uma ação que desaparece no callback.

**Melhor solução:** escolher uma intent principal com `requestedTopics[]`, manter metadados no envelope e publicar JSON Schemas únicos usados para validar todos os exemplos.

### P2.1 — Regra e exemplo de insights divergem

**Tipo:** inconsistência.

**Evidência:** `README.md`, seção **10 / Motor de insights**, define `VISIT_GAP` para intervalo maior ou igual a 45 dias, mas o exemplo gera o código com 29 dias. A tabela cita `DELIVERY_EXCEPTION`/`OPEN_ORDERS`, enquanto o exemplo usa `OPEN_DELIVERY`.

**Melhor solução:** catálogo versionado de insights, enum oficial, condições determinísticas e testes automatizados dos exemplos.

### P2.2 — Primeira mensagem do ticket tem comportamento ambíguo

**Tipo:** inconsistência.

**Evidência:** o texto manda criar ticket e adicionar a mensagem como comentário; o request de criação já inclui a mensagem em `description`; o diagrama só adiciona comentário quando o ticket já existe.

**Melhor solução:** escolher e documentar uma única regra. A opção mais simples é manter a primeira mensagem na descrição e registrar eventos posteriores como comentários, sem duplicação.

### P2.3 — Conversa e expiração de sessão não estão definidas

**Tipo:** lacuna.

**Evidência:** a chave é `channel + userId`, embora o texto fale em sessão e encerramento por expiração. Não há TTL, tenant, política após resolução ou handoff aberto.

**Melhor solução:** usar `conversationId` opaco, namespace por tenant/ambiente, TTL explícito, máximo absoluto e regra de renovação. Não expirar conversa com handoff ativo; mensagem posterior a ticket resolvido deve criar nova conversa ou seguir regra formal de reabertura.

### P2.4 — Versionamento é recomendado, mas não implementado nos contratos

**Tipo:** lacuna.

**Evidência:** `docs/riscos-integracao.md` exige schemas versionados, porém não há OpenAPI, JSON Schema publicado, compatibilidade, depreciação ou `schemaVersion` consistente.

**Melhor solução:** OpenAPI 3.1 para HTTP, JSON Schema imutável por versão, testes de contrato produtor/consumidor e política de mudanças aditivas e depreciação.

### P2.5 — Respostas HTTP, datas e dinheiro não têm padrão

**Tipo:** lacuna.

**Evidência:** endpoints não definem códigos HTTP e erros; timestamps alternam epoch string, RFC 3339 e data civil; dinheiro usa número JSON sem moeda ou precisão.

**Melhor solução:**

- RFC 9457 para erros;
- RFC 3339 UTC para instantes;
- `YYYY-MM-DD` somente para datas civis, com timezone de negócio;
- dinheiro como decimal string ou `{amountMinor, currency}`;
- limites de payload, campos obrigatórios e `additionalProperties` definidos no schema.

## 4. Arquitetura-alvo recomendada

```text
WhatsApp / Teams
  -> validação criptográfica do canal
  -> inbox durável + deduplicação atômica
  -> ACK
  -> sessão corporativa OIDC/MFA
  -> identidade derivada no servidor
  -> autorização ABAC por ação, carteira e finalidade
  -> consumidor serial por conversationId
  -> Digibee com deadline e contratos versionados
  -> sistemas de origem autorizados
  -> consolidação com fonte, versão e freshness
  -> minimização de dados
  -> renderer determinístico ou LLM não confiável + validador factual
  -> outbox
       -> ITSM
       -> WhatsApp
       -> Teams
  -> webhooks de confirmação + reconciliação
```

Identificadores devem ter responsabilidades distintas:

| Identificador | Responsabilidade |
| --- | --- |
| `traceId` | Observabilidade distribuída |
| `conversationId` | Identidade interna da sessão |
| `eventId` | Identidade imutável de um evento |
| `causationId` | Relação causal entre eventos |
| `messageId` | Identidade fornecida pelo canal |
| `operationId` | Idempotência de um efeito externo |
| `ticketId` | Referência opcional ao ITSM |
| `handoffId` | Processo de atendimento humano |
| `entityVersion` | Concorrência otimista |

## 5. Ordem recomendada de correção

1. Autenticação forte do RTV e identidade derivada exclusivamente no servidor.
2. Autorização ABAC antes de qualquer consulta e remoção do CPF como fator autorizador.
3. Inbox/outbox duráveis, ACK assíncrono e idempotência por efeito.
4. `conversationId` independente do ITSM e política formal para indisponibilidade.
5. Topologia Teams bot → Digibee, callback autenticado e não repetível.
6. Máquinas separadas de ticket, handoff e entrega.
7. OpenAPI/JSON Schemas versionados e validação de todos os exemplos.
8. Renderer determinístico/validador factual e política de minimização por destino.
9. Matriz de fonte de verdade, freshness e resultado mínimo por intenção.
10. Governança LGPD, retenção, ACL, DLP e auditoria imutável.

## 6. Critérios mínimos para produção

- RTV autenticado por identidade corporativa e MFA, com sessão vinculada ao número.
- `rtvId`, ticket e destinatário resolvidos no servidor, nunca confiados à LLM.
- Autorização de carteira testada antes das consultas.
- Inbox/outbox duráveis e testes de concorrência, retry e recuperação.
- Callback Teams autenticado, autorizado, idempotente e vinculado ao handoff.
- Política explícita para ITSM indisponível.
- Contratos formais e versionados com testes de consumidor.
- Saída da LLM validada contra os dados de origem ou substituída por template.
- DLP e classificação antes de OpenAI, Teams, ITSM e WhatsApp.
- Retenção, RIPD, trilha imutável e direitos LGPD definidos.
- Métricas de backlog, idade da fila, duplicação, respostas obsoletas, divergência ITSM/WhatsApp, handoffs estagnados e freshness.

Até que os itens P0 sejam resolvidos, a arquitetura deve ser considerada inadequada para produção com dados cadastrais, financeiros e histórico comercial.
