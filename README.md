# Portfólio — Marcio Cezar

Estudos de caso de projetos que desenvolvi na **VIAVOZ LTDA**, empresa de engenharia e arquitetura, onde atuei como técnico de suporte entre 2025 e 2026.

Cada documento descreve o problema, as decisões técnicas, os bugs enfrentados e os limites conhecidos de cada projeto.

---

| Projeto | O que é | Stack |
|---|---|---|
| **[Render BIM](CASO-01-Render-BIM.md)** | Plugin de renderização em tempo real para Autodesk Revit, com visualizador 3D próprio rodando em processo separado | C#, .NET 8, Direct3D 11, Revit API |
| **[Registro de Horas](CASO-02-Registro-de-Horas.md)** | Diagnóstico e correção de um sistema de apontamento de horas em produção, cujos totais mudavam sozinhos entre carregamentos | JavaScript, PostgreSQL, Supabase |
| **[MC-Blocks](CASO-03-MC-Blocks.md)** | Biblioteca de 32 mil famílias para Revit, substituindo uma ferramenta comercial que havia parado de funcionar | C#, WPF, Revit API, PowerShell |

---

## Por que estudos de caso e não código

Os três projetos foram desenvolvidos durante vínculo empregatício, e **o código-fonte pertence à empresa**.

Estes documentos descrevem arquitetura, decisões e resultados sem reproduzir código proprietário. Caminhos de servidor, nomes de colegas e identificadores de projeto foram omitidos ou generalizados.

## O que procurar em cada um

- **Render BIM** — a decisão de rodar o renderizador fora do processo do Revit, e por que ela pagou quando a integração oficial da Epic falhou
- **Registro de Horas** — um bug silencioso de paginação em produção, e por que eu não apliquei a correção
- **MC-Blocks** — o modo de falha que faz um plugin desaparecer da interface sem gerar erro, e dois diagnósticos meus que estavam errados

---

📫 marciogamercaldeira@gmail.com
