# Detalhes Tecnicos de Integracoes (Digibee + Nina + Sistemas Corporativos)

Este documento complementa o `README.md` com um recorte mais operacional para implementacao, troubleshooting e evolucao dos pipelines.

## 1. Visao de componentes

| Camada | Responsabilidade | Tecnologia/Servico |
| --- | --- | --- |
| Canal | Entrada e saida de mensagens | WhatsApp Cloud API |
| Assistente | Interpretacao, planejamento e composicao de resposta | Nina (Copilot) + OpenAI |
| Integracao | Orquestracao, governanca e contratos HTTP | Digibee |
| Sistemas core | Cadastro, pedidos, ERP, credito, logistica, suporte | Lecom, Portal, TOTVS, Tarken, LoogAI, ITSM |
| Fallback humano | Atendimento assistido e seguranca | Microsoft Teams |

## 2. Pipelines Digibee e contratos

| Pipeline | Trigger | Objetivo | Saida principal |
| --- | --- | --- | --- |
| `nina-whatsapp-inbound` | `POST /v1/nina/messages/inbound` | Receber webhook, validar assinatura, correlacionar ticket | Evento normalizado para Nina com `ticketId` |
| `nina-whatsapp-orchestrator` | `POST /v1/nina/orchestrator` | Consultar sistemas corporativos por intencao | Payload consolidado (`SUCCESS`/`PARTIAL_SUCCESS`) |
| `nina-human-fallback` | `POST /v1/nina/human-fallback` | Escalonar para Teams em `INFORMATION_NOT_FOUND` ou `SECURITY_RISK` | `handoffId`, status de fila e mensagem segura |
| `nina-itsm-ticket-update` | `POST /v1/nina/messages/outbound` e `POST /v1/nina/human-fallback/callback` | Atualizar ticket e enviar resposta no WhatsApp | Comentario no ITSM + entrega no canal |

## 3. Modelo de correlacao e idempotencia

### Chaves no Object Store

| Chave | Uso |
| --- | --- |
| `itsm:msg:{messageId}` | Evitar processamento duplicado no inbound |
| `itsm:conv:{channel}:{userId}` | Correlacionar conversa com ticket aberto |
| `handoff:{handoffId}` | Rastrear estado do fallback humano |

### Regras

1. O mesmo `messageId` nao pode abrir/atualizar ticket duas vezes.
2. Cada conversa ativa (`channel + userId`) aponta para um unico ticket aberto.
3. Reprocessos usam `correlationId` para deduplicar saida.

## 4. Contratos de payload (resumo pratico)

### 4.1 Inbound normalizado para Nina

```json
{
  "correlationId": "corr-...",
  "channel": "whatsapp",
  "ticketId": "INC-...",
  "input": {
    "userId": "5511...",
    "messageId": "wamid....",
    "text": "mensagem do usuario"
  }
}
```

### 4.2 Requisicao de orquestracao

```json
{
  "pipeline": "nina-whatsapp-orchestrator",
  "action": "query_order_credit_delivery",
  "ticketId": "INC-...",
  "input": {
    "orderNumber": "12345"
  }
}
```

### 4.3 Escalonamento humano dedicado

```json
{
  "pipeline": "nina-human-fallback",
  "action": "escalate_to_human",
  "reason": "INFORMATION_NOT_FOUND",
  "conversationContext": {
    "ticketId": "INC-..."
  }
}
```

## 5. Mapeamento de acoes por intencao

| Intencao Nina | Acao Digibee | Sistemas consultados |
| --- | --- | --- |
| `delivery_eta_and_credit_limit` | `query_order_credit_delivery` | TOTVS, Tarken, LoogAI |
| `visit_preparation` | `query_visit_preparation` | Lecom, TOTVS, Portal, Tarken, LoogAI, ITSM |
| `open_ticket` | `query_or_update_existing_ticket` | ITSM |
| `escalate_to_human` | `escalate_to_human` | Teams + ITSM |

## 6. Maquina de estados do ticket

```
ABERTO -> EM_ANDAMENTO -> AGUARDANDO_USUARIO -> EM_ANDAMENTO
EM_ANDAMENTO -> ESCALADO -> AGUARDANDO_USUARIO
qualquer estado aberto -> RESOLVIDO
```

## 7. Seguranca e conformidade

- Validar assinatura do webhook WhatsApp (`X-Hub-Signature-256`).
- Aplicar mascaramento de CPF/CNPJ e dados de credito em logs.
- Aplicar escopo por acao para tokens de servico (principio do menor privilegio).
- Bloquear acesso fora da carteira do RTV (`SECURITY_RISK`).

## 8. Resiliencia e operacao

### Politicas recomendadas

- Timeout por dependencia externa (nao usar timeout global unico).
- Retry com backoff exponencial apenas em 5xx/timeout.
- Circuit breaker para evitar cascata de falhas.
- Fila de reprocesso para efeitos colaterais nao criticos (ex.: comentario no ITSM).

### Resultado degradado

- Se uma dependencia falhar e houver dados suficientes, retornar `PARTIAL_SUCCESS`.
- Acionar fallback humano apenas quando nao houver dados utilizaveis ou em risco de seguranca.

## 9. Observabilidade minima

| Sinal | Descricao |
| --- | --- |
| `correlationId` | Identificador unico de ponta a ponta |
| `integrationStatus.overall` | `SUCCESS`, `PARTIAL_SUCCESS`, `NOT_FOUND`, `NEEDS_DISAMBIGUATION` |
| `handoff.status` | `QUEUED`, `ASSIGNED`, `REPLIED`, `CLOSED` |
| `ticketStatus` | Estado atual do ticket no ITSM |

Dashboards minimos:
- Taxa de erro por pipeline.
- Latencia p95 por acao.
- Taxa de fallback humano.
- Falhas de callback Teams.

## 10. Checklist de deploy seguro

1. Validar contratos JSON (schema) entre Nina e Digibee.
2. Rodar testes de contrato dos pipelines com payloads reais mascarados.
3. Verificar politicas de segredo e rotacao de credenciais.
4. Confirmar alertas ativos para ITSM, Teams e dependencias core.
5. Publicar versao com changelog de contratos e rollback definido.

