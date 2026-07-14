# 05 — Design Patterns clássicos aplicados em C#

## Por que este tópico está na Fase 5

Design Patterns não são exclusivos do .NET, mas o ecossistema tem **implementações idiomáticas** de cada padrão que tiram proveito de recursos da linguagem que outras linguagens não têm: LINQ, delegates, extension methods, generics, e DI nativo.

Este tópico não é uma introdução aos patterns — você já os conhece conceitualmente. O foco é **como implementá-los em C# idiomático**, aproveitando o que a linguagem oferece de melhor.

---

## Factory Pattern: escondendo a complexidade de criação

### Factory Method (o pattern, não o recurso da linguagem)

A forma mais idiomática em C# não é uma classe `XxxFactory` separada — é um **método estático na própria classe**:

```csharp
// Factory Method na entidade (não numa classe separada)
public class Ticket
{
    public Guid Id { get; private set; }
    public Guid SessionId { get; private set; }
    public Guid? OwnerId { get; private set; }
    public TicketStatus Status { get; private set; }
    public Money Price { get; private set; }

    private Ticket() { } // EF Core

    public static Ticket Create(Guid sessionId, Money price)
    {
        return new Ticket
        {
            Id = Guid.NewGuid(),
            SessionId = sessionId,
            Price = price,
            Status = TicketStatus.Available
        };
    }

    public static List<Ticket> CreateBatch(Guid sessionId, int quantity, Money pricePerTicket)
    {
        return Enumerable.Range(0, quantity)
            .Select(_ => Create(sessionId, pricePerTicket))
            .ToList();
    }
}
```

Isso é mais idiomático que `TicketFactory.Create(sessionId, price)` porque:
1. A lógica de criação vive junto com a classe que ela cria
2. O construtor privado garante que ninguém cria um `Ticket` inválido (`new Ticket { Status = ... }` não compila)
3. `CreateBatch` usa LINQ para criar N tickets de forma declarativa

### Abstract Factory com DI

Para cenários onde a criação depende de configuração externa:

```csharp
// Interface no Domain/Application
public interface IPaymentGatewayFactory
{
    IPaymentGateway Create(PaymentMethod method);
}

// Implementação na Infrastructure
public class PaymentGatewayFactory : IPaymentGatewayFactory
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IConfiguration _config;

    public PaymentGatewayFactory(IHttpClientFactory httpClientFactory, IConfiguration config)
    {
        _httpClientFactory = httpClientFactory;
        _config = config;
    }

    public IPaymentGateway Create(PaymentMethod method) => method switch
    {
        PaymentMethod.CreditCard => new CreditCardGateway(_httpClientFactory, _config),
        PaymentMethod.Pix => new PixGateway(_httpClientFactory.CreateClient("pixApi")),
        _ => throw new DomainException($"Unsupported payment method: {method}")
    };
}
```

A factory é injetada via DI e cria a implementação correta baseada no método de pagamento. O handler não sabe qual gateway está usando — só depende de `IPaymentGatewayFactory`.

---

## Strategy Pattern: algoritmos intercambiáveis

Strategy brilha quando você tem variações de comportamento que dependem de contexto. Em C#, delegates e DI tornam o pattern mais leve que em Java (onde você precisa de uma interface + N classes concretas).

### Strategy via delegate (leve)

```csharp
// Strategy como delegate — sem interfaces, sem classes extras
public delegate Money PricingStrategy(Session session, DateTime purchaseDate);

// Implementações como métodos estáticos ou lambdas
public static class PricingStrategies
{
    public static Money Standard(Session s, DateTime d) => s.BasePrice;

    public static Money EarlyBird(Session s, DateTime d)
    {
        var daysUntilEvent = (s.StartTime - d).Days;
        return daysUntilEvent >= 30 ? s.BasePrice.Multiply(0.8m) : s.BasePrice;
    }

    public static Money LastMinute(Session s, DateTime d)
    {
        var occupancyRate = (decimal)s.ReservedSeats / s.Capacity;
        return occupancyRate > 0.9m ? s.BasePrice.Multiply(1.3m) : s.BasePrice;
    }
}

// Uso
public Money CalculatePrice(Session session, DateTime purchaseDate, PricingStrategy strategy)
{
    return strategy(session, purchaseDate);
}

// Chamada:
var price = CalculatePrice(session, DateTime.UtcNow, PricingStrategies.EarlyBird);
```

Quando a estratégia é simples (recebe input, retorna output), um delegate é mais idiomático que uma interface.

### Strategy via interface + DI (robusto)

Quando a estratégia tem dependências (ex: precisa chamar uma API externa para calcular preço dinâmico):

```csharp
// Interface
public interface IPricingStrategy
{
    Task<Money> CalculatePriceAsync(Session session, DateTime purchaseDate, CancellationToken ct);
}

// Implementações injetáveis
public class DynamicPricingStrategy : IPricingStrategy
{
    private readonly HttpClient _http;

    public DynamicPricingStrategy(HttpClient http) => _http = http;

    public async Task<Money> CalculatePriceAsync(Session session, DateTime purchaseDate, CancellationToken ct)
    {
        // Chama API externa de pricing dinâmico
        var response = await _http.PostAsJsonAsync("/api/pricing/calculate", new { ... }, ct);
        // ...
    }
}

// Seleção da estratégia via DI
public class PricingService
{
    private readonly IEnumerable<IPricingStrategy> _strategies;

    public PricingService(IEnumerable<IPricingStrategy> strategies)
    {
        _strategies = strategies;
    }

    public IPricingStrategy GetStrategy(EventCategory category) => category switch
    {
        EventCategory.Music => _strategies.OfType<DynamicPricingStrategy>().First(),
        EventCategory.Sports => _strategies.OfType<FixedPricingStrategy>().First(),
        _ => _strategies.OfType<StandardPricingStrategy>().First()
    };
}
```

O DI registra todas as implementações:

```csharp
builder.Services.AddScoped<IPricingStrategy, StandardPricingStrategy>();
builder.Services.AddScoped<IPricingStrategy, DynamicPricingStrategy>();
builder.Services.AddScoped<IPricingStrategy, FixedPricingStrategy>();
```

O `IEnumerable<IPricingStrategy>` injetado contém todas elas. Você seleciona a certa baseada no contexto.

---

## Decorator Pattern: adicionando comportamento sem modificar

O pattern Decorator no .NET é implementado naturalmente via DI com Scrutor ou DI puro:

### Decorator com DI (scrutor)

```csharp
// Interface
public interface IEventRepository
{
    Task<Event?> GetByIdAsync(Guid id, CancellationToken ct);
    void Add(Event @event);
}

// Implementação base (EF Core)
public class EfEventRepository : IEventRepository { ... }

// Decorator: cache
public class CachedEventRepository : IEventRepository
{
    private readonly IEventRepository _inner;
    private readonly IDistributedCache _cache;

    public CachedEventRepository(IEventRepository inner, IDistributedCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Event?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var cached = await _cache.GetStringAsync($"event:{id}", ct);
        if (cached is not null)
            return JsonSerializer.Deserialize<Event>(cached);

        var @event = await _inner.GetByIdAsync(id, ct);
        if (@event is not null)
        {
            await _cache.SetStringAsync($"event:{id}",
                JsonSerializer.Serialize(@event), ct);
        }

        return @event;
    }

    public void Add(Event @event) => _inner.Add(@event);
    // Não cacheia na escrita — só decora leitura
}

// Registro no DI com Scrutor:
builder.Services.AddScoped<IEventRepository, EfEventRepository>();
builder.Services.Decorate<IEventRepository, CachedEventRepository>();
```

O `CachedEventRepository` recebe `IEventRepository` no construtor (que será o `EfEventRepository`), adiciona cache, e delega o resto. O código que depende de `IEventRepository` não sabe que está recebendo uma versão cacheada — Liskov em ação.

### Decorator manual (sem Scrutor)

```csharp
// Sem Scrutor, você registra manualmente:
builder.Services.AddScoped<EfEventRepository>();
builder.Services.AddScoped<IEventRepository>(sp =>
    new CachedEventRepository(
        sp.GetRequiredService<EfEventRepository>(),
        sp.GetRequiredService<IDistributedCache>()
    ));
```

### MediatR Behaviors como Decorators

Os `IPipelineBehavior<T, R>` do MediatR são essencialmente decorators aplicados a handlers. Em vez de decorar um repositório, você decora o pipeline de execução:

```csharp
// Cada behavior "decora" o handler:
// LoggingBehavior → ValidationBehavior → TransactionBehavior → Handler
```

Esse é o uso mais poderoso do Decorator Pattern no ecossistema .NET moderno.

---

## Repository Pattern: o que já cobrimos

O Repository Pattern foi coberto em profundidade na Fase 3. O que muda na Fase 5 é o **contexto**: na Clean Architecture, a interface do repositório está no **Domain** (junto com a entidade) e a implementação está na **Infrastructure** (com EF Core). Isso é DIP aplicado ao acesso a dados.

```csharp
// Domain/Interfaces/IEventRepository.cs — INTERFACE (zero dependências)
public interface IEventRepository
{
    Task<Event?> GetByIdAsync(Guid id, CancellationToken ct);
    void Add(Event @event);
    void Update(Event @event);
}

// Infrastructure/Persistence/Repositories/EventRepository.cs — IMPLEMENTAÇÃO
public class EventRepository : IEventRepository
{
    private readonly AppDbContext _db;

    public EventRepository(AppDbContext db) => _db = db;

    public async Task<Event?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        return await _db.Events
            .Include(e => e.Sessions).ThenInclude(s => s.Tickets)
            .FirstOrDefaultAsync(e => e.Id == id, ct);
    }

    public void Add(Event @event) => _db.Events.Add(@event);
    public void Update(Event @event) => _db.Events.Update(@event);
}
```

Esse é o Repository Pattern no contexto de DDD + Clean Architecture: a interface vive no coração do sistema, a implementação vive na borda.

---

## Resumo: quando usar cada pattern no eventhub

| Pattern | Onde usar no eventhub | Exemplo |
|---|---|---|
| **Factory Method** | Criação de entidades (`Event.Create()`, `Ticket.CreateBatch()`) | `Event.Create(title, desc, orgId)` |
| **Abstract Factory** | Criação de gateways de pagamento | `IPaymentGatewayFactory.Create(PaymentMethod.Pix)` |
| **Strategy** | Estratégias de precificação | `EarlyBirdStrategy`, `DynamicPricingStrategy` |
| **Decorator** | Cache de repositório, pipeline de behaviors | `CachedEventRepository` decorando `EfEventRepository` |
| **Repository** | Abstração de acesso a dados | `IEventRepository` no Domain, `EventRepository` na Infrastructure |

---

## O que NÃO implementar (só porque está no livro do GoF)

Alguns patterns são over-engineering no contexto .NET moderno:

- **Singleton**: o DI container já gerencia ciclo de vida (`AddSingleton<T>()`). Não implemente `GetInstance()` manualmente.
- **Builder**: para objetos complexos, prefira factory methods ou `record` com `with` expressions em vez de `EventBuilder.SetTitle().SetDate().Build()`.
- **Chain of Responsibility**: o middleware pipeline do ASP.NET Core e os behaviors do MediatR já implementam isso. Não recrie.
- **Observer**: eventos C# (`event`, `EventHandler<T>`) e `INotification` do MediatR já cobrem isso.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Implementar Factory Method como método estático na própria entidade (não em classe separada)
- [ ] Usar delegates para Strategy simples e interfaces + DI para Strategy com dependências
- [ ] Aplicar Decorator via `builder.Services.Decorate<T>()` (Scrutor) ou via MediatR Pipeline Behaviors
- [ ] Reconhecer que Singleton, Chain of Responsibility e Observer já são implementados pelo próprio ecossistema .NET
- [ ] Explicar por que Repository é um pattern de domínio (interface no Domain) e não de infraestrutura
