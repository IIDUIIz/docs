# Módulo de catálogo — arquitetura proposta

Status: proposta de arquitetura; implementação ainda não iniciada.

## Contrato aprovado para a entrega local — 2026-10-10

Revisão @architect: aprovado o recorte das stories 1.1 e 1.4, catálogo local e action `showProducts`. Meta, envio manual na tela de conversa e itens no funil permanecem fora desta implementação. As decisões amplas abaixo descrevem entregas futuras; não bloqueiam esta fundação.

- `CatalogModule` segue Nest/CQRS, com comandos de criação/atualização/inativação e queries de consulta/listagem/lookup. Controllers derivam o tenant do contexto autenticado; handlers e repositório exigem tenant explícito em todas as operações. Nunca consultar por identificador global e depois atualizar sem tenant.
- Produto possui identidade gerada pelo servidor, `name`, `description` opcional, SKU estável, `priceMinor`, `currency`, `active` e `photoFileIds` ordenado. Dinheiro usa inteiro seguro não negativo; moeda usa código ISO com três letras maiúsculas. Sem estoque, categorias, variações ou exclusão definitiva. Arquivar significa inativar e preservar a identidade.
- Limites técnicos locais: nome até 200 caracteres, descrição até 1.000, SKU até 100, até 10 fotos distintas por produto; seleção de 1 a 20 produtos distintos por action. São limites de payload desta implementação, não limites oficiais de catálogo Meta. O formato de apresentação usa nome, preço/moeda e descrição quando presente; manter caption compatível com as validações de mensagens existentes, rejeitando payload incompatível antes de enviar.
- Fotos reutilizam `StorageModule` e `FileStorageFacade`, sem novo storage nem reupload durante envio. Validar existência, `file.tenantId === tenantId` e MIME de imagem permitido pelo upload existente (JPEG/PNG/WebP), na gravação e antes do envio. A facade aceita arquivos legados sem tenant; esse comportamento não autoriza associá-los ao catálogo. URLs recebidas do cliente não substituem referências de arquivo validadas.
- Permissões `catalog:read`, `catalog:manage` e `catalog:lookup` seguem o registry existente. A capacidade `gestao-catalogo` administra cadastro/consulta; capacidades de edição de fluxo recebem somente lookup para seleção. Endpoint lookup expõe produtos ativos com identidade/nome e dados necessários ao seletor, sem oferecer mutações nem contornar tenant. Rota `/dashboard/settings/catalog` integra as configurações com o mesmo controle de acesso.
- Action `showProducts` recebe `productIds: string[]` e `nextAction` pelo contrato existente. Normalizador, DTO/validação de publicação, executor, registry, tipos e node do editor devem concordar. O backend resolve a seleção integralmente no tenant da instância e verifica todos os produtos ativos e todas as fotos antes do primeiro envio. Produto ausente/inativo ou referência inválida falha a action; não omitir itens silenciosamente.
- O handler envia na ordem da seleção e das fotos. Produto sem foto usa o comando de mensagem textual existente. Produto com fotos usa `SendFileMessageCommand` por imagem, `saveFile:false`, referência `/files/:id/download`, MIME e filename; caption nativa aparece somente na primeira imagem do produto. Não duplicar legenda como mensagem separada nem reenviar buffers para storage. Nesta entrega, várias fotos usam sequência; não prometer álbum nativo.
- Autor `ActorType.Automation`, canal/conversa/contato e tenant vêm do contexto confiável já usado pelas actions. Envio percorre os comandos/workflow existentes, preservando persistência, restrições do provider e despacho assíncrono. `ActionCompletedEvent` indica aceitação no workflow, não garantia de entrega no provider. Falha publica `ActionFailedEvent` com correlation/context/nextAction conforme `file-action`; não repetir automaticamente seleção já parcialmente aceita.
- Testes relevantes: isolamento entre tenants (inclusive fotos sem tenant), preço inteiro, inativação, seleção inválida antes de qualquer envio, produto sem foto, descrição ausente, múltiplos produtos/imagens em ordem e caption somente na primeira foto, falha parcial sem conclusão de sucesso.

Evidências verificadas: `back-end/src/shared/storage/application/file.service.ts`, `shared/storage/domain/file-record.ts`, `modules/automation/application/commands/action/file-action/file-action.handler.ts` e `modules/conversation/application/commands/message/send-file-message/send-file-message.command.ts`. Os documentos convencionais `docs/architecture/coding-standards.md`, `tech-stack.md` e `source-tree.md` não existem; padrões locais acima foram extraídos dos módulos existentes. A modelagem física/índices continua sob responsabilidade @data-engineer; esta aprovação fixa fronteiras, contratos e segurança da integração.

## Requisitos confirmados

- Cadastro de produtos pelo Hadem em `/dashboard/settings/`.
- Produtos compartilhados entre conversas, automações e funil.
- Cadastro e sincronização de produtos na Meta pelo Hadem.
- Uso do catálogo nativo WhatsApp quando o canal oficial permitir.
- Produtos vinculados ao negócio com quantidade e preço.
- `card.value` corresponde à soma dos itens do negócio.
- Opção de sincronização com a Meta dentro das configurações de cada canal oficial elegível; o catálogo do sistema permanece independente dessa opção.
- Reutilizar vínculo e autorização existentes quando válidos; solicitar login/OAuth na interface somente se faltar autorização de gerenciamento do catálogo.
- A ação de mostrar produtos permite selecionar um único produto ou vários; enviar somente a seleção.
- Envio nativo quando canal, vínculo e produto sincronizado permitirem; sem vínculo utilizável, usar fotos quando existentes e descrição quando cadastrada. Várias fotos usam álbum quando suportado ou sequência de imagens. Sem foto, apresentar somente texto com nome e descrição, se disponível; descrição ausente não bloqueia envio.

## Evidências do repositório

- `front-end/app/dashboard/settings/components/settings-shell.tsx`: grupo comercial atualmente contém funis; a navegação depende de autorização de rota.
- `front-end/app/dashboard/support/conversation/`: interface de conversas.
- `back-end/src/modules/conversation/application/commands/message/send-message/send-message.handler.ts`: resolve destino no servidor e usa `OutgoingMessageWorkflowService.saveAndDispatch`.
- `back-end/src/modules/conversation/application/providers/port/messaging-provider.port.ts`: somente texto, mídia, URL de mídia e templates.
- `back-end/src/modules/conversation/application/providers/registry/provider-capability-set.ts` e `default-provider-registry.ts`: registro por capacidades, sem catálogo.
- `back-end/src/modules/conversation/application/providers/dialog360/dialog360-messaging.provider.ts` e `infra/providers/360dialog/services/dialog360-outbound.service.ts`: implementação oficial atual via 360dialog, sem envio de produtos.
- `back-end/src/modules/conversation/infra/providers/360dialog/360dialog.instance.ts`: engine `business360`; endpoints Legacy e Cloud distintos.
- `back-end/src/modules/conversation/infra/providers/360dialog/types/dialog360-creds.types.ts`: credenciais limitadas a `apiKey`, `baseUrl`, `tenantId`, `isCloud`.
- `back-end/src/modules/conversation/infra/mongo/entities/meta-channel-binding.schema.ts`: vínculo exclusivamente Instagram; token cifrado e `select:false`.
- `back-end/src/modules/conversation/infra/providers/360dialog/normalizer/dialog360-message-normalizer.service.ts`: respostas interativas normalizadas como texto; sem tratamento explícito de `order` ou `referred_product`.
- `back-end/src/modules/automation/domain/normalizers/action-definition.ts`, `application/registries/action-execution.registry.ts` e `application/registries/executors/`: pontos de extensão de ações.
- `front-end/app/dashboard/automation/flow/`: editor de automações.
- `back-end/src/modules/pipelines/infra/mongo/entities/card.schema.ts`: negócio possui `value:number`, sem itens de produtos.
- `back-end/src/modules/pipelines/presentation/dtos/card.dto.ts`, `presentation/controllers/card.controller.ts`, `application/commands/card/update-card.handler.ts`: contrato e atualização do negócio.
- `front-end/app/dashboard/crm/components/card-detail/` e `hooks/use-card-detail.ts`: detalhe do negócio.

## Decisões propostas

1. Criar `back-end/src/modules/catalog/` seguindo Nest/CQRS e separação domain/application/infra/presentation existentes. O catálogo centraliza produtos e sincronização; todos os acessos são restritos pelo tenant. Disponibilizar operação por API/CLI antes da interface, conforme Constitution.
2. Separar gerenciamento Commerce Meta de envio de mensagens WhatsApp. A chave 360dialog não deve ser tratada como autorização Commerce. Criar vínculo de integração próprio com segredo cifrado; não reutilizar o vínculo Instagram como se já suportasse comércio WhatsApp. A opção fica no canal: verificar vínculo/permissões existentes e pedir autorização adicional apenas quando necessário. Sincronização é opcional para uso do catálogo local.
3. Persistir primeiro a alteração local e executar sincronização assíncrona com o mecanismo de fila existente. Manter identificador externo estável, estado de sincronização, resultado sanitizado e retry. Garantir idempotência e impedir versões antigas de sobrescreverem alterações recentes. Produto local permanece disponível em falha de sincronização; envio nativo requer mapeamento externo confirmado.
4. Adicionar capacidade específica de mensagens de catálogo ao registry, inicialmente implementada para 360dialog Cloud. Conversa e automação consomem o mesmo caso de uso e o workflow persistente de envio. Credenciais e destino são resolvidos no servidor. Escolher envio nativo apenas com vínculo utilizável e produto sincronizado; caso contrário apresentar produto por imagem/descrição ou texto sem foto. Não presumir suporte universal a álbum: usar sequência quando o provedor não oferecer agrupamento. Erros de envio nativo não devem causar reenvio automático por mídia sem certeza de que o primeiro envio falhou, evitando duplicidade; respeitar as restrições do canal também no envio alternativo.
5. Criar ação de automação dedicada, incluindo tipos, normalização, DTOs, validação de publicação, executor e node frontend. Seguir padrões existentes de próxima ação e fallback.
6. Manter itens de negócio sob responsabilidade de pipelines, referenciando produto e preservando snapshot comercial de nome, preço, moeda e quantidade. Alterações posteriores no catálogo não reprecificam negócios silenciosamente. A modelagem física e os índices ficam para revisão de @data-engineer.
7. Apresentar estado de sincronização e erros úteis sem expor credenciais. Respostas recebidas de catálogo precisam de representação coerente; converter pedido automaticamente em negócio não é requisito confirmado.

## Decisões ainda pendentes

- Campos mínimos do produto e requisitos Commerce: imagem, URL, descrição, moeda, disponibilidade e outros devem ser definidos a partir do contrato oficial verificado.
- Suporte e limites de múltiplas fotos/álbum por provedor, incluindo persistência e falhas parciais no envio em sequência.
- Catálogo Meta será criado pela integração ou vinculado a catálogo existente; sincronizar produtos não implica autorização para provisionar contas e ativos.
- Arquivamento/exclusão, alterações em referências usadas por automações publicadas e histórico de negócios precisam de critérios de aceitação explícitos.

## Evidência externa e limites

Documentação oficial 360dialog confirma mensagens `interactive.product` e `interactive.product_list` com `catalog_id` e `product_retailer_id`, além de webhook de pedido com quantidade, preço e moeda:
https://docs.360dialog.com/docs/messaging/catalogs/single-and-multi-product-messages

A investigação principal verificou também janela de atendimento de 24 horas e limite de 30 produtos para mensagens interativas. Esses limites devem ser validados no caminho de envio, sem presumir que mensagens de catálogo e templates tenham as mesmas regras.

Os contratos de criação/atualização de produtos Commerce Meta, permissões, credenciais e vínculo catálogo/WABA não foram verificados integralmente: a documentação Meta retornou HTTP 429 na investigação principal. Não fixar endpoints, payloads ou scopes com base neste documento. A story de sincronização depende dessa verificação oficial antes da implementação.

## Sequência sugerida

Catálogo local e autorização → vínculo Commerce e sincronização verificada → envio em conversas → ação de automação → itens no negócio. Stories devem registrar critérios, checklist e arquivos; implementação deve executar os gates definidos no projeto.

## Fundação local proposta para refinamento

Contrato mínimo proposto, derivado do cadastro reutilizável e do preço confirmado: identificador gerado pelo servidor, `tenantId` derivado da autenticação, nome obrigatório, descrição opcional, SKU estável, preço não negativo em unidade mínima inteira (`priceMinor`), moeda explícita (`currency`) e estado ativo/inativo. Imagem e URL devem ser opcionais no cadastro local e validadas como requisitos da sincronização somente após verificar o contrato Meta. Não incluir estoque, descontos, variações ou categorias sem requisito específico.

Operações iniciais: criar, listar com paginação e busca, consultar por identificador, atualizar e arquivar. Arquivamento preserva referências de negócios e automações; detalhes da exclusão externa permanecem pendentes. SKU deve permanecer estável após sincronização. Confirmar unicidade e índices com @data-engineer.

Dinheiro deve ser calculado em unidades mínimas inteiras; evitar multiplicação de valores monetários em ponto flutuante. Para compatibilidade, converter o total ao contrato legado `card.value` somente na fronteira. Negócio usa uma moeda; misturas precisam ser rejeitadas até haver requisito de conversão. A granularidade de quantidade deve ser definida antes de validar itens.

Autorização proposta: capacidades separadas para consultar produtos e gerenciar catálogo/integração. Usar guards e contexto de tenant existentes; cadastrar a rota de settings no registry de acesso. A seleção por atendentes e automações não deve depender de permissão administrativa de cadastro. Os nomes concretos das permissões precisam seguir o registry existente, sem criar bypass por UI.

Fonte adicional pesquisada pelo agente principal: SDK oficial Meta `ProductItem` lista campos como `name`, `description`, `price`, `currency`, `image_url`, `url`, `retailer_id` e `availability`: https://raw.githubusercontent.com/facebook/facebook-nodejs-business-sdk/main/src/objects/product-item.js. A presença dos campos no SDK não confirma quais são obrigatórios para criar itens, permissões OAuth ou o contrato completo de sincronização.

## Evolução: coleção dinâmica e limpeza de fotos

Inspeção da implementação local: produtos usam `photoFileIds`; mensagens persistem referências `/files/:id/download` em `value`; ações de arquivo usam `fileId`, `fileUrl` e `files[]` em `definition.actions[].definition`. `FileStorageFacade.delete` atualmente admite arquivos sem tenant por seu helper de leitura. O provider S3 já implementa `DeleteObjectCommand` e resolve configuração pelo tenant do arquivo, compatível com armazenamento R2 configurado; não criar cliente Cloudflare paralelo.

Decisão de arquitetura de coleção dinâmica: recurso tenant-scoped único como categoria/grupo nomeado com IDs ordenados dos produtos membros; action referencia `collectionId` em alternativa exclusiva a `productIds`. No momento da execução, carregar membros atuais, preservar ordem e aplicar o limite vigente de envio antes de começar. Coleção vazia/removida ou qualquer membro inativo falha de forma explícita antes do primeiro envio, preservando o contrato fail-fast de seleção existente. Não truncar silenciosamente. A composição pode mudar após publicação do fluxo, tornando a referência dinâmica; edição do fluxo não é necessária para refletir membros. Não criar seleção automática por preço ou outras regras sem requisito.

Limpeza pequena e conservadora proposta:

1. Persistir primeiro a remoção das fotos ou exclusão do produto; coletar somente IDs removidos de fato (diferença entre estado anterior e atual). Falha na persistência não inicia limpeza.
2. Para cada candidato, exigir arquivo existente e `file.tenantId === tenantId`, imagem permitida e referência de origem conhecida. Arquivo tenantless nunca entra na limpeza automática, mesmo que seja legível por APIs existentes.
3. Consultar referências de outros produtos (ativos e inativos), mensagens históricas (incluindo deleted) e todos os fluxos (incluindo inativos e rascunhos). Verificar IDs diretos e URLs conhecidas: `/files/:id/download`, `/files/:id/view` e URL do descriptor. Preservar arquivo ao encontrar qualquer referência ou ao falhar qualquer consulta. Arrays `files[]` também precisam ser inspecionados.
4. Sem referências, chamar exclusivamente facade/provider existente para apagar objeto e depois descriptor. Erro do storage mantém descriptor e gera log sanitizado/retry; a edição do produto já persistida não deve ser revertida nem reportada como falha total.

Uma consulta de referências seguida de delete não garante segurança contra vínculo concorrente. Para promessa estrita de preservação, as operações que criam referência e a limpeza precisam compartilhar exclusão mútua por tenant/fileId, ou protocolo de marcação/claim verificado por todos os escritores. Não afirmar atomicidade global com checagem simples. A opção menor sem mudança transversal é limpeza conservadora best-effort de candidatos conhecidos, preservando arquivos ambíguos; a implementação deve declarar esse limite. Arquivos com referências arbitrárias em outros módulos ou URLs assinadas não normalizadas são ambíguos e devem ser preservados até existir inventário de referências completo.

Novos endpoints de remoção/exclusão devem usar `catalog:manage`, tenant derivado do servidor e nunca aceitar path/bucket/URL como alvo de delete. Arquivamento `active:false` sozinho não remove fotos, pois o produto pode ser reativado. Exclusão física de produto e comportamento dos fluxos que o referenciam precisam de regra explícita; históricos de mensagens continuam preservados.

### Alternativa recomendada para garantia de preservação

Não depender de scans globais para autorizar delete. A menor limpeza garantida é descartar upload novo de catálogo ainda não vinculado e comprovadamente pertencente ao tenant; arquivos legados ou sem rastreamento de ownership devem permanecer. Para limpar também fotos salvas com garantia, adicionar tracking explícito de lifecycle de arquivo, revisado por @data-engineer: ownership exclusivo de catálogo, vínculo ao produto e pin permanente ao primeiro uso externo ao catálogo. Duplicação de produto deve compartilhar ownership registrado ou marcar arquivo como compartilhado. Salvar mensagem ou ação de arquivo com URL/ID deve adquirir pin antes de persistir a referência. Cleanup só apaga arquivo exclusivo, desvinculado e sem pin, por claim atômico; nenhum writer pode aceitar arquivo já claimed/deleting. Esse protocolo exige cobertura de todos os writers que aceitam arquivo de catálogo; se um writer não for coberto, não considerar arquivo exclusivo.

Arquivos legados não recebem exclusividade retroativa por ausência de referências em consultas. A remoção da foto do produto pode concluir e retornar estado de preservação da mídia compartilhada. Pin permanente é deliberadamente conservador: evita introduzir gerenciamento de expiração de histórico neste escopo.

Copiar arquivo independentemente para cada mensagem futura permitiria limpeza da cópia do catálogo sem apagar essas mensagens, mas duplica armazenamento, não resolve referências históricas ou fileURLs de flows e ainda demanda protocolo no envio concorrente. Não usar cópia como única garantia.

Escopo atual de limpeza: remoção de foto salva e descarte de upload órfão; não acrescentar exclusão física de produto apenas para executar limpeza.

### Protocolo concreto aprovado pelo architect

- Novos uploads exclusivos usam diretório `catalog-images`; uploads genéricos e arquivos antigos em `image` não são candidatos. Prefixo sozinho não comprova ausência de compartilhamento: exige protocolo abaixo.
- Descriptor recebe ownership de produtos, pin permanente (`catalogRetained`) e claim (`catalogDeleting`), com modelagem/indexação revisadas por @data-engineer. `acquireCatalogOwner` adiciona owner atomicamente apenas se arquivo pertence ao tenant e não está deleting. Criar/atualizar produto adquire owners antes de salvar; falha na gravação pode preservar owner conservadoramente. Remover foto libera owner somente depois de salvar produto. Arquivamento não libera owners.
- `retain` marca pin atomicamente apenas se arquivo não está deleting. Writers abortam referência quando retain/acquire falha. Não fazer pin em `get` genérico, porque validação e edição de catálogo também usam leitura.
- `cleanupIfUnretained` faz claim atômico condicionado a tenant exato, prefixo exclusivo, owners vazio, retained falso e deleting falso. Após claim, facade usa provider existente para apagar objeto e depois descriptor. Falha mantém claim para retry e impede novas associações; leitura comum deve ocultar deleting. Consulta posterior de referências não substitui claim.
- Integrar pin no caminho genérico `conversation/application/services/message/message-content-preprocessor.service.ts`, antes de persistência/queue, para referências de arquivo por ID e URLs locais conhecidas. A action `showProducts` deve passar pelo mesmo caminho. URLs públicas conhecidas precisam ser resolvidas ao descriptor; formatos que não puderem ser identificados exigem preservação conservadora.
- Integrar pin nas gravações de flows: `automation/application/commands/flow/create-flow/create-flow.handler.ts`, `update-flow/update-flow.handler.ts` e `build-flow/build-flow.handler.ts`; inspecionar `fileId`, `fileUrl` e `files[]`. Se a publicação persistir referências não passadas nesses caminhos, proteger também `publish-flow/publish-flow.handler.ts`. Incluir execução legada de `action/file-action/file-action.handler.ts` como proteção antes do envio.
- Qualquer outro writer que aceite referência a `catalog-images` precisa passar por retain/acquire; até cobertura completa, não habilitar limpeza desses candidatos. Referências legadas e tenantless permanecem preservadas.
- Descarte explícito de upload órfão usa o mesmo claim (sem owners/pin), nunca limpeza automática ao desmontar componente. Remoção de foto salva devolve sucesso de edição mesmo se mídia for preservada por compartilhamento; erro de cleanup é observável e retryable.

Uma alternativa menor é pin permanente ao compartilhar foto entre dois produtos em vez de owners completos, mas o produto único ainda precisa ter owner registrado para fechar a corrida de associação contra descarte. Esse tradeoff reduz limpeza futura e deve ser explícito.

## Exclusão física de produto autorizada posteriormente

O usuário autorizou exclusão física de produto. Endpoint `DELETE /settings/catalog/products/:id` usa `catalog:manage` e tenant do servidor. Retorna 404 para produto inexistente e 409 com código `CATALOG_PRODUCT_IN_USE` quando coleção ou fluxo com seleção direta referencia o produto. Não alterar coleções/flows automaticamente nem excluir suas referências. Considerar todos os fluxos persistidos, inclusive inativos e rascunhos; seleção dinâmica é protegida pela associação na coleção.

Usar adapter de referências no módulo catálogo, injetando `Connection` Mongo, para consultar coleção e `flows` sem import circular de AutomationModule. Nome de coleção deve vir dos schemas/modelos conhecidos, nunca de entrada do cliente. Buscar tenant e `productIds` somente nos campos efetivos da ação `showProducts`; evitar busca textual no JSON e falso bloqueio por metadados arbitrários. A coleção nomeada deve usar seu contrato real de membership. Repository de produtos permanece responsável pelo delete tenant-scoped.

Persistir exclusão do produto antes de liberar ownership de suas fotos; cleanup segue o claim protegido já descrito. Histórico de mensagens fica intacto, e fotos pinned permanecem. Se houver itens de negócios que dependam do identificador, preservar snapshot e revisar requisito antes de remover referência; não criar cascade comercial implícita.

Limite de concorrência: check-before-delete não garante integridade contra criação simultânea de coleção/flow. Nesta entrega, o bloqueio protege referências já persistidas; criação/atualização de coleções e publicação de flows devem validar a existência dos produtos para reduzir inconsistência. Não declarar garantia transacional global. Garantia estrita exigiria protocolo compartilhado de tombstone/reference acquisition ou transação entre todos os writers, com escopo e testes próprios. Esse limite não reduz a proteção atômica exigida para arquivos.

### Revisão para detecção de concorrência sem transações Mongo

Claim atômico `product.deleting` antes da consulta de referências; get/list/seleção ocultam produtos claimed. Se referências forem encontradas, restaurar claim e retornar 409; caso contrário concluir exclusão. Writers de coleção e fluxo validam referências antes e depois de persistir. Assim, referência gravada antes do claim aparece na consulta do deleter; referência gravada depois encontra claim/ausência na validação posterior e deve reportar erro, nunca sucesso.

Validação posterior deve desfazer gravação inválida via compare-and-swap da revision gravada (restaurar estado anterior em update ou remover create), sem sobrescrever edição concorrente. Se CAS impedir rollback, preservar estado explicitamente inválido/não-publicável e retornar conflito; não deixar gravação inválida aparentar publicação válida. Canonicalizar IDs novos para lowercase e reconhecer IDs legados com caixa diferente nas consultas. Esta revisão substitui o simples check-before-delete para os writers cobertos; não afirmar proteção para writer externo que ignore o protocolo.

Retry de liberação de ownership precisa de intent durável: registrar pendência antes do commit de alteração do produto; worker somente libera owner quando o produto já não referencia a foto (ou produto foi excluído). Produto ainda referenciando foto mantém owner. Depois do commit, executar liberação e claim do arquivo; falha continua rastreada para retry. Registrar pendência apenas depois do commit deixa janela em que falha de escrita perde o retry, portanto não basta para garantia de cleanup eventual.

### Contrato final: owners por revision substituem rollback

Adotar reference ownership atômico em produtos para novos writers, em substituição à proposta anterior de validar depois e restaurar gravação. Cada revision salva de coleção/flow recebe token idempotente. Antes do save, `CatalogService.acquireReferences` adquire esse token em todos os produtos referenciados, condicionado a tenant e `deleting != true`. Falha de aquisição impede save. Release parcial pode ser conservador; aquisição com erro incerto conserva tokens até reconciliação, nunca considera ausência de owner confirmada.

Coleção mantém `referenceToken`; flow mantém privados `catalogReferenceToken` e `catalogReferencedProductIds`, sem aceitar esses campos do cliente. Flow repository integra acquire/release pelo CatalogService exportado. Depois de commit confirmado, liberar token da revision anterior. Resultado de escrita incerto preserva ambos tokens. Update concorrente deve usar revision/CAS para não perder tracking; release remove somente token específico, nunca todos os owners de um produto.

Deleter faz claim atômico somente quando `referenceOwners` está vazio; novos writers já não conseguem adquirir após claim. Consultar também referências legadas sem tokens, normalizando IDs/case; se houver, restaurar claim e retornar 409. Esse fallback protege legado existente, e novos writers obrigatoriamente usam protocolo. Coleção/flow removido libera seu token somente após commit confirmado.

Cleanup de fotos usa journal durável `CatalogCleanupJob` gravado antes de alterar/excluir produto, contendo productId, photoIds candidatos e ownerToken da revision anterior. Processador/retry explícito compara token atual do produto: se ainda igual ao antigo, não libera owner (alteração não confirmou); se mudou ou produto foi excluído, libera somente token antigo e tenta claim de arquivo. Fotos mantidas na nova revision já possuem novo owner. Endpoint explícito de retry processa journal sem exigir cron nesta entrega. Journal só é concluído após liberação/cleanup ou preservação conhecida por pin; falhas mantêm pendência observável. Não interpretar apenas produto existente como prova de que o owner antigo deve permanecer: a identidade da revision é o critério.

## Envio manual e vídeos — evolução autorizada

Criar `CatalogProductSenderService` no CatalogModule, usando CatalogSelectionService, FileStorageFacade e CommandBus, sem importar ConversationModule. O serviço resolve seleção, valida todos os produtos/mídias e retém referências antes do primeiro envio; envia pelo workflow existente via comandos de conversa. Importar classes de comando não equivale a importar o módulo Nest e não cria dependência circular de módulos. AutomationModule e ConversationModule consomem o serviço exportado; ShowProductsHandler conserva responsabilidade pelos eventos de conclusão/falha e delega planejamento/envio.

Endpoint proposto `POST /mensagens/send-products` no módulo conversation: body contém somente conversationId e alternativa exclusiva productIds/collectionId. Usa autorização existente `conversation_attend` e `catalog_lookup`; usuário/actor/tenant vêm da borda autenticada. Resolver destino por ConversationAccessService/resolveClientOutgoingTarget, contato e canal no servidor; engine deve vir da configuração do canal. Ignorar/rejeitar tenantId, contato, canal, engine e actor enviados pelo cliente. Manter semântica existente de conversa encerrada (envio abre/acha conversa pelo contato/canal), preservando a autorização da conversa original.

Entrada do sender é destino/actor já confiável, sem pressupor usuário administrativo; automação passa actor Automation, API passa actor User. Usar SendMessageCommand para o envio manual com requester trusted, ou um adapter de dispatch equivalente que preserve a mesma autorização/workflow. Nunca usar actor Automation no envio manual. Preflight inválido não envia nenhuma mensagem e usa erros HTTP normais. Dispatch retorna resultado estruturado com status accepted/partial/failed, acceptedCount e IDs das mensagens aceitas; falha durante sequência inclui código/mensagem sanitizada e IDs/count acumulados. Accepted significa persistência/aceitação pelo workflow, não entrega ao destinatário. Não fazer retry automático da sequência nem UI repetir envio após resultado partial.

Modelo mínimo de mídia: `mediaFileIds:string[]` é sequência ordenada autoritativa de imagens/vídeos; tipo vem do descriptor, nunca do cliente. `photoFileIds` continua projeção compatível somente das imagens. Produtos legados sem mediaFileIds usam photoFileIds como sequência. Novos clientes escrevem mediaFileIds; body com ambos campos precisa ser rejeitado quando inconsistente. PATCH legado photoFileIds em produto sem vídeos mantém compatibilidade; em produto com vídeos deve rejeitar com erro explícito e exigir mediaFileIds, evitando apagar/reordenar vídeos silenciosamente. Limite total inicialmente preserva o cap existente de dez mídias, sem inventar limite adicional por tipo.

Upload de vídeo no catálogo usa UploadService.compressVideo existente (MP4/H.264, máximo de entrada 20MB e downscale máximo 720p). Inspeção confirmou que maxHeight é parâmetro e limite 20MB pertence ao caminho existente de actions; a nova rota deve aplicar seu próprio limite multipart e passar 720 explicitamente. Estender opção de propósito/diretório do compressor para armazenar em `catalog-videos` e marcar catalogManaged; não presumir que chamadas atuais fazem essa marcação. Aceitar no produto vídeo MP4 já convertido e imagens JPEG/PNG/WebP do mesmo tenant. Cleanup/ownership/pin/journal passam a cobrir ambos prefixos com tipo/MIME apropriado e tenant exato, preservando legacy.

Plano de envio mantém ordem de mídias. Primeiro item recebe texto do produto como legenda quando couber no limite existente de 1024 caracteres; texto maior vai em mensagem própria antes das mídias; produto sem mídia usa texto. Descriptor de vídeo gera content.type='video', imagem gera 'image'; não enviar vídeo como imagem ou documento genericamente. Preview frontend usa video para vídeo e mantém projeção photoFileIds para consumidores antigos. Essa evolução não implica anexar vídeos ao catálogo nativo Meta, cujo contrato permanece uma integração distinta.
