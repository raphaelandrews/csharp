# 01 — Clean Architecture no ecossistema .NET

## O problema que a Clean Architecture resolve

Toda aplicação que cresce além de um CRUD simples enfrenta o mesmo problema: o acoplamento entre regras de negócio e infraestrutura torna o código frágil. Trocar o banco de dados, mudar o framework HTTP ou adicionar um cache distribuído vira uma cirurgia de código — e pior, cada mudança de infraestrutura pode quebrar regras de negócio que não têm nada a ver com ela.

A Clean Architecture (proposta por Robert C. Martin em 2012) não é um framework nem uma estrutura de pastas — é um **princípio de organização**: as regras de negócio ficam no centro e não dependem de nada externo. A infraestrutura fica nas bordas e depende do centro, nunca o contrário.

### O diagrama clássico (traduzido para .NET)

```
┌─────────────────────────────────────────────────┐
│                  API / Presentation              │
│         (Minimal API, Controllers, DTOs)         │
│              Depende de: Application             │
├─────────────────────────────────────────────────┤
│                  Application                     │
│     (Casos de uso, Commands, Queries, Ports)     │
│          Depende de: Domain                      │
├─────────────────────────────────────────────────┤
│                    Domain                        │
│  (Entities, Value Objects, Aggregates, Domain    │
│   Events, Repository Interfaces)                 │
│          NÃO depende de nada                     │
├─────────────────────────────────────────────────┤
│                 Infrastructure                   │
│  (EF Core, Dapper, Email, Cache, File Storage)   │
│     Implementa interfaces do Domain/Application  │
│          Depende de: Domain, Application         │
└─────────────────────────────────────────────────┘
```

A seta de dependência sempre aponta **para dentro**. O Domain não conhece o Entity Framework. O Application não conhece o ASP.NET Core. A Infrastructure implementa interfaces definidas pelo Domain e pelo Application.

### O que isso resolve no mundo real

Sem Clean Architecture, o `HabitService` que construímos na Fase 3 depende diretamente de `IUnitOfWork` que depende de `AppDbContext` que depende de `Npgsql`. Se amanhã você quiser adicionar cache com Redis, precisa modificar o `HabitService`. Se quiser trocar PostgreSQL por SQL Server, precisa mexer em `AppDbContext` e rezar para nada quebrar.

Com Clean Architecture, o fluxo seria:

```
API recebe request
    → Application: CreateHabitCommand
        → Domain: Habit.Create(name, frequency) ← regra de negócio pura, sem EF Core
        → Application: IHabitRepository.Add(habit) ← interface definida no Domain
            → Infrastructure: HabitRepository.Add(habit) ← implementação com EF Core
```

A regra de negócio (`Habit.Create`) está no Domain e não sabe que banco de dados existe. O repositório é uma interface no Domain e uma implementação com EF Core na Infrastructure.

---

## As 4 camadas no .NET

### Camada 1: Domain (o coração)

**O que contém**: Entities, Value Objects, Aggregates, Domain Events, interfaces de repositório, exceções de domínio.

**O que NÃO contém**: referência a EF Core, ASP.NET, System.Net.Http, ou qualquer framework externo.

**Regra de ouro**: se você abrir o `.csproj` do Domain, ele deve ter **zero** dependências de pacotes NuGet (ou no máximo pacotes de linguagem como `MediatR.Contracts` para domain events). Idealmente é um projeto `netstandard2.0` ou `net9.0` puro, sem referências.

```xml
<!-- Domain.csproj — o projeto mais enxuto -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
  </PropertyGroup>
  <!-- Zero PackageReferences. Só C# puro. -->
</Project>
```

### Camada 2: Application (os casos de uso)

**O que contém**: Commands, Queries, Handlers, DTOs, interfaces para serviços externos (IEmailSender, IPaymentGateway), Behaviors do MediatR (validação, logging).

**O que NÃO contém**: implementações concretas de infraestrutura (EF Core, HttpClient, SMTP).

**Dependências**: Domain (óbvio) + MediatR + FluentValidation + pacotes de abstração.

```xml
<!-- Application.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\Domain\Domain.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="MediatR" Version="12.x" />
    <PackageReference Include="FluentValidation" Version="12.x" />
  </ItemGroup>
</Project>
```

### Camada 3: Infrastructure (implementações concretas)

**O que contém**: DbContext, Repository implementations, serviços externos (EmailSender com SMTP, PaymentGateway com HttpClient), configuração de DI.

**Dependências**: Domain + Application + pacotes de infra (EF Core, Npgsql, Redis, etc.).

```xml
<!-- Infrastructure.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\Application\Application.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.x" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.x" />
  </ItemGroup>
</Project>
```

### Camada 4: API / Presentation

**O que contém**: Program.cs, Minimal API endpoints ou Controllers, configuração de middleware (autenticação, CORS, Swagger), `appsettings.json`.

**Dependências**: Application + Infrastructure (só pra registrar DI).

```xml
<!-- Api.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\Infrastructure\Infrastructure.csproj" />
    <ProjectReference Include="..\Application\Application.csproj" />
  </ItemGroup>
</Project>
```

A API **não** referencia o Domain diretamente — ela só conhece Application. Isso garante que a API não consegue instanciar uma entidade de domínio sem passar pelo caso de uso.

---

## A regra da dependência: código vs projeto

O princípio central: **dependências de código fonte só podem apontar para dentro**. Mas na prática do .NET, isso se traduz em dependências de projeto (ProjectReference):

```
API → Infrastructure → Application → Domain
                              ↓
                        Application → Domain
API ──────────────────────────────→ Application (direto, pra DI)
```

Note que `API → Infrastructure` é uma dependência de **configuração**, não de lógica. A API só conhece Infrastructure para registrar serviços no DI container (`builder.Services.AddInfrastructure()`). O código da API não deve usar tipos concretos da Infrastructure diretamente — sempre via interfaces definidas no Application ou Domain.

### O "truque" do DI: Inversion of Control

A Clean Architecture funciona em .NET porque o container de DI inverte o controle. A API registra implementações concretas (da Infrastructure) para interfaces (do Domain/Application):

```csharp
// Program.cs — a API conhece Infrastructure, mas só pra configurar DI
builder.Services.AddScoped<IEventRepository, EventRepository>();  // IEventRepository está no Domain
builder.Services.AddScoped<IPaymentGateway, PaymentGateway>();    // IPaymentGateway está no Application
builder.Services.AddDbContext<AppDbContext>(...);                   // AppDbContext está na Infrastructure
```

Os handlers (na Application) dependem apenas das interfaces. O DI resolve as implementações concretas em runtime. Esse é o DIP (Dependency Inversion Principle) em ação.

---

## Estrutura de pastas: o padrão do mercado .NET

Existem duas variantes comuns. Ambas são válidas — a escolha é mais sobre cultura da equipe:

### Variante A: Pastas por camada (mais comum em vagas .NET)

```
src/
├── Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Enums/
│   └── Interfaces/        ← repositórios, serviços de domínio
├── Application/
│   ├── Events/            ← commands + queries
│   │   ├── CreateEvent/
│   │   ├── GetEventById/
│   │   └── ReserveTicket/
│   ├── Common/
│   │   ├── Interfaces/    ← IEmailSender, IPaymentGateway
│   │   ├── Behaviors/     ← MediatR pipeline behaviors
│   │   └── Exceptions/
│   └── DTOs/
├── Infrastructure/
│   ├── Persistence/
│   │   ├── Repositories/
│   │   └── Migrations/
│   ├── Services/          ← EmailSender, PaymentGateway
│   └── DependencyInjection.cs
└── Api/
    ├── Endpoints/
    ├── Middleware/
    └── Program.cs
```

### Variante B: Vertical Slices (menos comum, mas crescendo)

Cada feature tem seu próprio "fatia vertical" contendo handler, DTO, e validação juntos:

```
src/
├── Domain/
│   └── (mesmo de antes)
├── Application/
│   └── Features/
│       ├── Events/
│       │   ├── CreateEvent.cs     ← request + handler + validator + response
│       │   ├── GetEventById.cs
│       │   └── ReserveTicket.cs
│       └── Notifications/
│           └── SendConfirmation.cs
├── Infrastructure/
│   └── (mesmo de antes)
└── Api/
    └── (mesmo de antes)
```

**Recomendação**: a Variante A é o que você vai encontrar em 90% das vagas e projetos corporativos. A Variante B (Vertical Slices) é defendida por Jimmy Bogard (criador do MediatR e Automapper) e está ganhando tração, mas ainda não é o padrão do mercado. Nós usaremos a **Variante A** no eventhub para alinhar com o que as vagas esperam.

---

## Trade-offs: Clean Architecture não é de graça

### O que você ganha

- **Testabilidade extrema**: o Domain pode ser testado com zero dependências externas. Cada camada é testável isoladamente.
- **Substituibilidade**: trocar EF Core por Dapper, ou PostgreSQL por SQL Server, ou REST por gRPC, requer mudanças apenas na Infrastructure e na API — Application e Domain não mudam.
- **Foco no negócio**: o Domain contém apenas regras de negócio puras, sem ruído de infraestrutura. Isso força você a pensar no domínio antes de pensar em tabelas.
- **Proteção contra frameworks**: se o ASP.NET Core for descontinuado amanhã (improvável, mas didático), você troca a camada de API sem perder lógica de negócio.

### O que você perde

- **Overhead inicial**: cada feature nova requer arquivos em múltiplos projetos (request, handler, response, validator). Para um CRUD simples, isso é burocracia pura.
- **Indireção**: um dev novo no projeto leva mais tempo para rastrear o fluxo: "onde esse comando é tratado?" → Application → Infrastructure → Domain. Em um projeto monolítico, é tudo no mesmo lugar.
- **Mapeamento**: você precisa mapear Entities do Domain → DTOs da Application → Response da API. Isso gera código boilerplate que frameworks como AutoMapper tentam resolver (mas trazem seus próprios problemas).
- **Risco de over-engineering**: para uma API com 3 endpoints e 2 tabelas, Clean Architecture é complexidade desnecessária. O critério de "pronto" da Fase 3 do nosso roadmap (EF Core direto no service) é mais adequado para sistemas pequenos.

### Quando NÃO usar Clean Architecture

- API CRUD com menos de 10 endpoints e sem regras de negócio complexas
- Protótipo ou MVP que será reescrito em 3 meses
- Microsserviço pequeno com responsabilidade única e bem definida
- Equipe de 1-2 devs onde a comunicação é direta e o código é autoexplicativo

### Regra prática

> "Se o sistema tem regras de negócio que um não-programador consegue descrever, e essas regras mudam com frequência, Clean Architecture se paga. Se o sistema é um CRUD glorificado, é over-engineering." — adaptado de Martin Fowler

Nosso **eventhub** (venda de ingressos) se encaixa no primeiro caso: reserva de vagas com capacidade limitada, concorrência, confirmação de compra, cálculo de disponibilidade — são regras de negócio reais que justificam a arquitetura.

---

## Implementação prática: eventhub

Vamos criar a estrutura do eventhub seguindo Clean Architecture:

### Estrutura de projetos

```
fase-5/eventhub/
├── EventHub.sln
├── src/
│   ├── EventHub.Domain/
│   │   ├── EventHub.Domain.csproj
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Enums/
│   │   └── Interfaces/
│   ├── EventHub.Application/
│   │   ├── EventHub.Application.csproj
│   │   ├── Events/          ← Commands e Queries
│   │   ├── Common/
│   │   └── DependencyInjection.cs
│   ├── EventHub.Infrastructure/
│   │   ├── EventHub.Infrastructure.csproj
│   │   ├── Persistence/
│   │   └── DependencyInjection.cs
│   └── EventHub.Api/
│       ├── EventHub.Api.csproj
│       ├── Endpoints/
│       └── Program.cs
└── tests/
    ├── EventHub.Domain.Tests/
    ├── EventHub.Application.Tests/
    ├── EventHub.Infrastructure.Tests/
    └── EventHub.Api.Tests/
```

### Os comandos (criação via dotnet CLI)

```bash
mkdir -p fase-5/eventhub/src
mkdir -p fase-5/eventhub/tests

# Solution
dotnet new sln -n EventHub -o fase-5/eventhub

# Domain — class library sem dependências
dotnet new classlib -n EventHub.Domain -o fase-5/eventhub/src/EventHub.Domain

# Application — class library com dependência no Domain + MediatR
dotnet new classlib -n EventHub.Application -o fase-5/eventhub/src/EventHub.Application

# Infrastructure — class library com dependências em Domain, Application, EF Core
dotnet new classlib -n EventHub.Infrastructure -o fase-5/eventhub/src/EventHub.Infrastructure

# API — projeto web
dotnet new web -n EventHub.Api -o fase-5/eventhub/src/EventHub.Api

# Adicionar ao solution
dotnet sln fase-5/eventhub/EventHub.sln add fase-5/eventhub/src/EventHub.Domain/EventHub.Domain.csproj
dotnet sln fase-5/eventhub/EventHub.sln add fase-5/eventhub/src/EventHub.Application/EventHub.Application.csproj
dotnet sln fase-5/eventhub/EventHub.sln add fase-5/eventhub/src/EventHub.Infrastructure/EventHub.Infrastructure.csproj
dotnet sln fase-5/eventhub/EventHub.sln add fase-5/eventhub/src/EventHub.Api/EventHub.Api.csproj

# Adicionar referências entre projetos
dotnet add fase-5/eventhub/src/EventHub.Application reference fase-5/eventhub/src/EventHub.Domain
dotnet add fase-5/eventhub/src/EventHub.Infrastructure reference fase-5/eventhub/src/EventHub.Application
dotnet add fase-5/eventhub/src/EventHub.Api reference fase-5/eventhub/src/EventHub.Infrastructure
dotnet add fase-5/eventhub/src/EventHub.Api reference fase-5/eventhub/src/EventHub.Application
```

### O arquivo Domain.csproj — zero dependências

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

Isso é intencional. O Domain é C# puro — sem EF Core, sem Newtonsoft.Json, sem qualquer framework. Se alguém tentar adicionar `Microsoft.EntityFrameworkCore` nesse projeto, o code review deve rejeitar.

### O Domain contém as regras de negócio

```csharp
// EventHub.Domain/Entities/Event.cs
namespace EventHub.Domain.Entities;

public class Event
{
    public Guid Id { get; private set; }
    public string Title { get; private set; }
    public string Description { get; private set; }
    public Venue Venue { get; private set; }
    public DateTime StartDate { get; private set; }
    public DateTime EndDate { get; private set; }
    public EventStatus Status { get; private set; }
    private readonly List<Session> _sessions = new();
    public IReadOnlyCollection<Session> Sessions => _sessions.AsReadOnly();

    private Event() { } // EF Core

    public static Event Create(string title, string description, Venue venue,
        DateTime start, DateTime end)
    {
        if (string.IsNullOrWhiteSpace(title))
            throw new DomainException("Title is required");

        if (end <= start)
            throw new DomainException("End date must be after start date");

        return new Event
        {
            Id = Guid.NewGuid(),
            Title = title,
            Description = description,
            Venue = venue,
            StartDate = start,
            EndDate = end,
            Status = EventStatus.Draft
        };
    }

    public void Publish()
    {
        if (Status != EventStatus.Draft)
            throw new DomainException("Only draft events can be published");

        if (_sessions.Count == 0)
            throw new DomainException("Event must have at least one session");

        Status = EventStatus.Published;
    }

    public void AddSession(Session session)
    {
        if (Status == EventStatus.Cancelled)
            throw new DomainException("Cannot add sessions to a cancelled event");

        _sessions.Add(session);
    }

    public void Cancel()
    {
        if (Status == EventStatus.Cancelled)
            throw new DomainException("Event is already cancelled");

        Status = EventStatus.Cancelled;
    }
}
```

Note o padrão: **construtor privado para EF Core** + **factory method estático para criação** + **métodos com nomes de negócio** (`Publish()`, `Cancel()`, não `SetStatus()`). Isso é Domain-Driven Design tático — a entidade encapsula suas regras de transição de estado, não é um saco de propriedades públicas com setters.

### A interface de repositório no Domain

```csharp
// EventHub.Domain/Interfaces/IEventRepository.cs
namespace EventHub.Domain.Interfaces;

public interface IEventRepository
{
    Task<Event?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<Event>> GetPublishedAsync(CancellationToken ct = default);
    void Add(Event @event);
    void Update(Event @event);
}
```

A interface está no Domain, mas a **implementação** com EF Core (`EventRepository : IEventRepository`) está na Infrastructure. O Domain não sabe que EF Core existe — ele só conhece o contrato.

---

## Resumo: o que separa "copiar estrutura de pastas" de "entender Clean Architecture"

O erro mais comum em entrevistas é descrever Clean Architecture como "você cria 4 pastas e separa o código". Isso é sintoma de quem só copiou tutorial.

O que realmente importa:
1. **A regra da dependência**: código no centro não conhece código nas bordas. Se seu Domain referencia `Microsoft.EntityFrameworkCore`, você quebrou o princípio.
2. **Entities com comportamento, não DTOs**: a entidade de domínio tem métodos que expressam regras de negócio (`Publish()`, `Cancel()`, `Reserve()`), não é um saco de getters/setters.
3. **Interfaces no Domain, implementações na Infrastructure**: `IEventRepository` está no Domain (junto com a entidade `Event`). `EventRepository` (com EF Core) está na Infrastructure. A inversão de dependência é real, não cosmética.
4. **Application como orquestrador**: um handler da Application não contém regras de negócio — ele carrega a entidade do repositório, chama o método de domínio, e persiste. A lógica de negócio fica na entidade.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Desenhar o diagrama das 4 camadas e explicar a direção das setas de dependência
- [ ] Explicar por que o Domain.csproj não deve ter referência a pacotes NuGet
- [ ] Criar uma entidade com construtor privado, factory method e métodos de negócio
- [ ] Explicar onde fica a interface do repositório e onde fica a implementação (e por que)
- [ ] Argumentar quando Clean Architecture é over-engineering vs quando ela se paga
- [ ] Diferenciar a dependência de projeto (API → Infrastructure para DI) da dependência de código (Application → Domain para lógica)
