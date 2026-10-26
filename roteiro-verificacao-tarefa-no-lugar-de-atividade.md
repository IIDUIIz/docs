# Roteiro de verificação no app dev: Tarefa no lugar de Atividade

Para quem executa: o dono. Ambiente: **dev** (nunca produção). App: http://ethalia.lvh.me:3000

Cobre o que nenhum agente consegue verificar sozinho:
- **Parte A:** pré-condição PC-R1 da 29.3 (back 29.2 + front 23.2, ponta a ponta).
- **Parte B:** verificação visual da 23.1 (remoção de Atividade).
- **Parte C:** verificação visual da 23.2 (editor "Criar tarefa").

Ao terminar, me diga **o número de cada passo que falhou** e o que você viu. Passos sem falha não precisam de relato.

## Preparo

1. Reconstrua as imagens com o código atual do disco (na raiz `C:\Users\dudus\Hadem`):

   ```
   docker compose up -d --build backend frontend
   ```

   O compose watch não roda nesta máquina e `docker cp` não é necessário: o rebuild leva o código das stories 29.1, 29.2, 23.1 e 23.2.
2. Espere subir e confira que o back não tem erro de boot:

   ```
   docker compose logs backend --since 5m
   ```

3. Dados de teste (no funil de teste, com a etapa **A** e a etapa **X**):
   - Card **(a)**: com dono e contato chamado "Maria".
   - Card **(b)**: com dono e contato cujo nome é um telefone, por exemplo "+55 11 99999-0000".
   - Card **(c)**: sem dono (opcional: se a tela não deixar criar card sem dono, pule os passos marcados com (c); esse caso está coberto por teste automatizado).
   - Dois usuários: um **gestor** com `tasks_manage` e um **atendente sem** `tasks_manage`.
4. Antes de qualquer teste, anote (não precisa me passar agora): quantas atividades pendentes e quantas regras `create_activity` existem em **produção**. É a condição de deploy da 29.1, e eu não consulto produção.

## Parte A: Criar tarefa pela regra de etapa (29.2 + 23.2)

Entrar como **gestor**.

- **A1.** Gestão do funil (`/dashboard/settings/pipelines/<id>`), aba Etapas, etapa **X**, "Nova regra".
  - Escolha a ação **Criar tarefa**. Esperado: o gatilho fica em "Ao entrar na etapa" e é a única opção.
  - Sem tipo e sem prazo, o botão de salvar fica desabilitado.
  - Preencha: tipo **Ligação**, prazo **2 horas**, nota "teste 29.2" e salve. Esperado: salva sem erro e a regra aparece como "Criar tarefa · Ligação · 2 horas". O lápis reabre com tudo preenchido.
- **A2.** No kanban, arraste o card **(a)** para a etapa **X**. Esperado: o card move sem erro.
- **A3.** Abra `/dashboard/tasks`. Esperado, para a tarefa do card (a):
  - Título **"Entrar em contato com Maria"**, tipo **Ligação**.
  - Responsável = dono do card, prazo aproximadamente **agora + 2 horas**, descrição "teste 29.2".
  - Contexto = o card, e origem **automation** visível.
- **A4.** Arraste o card **(b)** (nome em formato de telefone) para **X**. Esperado: tarefa com título **"Entrar em contato"**, sem o telefone no título.
- **A5.** (c) Arraste o card sem dono para **X**. Esperado: tarefa criada **sem responsável**.
- **A6.** Abra o detalhe de cada tarefa criada. Esperado: a tela não quebra com autor ausente nem com timeline vazia. Anote como a tela exibe origem e contexto (é a dúvida em aberto do R1).
- **A7.** Entrar como **atendente sem `tasks_manage`**. Esperado: vê as tarefas dele e **não** vê a tarefa sem responsável (comportamento aceito, Q1).
- **A8.** Idempotência: mova o card (a) de **X** para outra etapa e de volta para **X**. Esperado: **uma** tarefa nova na reentrada e nenhuma duplicada.
- **A9.** Logs do back:

  ```
  docker compose logs backend --since 15m
  ```

  Procure por `StageRule`, `create_task`, `CommandHandlerNotFound` e `Falha ao criar tarefa`. Esperado: **sem erro**; um aviso (`warn`) para o card sem dono.

## Parte B: Remoção de Atividade (23.1)

- **B1.** `/dashboard/crm`, abra um card. Esperado: botão **Timeline**, sem bolinha de atividade, sem aba nem seção de Atividades. Eventos antigos de atividade aparecem como "Atividade criada/concluída/pulada".
- **B2.** Entrar como atendente **sem `crm:monitor`**. Esperado: o botão da Timeline não aparece, o card não quebra e o diálogo fica estreito.
- **B3.** Atendimento, aba **Negócio**. Esperado: **sem** "retorno agendado". Etapa, temperatura, Perder, Ganhar, Encerrar e Transferir funcionam.
- **B4.** Kanban: selo de dias parados e filtro de parados iguais ao que eram.
- **B5.** Teclado: foco no botão Timeline funciona; a largura do diálogo é coerente ao abrir e fechar o painel.

## Parte C: Editor de regras (23.2)

- **C1.** Gestão do funil, "Nova regra": a lista de ações tem **4** opções (avisar dono, mover, perder e Criar tarefa). **Não** há "Criar atividade".
- **C2.** Troque a ação para "Criar tarefa" com o gatilho "Sem atividade há" ou "Ao mudar o status" selecionado. Esperado: o gatilho volta para "Ao entrar na etapa".
- **C3.** Modal legado (paleta de comandos > Configurar funil > Etapas > engrenagem): a regra `create_task` aparece listada, **sem lápis**, com interruptor e lixeira. "Salvar regras" não a altera.
- **C4.** Se existir regra legada `create_activity` salva: aparece como **"Ação descontinuada"**, sem lápis, removível, e salvar outra alteração não dá erro.
- **C5.** Mude o idioma para en-US e repita C1 e B1. Esperado: textos traduzidos ("Timeline", "Discontinued action"). Os rótulos "Ligação" e "Tarefa" não traduzem (limitação conhecida).
- **C6.** Largura estreita da janela: os campos de prazo, unidade e nota não quebram o layout.

## O que fazer com o resultado

- Tudo passou: me diga "tudo ok" e eu libero a 29.3 (pré-condição PC-R1 cumprida) e registro nas stories.
- Algo falhou: me passe o número do passo e o que apareceu (print ou texto do erro). Eu abro o ajuste com o @dev antes de seguir.
