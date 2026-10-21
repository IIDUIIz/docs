# Manutenção de pacotes — 2026-10-21

Escopo autorizado: os seis projetos principais da pasta Hadem. Cópias de deploy e worktrees existentes não foram alteradas. Ambiente: Node 22.14.0, npm 11.17.0, Windows x64.

## Auditoria

Contagens de pacotes apontados pelo `npm audit`, incluindo as cadeias transitivas; não representam CVEs distintos.

| Projeto | Inicial | Final | Críticas inicial → final | Final em produção |
| --- | ---: | ---: | ---: | ---: |
| front-end | 21 | 9 | 2 → 0 | 0 |
| back-end | 62 | 3 | 2 → 0 | 3 |
| hadem-admin | 25 | 9 | 2 → 0 | 0 |
| hadem-admin-api | 26 | 0 | 2 → 0 | 0 |
| hadem-kit | 5 | 0 | 1 → 0 | 0 |
| hadem-lp | 6 | 0 | 1 → 0 | 0 |
| **Total** | **145** | **21** | **10 → 0** | **3** |

Os 21 apontamentos finais são de severidade alta e derivam de `braces@3.0.3`, sem versão corrigida publicada nesta data. Não restaram apontamentos críticos, moderados ou baixos. O advisory declara ausência de correção: [GHSA-vfj7-8cjw-p6xm](https://github.com/advisories/GHSA-vfj7-8cjw-p6xm); o problema também consta no [repositório do pacote](https://github.com/micromatch/braces/issues/70).

- `front-end` e `hadem-admin`: nove apontamentos em cada árvore de desenvolvimento, pelas cadeias de `eslint-config-next` e `shadcn`/`ts-morph`/`fast-glob`/`micromatch`.
- `back-end`: três apontamentos na cadeia de produção `nestjs-i18n → chokidar → braces`. O `nestjs-i18n@10.8.5` continua exigindo `chokidar@^3.6.0`; forçar outra versão principal do watcher não foi usado como correção presumida.
- As sugestões de downgrade para `shadcn@1.0.0`, `eslint-config-next@14` e `nestjs-i18n@5.1.0` não foram aplicadas. Não foi usado `npm audit fix --force`.

## Alterações

- Atualização dos seis lockfiles dentro das faixas declaradas, com correções de Next.js, Axios, Sharp, Jest, Vitest, Multer, parsers e outras dependências transitivas. Next.js ficou em 16.4.2, Axios em 1.20.0 e Sharp em 0.35.5 nas árvores que os utilizam.
- Dependências importadas pelo código declaradas diretamente: CodeMirror View, utilitários do DnD Kit, React types, Express, JWT, Socket.IO, Multer, Boom, reflect-metadata e ferramentas usadas em scripts/testes.
- Removidos o pacote stub `fs` e os tipos redundantes `@types/react-window` e `@types/styled-components` do front-end. A CLI `shadcn` passou a dependência de desenvolvimento.
- Prettier voltou a carregar sem o preset ausente `@vercel/style-guide`. Foram preservados os defaults explícitos e `singleQuote` do preset, mais os plugins de ordenação de package.json e Tailwind. O preset publicado exige ESLint <9, TypeScript <6 e plugin Next <15, incompatíveis com o projeto atual. Referência dos defaults: [configuração oficial](https://github.com/vercel/style-guide/blob/canary/prettier/index.js).
- Engine do front-end corrigida para `^20.19.0 || >=22.13.0`, coerente com as dependências efetivas. Nenhuma incompatibilidade de engine com Node 22.14.0 na plataforma atual foi encontrada ao final.
- Overrides restritos corrigiram YAML e Webpack: `@istanbuljs/load-nyc-config` usa `js-yaml@4.3.2`, cuja API `load` utilizada pelo consumidor foi verificada; Swagger usa `js-yaml@5.5.0`; CLI Nest usa Webpack 5.112.0 para satisfazer os peers atuais de `schema-utils`.
- Scripts `typecheck` adicionados ao painel e à API administrativa. Lint do backend passou a somente verificar; autofix continua disponível em `lint:fix`.
- Landing page migrou de `next lint`, removido no Next 16, para ESLint 10 com regras recomendadas de TypeScript. Não foi incorporado o preset Next vulnerável. [Guia oficial da migração](https://nextjs.org/docs/app/guides/upgrading/version-16).
- Contrato de leitura do mapper de equipes passou a aceitar projeções parciais do Mongoose. Logs de erros de canal RabbitMQ agora normalizam valores `unknown`. Foram adicionados casos de regressão para projeções e erros `Error`, string e null.

## Validação

| Projeto | Lint | Tipos | Testes | Build |
| --- | --- | --- | --- | --- |
| front-end | Passou; 69 warnings | Passou | 65 arquivos / 484 testes passaram | Passou |
| back-end | Passou; 141 warnings | Passou após ajustes | 348 suítes / 2.883 testes passaram | Passou |
| hadem-admin | Falhou: 16 erros / 4 warnings | Passou | Script ausente | Passou |
| hadem-admin-api | Script ausente | Passou | 19 suítes / 122 testes passaram | Passou, inclusive após ajuste de Webpack |
| hadem-kit | Script ausente | Passou | Script ausente | Script ausente |
| hadem-lp | Passou | Passou | Script ausente | Passou |

- `npm ls --all --json`: sem `problems` nos seis projetos após alinhamento de peers e remoção de um pacote de tipos órfão.
- `npm ci --dry-run --ignore-scripts --no-audit`: passou nos seis projetos. Foi uma simulação; não equivale a reinstalação limpa com execução de scripts nativos.
- Manifestos e seções raiz dos lockfiles comparados por nomes e valores: sem divergências. Varredura de imports estáticos, `require` e imports dinâmicos literais de código/configurações/testes: sem dependências externas não declaradas ao final.
- Smoke checks: bcrypt, isolated-vm, Sharp, Swagger, ExcelJS e Baileys carregaram; ExcelJS escreveu e releu uma planilha em memória; parser YAML usado pelo Istanbul e plugins do Prettier funcionaram.
- Checagem inicial de tipos, antes das atualizações: passou nos seis projetos. Na API administrativa, a suíte inicial também passou com os mesmos 122 testes.
- Total final de testes aprovados nas três aplicações com suíte: 3.489. Os sete casos adicionados para a manutenção passaram na suíte completa do backend.

## Pendências e limites

- Os alertas de `braces` exigem correção upstream ou migração validada das ferramentas que o utilizam. Não foram ocultados por overrides incompatíveis.
- O lint do `hadem-admin` aponta quatro ocorrências de CommonJS em hooks locais e doze ocorrências de regras React/TypeScript em código existente. Os arquivos da aplicação não foram alterados nesta manutenção. O plugin `eslint-plugin-react-hooks` já era 7.1.1 no lockfile inicial; a execução de lint anterior à atualização não foi concluída, portanto não se afirma uma comparação integral antes/depois desse gate.
- Avisos de depreciação continuam em ESLint 9 nos dois apps com preset Next e em dependências do backend, incluindo OpenTelemetry trace-base, audio-decode, fluent-ffmpeg e prom-client; front-end também usa lodash.isequal. A migração desses pacotes exige revisão de APIs/consumidores. Os plugins React/import/a11y atuais declaram suporte até ESLint 9; forçar ESLint 10 nesses apps introduziria conflitos de peer.
- npm 11 informa scripts de instalação pendentes de sua política `allowScripts`. A política local não foi ampliada automaticamente. Os módulos nativos carregaram na instalação existente; uma instalação limpa com toolchain nativa não foi validada.
- CodeRabbit não foi executado: WSL retorna `Wsl/Service/E_ACCESSDENIED`. A configuração do projeto permite registrar indisponibilidade e seguir com checks locais.
- Alterações preexistentes na story 21.1 do front-end e nas memórias de agentes do backend foram preservadas. Artefatos gerados da landing page foram restaurados. Não houve commit, push, publicação de pacote ou deploy.

Story: [fix.packages-workspace.story.md](../stories/fix.packages-workspace.story.md).
