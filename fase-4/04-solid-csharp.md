# 04 — SOLID aplicado a C#

## Por que este tópico volta

Você já conhece os princípios SOLID de Go e do frontend. O que muda em C# não são os princípios — é **como a linguagem e o ecossistema os viabilizam**. O objetivo aqui é ver os idioms específicos: como o compilador, o type system e as convenções do .NET tornam cada princípio mais (ou menos) natural de aplicar.

---

## S — Single Responsibility Principle

> Uma classe deve ter um, e apenas um, motivo para mudar.

### Como C# facilita

**Records para DTOs**: em Go você define structs e serializa com tags. Em C#, `record` expressa intenção: "esta classe é só um saco de dados". Um record com `init`-only properties deixa explícito que ele é imutável após construção.

```csharp
// Antes (classe normal — pode virar um deus-objeto com métodos misturados)
public class CreateHabitRequest
{
    public string Name { get; set; }  // mutable, pode ser alterado em qualquer lugar
}

// Depois (record — a linguagem impõe a responsabilidade única de transporte de dados)
public record CreateHabitRequest
{
    public string Name { get; init; } // init-only: só no construtor/with
}
```

**Extension methods**: separam comportamento de dados sem poluir a classe original.

```csharp
// Models.cs — responsabilidade: modelar dados
public class Habit
{
    public Guid Id { get; init; }
    public string Name { get; set; }
    // Sem lógica de apresentação aqui
}

// Dtos.cs — responsabilidade: mapeamento
public static class HabitMappingExtensions
{
    public static HabitDto ToDto(this Habit h, IEnumerable<HabitCompletion> completions)
    {
        // Lógica de mapeamento fica aqui, não na entidade
    }
}
```

### Anti-padrão comum em C#

**"Service deus"** — uma classe `HabitService` que faz CRUD, cálculos de streak, validação de negócio, formatação de resposta e envio de email. Em projetos .NET, isso geralmente vira um arquivo de 2000 linhas.

**Solução no ecossistema .NET**: CQRS + MediatR (Fase 5). Cada handler é uma classe com responsabilidade única. Mas mesmo sem MediatR, você pode quebrar o service:

```csharp
// Em vez de um HabitService monolítico:
HabitService → HabitCrudService + StreakCalculatorService + HabitNotificationService
```

---

## O — Open/Closed Principle

> Aberto para extensão, fechado para modificação.

### Como C# facilita

**Interfaces + DI**: o sistema de DI do ASP.NET Core é o mecanismo mais natural de OCP em C#. Você define uma interface, implementa uma vez, e pode trocar a implementação sem modificar o código que depende dela.

No nosso projeto, isso já está aplicado: `HabitService` depende de `IUnitOfWork`, não de `UnitOfWork` concreta. Se amanhã você quiser trocar EF Core por Dapper, cria `DapperUnitOfWork : IUnitOfWork` e registra no DI. O `HabitService` não muda uma linha.

```csharp
// Fechado para modificação — HabitService não muda
public class HabitService
{
    private readonly IUnitOfWork _uow;
    public HabitService(IUnitOfWork uow) => _uow = uow;
}

// Aberto para extensão — novas implementações da interface
public class EfUnitOfWork : IUnitOfWork { ... }
public class DapperUnitOfWork : IUnitOfWork { ... }  // nova, sem mexer no serviço
```

**Pipeline de middleware**: outro exemplo de OCP no ASP.NET Core. Cada middleware é uma extensão do pipeline. Você adiciona novos comportamentos (autenticação, logging, CORS) sem modificar os existentes.

### Quando OCP é over-engineering

Criar uma interface `IHabitService` quando você tem **uma única implementação** e nenhuma previsão de segunda implementação. Isso é "interface-ite" — um sintoma comum em projetos .NET corporativos. A interface só se justifica se:
1. Você precisa mockar o serviço em testes (mas você mocka `IUnitOfWork`, não o serviço)
2. Há duas implementações reais concorrentes
3. A interface é um contrato de biblioteca pública (NuGet package)

---

## L — Liskov Substitution Principle

> Subtipos devem poder substituir seus tipos base sem quebrar o programa.

### Como C# facilita

**Nullable reference types**: o compilador avisa quando uma subclasse viola expectativas de nullability.

```csharp
public class BaseRepository
{
    public virtual Habit? GetById(Guid id) => null; // pode retornar null
}

public class CachedRepository : BaseRepository
{
    // Violação de LSP: a subclasse promete nunca retornar null, mas a base permite
    public override Habit GetById(Guid id) => throw new NotImplementedException();
    //                                    ^^^ deveria ser Habit? mas o compilador deixa...
}
```

Com nullable enabled, você teria um warning se o tipo de retorno não bater. Mas a violação mais comum de LSP em C# é **lançar `NotImplementedException` em métodos herdados de interfaces** — sinal de que a interface é grande demais (ferindo ISP também).

### O problema da herança em C# corporativo

É comum ver:

```csharp
public class BaseService
{
    protected readonly AppDbContext _db;
    // Assume que todo serviço precisa de DbContext
}

public class ReportService : BaseService
{
    // Mas ReportService só lê dados, nunca escreve
    // Herdou Update e Delete sem querer
}
```

**Alternativa idiomática em C#**: composição via DI em vez de herança. Em vez de `BaseService`, injete `AppDbContext` (ou melhor, `IUnitOfWork`) diretamente em cada serviço — que é exatamente o que fizemos no habitrack-api.

---

## I — Interface Segregation Principle

> Uma classe não deve ser forçada a implementar métodos que não usa.

### Como C# facilita

Interfaces pequenas e focadas são naturais em C# com DI. Cada interface representa um contrato enxuto:

```csharp
// Ruim: interface monolítica
public interface IRepository<T>
{
    Task<T?> GetById(Guid id);
    Task<IEnumerable<T>> GetAll();
    Task Add(T entity);
    Task Update(T entity);
    Task Delete(T entity);
    Task SaveChanges();  // ISP violation: repositório não deveria saber de transação
}

// Bom: interfaces segregadas
public interface IHabitReader
{
    Task<Habit?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IEnumerable<Habit>> GetActiveByUserAsync(Guid userId, CancellationToken ct);
}

public interface IHabitWriter
{
    void Add(Habit habit);
    void Update(Habit habit);
}

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

// Serviço de leitura (dashboard) depende só de IHabitReader
// Serviço de escrita (CRUD) depende de IHabitReader + IHabitWriter + IUnitOfWork
// Cada um recebe só o que precisa
```

Essa segregação segue o **CQRS** (Command Query Responsibility Segregation) que é aprofundado na Fase 5. A ideia é a mesma do ISP: comandos e consultas são interfaces diferentes porque têm necessidades diferentes.

---

## D — Dependency Inversion Principle

> Dependa de abstrações, não de implementações concretas.

### Como C# facilita

O sistema de DI do ASP.NET Core é a personificação do DIP. O container resolve dependências automaticamente:

```csharp
// Program.cs — registra abstração → implementação
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<HabitService>();

// HabitService.cs — depende da abstração, não da implementação
public class HabitService
{
    private readonly IUnitOfWork _uow; // IUnitOfWork, não UnitOfWork

    public HabitService(IUnitOfWork uow) => _uow = uow;
}
```

Três níveis de maturidade do DIP em projetos .NET:

### Nível 1: Injeção direta do DbContext (júnior)

```csharp
public class HabitService
{
    private readonly AppDbContext _db; // dependência concreta
}
```

Funciona, mas o serviço está acoplado ao EF Core. Testar requer banco real ou in-memory provider.

### Nível 2: Interface de repositório (pleno)

```csharp
public class HabitService
{
    private readonly IHabitRepository _repo; // abstração
}
```

Testável, mas cada repositório salva independentemente — sem transação coordenada.

### Nível 3: Unit of Work (sênior)

```csharp
public class HabitService
{
    private readonly IUnitOfWork _uow; // abstração que coordena múltiplos repositórios

    public async Task CreateWithCompletion(...)
    {
        _uow.Habits.Add(habit);
        _uow.Habits.AddCompletion(completion);
        await _uow.SaveChangesAsync(); // uma transação
    }
}
```

Nós já implementamos o nível 3 no habitrack-api.

---

## SOLID não é um checklist — é um sistema

Aplicar SOLID isoladamente gera código fragmentado. Aplicar em conjunto gera coesão:

```
SRP: cada classe tem uma razão para mudar
  ↓
OCP: interfaces permitem estender sem modificar
  ↓
LSP: implementações são substituíveis
  ↓
ISP: interfaces são enxutas (quem usa IHabitReader não precisa de IHabitWriter)
  ↓
DIP: tudo depende de abstrações, injetadas via DI
```

O ecossistema .NET torna esse ciclo natural: DI + interfaces + extension methods + records são os idioms que materializam SOLID sem atrito.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar como `record` com `init` ajuda no SRP separando DTOs de lógica
- [ ] Mostrar como DI + interfaces implementam OCP sem esforço extra
- [ ] Identificar uma violação de LSP comum: `throw new NotImplementedException()` em métodos de interface
- [ ] Propor segregação de uma interface monolítica `IRepository<T>` em `IReader<T>` + `IWriter<T>`
- [ ] Diferenciar os 3 níveis de DIP em .NET: DbContext direto → Repository → Unit of Work
- [ ] Argumentar quando criar uma interface `IService` é necessário vs. "interface-ite"
