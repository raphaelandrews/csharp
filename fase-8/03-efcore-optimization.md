# 03 — Otimização de queries EF Core: o problema N+1 e outros clássicos

## O problema N+1

O bug de performance mais comum em projetos EF Core:

```csharp
// 1 query para buscar eventos
var events = await _db.Events.Where(e => e.Status == Published).ToListAsync();
// N queries — uma para cada evento!
foreach (var e in events)
{
    var sessions = await _db.Sessions.Where(s => s.EventId == e.Id).ToListAsync();
}
```

Resultado: 1 + N queries. Para 50 eventos, são 51 idas ao banco em vez de 1.

### Solução: Include (Eager Loading)

```csharp
var events = await _db.Events
    .Where(e => e.Status == Published)
    .Include(e => e.Sessions)
        .ThenInclude(s => s.Tickets)
    .ToListAsync(); // 1 query com JOINs
```

### Solução: Projection (Select)

```csharp
var events = await _db.Events
    .Where(e => e.Status == Published)
    .Select(e => new EventSummaryDto(
        e.Id, e.Title, e.StartDate,
        Sessions: e.Sessions.Select(s => new SessionDto(
            s.Id, s.Name, s.Capacity,
            Available: s.Capacity - s.Tickets.Count(t => t.Status != Available)
        ))
    ))
    .ToListAsync(); // 1 query, traz só o que precisa
```

`Select()` é mais eficiente que `Include()` porque:
1. Traz apenas as colunas necessárias (não `SELECT *`)
2. Não carrega o Change Tracker (objetos anônimos/DTOs não são rastreados)
3. Pode usar funções SQL diretamente na projeção (`s.Tickets.Count()` vira subquery)

---

## AsSplitQuery: quando Include gera explosão cartesiana

```csharp
var events = await _db.Events
    .Include(e => e.Sessions)
    .Include(e => e.Organizer)
    .ToListAsync();
```

Se um evento tem 5 sessions, o JOIN gera 5 linhas por evento. Cada linha duplica os dados do evento e do organizador. Com múltiplos Includes, isso vira um produto cartesiano que envia dados redundantes pela rede.

```csharp
var events = await _db.Events
    .Include(e => e.Sessions)
    .Include(e => e.Organizer)
    .AsSplitQuery()            // queries separadas, sem produto cartesiano
    .ToListAsync();
```

`AsSplitQuery()` executa uma query para cada `Include` em vez de um único JOIN. Trade-off: mais round-trips ao banco, mas menos dados transferidos e sem explosão cartesiana.

---

## Paginação: nunca retorne tudo

```csharp
// RUIM: traz todos os registros do banco
var events = await _db.Events.ToListAsync(); // 100.000 registros!

// BOM: paginação no banco (Skip/Take)
var events = await _db.Events
    .OrderByDescending(e => e.CreatedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync(); // 20 registros
```

### Paginação por cursor (mais eficiente que offset)

```csharp
// Offset: Skip(10000).Take(20) — o banco precisa pular 10k linhas
// Cursor: WHERE Id > @lastId — usa índice, O(1)
var events = await _db.Events
    .Where(e => e.Id > lastId)
    .OrderBy(e => e.Id)
    .Take(20)
    .ToListAsync();
```

Cursor é mais eficiente que offset em tabelas grandes porque usa o índice clusterizado em vez de scan + skip.

---

## Compiled Queries (casos extremos)

```csharp
// Query compilada — o EF Core compila a expression tree uma vez e reusa
private static readonly Func<AppDbContext, Guid, Task<Event?>> GetEventById =
    EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
        db.Events
            .Include(e => e.Sessions)
            .FirstOrDefault(e => e.Id == id));

// Uso
var @event = await GetEventById(_db, eventId);
```

Para queries executadas milhares de vezes com parâmetros diferentes, compiled queries reduzem o overhead de compilação da expression tree em ~10-15%. Mas só use se você mediu que isso é o gargalo.

---

## Índices: o básico que resolve 80% dos problemas

```csharp
// Índice para busca por OrganizerId (Filtro WHERE)
modelBuilder.Entity<Event>()
    .HasIndex(e => e.OrganizerId);

// Índice composto para busca combinada
modelBuilder.Entity<Event>()
    .HasIndex(e => new { e.Status, e.StartDate });

// Índice unique
modelBuilder.Entity<Event>()
    .HasIndex(e => e.Slug).IsUnique();

// Índice com filtro (PostgreSQL partial index)
modelBuilder.Entity<Event>()
    .HasIndex(e => e.StartDate)
    .HasFilter("\"Status\" = 'Published'");
```

Lembre-se: índice acelera leitura e penaliza escrita. Cada INSERT/UPDATE/DELETE precisa atualizar os índices. Não indexe tudo.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Identificar N+1 no código e corrigir com `Include` ou `Select`
- [ ] Explicar a diferença entre `Include` (eager, traz tudo) e `Select` (projeção, traz só o necessário)
- [ ] Saber quando usar `AsSplitQuery()` (múltiplos Includes, explosão cartesiana)
- [ ] Implementar paginação com Skip/Take e cursor (WHERE Id > @lastId)
- [ ] Explicar por que `AsNoTracking()` é importante para queries somente-leitura
- [ ] Criar índices compostos e parciais via Fluent API
