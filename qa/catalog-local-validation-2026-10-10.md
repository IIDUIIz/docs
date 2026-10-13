# Validação — catálogo local e action Mostrar produtos

Data: 2026-10-10. Escopo: stories 1.1 e 1.4. Revisão de código por @qa: PASS.

## Resultado dos gates

| Projeto | Verificação | Resultado |
|---|---|---|
| Backend | `npm run lint -- --no-fix` | 0 erros; 137 avisos existentes. Lint direcionado nos arquivos novos/alterados de catálogo e action: sem avisos. |
| Backend | `npm run typecheck` | Passou. |
| Backend | `npm test -- --runInBand` | 305 suítes, 2.364 testes passaram após congelar os arquivos. |
| Backend | `npm run build` | Passou. |
| Frontend | `npm run lint` | 0 erros; avisos existentes. Lint direcionado em catálogo, action e testes novos: sem avisos. |
| Frontend | `npm run typecheck` | Passou na versão final. |
| Frontend | `npm test` | 47 suítes, 352 testes passaram na versão final. |
| Frontend | `npm run build` | Passou; rota `/dashboard/settings/catalog` presente no resultado. |
| Frontend | `npm run check:i18n` | 3.895 chaves com paridade pt-BR/en-US. |
| Ambos | `git diff --check` | Passou. |

Scripts `typecheck` adicionados nos dois projetos; `test` no frontend executa a suíte Vitest existente. Nenhuma dependência adicionada.

## Cobertura específica

- Produtos sem fotos/descrição, normalização, preço inteiro e seleção válida/distinta (inclusive variação de maiúsculas em IDs).
- Consulta por identidade e busca sempre filtradas pelo tenant; busca escapa regex. Referências a fotos legadas sem tenant e fotos de outro tenant são rejeitadas.
- PATCH altera apenas campos explicitamente enviados; arquivamento preserva dados. Edição na interface não reativa produtos arquivados implicitamente.
- Interface com acesso somente de leitura, cadastro opcional de mídia, preço decimal e proteção contra resultados de busca antigos.
- Seleção e próximo passo preservados ao salvar/reabrir o grafo. Rascunho vazio é permitido; publicação exige seleção e próximo passo.
- Publicação e execução validam produtos ativos, existência e tenant antes de enviar. Fotos são validadas antes do primeiro envio.
- Imagens enviadas sequencialmente; legenda somente na primeira foto do produto; texto separado acima do limite de legenda. Falha parcial interrompe o lote sem repetir mensagens aceitas.
- Regressão completa de conversa, automação, permissões e demais módulos incluída nas suítes acima.

## Revisão e limitações

CodeRabbit foi verificado no WSL e não está instalado. Aplicado o fallback previsto em `.aiox-core/core-config.yaml` (`graceful_degradation.skip_if_not_installed: true`); realizada revisão @qa do código.

O build frontend inicialmente falhou ao baixar fontes Geist sob rede restrita; passou ao permitir acesso à rede. Uma execução de build backend sobreposta falhou ao limpar `dist`; o build final isolado passou. O fixture do teste de startup foi corrigido para simular `src/` em vez de `src/runtime/`, correspondendo à configuração real de i18n; configuração de runtime não alterada.

Não foi feito teste manual com banco/canal real nem envio a contatos. Os testes de saída exercitam os comandos/eventos do workflow com mocks; entrega real permanece sob os estados existentes das mensagens. Validação operacional deve ser feita em um fluxo e canal de teste.

## DoD

- [x] Requisitos e critérios do recorte local implementados e verificados pelos testes.
- [x] Estrutura Nest/CQRS, armazenamento, isolamento, permissões e erros existentes respeitados; nenhuma dependência ou segredo adicionado.
- [x] Testes unitários e de componentes, regressão, lint, typecheck, build e traduções verificados.
- [x] Documentação técnica, checklist, File List e notas de conclusão atualizados.
- [x] Revisão @qa concluída; CodeRabbit indisponível registrado.
- [ ] Verificação manual com sessão autenticada, banco e canal de teste — não executada neste ambiente.
- [N/A] Integração Meta, interface manual na conversa e produtos do funil — fora do recorte autorizado nesta entrega.

Resultado: implementação pronta para revisão; teste operacional real ainda pendente.
