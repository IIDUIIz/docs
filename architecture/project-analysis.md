# Análise do projeto: módulo Contatos

Data: 25/10/2026. Responsável: @architect (Aria).
Estado: investigação do código e proposta para discussão; não é ADR aceita nem autorização de migração de dados.
Documento complementar: [Abordagem recomendada](recommended-approach.md).

Atualização de 26/10/2026: o owner confirmou **uma ficha da pessoa com vários meios de contato**.
A modelagem aprofundada e as revisões de dados/produto estão em
[Modelo de domínio](contact-domain-model.md). As observações deste documento sobre falta de decisão
de agrupamento descrevem o estado da investigação de 25/10; foram superadas por essa confirmação.

## Pedido e limites

O owner pediu avaliar a evolução de `/dashboard/reports/contacts` para uma área própria em
`/dashboard/contacts`, com cadastro ampliado, endereço e histórico dos negócios do contato.
Também pediu examinar se isso deve ser um módulo separado e olhar as dependências existentes.

Os requisitos já expressos são: enriquecer o cadastro, consultar os negócios anteriores com seus
resultados existentes e avaliar a nova área. Campos adicionais, novos grants para os perfis,
unificação de identidades e eliminação do histórico não foram decididos.

Entradas da análise: feature descrita nesta conversa; projetos `front-end` e `back-end` no workspace;
integração externa não necessária para o cadastro manual proposto; mudança de dados necessária para
persistir endereço, que não existe no modelo atual. Consulta de CEP não foi solicitada.

Escopo verificado: rotas, navegação, guards, políticas por equipe, cadastro/identidades, busca,
mensagens/REST/socket, cards/eventos de CRM, tarefas, variáveis e artefatos relacionados.
Não houve acesso ao banco, execução de backfill, validação visual ou chamadas a provedores.

## Estrutura e padrões

| Área | Estrutura atual | Consequência para a mudança |
|---|---|---|
| Front-end | Next App Router, React, TypeScript, kit compartilhado | Nova área cabe em `app/dashboard/contacts`; registry controla menu e acesso |
| Back-end | NestJS, CQRS, Mongo/Mongoose, módulos por domínio | Reusar casos de uso existentes; composição precisa aplicar autorização própria |
| Contatos | Dentro de `modules/conversation` | Área independente no produto não exige mover a identidade de mensagens imediatamente |
| CRM | `modules/pipelines`, rotas `/pipelines/*` | Cards e seus eventos continuam pertencendo ao CRM |
| Tarefas | `modules/tasks` | Consulta por contato precisa combinar vínculos e preservar escopo das tarefas |
| Automação | `modules/automation` + `modules/variables` | Dados capturados não são automaticamente campos de cadastro |
| Acesso | Permissions/capability bundles, não entitlement por módulo | Reusar RBAC; não criar eixo novo de billing/planos |
| Testes | Front Vitest; back Jest e testes de wiring de módulos | Validar navegação, autorização, contratos, isolamento e regressões de mensagens |

Ambos os `package.json` atuais têm `lint`, `typecheck`, `test` e `build`.
A análise usa o workflow de estrutura do AIOX aplicado aos módulos do produto; não há razão para
criar um serviço desta feature dentro de `.aiox-core/infrastructure/services`.

## Inventário e achados

### F1 — A página atual é histórico de mensagens

- `front-end/app/dashboard/reports/contacts/layout.tsx`: lista de contatos + detalhe.
- `contacts-list.tsx`: busca a partir de dois caracteres; até 50 resultados; navega pelo ID.
- `[contactId]/page.tsx`: mensagens consolidadas, filtro por canal, paginação por cursor,
  limites visuais entre conversas e atualização por socket.
- `page.tsx`: estado vazio pedindo seleção de contato.
- `front-end/config/route-registry.ts`: a área está em Relatórios, com `conversation:monitor`.

O comentário da tela registra a decisão anterior de manter a consulta sem envio de mensagens fora
de uma conversa real. Transformar a navegação não implica revogar essa regra.

### F2 — Permissões próprias de Contatos já existem

`back-end/src/modules/settings/users/domain/permissions.ts` define `contacts:read`,
`contacts:manage` e `contacts:lookup`. `capabilities.ts` já tem `capability:gestao-contatos`.
`contact.controller.ts` usa read para listagem/busca e manage para criação/edição.

Os perfis de sistema Gerente e Atendente não recebem gestão global de contatos nos bundles
examinados; recebem lookup em atendimento/CRM. O CRM tem caminhos próprios de busca/criação
com `crm:operate`. Portanto, mover a rota para `contacts:read` não determina sozinho quais
usuários existentes passam a enxergá-la. Grants personalizados não foram consultados no banco.

### F3 — Os gates do histórico divergem hoje

| Superfície | Gate atual verificado |
|---|---|
| Navegação de Relatórios/Contatos | `conversation:monitor` |
| Busca global de contatos usada pela lista | `contacts:read` |
| `POST /mensagens/get-messages-by-timestamp` | `conversation:attend` |
| `ConversationAccessService.canSpyContact` | `conversation:attend` + contato no tenant |

O endpoint de mensagens é usado pela tela de histórico **e** por
`support/conversation/context/conversation-message-context.tsx`.
Não trocar seu guard indiscriminadamente por `contacts:read` ou `conversation:monitor`.

O handler/repositório desse histórico filtram tenant, contato e canal opcional. Isso não equivale
a um filtro por atendente/equipe. A autorização da aba consolidada precisa ser explícita.

### F4 — Contact ainda é identidade de comunicação

`conversation/infra/mongo/entities/contact.schema.ts` e `domain/types/contact.types.ts`:
`tenantId`, `type`, `value`, `name`, `profile`, disponibilidade por provider e timestamps.
`type` é phone/email/instagram; a chave única é tenant + value.

Não existem endereço, observações ou definições de campos cadastrais nesse modelo.
`UpdateContactHandler` e o repositório só atualizam nome/foto; value é imutável.
DTO, aggregate, mapper, repositório, respostas e clientes precisam evoluir em conjunto.

`ProviderIdentity` relaciona identificadores de provedores ao Contact, incluindo JID/LID e
escopo de conta para Instagram. Essa infraestrutura resolve entrega/recebimento; não prova que
um telefone, um e-mail e um Instagram pertencem à mesma pessoa. Não foi encontrado um
cadastro de pessoa com agrupamento explícito desses Contact IDs no escopo examinado.

### F5 — Há referência legada a Customer, sem módulo correspondente

O card conserva `customerId` com ref Customer; o front também possui tipos/apresentações legados.
Não há registro de model/módulo Customer nos arquivos de módulo/schema examinados.
O epic 26 já registra esse problema. Não usar essa referência como prova de cadastro mestre pronto.

### F6 — Histórico de negócios tem base pronta

- `pipelines/infra/mongo/entities/card.schema.ts`: `contactId`, valor, funil, etapa, status,
  tipo/ID de motivo, `postponedUntil`, campos e timestamps.
- `GET /pipelines/cards`: `contactId` + `includeClosed=true`, gate `crm:read`.
- `use-contact-deal.ts`: já busca todos pelo contato, mas `pickDeal` apresenta só um aberto
  ou o primeiro resultado quando não há aberto.
- `pipeline-event.schema.ts`/`GET /pipelines/events`: eventos por card; leitura usa `crm:monitor`.
- `CardTimeline.tsx`: renderização existente, inclusive eventos antigos de atividade.

A lista histórica pode ser construída sem copiar cards para o Contact. Ela deve preservar os
resultados/motivos já existentes, inclusive perda, prorrogação e descarte conforme os contratos
atuais. Não derivar uma data de fechamento de `updatedAt`: qualquer edição muda essa data.

Limites: cards sem `contactId` ficam fora da busca; há runbook/script `cards:link-contacts` para o
legado, mas a cobertura atual do banco não foi medida. `DeleteCardHandler` remove o registro:
uma lista de cards existentes não é um arquivo imutável de toda negociação já ocorrida.
Eventos não são um substituto automático para snapshots completos de cards removidos.

### F7 — Visibilidade depende do contexto

ADR-006 D4 e implementação atual:

- Cadastro/lista geral de contato: acesso pela role, dados completos quando autorizado.
- CRM: dados do contato seguem a equipe do card, via `ContactPresenter`.
- Rotas próprias de monitoramento: têm regra de dados completos.
- Tarefas ligadas a conversa: seguem a equipe daquela conversa.

A nova área pode reunir seções com regras diferentes. O acesso ao cadastro não dá, por si só,
permissão para consultar todos os negócios, eventos ou mensagens. CQRS interno não executa os
guards dos controllers REST: uma composição nova precisa reaplicar os gates e o escopo necessários.

### F8 — Tarefas não estão todas ligadas diretamente ao contato

`CardTaskService` cria tarefas com `contextType: crm_card` e `contextId: card.id`.
O schema também suporta contexto de conversa e campos `conversationContactId` legados/específicos.
O campo textual `contact` não é uma FK confiável de pessoa.

`TaskRepository.findByContext` existe, mas a listagem HTTP geral e `TaskListFilters` não expõem
um filtro de contato/contexto para montar essa ficha. `TaskRequestAccessService` restringe a
listagem ao responsável quando o usuário não gerencia tarefas.
Uma futura aba precisa obter tarefas dos cards/conversas do contato com consulta em lote,
paginação/deduplicação e autorização do próprio módulo; não chamar uma consulta por card.

### F9 — Existem três espaços diferentes de dados

| Dados | Persistência atual | Uso |
|---|---|---|
| Identidade/cadastro básico | Contact | Resolver e exibir o contato |
| Campos da negociação | `Card.fields`, definições no funil | Dados que variam entre negócios |
| Variáveis capturadas | Variable com `contactID` | Execução de fluxos e placeholders |

O Input da automação pode escrever variável do contato. Isso não atualiza o cadastro nem prova
que aquele dado está correto/atual. A query de variáveis por contexto examinada não recebe tenant
explicitamente; não a expor na nova ficha sem validar o isolamento completo do caminho.
Qualquer sincronização cadastro/variáveis precisa de mapeamento e precedência deliberados.

### F10 — A busca atual é reaproveitável, com limites

`POST /contacts/list` aceita página/limite/busca; sem paginação pode retornar todos os contatos.
`findAll` aplica paginação se informada, mas não define ordenação estável.
`GET /contacts/search` busca os resultados e só depois limita em memória.
A nova listagem deve paginar/ordenar no servidor e devolver contagens coerentes; não carregar
todo o tenant para produzir filtros ou resumos locais.

### F11 — A extração física tem alcance maior que a tela

`ConversationModule` registra contatos junto de mensagens/canais e exporta o presenter/policy
consumidos por CRM e tarefas; `PipelinesModule` já importa ConversationModule.
Busca dirigida encontrou 39 arquivos do back com imports dos contratos de contato examinados
e 25 arquivos do front importando a pasta compartilhada atual `support/contacts`.
São indicadores de impacto, não um inventário exaustivo de todo acoplamento.

Consequência: criar módulo de navegação/feature agora é viável. Extrair a propriedade de Contact
para um novo módulo Nest requer trabalho próprio de dependências, registro único do schema,
contratos CQRS e testes de wiring. Não registrar o mesmo handler em dois módulos.

## Restrições existentes que continuam valendo

1. Tenant obrigatório nos dados, consultas, edição e subscriptions.
2. A política da ADR-006 continua contextual; autorização de cadastro não substitui as demais.
3. Envio continua pertencendo a conversa real; nova área não ganha envio paralelo implicitamente.
4. Campos do funil continuam no negócio; variáveis de automação mantêm seu contrato.
5. Módulo funcional não reintroduz entitlement/billing por módulo (front Story 5.3).
6. Preservar alterações simultâneas; os dois repos tinham mudanças de outras tarefas no início.

## Evidência, validação e lacunas

A análise foi feita por leitura e busca dos consumidores, contratos e artefatos citados.
Foram conferidos os scripts de verificação disponíveis e a existência dos testes pertinentes.
Não foram rodados lint/typecheck/test/build: este trabalho só acrescenta documentação de arquitetura.
Não houve QA de runtime, auditoria de dados, prova de deploy ou verificação de quais grants cada
tenant realmente possui. Os achados descrevem o checkout de 25/10/2026 e podem evoluir por outras sessões.

Os testes de implementação deverão cobrir contatos/isolamento, rota/menu, histórico REST/socket,
vínculos de card, política de equipe, escopo de tarefas e wiring de módulos quando houver extração.

Complemento de verificação de 26/10: quatro suítes existentes passaram (37 testes unitários),
conforme o registro no modelo de domínio. A modelagem nova não foi implementada ou testada em runtime.
