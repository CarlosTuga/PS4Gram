# PS4Gram
Telegram para PS4
# PS4Gram — objetivo, trabalho realizado e estado do projeto

Atualizado em 15 de novembro de 2026. Versão do núcleo: **v0.3.0**.

## Objetivo

Criar uma aplicação homebrew para PS4 que permita navegar em grupos e canais Telegram com o comando da consola, descarregar ficheiros e lançar payloads individuais BIN/ELF. O alvo inicial é a PS4 do utilizador, com firmware 12.00 e GoldHEN. Esta combinação ainda não foi validada pelo projeto.

A utilização nativa na consola é o objetivo principal. Um eventual PC ou NAS poderá ajudar em tarefas exigentes como funcionalidade opcional futura; não deve ser obrigatório para o fluxo principal. Essa integração auxiliar não está implementada.

A v0.3 segue o comportamento de payloads descrito pelo utilizador para o KingNext: cada payload pertence ao seu jogo e já inclui o servidor, o URL e a lógica para iniciar a descarga. Não foi analisado nem reutilizado código KingNext nesta tarefa, pelo que não se afirma equivalência interna entre as aplicações.

## Funcionamento obrigatório dos payloads

1. O adaptador Telegram autenticado obtém os anexos das fontes autorizadas e fornece os seus identificadores reais.
2. A aplicação apresenta a lista e permite navegar com o comando.
3. Uma pressão de X sobre um payload autoriza o seu download e lançamento. Não existe segunda confirmação.
4. O PS4Gram descarrega apenas esse BIN/ELF para memória temporária, com limite de tamanho.
5. Valida silenciosamente a origem, a integridade e o formato. Quando configurado um hash confiável para o anexo, exige correspondência exata.
6. Envia os bytes sem alterações ao loader configurado. A disponibilidade da ligação é verificada no próprio envio.
7. Liberta o buffer após o envio síncrono ou após falha. Não pergunta por HDD/USB nem grava o payload numa USB.
8. Apresenta o resultado real conhecido: falha, recusa ou envio completo. O protocolo TCP raw não confirma execução.
9. Quando executado pela consola, o payload é responsável por contactar o servidor incorporado e iniciar o download do seu jogo. Esse comportamento externo ainda não foi observado nestes testes.

Não existe payload universal, substituição de URLs incorporados nem download do jogo efetuado pelo fluxo de payloads do PS4Gram. As preferências de armazenamento dos ficheiros comuns não controlam o destino escolhido internamente por um payload.

## Funcionamento dos ficheiros PKG, ZIP e RAR

Estes ficheiros têm um fluxo independente: selecionar o ficheiro, escolher HDD interno ou USB e pasta predefinida ou personalizada, descarregar e gerir o resultado. A preferência pode ser guardada para utilizações futuras.

| Situação | Política |
|---|---|
| Download simples concluído | Conservar o ficheiro descarregado |
| Processamento no HDD interno | Eliminar o original apenas após sucesso verificado |
| Instalação de PKG | Exigir confirmação fiável de instalação antes da eliminação |
| Processamento de original na USB | Conservar por defeito; permitir opção de eliminar após conclusão verificada |
| Falha, cancelamento ou dúvida | Conservar o original |

O núcleo disponibiliza os contratos e a política. Instaladores e extratores reais ainda têm de executar e verificar as operações; um parâmetro de teste não representa prova de instalação na consola.

## Base utilizada e correções preservadas

O ZIP v0.2 fornecido continha uma árvore v0.1 e uma árvore mais recente em `PS4Gram/PS4Gram-v02`. Foi usada exclusivamente esta última como base.

Foram preservados sem alterações os componentes de armazenamento, SHA-256, DownloadManager, política de originais e plataforma POSIX da v0.2, bem como os testes e documentos históricos correspondentes. Entre os comportamentos conservados estão a proteção por descritores contra trocas de symlinks, validação de raízes, deteção de substituição de volumes, retoma com validadores HTTP, metadados de transferências parciais, verificação opcional de hash, pausa/cancelamento, controlo de destinos e limpeza pendente após instalação.

A comparação da árvore está em [base-audit-v0.3.md](base-audit-v0.3.md). Não foi removido nenhum ficheiro da árvore v0.2 selecionada.

## Trabalho implementado na v0.3

| Componente | O que faz |
|---|---|
| `AuthorizedPayloadCatalog` | Restringe o catálogo a chats configurados; associa hashes confiáveis a chat, mensagem e anexo |
| `PayloadExecutionFlow` | Orquestra seleção com X, download integral em memória, verificações, revalidação de origem e envio |
| `PayloadBrowserController` | Fornece lista, seleção, navegação e ativação por transição de X; impede repetições por manter o botão premido |
| `PayloadManager::sendBytes` | Envia o buffer verificado sem o gravar nem alterar |
| Validação ELF | Verifica cabeçalho, tabela e limites de segmentos; não certifica ABI ou firmware |
| Notificações | Distinguem envio de execução; TCP raw nunca é apresentado como confirmação de execução |
| `FileDownloadFlow` | Trata PKG/ZIP/RAR, volumes, pastas, preferências e integração com a política de processamento |
| Testes e documentação | Cobrem os novos fluxos, regressões, integração e limitações de compatibilidade |

As configurações dos dois fluxos são independentes. O fluxo de payloads não depende de StorageManager, destinos de ficheiros ou política USB. Não há reenvio automático de payloads após erro ou envio parcial, pois isso poderia executar o mesmo payload mais de uma vez.

## Confiança e integridade

A configuração local contém os chats autorizados. O adaptador Telegram ainda por implementar deve comprovar a origem original e fornecer IDs estáveis; nomes visíveis, mensagens encaminhadas e legendas não constituem prova de autorização.

Por defeito, o lançamento exige SHA-256 confiável associado ao anexo exato. Um hash copiado de uma legenda não autenticada não autentica o payload. É possível permitir localmente fontes autorizadas sem hash, mantendo as restantes verificações. Não há uma confirmação adicional no botão X.

A verificação de assinaturas de manifestos não está implementada. O catálogo pode receber hashes de um manifesto verificado por outro componente confiável, mas o núcleo não deve ser descrito como tendo verificado essa assinatura.

## Testes realmente executados

Testes locais em Linux com g++ 14.2.0, C++17 e pthread:

| Suite | ASan + UBSan | Release |
|---|---:|---:|
| storage | 50/50 | 50/50 |
| downloads | 42/42 | 42/42 |
| policy | 70/70 | 70/70 |
| payloads | 25/25 | 25/25 |
| flows | 101/101 | 101/101 |
| Total | **288/288** | **288/288** |

O script final terminou com código 0. LeakSanitizer foi desativado devido à incompatibilidade do ambiente com ptrace; não se afirma teste de fugas. ASan e UBSan permaneceram ativos. CMake/CTest não foram executados porque CMake não está instalado neste ambiente.

As suites de fluxo usam adaptadores simulados. Os testes TCP reais utilizam sockets de loopback num PC, não uma PS4 ou GoldHEN. O registo e as limitações encontram-se em [test-report-v0.3.md](test-report-v0.3.md) e [test-output-v0.3.log](test-output-v0.3.log).

## Estado atual e trabalho ainda necessário

**A entrega atual é uma biblioteca C++17 testável em PC. Não é uma aplicação PS4 instalável. Não foi compilada nem executada na consola e não inclui um PKG.**

Falta implementar e integrar:

- Telegram TDLib/MTProto: autenticação, conversas, catálogo e download autenticado de anexos.
- Transporte HTTP/HTTPS/TLS de produção, com validação de certificados e prazos de rede.
- Interface gráfica PS4, leitura do comando e apresentação nativa de notificações, ligadas aos controladores existentes.
- Adaptadores OpenOrbis de rede, threads, volumes, logging e chamadas de armazenamento.
- Loader ELF identificado e validado, com protocolo compatível. Não se assume suporte ELF na porta BIN 9090.
- Instalador PKG e extratores ZIP/RAR seguros, com provas reais de processamento concluído.
- Empacotamento e testes de hardware, incluindo caminhos e permissões de HDD/USB.
- Eventual integração opcional PC/NAS, quando o funcionamento nativo principal estiver definido.

O README oficial do GoldHEN documenta o BinLoader na porta 9090. Isso não comprova compatibilidade deste projeto com todos os payloads, ELF ou com a combinação específica instalada pelo utilizador. Ver [compatibility-v0.3.md](compatibility-v0.3.md).

## Próximos passos e critérios de conclusão

1. Publicar e manter o núcleo, os testes e os relatórios no repositório do utilizador, preservando o histórico.
2. Implementar os adaptadores Telegram e PS4 e ligá-los aos dois fluxos existentes.
3. Compilar com o SDK real, produzir a aplicação e confirmar os requisitos de rede/armazenamento na consola.
4. Validar um payload BIN conhecido: uma pressão de X deve originar um download de anexo e um envio, sem confirmação extra ou acesso à USB. Confirmar a execução na consola e o início da descarga no servidor do próprio payload.
5. Validar ELF separadamente e testar os casos de falha, hash divergente e envio interrompido, sem execução ou retry indevidos.
6. Validar ficheiros comuns, preferências entre reinícios e conservação/eliminação após processamento real verificado.

Só depois desses testes se poderá declarar funcionamento na PS4. O avanço do núcleo e os testes de PC não substituem essa validação.

## Entregas e publicação

Foi criado o pacote `PS4Gram-v0.3.zip` com código-fonte, testes, registo de testes, auditoria da base e relatórios de integração e compatibilidade. Este relatório geral foi acrescentado depois da criação desse pacote e deve acompanhar a evolução do repositório.

Repositório de destino: [CarlosTuga/PS4Gram](https://github.com/CarlosTuga/PS4Gram).

Na tentativa inicial de publicação, a ligação GitHub permitiu leitura mas recusou operações de escrita com erro 403, «Resource not accessible by integration». Esse erro não representa uma publicação concluída. A existência de código no repositório deve ser confirmada pelo commit remoto, e não apenas pela preparação local.

A tentativa alternativa de push por Git também não conseguiu autenticar: não havia credenciais GitHub disponíveis neste ambiente. O projeto e este relatório ficaram preparados em commits locais, sem confirmação de publicação remota.
