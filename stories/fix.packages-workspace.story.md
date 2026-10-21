# Story fix.packages-workspace: Corrigir problemas de pacotes dos projetos Hadem

## Status

In Progress — correções e validações executadas. Restam alertas upstream de braces e falhas de lint no código existente do painel administrativo; gate geral não aprovado.

executor: @dev
quality_gate: @qa
quality_gate_tools: [dependency_tree, npm_audit, lint, typecheck, test, build]

## Story

Como responsável pelos projetos Hadem, quero resolver os problemas de pacotes de todos os projetos da pasta Hadem, para instalar e validar suas dependências de forma consistente e compatível com o código existente.

## Contexto e fontes

- Pedido original: “Resolve todos os problemas de pacotes?”
- Escopo confirmado pelo usuário: “Todos os projetos da pasta Hadem”.
- Projetos com `package.json` identificados na raiz das aplicações: `front-end`, `back-end`, `hadem-admin`, `hadem-admin-api`, `hadem-kit` e `hadem-lp`.
- Esta manutenção não depende da conclusão das stories funcionais do catálogo e não altera seus requisitos.
- Regras de trabalho e evidência: `AGENTS.md`, `.aiox-core/constitution.md#iii-story-driven-development-must` e `.aiox-core/constitution.md#v-quality-first-must`.

## Acceptance Criteria

1. [ ] Os seis projetos têm diagnóstico registrado de manifestos, lockfiles, dependências instaladas, peer dependencies, engines e scripts disponíveis. Inconsistências encontradas são identificadas por projeto, sem tratar ausência de script como execução bem-sucedida.
2. [ ] Os problemas confirmados de resolução e compatibilidade de pacotes são corrigidos com manifestos e lockfiles coerentes. Priorizar versões compatíveis e mudanças mínimas; uma atualização major somente deve ocorrer quando necessária para resolver um problema confirmado, com justificativa e validação registradas. Não usar supressão de conflitos como comprovação de correção.
3. [ ] Executar `npm audit` por projeto, registrar o resultado inicial e final e corrigir vulnerabilidades com solução compatível disponível. Resultados sem correção compatível ou bloqueados por fatores externos são descritos com pacote/cadeia afetada, motivo e limitação restante; não declarar ausência de vulnerabilidades sem evidência.
4. [ ] Validar a árvore de dependências após as correções, registrando peers inválidos, pacotes ausentes ou extraneous que permaneçam e sua causa. Confirmar a coerência da instalação com o lockfile sem apagar alterações de código preexistentes.
5. [ ] Executar `npm run lint`, `npm run typecheck`, `npm test` e `npm run build` nos projetos que oferecem esses scripts. Registrar comandos, resultados, scripts ausentes e impedimentos por projeto, distinguindo falhas preexistentes das introduzidas pela correção de pacotes. Ajustes de código ou configuração devem ser limitados aos necessários à compatibilidade e validação desta correção.
6. [ ] Atualizar checklist, resultados de validação, limitações e File List antes de concluir, preservando alterações preexistentes do usuário e sem afirmar que todos os problemas foram resolvidos quando houver pendências.

## Tasks / Subtasks

- [x] Levantar baseline de versões Node/npm, manifestos, lockfiles, scripts e estado dos arquivos por projeto (AC: 1, 6).
  - [x] Registrar alterações preexistentes antes de modificar manifestos, configurações ou arquivos de código.
  - [x] Identificar dependências locais entre os projetos e restrições de engines/peers efetivamente declaradas.
- [x] Diagnosticar a árvore instalada e auditoria de cada projeto (AC: 1, 3).
  - [x] Comparar manifestos e lockfiles e executar inspeção de dependências instaladas.
  - [x] Executar `npm audit` e registrar problemas confirmados antes de escolher correções.
- [x] Corrigir dependências, lockfiles e incompatibilidades decorrentes das versões escolhidas (AC: 2, 4).
  - [x] Priorizar atualizações compatíveis e justificar mudanças major que forem necessárias.
  - [x] Revalidar árvore e auditoria após as alterações; documentar problemas sem solução compatível.
- [x] Executar os checks existentes em todos os projetos (AC: 5).
  - [x] Executar lint, typecheck, testes e build conforme scripts de cada projeto.
  - [x] Investigar falhas relacionadas a dependências; registrar falhas preexistentes, ausências e impedimentos.
  - [x] Usar testes existentes para verificar compatibilidade e acrescentar ou ajustar testes somente quando uma mudança de comportamento exigir cobertura útil.
- [x] Consolidar resultados, limitações, checklist e File List (AC: 6).

## Dev Notes

Não foi encontrada orientação específica de arquitetura sobre manutenção de dependências. Os caminhos de documentação geral configurados em `.aiox-core/core-config.yaml` (`docs/framework`, `docs/pt/framework` e `docs/es/framework`) não estão presentes nesta pasta. A única documentação encontrada em `docs/architecture` é específica do catálogo; esta story não prescreve mudanças na arquitetura nem novas tecnologias.

## Inventário inicial para implementação

Fonte: campos `scripts` dos seis `package.json` existentes, consultados em 2026-10-21. A presença do script não significa que seu check já passou.

| Projeto | lint | typecheck | test | build |
| --- | --- | --- | --- | --- |
| `front-end` | Existe | Existe | Existe | Existe |
| `back-end` | Existe; usa `--fix` | Existe | Existe | Existe |
| `hadem-admin` | Existe | Ausente | Ausente | Existe |
| `hadem-admin-api` | Ausente | Ausente | Existe | Existe |
| `hadem-kit` | Ausente | Existe | Ausente | Ausente |
| `hadem-lp` | Existe; chama `next lint` | Existe | Ausente | Existe |

Confirmar o inventário durante a implementação. O script de lint do `back-end` altera arquivos; qualquer alteração resultante precisa ser revisada em relação ao baseline. `front-end/package.json` declara Node `>=20.12.0`; as demais restrições devem ser verificadas nas dependências efetivas.

## Testing e resultados de implementação

A validação deve demonstrar resolução da árvore e coerência do lockfile, comparar auditoria antes/depois e executar os checks declarados. Usar os testes existentes dos projetos, sem serviços externos ou envios reais quando a suíte permitir isolamento. Acesso indisponível ao registro, requisitos de ambiente ou falhas preexistentes devem ter o comando e o impedimento registrados.

| Projeto | Árvore/lockfile | Audit inicial/final | Lint | Typecheck | Test | Build |
| --- | --- | --- | --- | --- | --- | --- |
| `front-end` | OK; ci simulado OK | 21 → 9; produção 0 | Passou; 69 warnings | Passou | 484 passaram | Passou |
| `back-end` | OK; ci simulado OK | 62 → 3; produção 3 | Passou; 141 warnings | Passou após ajustes | 2.883 passaram | Passou |
| `hadem-admin` | OK; ci simulado OK | 25 → 9; produção 0 | 16 erros / 4 warnings | Script adicionado; passou | Script ausente | Passou |
| `hadem-admin-api` | OK; ci simulado OK | 26 → 0; produção 0 | Script ausente | Script adicionado; passou | 122 passaram | Passou |
| `hadem-kit` | OK; ci simulado OK | 5 → 0; produção 0 | Script ausente | Passou | Script ausente | Script ausente |
| `hadem-lp` | OK; ci simulado OK | 6 → 0; produção 0 | Migrado; passou | Passou | Script ausente | Passou |

## CodeRabbit Integration

Habilitado via WSL em `.aiox-core/core-config.yaml#coderabbit_integration`.

- Tipo: Integration; secundário Security pela auditoria de pacotes. Complexidade alta pelo escopo de seis projetos.
- Agente principal: @dev. Revisão de qualidade: @qa. Consultar @architect se uma correção exigir decisão de arquitetura. @devops mantém autoridade exclusiva de push/PR, fora do trabalho solicitado nesta story.
- [x] Pre-Commit (@dev): disponibilidade verificada; WSL retorna `Wsl/Service/E_ACCESSDENIED`. Revisão não executada, conforme `graceful_degradation.skip_if_not_installed`.
- Self-healing de @dev: light, até duas iterações ou 15 minutos, correção de CRITICAL; HIGH documentado. Não há aprovação de qualidade presumida pela criação desta story.
- Focos: compatibilidade de dependências e peers, coerência dos lockfiles, vulnerabilidades corrigíveis, regressões dos checks existentes e preservação de alterações preexistentes.

## Validação do draft

Checklist `.aiox-core/product/checklists/story-draft-checklist.md` aplicado em modo autônomo: objetivo/contexto, orientação técnica, referências, autocontenção, testes e CodeRabbit com contexto suficiente para iniciar. As versões e vulnerabilidades específicas ficam condicionadas ao diagnóstico, sem inventar findings. Story de manutenção derivada diretamente do pedido, sem novo epic funcional ou sincronização externa.

## Dev Agent Record

### Implementação e evidências

- Modelo: Codex, persona Dex (@dev). Pedido do usuário e escopo confirmado autorizam a manutenção dos seis projetos; sem publicação ou alteração de arquitetura.
- Relatório detalhado: [manutenção de pacotes em 2026-10-21](../qa/package-maintenance-2026-10-21.md).
- Node 22.14.0 / npm 11.17.0. Baseline preservado: story 21.1 do front-end e memórias locais de agentes do backend; demais fontes estavam sem alterações. Documentação geral configurada em devLoadAlwaysFiles ausente, conforme diagnóstico de @sm.
- Auditoria total: 145 → 21 apontamentos; críticas 10 → 0. Restantes: um pacote upstream sem versão corrigida (`braces`), propagado em ferramentas de desenvolvimento dos dois painéis e no watcher do nestjs-i18n no backend. Detalhes e fontes no relatório.
- Manifestos/lockfiles coerentes, árvores completas sem `problems` e `npm ci --dry-run --ignore-scripts --no-audit` aceito em todos os projetos. A simulação não substitui instalação limpa de módulos nativos.
- Dependências utilizadas diretamente declaradas, stubs e tipos redundantes removidos, engine corrigida e configuração Prettier restaurada sem o preset incompatível. Lint da landing page refeito usando ESLint 10/TypeScript ESLint.
- Overrides específicos: YAML 3 → 4 somente para o loader Istanbul, com API consumida validada; YAML 5.5 para Swagger; Webpack 5.112 para satisfazer peers do CLI Nest. Nenhum downgrade destrutivo sugerido pelo audit foi aplicado.
- Ajustes de integração limitados à projeção tipada do Mongoose e normalização de erros unknown em canais RabbitMQ. Um caso de mapper e seis casos de eventos de erro adicionados; sem conexões reais de fila nos testes.
- Smoke checks de carregamento: bcrypt, isolated-vm, Sharp, Swagger, ExcelJS e Baileys; roundtrip XLSX, parser YAML e plugins Prettier funcionaram.
- Lint do painel administrativo falha em arquivos de código preexistentes; o plugin React Hooks já era 7.1.1 no baseline. Não houve execução completa desse lint antes das atualizações, portanto não há afirmação de comparação integral antes/depois. Falha não foi ocultada com desativação de regras.
- Permanecem avisos de depreciação e da política local npm allowScripts, listados no relatório. A política não foi ampliada automaticamente.

### DoD e pendências

- [x] Diagnóstico, correções compatíveis, testes úteis, árvore, engines e lockfiles verificados.
- [x] Pendências de segurança upstream e limitações locais registradas, sem afirmar auditoria zerada no workspace.
- [x] Tipos passam nos seis projetos, builds disponíveis passam e 3.489 testes passam. Checklist, File List e relatório atualizados; `git diff --check` sem erros.
- [ ] Todos os gates passam: lint administrativo falha com 16 erros / 4 warnings. Permanecem alertas upstream de braces e avisos de depreciação descritos no relatório.
- [ ] Story Ready for Review: condicionada à avaliação das pendências; sem aprovação de QA presumida.

## File List

- `docs/stories/fix.packages-workspace.story.md` — story e acompanhamento.
- `docs/qa/package-maintenance-2026-10-21.md` — auditoria, validação, decisões e limitações.
- `front-end/package.json` e `front-end/package-lock.json` — dependências, tipos e engine.
- `front-end/prettier.config.js` — configuração compatível e plugins explícitos.
- `back-end/package.json` e `back-end/package-lock.json` — dependências, overrides e scripts.
- `back-end/src/modules/conversation/infra/mongo/mapper/mongoose-team-mapper.ts` — contrato de documentos projetados.
- `back-end/src/shared/queue/rabbitmq/rabbitmq-consumer.adapter.ts` — normalização de erro unknown.
- `back-end/src/shared/queue/rabbitmq/rabbitmq.connection.ts` — normalização de erro unknown.
- `back-end/test/unit/modules/conversation/infra/mongo/mapper/mongoose-team-mapper.spec.ts` — projeção parcial.
- `back-end/test/unit/shared/queue/rabbitmq-error-logging.spec.ts` — seis casos de eventos de erro.
- `hadem-admin/package.json` e `hadem-admin/package-lock.json` — dependências e typecheck.
- `hadem-admin-api/package.json` e `hadem-admin-api/package-lock.json` — dependências, overrides e typecheck.
- `hadem-kit/package.json` e `hadem-kit/package-lock.json` — dependências.
- `hadem-lp/package.json` e `hadem-lp/package-lock.json` — dependências e lint.
- `hadem-lp/eslint.config.mjs` — regras recomendadas de TypeScript.

## Change Log

- 2026-10-21 — @sm: criada story Ready for Dev a partir do pedido explícito e do escopo confirmado para todos os projetos Hadem.
- 2026-10-21 — @dev: corrigidos pacotes, configurações e contratos afetados nos seis projetos; auditoria 145 → 21, críticas zeradas; validações e pendências registradas no relatório.
