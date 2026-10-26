# Roteiro de verificação no app dev: Prorrogar com data de retorno e follow-up no motivo

Para quem executa: o dono. Ambiente: **dev** (nunca produção). App: http://ethalia.lvh.me:3000

Cobre as stories **back 29.3** e **front 23.3** (commits locais `14903eee` e `fbe8e89f` no back, `fceec56` no front). O roteiro anterior (`roteiro-verificacao-tarefa-no-lugar-de-atividade.md`) cobriu 29.1, 29.2, 23.1 e 23.2; este cobre o que veio depois.

Ao terminar, me diga **o número de cada passo que falhou** e o que viu. Passos sem falha não precisam de relato.

## Preparo

1. Reconstrua as imagens com o código atual do disco (na raiz `C:\Users\dudus\Hadem`):

   ```
   docker compose up -d --build backend frontend
   ```

   Atenção: o back **e** o front precisam subir juntos. Com o back novo e o front antigo, o Prorrogar do kanban responde 400.
2. Confira o boot do back:

   ```
   docker compose logs backend --since 5m
   ```

3. Dados: o funil de teste (etapas a, b, c) com **3 cards abertos**: um com dono e contato (por exemplo "Duchat comercial"), outro também com dono, e um terceiro para o motivo "Perder". Um usuário gestor e um atendente sem `tasks_manage`.
4. Cadastre em **Gestão do funil > Motivos** (passo C abaixo): ao menos um motivo de **Prorrogado**, um de **Perdido** e um de **Encerrado**.

## Parte A: Prorrogar com data (kanban)

- **A1.** Arraste um card para a zona **Prorrogar**. Esperado: abre **sempre** um modal, com o botão Confirmar desabilitado até haver data.
- **A2.** Os atalhos **Amanhã**, **Em 3 dias** e **Próx. semana** preenchem a data às 9h. Editar a data à mão desmarca o atalho. Uma data no passado mostra erro em texto e mantém Confirmar desabilitado.
- **A3.** Motivo: se o funil **não** exige motivo para Prorrogado, o seletor é opcional. Ligue "exigir motivo" em Prorrogado na gestão e repita: o motivo passa a ser obrigatório. Sem motivos cadastrados, só a data aparece.
- **A4.** Confirme com uma data futura (use "Em 3 dias"). Esperado:
  - o card some do quadro e aparece o aviso de confirmação;
  - em `/dashboard/tasks` há **uma tarefa nova**: título "Entrar em contato com <contato>", tipo **Tarefa**, responsável = dono do card, **prazo = a data escolhida**;
  - anote como a tela mostra origem e contexto (continua sem mostrar origem nem o card, como na rodada anterior; só me diga se mudou).
- **A5.** Erro do back: derrube o back por alguns segundos (ou use um card sem permissão) e confirme. Esperado: toast "Falha ao encerrar card", **modal continua aberto** e dá para tentar de novo, sem erro solto no console.
- **A6.** **Ganhar**, **Perder** e **Encerrar** continuam como antes: sem modal quando o motivo não é exigido, com o modal de motivo quando é.

## Parte B: Rede (opcional, com as ferramentas do navegador)

- **B1.** No Network, o PATCH do Prorrogar leva `postponedUntil` terminando em **`Z`**. Nenhum POST de card leva `postponedUntil`.
- **B2.** Reabrir um card prorrogado não é possível pela tela (limitação conhecida, story futura). Se você reabrir por API (`PATCH {status:"open"}`), `postponedUntil` volta a `null`.

## Parte C: Follow-up no motivo (gestão do funil)

- **C1.** Gestão do funil, aba **Motivos**. O interruptor "Follow-up" aparece em **Encerrado** e **Perdido**. **Ganho** não tem o controle. **Prorrogado** mostra só uma linha explicando que a data é escolhida pelo atendente.
- **C2.** Ligue o follow-up em um motivo de **Perdido**: tipo **Ligação**, prazo **2 dias**, nota "retomar contato". Salve. Esperado: recarregar a tela mantém os valores.
- **C3.** Validações: prazo **0** bloqueia o salvar com mensagem na barra; prazo **acima do teto** (mais de 365 dias ou mais de 8760 horas) também bloqueia.
- **C4.** Desligue o follow-up e salve. No Network, o PATCH leva `followUp: {"enabled": false}`. Ligar e desligar de volta sem salvar não gera PATCH.
- **C5.** Perca um card (**Perder**) escolhendo o motivo que tem follow-up. Esperado: **uma tarefa nova** em `/dashboard/tasks`: tipo Ligação, prazo aproximadamente **agora + 2 dias**, descrição "retomar contato", responsável = dono.
- **C6.** Perca outro card com um motivo **sem** follow-up. Esperado: **nenhuma** tarefa nova.
- **C7.** Filtrar motivos por tipo não perde o que você editou e não salvou.
- **C8.** Modal legado (paleta de comandos > Configurar funil): renomeie um motivo que tem follow-up e confira que o **follow-up permanece**.

## Parte D: Layout e idioma

- **D1.** A linha de motivo ocupa duas linhas, com recuo só a partir de largura `sm`. Confira em tela estreita (celular) que nada quebra.
- **D2.** Mude para en-US e repita A1 e C1. Textos traduzidos.

## Parte E: Rejeições do back (só se você usar API/ferramenta de requisição; opcional)

Com token de dev:

1. `PATCH` do card com `{status:"postponed"}` **sem** data → 400 (data obrigatória).
2. `postponedUntil` `"2026-11-01T10:00"` ou `"2026-11-01"` → 400 (precisa de fuso).
3. Data passada com `Z` → 400.
4. Reenviar **a mesma** data de um card já prorrogado → 200, **sem tarefa nova**.
5. `POST` de card com `postponedUntil` → 400.
6. Motivo `won` ou `postponed` com `followUp.enabled: true` → 400; `followUp: null` → 400.

## O que fazer com o resultado

- Tudo passou: me diga "tudo ok". Eu fecho o gate da 23.3 e preparo o handoff para o @devops (push e deploy conjunto na ordem 23.1 → 29.1 → 29.2 → 23.2 → 29.3 + 23.3).
- Algo falhou: me passe o número do passo e o que apareceu (print ou texto). Eu abro o ajuste com o @dev antes de seguir.
