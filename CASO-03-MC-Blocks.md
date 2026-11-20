# MC-Blocks — Biblioteca de famílias para Revit, em substituição a ferramenta comercial

**C# · .NET 8 e .NET Framework 4.8 · WPF · Revit API 2023/2025/2026**
Desenvolvido durante atuação na VIAVOZ LTDA (engenharia e arquitetura) · 2026

---

## O problema

A equipe de arquitetura usava o **Blocks**, um plugin comercial que dá acesso a uma biblioteca de famílias dentro do Revit. Ele parou de funcionar:

> *"O plugin simplesmente não está funcionando mais no Revit de muitas pessoas. Ou ele buga, ou ele simplesmente **some da aba de ferramentas**."*

Esse sintoma — a aba desaparecer — viria a ser importante mais tarde.

A empresa já tinha a própria biblioteca de famílias num servidor compartilhado. A pergunta foi: **dá para fazer um plugin nosso, que leia a nossa biblioteca, e não dependa de licença de ninguém?**

---

## A escala real

O projeto começou apontando para uma pasta de teste com **91 famílias**. Quando passou para o servidor de produção, o número era outro:

| | |
|---|---|
| Famílias (`.rfa`) | **32.384** |
| Volume só de `.rfa` | **~13 GB** |
| Volume com pastas completas | ~28 GB |
| Disciplinas | 12 (`01 - ARQUITETURA` a `12 - LUMINOTÉCNICO`) |
| Grupos de nomes de arquivo repetidos | **5.198** |
| Backups do Revit a ignorar (`nome.0001.rfa`) | 1.216 |

Essa diferença de escala — de 91 para 32 mil — forçou decisões que não existiam no protótipo: árvore de pastas em vez de lista, limite de 200 cards por tela, miniaturas carregadas sob demanda e cache local das famílias.

---

## A descoberta mais interessante

Durante uma auditoria do próprio código, encontrei o seguinte: **se a pasta do servidor estivesse fora do ar, ou se qualquer subpasta desse erro, a varredura lançava exceção durante a inicialização do Revit.**

Quando um add-in lança exceção ao iniciar, o Revit não o carrega — e **a aba dele não aparece na faixa de opções**.

Que é exatamente o sintoma relatado do plugin antigo.

Não tenho o código do Blocks e não posso afirmar que a causa era a mesma. Mas a coincidência foi suficiente para tratar isso como requisito de projeto, e não como detalhe:

- A varredura passou a rodar **em segundo plano**, nunca durante a inicialização
- Pastas inacessíveis são **ignoradas individualmente**, não derrubam a varredura inteira
- O registro do plugin tem proteção própria: uma falha ao montar o painel **não impede o plugin de carregar**
- O rodapé do painel mostra o que aconteceu, em vez de falhar em silêncio

**A lição:** o modo de falha mais perigoso não é o que dá erro na tela. É o que faz a ferramenta sumir sem explicação — porque o usuário não tem o que reportar, e quem mantém não tem o que investigar.

---

## Arquitetura: três versões do Revit, um código-fonte

O Revit mudou de runtime no meio do caminho:

| Versão do Revit | Runtime |
|---|---|
| 2023 | **.NET Framework 4.8** |
| 2025 e 2026 | **.NET 8** |

Não existe DLL única que rode nas duas linhas. A solução foi **multi-targeting**: o mesmo código-fonte compila em três saídas, e o instalador detecta a versão do Revit presente na máquina e copia a DLL correspondente.

Isso trouxe armadilhas reais:

- **`Path.GetRelativePath` não existe no .NET Framework 4.8.** Quebrava só o build do 2023 — que era justamente o que eu não tinha instalado para testar.
- **`ElementId.IntegerValue` foi removido no Revit 2026.** Compilava no 2025 e falhava no 2026.

Ambos só apareceram porque passei a compilar as três versões a cada mudança, em vez de confiar na que estava na minha máquina.

---

## Bugs que custaram caro, e o que ensinaram

### O objeto que flutuava — onde eu errei duas vezes

Famílias inseridas apareciam acima do piso. A lixeira do exemplo ficou a 15 unidades do chão.

**Primeira explicação, errada:** afirmei que o valor vinha gravado na própria família, herdado de quando ela foi criada.

**Primeira correção, também errada:** zerei o parâmetro "Elevação do nível" — que é **calculado e somente leitura**. A correção não fazia absolutamente nada.

**O que era de verdade:** o parâmetro editável é o "Deslocamento do hospedeiro". E o valor de 15 não vinha da família — vinha do **clique do usuário**: o diagnóstico mostrou Z = 0,49 pés, que são aproximadamente 15 cm.

Só descobri porque instrumentei o código com uma caixa de diagnóstico temporária, mostrando nível e coordenadas a cada inserção, e pedi os valores reais em vez de continuar deduzindo.

**A lição:** duas hipóteses plausíveis seguidas, ambas erradas, custaram mais tempo que medir uma vez.

### A API que travava o Revit

Para posicionar famílias com a precisão nativa — nível, hospedeiro, orientação, prévia junto ao mouse — o caminho certo é delegar ao próprio Revit em vez de reimplementar a lógica.

A primeira tentativa **congelou o Revit**: a chamada escolhida bloqueia até a inserção terminar, e não convive com um painel *modeless*.

Reverti para o fluxo manual, que era pior mas estável. Depois encontrei a chamada correta — disponível desde 2015, e que não bloqueia — e aí sim portas, janelas e objetos passaram a ser posicionados pelo próprio Revit, com prévia no cursor.

**A lição:** reverter rápido para o que funciona vale mais que insistir na solução elegante. E "funciona pior, mas não trava" é um estado aceitável enquanto se investiga.

### Os que vieram do ambiente, não do código

- **Encoding.** O `settings.json` gravado via PowerShell corrompia acentos (`Í` virava `?`), e o caminho não batia com a pasta real. Resolvido escrevendo o arquivo direto, sem passar pelo shell.
- **Conta de domínio.** A máquina usava `viavoz\suporte`, e o perfil `%AppData%` não era o esperado. A configuração foi movida para `C:\ProgramData` — local fixo, independente de quem está logado. Melhor também para uso em equipe.
- **DLL bloqueada.** Um `.zip` recebido por e-mail ou rede deixa a DLL marcada como bloqueada pelo Windows, e o Revit se recusa a carregá-la. O instalador passou a desbloquear automaticamente.

---

## Decisões de produto

### Apontar para o servidor, não para cópias locais

A recomendação foi ler direto do caminho de rede, e não distribuir cópias.

**Motivo:** com cópia local, toda família nova precisa ser redistribuída manualmente a todo mundo — *exatamente o tipo de problema que causou a dor do plugin antigo*. Com caminho de rede, atualiza-se uma vez e todos veem.

### Busca que funciona como as pessoas pensam

A busca original casava a string literal. `"janela correr"` retornava **7 resultados** numa biblioteca com milhares de janelas.

Reescrita para buscar por palavras, ignorando acentos, `_` e `-`: os mesmos termos passaram a retornar **196**.

### Miniatura tirada do próprio arquivo

Em vez de exigir uma imagem externa ao lado de cada `.rfa`, o plugin extrai a **miniatura nativa do arquivo** — a mesma que o Explorador do Windows mostra. Funciona para qualquer família, sem preparação.

### Hospedeiro por tipo de família

Famílias do Revit têm regras de inserção diferentes, e tratá-las igual era a origem de boa parte dos defeitos:

| Tipo | Como insere |
|---|---|
| Porta, janela | clique na **parede** define hospedeiro e posição |
| Portão, gradil (*work-plane based*) | clique na **face** |
| Mesa, lixeira, escorregador | clique no **piso ou terreno**; nível vem dali |
| Carimbo | **só em prancha** — com aviso claro, em português |
| Símbolo de anotação (2D) | **só em vista 2D**, nunca em 3D |

---

## Onde eu disse não

A empresa tem uma parceira no mesmo prédio, sem acesso ao servidor. A solução era copiar a biblioteca para o servidor deles e apontar o plugin para lá.

**Não copiei nada.** Deixei tudo pronto — pacote de instalação configurado e script de cópia — mas com três ressalvas registradas:

1. **Gravar 13 GB no servidor de outra empresa exige autorização das duas partes.** Não é decisão técnica.
2. **A biblioteca contém famílias de fabricantes e material interno.** Sugeri verificar com a direção se esse conteúdo pode sair da empresa.
3. **Sugeri compartilhar apenas parte**, começando por `01 - ARQUITETURA`, em vez de espelhar o servidor inteiro.

O script foi escrito com **modo de simulação obrigatório antes da cópia real**, exige digitar "SIM" para executar, copia apenas `.rfa`, e **nunca apaga nada no destino**.

Da minha parte, o servidor da empresa foi **sempre lido, nunca escrito** — em todas as etapas do projeto.

---

## O que foi entregue

- **Plugin instalado na equipe de arquitetura**, substituindo a ferramenta comercial
- **Segundo pacote para a empresa parceira**, apontando para o servidor deles, com script de sincronização
- **Instalador de um clique** (`INSTALAR.bat`): detecta a versão do Revit, recusa rodar com o Revit aberto, desbloqueia a DLL, preserva a configuração existente e dispensa desinstalação para atualizar
- **Botão "Verificar Biblioteca"**: testa o carregamento das famílias da pasta selecionada e gera relatório com as que falham e as que têm múltiplos tipos
- **Documentação de instalação** escrita para quem não é técnico

---

## Limites conhecidos

- **Revit 2023 e 2026 foram compilados, não testados em uso real.** Apenas o 2025 estava disponível na máquina de desenvolvimento.
- **Parte das entregas foi validada fora do Revit** — build, leitura do servidor, renderização da interface — e dependia do teste da equipe para confirmação final. Isso foi declarado a cada entrega.
- **Nomes repetidos** (5.198 grupos) são tratados com uma pergunta ao usuário, não resolvidos automaticamente.
- **A cópia da parceira envelhece**: precisa de alguém designado para rodar a sincronização.

---

## Stack

`C#` · `.NET 8` + `.NET Framework 4.8` (multi-targeting) · `WPF` · `Autodesk Revit API 2023/2025/2026` · `PowerShell` (instalador e empacotamento) · caminhos UNC de rede

---

> **Sobre o código:** projeto desenvolvido no contexto de vínculo empregatício. Este documento
> descreve decisões técnicas, problemas e resultados, sem reproduzir código proprietário.
> Caminhos de servidor e nomes de terceiros foram omitidos ou generalizados.
