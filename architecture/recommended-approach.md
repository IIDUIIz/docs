# Abordagem recomendada: Contatos como área própria

Data: 25/10/2026. Responsável: @architect (Aria).
Estado: proposta; decisões de produto pendentes estão identificadas abaixo.
Base factual: [Análise do projeto](project-analysis.md), achados F1–F11.

Atualização de 26/10/2026: o owner confirmou uma ficha com vários meios de contato.
O [modelo de domínio](contact-domain-model.md) passa a orientar a persistência, os vínculos e os IDs.
A recomendação anterior de ampliar o Contact legado diretamente ficou superada para o cadastro.

## Recomendação

Criar **Contatos** como módulo funcional do Hadem, com entrada própria e rotas
`/dashboard/contacts` e `/dashboard/contacts/[contactProfileId]`.
O histórico atual vira uma seção da ficha. A URL antiga precisa resolver o Contact ID operacional
para o novo ID da ficha antes do redirecionamento, preservando a seção de conversas e links.

Separação sugerida de responsabilidades:

| Área | Responsabilidade |
|---|---|
| Contatos | Cadastro e consulta do relacionamento do contato |
| CRM | Negociações, funis, etapas, resultados/motivos, campos e eventos de negócio |
| Conversas | Atendimento, mensagens, envio, distribuição e canais |
| Tarefas | Pendências, execução e resultado de tarefas |
| Relatórios | Indicadores agregados da operação |

A área Contatos compõe informações dos outros domínios. Não passa a possuir cópias de cards,
mensagens ou tarefas. Módulo significa organização funcional e RBAC, sem serviço separado,
novo deploy ou cobrança por módulo (F2, F6–F9, F11 e restrições da análise).

## Ficha proposta e prioridades

| Seção | Conteúdo | Situação |
|---|---|---|
| Cadastro | Nome/foto cadastrais, endereço opcional e vários meios associados | Primeira entrega proposta; pessoa multimeios confirmada em 26/10 |
| Negócios | Lista completa vinculada, abertos e finalizados, com funil, valor, resultado/motivo e retorno quando existente | Primeira entrega proposta; API base já existe |
| Conversas | Histórico atual com paginação, filtro por canal e atualização | Migrar preservando comportamento autorizado |
| Tarefas | Tarefas de cards/conversas do contato | Próxima entrega; falta consulta adequada |
| Resumo | Contagens/valores e interação recente | Evolução após fechar as regras e a cobertura dos dados |
| Campos personalizados do contato | Definições específicas do tenant | Proposta posterior; contrato não decidido |

Não incluir por inferência documento fiscal, aniversário, empresa, enriquecimento externo,
importação, exportação, bloqueio, exclusão, merge automático por coincidência ou cadastro de organizações.
São escopos adicionais. Associação explícita de meios existentes da mesma pessoa integra o modelo
multimeios e precisa do contrato de conflitos/correção descrito no modelo de domínio.
Observações e outros dados cadastrais podem ser refinados depois do endereço.

## Decisões de produto que precisam ser fechadas

### P1 — Quem consulta/edita a área

Recomendação técnica: entrada/listagem/cadastro com `contacts:read`; edição com `contacts:manage`.
Reusar a capability existente. A decisão sobre conceder esses acessos aos perfis padrão
Gerente/Atendente continua com o owner. Lookup e operação do CRM não são equivalentes à gestão
global de contatos (F2). A concessão também libera a visão completa prevista na ADR-006 para essa área.

### P2 — Quem consulta o histórico consolidado de mensagens

Recomendação para preservar a intenção da área anterior: seção consolidada condicionada a
`conversation:monitor`, alinhando o novo caminho REST e a subscription desse contexto.
Continuar permitindo acesso ao cadastro sem dar automaticamente acesso ao histórico completo.
Essa proposta precisa ser confirmada como regra de produto.

O endpoint legado é usado no atendimento. Projetar um caminho explícito para a consulta consolidada
ou adequar o serviço por contexto validado no servidor, preservando o caso de uso do atendimento.
Não alterar o guard compartilhado sem prova dos dois consumidores; nenhum parâmetro arbitrário
do cliente deve elevar uma consulta comum ao contexto de monitoramento (F3 e F7).

### P3 — Pessoa com vários meios de contato: confirmado em 26/10

O owner escolheu uma ficha reunindo telefone, e-mail e Instagram da pessoa.
A recomendação técnica é cadastro separado acima dos Contact IDs atuais, conforme o modelo B
no documento de domínio. Isso preserva os vínculos operacionais e coloca endereço no cadastro.
Associação de meios existentes exige operação explícita; a decisão não autoriza merge automático
por nome, mudança de titularidade nem alteração de `Contact.value`, hoje imutável.

### P4 — Endereço

Proposta inicial: um endereço atual opcional, estruturado para logradouro, número, complemento,
bairro, cidade, estado, código postal e país; detalhes de validação pertencem à story.
Quantidade de endereços, etiquetas de entrega/cobrança e consulta automática de CEP não estão decididas.
O endereço atual do cadastro não deve reescrever o endereço histórico de uma eventual venda/entrega.

## Sequência de implementação proposta

### Etapa 0 — Fechar escopo e contratos

- P3 confirmado; refinar P1/P2/P4 e os contratos de vínculo/correção do modelo de domínio.
- Criar epics/stories irmãos com @pm/@sm/@po conforme o workflow do projeto.
- Definir ACs do cadastro, navegação, compatibilidade, histórico e permissões por seção.
- Inventariar índices e schema com @data-engineer se houver mudança de modelagem.

As etapas abaixo são pacotes de trabalho para refinamento, não stories aprovadas nem números reservados.

### Etapa 1 — Fundar a área e ampliar o cadastro

- Criar listagem paginada com busca por nome/identificador e ordenação estável no servidor (F10).
- Expor detalhe por ID com validação de tenant e contrato próprio; não depender de endpoints legados Customer.
- Persistir endereço opcional no novo cadastro, atravessando DTO, aggregate/tipos, mapper,
  repository, serviço e cliente. Contact legado recebe o vínculo ao cadastro, sem mudar o valor canônico.
- Preservar valor canônico, identidades e criação pelo atendimento/CRM/automação.
- Campos ausentes no legado continuam vazios; não inventar endereço por backfill.
- Edição publica invalidação apropriada, preservando atualização de nome no inbox e evitando
  carregar cadastro sensível em eventos de sala compartilhada (F7).

### Etapa 2 — Consolidar negócios e migrar histórico

- Compor leitura pelos Contact IDs associados à ficha, preservando resultados e paginação (F6).
- Respeitar `crm:read` para cards e `crm:monitor` para eventos, mantendo regras de equipe.
- Resumos, quando implementados, usam exatamente o mesmo escopo dos registros exibidos.
- Separar o componente de histórico da antiga página e montar a seção autorizada em Contatos.
- Migrar a entrada do registry, breadcrumbs, links e traduções; manter acesso pelo menu Mais no celular.
- Instalar redirects base/detalhe de `/dashboard/reports/contacts` para a área nova, sem rota duplicada
  com regras diferentes. O detalhe resolve Contact ID legado para o ID da ficha; não trocar só o prefixo.
- Rever acesso antigo: usuários com histórico autorizado e sem acesso ao cadastro novo não podem
  perder a funcionalidade por efeito acidental do redirect. A regra para essa combinação depende de P1/P2.

É possível entregar as etapas 1 e 2 juntas como primeira versão, depois das decisões acima.

### Etapa 3 — Integrar tarefas

- Definir consulta paginada que encontre tarefas diretamente vinculadas e tarefas dos cards/conversas
  relacionados, com tenant e escopo por responsável/gestão (F8).
- Expor essa consulta pelo próprio domínio Tasks ou por uma porta de leitura autorizada.
- Consultar em lote, deduplicar e evitar carregar todas as tarefas do tenant no front.
- Tarefas continuam sendo criadas/concluídas pelo módulo de origem; não recriar Atividades do CRM.

### Etapa 4 — Extrair a propriedade técnica de Contacts, se necessária

Separar agora a feature do front e suas responsabilidades de API. Manter o cadastro canônico e
o roteamento existentes funcionando durante a primeira entrega.

Uma extração para `back-end/src/modules/contacts` pode ser uma story estrutural própria:

- Contatos passa a possuir aggregate, schema, repository e seus casos de uso.
- Conversation consome contratos de identidade; Pipelines/Tasks consomem consultas e políticas apropriadas.
- Handlers CQRS e modelos são registrados uma vez, com imports explícitos e sem ciclo.
- Presenter/policy compartilhados não arrastam desnecessariamente todo o ConversationModule.
- Contato, IDs, coleção, índices e rotas existentes permanecem compatíveis durante a transição.

Não misturar automaticamente essa extração com a modelagem de pessoa multicanal. A primeira muda
propriedade do código; a segunda muda o significado e os vínculos dos dados (F4/F11).

## Estrutura sugerida da feature do front

```text
app/dashboard/contacts/
  page.tsx                    # listagem
  [contactProfileId]/page.tsx # ficha da pessoa; ID diferente do Contact operacional
  components/                # cadastro, negócios e histórico autorizado
  hooks/                     # consultas paginadas e mutations
  services/                  # contratos de cadastro/composição
  types/                     # ficha de contato
```

Componentes compartilhados com atendimento devem ter um dono claro e contratos neutros.
Não migrar os 25 consumidores da pasta support/contacts de uma vez sem necessidade (F11).
Seletores de dados da API seguem o padrão pesquisável Popover+Command do produto.

## Compatibilidade dos dados e integrações

1. Manter IDs e identidades usados por mensagens, conversas, automações e cards.
2. Endereço é aditivo e opcional; cadastro antigo continua legível/editável.
3. Não copiar `Card.fields` nem variáveis indiscriminadamente para o cadastro (F9).
4. Auditar os cards sem contactId com o script existente antes de prometer cobertura histórica total;
   aplicação de backfill é uma operação separada, com evidência e reversão, não parte desta análise.
5. Lista de negócios existentes não garante retenção de cards apagados; se retenção for requisito,
   criar requisito próprio e examinar exclusão/snapshots (F6).
6. Integração cadastro → automação exige mapeamento e precedência; não tornar todo campo privado
   automaticamente uma variável acessível ao agente.

## Plano de verificação para as stories

| Frente | Prova necessária |
|---|---|
| Cadastro | Endereço opcional, patch parcial, preservação de campos, identificador imutável, legacy sem endereço |
| Tenant | Lista, busca, detalhe, edição e subscription não cruzam tenants |
| Acesso | Matriz read/manage/lookup, grants de perfil, seções independentes e chamadas diretas da API |
| Contextos | Cadastro global segue role; card/tarefa preservam as políticas existentes |
| Mensagens | Histórico autorizado funciona com cursor e filtro; atendimento mantém busca/envio/ownership |
| CRM | Vários cards e resultados existentes; evento por card; sem reabrir/criar negócio por consultar a ficha |
| Tarefas | Escopo por responsável, contexto de card/conversa, deduplicação e paginação |
| Rotas | Registry/guard/sidebar/breadcrumb, base/detalhe antigos redirecionam e abrem a seção correta |
| Interface | Estados vazios/erro/parcial, mobile e edição/reflexo no atendimento |
| Wiring | Registro único de modelos/handlers e ausência de ciclos se houver extração Nest |

Usar os testes pertinentes existentes como base. Por entrega de código, rodar lint, typecheck,
test e build de cada projeto alterado, além da validação visual autenticada apropriada.
Nenhum desses resultados de implementação foi produzido nesta análise.

## Encaminhamento

Pessoa com vários meios foi confirmada pelo owner. Refinar o modelo B, os acessos e a associação
de identidades conforme o documento de domínio; então derivar as stories com esses contratos explícitos.
Esta investigação não altera código, permissões, dados, navegação ou status das stories existentes.
