**RTVgpt — Governança e Proteção de Dados**

*Texto pronto para os itens 4 (Governança, Indicadores e Valor) e parte do item 3 (riscos de integração, segurança e LGPD) do Tech Challenge*

O texto abaixo já está redigido para ser incorporado ao PDF final do Tech Challenge, cobrindo especificamente o item "4. Governança, Indicadores e Valor para o Negócio" e o bloco de riscos de segurança/LGPD exigido dentro do item 3 ("Arquitetura de Integração, Dados e IA"). Ele parte da arquitetura já pivotada do grupo — RTV no WhatsApp, atendido pela Nina, com escalonamento para atendimento humano via Microsoft Teams nos casos que exigem julgamento (crédito, exceções de logística) — e incorpora o que foi validado na mentoria com o Alisson (Nina, Digibee, Tarken, Jornada do RTV, escopo agro Brasil).

# **4\. Governança, Indicadores e Valor para o Negócio**

## **4.1 Dados críticos por domínio da jornada do RTV**

A camada WhatsApp → Nina não cria dados novos: ela consolida, para o RTV, dados que já existem em sistemas especialistas. A tabela abaixo mapeia, por domínio, qual dado é crítico para o fluxo e de onde ele deve vir — distinção importante porque nem toda fonte é confiável para responder em tempo real (ver seção 4.5).

| Domínio | Dado crítico | Sistema de origem | Uso no fluxo WhatsApp → Nina |
| :---- | :---- | :---- | :---- |
| Cadastro do cliente | Razão social, CNPJ, contatos, endereço | Lecom, replicado no Totvs/Datasul | Identificar o cliente citado pelo RTV na conversa |
| Pedido | Status (aberto/faturado/produção/aguardando crédito/cancelado), motivo de pendência (frete, natureza de operação), previsão de entrega | Portal de Pedidos / Totvs-Datasul, via Digibee | Responder "em que etapa está meu pedido" |
| Crédito | Pré-crédito aprovado, saldo utilizado, faturas em atraso, status de bloqueio | Tarken — status via Digibee (tempo real); liberação plena via chamado | Responder "posso vender mais" e decidir se escala para Teams |
| Entrega / Logística | Status de envio, previsão de entrega, risco de atraso, composição de carga | LoogAI | Responder dúvidas de entrega |
| Atendimento / Suporte | Histórico de chamados, status, SLA da área responsável, motivo | Nina / ITSM | Abrir ou consultar chamado e decidir o transbordo |

## **4.2 Donos dos dados — quem responde pela qualidade e atualização**

| Domínio de dado | Área responsável | Observação |
| :---- | :---- | :---- |
| Cadastro do cliente | Time de Master Data / Cadastro (aprovação no Lecom) | Endereço e razão social já vêm automatizados por integração com a Receita Federal, reduzindo erro de digitação |
| Pedido | Time Comercial / donos do Portal de Pedidos | O Totvs/Datasul é a fonte de verdade; se um relatório de BI divergir, a correção é no relatório, não no banco |
| Crédito | Time de Crédito / Copernitro (donos do Tarken) | Pré-crédito já é automatizado; a liberação plena continua sob responsabilidade humana |
| Entrega / Logística | Time de Logística / donos do LoogAI | — |
| Atendimento / Suporte (Nina) | Time de TI Digital / donos da Nina e do ITSM | Passa a responder também pela integração WhatsApp ↔ Nina |
| Integração (Digibee) e novo canal WhatsApp | Time de Arquitetura / Integração | Recomenda-se que este time assuma também a governança do catálogo de ferramentas que a Nina passa a acionar |

Recomenda-se instituir um fórum leve de governança — um "Comitê do RTV Experience Hub", com representantes de crédito, logística, comercial e TI, revisando trimestralmente os indicadores da seção 4.3 e os incidentes de dado divergente da seção 4.5. Isso evita que a nova camada vire mais um sistema sem dono claro, repetindo o problema que a própria dor do RTV descreve hoje.

## **4.3 Indicadores que medem produtividade, experiência e valor comercial**

| Indicador | O que mede | Baseline (levantado na mentoria) | Meta do MVP |
| :---- | :---- | :---- | :---- |
| Nº de sistemas acessados por atendimento | Fricção operacional | 4 sistemas hoje (Lecom, Portal, Tarken, LoogAI) navegados separadamente | 1 (WhatsApp/Nina) |
| Taxa de resolução via self-service (sem transbordo) | Eficácia do agente | Não medido hoje; o bot atual (Blip) depende de menu e gera muitos transbordos | ≥ 60% das interações resolvidas sem transbordo para Teams |
| Taxa de roteamento correto na primeira tentativa | Ataca a dor nº 1 citada pela Nitro: o RTV não sabe a quem recorrer | Baixa hoje — reencaminhamentos sucessivos são a principal reclamação | ≥ 90% direcionados corretamente já na primeira tentativa |
| Tempo até resposta útil | Experiência do RTV | SLA por área: 2 a 4 horas quando o roteamento é correto | \< 2 min para consultas de self-service; manter o SLA da área para os casos escalonados |
| Volume de solicitações por frete / natureza de operação / crédito | Dimensiona o problema operacional | 20 a 30 por dia entre 130 RTVs | Redução de 40–60% dos casos que hoje geram chamado, via autoatendimento |
| Adesão dos RTVs ao novo canal | Adoção | 0% (RTVs não usam a Nina hoje, por estar no Teams) | ≥ 70% dos RTVs do piloto usando o canal WhatsApp ativamente |
| Precisão da resposta do agente (auditoria por amostragem) | Confiabilidade | N/A | ≥ 95% |
| Taxa de mensagens dentro da janela de 24h do WhatsApp | Saúde técnica do canal | N/A — gap conhecido da Nina atual (sem aviso proativo de chamado resolvido) | Monitorar continuamente; usar templates aprovados quando fora da janela |

## **4.4 Como acompanhar redução de tempo, retrabalho e nº de sistemas acessados**

* Cada interação deve gerar um registro estruturado (RTV, cliente, sistemas/ferramentas acionadas, houve ou não transbordo, tempo até resposta) — a mesma trilha de auditoria da seção de segurança serve de fonte para os indicadores, evitando duas instrumentações separadas.

* Usar o Reportload — o BI que os RTVs já conhecem — como vitrine dos indicadores de governança, em vez de introduzir mais uma ferramenta que eles resistem a adotar (lição já observada com o próprio Reportload e antes com relatórios automáticos de comissão).

* Comparar mensalmente contra a baseline hoje registrada na Blip (20-30 solicitações/dia, tempo de resposta por área) para demonstrar de forma objetiva a redução de retrabalho e fricção.

## **4.5 Riscos se os dados forem divergentes, incompletos ou desatualizados**

* **Fonte de verdade mal definida:** já ocorreu na Nitro divergência entre um relatório de BI e o sistema de origem por um status mal mapeado no relatório. Nesses casos a correção é no relatório, nunca no banco — a governança deve deixar isso explícito: Totvs/Datasul (e o Tarken, quando aplicável) são a fonte de verdade, nunca uma réplica de BI.

* **Dado desatualizado por delay de plataforma:** o Databricks atualiza de horas a um dia; nunca deve alimentar respostas de tempo real do agente (ex.: "esse pedido foi aprovado agora?"). Somente o Digibee, que consulta ao vivo, deve alimentar respostas críticas de status.

* **Identificação divergente do cliente entre sistemas:** Lecom e Totvs usam o mesmo código, mas o Tarken às vezes é referenciado por CNPJ. A integração deve sempre confirmar a identidade do cliente antes de expor qualquer dado sensível, para não responder sobre o cliente errado.

* **Risco mais crítico — bloqueio de crédito desatualizado:** se o dado de inadimplência não estiver em tempo real, o RTV pode vender para um cliente já bloqueado. Por isso toda resposta sobre crédito deve ser tratada como recomendação, citando fonte e horário da consulta, e o MVP precisa garantir que esse dado específico vem sempre do Digibee em tempo real, nunca de cache ou de uma réplica.

# **3.1. Riscos de Segurança, LGPD e Qualidade de Dados na Integração WhatsApp → Nina → Sistemas**

Este bloco atende diretamente ao item 3 do desafio ("quais riscos técnicos existem em integração, segurança, LGPD e qualidade dos dados"), com foco no que muda ao trocar o canal de Teams para WhatsApp.

## **3.1.1 Por que o WhatsApp exige atenção redobrada em relação à Nina no Teams**

A Nina, restrita ao Teams, operava inteiramente dentro do perímetro corporativo Microsoft da Nitro. Ao abrir a Nina para o WhatsApp, entram novos elementos de risco: o número de telefone como identificador pessoal, a infraestrutura de um provedor de canal (Blip hoje, ou o canal nativo do Copilot Studio) como intermediário no tratamento das mensagens, e a janela de 24 horas do WhatsApp Business (Meta), que já se mostrou um problema real — é o motivo pelo qual a Nina hoje não consegue avisar proativamente o RTV quando um chamado é resolvido.

## **3.1.2 Novos dados e atores envolvidos com a entrada do WhatsApp**

| Elemento | O que é | Cuidado de segurança / LGPD |
| :---- | :---- | :---- |
| Número de telefone do RTV | Dado pessoal, usado como identificador de sessão | Deve estar vinculado ao cadastro corporativo do RTV mantido pela "Jornada do RTV", para que a troca de número não exponha dados a uma pessoa errada |
| Conteúdo das mensagens (pergunta e resposta) | Pode conter dados de crédito, pedidos e do cliente final | Trafega por infraestrutura de terceiro (Meta / provedor do canal) — mapear onde fica armazenado e por quanto tempo |
| Templates de notificação (mensagens ativas) | Usados quando a janela de 24h expira | Não devem trazer dado sensível no texto (valor de dívida, nome de cliente); usar mensagem genérica que direcione a uma consulta segura dentro do fluxo |
| Provedor do canal WhatsApp (Blip hoje, ou canal nativo do Copilot Studio) | Operador de dados pessoais na cadeia | Precisa de contrato de tratamento de dados (DPA), no mesmo padrão já exigido para o Digibee e para o fornecedor do modelo de IA |

## **3.1.3 Riscos técnicos herdados da arquitetura atual (confirmados na mentoria)**

* **Consulta direta ao banco Oracle via Digibee, sem API formal:** funciona hoje sem histórico de falha, mas qualquer mudança de schema no Totvs/Datasul quebra a integração sem aviso. Recomenda-se monitoramento ativo e testes de regressão sempre que o ERP for atualizado.

* **Janela de 24h do WhatsApp Business:** já é um gap conhecido da Nina atual. A nova solução precisa tratar isso explicitamente com mensagens de template pré-aprovadas pela Meta, e não deve prometer notificação proativa sem esse desenho.

* **Acesso indireto ao Tarken:** o licenciamento por usuário inviabiliza acesso direto dos 130 RTVs. Isso reforça que a liberação plena de crédito deve continuar como fluxo assistido por humano (chamado), nunca uma promessa de decisão automática.

## **3.1.4 Controles propostos**

* **Autenticação e identidade:** vincular o número de WhatsApp do RTV ao cadastro corporativo já mantido pela "Jornada do RTV" — o mesmo processo que hoje cria e revoga acesso a e-mail e ao Portal de Pedidos — garantindo que só RTVs ativos interajam com a Nina por esse canal, com atualização automática no desligamento.

* **Privilégio mínimo:** manter, para a nova camada, o mesmo padrão já usado hoje no Digibee — usuário técnico somente leitura, restrito às tabelas necessárias — e aplicar o mesmo princípio a qualquer nova ferramenta exposta à Nina (ex.: consulta de crédito).

* **Minimização de dado exposto:** a resposta da Nina no WhatsApp deve trazer o resumo necessário (ex.: "bloqueado por inadimplência"), nunca o extrato financeiro completo do cliente.

* **Rótulo de recomendação:** toda resposta envolvendo crédito deve indicar que é uma consulta de status, não uma decisão automática de liberação — mantendo o time de crédito no controle final, como já ocorre hoje.

* **Trilha de auditoria:** registrar cada interação (RTV, cliente consultado, dado retornado, se houve transbordo para Teams) — o mesmo registro alimenta tanto a segurança quanto os indicadores de governança da seção 4\.

* **Contratos de tratamento de dados (DPA):** formalizar com o provedor do canal WhatsApp (Blip ou o canal nativo do Copilot Studio, se adotado) e com o provedor do modelo de IA, definindo finalidade, retenção e proibição de uso dos dados para treinar modelos de terceiros.

* **Avaliação de impacto (RIPD/DPIA):** recomenda-se uma avaliação específica antes de abrir o canal WhatsApp para dados de crédito, já que esse canal envolve atores (Meta / provedor do canal) que a Nina, restrita ao Teams, não tinha.

| Por que isso resolve o problema real observado pela Nitro Esse desenho ataca diretamente os dois gaps que a própria equipe da Nitro relatou na mentoria: (1) a Nina não conseguia notificar o RTV proativamente pelo WhatsApp por causa da janela de 24h — resolvido com mensagens de template; e (2) o RTV não tinha um canal natural para tratar crédito — resolvido reaproveitando a integração Nina↔Tarken já existente, apenas estendendo o canal de entrada para o WhatsApp em vez do Teams. |
| :---- |

