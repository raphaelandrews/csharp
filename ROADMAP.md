# Plano de Estudos: C# Backend

---

## Fase 0 — Setup do ambiente

- [x] Instalar .NET SDK (LTS) via pacman/AUR
- [x] Configurar LazyVim com extra `omnisharp`
- [x] Rodar `dotnet new console` e `dotnet run`
- [x] Instalar `netcoredbg` para debug via nvim-dap
- [x] Criar conta no NuGet (skip — consumo funciona anônimo)

---

## Fase 1 — Fundamentos da linguagem C#

### Tópicos
- [x] Sintaxe base: tipos, classes, structs, records, enums
- [x] Programação orientada a objetos
- [x] Generics
- [x] Nullable reference types e null-safety
- [x] LINQ
- [x] Async/await
- [x] Exceptions e tratamento de erros
- [x] Delegates, Func/Action, eventos

### Projeto prático
**`csharp-katas`** — console app com exercícios de estruturas de dados implementadas do zero (Linked List genérica, Binary Search Tree, algoritmo de ordenação) e problemas resolvidos com LINQ. Testes com xUnit.

---

## Fase 2 — ASP.NET Core e Web APIs

### Tópicos
- [x] Minimal APIs vs Controllers
- [x] Dependency Injection nativo do .NET
- [x] Middleware pipeline
- [x] Model binding e validação com FluentValidation
- [x] Configuração (`appsettings.json`, variáveis de ambiente, `IOptions<T>`)
- [x] Logging estruturado com Serilog
- [x] Documentação de API com Swagger/OpenAPI

### Projeto prático
**`habitrack-api`** — API de acompanhamento de hábitos: CRUD de hábitos, registro de conclusões diárias, cálculo de sequência (streak), autenticação JWT.

---

## Fase 3 — Persistência de dados

### Tópicos
- [x] Entity Framework Core (ORM)
- [x] Dapper (micro-ORM)
- [x] Code First vs Database First
- [x] Transações e concorrência otimista/pessimista
- [x] Repository Pattern e Unit of Work
- [x] SQL Server

### Projeto prático
Adicionar persistência real ao `habitrack-api` com EF Core + PostgreSQL e migrations versionadas.

---

## Fase 4 — Testes, qualidade e boas práticas

### Tópicos
- [x] xUnit (testes unitários)
- [x] Testes de integração com `WebApplicationFactory`
- [x] Mocking com Moq
- [x] SOLID aplicado a C#
- [x] Clean Code e convenções da comunidade .NET

### Projeto prático
Cobrir o `habitrack-api` com testes unitários + teste de integração end-to-end do fluxo de autenticação.

---

## Fase 5 — Arquitetura de software

### Tópicos
- [x] Clean Architecture / Arquitetura em camadas (Domain, Application, Infrastructure, API)
- [x] Domain-Driven Design: conceitos táticos (Entities, Value Objects, Aggregates)
- [x] CQRS (Command Query Responsibility Segregation)
- [x] MediatR (padrão Mediator)
- [x] Design Patterns: Factory, Strategy, Decorator, Repository

### Projeto prático
**`eventhub`** — sistema de venda/reserva de ingressos para eventos: cadastro de eventos, sessões com capacidade limitada, reserva de vagas, confirmação de compra. Construído com Clean Architecture + CQRS via MediatR.

---

## Fase 6 — Infraestrutura e Cloud

### Tópicos
- [x] Docker (containerizar API .NET)
- [x] Kubernetes (conceitos básicos: pods, deployments, services)
- [x] CI/CD (GitHub Actions)
- [x] Azure fundamentals (App Service, Azure SQL, Key Vault)
- [x] Health checks e observabilidade (OpenTelemetry)
- [x] Variáveis de ambiente e secrets management

### Projeto prático
Dockerizar o `eventhub` (Dockerfile + docker-compose) e criar pipeline de CI.

---

## Fase 7 — Sistemas distribuídos e mensageria

### Tópicos
- [x] Microsserviços: quando faz sentido e quando é over-engineering
- [x] Mensageria assíncrona (RabbitMQ)
- [x] Padrões de resiliência: Circuit Breaker, Retry (Polly)
- [x] Cache distribuído com Redis
- [x] gRPC como alternativa a REST
- [x] Idempotência e consistência eventual

### Projeto prático
Quebrar o `eventhub` em 2 serviços comunicando via RabbitMQ (MassTransit): reservas publica `ReservaConfirmada`, notificador consome e simula envio de email/SMS. Adicionar Redis para cache de disponibilidade.

---

## Fase 8 — Performance e produção

### Tópicos
- [x] Profiling (`dotnet-trace`, `dotnet-counters`)
- [x] Garbage Collector do .NET (gerações, alocação, modos)
- [x] Otimização de queries EF Core (N+1, projeções, paginação)
- [x] Rate limiting e throttling
- [x] Segurança (OWASP Top 10, JWT refresh tokens, CORS)

### Projeto prático
**`api-performance-lab`** — API satélite com endpoints problemáticos (N+1, sem paginação, queries ineficientes) para benchmark e documentação de otimizações com métricas de antes/depois.

---

## Projetos

| Projeto | Fases | Descrição |
|---|---|---|
| `csharp-katas` | 1 | Estruturas de dados + LINQ, console app com xUnit |
| `habitrack-api` | 2-4 | API de hábitos com ASP.NET Core + EF Core + JWT |
| `eventhub` | 5-8 | Sistema de vendas de ingressos (Clean Architecture + CQRS + mensageria) |
| `api-performance-lab` | 8 | Satélite para experimentos de performance |
