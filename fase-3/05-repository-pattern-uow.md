# 05 — Repository Pattern e Unit of Work

## Por que este tópico existe

Se você abrir uma vaga de C# .NET pleno/sênior agora, a probabilidade de encontrar "Repository Pattern" nos requisitos é altíssima. Junto com Dependency Injection, é um dos padrões mais onipresentes no ecossistema .NET corporativo — mesmo quando o projeto não precisaria dele.

Isso cria uma situação peculiar: o padrão é tão difundido que virou uma espécie de "custo de entrada" cultural. Você pode discordar dele, mas precisa saber implementá-lo e, principalmente, **argumentar sobre ele em entrevista**. Um marcador claro de senioridade em .NET é conseguir discutir quando Repository Pattern faz sentido e quando é complexidade desnecessária.

---

## O problema que o Repository Pattern resolve

Imagine um serviço que acessa o banco diretamente:

```csharp
public class HabitService
{
    private readonly AppDbContext _db;

    public async Task<List<Habit>> GetActiveHabits(Guid userId)
    {
        return await _db.Habits
            .Where(h => h.UserId == userId && h.IsActive)
            .Include(h => h.Completions)
            .ToListAsync();
    }
}
```

Isso funciona. Mas três problemas surgem quando o sistema cresce:

### Problema 1: Lógica de query espalhada

A regra "hábitos ativos de um usuário, com suas completions" pode ser necessária em 5 lugares diferentes: endpoint de listagem, dashboard de estatísticas, exportação de dados, job de notificação, etc. Sem repositório, você repete `.Where(h => h.UserId == userId && h.IsActive).Include(...)` em cada um desses lugares. Se amanhã a definição de "ativo" mudar para incluir `EndedAt == null`, você precisa caçar todas as ocorrências.

### Problema 2: Testabilidade

Para testar `HabitService`, você precisa de um banco de dados real ou de um provider de EF Core in-memory (que tem comportamentos diferentes do banco real — `Include` funciona diferente no in-memory, constraints de unique não são aplicadas, etc.). Com uma interface `IHabitRepository`, você pode mockar o repositório e testar o service isoladamente.

### Problema 3: Acoplamento direto ao EF Core

Se o serviço depende de `AppDbContext`, ele está acoplado ao Entity Framework. Numa aplicação pequena, trocar de ORM é improvável. Mas o acoplamento vai além da troca de ORM: você não consegue testar o serviço sem um `DbContext` configurado, e qualquer mudança no schema do banco impacta diretamente os testes do serviço.

O Repository Pattern resolve esses três problemas com uma camada de abstração entre a lógica de negócio e o acesso a dados.

---

## Como o Repository Pattern funciona

A ideia central é simples: **cada agregado/entidade tem um repositório que encapsula todo o acesso a dados relacionado a ela**. O serviço depende de uma interface, não de uma implementação concreta.

O repositório tem dois sabores principais no mundo .NET:

### Sabor 1: Repository Genérico

Uma interface genérica que funciona para qualquer entidade:

```csharp
public interface IGenericRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Delete(T entity);
}
```

**Prós**: menos código (uma implementação serve para todas as entidades), rápido de fazer CRUD simples.

**Contras**: não encapsula queries específicas de negócio. Para qualquer query além de `GetById`, o serviço precisa usar `GetAllAsync().Where(...)` — o que joga a lógica de query de volta para o serviço e perde a graça do padrão.

### Sabor 2: Repository Específico

Uma interface por agregado, com métodos que expressam intenções de negócio:

```csharp
public interface IHabitRepository
{
    Task<Habit?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<Habit>> GetActiveByUserAsync(Guid userId, CancellationToken ct = default);
    Task<IEnumerable<Habit>> GetWithCompletionsAsync(Guid userId, CancellationToken ct = default);
    Task<bool> HasCompletionOnDateAsync(Guid habitId, DateOnly date, CancellationToken ct = default);
    void Add(Habit habit);
    void Update(Habit habit);
    void Delete(Habit habit);
}
```

**Prós**: cada método expressa uma intenção de negócio. `GetActiveByUserAsync` é autoexplicativo. Se a definição de "ativo" mudar, o repositório é o único lugar que muda.

**Contras**: mais código, mais interfaces. Para 10 entidades, são 10 interfaces + 10 implementações. Pode ser burocrático demais para CRUD simples.

### Qual usar?

A resposta padrão na comunidade .NET: **repository específico para agregados com regras de negócio, repository genérico para entidades de suporte/lookup que são CRUD puro.** Na prática, muitos projetos usam o genérico porque é mais rápido, e aceitam a perda de expressividade. Em projetos que seguem DDD, o específico é obrigatório — o repositório é parte do domínio, não da infraestrutura.

---

## Repository + EF Core: a controvérsia

Aqui está o ponto mais importante deste tópico: **o EF Core já implementa os padrões Repository e Unit of Work nativamente.**

- `DbSet<T>` **é** um repositório genérico — tem `Add`, `Find`, `Remove`, e queries via LINQ.
- `DbContext` **é** uma Unit of Work — `SaveChangesAsync()` persiste todas as mudanças rastreadas pelo Change Tracker em uma transação.

Isso gera a pergunta óbvia: **por que criar uma camada de abstração em cima de uma abstração?**

A resposta não é técnica — é **contextual**:

| Contexto | Repository faz sentido? |
|---|---|
| Projeto solo ou equipe pequena (1-3 devs) | Provavelmente não. O `DbContext` já é suficiente. |
| API CRUD simples (cria/lê/atualiza/deleta sem regras complexas) | Não. Repository genérico seria redundante. |
| Domínio rico com queries complexas e reutilizáveis | Sim. Evita duplicação de lógica de query. |
| Múltiplas fontes de dados (EF Core + Dapper + Redis) | Sim. O serviço não precisa saber de onde o dado vem. |
| Equipe grande com alta rotatividade | Sim. A interface serve como contrato claro do que pode ser feito. |
| Preciso de testes unitários realmente isolados do banco | Sim. Mockar `IHabitRepository` é trivial; mockar `DbSet<T>` é um pesadelo. |
| Entrevista de emprego | Sim. Vão te perguntar sobre isso. |

### O argumento contra (que você precisa conhecer)

Rob Conery, David Fowler (arquiteto do ASP.NET Core) e outros nomes do ecossistema .NET já defenderam publicamente que **Repository Pattern em cima de EF Core é desnecessário na maioria dos casos**. O argumento deles:

1. EF Core já abstrai o banco. Adicionar outra camada não reduz acoplamento real — você continua acoplado ao EF Core porque o repositório internamente usa `DbContext`.
2. `DbSet<T>` já implementa `IQueryable<T>`, o que permite compor queries dinamicamente. Um repositório com métodos fixos perde essa flexibilidade.
3. Mocking de `DbSet<T>` é chato, mas bibliotecas como `MockQueryable` ou o provider in-memory resolvem isso.

**Meu take**: essa discussão é saudável, mas na prática do mercado .NET brasileiro, o Repository Pattern ainda é esperado. Saber os dois lados do argumento é o que te diferencia numa entrevista.

---

## Unit of Work

### O problema que resolve

Sem Unit of Work, cada repositório salva suas mudanças independentemente:

```csharp
habitRepository.Add(newHabit);
await habitRepository.SaveChangesAsync(); // COMMIT 1

completionRepository.Add(newCompletion);
await completionRepository.SaveChangesAsync(); // COMMIT 2
```

Se o segundo `SaveChangesAsync` falhar, o primeiro já foi commitado. O hábito fica órfão sem completion, ou pior, o estado do sistema fica inconsistente.

### Como funciona

Unit of Work coordena múltiplos repositórios para que todas as operações sejam persistidas em uma única transação:

```csharp
public interface IUnitOfWork
{
    IHabitRepository Habits { get; }
    IHabitCompletionRepository Completions { get; }
    IUserRepository Users { get; }

    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

O serviço usa a UOW em vez de repositórios individuais:

```csharp
public async Task MarkHabitComplete(Guid habitId, Guid userId, DateOnly date)
{
    var habit = await _uow.Habits.GetByIdAsync(habitId);
    if (habit is null) throw new NotFoundException();

    var completion = new HabitCompletion
    {
        Id = Guid.NewGuid(),
        HabitId = habitId,
        UserId = userId,
        Date = date,
        CompletedAt = DateTime.UtcNow
    };

    _uow.Completions.Add(completion);
    await _uow.SaveChangesAsync(); // Único COMMIT — hábito e completion juntos
}
```

Todas as operações do Change Tracker (`_db.Habits.Add`, `_db.Completions.Add`, updates, deletes) são persistidas juntas quando `SaveChangesAsync()` é chamado uma única vez.

### Como a "mágica" funciona

O segredo está em como o `DbContext` é compartilhado:

```csharp
// Program.cs — registra DbContext como Scoped
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString));

// DI, por padrão, registra DbContext como Scoped
// Scoped = uma instância por requisição HTTP
```

Cada requisição HTTP recebe **uma** instância de `AppDbContext`. Todos os repositórios dentro dessa requisição compartilham o **mesmo** DbContext, e portanto o **mesmo** Change Tracker. Quando `SaveChangesAsync` é chamado em qualquer um deles, ele persiste tudo que foi rastreado.

```
Requisição HTTP 1                         Requisição HTTP 2
┌─────────────────────┐                  ┌─────────────────────┐
│    DbContext #1      │                  │    DbContext #2      │
│  ┌─────────────────┐ │                  │  ┌─────────────────┐ │
│  │ Change Tracker   │ │                  │  │ Change Tracker   │ │
│  │  Habit (Added)   │ │                  │  │ User (Modified)  │ │
│  │  Completion      │ │                  │  │ Log (Added)      │ │
│  │  (Added)        │ │                  │  │                  │ │
│  └─────────────────┘ │                  │  └─────────────────┘ │
└─────────────────────┘                  └─────────────────────┘
```

Esse comportamento de "uma instância por requisição" é o que faz a Unit of Work funcionar de forma transparente no ASP.NET Core. Você não precisa de nenhuma infraestrutura extra — o DI container já gerencia o ciclo de vida do DbContext automaticamente.

---

## Implementação prática: habitrack-api

Vamos implementar Repository + Unit of Work no projeto real, substituindo os dicionários em memória por EF Core com persistência real.

### Estrutura que vamos criar

```
fase-2/habitrack-api/
├── Data/
│   ├── AppDbContext.cs
│   └── Migrations/
├── Repositories/
│   ├── IHabitRepository.cs
│   ├── HabitRepository.cs
│   ├── IUserRepository.cs
│   └── UserRepository.cs
├── UnitOfWork/
│   ├── IUnitOfWork.cs
│   └── UnitOfWork.cs
├── Services.cs          (modificado — usa IUnitOfWork)
├── Models.cs            (modificado — adiciona configuração EF)
├── Program.cs           (modificado — registra DI)
└── ...
```

### Passo 1: Instalar os pacotes

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet tool install --global dotnet-ef   # se ainda não tiver
```

### Passo 2: DbContext

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;

namespace Habitrack.Api.Data;

public class AppDbContext : DbContext
{
    public DbSet<Habit> Habits => Set<Habit>();
    public DbSet<HabitCompletion> HabitCompletions => Set<HabitCompletion>();
    public DbSet<User> Users => Set<User>();

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Habit>(entity =>
        {
            entity.HasKey(h => h.Id);
            entity.Property(h => h.Name).HasMaxLength(100).IsRequired();
            entity.Property(h => h.Color).HasMaxLength(7).HasDefaultValue("#3B82F6");
            entity.HasIndex(h => h.UserId);

            entity.HasMany(h => h.Completions)
                  .WithOne(c => c.Habit)
                  .HasForeignKey(c => c.HabitId)
                  .OnDelete(DeleteBehavior.Cascade);
        });

        modelBuilder.Entity<HabitCompletion>(entity =>
        {
            entity.HasKey(c => c.Id);
            entity.HasIndex(c => new { c.HabitId, c.Date }).IsUnique();
        });

        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(u => u.Id);
            entity.HasIndex(u => u.Email).IsUnique();
            entity.Property(u => u.Email).HasMaxLength(255).IsRequired();
        });
    }
}
```

### Passo 3: Interfaces de repositório

```csharp
// Repositories/IHabitRepository.cs
namespace Habitrack.Api.Repositories;

public interface IHabitRepository
{
    Task<IEnumerable<Habit>> GetActiveByUserAsync(Guid userId, CancellationToken ct = default);
    Task<Habit?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<Habit>> GetWithCompletionsAsync(Guid userId, CancellationToken ct = default);
    Task<bool> HasCompletionOnDateAsync(Guid habitId, DateOnly date, CancellationToken ct = default);
    void Add(Habit habit);
    void Update(Habit habit);
    void Delete(Habit habit);
}
```

```csharp
// Repositories/IUserRepository.cs
namespace Habitrack.Api.Repositories;

public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<User?> GetByEmailAsync(string email, CancellationToken ct = default);
    void Add(User user);
}
```

### Passo 4: Implementações dos repositórios

```csharp
// Repositories/HabitRepository.cs
using Habitrack.Api.Data;
using Microsoft.EntityFrameworkCore;

namespace Habitrack.Api.Repositories;

public class HabitRepository : IHabitRepository
{
    private readonly AppDbContext _db;

    public HabitRepository(AppDbContext db)
    {
        _db = db;
    }

    public async Task<IEnumerable<Habit>> GetActiveByUserAsync(Guid userId, CancellationToken ct = default)
    {
        return await _db.Habits
            .Where(h => h.UserId == userId && h.IsActive)
            .OrderByDescending(h => h.CreatedAt)
            .ToListAsync(ct);
    }

    public async Task<Habit?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        return await _db.Habits.FindAsync([id], ct);
    }

    public async Task<IEnumerable<Habit>> GetWithCompletionsAsync(Guid userId, CancellationToken ct = default)
    {
        return await _db.Habits
            .Where(h => h.UserId == userId && h.IsActive)
            .Include(h => h.Completions)
            .AsSplitQuery()
            .ToListAsync(ct);
    }

    public async Task<bool> HasCompletionOnDateAsync(Guid habitId, DateOnly date, CancellationToken ct = default)
    {
        return await _db.HabitCompletions
            .AnyAsync(c => c.HabitId == habitId && c.Date == date, ct);
    }

    public void Add(Habit habit) => _db.Habits.Add(habit);
    public void Update(Habit habit) => _db.Habits.Update(habit);
    public void Delete(Habit habit) => _db.Habits.Remove(habit);
}
```

```csharp
// Repositories/UserRepository.cs
using Habitrack.Api.Data;
using Microsoft.EntityFrameworkCore;

namespace Habitrack.Api.Repositories;

public class UserRepository : IUserRepository
{
    private readonly AppDbContext _db;

    public UserRepository(AppDbContext db)
    {
        _db = db;
    }

    public async Task<User?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        return await _db.Users.FindAsync([id], ct);
    }

    public async Task<User?> GetByEmailAsync(string email, CancellationToken ct = default)
    {
        return await _db.Users
            .FirstOrDefaultAsync(u => u.Email == email, ct);
    }

    public void Add(User user) => _db.Users.Add(user);
}
```

### Passo 5: Unit of Work

```csharp
// UnitOfWork/IUnitOfWork.cs
namespace Habitrack.Api.UnitOfWork;

public interface IUnitOfWork : IDisposable
{
    IHabitRepository Habits { get; }
    IUserRepository Users { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

```csharp
// UnitOfWork/UnitOfWork.cs
using Habitrack.Api.Data;
using Habitrack.Api.Repositories;

namespace Habitrack.Api.UnitOfWork;

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _db;
    private IHabitRepository? _habits;
    private IUserRepository? _users;

    public UnitOfWork(AppDbContext db)
    {
        _db = db;
    }

    public IHabitRepository Habits =>
        _habits ??= new HabitRepository(_db);

    public IUserRepository Users =>
        _users ??= new UserRepository(_db);

    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        return await _db.SaveChangesAsync(ct);
    }

    public void Dispose()
    {
        _db.Dispose();
    }
}
```

Os repositórios são criados com **lazy initialization** (`??=`). Isso significa que se a requisição só usa `Habits`, o `UserRepository` nunca é instanciado. Economiza alocação e mantém a simplicidade.

**Por que não injetar os repositórios diretamente na Unit of Work via DI?** Duas razões:

1. **Simplicidade**: cada repositório precisaria receber o DbContext via injeção, e a UOW precisaria receber todos os repositórios. Vira uma árvore de dependências desnecessária para algo que é essencialmente um wrapper.
2. **Garantia de mesmo DbContext**: se cada repositório fosse registrado como Scoped no DI, ele receberia o DbContext da requisição correta. Funcionaria tecnicamente, mas a lazy init torna explícito que os repositórios compartilham o mesmo DbContext, e o código fica mais direto.

### Passo 6: Registrar no DI

```csharp
// Program.cs (trecho)
var connectionString = builder.Configuration.GetConnectionString("Default");

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString));

builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Substitui os Singletons antigos:
// builder.Services.AddSingleton<HabitService>();
// builder.Services.AddSingleton<AuthService>();
builder.Services.AddScoped<HabitService>();
builder.Services.AddScoped<AuthService>();
```

### Passo 7: Adaptar os serviços

```csharp
// Services.cs — HabitService agora usa IUnitOfWork
public class HabitService
{
    private readonly IUnitOfWork _uow;

    public HabitService(IUnitOfWork uow)
    {
        _uow = uow;
    }

    public async Task<HabitDto> CreateAsync(CreateHabitRequest request, Guid userId)
    {
        var habit = new Habit
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Name = request.Name,
            Description = request.Description,
            WeeklyFrequency = request.WeeklyFrequency,
            Color = request.Color ?? "#3B82F6",
            CreatedAt = DateTime.UtcNow,
            IsActive = true
        };

        _uow.Habits.Add(habit);
        await _uow.SaveChangesAsync();

        return habit.ToDto();
    }

    public async Task<IEnumerable<HabitDto>> GetAllAsync(Guid userId)
    {
        var habits = await _uow.Habits.GetActiveByUserAsync(userId);
        return habits.Select(h => h.ToDto());
    }

    public async Task<HabitDto?> GetByIdAsync(Guid id, Guid userId)
    {
        var habit = await _uow.Habits.GetByIdAsync(id);
        if (habit is null || habit.UserId != userId)
            return null;

        return habit.ToDto();
    }

    // ... outros métodos seguem o mesmo padrão
}
```

### Passo 8: Configurar connection string + aplicar migrations

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=habitrack;Username=postgres;Password=postgres"
  }
}
```

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

---

## Trade-offs e decisões de design

### IQueryable nos repositórios: devolvo ou não?

Uma decisão polêmica no design de repositórios .NET: devolver `IQueryable<T>` ou coleções materializadas (`List<T>`, `IEnumerable<T>`)?

```csharp
// Opção A: IQueryable (flexível, mas vaza abstração)
public interface IHabitRepository
{
    IQueryable<Habit> GetActiveByUser(Guid userId);
}
// O serviço pode compor: repo.GetActiveByUser(id).Where(h => h.WeeklyFrequency > 3)

// Opção B: Coleção materializada (rígida, mas encapsulada)
public interface IHabitRepository
{
    Task<IEnumerable<Habit>> GetActiveByUserAsync(Guid userId, CancellationToken ct);
}
// O serviço recebe os dados já do banco, processa em memória
```

**Argumento a favor de `IQueryable`**: permite que o serviço componha queries adicionais que são traduzidas para SQL. Se o serviço precisa de `GetActiveByUser` com filtro adicional de `WeeklyFrequency`, com `IQueryable` o filtro vira SQL. Com `IEnumerable`, o filtro é aplicado em memória depois de carregar todos os registros.

**Argumento contra `IQueryable`**: vaza a abstração. O serviço sabe que está falando com um banco SQL que entende LINQ. Se você trocar o EF Core por Dapper ou por uma API HTTP externa, `IQueryable` não funciona mais. Você perdeu o propósito original do repositório (abstrair a fonte de dados).

**Recomendação prática**: use coleções materializadas (`Task<IEnumerable<T>>`) com parâmetros que expressem a intenção. Se precisar de queries dinâmicas e complexas, use o **Specification Pattern** (uma alternativa mais expressiva que `IQueryable` genérico).

```csharp
// Specification Pattern — alternativa a IQueryable
public interface IHabitRepository
{
    Task<IEnumerable<Habit>> FindAsync(Specification<Habit> spec, CancellationToken ct = default);
}

// Uso:
var activeWithHighFreq = await _repo.FindAsync(
    new ActiveHabitsSpec(userId).And(new HighFrequencySpec()), ct);
```

### Repositório genérico + específico: o híbrido pragmático

Muitos projetos .NET usam uma combinação:

```csharp
// Genérico para operações CRUD universais
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}

// Específico para queries de negócio
public interface IHabitRepository : IRepository<Habit>
{
    Task<IEnumerable<Habit>> GetActiveByUserAsync(Guid userId, CancellationToken ct = default);
    Task<bool> HasCompletionOnDateAsync(Guid habitId, DateOnly date, CancellationToken ct = default);
}
```

Isso resolve o "melhor dos dois mundos": operações básicas são reaproveitadas via genérico, queries de negócio ficam explícitas na interface específica. É o padrão mais comum em projetos corporativos .NET.

### Repository + Dapper

Se você usar Dapper em vez de EF Core, o Repository Pattern se torna **muito mais necessário e menos controverso**. Dapper não tem Change Tracker, nem Unit of Work nativa — você precisa implementar transações manualmente. O repositório com Dapper tipicamente recebe uma `IDbConnection` + `IDbTransaction`:

```csharp
public class HabitDapperRepository : IHabitRepository
{
    private readonly IDbConnection _conn;
    private readonly IDbTransaction _tx;

    public HabitDapperRepository(IDbConnection conn, IDbTransaction tx)
    {
        _conn = conn;
        _tx = tx;
    }

    public async Task<IEnumerable<Habit>> GetActiveByUserAsync(Guid userId)
    {
        return await _conn.QueryAsync<Habit>(
            "SELECT * FROM Habits WHERE UserId = @UserId AND IsActive = true",
            new { UserId = userId },
            transaction: _tx);
    }
}
```

Nesse caso, ninguém questiona o valor do Repository Pattern — ele é essencial para organizar queries SQL sob uma interface de negócio.

---

## Perguntas clássicas de entrevista

### "Repository Pattern em cima de EF Core não é redundante?"

**Resposta esperada**: "Depende do contexto. O EF Core já implementa Repository com `DbSet<T>` e Unit of Work com `DbContext`. Em projetos simples, adicionar outra camada é complexidade desnecessária. Mas em domínios complexos, o repositório específico encapsula queries de negócio reutilizáveis e facilita testes unitários isolados do banco. Eu avalio a complexidade do domínio antes de decidir."

### "Qual a diferença entre Repository e DAO?"

**Resposta**: "DAO (Data Access Object) é um padrão mais antigo, focado em esconder detalhes de infraestrutura de banco (qual driver, qual dialect SQL). Repository é um padrão de domínio — ele fala a linguagem do negócio. `IHabitRepository.GetActiveByUserAsync(userId)` é linguagem de negócio. `HabitDao.FindByUserIdAndStatus(userId, 'active')` é linguagem de banco."

### "Como você lida com queries complexas que não cabem em métodos de repositório?"

**Resposta**: "Três opções, em ordem de preferência: 1) Specification Pattern para queries compostas e reutilizáveis, 2) Query Objects (classes dedicadas como `GetActiveUsersReportQuery`) que recebem o DbContext diretamente — é o approach do CQRS com MediatR, 3) Em último caso, exponho `IQueryable` no repositório, mas isso vaza a abstração e só faço se as outras opções forem impraticáveis."

---

## Resumo visual: o ciclo de vida de uma requisição com Repository + UOW

```
HTTP Request
    │
    ▼
┌──────────────┐
│   Endpoint   │  Recebe IUnitOfWork via DI
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Service    │  Usa _uow.Habits.GetActiveByUserAsync(userId)
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────────┐
│  UnitOfWork  │────▶│  AppDbContext    │
│              │     │  (Scoped)        │
│  .Habits ────┼────▶│                  │
│  .Users ─────┼────▶│  Change Tracker  │
│  .Save()     │     │  ┌──────────────┐│
└──────────────┘     │  │ Habit (Added) ││
                     │  │ User (Modif) ││
                     │  └──────────────┘│
                     └────────┬─────────┘
                              │
                     SaveChangesAsync()
                              │
                              ▼
                     ┌─────────────────┐
                     │   PostgreSQL    │
                     │  BEGIN + INSERT │
                     │  + UPDATE +     │
                     │  COMMIT         │
                     └─────────────────┘
```

Uma requisição. Um DbContext. Um Change Tracker. Um `SaveChangesAsync`. Tudo orquestrado pela Unit of Work, com repositórios que encapsulam as queries específicas de cada entidade.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar por que o EF Core já implementa Repository e UOW nativamente, e mesmo assim projetos corporativos adicionam outra camada
- [ ] Argumentar quando Repository Pattern é necessário vs. quando é over-engineering
- [ ] Implementar um repositório específico com queries que expressam intenção de negócio (não só CRUD genérico)
- [ ] Explicar como o ciclo de vida Scoped do DbContext faz a Unit of Work funcionar "de graça" no ASP.NET Core
- [ ] Defender (ou criticar) o uso de `IQueryable` em interfaces de repositório com argumentos técnicos sólidos
- [ ] Saber que o Specification Pattern e Query Objects são alternativas para queries complexas que não cabem em métodos de repositório
