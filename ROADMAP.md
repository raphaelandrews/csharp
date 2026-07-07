# Plano de Estudos: C# Backend — Júnior ao Sênior

> Contexto: você já vem do frontend (React/TS/Next.js) e já tem experiência prática de backend em Go (API REST com Chi, RBAC com JWT+bcrypt, estruturas de dados na mão, persistência com SQLite). Isso significa que você **não vai aprender "o que é uma API" do zero** — vai traduzir conceitos que já domina para o ecossistema .NET. Esse plano assume isso e pula fundamentos genéricos de backend que você já tem.

---

## Como usar este plano

- Cada fase tem **objetivo**, **tópicos**, **projeto prático** e **critério de "pronto para a próxima fase"**.
- Não avance de fase só por tempo — avance quando conseguir entregar o projeto prático sem colar código de tutorial.
- Duração é uma estimativa para quem já programa (seu caso), estudando de forma consistente, não full-time.

---

## Fase 0 — Setup do ambiente (2-3 dias)

- [x] Instalar .NET SDK (versão LTS mais recente) via pacman/AUR
- [x] Configurar LazyVim com extra `omnisharp` (ou testar Rider se quiser algo mais IDE-like)
- [x] Rodar `dotnet new console` e `dotnet run` só pra validar o ambiente
- [x] Instalar `netcoredbg` para debug, se for usar nvim-dap
- [x] Criar conta no NuGet (gerenciador de pacotes do .NET) — é o `npm`/`go mod` de vocês (skip — conta só necessária para publicar pacotes; consumo funciona anônimo)

**Critério de pronto:** conseguir criar, rodar e debugar um "hello world" em C# sem depender de IDE gráfica.

---

## Fase 1 — Fundamentos da linguagem C# (2-3 semanas)

### Tópicos
- [x] Sintaxe base: tipos, classes, structs, records, enums
- [x] Programação orientada a objetos
- [x] Generics
- [x] Nullable reference types (`string?`) e null-safety
- [x] `LINQ`
- [x] Async/await
- [x] Exceptions e tratamento de erros
- [x] Delegates, Func/Action, eventos

### Projeto prático
**`csharp-katas`** — console app com uma coleção de exercícios curtos e independentes: implementar do zero uma Linked List genérica, uma Binary Search Tree, um algoritmo de ordenação (ex: quicksort) e 2-3 problemas resolvidos com LINQ (ex: agrupar/transformar uma coleção de dados fictícios de várias formas). Cada exercício com testes xUnit.

**Critério de pronto:** você consegue explicar a diferença entre `struct` e `class` em C# e por que isso importa para performance/memória, e todos os testes passam.

---

## Fase 2 — ASP.NET Core e Web APIs (3-4 semanas)

### Tópicos
- [x] Minimal APIs vs Controllers
- [x] Dependency Injection nativo do .NET
- [x] Middleware pipeline
- [x] Model binding e validação (`FluentValidation` ou Data Annotations)
- [x] Configuração (`appsettings.json`, variáveis de ambiente, `IOptions<T>`)
- [x] Logging estruturado (`Serilog`)
- [x] Documentação de API com Swagger/OpenAPI (nativo no ASP.NET Core)

### Projeto prático
**`habitrack-api`** — API de acompanhamento de hábitos (tipo um "streak tracker" simples): usuário cria hábitos, marca conclusão diária, vê sequência atual. Endpoints de CRUD de hábitos + registro de conclusões + autenticação JWT básica.

**Critério de pronto:** API rodando com pelo menos 3 endpoints CRUD, com DI configurada corretamente e Swagger documentando tudo.

---

## Fase 3 — Persistência de dados (3-4 semanas)

### Tópicos
- Entity Framework Core (ORM) — migrations, DbContext, relacionamentos (1:N, N:N)
- Dapper (micro-ORM, mais próximo do que você já fez com SQL puro em Go) — muitas vagas pedem os dois
- Diferença entre Code First e Database First
- Transações e concorrência otimista/pessimista
- Repository Pattern e Unit of Work (padrões que aparecem MUITO em entrevistas de C#)
- SQL Server (o banco mais associado ao ecossistema .NET no mercado corporativo brasileiro) — pelo menos noções básicas, mesmo usando Postgres no dia a dia

### Projeto prático
Adicionar persistência real ao seu projeto da Fase 2 usando EF Core com SQL Server (ou Postgres), incluindo migrations versionadas.

> **Nota prática (Arch/Linux):** SQL Server não tem pacote nativo pro Arch — a forma de rodar localmente é via Docker (`docker run` com a imagem oficial `mcr.microsoft.com/mssql/server`). Se preferir simplicidade no dia a dia, comece com **Postgres** (nativo via pacman, `sudo pacman -S postgresql`) e só suba o SQL Server em container quando quiser praticar especificamente ele (bom pra ter na bagagem, já que é o banco mais citado nas vagas .NET corporativas).

**Critério de pronto:** você consegue explicar quando usaria EF Core vs Dapper numa entrevista, e seu projeto tem migrations aplicáveis do zero.

---

## Fase 4 — Testes, qualidade e boas práticas (2-3 semanas)

### Tópicos
- xUnit (testes unitários) — já viu na Fase 1, agora aprofundar
- Testes de integração com `WebApplicationFactory`
- Mocking com Moq ou NSubstitute
- SOLID aplicado a C# (você já deve conhecer os princípios, aqui é ver os idiomas específicos da linguagem)
- Clean Code e convenções da comunidade .NET (bem mais rígidas em convenção de nomenclatura que Go)

### Projeto prático
Cobrir o projeto da Fase 3 com testes unitários + pelo menos 1 teste de integração end-to-end do fluxo de autenticação.

**Critério de pronto:** cobertura de testes nas regras de negócio principais, não só em getters/setters.

---

## ✅ Checkpoint: Nível Júnior

Se você chegou até aqui com os projetos práticos funcionando, você já está empregável como **Júnior C#/.NET**. A partir daqui o foco muda de "aprender a linguagem" para "aprender a construir sistemas".

---

## Fase 5 — Arquitetura de software (4-6 semanas)

### Tópicos
- Clean Architecture / Arquitetura em camadas (Domain, Application, Infrastructure, API)
- Domain-Driven Design (DDD) — pelo menos os conceitos táticos (Entities, Value Objects, Aggregates)
- CQRS (Command Query Responsibility Segregation) — muito comum em vagas .NET pleno/sênior
- MediatR (biblioteca que implementa o padrão Mediator, quase onipresente em projetos .NET corporativos)
- Design Patterns clássicos aplicados: Factory, Strategy, Decorator, Repository

### Projeto prático
**`eventhub`** (novo projeto-âncora) — sistema de venda/reserva de ingressos para eventos: cadastro de eventos, sessões com capacidade limitada, reserva de assentos/vagas, confirmação de compra. É um domínio rico o suficiente pra justificar CQRS de verdade (comandos como "ReservarVaga" têm regras de negócio e concorrência reais, diferente de um CRUD simples). Construir com Clean Architecture (Domain, Application, Infrastructure, API) + CQRS via MediatR desde o início.

**Critério de pronto:** você consegue justificar cada camada da sua arquitetura numa entrevista técnica, não só copiar a estrutura de pastas de um tutorial.

---

## Fase 6 — Infraestrutura e Cloud (4-5 semanas)

### Tópicos
- Docker (containerizar a API .NET)
- **Kubernetes (conceitos básicos: pods, deployments, services)** — cada vez mais pedido junto com Docker, mesmo em vagas pleno; não precisa virar especialista, mas precisa saber orquestrar seus próprios containers localmente (ex: via Minikube ou Kind)
- CI/CD básico (GitHub Actions é suficiente pra portfólio)
- Azure fundamentals (App Service, Azure SQL, Key Vault) — Azure é o cloud mais associado a .NET no mercado, vale ter pelo menos noção mesmo se as vagas usarem AWS
- Health checks e observabilidade (Application Insights, OpenTelemetry, ou ferramentas como Dynatrace — aparecem com frequência em vagas pleno/sênior)
- Variáveis de ambiente e secrets management em produção

### Projeto prático
Dockerizar o **`eventhub`** e criar um pipeline de CI que roda testes automaticamente a cada push, com deploy em algum serviço gratuito (Azure App Service free tier, Render, ou Railway).

**Critério de pronto:** pipeline verde rodando testes + build + deploy automatizado.

---

## ✅ Checkpoint: Nível Pleno

Com arquitetura sólida + CI/CD + testes consistentes, você está no patamar de **Pleno**. Aqui o jogo muda de novo: agora é sobre sistemas distribuídos, performance e decisões de trade-off.

---

## Fase 7 — Sistemas distribuídos e mensageria (5-6 semanas)

### Tópicos
- Microsserviços: quando faz sentido e quando é over-engineering (pergunta clássica de entrevista sênior)
- Mensageria assíncrona: RabbitMQ ou Kafka (pelo menos um dos dois a fundo) — se pretende mirar vagas com stack AWS, vale saber que SQS/SNS resolvem o mesmo problema com outra sintaxe, os conceitos (fila, tópico, consumidor, at-least-once delivery) são os mesmos
- Padrões de resiliência: Circuit Breaker, Retry, Polly (biblioteca .NET pra isso)
- Cache distribuído com Redis
- gRPC como alternativa a REST para comunicação interna entre serviços
- Idempotência e consistência eventual

### Projeto prático
Quebrar o **`eventhub`** em pelo menos 2 serviços que se comunicam via mensageria: o serviço de reservas publica um evento "ReservaConfirmada", e um serviço de notificações consome esse evento e simula o envio de um e-mail/SMS de confirmação. Adicionar Redis pra cache de disponibilidade de vagas (evita bater no banco a cada consulta de "quantas vagas restam").

**Critério de pronto:** você consegue desenhar no quadro (ou papel) o fluxo de uma mensagem entre os serviços e explicar o que acontece se um deles cair.

---

## Fase 8 — Performance e produção (3-4 semanas)

### Tópicos
- Profiling de aplicações .NET (dotnet-trace, dotnet-counters)
- Garbage Collector do .NET — entender geração de objetos, quando isso importa
- Otimização de queries EF Core (N+1 problem é clássico aqui)
- Rate limiting e throttling de API
- Segurança: OWASP Top 10 aplicado a APIs .NET, JWT refresh tokens, CORS bem configurado

### Projeto prático
Pegar sua aplicação existente e fazer uma "auditoria" de performance: identificar e corrigir pelo menos 2 gargalos reais (query N+1, endpoint sem paginação, etc).

**Critério de pronto:** antes/depois documentado com números (latência, uso de memória).

---

## ✅ Checkpoint: Nível Sênior

Nesse ponto você já tem: linguagem dominada, arquitetura sólida, sistemas distribuídos, performance e produção. O que separa sênior de pleno daqui pra frente não é mais tópico técnico novo — é **julgamento**: saber quando NÃO aplicar um padrão, mentoria de outros devs, decisões de trade-off em discussões de arquitetura, e comunicação com stakeholders não-técnicos.

---

## Projetos para portfólio no GitHub

Estratégia: projetos pequenos e independentes nas fases de fundamentos (aprendizado isolado, sem bagagem de projeto anterior), um projeto-âncora que nasce na Fase 5 e evolui até o fim (mostra profundidade e progressão real), e satélites pontuais quando o conceito pede um contexto próprio. Cada repositório com README explicando decisões técnicas (não só "o que é"), diagrama simples de arquitetura, e link pro deploy ao vivo quando possível — recrutador olha README antes de olhar código.

### Nível Júnior (Fases 1-4) — pequenos e descartáveis

**1. `csharp-katas`** (Fase 1)
- Estruturas de dados e algoritmos implementados do zero em C# (Linked List genérica, Binary Search Tree, um algoritmo de ordenação) + exercícios resolvidos com LINQ.
- Objetivo: mostrar fundamentos sólidos na linguagem, sem depender de framework nenhum.

**2. `habitrack-api`** (Fases 2-4)
- API de acompanhamento de hábitos: cadastro de hábitos, registro de conclusões diárias, cálculo de sequência (streak).
- ASP.NET Core Minimal API + EF Core + JWT + testes xUnit/integração.
- Objetivo: primeiro projeto "de verdade" em C#, mostra o ciclo completo (API + banco + auth + testes) num domínio simples e visual, fácil de demonstrar numa entrevista ou até com um front simples depois.

### Nível Pleno (Fases 5-6) — nasce o projeto-âncora

**3. `eventhub`**
- Sistema de venda/reserva de ingressos para eventos: eventos, sessões com capacidade limitada, reserva de vagas, confirmação de compra.
- Construído com Clean Architecture (Domain, Application, Infrastructure, API) + CQRS via MediatR desde o início.
- Domínio rico o bastante pra ter regras de negócio reais (ex: não deixar reservar mais vagas do que a capacidade, lidar com concorrência quando dois usuários tentam pegar a última vaga ao mesmo tempo) — isso gera discussão técnica de verdade em entrevista, diferente de um CRUD genérico.
- Na Fase 6, ganha Docker + CI/CD + deploy — vira sua entrega mais completa até aqui.

### Nível Sênior (Fases 7-8) — evolução do âncora + satélite

**4. `eventhub` distribuído** (Fase 7)
- Quebrar o `eventhub` em serviços: o serviço de reservas publica um evento "ReservaConfirmada" via RabbitMQ, um serviço de notificações consome e simula envio de e-mail/SMS.
- Adicionar Redis como cache de disponibilidade de vagas.
- Adicionar Polly para retry/circuit breaker na comunicação entre serviços.
- Objetivo: mostrar raciocínio de sistemas distribuídos aplicado a um domínio que já existe e que você conhece bem — não é só "seguir um tutorial de microsserviços", é evoluir um sistema real.

**5. `api-performance-lab`** (Fase 8, satélite)
- Projeto isolado e pequeno, focado só em performance: uma API simples com um dataset propositalmente "problemático" (relacionamentos que geram N+1, endpoints sem paginação).
- Documentar no README o antes/depois de cada otimização (query, cache com Redis, paginação) com números reais de latência via load test (k6 ou similar).
- Objetivo: isolar o aprendizado de performance num ambiente controlado, onde fica fácil mostrar métricas claras de "antes e depois" sem o ruído de um sistema grande. Depois, aplique as mesmas otimizações que fizerem sentido de volta no `eventhub`.

### Dica geral de portfólio

- Priorize **4-5 projetos bem documentados** em vez de muitos projetos rasos.
- O `eventhub` é seu projeto mais importante — ele sozinho consegue mostrar progressão real (CRUD simples → Clean Architecture → distribuído) só de olhar o histórico de commits/branches ao longo do tempo.
- Sempre que possível, deixe pelo menos 1-2 projetos com deploy ao vivo (link funcional), não só "roda localmente".

---

## Fase contínua — Soft skills e carreira (paralelo a tudo acima)

- [ ] Manter um portfólio público no GitHub com os projetos de cada fase, documentados
- [ ] Escrever posts curtos (LinkedIn/blog) explicando decisões técnicas dos seus projetos — isso conta muito em entrevista sênior
- [ ] Contribuir com issues simples em projetos open source .NET (mesmo pequenas, como documentação)
- [ ] Praticar explicar arquitetura em voz alta / desenhando (entrevistas de sênior são menos "resolver algoritmo" e mais "discutir sistema")
- [ ] Ler releases notes de cada versão nova do .NET — o ecossistema evolui rápido e isso é sinal de profissional atualizado
- [ ] **Vocabulário de metodologias ágeis** (Scrum, Kanban, cerimônias como daily/retro/planning) — quase toda vaga cita isso nos requisitos, mesmo sendo mais processo do que técnica pura; não precisa de curso, só entender os termos e participar ativamente se estiver em equipe
- [ ] **Inglês técnico** — ler documentação, error messages e ocasionalmente participar de reunião em inglês aparece como requisito em uma fatia relevante das vagas pleno/sênior, principalmente em consultorias e empresas com cliente internacional. Não precisa ser fluente, mas treine leitura técnica desde já
- [ ] **Nota sobre sistemas legados:** uma parte considerável do mercado .NET brasileiro ainda mantém aplicações em **.NET Framework** (não só .NET Core/5+) e às vezes até Web Forms. Você não precisa estudar isso a fundo, mas saiba que existe — é comum ver isso pedido em vagas de sustentação, e vale mencionar em entrevista que você sabe da diferença entre Framework e Core/moderno
- [ ] **Seu diferencial de fullstack:** vagas .NET pleno/sênior frequentemente pedem React/TypeScript/Angular como parte do pacote (perfil "fullstack .NET"). Você já vem forte de frontend — isso é uma vantagem competitiva real, vale destacar no currículo e no LinkedIn desde o início, não só depois de virar backend "puro"

---

## Recursos sugeridos (pontos de partida, não lista fechada)

- **Documentação oficial da Microsoft Learn** — gratuita e muito bem estruturada, especialmente para ASP.NET Core e EF Core
- **Livro:** "C# in Depth" (Jon Skeet) — referência pra entender a linguagem a fundo, não só sintaxe
- **Livro:** "Clean Architecture" (Robert C. Martin) — para a Fase 5
- Canais e comunidades brasileiras de .NET (procure grupos como ".NET Brasil" no Telegram/Discord) — bom pra tirar dúvida e ver vagas

---

## Cronograma estimado (visão geral)

| Fase | Nível alvo | Duração estimada |
|---|---|---|
| 0-1 | Setup + Fundamentos | ~1 mês |
| 2-4 | Web API + Dados + Testes | ~2,5 meses |
| **Checkpoint Júnior** | | **~3,5 meses** |
| 5-6 | Arquitetura + Cloud | ~2,5 meses |
| **Checkpoint Pleno** | | **~6 meses** |
| 7-8 | Distribuído + Performance | ~2 meses |
| **Checkpoint Sênior (base técnica)** | | **~8 meses** |

> Sênior de verdade também exige tempo de experiência real em produção e decisões sob pressão — isso não se acelera só estudando, mas aos 8 meses você tem a **base técnica completa** para ser sênior, faltando principalmente bagagem de projetos reais.
