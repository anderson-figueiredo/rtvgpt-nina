# Riscos de Integracao entre Digibee, Nina e Sistemas Corporativos

Este documento consolida os principais riscos tecnicos e operacionais da arquitetura descrita no `README.md`, com foco em reducao de impacto, continuidade do atendimento e seguranca de dados.

## Escopo analisado

- WhatsApp (entrada e saida de mensagens)
- Nina/Copilot + provedor LLM
- Digibee (hub de orquestracao)
- Lecom, Portal de Pedidos, TOTVS/Datasul, Tarken, LoogAI
- ITSM e fallback humano no Microsoft Teams

## Matriz de risco

| ID | Risco | Probabilidade | Impacto | Sinal de deteccao | Mitigacao recomendada |
| --- | --- | --- | --- | --- | --- |
| R1 | Indisponibilidade de um sistema satelite (TOTVS, Portal, Tarken, LoogAI) | Media | Alto | Timeouts e aumento de erros 5xx por conector | Timeout por dependencia, retries com backoff, respostas com `PARTIAL_SUCCESS`, fila de reprocesso para conciliacao posterior |
| R2 | Abertura duplicada de tickets no ITSM por reenvio de webhook | Media | Medio | Dois tickets para o mesmo `wamid` | Idempotencia com chave `itsm:msg:{messageId}` e deduplicacao por `conversationKey` |
| R3 | Exposicao de dados sensiveis em logs (CPF/CNPJ, score, credito) | Baixa | Alto | Logs com payload completo sem mascaramento | Mascaramento em tempo de pipeline, segregacao de logs e revisao automatica de padroes sensiveis |
| R4 | Resposta do LLM com alucinacao ou dado sem lastro no payload | Media | Alto | Divergencia entre resposta e bloco consolidado do Digibee | Restringir composicao ao payload canonico, guardrails de resposta e testes de regressao de prompts |
| R5 | Acesso cruzado fora da carteira do RTV | Baixa | Critico | Solicitacao de cliente nao associado ao RTV autenticado | Regra `rtv_may_only_access_own_portfolio`, bloqueio preventivo e fallback `SECURITY_RISK` |
| R6 | Falha no callback Teams -> Digibee (atendimento humano nao entregue ao WhatsApp) | Media | Alto | Handoff sem `REPLIED` dentro do SLA esperado | Retry do callback, monitor de handoff estagnado e canal de contingencia manual no ITSM |
| R7 | Duplicidade na criacao de pedidos por upload repetido do mesmo arquivo | Media | Medio | Pedidos com itens/valores iguais em janela curta | Hash do arquivo + janela temporal + confirmacao explicita do usuario antes da criacao |
| R8 | Quebra de contrato entre pipelines (mudanca de schema sem versionamento) | Media | Alto | Erros de validacao em runtime apos deploy | Versionamento de schema (`*_v1`, `*_v2`), contract tests e deploy canario |

## Controles minimos obrigatorios

1. **Idempotencia ponta a ponta**
   - Entrada: `messageId` (WhatsApp).
   - Saida: `correlationId + ticketId`.
   - Upload: hash do arquivo + remetente + janela temporal.

2. **Resiliencia por dependencia**
   - Timeout individual por sistema.
   - Retry com backoff apenas para falhas transientes.
   - Circuit breaker para proteger o orquestrador.

3. **Governanca de acesso**
   - Escopo por acao (`query_*`, `escalate_to_human`, `reply_and_update_ticket`).
   - Restricao de carteira do RTV em todas as consultas por cliente.

4. **Observabilidade e auditoria**
   - Correlation ID unico em todos os hops.
   - Dashboards de erro por pipeline e por dependencia externa.
   - Trilha de auditoria no ticket ITSM para cada resposta enviada.

## SLIs e alertas recomendados

| SLI | Objetivo inicial | Alerta |
| --- | --- | --- |
| Taxa de sucesso do `nina-whatsapp-inbound` | >= 99.5% | < 98.5% em 15 minutos |
| Latencia p95 da resposta automatica | <= 8s | > 12s em 10 minutos |
| Taxa de fallback humano | <= 8% | > 15% em 30 minutos |
| Taxa de `PARTIAL_SUCCESS` | <= 12% | > 20% em 30 minutos |
| Falhas de callback Teams | <= 1% | > 3% em 15 minutos |

## Priorizacao de mitigacao (ordem sugerida)

1. Bloqueio de acesso cruzado (R5) e mascaramento (R3), por risco de seguranca/LGPD.
2. Resiliencia de dependencias (R1, R6, R8), por continuidade operacional.
3. Duplicidade operacional (R2, R7), por impacto em atendimento e retrabalho.
4. Qualidade de resposta da IA (R4), com testes e guardrails recorrentes.

