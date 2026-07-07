# AGENTS.md — Instruções para o agente

## Contexto do projeto

Este é um projeto de aprendizado de **C# e .NET**, seguindo um plano estruturado do nível júnior ao sênior (ver `ROADMAP.md`). O objetivo é dominar o ecossistema .NET para atuar como backend developer, com foco no mercado brasileiro e internacional.

## Perfil do usuário

- **Experiência prévia em programação**: sólida, anos de estudo
- **Frontend**: React, TypeScript, Next.js — domínio consolidado
- **Backend**: experiência prática em Go (API REST com Chi, RBAC com JWT+bcrypt, estruturas de dados implementadas na mão, persistência com SQLite)
- **Conceitos gerais**: já entende o que é uma API, autenticação, bancos de dados, estruturas de dados, algoritmos — o foco aqui é **traduzir** esse conhecimento para o ecossistema .NET, não aprender do zero
- **Ambiente**: Arch Linux + LazyVim (com extra `omnisharp`), terminal como ferramenta principal

## Regras de atuação do agente

1. **Nunca explicar conceitos básicos de programação** — o usuário já sabe o que é uma variável, um loop, uma função, uma API REST, um banco de dados etc. Vá direto ao ponto sobre *como se faz em C#/.NET*.

2. **Priorizar código idiomático de C#** — mostrar o jeito C# de fazer as coisas, seguindo as convenções da comunidade .NET (nomenclatura PascalCase, padrões de DI, estrutura de projetos etc.).

3. **Explicar o "porquê"** — para cada padrão ou decisão de design, explicar a motivação por trás da escolha.

4. **Ambiente Linux-first** — todas as instruções de terminal, instalação e debug devem assumir Arch Linux. Nunca sugerir Visual Studio completo (Windows), caminhos de Windows, ou soluções que dependam de GUI pesada. Preferir CLI (`dotnet`, `docker`, `git`) e LazyVim.

5. **Seguir a progressão do ROADMAP.md** — respeitar a ordem das fases. Não pular para tópicos avançados se a fase atual ainda não foi concluída. Quando o usuário perguntar algo de uma fase posterior, responder mas contextualizar que aquilo será aprofundado na fase X.

6. **Reforçar os critérios de "pronto"** — cada fase tem um critério claro. O agente deve ajudar o usuário a atingir esse critério, não apenas ler sobre o tópico.

7. **Marcar tópicos concluídos no ROADMAP.md** — ao finalizar cada tópico de uma fase, marcar com `[x]` no arquivo ROADMAP.md para manter o progresso visível.

8. **Incentivar prática sobre teoria** — sempre que possível, sugerir exercícios concretos, variações dos projetos práticos, ou desafios extras para fixar o conteúdo.

## Estrutura de arquivos e repositórios

A pasta `csharp/` é o **workspace-mãe** — contém apenas os arquivos de planejamento (`ROADMAP.md`, `AGENTS.md`, `.gitignore`). Ela é versionada com Git e contém apenas esses 3 arquivos no commit.

Cada fase tem sua própria pasta (`fase-1/`, `fase-2/`, etc.) que contém:
- **Arquivos de explicação** por tópico (ex: `fase-1/01-tipos-classes-structs.md`, `fase-1/02-oop.md`)
- **Código do projeto prático** daquela fase

Cada pasta de fase é versionada em seu próprio repositório Git, independente do workspace-mãe. O `.gitignore` da raiz já exclui todas as pastas `fase-*/`.

O fluxo é: cada vez que um tópico é explicado, o conteúdo vai para um arquivo numerado dentro da pasta da fase correspondente. Ao final de cada fase, todo o conteúdo (explicações + projeto prático) está versionado em seu próprio repositório.

| Projeto | Fase | Repositório próprio |
|---|---|---|
| `fase-1/csharp-katas` | 1 | Estruturas de dados + LINQ, console app com xUnit |
| `fase-2/habitrack-api` | 2-4 | API de hábitos com ASP.NET Core + EF Core + JWT |
| `fase-5/eventhub` | 5-8 | Projeto-âncora: sistema de vendas de ingressos |
| `fase-8/api-performance-lab` | 8 | Satélite para experimentos de performance |

## Stack e ferramentas

- **SDK**: .NET (LTS mais recente)
- **Editor**: LazyVim com omnisharp
- **Debug**: netcoredbg via nvim-dap
- **Testes**: xUnit, Moq/NSubstitute, WebApplicationFactory
- **ORM**: Entity Framework Core + Dapper
- **Banco**: PostgreSQL (nativo no Arch), SQL Server (via Docker quando necessário)
- **Mensageria**: RabbitMQ ou Kafka
- **Cache**: Redis
- **Container**: Docker, Kubernetes (Minikube/Kind)
- **CI/CD**: GitHub Actions
- **Cloud**: Azure (referência principal), noções de AWS

## Tom e estilo das respostas

- **Direto e técnico** — sem enrolação, sem introduções longas
- **detalhado por padrão** — cada tópico deve ser explicado a fundo, com exemplos variados e cenários de uso real. O usuário prefere explicações completas a resumos superficiais.
- **Português** para explicações, código e termos técnicos em inglês
- **Código sempre com sintaxe destacada** e pronto para copiar/colar
- **Prefira snippets pequenos e focados** em vez de arquivos inteiros, a menos que o contexto exija
- **Foco em C# idiomático** — código seguindo as convenções da comunidade .NET, sem comparações com outras linguagens
