# Registro de Horas — Diagnóstico e correção de um sistema em produção

**JavaScript · Supabase (PostgreSQL) · SQL · React · Excel**
Desenvolvido durante atuação na VIAVOZ LTDA (engenharia e arquitetura) · 2026

---

## O problema

A empresa registrava horas trabalhadas num sistema web interno. Um segundo painel lia o mesmo banco para mostrar produtividade por projeto, por pessoa e por disciplina.

**Os dois mostravam números diferentes.** E ninguém sabia qual estava certo.

Minha tarefa começou como *"montar um sistema de registro de horas"*. O diagnóstico mudou a pergunta: o sistema já existia e funcionava. O que estava quebrado era a confiança nos números.

---

## A descoberta principal: um bug silencioso de paginação

### O sintoma

Medi o total de horas da mesma obra três vezes seguidas, sem ninguém lançar nada entre as medições:

```
4578:12   →   4612:42   →   4631:42
```

**O número mudava sozinho a cada atualização da página.**

### A causa

O sistema buscava os registros em páginas de 1.000 linhas, ordenando por data:

```js
.order('data', { ascending: false })
.range(from, from + tamanho - 1);
```

O problema: **`data` se repete centenas de vezes.** Quando várias linhas empatam no critério de ordenação, o PostgreSQL **não garante a mesma ordem entre elas de uma consulta para outra** — e cada página é uma consulta nova.

Resultado: a cada carregamento, cerca de **70 lançamentos vinham duplicados e outros 70 não vinham**, sorteados de novo toda vez. Os duplicados tinham as horas somadas em dobro; os ausentes sumiam da conta.

### Por que ninguém tinha percebido

O erro é **silencioso**. Não gera exceção, não trava nada, e os números continuam plausíveis. Só aparece quando alguém compara duas leituras da mesma coisa — que foi exatamente o que eu fiz.

Ele contaminava a tela por obra, a por colaborador, o dashboard **e todos os Excel exportados**.

### A verificação

Conferi obra por obra contra o banco:

| | Obras corretas |
|---|---|
| Painel novo (ordenado por `id`) | **11 de 11** |
| Sistema em produção (ordenado por `data`) | **4 de 11** |

### A correção

Três linhas. Trocar `.order('data', …)` por `.order('id', …)`.

`id` é único — sem empate, não há o que reordenar. A ordem passa a ser idêntica em toda consulta.

> **A frase que resume:** *o sistema dividia a lista por um critério que empata; o novo divide por um critério que nunca empata.*

---

## A decisão que eu não tomei

O sistema com o bug **não era meu**. Era de outro desenvolvedor, já integrado a outros processos da empresa.

A orientação foi clara: *"não pode alterar nada do dele — se alterarmos pode causar danos e atrasar informações necessárias."*

**Concordei, e isso mudou minha abordagem.** Sistema integrado em produção não se altera por causa de um relatório.

O que fiz em vez disso:

1. **Documentei a correção** com o código exato, o antes/depois e o número de cada linha, para o autor aplicar se quisesse
2. **Ranqueei cinco ajustes possíveis por impacto ÷ risco** — o critério certo num sistema que não pode parar:

| # | O quê | Linhas | Risco | Ganho |
|---|---|---|---|---|
| 1 | Paginação por `id` | **3** | muito baixo | números param de oscilar |
| 2 | Bloquear data futura | 4 | muito baixo | elimina 77 registros inválidos |
| 3 | Botão de atualizar | ~6 | baixo | tela deixa de congelar |
| 4 | Obra por lista, não texto livre | ~10 | médio | acaba com nome duplicado |
| 5 | Campo `tipo_hora` no banco | migração | médio | produtividade confiável |

3. **Escrevi a mensagem técnica** para o autor do sistema — explicando o problema, a evidência medida e a correção, sem acusação
4. **Recomendei explicitamente não fazer os itens 4 e 5 naquele momento**, por travarem o fluxo de trabalho da equipe

---

## O diagnóstico inicial: seis problemas

Antes de escrever uma linha, li os dois sistemas (7.582 e 1.249 linhas de HTML) e levantei o que estava errado:

**1. A métrica central do painel vinha de um campo que não existia.**
O tipo de hora (produtiva, reunião, gestão, férias, treinamento) era gravado como `0` fixo. Para compensar, havia uma regra que classificava como improdutiva qualquer obra cujo **nome em texto** fosse "FERIAS", "ATESTADO", "REUNIAO GERAL". Bastava alguém digitar `"Férias "` com espaço, ou `"Ferias 2026"`, para a produtividade sair errada.

**2. Salário no `localStorage` do navegador.**
Dado de RH guardado no navegador de cada gestor: sumia ao limpar o histórico, não batia entre pessoas e não tinha controle de acesso.

**3. Obra, disciplina e tarefa em texto livre.**
`"Ed. Aurora"`, `"ED AURORA"` e `"Edificio Aurora"` viravam três projetos distintos nos relatórios. As tabelas de cadastro já existiam — o formulário simplesmente não obrigava a usá-las.

**4. Agregação inteira no navegador.**
Os dois apps baixavam a tabela de registros completa. Com 30 pessoas e 3 lançamentos por dia, chega-se a ~100 mil linhas em quatro anos — trava no celular muito antes disso.

**5. Manutenibilidade.**
Um HTML de 7.582 linhas com CSS e JS embutidos. E o painel compilava React **no navegador** via Babel — modo de demonstração, explicitamente desaconselhado em produção.

**6. Segurança por verificar.**
A flag de administrador era lida do perfil e usada apenas para **mostrar ou esconder abas no front-end**. Se as políticas de RLS do banco permitissem leitura ampla, qualquer funcionário com o DevTools aberto leria as horas de todo mundo. Isso entrou como item de auditoria obrigatória.

---

## Decisões técnicas

### Não trocar de linguagem

A pergunta original era *"qual a melhor linguagem?"*. A resposta honesta foi: **continuar em JavaScript + Supabase.**

Trocar para Python, PHP ou .NET jogaria fora autenticação, banco, notificações e ~9 mil linhas que funcionavam — para resolver problemas que **não eram de linguagem, e sim de modelagem de dados.** O ganho seria zero e o custo, meses.

Recomendar o que já existe, em vez de propor a reescrita que daria mais trabalho visível, foi a decisão certa.

### Migrações aditivas, nunca destrutivas

O banco estava **em produção**. Toda migração foi escrita como aditiva e reversível:

- Backup das tabelas para um schema datado, antes de qualquer alteração
- `ALTER TABLE ... ADD COLUMN` com default — nunca `DROP`, nunca renomear coluna em uso
- Colunas de texto mantidas durante a transição para chaves estrangeiras
- Um `SELECT` de conferência **antes** de cada `UPDATE`, mostrando quantas linhas e quantas horas seriam afetadas

### Custo por hora com vigência

A tabela de custo não guarda apenas `valor_hora` por pessoa. Guarda `vigencia_inicio` e `vigencia_fim`.

**Por quê:** sem vigência, uma promoção recalcularia o custo de todos os meses passados, e o histórico mudaria sozinho. Com vigência, março de 2025 continua calculado pelo salário de março de 2025.

### Recusar a credencial de administrador

Quando o acesso ao banco virou necessário, a oferta era a *service role key* do Supabase — que dá poder total sobre os dados.

**Recusei e propus a alternativa:** rodar consultas somente-leitura no SQL Editor e me passar o resultado. Eu via o schema real sem que nenhuma credencial mudasse de mãos.

### Deixar uma decisão em aberto, de propósito

Uma função de leitura de custo ficou marcada como `SECURITY DEFINER`, o que a torna legível por qualquer usuário autenticado. Deixei a escolha documentada e **não decidida** no código, com a alternativa comentada ao lado.

**Motivo:** se salário é restrito a administradores naquela empresa é política de RH, não decisão de quem escreve o código.

---

## O que foi entregue

**Painel de produtividade corrigido**
- Leitura determinística (paginação por `id`)
- Modo **"igual ao sistema"**: alinha contagem de obras, critérios de exclusão e formato de horas com o sistema legado, para comparação lado a lado numa apresentação
- Aba **Alertas** com botão "Conferir agora", que mede na hora quantas linhas o sistema legado duplica e perde
- Abas de **Cobertura** e **Quando Lançaram**, expondo os problemas de dado

**Migrações SQL** — introspecção somente-leitura, backup, catálogo de tipos de hora com backfill normalizado (removendo acentos e espaços), tabela de custos com vigência e RLS

**Exportação em Excel com seis abas** — por pessoa, por projeto, pessoa × projeto, detalhe por dia, conferências e um "Leia-me" com 31 verbetes explicando a origem de cada número

---

## Achados de qualidade de dado

O diagnóstico também expôs problemas que não eram de software:

| Achado | Natureza |
|---|---|
| **77 lançamentos com data futura** | erro puro — horas lançadas para dias que não chegaram |
| **Apenas 46% lançam no mesmo dia** | 731 lançamentos com mais de 7 dias de atraso |
| **240 horas lançadas de uma vez**, retroativas | por isso um mês "fechado" muda de valor depois |
| Nomes de obra escritos de formas diferentes | inflam a contagem de projetos |

A separação importava: o item 1 é **erro**, corrigível hoje. Os itens 2 e 3 **não são defeito** — são comportamento real das pessoas, e a solução é de gestão (uma data de corte), não de código.

Sugerir "feche o mês dia 5" quando o problema não é técnico é mais útil que escrever código.

---

## A parte de comunicação

Boa parte do valor deste trabalho não foi código.

Precisei explicar a um gestor **por que dois números não podem ficar iguais** — porque o número do sistema legado não é estável, então não existe valor para copiar. E precisei dizer isso sem desqualificar o trabalho de outra pessoa.

A analogia que usei, e que funcionou:

> Imagine uma gaveta com 10.000 fichas. O navegador pede de 1.000 em 1.000.
> Mas **primeiras em que ordem?** Se centenas de fichas têm a mesma data, o banco
> entrega os empates numa ordem qualquer — e cada página é uma pergunta nova.
> Umas fichas vêm duas vezes, outras não vêm nenhuma.

Também preparei a saída prática para a apresentação: ligar o modo "igual ao sistema", e ter o botão de conferência à mão para o caso de alguém questionar a diferença de ~0,1% nas horas.

---

## Limites conhecidos

- **A correção na paginação do sistema legado nunca foi aplicada** — a decisão de não mexer era da empresa, e foi respeitada. O documento ficou pronto para quando quisessem.
- **A unificação em Next.js não chegou a ser feita.** Ficou como recomendação; a prioridade foi corrigir o dado.
- **Validação contra dados reais foi parcial** — parte das funcionalidades foi testada com dados sintéticos, por limitação de sessão no banco. Isso está declarado nas entregas.
- O campo `tipo_hora` foi projetado e migrado, mas depende de mudança no formulário de registro, que pertence ao sistema legado.

---

## Stack

`JavaScript` · `SQL (PostgreSQL)` · `Supabase` (Auth, RLS, Row Level Security) · `React` · `Recharts` · `SheetJS/openpyxl`

---

> **Sobre o código:** projeto desenvolvido no contexto de vínculo empregatício. Este documento
> descreve diagnóstico, decisões e resultados, sem reproduzir código proprietário. Nomes de
> colegas e identificadores de projeto foram omitidos.
