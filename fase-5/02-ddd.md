# 02 — Domain-Driven Design: conceitos táticos

## Por que DDD aparece em vagas .NET

Domain-Driven Design (Eric Evans, 2003) é um livro de 560 páginas sobre como modelar software complexo alinhado com o domínio do negócio. Mas nas vagas .NET pleno/sênior, "DDD" geralmente significa **dominar os padrões táticos**: Entities, Value Objects, Aggregates, Domain Events e Repositories. Ninguém espera que você tenha lido o livro inteiro — mas esperam que você saiba aplicar esses conceitos em código C#.

A razão pela qual DDD é tão associado a .NET é o ecossistema: Clean Architecture + EF Core + MediatR formam uma combinação que implementa DDD tático de forma natural. O .NET tem idioms que encaixam bem com os conceitos do Evans — construtores privados, `IReadOnlyCollection<T>`, `record` para value objects, e DI para serviços de domínio.

---

## Entities: objetos com identidade contínua

Uma entidade é um objeto definido por sua **identidade**, não por seus atributos. Dois objetos `Event` com o mesmo título, mesma data e mesmo local são entidades diferentes se tiverem `Id` diferentes.

### Características de uma entidade bem modelada em DDD

1. **Identidade explícita**: `Id` como propriedade fundamental
2. **Construtor privado**: para forçar criação via factory method
3. **Métodos de negócio, não setters**: `Publish()` em vez de `Status = Published`
4. **Encapsulamento de coleções**: `IReadOnlyCollection<T>` com backing field privado
5. **Invariantes protegidas**: validações no método, não em atributos externos

```csharp
// EventHub.Domain/Entities/Event.cs
public class Event
{
    public Guid Id { get; private set; }
    public string Title { get; private set; }
    public string Description { get; private set; }
    public EventStatus Status { get; private set; }
    public Guid OrganizerId { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private readonly List<Session> _sessions = new();
    public IReadOnlyCollection<Session> Sessions => _sessions.AsReadOnly();

    private Event() { } // EF Core

    public static Event Create(string title, string description, Guid organizerId)
    {
        if (string.IsNullOrWhiteSpace(title))
            throw new DomainException("Title is required");

        return new Event
        {
            Id = Guid.NewGuid(),
            Title = title,
            Description = description,
            OrganizerId = organizerId,
            Status = EventStatus.Draft,
            CreatedAt = DateTime.UtcNow
        };
    }

    public Session AddSession(string name, DateTime start, DateTime end, int capacity)
    {
        if (Status == EventStatus.Cancelled)
            throw new DomainException("Cannot add sessions to a cancelled event");

        var session = Session.Create(Id, name, start, end, capacity);
        _sessions.Add(session);
        return session;
    }

    public void Publish()
    {
        if (Status != EventStatus.Draft)
            throw new DomainException("Only draft events can be published");

        if (_sessions.Count == 0)
            throw new DomainException("Event must have at least one session to be published");

        Status = EventStatus.Published;
    }

    public void Cancel(string reason)
    {
        if (Status == EventStatus.Cancelled)
            throw new DomainException("Event is already cancelled");

        Status = EventStatus.Cancelled;
        // Domain event poderia ser disparado aqui: AddDomainEvent(new EventCancelled(Id, reason));
    }
}
```

### O que diferencia isso de um "model anêmico"

Um modelo anêmico (anti-padrão comum em projetos .NET) seria:

```csharp
// Modelo ANÊMICO — NÃO é DDD
public class Event
{
    public Guid Id { get; set; }
    public string Title { get; set; }
    public EventStatus Status { get; set; }
    public List<Session> Sessions { get; set; } = new();
}

// Toda a lógica fica em um "service":
public class EventService
{
    public void PublishEvent(Event @event)
    {
        if (@event.Status != EventStatus.Draft)
            throw new Exception("...");
        @event.Status = EventStatus.Published;
    }
}
```

O modelo anêmico é um saco de dados. A lógica de negócio está espalhada em services. Em DDD, a lógica vive na entidade. O service orquestra, não decide.

---

## Value Objects: objetos definidos por seus atributos

Um Value Object é imutável, não tem identidade, e dois VOs com os mesmos valores são considerados iguais.

### Exemplos clássicos no domínio de eventos

```csharp
// EventHub.Domain/ValueObjects/Money.cs
public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new DomainException("Amount cannot be negative");

        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new DomainException("Currency must be a 3-letter ISO code");

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} to {other.Currency}");

        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Currency} {Amount:N2}";
}
```

```csharp
// EventHub.Domain/ValueObjects/Email.cs
public record Email
{
    public string Value { get; }

    public Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Email is required");

        if (!value.Contains('@') || !value.Contains('.'))
            throw new DomainException("Invalid email format");

        Value = value.ToLowerInvariant().Trim();
    }

    public static implicit operator string(Email email) => email.Value;
    public static explicit operator Email(string value) => new(value);
    public override string ToString() => Value;
}
```

```csharp
// EventHub.Domain/ValueObjects/Address.cs
public record Address
{
    public string Street { get; }
    public string City { get; }
    public string State { get; }
    public string ZipCode { get; }
    public string Country { get; }

    public Address(string street, string city, string state, string zipCode, string country)
    {
        Street = street;
        City = city;
        State = state;
        ZipCode = zipCode;
        Country = country;
    }
}
```

### Por que `record` é perfeito para Value Objects

C# 9+ introduziu `record` com value-based equality — dois records com as mesmas propriedades são iguais. Isso é exatamente o que um Value Object precisa. Além disso, `init`-only properties garantem imutabilidade.

```csharp
var price1 = new Money(100, "BRL");
var price2 = new Money(100, "BRL");
Assert.Equal(price1, price2); // value equality — TRUE

var entity1 = Event.Create("Show", "desc", orgId);
var entity2 = Event.Create("Show", "desc", orgId);
Assert.NotEqual(entity1, entity2); // reference equality — FALSE (Ids diferentes)
```

### Quando usar Value Object vs primitivo

Se você tem uma propriedade `Price` como `decimal`, você perde contexto: é BRL? USD? Pode ser negativo? Precisa formatar com 2 casas decimais?

Transformar `decimal` em `Money` resolve todos esses problemas de uma vez:
- Validação no construtor (não pode ser negativo)
- Imutabilidade (não pode ser alterado acidentalmente)
- Comportamento (`Add()`, `ToString()` com formatação)
- Igualdade por valor (dois `Money(100, "BRL")` são iguais)

O custo: mais código, mais classes, mapeamento adicional para o banco (EF Core com owned types ou conversion).

---

## Aggregates: o conceito mais importante (e mais mal compreendido)

Um Aggregate é um cluster de entidades e value objects tratados como uma **unidade coesa**. Você acessa o aggregate pela raiz (Aggregate Root) e nunca diretamente pelos objetos internos.

### O aggregate Event no eventhub

```
Event (Aggregate Root)
├── Id, Title, Description, Status
├── Venue (Value Object)
├── Sessions (Entity Collection)
│   ├── Session 1
│   │   ├── Tickets (Entity Collection)
│   │   │   ├── Ticket A1
│   │   │   └── Ticket A2
│   │   └── Capacity, StartTime, EndTime
│   └── Session 2
│       └── Tickets...
└── OrganizerId
```

**Regras do aggregate**:
1. Referências externas só apontam para o **Aggregate Root** (`Event`), nunca para `Session` ou `Ticket` diretamente
2. Objetos internos (`Session`, `Ticket`) só podem ser acessados via o root: `@event.Sessions.First().Tickets`
3. Invariantes que cruzam múltiplos objetos do aggregate são garantidas pelo root
4. Cada aggregate tem seu próprio repositório (`IEventRepository`), não existe `ISessionRepository`

### Por que aggregates existem

Sem aggregates, você poderia ter:
```csharp
// CÓDIGO PERIGOSO — viola invariantes
var session = _sessionRepository.GetById(sessionId);
session.ReserveTicket(userId); // reserva o último ingresso
// ... mas e se outro usuário reservou ao mesmo tempo?
// O session não sabe quantos tickets sobraram porque os tickets são carregados separadamente
```

Com aggregates, a operação é atômica dentro do aggregate:
```csharp
// CÓDIGO SEGURO — invariantes garantidas pelo aggregate
var @event = await _eventRepository.GetByIdAsync(eventId);
@event.ReserveTicket(sessionId, userId); // O EVENT verifica disponibilidade, não o session
// Se não houver vagas, o próprio Event lança DomainException
await _eventRepository.Update(@event);
```

### Tamanho do aggregate: o trade-off

O aggregate `Event` no eventhub tem um tamanho razoável: um evento tem poucas sessions (1-10), cada session tem tickets (50-5000). Carregar tudo junto via `Include()` é viável.

Mas se um evento tivesse 100 sessions com 50.000 tickets cada, carregar o aggregate inteiro seria inviável. Nesse caso, você **quebraria o aggregate**: `Ticket` viraria um aggregate separado, e a verificação de disponibilidade usaria um serviço de domínio ou uma query otimizada.

**Regra de ouro**: um aggregate deve ser carregado inteiro em memória para garantir invariantes. Se não cabe em memória, está grande demais.

---

## Domain Events: comunicação entre aggregates

Quando algo significativo acontece no domínio, um Domain Event notifica outros componentes. Em .NET, o padrão mais comum é usar `MediatR.INotification`:

```csharp
// EventHub.Domain/Events/EventPublished.cs
public record EventPublished(Guid EventId, DateTime PublishedAt) : INotification;

// EventHub.Domain/Events/TicketReserved.cs
public record TicketReserved(Guid TicketId, Guid SessionId, Guid UserId) : INotification;

// Disparado pela entidade:
public void Publish()
{
    Status = EventStatus.Published;
    _domainEvents.Add(new EventPublished(Id, DateTime.UtcNow));
}

// O handler na Application:
public class EventPublishedHandler : INotificationHandler<EventPublished>
{
    public async Task Handle(EventPublished notification, CancellationToken ct)
    {
        // Enviar email para organizador, notificar seguidores, etc.
    }
}
```

Isso desacopla efeitos colaterais da lógica principal. A entidade `Publish()` não sabe que existe um sistema de notificação — ela só registra o evento. O handler decide o que fazer.

---

## Domain Services: quando a lógica não pertence a uma entidade

Se uma operação envolve múltiplos aggregates e não pertence naturalmente a nenhum deles, você usa um Domain Service. Diferente de um Application Service (que orquestra), um Domain Service contém **lógica de negócio pura**:

```csharp
// EventHub.Domain/Services/SeatPricingService.cs
public class SeatPricingService
{
    public Money CalculatePrice(Session session, SeatCategory category, DateTime purchaseDate)
    {
        var basePrice = session.BasePrice;

        // Early bird: 20% de desconto se comprar com 30+ dias de antecedência
        if ((session.StartTime - purchaseDate).Days >= 30)
            basePrice = basePrice.Multiply(0.8m);

        // Categoria VIP: 50% de acréscimo
        if (category == SeatCategory.VIP)
            basePrice = basePrice.Multiply(1.5m);

        // Última hora: se restam menos de 10% dos ingressos, +30%
        var occupancyRate = (decimal)session.ReservedSeats / session.Capacity;
        if (occupancyRate > 0.9m)
            basePrice = basePrice.Multiply(1.3m);

        return basePrice;
    }
}
```

Quando usar Domain Service vs método na entidade:
- **Método na entidade**: a lógica usa apenas dados da própria entidade (`Event.Publish()`, `Session.ReserveSeat()`)
- **Domain Service**: a lógica precisa de dados de múltiplos sources ou de cálculos complexos (`SeatPricingService`, `AvailabilityService`)

---

## Estratégia de persistência com EF Core e DDD

### Owned Types para Value Objects

```csharp
// Infrastructure/Persistence/Configurations/EventConfiguration.cs
public class EventConfiguration : IEntityTypeConfiguration<Event>
{
    public void Configure(EntityTypeBuilder<Event> builder)
    {
        builder.HasKey(e => e.Id);

        builder.OwnsOne(e => e.Venue, venue =>
        {
            venue.Property(v => v.Name).HasColumnName("VenueName").HasMaxLength(200);
            venue.Property(v => v.Capacity).HasColumnName("VenueCapacity");
            venue.OwnsOne(v => v.Address, address =>
            {
                address.Property(a => a.Street).HasColumnName("VenueStreet");
                address.Property(a => a.City).HasColumnName("VenueCity");
                address.Property(a => a.State).HasColumnName("VenueState");
            });
        });

        builder.OwnsMany(e => e.Sessions, session =>
        {
            session.HasKey(s => s.Id);
            session.WithOwner().HasForeignKey(s => s.EventId);
            session.Property(s => s.Name).HasMaxLength(200);
        });

        builder.Navigation(e => e.Sessions)
               .UsePropertyAccessMode(PropertyAccessMode.Field);
    }
}
```

### Backing fields para coleções encapsuladas

```csharp
// A entidade expõe IReadOnlyCollection, mas o EF Core usa o campo privado
private readonly List<Session> _sessions = new();
public IReadOnlyCollection<Session> Sessions => _sessions.AsReadOnly();

// EF Core config:
builder.Navigation(e => e.Sessions)
       .UsePropertyAccessMode(PropertyAccessMode.Field);
```

Isso garante que ninguém faça `@event.Sessions.Add(session)` — isso não compila, porque `IReadOnlyCollection` não tem `Add()`. O único jeito de adicionar uma session é via `@event.AddSession(session)`, que contém as validações de negócio.

### Construtor privado e EF Core

O EF Core consegue preencher entidades com construtores privados via reflection. O parâmetro padrão é um construtor sem parâmetros (que o EF Core usa para materialização) + um construtor com parâmetros (que você usa nos factory methods).

```csharp
private Event() { } // EF Core

// Seus factory methods usam construtores parametrizados:
private Event(Guid id, string title, Guid organizerId)
{
    Id = id;
    Title = title;
    OrganizerId = organizerId;
}
```

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Diferenciar Entity (identidade) de Value Object (valor) em código C#
- [ ] Modelar uma entidade com construtor privado, factory method e comportamento encapsulado
- [ ] Criar um Value Object como `record` com validação no construtor
- [ ] Desenhar um aggregate e explicar por que objetos internos não têm repositório próprio
- [ ] Explicar quando usar Domain Service vs método na entidade
- [ ] Configurar EF Core com backing fields e owned types para preservar o encapsulamento do Domain
- [ ] Disparar um Domain Event da entidade e tratá-lo com `INotificationHandler<T>`
