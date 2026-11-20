# Render BIM — Plugin de renderização em tempo real para Autodesk Revit

**C# · .NET 8 · Direct3D 11 · Revit API 2025/2026**
Desenvolvido durante atuação na VIAVOZ LTDA (engenharia e arquitetura) · 2026

---

## O problema

As equipes de arquitetura dependiam de **Enscape** e **Lumion** para apresentar projetos a clientes. São ferramentas caras, licenciadas por posto de trabalho, e nem todo mundo da equipe tinha acesso.

A pergunta que originou o projeto foi direta: **dá para ter uma prévia navegável do modelo, dentro do próprio Revit, sem depender dessas licenças?**

Eu atuava como suporte técnico. Esse projeto nasceu de observar o gargalo da equipe de arquitetura e decidir atacá-lo com código.

---

## A decisão de arquitetura

A escolha mais importante do projeto foi feita antes de escrever a primeira linha: **o renderizador roda em um processo separado do Revit.**

```
┌─────────────────┐         IPC          ┌──────────────────────┐
│   Revit 2025    │   named pipe +       │  RenderBIM.Viewer    │
│   ┌───────────┐ │   memory-mapped      │  ┌────────────────┐  │
│   │ RevitAddin│◄├──────── file ───────►│  │  Direct3D 11   │  │
│   └───────────┘ │                      │  └────────────────┘  │
└─────────────────┘                      └──────────────────────┘
      processo 1                                processo 2
```

**Por quê:**

1. **Isolamento de falha.** Um crash de GPU ou do motor gráfico não derruba o Revit — e com ele, horas de trabalho não salvo do arquiteto. Em ferramenta que roda dentro do software de produção de outra pessoa, isso não é detalhe.
2. **Motor substituível.** O renderizador pode ser trocado sem mexer na integração com o Revit.
3. **A thread da interface do Revit fica livre.** Operações pesadas não travam a tela de quem está projetando.

A fronteira foi mantida de forma estrita: **apenas os projetos `RevitIntegration` e `RevitAddin` conhecem a API do Revit.** A comunicação acontece por um contrato de dados em `SharedContracts`. O resto do sistema não sabe que o Revit existe.

Isso pagou dividendos mais tarde — quando a ponte oficial da Epic falhou, foi possível escrever um exportador próprio reaproveitando o contrato de cena que já existia.

**Compatibilidade multiversão:** Revit 2025 e 2026 em `net8.0-windows`, com a arquitetura preparada para o 2027 (cuja API exige `net10.0-windows`). Revit 2024 e anteriores ficaram explicitamente fora de escopo, por usarem .NET Framework 4.8.

---

## O que foi construído

**Integração com o Revit**
- Aba própria na Ribbon (`IExternalApplication`, `IExternalCommand`, `PushButtonData`)
- Leitura da vista 3D com classificação por disciplina (arquitetura, estrutura, elétrica, hidrossanitária, HVAC, terreno, vínculos)
- Sincronização do modelo e câmera ao vivo
- Instalador próprio com alvo duplo (2025/2026) e empacotamento automatizado via `build.ps1 -Package -RunTests`

**Navegação estilo Enscape/Lumion**
- Caminhar e voar (tecla `V`), com altura de olhos de 1,65 m e acompanhamento do piso
- Aceleração e frenagem suaves, em vez de parada seca
- Duplo clique para ir até um ponto, com transição animada
- Vistas guardadas em `Ctrl+1`–`Ctrl+9`, recuperadas com animação
- Controle de velocidade, correr e andar devagar

**Biblioteca de materiais**
- 91 materiais em 11 categorias, mais 40 pacotes adicionais do ambientCG
- Todos com licença **CC0** (Poly Haven e ambientCG) — escolha deliberada, sem risco de licenciamento
- Prévia em esfera iluminada, busca por nome e filtro por categoria
- Mapas de relevo e brilho variável por imagem
- Troca por elemento individual **ou** por todos os materiais de mesmo nome

**Vegetação e mobiliário procedurais**
- 6 tipos de árvore (copa larga, ipê, jacarandá, palmeira, conífera, arbusto)
- Copas construídas com centenas de cartões de folhas recortados por transparência, com translucidez e variação de tom
- ~1.000 triângulos por árvore — dezenas de árvores rodam numa GTX 1650
- Substituição de objetos do modelo (cadeira, mesa, sofá, cama, banco, poste, vaso, pessoa, carro), com retorno da peça original ao desfazer

**Exportação para Unreal Engine**
- Exportador glTF próprio (`GlbExporter.cs`), **sem dependências externas**
- Agrupamento de malhas por material: um projeto real saiu de **2.215 malhas para 84**, mantendo os mesmos 1.508.700 triângulos
- Texturas PNG e JPEG embutidas no arquivo

---

## Bugs reais e como foram resolvidos

Esta é a parte que mais ensina, e a que eu levaria para uma entrevista.

### 1. O bug que não aparecia na minha máquina

Uma avaliadora com **RTX 3080** recebia "Não foi possível iniciar o visualizador 3D". Na minha GTX 1650, nunca falhava.

**Causa:** a leitura da memória de vídeo estourava o tipo em placas com 4 GB ou mais. Minha placa ficava exatamente **abaixo** do limite — por isso o código nunca quebrava para mim.

**Lição:** o "funciona na minha máquina" tem causa técnica identificável. Sem um testador com hardware diferente, esse bug chegaria ao usuário final.

### 2. Z-fighting nas paredes

Durante a navegação, paredes e portas **piscavam** ao mover a câmera. Inaceitável numa ferramenta de apresentação para cliente.

**Causa:** o Revit gera faces coplanares — camadas de parede, batentes, acabamentos. A GPU ficava "decidindo" qual face aparecia e mudava a decisão a cada quadro.

**Solução:** prioridade fixa de profundidade por material, com deslocamento de décimos de milímetro, invisível a olho nu. A primeira versão tinha 12 níveis e ainda deixava ~8% de disputas; subindo para **48 níveis**, caiu para ~2%.

**O que eu faria diferente:** medir a taxa de disputa *antes* de escolher o número de níveis, em vez de depois.

### 3. Seleção disputando com a topografia

Clicar num piso às vezes selecionava o terreno.

**Causa:** o raycast retornava a primeira superfície no caminho. Pisos assentados na mesma cota do terreno empatavam, e o terreno às vezes vencia.

**Solução:** critério de desempate — em empate técnico de distância, o elemento menor vence e a topografia perde.

### 4. Escopo errado na edição de material

Mudar a cor de uma parede mudava **todos** os elementos que usassem aquele material, inclusive o terreno.

**Solução:** separação explícita entre editar o *material* e editar o *elemento*, com a opção visível na interface e padrão seguro.

### 5. Quando a ponte oficial não funciona

A integração recomendada entre Revit e Unreal é o **Datasmith**, da Epic. Ele não funcionava no Revit 2025 — confirmado por relatos em fórum e pelo teste local.

**Decisão:** em vez de esperar a Epic, escrevi um exportador glTF próprio, reaproveitando o contrato de cena que a arquitetura já fornecia. Virou o botão "Exportar para Unreal", entregue na versão 0.7.0.

---

## Números

| | |
|---|---|
| Versões entregues | **21**, da 0.1.0 à 0.17.0 |
| Testes automatizados | **147** (cresceu de 13, junto com o projeto) |
| Desempenho | ~62 FPS em GTX 1650 (4 GB) |
| Maior modelo processado | 1,5 milhão de triângulos · 85 materiais · cena de 68 MB |
| Projetos na solução | 7 |
| Redução de malhas na exportação | 2.215 → 84 |
| Instalador | alvo duplo Revit 2025/2026, com verificação de hash SHA-256 |

---

## A decisão mais difícil: motor próprio ou Unreal?

Chegado o pedido de **qualidade fotorrealista e link navegável para o cliente**, ficou claro que o motor próprio não chegaria lá em tempo hábil.

Levantei a alternativa a sério, em vez de defender o que já tinha construído:

| | Motor próprio | Unreal Engine |
|---|---|---|
| Qualidade visual | Evolui devagar, meses até se aproximar | Alta desde o primeiro dia (Lumen, Nanite) |
| Link/QR para cliente | Construir do zero | Pixel Streaming pronto |
| Licença | Zero | Gratuito abaixo de US$ 1 mi de faturamento; ~US$ 1.850/ano/usuário acima |
| Peso | Roda em GTX 1650 | Exige RTX, 8–12 GB de VRAM |
| Controle | Total | Limitado ao motor |

**Recomendação: modelo híbrido.** O plugin próprio permanece como prévia rápida e leve dentro do Revit — função que o Unreal não cumpre bem, por ser um programa separado e pesado. O Unreal entra na entrega final ao cliente.

Isso também significou levantar o custo de hardware (RTX com 8–12 GB, 32 GB de RAM) e apresentar à empresa como decisão de investimento, não como detalhe técnico.

**O ponto que importa:** a resposta honesta era que a ferramenta que eu construí não resolvia sozinha o problema completo. Dizer isso é mais útil para a empresa do que defender o próprio trabalho.

---

## Limites conhecidos

Nenhuma ferramenta é só o que funciona. O que ficou de fora:

- **Não é Enscape.** Sem iluminação global, reflexos de espelho ou path tracing. A qualidade é de prévia, não de imagem final.
- **Vegetação procedural tem teto.** Folhas desenhadas por código sempre ficarão abaixo de modelos reais escaneados. O salto de qualidade real exigiria uma biblioteca de assets pesada, que a GTX 1650 não comporta.
- **Instalador sem assinatura digital.** O SmartScreen alerta. Para distribuição ampla, precisaria de certificado.
- **Edições só no visualizador.** Trocas de material e objetos não retornam ao Revit nem ao arquivo exportado.
- **Sincronização relê a vista inteira**, em vez de atualizar incrementalmente.

---

## Stack

`C#` · `.NET 8` · `Autodesk Revit API 2025/2026` · `Direct3D 11` · `WPF`
IPC por named pipe + memory-mapped file · glTF/GLB · PowerShell (build e empacotamento) · testes unitários

---

> **Sobre o código:** este projeto foi desenvolvido no contexto de vínculo empregatício, e o
> código-fonte pertence à empresa. Este documento descreve decisões de arquitetura,
> problemas enfrentados e resultados — sem reproduzir código proprietário.
