# 03 — CQRS: Command Query Responsibility Segregation

## O problema que CQRS resolve

Toda aplicação CRUD trata leitura e escrita da mesma forma: mesmo modelo, mesmo repositório, mesmo banco. Isso funciona para sistemas simples, mas cria tensão quando o sistema cresce:

1. **Modelos de leitura são diferentes de modelos de escrita**: para listar eventos, você quer título + data + status + vagas restantes. Para criar um evento, você quer título + descrição + local + capacity + preço. Usar o mesmo modelo para ambos significa que a listagem carrega propriedades que nunca usa (descrição longa, preços), e a criação expõe propriedades que deveriam ser calculadas (vagas restantes).

2. **Otimizações conflitantes**: você quer índice para busca textual no título (leitura), mas quer integridade referencial com constraints (escrita). Você quer cache para listagens (leitura), mas consistência forte para reservas (escrita). O mesmo modelo não atende bem aos dois.

3. **Complexidade de queries**: queries complexas com múltiplos joins, agregações e projeções não pertencem ao mesmo lugar que comandos com regras de negócio e validações. Misturar os dois no mesmo repositório/entidade gera código difícil de manter.

CQRS resolve isso com uma separação radical: **commands mudam estado, queries retornam dados, e eles usam modelos diferentes**.

---

## O modelo mental do CQRS

```
┌──────────────────────────────────────────────┐
│                   HTTP Request                 │
│                                                  │
│  POST /api/events     GET /api/events?page=1   │
│       │                      │                   │
│       ▼                      ▼                   │
│  ┌─────────┐          ┌─────────┐              │
│  │ Command  │          │ Query   │              │
│  │ (Write) │          │ (Read)  │              │
│  └────┬────┘          └────┬────┘              │
│       │                      │                   │
│       ▼                      ▼                   │
│  ┌──────────────┐    ┌──────────────┐          │
│  │ Domain Model  │    │ Read Model    │          │
│  │ (Entity rica) │    │ (DTO achatado)│          │
│  │ - invariantes │    │ - projeção    │          │
│  │ - métodos     │    │ - otimizado   │          │
│  │ - validações  │    │ - cacheável   │          │
│  └──────┬───────┘    └──────┬───────┘          │
│         │                    │                   │
│         ▼                    ▼                   │
│  ┌──────────────┐    ┌──────────────┐          │
│  │  Write DB     │    │  Read DB      │          │
│  │  (normalized) │    │  (projection) │          │
│  └──────────────┘    └──────────────┘          │
└──────────────────────────────────────────────┘
```

### CQRS simples (mesmo banco, modelos diferentes)

A versão mais comum em projetos .NET: mesmo banco PostgreSQL, mas commands usam as entidades ricas do Domain e queries usam Dapper ou projeções EF Core direto em DTOs:

```csharp
// Command — usa a entidade rica (Domain)
public record CreateEventCommand(string Title, string Description, Guid OrganizerId) : IRequest<EventDto>;

public class CreateEventCommandHandler : IRequestHandler<CreateEventCommand, EventDto>
{
    private readonly IEventRepository _repo;

    public async Task<EventDto> Handle(CreateEventCommand cmd, CancellationToken ct)
    {
        var @event = Event.Create(cmd.Title, cmd.Description, cmd.OrganizerId);
        _repo.Add(@event);
        await _repo.SaveChangesAsync(ct);
        return EventDto.FromEntity(@event);
    }
}

// Query — usa projeção direta (sem carregar entidade)
public record GetEventsQuery(int Page, int PageSize) : IRequest<PagedList<EventSummaryDto>>;

public class GetEventsQueryHandler : IRequestHandler<GetEventsQuery, PagedList<EventSummaryDto>>
{
    private readonly AppDbContext _db; // DbContext direto, sem repositório

    public async Task<PagedList<EventSummaryDto>> Handle(GetEventsQuery query, CancellationToken ct)
    {
        var events = await _db.Events
            .Where(e => e.Status == EventStatus.Published)
            .OrderByDescending(e => e.StartDate)
            .Select(e => new EventSummaryDto(
                e.Id,
                e.Title,
                e.StartDate,
                e.Venue.Name,
                e.Sessions.Sum(s => s.Capacity),
                e.Sessions.Sum(s => s.ReservedSeats)
            ))
            .ToPagedListAsync(query.Page, query.PageSize, ct);

        return events;
    }
}
```

Note: o handler de query usa `AppDbContext` diretamente (sem `IEventRepository`), fazendo projeção via `Select()`. Isso é intencional — queries não precisam da entidade rica, elas precisam de DTOs otimizados.

### CQRS avançado (bancos separados)

Em sistemas de alta escala, os bancos de leitura e escrita são fisicamente separados:

```
Command → PostgreSQL (escrita, normalizado)
                        │
                        ▼ (evento: "EventoCriado")
                  ┌─────────────┐
                  │  Projeção    │ → MongoDB ou Elasticsearch (leitura, desnormalizado)
                  └─────────────┘
                                              │
                                              ▼
Query ← Materialized View (dados já no formato que o frontend precisa)
```

Isso permite que a listagem de eventos seja uma query simples em MongoDB sem joins, enquanto a reserva de ingressos mantém a integridade transacional do PostgreSQL.

Para o eventhub, usaremos **CQRS simples** (mesmo banco), que é o que 95% dos projetos .NET usam.

---

## Commands: intenção de mudança

### Todo comando é um objeto imutável

```csharp
// EventHub.Application/Events/Commands/CreateEvent/CreateEventCommand.cs
public record CreateEventCommand(
    string Title,
    string Description,
    DateTime StartDate,
    DateTime EndDate,
    string VenueName,
    string VenueStreet,
    string VenueCity,
    string VenueState,
    List<CreateSessionDto> Sessions
) : IRequest<EventDto>;
```

Commands são `record` (imutáveis), implementam `IRequest<TResponse>`, e são nomeados como verbo no imperativo + substantivo.

### Todo comando tem exatamente um handler

```csharp
// EventHub.Application/Events/Commands/CreateEvent/CreateEventCommandHandler.cs
public class CreateEventCommandHandler : IRequestHandler<CreateEventCommand, EventDto>
{
    private readonly IEventRepository _eventRepository;

    public CreateEventCommandHandler(IEventRepository eventRepository)
    {
        _eventRepository = eventRepository;
    }

    public async Task<EventDto> Handle(CreateEventCommand cmd, CancellationToken ct)
    {
        var venue = new Venue(cmd.VenueName, new Address(
            cmd.VenueStreet, cmd.VenueCity, cmd.VenueState, "Brasil"));

        var @event = Event.Create(cmd.Title, cmd.Description, cmd.OrganizerId);

        foreach (var sessionDto in cmd.Sessions)
        {
            @event.AddSession(sessionDto.Name, sessionDto.StartTime,
                sessionDto.EndTime, sessionDto.Capacity, sessionDto.BasePrice);
        }

        @event.Publish();

        _eventRepository.Add(@event);
        await _eventRepository.SaveChangesAsync(ct);

        return EventDto.FromEntity(@event);
    }
}
```

O handler:
1. Cria Value Objects a partir do comando
2. Usa factory methods da entidade para criar o aggregate
3. Chama métodos de negócio (`Publish()`)
4. Persiste via repositório
5. Retorna um DTO

Nenhuma lógica de negócio no handler — só orquestração.

---

## Queries: leitura otimizada

Queries usam `IRequest<TResponse>` também, mas seus handlers leem direto do banco sem passar pelo Domain:

```csharp
// EventHub.Application/Events/Queries/GetEvents/GetEventsQuery.cs
public record GetEventsQuery(int Page = 1, int PageSize = 20, string? Search = null)
    : IRequest<PagedList<EventSummaryDto>>;

// EventHub.Application/Events/Queries/GetEvents/GetEventsQueryHandler.cs
public class GetEventsQueryHandler : IRequestHandler<GetEventsQuery, PagedList<EventSummaryDto>>
{
    private readonly AppDbContext _db;

    public GetEventsQueryHandler(AppDbContext db)
    {
        _db = db;
    }

    public async Task<PagedList<EventSummaryDto>> Handle(
        GetEventsQuery query, CancellationToken ct)
    {
        var eventsQuery = _db.Events
            .Where(e => e.Status == EventStatus.Published);

        if (!string.IsNullOrWhiteSpace(query.Search))
        {
            eventsQuery = eventsQuery
                .Where(e => EF.Functions.ILike(e.Title, $"%{query.Search}%"));
        }

        var total = await eventsQuery.CountAsync(ct);

        var items = await eventsQuery
            .OrderByDescending(e => e.StartDate)
            .Skip((query.Page - 1) * query.PageSize)
            .Take(query.PageSize)
            .Select(e => new EventSummaryDto(
                e.Id,
                e.Title,
                e.StartDate,
                e.Venue.Name,
                e.Sessions.Sum(s => s.Capacity) - e.Sessions.Sum(s => s.SoldTickets.Count),
                e.Sessions.Min(s => s.BasePrice.Amount)
            ))
            .ToListAsync(ct);

        return new PagedList<EventSummaryDto>(items, total, query.Page, query.PageSize);
    }
}
```

**Padrão importante**: query handlers injetam `AppDbContext` diretamente, não `IEventRepository`. O repositório é para comandos (escrita), o DbContext direto é para queries (leitura). Isso é intencional — queries precisam de projeções otimizadas (`Select()`), e o repositório não deveria expor `IQueryable`.

---

## O que NÃO é CQRS (erros comuns)

### Erro: usar o mesmo handler para command e query

```csharp
// ERRADO: GetOrCreate não é CQRS
public record GetOrCreateEventCommand(string Title) : IRequest<EventDto>;
```

CQRS é sobre separação. Se você tem `GetOrCreate`, você não está segregando.

### Erro: comando que retorna entidade do domínio

```csharp
// ERRADO: vaza o Domain para fora da Application
public class CreateEventCommandHandler : IRequestHandler<CreateEventCommand, Event>
{
    public async Task<Event> Handle(...) { ... return @event; }
}
```

Sempre retorne DTOs. A entidade do domínio não deve sair da camada Application.

### Erro: query que modifica estado

```csharp
// ERRADO: query com efeito colateral
public record GetEventsQuery : IRequest<List<EventDto>>
{
    public bool IncrementViewCount { get; set; } // NUNCA faça isso
}
```

Se modifica estado, é comando. Se retorna dados, é query. Nunca os dois.

### Erro: repositório único para leitura e escrita

```csharp
// ERRADO: mesmo repositório para command e query
public class GetEventsQueryHandler : IRequestHandler<...>
{
    private readonly IEventRepository _repo; // ← repositório é para commands
}
```

Em CQRS, queries podem (e devem) usar DbContext direto ou Dapper. O repositório é uma abstração de escrita, não de leitura.

---

## CQRS com MediatR: como tudo se conecta

O MediatR é o mediador que roteia commands e queries para seus handlers. A API não sabe qual handler trata qual comando:

```csharp
// Endpoint da API
app.MapPost("/api/events", async (CreateEventCommand cmd, IMediator mediator, CancellationToken ct) =>
{
    var result = await mediator.Send(cmd, ct);
    return Results.Created($"/api/events/{result.Id}", result);
});

// O MediatR encontra o handler automaticamente:
// CreateEventCommand → CreateEventCommandHandler
```

Isso será aprofundado no próximo tópico (04-mediatr.md). Mas a conexão CQRS ↔ MediatR é tão forte no ecossistema .NET que é difícil falar de um sem o outro.

---

## Trade-offs do CQRS

| Vantagem | Custo |
|---|---|
| Modelos de leitura otimizados (projeções, sem joins desnecessários) | Duas vezes mais código (um handler de command + um de query por feature) |
| Commands e queries podem escalar independentemente | Maior complexidade para devs novos entenderem o fluxo |
| Segurança: queries nunca modificam estado por design | Overhead para CRUD simples (um Create + Read simples não justifica CQRS) |
| Testabilidade: testar query handlers sem mockar entidades | Duplicação de modelos (entidade rica + DTO de leitura) |

### Quando NÃO usar CQRS

- API com menos de 10 endpoints
- Domínio onde leitura e escrita são simétricos (CRUD de configurações, por exemplo)
- Equipe pequena que não justifica a complexidade adicional
- MVP que será reescrito

### Quando usar CQRS

- Sistema com leituras complexas que não batem com o modelo de escrita (ex: dashboard de vendas vs formulário de criação de evento)
- Necessidade de escalar leitura e escrita independentemente
- Múltiplos consumidores de leitura (web, mobile, relatórios) com formatos diferentes
- Domínio com regras de negócio complexas nos comandos (ex: reserva de ingressos)

O eventhub se encaixa: comandos têm regras de concorrência e validações de capacidade, queries precisam de projeções otimizadas (vagas restantes, preço mínimo).

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar a diferença fundamental: command muda estado, query retorna dados
- [ ] Criar um command como `record : IRequest<TResponse>` com handler separado
- [ ] Criar uma query com projeção `Select()` direto no DbContext (sem repositório)
- [ ] Explicar por que query handlers não devem usar `IEventRepository`
- [ ] Identificar anti-padrões: comando retornando entidade, query com efeito colateral, GetOrCreate
- [ ] Argumentar quando CQRS simples (mesmo banco) é suficiente vs quando bancos separados se justificam
