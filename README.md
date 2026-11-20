# Portfólio — Marcio Cezar

Estudos de caso de projetos que desenvolvi na **VIAVOZ LTDA**, empresa de engenharia e arquitetura, onde atuei como técnico de suporte entre 2025 e 2026.

Cada documento descreve o problema, as decisões técnicas, os bugs enfrentados e os limites conhecidos de cada projeto.

### → **[Ver o portfólio no navegador](https://marcio282003.github.io/portfolio/)**

---

| Projeto | O que é | Stack |
|---|---|---|
| **[Render BIM](CASO-01-Render-BIM.md)** | Plugin de renderização em tempo real para Autodesk Revit, com visualizador 3D próprio rodando em processo separado | C#, .NET 8, Direct3D 11, Revit API |
| **[Registro de Horas](CASO-02-Registro-de-Horas.md)** | Diagnóstico e correção de um sistema de apontamento de horas em produção, cujos totais mudavam sozinhos entre carregamentos | JavaScript, PostgreSQL, Supabase |
| **[MC-Blocks](CASO-03-MC-Blocks.md)** | Biblioteca de 32 mil famílias para Revit, substituindo uma ferramenta comercial que havia parado de funcionar | C#, WPF, Revit API, PowerShell |

---

## Como estes projetos foram construídos

Os três foram desenvolvidos **com apoio intensivo de IA**. Meu papel foi identificar o problema, escrever a especificação, testar cada versão e reportar os defeitos, decidir escopo e risco, e conduzir a implantação pelas equipes. O código foi escrito pela IA a partir disso.

Não programo C# de forma fluente — estou cursando Análise e Desenvolvimento de Sistemas e aprendendo os fundamentos agora. Cada documento traz essa separação de forma explícita, porque ela explica o resultado.

## Por que estudos de caso e não código

Os três projetos foram desenvolvidos durante vínculo empregatício na VIAVOZ LTDA, e **o código-fonte pertence à empresa**.

Estes documentos descrevem arquitetura, decisões e resultados sem reproduzir código proprietário. Caminhos de servidor, nomes de colegas e identificadores de projeto foram omitidos ou generalizados.

## O que procurar em cada um

- **Render BIM** — a decisão de rodar o renderizador fora do processo do Revit, e por que ela pagou quando a integração oficial da Epic falhou
- **Registro de Horas** — um bug silencioso de paginação em produção, e por que eu não apliquei a correção
- **MC-Blocks** — o modo de falha que faz um plugin desaparecer da interface sem gerar erro, e dois diagnósticos que estavam errados antes de chegar na causa real

---

📫 marciogamercaldeira@gmail.com
