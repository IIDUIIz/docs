# Noite de 30/10/2026 — perguntas para o owner (8h30)

Trabalho da madrugada: escopo de negócios por funil (story nova do back) e fase 1 da Pessoa
(back 30.1 a 30.9, front 24.1 a 24.6). Commit local, sem push.

Cada linha abaixo é uma dúvida que ficou para você. Entre parênteses, o que foi feito enquanto isso
(sempre a opção mais conservadora e fácil de desfazer).

## Decisões técnicas já tomadas (só para você saber)

- [30.2] A rotina diária de reconciliação nasce desligada (`PERSON_RECONCILIATION_ENABLED`) e só liga
  depois que a migração da 30.3 rodar com `--apply`; a auto-cura de contato antigo segue o mesmo
  interruptor. (Continua diária, como você decidiu; o interruptor só evita criar pessoa para a base
  inteira de uma vez.)

## Perguntas

- **RESPONDIDA (30/10, 10h40): sim, o responsável vê o seu.** Implementação nas stories
  `fix.crm-responsavel-ve-o-proprio-negocio` (back e front).
  [fix.crm-negocios-so-dos-funis-visiveis] O atendente que continua responsável por um negócio de funil que ele não vê (a equipe do funil mudou, ou o responsável veio de captura de conversa ou de reatribuição do gestor) deve continuar vendo esse negócio? (Enquanto isso: regra literal, só funil visível; o responsável sem acesso ao funil não vê o próprio negócio. Se quiser a exceção "o responsável sempre vê o seu", é um OU a mais na regra do card, sem mexer no filtro de funil.)
- **RESPONDIDA (31/10): sim, e rodou no dev.** 30.3: 137 pessoas, desfazer testado, critério de LID
  conferido (39). 30.4: 5 negócios ligados, `--verify` sem divergência. Números nas stories. Falta o
  owner ligar o `PERSON_RECONCILIATION_ENABLED` no Infisical do dev.
  [30.3] Posso rodar o roteiro do script de migração no Mongo de dev (dry-run, --apply, reexecução, --after e --rollback, com PERSON_RECONCILIATION_ENABLED desligado) e conferir o critério de LID com dado real? (feito enquanto isso: script pronto e roteiro inteiro validado só no container descartável hadem-test-mongo; nada rodou no Mongo de dev nem de prod; comandos em "Roteiro de execução" da story 30.3)
- **RESPONDIDA (31/10): rodou no dev** (ver acima).
  [30.4] Posso rodar no dev o backfill de cards (`npm run cards:backfill-person`: dry-run, `--apply`, reexecução e `--verify`) depois que a 30.3 for aplicada lá? (Até agora nada foi gravado no Mongo de dev nem no de prod. O roteiro rodou completo só no Mongo descartável `hadem-test-mongo`, e esse item da DoD da 30.4 continua aberto.)

- **RESPONDIDA (31/10): histórico não se grava à mão; a rota sai.** Story
  `fix.crm-historico-sem-evento-manual` (back e front).
  **(31/10) [fix.crm-evento-manual-sem-forjar] SEC-001:** quem grava evento à mão ainda consegue
  fazer uma linha da linha do tempo igual a "status alterado", "responsável alterado" ou "criado"
  (rótulo traduzido, letra de outro alfabeto ou caractere invisível). (1) A rota aceita só `note`;
  ou (2) o front marca o evento feito à mão e dá desenho próprio a esses três. (Enquanto isso: só os
  tipos de sistema são recusados.)
- **RESPONDIDA (31/10): sim, abrir story** (`fix.erro-validacao-com-codigo`, back; só abrir, sem implementar).
  **(31/10) Erro de validação:** hoje todo 400 de campo inválido sai com `errorCode` nulo, em todas
  as rotas, e o front mostra a mensagem genérica. Abrir story própria? (Enquanto isso: nada mudou.)
- **RESPONDIDA (31/10): troca de titularidade de número fica para depois** (doc de domínio, seção 0,
  item 13).
- **RESPONDIDA (31/10): os dois riscos de produto abaixo ficam aceitos** (doc de domínio, seção 0,
  item 14).

## Resultado da noite (conferido em 30/10 às 10h34)

- 16 stories Done: escopo por funil e 30.1 a 30.9 no back; 24.1 a 24.6 no front. O gate do @qa deu
  CONCERNS em todas, sem nenhum FAIL. A 30.1, a 30.3 e a 30.7 passaram por uma rodada de correção.
- Tudo commitado localmente, nada pushado:
  - back: 28 commits novos, de `2660dd5b` a `86088639` (42 à frente do origin, contando os de antes);
  - front: 13 commits novos, de `97899e4` a `79b41bf` (17 à frente do origin).
- Testes:
  - back: a suíte completa passa, menos a `executeScript.spec.ts`, que já falhava antes;
  - front: 94 arquivos e 936 testes verdes;
  - typecheck e lint limpos nos dois, só com os avisos que já existiam.
- Contrato front x back conferido rota por rota (13 rotas). Havia 3 divergências do lado do front,
  corrigidas em `79b41bf`: regra de e-mail, limites de tamanho dos campos e chave de idempotência da
  junção. Esse commit veio depois do gate da 24.2 e da 24.3 e não passou por gate novo.

## O que não foi feito ou não foi verificado

- Nada foi testado no app, porque os containers não pegam o código do host. É preciso rebuild do
  back e do front.
- O gate do @architect não está registrado em nenhuma story. O @qa cobriu os pontos de segurança, mas
  as `[AUTO-DECISION]` técnicas continuam esperando esse gate.
- A migração da 30.3 e o backfill da 30.4 só rodaram no Mongo descartável. O Mongo de dev e o de prod
  não foram tocados.
- O CodeRabbit não rodou (o CLI não está instalado no WSL). O `npm run generate:openapi` também não,
  porque precisa da infra real.
- As tabelas dos epics 30 e 24 ainda mostram Ready. Atualizar é papel do @po.

## Pontos técnicos médios para o gate do @architect (antes do primeiro deploy)

- **Índice:** trocar `{tenantId, personId}` por `{tenantId, personId, _id}` nos contatos, e fazer o
  equivalente nos cards. Hoje cada página de "sem pessoa" lê todos os pendentes do tenant. Decidir
  antes do dry-run em prod.
- **Card desatualizado:** um card criado ou editado no CRM durante uma junção, um desvínculo ou um
  desfazer pode ficar apontando para a pessoa errada. O `--verify` da 30.4 detecta o caso; corrigir
  exige update condicional no handler do card.
- **Apagar pessoa sem contato:** há corrida com um vínculo feito ao mesmo tempo (30.1, provada no
  Mongo real).
- **Rollback da 30.3:** o `--rollback` só desfaz o arquivo de relatório passado. Uma retomada ou um
  apply por tenant gera outro arquivo.
- **Busca de pessoas:** a busca `q` lê o tenant inteiro quando há poucos resultados.

## Para o deploy (quando for)

- Ordem: o back antes do front. A 24.3 e a 24.4 sobem juntas. A 24.6 só sobe depois da 30.5.
- Rodar o sweep de system roles, que dá `people:manage` e `people:merge` a Dono e Gerente.
- O Mongo de prod precisa suportar transação, porque a junção (30.7) usa.
- Sequência: `--apply` da 30.3, depois o `cards:backfill-person` da 30.4, e só então ligar o
  `PERSON_RECONCILIATION_ENABLED`. A imagem de prod não tem `scripts/` nem ts-node, então falta
  definir onde rodar esses scripts.

## Riscos de produto (para você saber)

- O nome da pessoa aparece sem máscara no card, na lista e no detalhe. Se alguém digitar um telefone
  como nome, o telefone fica visível. O risco foi aceito nas stories e a decisão técnica é do @architect.
- A busca parcial pelo valor real (ADR-006 D7) deixa um atendente sem permissão confirmar, por
  tentativa, um telefone ou e-mail oculto. A busca de contato do CRM já funcionava assim.
