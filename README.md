# Integração Digibee, Nina e Sistemas Corporativos

Este documento define a arquitetura de referência para o atendimento de RTVs pelo WhatsApp. O Digibee é o hub obrigatório de integração; a Nina interpreta solicitações e compõe respostas; Lecom, Portal de Pedidos, TOTVS/Datasul, Tarken, LoogAI e ITSM são acessados somente por contratos controlados.

Documentos complementares:

- [`docs/detalhes-tecnicos-integracoes.md`](docs/detalhes-tecnicos-integracoes.md): contratos, estados, idempotência e operação;
- [`docs/riscos-integracao.md`](docs/riscos-integracao.md): riscos, controles e critérios de produção;
- [`docs/validacao-rtv-cpf.md`](docs/validacao-rtv-cpf.md): autenticação corporativa, vínculo do RTV e uso restrito do CPF;
- [`docs/interpretacao-nina.md`](docs/interpretacao-nina.md): especificação complementar de NLU, provedores e resolução na carteira;
- [`docs/plano-implementacao-interpretacao-nina.md`](docs/plano-implementacao-interpretacao-nina.md): fases para tornar a interpretação do WhatsApp robusta.

## Princípios obrigatórios

1. Toda operação de negócio passa pelo Digibee.
2. O recebimento é assíncrono: `200 OK` significa somente que o evento foi validado e persistido de forma durável.
3. `conversationId` é a identidade interna obrigatória; `ticketId` é uma referência opcional ao ITSM.
4. Identidade e autorização são derivadas no servidor. A LLM nunca escolhe `rtvId`, destinatário, tenant ou recurso autorizado.
5. Cada efeito externo é idempotente e parte de uma outbox durável.
6. Eventos de uma conversa são consumidos em ordem.
7. ITSM é uma projeção operacional mínima, não a fonte primária de auditoria nem a identidade da conversa.
8. A LLM é um componente não confiável: fatos devem vir de campos de origem e passar por validação.
9. Dados são classificados e minimizados antes de atravessar qualquer fronteira.
10. Contratos HTTP e de eventos são versionados e validados.
11. A interpretação é catálogo fechado: menção não é identidade; fan-out só ocorre após resolução na carteira.

## Visão de componentes

| Camada | Responsabilidade |
| --- | --- |
| WhatsApp / Teams | Canais; emissão de eventos e recebimento de mensagens |
| Adapter de canal | Validação criptográfica e normalização do envelope nativo |
| Inbox | Aceite durável, unicidade de `messageId` e ordenação por conversa |
| Identidade | OIDC Authorization Code + PKCE, MFA, sessão e vínculo telefone–RTV |
| Autorização | ABAC por sujeito, ação, carteira, finalidade e nível de autenticação |
| Nina NLU | Intenção de catálogo e menções não confiáveis; Copilot ou OpenAI via adapter |
| Resolução | Cliente/pedido na carteira vigente; IDs nunca vêm da LLM |
| Nina composição | Template determinístico ou LLM + validador factual |
| Digibee | Orquestração, deadlines, contratos, consolidação e regras determinísticas |
| Sistemas de origem | Lecom, Portal, TOTVS, Tarken e LoogAI |
| Outbox | Comandos duráveis para ITSM, WhatsApp e Teams |
| ITSM | Projeção operacional com resumo mínimo e ACL |
| Auditoria | Trilha imutável e segregada, independente de ITSM e Teams |

```mermaid
flowchart LR
    C[WhatsApp / Teams] --> V[Validação criptográfica]
    V --> I[Inbox durável<br/>deduplicação atômica]
    I --> ACK[ACK 200]
    I --> Q[Fila particionada<br/>por conversationId]
    Q --> AUTH[OIDC/MFA + identidade servidor]
    AUTH --> ABAC[ABAC de conversa]
    ABAC --> NLU[Nina NLU<br/>catálogo + menções]
    NLU --> RES[Resolução na carteira]
    RES --> ABAC2[ABAC do recurso]
    ABAC2 --> D[Digibee]
    D --> S[Sistemas de origem]
    S --> D
    D --> CONS[Consolidação<br/>fonte, versão e freshness]
    CONS --> MIN[Minimização / DLP]
    MIN --> R[Renderer determinístico<br/>ou LLM + validador factual]
    R --> O[Outbox durável]
    O --> ITSM[ITSM]
    O --> W[WhatsApp]
    O --> T[Teams bot]
    ITSM --> REC[Confirmação e reconciliação]
    W --> REC
    T --> REC
```

## Identificadores e causalidade

| Identificador | Obrigatoriedade | Responsabilidade |
| --- | --- | --- |
| `traceId` | Sim | Observabilidade distribuída |
| `conversationId` | Sim | Identidade opaca da sessão |
| `eventId` | Sim | Identidade imutável do evento |
| `causationId` | Sim, salvo evento raiz | Evento que causou o evento atual |
| `messageId` | Em eventos de canal | Identidade fornecida pelo canal |
| `inReplyToMessageId` | Em respostas | Mensagem à qual a resposta se refere |
| `conversationSequence` | Sim | Sequência monotônica por conversa |
| `conversationVersion` | Sim | Compare-and-set do estado da conversa |
| `operationId` | Em cada efeito | Idempotência da escrita externa |
| `ticketId` | Não | Referência reconciliável ao ITSM |
| `handoffId` | Em handoff | Processo de atendimento humano |
| `entityVersion` | Quando aplicável | Concorrência otimista do recurso |

Eventos são particionados por `conversationId`. Uma resposta cuja versão já tenha sido superada recebe resultado `STALE`, é auditada e não altera contexto nem é enviada automaticamente.

## Fluxo de entrada do WhatsApp

Cloud API e cada BSP possuem adapters separados, pois headers e envelopes podem divergir.

1. O endpoint `GET` realiza a verificação exigida pelo provedor.
2. O `POST` limita tamanho e tipo e valida a assinatura sobre os **bytes originais**, antes de parsear JSON.
3. O adapter extrai o `messageId`, resolve ou cria `conversationId` e tenta inserir o evento na inbox com restrição única.
4. Após confirmação da persistência, responde `200 OK`.
5. Um consumidor com lease e fencing token adquire o próximo evento da conversa.
6. A sessão corporativa e a autorização são verificadas antes de qualquer consulta.
7. Ticket, Nina e integrações são processados assincronamente.

```mermaid
sequenceDiagram
    participant W as WhatsApp/BSP
    participant A as Adapter Digibee
    participant I as Inbox durável
    participant Q as Consumidor serial
    participant N as Nina
    participant D as Digibee
    participant T as ITSM

    W->>A: POST webhook (envelope nativo)
    A->>A: Limite + assinatura nos bytes originais
    A->>I: INSERT event (UNIQUE provider,messageId)
    I-->>A: Persistência confirmada ou duplicata conhecida
    A-->>W: 200 OK
    I->>Q: Evento canônico
    Q->>Q: OIDC/MFA, ABAC e sequência
    Q->>T: Provisionar/reconciliar ticket via outbox
    Q->>N: Evento canônico com ticketId opcional
    N->>N: NLU estruturada Copilot/OpenAI
    N->>D: Resolver menções na carteira
    D-->>N: Cliente autorizado ou FORBIDDEN
    N->>D: Intenção + menções não confiáveis
```

O `200` não declara conclusão do negócio. Payload inválido ou assinatura incorreta é rejeitado sem persistência; indisponibilidade da inbox não recebe ACK de sucesso. O exemplo ponta a ponta com cliente por nome, último pedido e ETA está em [Fluxo: estimativa de entrega do último pedido](#fluxo-estimativa-de-entrega-do-último-pedido).

### Evento canônico do adapter para a Nina

O envelope abaixo não é o webhook nativo:

```json
{
  "schemaVersion": "1.0.0",
  "eventId": "evt_01J...",
  "traceId": "trc_01J...",
  "conversationId": "cnv_01J...",
  "conversationSequence": 14,
  "conversationVersion": 8,
  "causationId": "evt_01J_previous",
  "messageId": "wamid.HBgL...",
  "occurredAt": "2026-09-10T01:40:00Z",
  "channel": "whatsapp",
  "input": {
    "text": "Qual a previsão de entrega do pedido 12345 e meu limite de crédito?"
  },
  "ticket": {
    "ticketId": null,
    "ticketLinkStatus": "PENDING"
  }
}
```

O número do telefone permanece em um cofre de identidade/canal. O evento usa identificadores tokenizados.

## Conversa independente do ITSM

`conversationId` é criado antes de integrações externas. A conversa pode seguir quando `ticketLinkStatus` for `PENDING` ou `UNAVAILABLE`; eventos e decisões permanecem na trilha durável e são reproduzidos no ITSM, em ordem, após recuperação.

| `ticketLinkStatus` | Significado | Comportamento |
| --- | --- | --- |
| `PENDING` | Provisionamento ainda não concluído | Conversa segue; outbox tenta criar o ticket |
| `LINKED` | `ticketId` confirmado | Projeções posteriores usam a referência |
| `UNAVAILABLE` | ITSM indisponível após tentativas | Conversa segue conforme política da intenção; reconciliador atua |
| `CONFLICT` | Resultado remoto incompatível | Suspende apenas efeitos dependentes e abre incidente operacional |

Operações sensíveis exigem que o evento de autorização esteja persistido na auditoria imutável, mas não dependem da disponibilidade do ITSM.

### Sessão

- chave interna: `tenantId + environment + conversationId + subjectId`;
- inatividade padrão: 30 minutos;
- duração absoluta: 8 horas;
- nova mensagem renova apenas o prazo de inatividade;
- handoff `QUEUED` ou `ASSIGNED` impede o encerramento da conversa, mas a credencial do RTV ainda pode expirar e exigir nova autenticação;
- mensagem posterior a uma conversa resolvida cria nova conversa;
- reabertura só ocorre por evento explícito e regra versionada.

## Inbox, idempotência e recuperação

Não se usa “consultar e depois gravar”. O aceite é uma inserção atômica `put-if-absent` ou `INSERT` com restrição única.

| Estado da inbox | Uso |
| --- | --- |
| `RECEIVED` | Evento persistido e elegível |
| `PROCESSING` | Consumidor possui lease e fencing token válidos |
| `COMPLETED` | Regras concluídas e efeitos registrados na outbox |
| `FAILED_RETRYABLE` | Nova tentativa permitida após o lease |
| `FAILED_FINAL` | Falha não recuperável, auditada e encaminhada |

Toda transição usa compare-and-set. Lease expirado permite retomada; fencing token impede que um consumidor antigo finalize depois do novo. Timeout de destino é resultado desconhecido, não prova de falha: consultar e reconciliar antes de repetir.

## Identidade e autorização do RTV

O RTV autentica-se no provedor corporativo com OIDC Authorization Code + PKCE e MFA. O servidor mantém vínculo verificado entre número de WhatsApp, `subjectId` imutável, tenant e `rtvId`.

A sessão curta registra `auth_time`, `acr`/nível de autenticação e versão das permissões. Crédito, dados financeiros e mutações exigem step-up MFA conforme política.

O backend deriva `subjectId`, `rtvId`, tenant, destinatário e permissões das claims validadas. Esses campos não são aceitos da Nina, da LLM nem do cliente.

```text
subject autenticado
AND ação permitida
AND cliente pertence à carteira vigente
AND finalidade autorizada
AND nível de autenticação suficiente
```

A autorização ABAC tem negação por padrão, ocorre antes do fan-out e é reforçada na fonte quando possível. Os três primeiros dígitos do CPF são somente sinal antifraude; não autenticam nem autorizam. Consulte o documento específico de validação.

## Interpretação da Nina

A Nina existente no Teams foi construída com Microsoft Copilot Studio. Esse agente permanece superfície de canal e de handoff humano. Ele **não** é a fonte de verdade da interpretação do WhatsApp e **não** escolhe, por orquestração generativa, quais sistemas corporativos consultar.

O runtime de interpretação (`nina-nlu`) é um classificador de catálogo fechado. Ele devolve uma intenção, tópicos e menções não confiáveis; o Digibee resolve cliente ou pedido na carteira do RTV; só então ocorre o fan-out já previsto nesta arquitetura. Copilot (Azure OpenAI / Microsoft Foundry) e OpenAI são provedores atrás do mesmo adapter interno.

Análise de crédito é decisão assistida com fatos de origem, não aprovação de pedido.

### Fronteiras de confiança

| Componente | Confia em | Não confia em | Efeito permitido |
| --- | --- | --- | --- |
| Adapter de canal | Assinatura e `messageId` | Texto do usuário como autoridade | Persistência na inbox |
| Identidade / ABAC de conversa | Claims OIDC e vínculo servidor-side | `rtvId`, tenant ou carteira no texto | Autorizar o uso da Nina neste canal |
| Extratores determinísticos | Parsers versionados (`pt-BR`) | Valor ambíguo da LLM | Hints de pedido, documento e dinheiro |
| Adapter NLU | Schema e catálogo `intents-v1` | IDs, identidade e tools inventados | Intenção + menções |
| Resolução na carteira | `rtvId` de servidor + TOTVS/Lecom filtrados | Nome cru ou `customerId` da LLM | 0, 1 ou N candidatos da carteira |
| ABAC do recurso | Policy engine e AAL | Intenção “mais permissiva” do modelo | Fan-out Digibee ou recusa genérica |
| Digibee / origem | Contratos e freshness | Texto livre e Copilot Studio | Consultas e mutações autorizadas |
| Composição | `sourceField` e consolidado | Histórico conversacional livre | Mensagem validada na outbox |

```mermaid
flowchart TD
    subgraph canal [Canal]
        M[Evento canônico WhatsApp]
    end
    subgraph confianca [Confiança de servidor]
        S{Sessão OIDC válida?}
        A1{ABAC de conversa}
    end
    subgraph nlu [Interpretação não confiável]
        HYB[Extratores determinísticos]
        LLM[Adapter NLU Copilot ou OpenAI]
        VAL[Schema, catálogo e guardrails]
    end
    subgraph resolucao [Resolução autorizada]
        RES[Cliente ou pedido na carteira]
        A2{ABAC do recurso + AAL}
    end
    subgraph negocio [Efeito de negócio]
        ORCH[Fan-out Digibee da intenção]
        CONS[Consolidação e insights]
        R[Renderer ou LLM + validador]
    end

    M --> S
    S -->|Não| AUTH[AUTHENTICATION_REQUIRED]
    S -->|Sim| A1
    A1 -->|Não| DENY[FORBIDDEN genérico]
    A1 -->|Sim| HYB
    HYB --> LLM
    LLM --> VAL
    VAL -->|Recusa / baixa confiança| CLAR[Clarificação ou out_of_scope]
    VAL -->|ok| RES
    RES -->|0 na carteira| DENY
    RES -->|N candidatos| CLAR
    RES -->|1 candidato| A2
    A2 -->|STEP_UP| MFA[Solicitar MFA]
    A2 -->|Não| DENY
    A2 -->|Sim| ORCH
    ORCH --> CONS
    CONS --> R
```

Ordem obrigatória: identidade e ABAC de conversa → minimização do texto → extração determinística → NLU estruturada → validação de schema e guardrails → resolução filtrada pela carteira → ABAC do recurso e step-up → fan-out Digibee → composição factual. A busca de cliente fora da carteira não é consulta permitida: zero resultado na carteira e cliente existente em outra carteira produzem a mesma resposta genérica.

### Papel do Copilot Studio

| Superfície | Papel permitido | Papel proibido |
| --- | --- | --- |
| Teams bot atual | Canal, Adaptive Card e handoff humano | Orquestrar fan-out, escolher `rtvId` ou cliente, responder fatos sem validador |
| AI prompts / Foundry | Provedor de NLU com JSON estruturado, atrás do adapter | Schema aberto, texto livre como autoridade, grounding em Dataverse com PII |
| Generative orchestration | Desligada para WhatsApp e sistemas corporativos | Encadear tools contra ERP, crédito ou ITSM |
| Conectores HTTP/MCP | Somente o gateway Nina/Digibee já autenticado | OpenAPI dos sistemas de origem publicado ao agente |

O inventário do agente atual (tópicos, frases de gatilho, entidades, actions e knowledge) é insumo de exemplos e vocabulário. Não bloqueia o runtime novo e não vira contrato.

### Contrato Nina → adapter NLU

Envelope interno, distinto do request nativo de Copilot ou OpenAI. Campos ausentes de propósito: `rtvId`, `subjectId`, tenant, telefone, carteira, nomes de outros clientes e histórico conversacional livre.

```json
{
  "schemaVersion": "1.0.0",
  "operation": "interpret_utterance",
  "traceId": "trc_01J...",
  "conversationId": "cnv_01J...",
  "locale": "pt-BR",
  "catalogVersion": "intents-v1",
  "input": {
    "text": "Preciso de uma análise de crédito para o cliente Hommerson Agro para ver se ele consegue fazer um pedido de 1milhão."
  },
  "deterministicHints": {
    "amounts": [
      {
        "raw": "1milhão",
        "amountMinor": 100000000,
        "currency": "BRL"
      }
    ],
    "orderNumbers": [],
    "taxIds": []
  },
  "pendingClarification": null
}
```

Saída em JSON Schema estrito (`additionalProperties: false`):

```json
{
  "schemaVersion": "1.0.0",
  "catalogVersion": "intents-v1",
  "intent": "credit_analysis",
  "requestedTopics": ["credit_limit", "credit_available", "credit_check_amount", "overdue_titles"],
  "mentions": {
    "customer": {
      "raw": "Hommerson Agro",
      "type": "CUSTOMER_NAME"
    },
    "requestedOrderAmount": {
      "raw": "1milhão",
      "amountMinor": 100000000,
      "currency": "BRL"
    }
  },
  "confidence": 0.91,
  "needsClarification": false,
  "clarificationCode": null,
  "guardrailFlags": []
}
```

| Campo | Autoridade |
| --- | --- |
| `intent`, `requestedTopics`, `mentions.*.raw` | LLM, não confiável |
| `mentions.*.amountMinor` | LLM só se coincidir com `deterministicHints`; senão clarificação |
| `confidence` | Sinal; limiar versionado no servidor |
| IDs de cliente, pedido, RTV | Nunca aceitos da LLM |
| `guardrailFlags` | LLM pode sugerir; o motor de guardrail do servidor decide |

Recusa, timeout, JSON inválido, propriedade extra, intenção fora do enum ou confiança abaixo do limiar viram `NLU_REJECTED` e seguem política de clarificação, não fan-out.

### Catálogo `intents-v1`

Uma intenção principal por turno. Tópicos extras entram em `requestedTopics[]`.

| Intenção | Quando usar | Tópicos | Fan-out após ABAC | Parcial |
| --- | --- | --- | --- | --- |
| `credit_analysis` | Limite, score, “cabe um pedido de X” | `credit_limit`, `credit_available`, `credit_check_amount`, `overdue_titles` | Tarken + títulos TOTVS | Não |
| `order_query` | Status, entrega, itens de um pedido | `delivery_eta`, `credit_limit`, `order_status` | TOTVS, LoogAI, crédito só se pedido | Sim, por tópico |
| `visit_preparation` | Briefing de visita | visitas, pedidos, crédito, logística | Consolidado já especificado | Sim |
| `customer_lookup` | Cadastro básico autorizado | cadastro | Lecom/TOTVS cadastral | Sim |
| `customer_update` | Alterar cadastro | campos declarados | Mutação com MFA | Não |
| `order_create` | Criar pedido | rascunho | Outbox após confirmação | Não |
| `clarification_response` | Resposta a pergunta da Nina | herda a intenção pendente | Retoma o fluxo pendente | Conforme a original |
| `human_handoff_request` | Pedido explícito de humano | — | Handoff Teams | — |
| `out_of_scope` | Fora do atendimento RTV | — | Sem fan-out | — |

`credit_analysis` não cria pedido. “Consegue fazer um pedido de 1 milhão” é checagem de valor contra fatos financeiros, com insight determinístico.

### Extração híbrida

| Sinal | Método | Exemplo |
| --- | --- | --- |
| Valor monetário | Parser `pt-BR` versionado | `1milhão`, `1 milhão`, `1.000.000`, `R$ 1m` → `100000000` BRL |
| Número de pedido | Regex de catálogo | `12345`, `pedido 12345` |
| CNPJ/CPF | Detector; tokenização imediata | nunca enviado completo ao modelo nem ao WhatsApp |
| Nome de cliente | LLM + busca na carteira | `Hommerson Agro` |
| Datas | Parser civil + timezone de negócio | `hoje`, `10/09` |

```mermaid
flowchart LR
    T[Texto do turno] --> P[Parser determinístico]
    T --> L[LLM estruturada]
    P --> F[Fusão versionada]
    L --> F
    F -->|Unívoco| M[Menções + hints]
    F -->|Conflito ou ambíguo| C[Clarificação]
    F -->|Fora do catálogo| O[out_of_scope / NLU_REJECTED]
```

O parser de valores aceita sufixos comuns (`mil`, `milhão`/`milhões`, `k`, `m`) e rejeita entradas ambíguas (`1,000` sem contexto de milhar versus decimal). Se a LLM devolver um `amountMinor` diferente do parser, prevalece o parser somente quando o texto original casa de forma unívoca; caso contrário, clarifica.

### Resolução de entidades

Menção (`Hommerson Agro`) não é identidade. Identidade de cliente só existe depois da resolução filtrada pela carteira vigente. A LLM não participa desta etapa.

```json
{
  "schemaVersion": "1.0.0",
  "operation": "resolve_customer_mention",
  "traceId": "trc_01J...",
  "mention": {
    "raw": "Hommerson Agro",
    "type": "CUSTOMER_NAME"
  }
}
```

O gateway acrescenta o contexto interno assinado (`rtvId`, tenant, finalidade). O Digibee consulta a carteira vigente no TOTVS e, se necessário, o cadastro Lecom **já filtrado**.

```mermaid
flowchart TD
    MEN[Menção textual] --> GW[Gateway assina rtvId de servidor]
    GW --> D[Digibee: carteira TOTVS + Lecom filtrado]
    D --> R{Candidatos na carteira}
    R -->|1 acima do limiar| ABAC[ABAC do recurso]
    R -->|N| AMB[AMBIGUOUS_CUSTOMER]
    R -->|0| FORB[FORBIDDEN genérico]
    R -->|Timeout| LIM[Limitação sem fan-out financeiro]
    AMB --> WHATS[Lista curta: nome fantasia e cidade]
    FORB --> GEN[Mesma resposta se existir fora da carteira]
```

| Resultado | Ação |
| --- | --- |
| 1 candidato acima do limiar | Seguir para ABAC do recurso |
| N candidatos | `clarificationCode=AMBIGUOUS_CUSTOMER` com rótulos mínimos da carteira |
| 0 na carteira | `FORBIDDEN` genérico; não informar se o nome existe em outra carteira |
| Timeout da busca | Sem fan-out financeiro; informar limitação |

Rótulos de clarificação usam nome fantasia autorizado e, se preciso, cidade. Não incluem CNPJ completo, limite, score nem indício de clientes fora da carteira. Pedido mencionado é resolvido da mesma forma: precisa pertencer a cliente da carteira.

### Guardrails na interpretação

Os controles de identidade, ABAC e DLP aplicam-se antes e depois da NLU. A interpretação acrescenta:

| Código | Sinal | Efeito |
| --- | --- | --- |
| `PROMPT_INJECTION` | Alterar sistema, listar tools, ignorar catálogo | Recusa, auditoria `SECURITY_EVIDENCE`, resposta genérica |
| `CROSS_PORTFOLIO` | Cliente “de outro RTV”, dump de carteira | Mesmo tratamento de `FORBIDDEN` |
| `IDENTITY_SPOOF` | Usuário informa `rtvId`, CPF de colega, tenant | Ignorar menção; identidade permanece a da sessão |
| `PII_EXFILTRATION` | Pedido para repetir CPF, telefone, token | Recusa e DLP |
| `OUT_OF_SCOPE` | RH, TI pessoal, outros negócios | `out_of_scope` sem fan-out |
| `LOW_CONFIDENCE` | Abaixo do limiar versionado | Clarificação |
| `UNGROUNDED_ID` | LLM emitiu código de cliente/pedido | Descartar ID; resolver só pela menção textual |

O WhatsApp nunca recebe o motivo interno. `FORBIDDEN` e cliente inexistente na carteira são indistinguíveis para o usuário. Crédito, limite, score e títulos exigem AAL financeiro; a NLU não reduz esse requisito.

### Provedores LLM

```text
nina-nlu
  -> provider router (política versionada)
       -> microsoft_copilot (Azure OpenAI / Microsoft Foundry)
       -> openai (API aprovada)
```

```mermaid
flowchart LR
    N[nina-nlu] --> R[Router versionado]
    R --> C[microsoft_copilot]
    R --> O[openai]
    C --> V[Validador de schema]
    O --> V
    V -->|ok| I[Intenção + menções]
    V -->|recusa / timeout / extra| X[NLU_REJECTED]
    R -->|429 / 5xx / transporte| F[Failover para reserva]
    F --> V
```

| Tópico | Regra |
| --- | --- |
| Contrato interno | Sempre o envelope Nina → adapter |
| Contrato externo | Structured output nativo de cada provedor; schema equivalente |
| Modelo | Allowlist; sem troca silenciosa de família |
| Timeout | Envelopado; estouro = `NLU_TIMEOUT` |
| Failover | Só para o provedor reserva se a falha for de transporte/`429`/`5xx`; não por “não gostei da intenção” |
| Divergência | Canário compara intenções; divergência alta alerta, não escolhe a mais permissiva |
| Dados | Treinamento desabilitado, retenção mínima, região aprovada, DLP antes do envio |
| Composição | Etapa separada; histórico livre continua proibido |

Copilot Studio AI prompts podem implementar o provedor `microsoft_copilot` somente se o JSON for revalidado no adapter contra o mesmo schema. O formato auto-detectado do Studio **não** substitui o JSON Schema versionado. A etapa de composição não reutiliza o mesmo prompt de NLU.

### Clarificação

A conversa guarda um slot pendente versionado (`pendingClarification`), não um chat livre para a LLM.

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> PENDING: needsClarification
    PENDING --> RESUME: clarification_response válida
    PENDING --> IDLE: nova intenção cancela o slot
    PENDING --> IDLE: TTL de sessão expirado
    RESUME --> IDLE: resolução 1 candidato ou recusa
```

| Código | Pergunta ao RTV | Retomada |
| --- | --- | --- |
| `MISSING_CUSTOMER` | Qual cliente? | Nova menção → resolução |
| `AMBIGUOUS_CUSTOMER` | Lista curta da carteira | `clarification_response` com índice ou nome |
| `MISSING_AMOUNT` | Qual valor do pedido a checar? | Parser + NLU |
| `AMBIGUOUS_AMOUNT` | Confirmar o valor em reais | Parser |
| `LOW_CONFIDENCE` | Reformular o pedido | Nova interpretação |
| `NLU_REJECTED` / `NLU_TIMEOUT` | Não entendi; oferecer opções do catálogo | Sem fan-out |

Timeout de slot segue o TTL de sessão. Mensagem nova que muda de intenção cancela o slot e reinterpreta.

### Exemplo: Hommerson Agro, R$ 1 milhão

Utterance:

```text
Preciso de uma análise de crédito para o cliente Hommerson Agro para ver se ele consegue fazer um pedido de 1milhão.
```

```mermaid
sequenceDiagram
    participant U as RTV no WhatsApp
    participant N as nina-nlu
    participant P as Parser pt-BR
    participant L as Adapter Copilot/OpenAI
    participant D as Digibee
    participant K as Tarken
    participant T as TOTVS
    participant R as Renderer

    U->>N: Evento canônico autorizado
    N->>N: ABAC de conversa
    N->>P: 1milhão
    P-->>N: 100000000 BRL
    N->>L: interpret_utterance
    L-->>N: credit_analysis + Hommerson Agro
    N->>D: resolve_customer_mention na carteira
    alt Fora da carteira
        D-->>N: 0 candidatos
        N-->>U: FORBIDDEN genérico sem Tarken
    else AAL insuficiente
        N-->>U: STEP_UP_REQUIRED
    else Autorizado
        D->>K: limite e disponibilidade
        D->>T: títulos
        D->>R: CREDIT_* + OVERDUE_TITLES
        R-->>U: Fatos lastreados, sem aprovacao
    end
```

| Passo | Componente | Resultado arquitetural |
| --- | --- | --- |
| 1 | Sessão + ABAC de conversa | RTV autenticado pode usar a Nina neste canal |
| 2 | Parser | `1milhão` → `100000000` BRL |
| 3 | NLU | `credit_analysis`, menção `Hommerson Agro`, tópicos de limite/títulos |
| 4 | Resolução | Somente carteira do `rtvId` de servidor |
| 5 | Fora da carteira | Resposta genérica, evento de segurança, **nenhuma** chamada Tarken |
| 6 | AAL financeiro insuficiente | Step-up, sem consultar crédito |
| 7 | Autorizado | Tarken e TOTVS em paralelo, com deadlines e freshness de 5 min |
| 8 | Insights | `CREDIT_INSUFFICIENT` ou `CREDIT_SUFFICIENT_FOR_AMOUNT`; reuso de `OVERDUE_TITLES` / `CREDIT_NEAR_LIMIT` |
| 9 | Composição | Renderer determinístico com `asOf`; sem frase do tipo “está aprovado” |

A comparação `requestedOrderAmount` versus limite disponível só ocorre com bloco Tarken `SUCCESS` dentro da janela. Esses códigos não são aprovação de crédito.

## Orquestração e contratos de intenção

O runtime `nina-nlu` classifica a utterance contra o catálogo `intents-v1` e devolve menções não confiáveis. O Digibee resolve cliente ou pedido na carteira e só então constrói as chamadas. A LLM retorna uma intenção principal e `requestedTopics[]`. Metadados permanecem no envelope. IDs de cliente, `rtvId` e códigos emitidos pelo modelo são descartados.

O envelope abaixo é o evento pós-validação enviado à orquestração, não a saída crua do modelo:

```json
{
  "schemaVersion": "1.0.0",
  "eventId": "evt_01J...",
  "traceId": "trc_01J...",
  "conversationId": "cnv_01J...",
  "conversationSequence": 14,
  "conversationVersion": 8,
  "causationId": "evt_01J_inbound",
  "ticketId": null,
  "intent": "credit_analysis",
  "requestedTopics": ["credit_limit", "credit_available", "credit_check_amount", "overdue_titles"],
  "mentions": {
    "customer": {
      "raw": "Hommerson Agro",
      "type": "CUSTOMER_NAME"
    },
    "requestedOrderAmount": {
      "raw": "1milhão",
      "amountMinor": 100000000,
      "currency": "BRL"
    }
  }
}
```

`mentions` é não confiável. O backend resolve cliente e pedido na carteira, acrescenta o contexto de segurança derivado no servidor e só então constrói as chamadas.

### Matriz de resultado mínimo

| Intenção | Obrigatório | Opcional | Freshness máxima | `NOT_FOUND` | `TIMEOUT` / `STALE` | `PARTIAL_SUCCESS` |
| --- | --- | --- | --- | --- | --- | --- |
| `credit_analysis` | Cliente na carteira e AAL financeiro | Checagem de valor pedido | Crédito 5 min; títulos 5 min | Resposta genérica, sem revelar cliente fora da carteira | Não decidir capacidade; sinalizar limitação | Não permitido |
| `order_query` | Pedido autorizado | ETA, crédito | Pedido 5 min; ETA 15 min; crédito 5 min | Informar ausência sem revelar cliente | Omitir tópico e sinalizar indisponibilidade | Responder apenas tópicos válidos |
| `visit_preparation` | Cliente autorizado | Visitas, pedidos, crédito, logística | Cadastro 24 h; demais 15 min | Handoff se cliente não puder ser resolvido | Omitir bloco e indicar limitação | Briefing com blocos válidos |
| `customer_lookup` | Cliente autorizado | Cadastro | Cadastro 24 h | Resposta genérica | Omitir bloco | Responder campos válidos |
| `customer_update` | Cliente, campos e MFA | — | Autorização em tempo real | Não executar | Não repetir sem reconciliação | Não permitido |
| `order_create` | Rascunho confirmado, MFA e versão | — | Autorização em tempo real | Não executar | Consultar por `operationId` | Não permitido |

`FORBIDDEN` nunca vira `NOT_FOUND` detalhado nem expõe existência do recurso. Gera resposta genérica e evento de segurança.

## Fluxo: estimativa de entrega do último pedido

Pergunta típica no WhatsApp:

> Quero saber a estimativa de entrega do último pedido do cliente Fazenda Esperança

Não há `orderNumber`. A Nina extrai intenção e entidades não confiáveis; o Digibee resolve cliente, carteira, último pedido e ETA no servidor. O `200` do webhook só confirma persistência na inbox; a resposta ao RTV sai depois, pela outbox.

```mermaid
sequenceDiagram
    participant U as RTV no WhatsApp
    participant A as Adapter Digibee
    participant I as Inbox durável
    participant Q as Consumidor serial
    participant N as Nina
    participant D as Digibee
    participant L as Lecom
    participant T as TOTVS Datasul
    participant G as LoogAI
    participant R as Renderer validador
    participant O as Outbox
    participant W as WhatsApp
    participant S as ITSM

    U->>A: POST webhook (envelope nativo)
    A->>A: Limite + assinatura nos bytes originais
    A->>I: INSERT UNIQUE provider,messageId
    A-->>U: 200 OK
    I->>Q: Evento canônico por conversationId
    Q->>Q: OIDC/MFA, vínculo telefone-RTV e ABAC
    Q->>O: ticket-create:{eventId}
    Q->>N: Evento canônico
    N->>N: NLU: order_query + delivery_eta + menções
    N->>D: Resolver Fazenda Esperança na carteira
    D->>L: Cadastro filtrado pelo rtvId de servidor
    D->>T: Carteira vigente e último pedido autorizado
    D->>G: ETA/tracking
    D->>R: Consolidado versionado com proveniência
    R->>O: OUTBOUND_ACCEPTED
    par Efeitos independentes
        O->>W: whatsapp-send:{outboundCommandId}
        O->>S: ticket-comment:{eventId}
    end
    W-->>O: ACCEPTED / DELIVERED / READ / FAILED
```

### 1. Entrada e evento canônico

O adapter valida o envelope nativo, persiste o evento e devolve `200`. O telefone permanece no cofre de identidade. O texto segue tokenizado:

```json
{
  "schemaVersion": "1.0.0",
  "eventId": "evt_01J...",
  "traceId": "trc_01J...",
  "conversationId": "cnv_01J...",
  "conversationSequence": 14,
  "conversationVersion": 8,
  "causationId": "evt_01J_previous",
  "messageId": "wamid.HBgL...",
  "occurredAt": "2026-09-10T01:40:00Z",
  "channel": "whatsapp",
  "input": {
    "type": "text",
    "text": "Quero saber a estimativa de entrega do ultimo pedido do cliente Fazenda Esperança"
  },
  "ticket": {
    "ticketId": null,
    "ticketLinkStatus": "PENDING"
  }
}
```

Sem sessão OIDC válida, o fluxo interrompe e o WhatsApp recebe apenas o link de autenticação. A conversa não espera o ITSM: `ticketLinkStatus` pode permanecer `PENDING` ou `UNAVAILABLE`.

### 2. Intenção extraída pela Nina

A NLU devolve menções, não identidade. `LAST` é um seletor de pedido, não um `orderNumber` autorizado.

```json
{
  "schemaVersion": "1.0.0",
  "intent": "order_query",
  "requestedTopics": ["delivery_eta"],
  "mentions": {
    "customer": {
      "raw": "Fazenda Esperança",
      "type": "CUSTOMER_NAME"
    },
    "orderSelector": "LAST"
  }
}
```

`mentions.customer.raw` e `orderSelector` não autorizam nada. O gateway resolve o cliente na carteira, escolhe o último pedido autorizado e acrescenta o contexto interno derivado das claims (`subjectId`, `rtvId`, tenant, AAL, finalidade). Esse bloco não volta para o modelo e não é aceito da LLM.

Consulta de pedido e entrega exige AAL2 e a regra ABAC completa, avaliada **antes** do fan-out:

```text
subject autenticado
AND ação permitida
AND cliente pertence à carteira vigente
AND pedido pertence a cliente autorizado
AND finalidade autorizada
AND nível de autenticação suficiente
```

### 3. Resolução no Digibee

| Passo | Fonte | Responsabilidade |
| --- | --- | --- |
| Resolver o cliente pelo nome | Lecom | Cadastro fiscal; nome ambíguo não gera match inventado |
| Autorizar a carteira | TOTVS/Datasul | Cliente na carteira vigente do RTV; Lecom não amplia autorização |
| Selecionar o último pedido | TOTVS | Pedido integrado mais recente **autorizado**; Portal só durante captura |
| Obter a estimativa de entrega | LoogAI | Tracking/ETA; faturamento permanece no TOTVS |

Freshness desta intenção: pedido 5 min; ETA 15 min. Cada bloco consolidado inclui `source`, `sourceUpdatedAt`, `observedAt`, `version` e `staleness`. Dados fora da janela são omitidos ou marcados `STALE`.

| Resultado | Tratamento no WhatsApp |
| --- | --- |
| Cliente/pedido autorizado e ETA fresco | Responder com fatos lastreados |
| Nome irresolúvel ou ambíguo | Não inventar cliente; handoff se a política da intenção exigir |
| `NOT_FOUND` | Informar ausência sem revelar o cliente |
| `FORBIDDEN` | Resposta genérica e evento de segurança; não expor existência nem carteira |
| `TIMEOUT` / `STALE` da LoogAI | Omitir `delivery_eta` e sinalizar indisponibilidade |
| Pedido ok e ETA indisponível | `PARTIAL_SUCCESS`: só tópicos válidos |

### 4. Composição e envio

Fatos preferem template determinístico. Se houver LLM, cada segmento exige `sourceField`; o validador rejeita nome, número ou data ausentes no consolidado. Recusa ou falha usa o renderer. DLP classifica o payload como `COMMERCIAL_CONFIDENTIAL` antes de WhatsApp, ITSM e OpenAI.

```json
{
  "schemaVersion": "1.0.0",
  "message": {
    "segments": [
      {
        "text": "O último pedido autorizado está liberado.",
        "sourceField": "$.pedido.data.statusErp"
      },
      {
        "text": "A previsão de entrega é 2026-09-10.",
        "sourceField": "$.logistica.previsaoEntrega"
      }
    ]
  },
  "dataClasses": ["COMMERCIAL_CONFIDENTIAL"]
}
```

ITSM e WhatsApp são efeitos independentes da outbox (`ticket-comment:{eventId}` e `whatsapp-send:{outboundCommandId}`). Falha de ITSM não bloqueia o envio. O ticket só entra em `WAITING_USER` no marco `DELIVERED`. Resposta cuja `conversationVersion` já foi superada recebe `STALE` e não é enviada.

## Fonte de verdade e consistência temporal

| Entidade/campo | Fonte de verdade | Fontes auxiliares | Regra de conflito |
| --- | --- | --- | --- |
| Identidade RTV e vínculo | IAM corporativo | Diretório de pessoas | IAM prevalece |
| Carteira vigente | TOTVS/Datasul | Lecom | versão vigente do TOTVS |
| Cadastro fiscal | Lecom | TOTVS | ownership definido por campo |
| Estado financeiro/crédito | Tarken | TOTVS | Tarken para decisão; TOTVS para títulos |
| Pedido | TOTVS | Portal | TOTVS após integração; Portal durante captura |
| Entrega/ETA | LoogAI | TOTVS | LoogAI para tracking; TOTVS para faturamento |
| Conversa/eventos | Event store | ITSM | event store prevalece |

Todo bloco consolidado inclui `source`, `sourceUpdatedAt`, `observedAt`, `version` e `staleness`. A resposta possui `asOf`; dados fora da janela são omitidos ou marcados `STALE`, nunca combinados silenciosamente.

Dinheiro usa `{ "amountMinor": 5000000, "currency": "BRL" }`. Instantes usam RFC 3339 UTC; datas civis usam `YYYY-MM-DD` e timezone de negócio explícito.

## Insights e composição

Insights são determinísticos, versionados e testados:

| Código | Condição |
| --- | --- |
| `CREDIT_NEAR_LIMIT` | uso do limite maior ou igual a 80% |
| `CREDIT_INSUFFICIENT` | valor pedido maior que o disponível autorizado |
| `CREDIT_SUFFICIENT_FOR_AMOUNT` | valor pedido menor ou igual ao disponível, sem fato de bloqueio na origem |
| `OVERDUE_TITLES` | há título vencido autorizado |
| `DELIVERY_EXCEPTION` | entrega possui ocorrência |
| `OPEN_ORDERS` | há pedidos ainda abertos |
| `VISIT_GAP` | 45 dias ou mais desde a última visita |
| `VOLUME_DROP` | volume da janela inferior ao período comparável |
| `MISSING_RECURRING_SKU` | item recorrente ausente na janela atual |
| `TALKING_POINT` | pauta derivada de um ou mais códigos anteriores |

Fatos devem ser renderizados por template sempre que possível. Se uma LLM for usada, cada afirmação factual referencia `sourceField`; schema estrito e validador rejeitam nomes, números, datas ou valores ausentes. Em falha ou recusa, usa-se renderer determinístico. Histórico conversacional livre não é enviado à etapa de composição.

“Nina → adapter LLM” é um contrato interno, usado na NLU e, se necessário, na composição. O adapter mapeia para o provedor `microsoft_copilot` (Azure OpenAI / Microsoft Foundry) ou `openai`, com modelo em allowlist, structured output, timeout, recusa e resposta incompleta. Esse envelope não é o request nativo da Responses API nem o JSON auto-detectado do Copilot Studio. O bot Copilot no Teams permanece canal e handoff; não orquestra sistemas de origem.

Exemplo de saída validável:

```json
{
  "schemaVersion": "1.0.0",
  "message": {
    "segments": [
      {
        "text": "O pedido 12345 está liberado.",
        "sourceField": "$.pedido.statusErp"
      },
      {
        "text": "A previsão de entrega é 2026-09-10.",
        "sourceField": "$.logistica.previsaoEntrega"
      }
    ]
  },
  "dataClasses": ["COMMERCIAL_CONFIDENTIAL"]
}
```

## Outbox, entrega e reconciliação

Atualizar ITSM e enviar ao WhatsApp são efeitos independentes:

```text
OUTBOUND_ACCEPTED
  -> ITSM_COMMENT_PENDING / RECORDED
  -> WHATSAPP_SEND_PENDING / ACCEPTED
  -> DELIVERY_CONFIRMED / FAILED
  -> RECONCILED
```

| Efeito | Exemplo de `operationId` |
| --- | --- |
| Criar ticket | `ticket-create:{eventId}` |
| Comentar ticket | `ticket-comment:{eventId}` |
| Enviar WhatsApp | `whatsapp-send:{outboundCommandId}` |
| Publicar handoff | `teams-handoff:{handoffId}` |
| Criar pedido | `order-create:{confirmedDraftId}` |

Cada destino recebe o mesmo `operationId` nas tentativas. Sem idempotência nativa, o worker consulta o efeito antes de repeti-lo. Webhooks do WhatsApp atualizam `ACCEPTED`, `DELIVERED`, `READ` e `FAILED`.

O ticket só entra em `WAITING_USER` após o marco de negócio configurado, por padrão `DELIVERED`. Aceite da API não equivale a entrega. Falha de ITSM não bloqueia o envio, pois a reconciliação é independente.

```mermaid
sequenceDiagram
    participant N as Nina/Renderer
    participant O as Outbox
    participant I as Worker ITSM
    participant W as Worker WhatsApp
    participant T as ITSM
    participant M as WhatsApp

    N->>O: OUTBOUND_ACCEPTED
    par Efeitos independentes
        O->>I: ticket-comment:{eventId}
        I->>T: comentário mínimo
        T-->>I: confirmado ou desconhecido
    and
        O->>W: whatsapp-send:{commandId}
        W->>M: mensagem
        M-->>W: ACCEPTED
    end
    M-->>O: webhook DELIVERED/READ/FAILED
    O->>O: Reconciliar e projetar estados
```

## Máquinas de estado separadas

Ticket, handoff e entrega não compartilham o mesmo estado.

```mermaid
stateDiagram-v2
    [*] --> PROVISIONING
    PROVISIONING --> OPEN: ticket_linked
    PROVISIONING --> PROVISIONING: retry_scheduled
    OPEN --> PROCESSING: work_started
    OPEN --> RESOLVED: resolved
    PROCESSING --> WAITING_USER: delivery_milestone_reached
    PROCESSING --> RESOLVED: resolved
    WAITING_USER --> PROCESSING: user_message_received
    WAITING_USER --> RESOLVED: resolved
```

```mermaid
stateDiagram-v2
    [*] --> QUEUED
    QUEUED --> ASSIGNED: assign
    QUEUED --> CLOSED: close
    ASSIGNED --> REPLIED: reply
    ASSIGNED --> CLOSED: close
    REPLIED --> ASSIGNED: follow_up_required
    REPLIED --> CLOSED: close
```

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> ACCEPTED: provider_accept
    PENDING --> FAILED: permanent_failure
    ACCEPTED --> DELIVERED: delivery_webhook
    ACCEPTED --> FAILED: failure_webhook
    DELIVERED --> READ: read_webhook
```

Cada transição possui evento, precondição, ator responsável, `entityVersion` e regra para evento tardio. O fallback imediato cabe em handoff `QUEUED`; não se tenta representar “escalado” como estado do ticket.

## Fallback humano no Teams

Topologia única:

1. Digibee cria o handoff e publica uma mensagem com Adaptive Card via Microsoft Graph.
2. Um Teams bot/app instalado recebe a atividade de Universal Action, incluindo os inputs do card.
3. O bot valida o token Bot Framework/Entra (`iss`, `aud`, tenant, assinatura e validade) e extrai o agente de `from.aadObjectId`.
4. O bot chama o endpoint interno com identidade de workload, escopo `handoff.callback` e uma asserção de ator assinada que vincula agente, tenant e atividade original.
5. O Digibee valida a asserção e resolve servidor-side conversa, ticket e destinatário.
6. Nonce de uso único, expiração, `handoffEventId` e `handoffVersion` bloqueiam replay e corrida.

O `attachments[].content` é uma string JSON serializada, e `body.content` referencia o attachment:

```json
{
  "body": {
    "contentType": "html",
    "content": "Nova solicitação humana <attachment id=\"handoff-card\"></attachment>"
  },
  "attachments": [
    {
      "id": "handoff-card",
      "contentType": "application/vnd.microsoft.card.adaptive",
      "contentUrl": null,
      "content": "{\"type\":\"AdaptiveCard\",\"version\":\"1.5\",\"$schema\":\"http://adaptivecards.io/schemas/adaptive-card.json\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Handoff HO-20260910-4412\"},{\"type\":\"Input.Text\",\"id\":\"agentReply\",\"isMultiline\":true,\"label\":\"Resposta ao usuário\"}],\"actions\":[{\"type\":\"Action.Execute\",\"title\":\"Assumir\",\"verb\":\"assign\",\"data\":{\"handoffId\":\"HO-20260910-4412\",\"handoffVersion\":1,\"nonce\":\"nonce-assign\"}},{\"type\":\"Action.Execute\",\"title\":\"Responder\",\"verb\":\"reply\",\"associatedInputs\":\"auto\",\"data\":{\"handoffId\":\"HO-20260910-4412\",\"handoffVersion\":1,\"nonce\":\"nonce-reply\"}},{\"type\":\"Action.Execute\",\"title\":\"Encerrar\",\"verb\":\"close\",\"data\":{\"handoffId\":\"HO-20260910-4412\",\"handoffVersion\":1,\"nonce\":\"nonce-close\"}}]}"
    }
  ]
}
```

O callback não aceita `ticketId`, `userId`, destinatário ou identidade do agente como autoridade:

```http
POST /v1/nina/human-fallback/events
Authorization: Bearer {teams-bot-workload-token}
Content-Type: application/json
Idempotency-Key: hfe_01J...
```

```json
{
  "schemaVersion": "1.0.0",
  "eventId": "hfe_01J...",
  "traceId": "trc_01J...",
  "causationId": "teams-activity-174...",
  "pipelineAction": "process_handoff_event",
  "handoffAction": "reply",
  "handoffId": "HO-20260910-4412",
  "handoffEventId": "hfe_01J...",
  "handoffVersion": 3,
  "nonce": "nonce-use-once",
  "occurredAt": "2026-09-10T01:44:00Z",
  "actorAssertion": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...",
  "reply": {
    "text": "Confirme o número do pedido para continuarmos."
  }
}
```

O bot monta `reply.text` a partir do input `agentReply` somente na ação `reply`. O Digibee verifica a asserção assinada contra a identidade de workload e a atividade original, deriva o agente de `aadObjectId`, exige ownership ou permissão de supervisor e atualiza com compare-and-set. `assign`, `reply` e `close` têm precondições distintas.

## Minimização, ITSM e LGPD

Antes de OpenAI, Teams, ITSM e WhatsApp, a política DLP calcula classes; não confia em booleano informado pelo chamador.

| Classe | Exemplos | Tratamento |
| --- | --- | --- |
| `PERSONAL_IDENTIFIER` | CPF, CNPJ, telefone, nome | Tokenizar ou mascarar conforme finalidade |
| `COMMERCIAL_CONFIDENTIAL` | pedidos, condições, anotações | Menor conjunto necessário e ACL |
| `FINANCIAL_PROFILE` | limite, score, títulos | Step-up, mascaramento forte e destino restrito |
| `SECURITY_EVIDENCE` | sinais de fraude/injeção | Trilha segregada; nunca WhatsApp/ITSM público |

O ITSM recebe resumo mínimo, IDs tokenizados e `visibility=restricted`, com ACL por fila e finalidade. A primeira mensagem fica apenas na descrição mínima do ticket; eventos posteriores viram comentários, sem duplicação.

A trilha imutável registra ator autenticado, decisão de autorização, ação, recurso, finalidade, instante e resultado. O programa deve manter inventário de processamento/destino, RIPD, base legal, operadores, transferência internacional, retenção, descarte, direitos do titular e revisão de decisões automatizadas. Exclusões são propagadas. Uso da OpenAI exige dados minimizados/tokenizados, treinamento desabilitado e retenção/região contratualmente aprovadas.

## Contratos e padrões

- OpenAPI 3.1 para HTTP;
- JSON Schema imutável por versão para eventos;
- `schemaVersion` obrigatório;
- mudanças aditivas dentro da versão compatível; ruptura exige nova versão;
- testes produtor/consumidor e validação de todos os exemplos;
- período de depreciação publicado;
- RFC 9457 (`application/problem+json`) para erros;
- RFC 3339 UTC para instantes;
- `YYYY-MM-DD` apenas para data civil, acompanhado de timezone;
- dinheiro em minor units e moeda;
- limites de payload, campos obrigatórios e `additionalProperties: false`.

Exemplo de erro:

```json
{
  "type": "https://example.internal/problems/stale-conversation",
  "title": "Versão de conversa obsoleta",
  "status": 409,
  "detail": "O evento não pode alterar uma versão mais recente.",
  "instance": "evt_01J...",
  "code": "STALE",
  "traceId": "trc_01J..."
}
```

## Roadmap: upload inteligente de pedidos

O fluxo planejado recebe PDF ou imagem, valida assinatura, tamanho, formato e malware, armazena temporariamente de forma criptografada, executa OCR e cria um rascunho versionado. A confirmação explícita com step-up MFA gera `confirmedDraftId`; somente então a outbox emite `order-create:{confirmedDraftId}`.

Arquivo corrompido, imagem ilegível e campos ambíguos nunca criam pedido. Hash do arquivo auxilia detecção, mas a idempotência efetiva é o `operationId` do rascunho confirmado. Após timeout, consultar Portal/TOTVS antes de repetir.

## Observabilidade mínima

- backlog e idade máxima de inbox/outbox;
- duplicatas rejeitadas e leases recuperados;
- respostas `STALE`;
- latência por intenção e dependência;
- divergência entre ITSM e WhatsApp;
- mensagens `ACCEPTED`, `DELIVERED`, `READ` e `FAILED`;
- handoffs estagnados e callbacks rejeitados;
- freshness por fonte;
- decisões ABAC negadas;
- rejeições de NLU, injeção, ID inventado e clarificações;
- divergência canário Copilot versus OpenAI;
- falhas do validador factual e uso de fallback determinístico;
- pendências de exclusão e retenção.

## Critérios mínimos para produção

- OIDC + PKCE + MFA e sessão vinculada ao número;
- identidade, ticket e destinatário resolvidos no servidor;
- ABAC de carteira antes do fan-out;
- inbox/outbox duráveis com testes de concorrência e recuperação;
- callback Teams autenticado, autorizado, idempotente e não repetível;
- política de indisponibilidade do ITSM comprovada por teste;
- OpenAPI/JSON Schemas versionados e contract tests;
- NLU de catálogo fechado, resolução na carteira e testes de acesso cruzado;
- Copilot Studio sem generative orchestration contra ERP/crédito;
- renderer determinístico ou validação factual estrita;
- DLP antes de cada fronteira;
- RIPD, retenção, ACL, trilha imutável e direitos LGPD definidos;
- reconciliação e métricas operacionais ativas.

## Referências

- [Digibee API Trigger](https://docs.digibee.com/documentation/connectors-and-triggers/triggers/web-protocols/api)
- [Digibee REST V2](https://docs.digibee.com/documentation/connectors-and-triggers/connectors/web-protocols/rest-v2)
- [WhatsApp Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api/)
- [Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/)
- [Microsoft Graph Teams messages](https://learn.microsoft.com/en-us/graph/api/channel-post-messages)
- [Teams Universal Actions](https://learn.microsoft.com/en-us/adaptive-cards/authoring-cards/universal-action-model)
- [OpenAI API](https://developers.openai.com/api/reference/)
