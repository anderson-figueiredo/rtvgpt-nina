# Validacao de RTV com 3 primeiros digitos do CPF

Este documento define a regra de validacao de identidade do RTV antes de qualquer consulta de negocio na Nina.

## Objetivo

Garantir que o usuario autenticado na sessao corresponde ao `rtvId` esperado, reduzindo risco de acesso cruzado entre carteiras comerciais.

## Regra de validacao

1. A camada de autenticacao resolve:
   - `rtvId` da sessao ativa.
   - CPF informado/derivado do token de identidade.
2. O Digibee normaliza o CPF (somente numeros).
3. O sistema compara o **prefixo de 3 digitos** do CPF com o prefixo cadastrado no IAM para aquele `rtvId`.
4. Se houver match, o fluxo continua normalmente.
5. Se nao houver match, a requisicao eh bloqueada com status `UNAUTHORIZED_RTV`.

## Ponto de aplicacao no fluxo

- A validacao acontece antes das acoes:
  - `query_order_credit_delivery`
  - `query_visit_preparation`
  - `open_ticket` (quando envolver dados do cliente)
- Sem validacao aprovada, nao pode haver consulta em TOTVS, Portal, Tarken, LoogAI, Lecom ou ITSM.

## Contrato de erro sugerido

```json
{
  "correlationId": "corr-20260909-3001",
  "status": "UNAUTHORIZED_RTV",
  "message": "Nao foi possivel validar a identidade do RTV para esta sessao.",
  "retryable": false,
  "action": "escalate_to_human",
  "reason": "SECURITY_RISK"
}
```

## Comportamento esperado na Nina

- A Nina nao expõe detalhes de seguranca no WhatsApp.
- Mensagem ao usuario:
  - "Nao consigo concluir essa solicitacao agora. Encaminhei para um especialista, que retorna neste chat."
- A Nina chama `POST /v1/nina/human-fallback` com `reason=SECURITY_RISK`.

## Consideracoes de seguranca e LGPD

- O CPF completo nao deve ser logado em texto puro.
- Persistir apenas:
  - hash do CPF normalizado (quando necessario para auditoria),
  - prefixo validado,
  - resultado da validacao (aprovado/reprovado).
- O prefixo de 3 digitos deve ser tratado como sinal de controle adicional e nao como fator unico de autenticacao.

## Observabilidade minima

- Contador de validacoes por resultado (`success`, `failure`).
- Alerta para taxa de falha de validacao acima do baseline.
- Correlation ID em todos os logs de seguranca.

