# Auditoria das regras de arquitetura do Hadem — 07/11/2026

Só leitura. Nenhuma regra, ADR, story ou código foi editado. Este arquivo é o único produto.

Caminhos relativos a `C:\Users\dudus\Hadem\` (a pasta raiz não é repo). Citações no formato `arquivo:linha`.

## 0. Escopo e versão lida

- Lidos: raiz (`CLAUDE.md`, `AGENTS.md`, `.claude/`, `docs/architecture/`); `back-end` (`.claude/CLAUDE.md`, 13 rules, ADR-001 a 010, `ai-agents.md`, briefing, design, 3 docs soltos, 6 READMEs de módulo, epics 5, 8, 11, 30, stories 25.9, 30.x, 31.x, agent-memory do architect e do sm); comparação por conteúdo das rules entre a raiz e 5 repos; `front-end`, `hadem-admin`, `hadem-admin-api`, `hadem-kit`, `hadem-edge`, `hadem-keycloak` (READMEs, `.claude/`, `DECISIONS.md`); página "ADR-010" do Notion (só o trecho devolvido pela busca).
- **Versão final usada para a ADR-010 e a regra** (relidas às 20:02): `back-end/docs/architecture/adr-010-ddd-estrutura-dos-modulos.md` com 174 linhas (gravada 20:00:50) e `back-end/.claude/rules/ddd-modulos.md` com 67 linhas (20:00:50). Elas ganharam a D7 (regras 2 e 3) durante a auditoria; a versão anterior (126 e 59 linhas) não tinha a regra 3, e a primeira parte da minha leitura foi feita nela. As seções abaixo já estão ajustadas à versão final.
- As stories 31.2 a 31.4 estão sendo reescritas por outro agente (arquivos de 19:47-19:48, ainda sem a D7 no texto). Cito o que estava em disco.
- Não verifiquei código para provar violações; onde cito código é porque uma doc o descreve.

## 1. As três regras: onde estão escritas

### Regra 1 — DDD em todo módulo (camadas, aggregate, domínio puro)

| Onde | O que diz | Completa? |
|---|---|---|
| `back-end/docs/architecture/adr-010-…md:50-66` (D1) | Camadas `application/`, `domain/`, `infra/`, `presentation/`; nada mais na raiz; `README.md` obrigatório (l.57); "`infra`, nunca `infrastructure`" (l.64) | Sim |
| `adr-010:68-82` (D2) | Pastas de `domain/` | Sim |
| `adr-010:84-91` (D3) | Aggregate é classe com invariantes; concorrência no repositório | Sim |
| `adr-010:93-95` (D4) | Domínio puro | Sim |
| `back-end/.claude/rules/ddd-modulos.md:10-23, 25-38, 42-46` | Versão curta | Sim; carrega só ao editar `src/modules/**` (l.2-4) |
| `back-end/docs/conversation-module-reorganization.md:23-35, 160-175` | Camadas, só do `conversation` | Parcial (escopo antigo) |

Veredito: **escrita e completa para o back-end.** Furos: (a) `README.md` obrigatório (`adr-010:57`), mas só 6 dos 21 módulos têm (`automation`, `catalog`, `causes`, `conversation`, `goals`, `tasks`); (b) "nunca `infrastructure`" (`adr-010:64`, `ddd-modulos.md:22`) convive com `src/shared/infrastructure/` (usado pela própria D7.5, `adr-010:125`, e imposto por `back-end/.claude/rules/observability-structure.md:25`); falta dizer que D1 vale só para `src/modules`; (c) a regra não diz se vale para `hadem-admin-api` (ver 3.4).

### Regra 2 — Um módulo por bounded context; módulos não se acoplam; cada módulo pode virar microsserviço

| Onde | O que diz | Completa? |
|---|---|---|
| `adr-010:99-105` (D5) | Contexto com ciclo próprio é módulo próprio; só fala com outro por porta, command ou evento (D7); nunca importar `domain/`, `infra/`, `application/services/` de outro (l.103); exporta só portas (l.101) | Sim |
| `adr-010:116` (D7) | "módulos não se acoplam; cada módulo vai virar microsserviço no futuro" | Sim |
| `adr-010:159` (Consequências) | "um módulo sai para serviço próprio … muda o transporte de porta, command e evento" | Sim |
| `ddd-modulos.md:47, 52` | "Módulos não se acoplam; cada um vai virar microsserviço" | Sim |
| Qualquer outro arquivo | — | **Ausente** (só `adr-008:232` e `adr-009` falam de microsserviço, para borda e gateway) |

Veredito: **completa na ADR e na regra; ausente de todo o resto** (`CLAUDE.md`, READMEs, epics, templates). Furo no conteúdo: o `bounded context` só é obrigatório para "ciclo de vida próprio" (l.99) e a tabela de débito só trata Person (l.140), deixando outros contextos dentro do `conversation` sem registro (2.6).

### Regra 3 — Só contrato, command ou evento; sem transação nem coleção alheia; escrita multi-módulo por evento (outbox, idempotente)

| Onde | O que diz | Completa? |
|---|---|---|
| `adr-010:118-121` (D7.1) | Porta (DTO próprio), command, evento, "e só elas"; sem tipo de outro módulo | Sim |
| `adr-010:122` (D7.2) | Nenhuma transação atravessa módulos; sessão (ou "tipo que a esconda") nunca passa por porta, command ou evento | Sim |
| `adr-010:123` (D7.3) | Nenhum módulo lê/grava coleção de outro, nem por schema, nem por nome de model/coleção | Sim |
| `adr-010:124-127` (D7.4-7) | Escrita multi-módulo por evento; outbox (`src/shared/infrastructure/outbox/`, existe no código); handlers idempotentes; seguir redirecionamento | Sim |
| `adr-010:112` (D6.4) | Trocar transação por evento muda a garantia; story própria | Sim |
| `adr-010:163-174` | 8 questões abertas (contrato de command/evento, CAS perdido, journal, `tenant-sync`…) | Pendente do owner |
| `ddd-modulos.md:52-58, 66` | Resumo da D7 | Sim |

Veredito: **escrita e completa no essencial.** Pendências de conteúdo: **Query** entre módulos não aparece (nem `QueryBus`) em D7 nem na regra (2.7); onde mora a classe do command/evento é a Questão 5 (`adr-010:171`), em aberto.

## 2. Contradições

Gravidade: **A** = manda fazer o oposto da regra; **B** = descreve a prática contrária como normal; **C** = ambígua ou desatualizada.

### 2.1 (A) Epic 30, P1: Person dentro do `ConversationModule`, "nenhum módulo novo nem microserviço"

- `back-end/docs/stories/epic-30-pessoa-varios-meios-de-contato.md:33` (P1): "`Person` vive no `ConversationModule` (pasta `person/`) … Nenhum módulo novo nem microserviço. O CRM lê pessoas por `QueryBus`."
- Origem: `docs/architecture/contact-domain-model.md:396-398` ("O módulo funcional Contatos é próprio; … não exige microserviço … A extração física deve preservar registro único de schema/handlers"). Reforço: `docs/architecture/recommended-approach.md:29-30` ("sem serviço separado, novo deploy") e `docs/architecture/project-analysis.md:182-184`.
- O epic 30 não menciona ADR-010 em nenhuma linha (busca por "ADR-010/D7/ddd-modulos" sem resultado). Não há nota de substituição. O cabeçalho (l.4) ainda chama o documento de domínio de "fonte única do desenho". A story 31.2 diz "só o P1 é substituído" (`31.2…story.md:211`) e a 31.4 só **propõe** a edição (`31.4…story.md:132`, Task 7).
- P2 (`epic-30…:34`): `Contact.personId` fica no `conversation`; vira FK entre módulos na regra nova. A ADR cobre como `contacts` ↔ `people` por evento (`adr-010:140`), mas o epic não.

### 2.2 (A) Epic 30, P7, e stories 30.x: transação Mongo única em coleções de vários módulos

- `epic-30…:39` (P7): "Usar transação Mongo onde a topologia suportar". `epic-30…:79`: `PersonCardsPort.repoint(..., session)` na "mesma unidade de trabalho".
- `30.4…story.md:52, 187`: a porta carrega `ClientSession`. `30.7…story.md:111, 334, 350`, `30.8…:168, 248-249`, `30.9…:53, 298`: merge, unlink e undo em `session.withTransaction` gravando `people`, `contacts`, `person_operations`, `cards`.
- Contradiz `adr-010:122` (D7.2). A ADR já trata isso como débito (`adr-010:37, 140, 157`), mas nenhum dos documentos de origem foi marcado como superado. Quem lê o epic 30 hoje lê o contrário da regra como desenho vigente.

### 2.3 (B) O exemplo de porta exigida da ADR e da regra é a porta que carrega a transação

- `adr-010:102` e `ddd-modulos.md:49` citam `PersonCardsPort` como exemplo ("como `PersonCardsPort`, implementada pelo `pipelines`"). A implementação atual recebe `ClientSession` (`30.4…:52, 187`) e a própria ADR lista essa sessão como coisa a sair (`adr-010:140`). O exemplo ensina a forma certa (porta exigida) com um conteúdo proibido. Pede nota ou outro exemplo.

### 2.4 (A) Stories 31.2 a 31.4 (em reescrita): preservam a transação atrás de um tipo opaco

- `31.3…story.md:40-41, 62, 88, 100, 141` e `31.2…story.md:60, 178`: `PersonUnitOfWork` / `PersonTransaction` "opaco" mantém a atomicidade entre `people`, `contacts` e `cards`.
- `adr-010:122` proíbe exatamente isso: "`ClientSession` **ou tipo que a esconda**". A ADR já registra o descompasso: a decisão 3 do epic 31 "deixa de valer", 31.3 e 31.4 voltam ao @sm, e a 31.2 segue mantendo a transação até a story do evento (`adr-010:140, 174`). Conferir o texto final.

### 2.5 (C) Catalog: desenho original prescreve a leitura proibida

- `docs/architecture/catalog-module.md:138`: "adapter de referências … injetando `Connection` Mongo, para consultar coleção e `flows`" — leitura de coleção alheia. A ADR agora classifica como débito aberto (`adr-010:139`, 1c, risco "Médio"); `back-end/src/modules/catalog/README.md:136-139` lista as dívidas mas sem dizer que violam a D7. O doc de desenho continua mandando fazer.

### 2.6 (B) Mais de um bounded context ainda mora no `conversation`, e o débito só lista a Person

- `adr-010:99` exige módulo próprio para contexto com ciclo próprio; `adr-010:140` só trata Person. A palavra "Team", "Channel" e "Broadcast" não aparece em nenhum item de débito.
- `back-end/docs/stories/epic-8-crm-modulo-proprio-scoping.md:18`: "Team vive no módulo `conversation`". `25.9.contexto-sem-equipe.story.md:48`: `team_access_settings` registrada no `conversation.module.ts`. `conversation/README.md:3-12` junta canal, contato, conversa, mensagem e broadcasts.

### 2.7 (C) Query entre módulos: nenhuma decisão

- `adr-010` e `ddd-modulos.md` não mencionam Query nem `QueryBus` (busca por "query" sem resultado). D7.3 manda ler "por porta" ou guardar cópia por evento (l.123).
- Prática e docs dizem outra coisa: `epic-5-integracao-conversation-crm.md:48` ("toda comunicação via CommandBus/QueryBus/EventBus"), `adr-002-agent-memory.md:34` (`ai-agents` ↔ `variables` por CommandBus/QueryBus), `epic-30…:33` ("O CRM lê pessoas por `QueryBus`"), `back-end/.claude/agent-memory/aiox-architect/project_crosss_module_cqrs.md:8-14` (QueryBus aprovado como idioma). Resultado: o que valia como regra (só na memória de um agente) não foi nem adotado nem revogado.

### 2.8 (C) READMEs de módulo

- `conversation/README.md:129`: `automation` como "integração interna", sem dizer por qual contrato; não menciona Person, Team nem as leituras de coleção alheia que a ADR lista (`adr-010:141`).
- `automation/README.md:89-90`: consome `ConversationTransferredEvent` e `ConversationClosedEvent` "do módulo `conversation`"; não diz onde mora a classe (Questão 5 da ADR, aberta).
- `goals/README.md:7` e `tasks/README.md:9` apontam para `docs/GOALS_MODULE_DESIGN.md`, que **não existe** em lugar nenhum; `goals/README.md:17` lista como "próximo passo" implementar handlers de evento que já existem (`goals/application/events/tarefa-concluida.handler.ts`, `tarefa-reaberta.handler.ts`).

## 3. Documentos velhos ou duplicados

### 3.1 Tabela de rules do `back-end/.claude/CLAUDE.md` não lista a `ddd-modulos.md`

- `back-end/.claude/CLAUDE.md:133-150` (bloco `AIOX-MANAGED rules-system`) lista 8 arquivos; o back-end tem 13 rules. Faltam: **`ddd-modulos.md`**, `observability-structure.md`, `handoff-consolidation.md`, `tool-examples.md`, `tool-response-filtering.md`.
- O bloco está entre marcadores `AIOX-MANAGED-START/END` (l.133, l.150): edição ali é apagada no próximo sync do AIOX. Inclusão tem de ficar **fora** dos marcadores.

### 3.2 `CLAUDE.md` e `README` do back-end não apontam para a arquitetura

- O `.claude/CLAUDE.md` da raiz e o dos 5 repos (`back-end`, `front-end`, `hadem-admin`, `hadem-admin-api`, `hadem-kit`) têm **conteúdo igual** (comparação ignorando CRLF): template AIOX, sem palavra sobre módulos, DDD ou ADRs.
- `back-end/` não tem `CLAUDE.md` na raiz do repo. `back-end/README.md:50` ("Ver `docs/` … conversation-module-reorganization, message-flow") não cita `docs/architecture/` nem as ADRs.

### 3.3 Raiz: `CLAUDE.md` e `AGENTS.md` não descrevem o projeto

- `CLAUDE.md` (64 linhas): guia genérico de comportamento, sem arquitetura.
- `AGENTS.md:23-31` ("Project Map": `bin/`, `packages/`, `tests/`) e `:36-41` (`npm run sync:ide`, `validate:structure`…): descrevem o framework AIOX; essas pastas e scripts não existem na raiz do Hadem (sem `package.json`).

### 3.4 Rules copiadas da raiz para os repos

- Dez rules são **idênticas** na raiz e nos 5 repos: `agent-handoff`, `agent-memory-imports`, `coderabbit-integration`, `handoff-consolidation`, `ids-principles`, `mcp-usage`, `story-lifecycle`, `tool-examples`, `tool-response-filtering`, `workflow-execution`. `agent-authority.md` difere só por uma nota de 16.4 (l.3-4) nos 5 repos. Nenhuma é de arquitetura do Hadem.
- `story-lifecycle.md` (checklist de 10 pontos do @po e 7 verificações do @qa) **não tem item de DDD, fronteira de módulo, transação ou acoplamento**, embora `adr-010:109` diga que "revisão de código e gate de QA recusam o que estiver fora". O gate prometido não está no checklist.
- Só o `back-end` tem `ddd-modulos.md` e `observability-structure.md`. O `hadem-admin-api` (NestJS, `src/modules/{billing,tenants,…}`) não tem rule de estrutura, e seus módulos misturam `infra/` e `infrastructure/` (ex.: `hadem-admin-api/src/modules/billing/`, `…/tenants/`), contra o espírito de `adr-010:64`.
- Não há lint, `dependency-cruiser`, `eslint-plugin-boundaries` ou `no-restricted-paths` no back (busca em `package.json` e configs sem resultado). Existe `back-end/test/module-graph/`, mas nada o liga à regra.

### 3.5 Quatro cópias da ADR-010

| Cópia | Estado |
|---|---|
| `back-end/docs/architecture/adr-010-…md` (174 linhas) | Texto completo, 2 revisões datadas (l.6-7) |
| `back-end/.claude/rules/ddd-modulos.md` (67 linhas) | "Versão curta", já com a D7 |
| Notion "ADR-010 — DDD como regra" (`app.notion.com/p/4107b72f3d1f8116bb09d97819095699`, 06/11) | Trecho devolvido: "Entre módulos, só port/adapter ou evento". **Anterior às duas revisões de 07/11.** A página "Arquitetura" do Notion (04/11) ainda lista "ADRs 001 a 009" |
| READMEs (`catalog/README.md:123`) | Citam D5 por nome |

Processo: a página "6. Decisões e ADRs" do Notion diz que "ADR aceita não é reescrita: decisão nova vira outra ADR, ou registro datado na própria". A ADR-010 usou o registro datado (l.6-7), dentro do que o processo permite.

### 3.6 Outras velhas

- `docs/architecture/contact-domain-model.md:3-5`: "proposta de modelagem; não há alteração de schema" — as stories 30.x já foram implementadas; segue como "fonte única" do epic 30 (`:4`). `:325-328` oferece "transação" como opção (superada).
- `epic-30:4` chama esse arquivo de "monorepo `Hadem`", mas `back-end/README.md:7-11` diz que o monorepo foi arquivado. Na prática o arquivo está em `docs/architecture/` na raiz, num repo próprio (`github.com/IIDUIIz/docs`). Desenhos de módulos do back vivem fora do repo do back.
- `back-end/docs/conversation-module-reorganization.md:140-197`: "documento vivo" com fases 2-4 planejadas, em paralelo à tabela de débito da ADR-010; duas listas de "o que falta arrumar".
- `back-end/.claude/skills/architect-first/SKILL.md:25, 93, 163`: "Zero Coupling" para expansion packs do AIOX, não para módulos Nest; pode ser lido pelo @architect como regra do Hadem.
- `front-end/DECISIONS.md` só tem ADR de i18n; `hadem-kit/CLAUDE.md` é só UI. Os repos de front não têm regra de módulo que contradiga o back.

## 4. Lacunas (o que deveria estar escrito e não está)

O princípio e os três mecanismos já estão na ADR e na regra (seção 1). Falta:

1. **Um ponto de entrada que leve à regra.** `ddd-modulos.md` só carrega ao editar `src/modules/**` (l.2-4). Quem escreve epic, story ou ADR (@sm, @po, @architect) não a carrega; foi assim que P1 e P7 do epic 30 nasceram e foram validadas. Nem `CLAUDE.md`, nem `README.md`, nem os templates de story/epic apontam para ela, e nenhum template pede uma seção "fronteira de módulo" (portas, commands e eventos usados e expostos).
2. **Enforcement.** Nenhum lint/regra de import entre módulos; nenhum item no checklist do @po/@qa (3.4). A D6.1 promete um gate que não existe.
3. **Query entre módulos** (2.7): permitida por `QueryBus`? só por porta? por cópia via evento?
4. **Onde mora a classe do command/evento/porta** que atravessa módulos e como é versionada (Questão 5, `adr-010:171`, aberta); formato e versionamento do payload do evento (os READMEs divergem: `automation/README.md:153` cita `correlationId` e `version`, os outros não).
5. **Guia de uso do outbox** para quem escreve handler: convenção de `type`, ordem, retry/dead-letter, como guardar "cópia por evento" no consumidor, compensação quando o CAS do consumidor perde (Questões 1-4). A ADR cita o código (`adr-010:38`), mas não há texto de "como fazer".
6. **Mapa de propriedade**: qual módulo é dono de qual coleção e o grafo de dependências hoje entre os 21 módulos. Sem isso, o débito de D7.3 se descobre por busca; o número que existe (`adr-010:30`: 69 arquivos) cobre só `conversation`.
7. **Débito completo**: Team, Channel, Contact, Broadcast dentro do `conversation` (2.6) e a FK `Contact.personId` como contrato.
8. **Alcance**: a regra vale para `hadem-admin-api`? Para `hadem-channels` (ADR-009) e `hadem-edge` (ADR-008), que já são fronteiras de serviço?
9. **Leituras agregadas entre módulos** (listas combinadas, ex.: negócios da pessoa; resumos): sem orientação de paginação, junção e consistência quando cada módulo tem a sua coleção.

## 5. Proposta de fonte única (só proposta; o owner decide)

**Fonte normativa:** `back-end/.claude/rules/ddd-modulos.md` (curta, em português, sem fatos) + `adr-010` (porquê, fatos, débito, questões). Todo o resto **aponta**, não repete.

| Arquivo | Ação proposta |
|---|---|
| `adr-010-…md` | Decidir Query (Questão nova); nota em `:102` sobre o exemplo `PersonCardsPort`; incluir Team/Channel/Broadcast/Contact no débito ou dizer que não são contextos separados; dizer que D1 (`infra`) vale só em `src/modules` |
| `ddd-modulos.md` | Mesmo ajuste de exemplo (`:49`); linha sobre Query; ampliar `paths:` para `docs/stories/**` e `docs/architecture/**` (ou rule curta sem `paths` que mande ler esta) |
| `back-end/.claude/CLAUDE.md` | Bloco "Arquitetura do Hadem" **fora dos marcadores** `AIOX-MANAGED` (5 linhas + link para a rule e a ADR); incluir as 5 rules ausentes na tabela (`:138-147`), também fora do bloco gerenciado |
| `back-end/README.md:50` | Apontar para `docs/architecture/` e a rule |
| `epic-30-…md:33-34, 39, 79` | Nota datada de **superação** de P1, P2, P7 e do trecho de `:79` citando ADR-010 D5/D7; manter o texto original |
| `30.4`, `30.7`, `30.8`, `30.9` | Nota curta no topo: transação entre módulos é débito (`adr-010:140`) |
| `31.2`, `31.3`, `31.4` | (em reescrita) sem `PersonTransaction`/`UnitOfWork` atravessando módulos; ver `adr-010:122` |
| `docs/architecture/contact-domain-model.md` | Status no topo (`:3-5`): parcialmente implementado; §7 (`:325-328`) e §10 (`:396-398`) superados pela ADR-010 |
| `docs/architecture/recommended-approach.md:29-30`, `project-analysis.md:182-184` | Nota de histórico, ou tirar "sem serviço separado" |
| `docs/architecture/catalog-module.md:138` | Nota: leitura de `flows` por `Connection` contraria D7.3 (já débito 1c) |
| `back-end/src/modules/*/README.md` | `conversation/README.md:129` e `automation/README.md:89-90`: dizer o contrato (porta, command, evento) e onde mora; `goals`/`tasks`: tirar o link quebrado e a tarefa já feita; criar README nos 15 módulos que faltam ou relaxar `adr-010:57` |
| `back-end/.claude/rules/story-lifecycle.md` (+ cópias) | Itens de fronteira no checklist do @po e no gate do @qa: só porta/command/evento? transação ou coleção alheia? |
| `back-end/.claude/agent-memory/aiox-architect/project_crosss_module_cqrs.md` | Depois da decisão sobre Query, atualizar a memória e apontar para a rule |
| Notion "ADR-010" e "Arquitetura" | Alinhar à versão final; atualizar "ADRs 001 a 009" |
| Raiz `AGENTS.md:23-41` | Corrigir ou remover "Project Map" e scripts que não existem |
| `hadem-admin-api` | Decidir se a rule vale; se sim, copiar adaptada e resolver `infra/` × `infrastructure/` |
| `back-end/.claude/skills/architect-first/SKILL.md` | Nota: "coupling" ali é de expansion pack |
| Enforcement | Avaliar regra de import entre módulos no lint (ou teste em `test/module-graph/`) |

Sem edição proposta (conferidos, sem conflito): `front-end/DECISIONS.md`, `hadem-kit/CLAUDE.md`, `hadem-edge/README.md`, `hadem-keycloak/README.md`, ADR-001 a 009 (ADR-008 e 009 já tratam gateway e borda como serviços separados).

## 6. Perguntas para o owner

1. **Query entre módulos**: continua por `QueryBus` (como no epic 5, ADR-002 e na memória do architect), vira só porta, ou vira cópia por evento?
2. **Team, Channel, Broadcast e Contact** são contextos separados a tirar do `conversation`, ou ficam?
3. **Epic 30 e stories 30.x já Done**: bastam as notas de superação, ou a transação de merge/unlink/undo entra no plano como story própria (já é o item 2 do débito)?
4. **Alcance da regra**: vale para `hadem-admin-api`?
5. **Entrada da regra**: aceita carregar a rule também ao editar `docs/stories/**` e `docs/architecture/**`?

## 7. Releitura final

ADR-010 (174 linhas) e `ddd-modulos.md` (67 linhas) relidos às 20:02 de 07/11/2026. Se mudarem depois, as linhas citadas em `adr-010:` podem deslocar.
