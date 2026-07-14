# 02 — Dapper (Micro-ORM)

Dapper é mantido pelo Stack Overflow. Não tem Change Tracker, migrations nem LINQ→SQL. O que faz: mapeia resultados de SQL pra objetos C# com performance próxima de ADO.NET puro.

```
ADO.NET puro  ←  Dapper  ←  EF Core
(mais rápido,    (SQL explícito,  (LINQ→SQL,
 mais código)    zero tracking)   Change Tracker)
```

```bash
dotnet add package Dapper
```

---

## Uso básico

```csharp
using Dapper;
using Npgsql;

// Query — múltiplas linhas
var habits = await conn.QueryAsync<Habit>(
    "SELECT * FROM Habits WHERE UserId = @UserId",
    new { UserId = userId });

// QuerySingleOrDefault — 1 ou null
var habit = await conn.QuerySingleOrDefaultAsync<Habit>(
    "SELECT * FROM Habits WHERE Id = @Id", new { Id = id });

// Execute — INSERT/UPDATE/DELETE sem retorno
var rows = await conn.ExecuteAsync(
    "DELETE FROM Habits WHERE Id = @Id", new { Id = id });

// INSERT com RETURNING
var id = await conn.QuerySingleAsync<Guid>(
    @"INSERT INTO Habits (Name, UserId) VALUES (@Name, @UserId)
      RETURNING Id", habit);
```

---

## Multi-mapping — JOIN manual

```csharp
var habitDict = new Dictionary<Guid, Habit>();

await conn.QueryAsync<Habit, HabitCompletion, Habit>(
    @"SELECT h.*, c.* FROM Habits h
      LEFT JOIN HabitCompletions c ON c.HabitId = h.Id
      WHERE h.UserId = @UserId",
    (habit, completion) =>
    {
        if (!habitDict.TryGetValue(habit.Id, out var existing))
        {
            existing = habit;
            existing.Completions = new List<HabitCompletion>();
            habitDict.Add(habit.Id, existing);
        }
        if (completion is not null)
            existing.Completions.Add(completion);
        return existing;
    },
    new { UserId = userId },
    splitOn: "Id");
```

## QueryMultiple — múltiplos result sets

```csharp
using var multi = await conn.QueryMultipleAsync(
    @"SELECT * FROM Habits WHERE UserId = @Id;
      SELECT * FROM Completions WHERE HabitId = ANY(
        SELECT Id FROM Habits WHERE UserId = @Id)",
    new { Id = userId });

var habits = await multi.ReadAsync<Habit>();
var completions = await multi.ReadAsync<HabitCompletion>();
```

---

## Stored Procedures

```csharp
var result = await conn.QueryAsync<Habit>(
    "sp_get_habits",
    new { UserId = userId },
    commandType: CommandType.StoredProcedure);
```

---

## Dapper.Contrib — atalhos CRUD

```bash
dotnet add package Dapper.Contrib
```

```csharp
using Dapper.Contrib.Extensions;

[Table("Habits")]
public class Habit
{
    [ExplicitKey] public Guid Id { get; set; }
    public string Name { get; set; }
    [Write(false)] public DateTime CreatedAt { get; set; }
    [Computed] public int CompletionCount { get; set; }
}

await conn.InsertAsync(habit);
await conn.UpdateAsync(habit);
await conn.DeleteAsync(habit);
var habit = await conn.GetAsync<Habit>(id);
var all = await conn.GetAllAsync<Habit>();
```

Só serve pra CRUD simples. Queries complexas = Dapper puro.

---

## Performance comparativa

| Operação | EF Core | Dapper | ADO.NET puro |
|---|---|---|---|
| SELECT 1 row | 0.3ms | 0.15ms | 0.1ms |
| SELECT 1000 rows | 8ms | 3ms | 2ms |
| INSERT 1000 rows | 50ms | 20ms | 15ms |

**Na prática**: o gargalo é a query/rede, não o ORM. A diferença importa em hot paths e relatórios pesados.

---

## Quando usar cada um?

| EF Core | Dapper |
|---|---|
| Projeto novo, CRUD padrão | SQL complexo que EF não traduz bem |
| Equipe grande, padronização | Controle total sobre a query |
| Migrations automáticas | Performance crítica (hot path) |
| Relacionamentos complexos | Relatórios, queries analíticas |
| Prototipagem rápida | Stored procedures, banco legado |

**Abordagem híbrida** (comum em projetos reais): EF Core pra transações/CRUD, Dapper pra queries de leitura (CQRS leve). Na mesma transação:

```csharp
using var transaction = await _db.Database.BeginTransactionAsync();
var conn = _db.Database.GetDbConnection();

_db.Habits.Add(habit);
await _db.SaveChangesAsync(); // EF Core

await conn.ExecuteAsync(       // Dapper — mesma transação
    "INSERT INTO AuditLog (...) VALUES (...)",
    new { habit.Id, Action = "CREATE" },
    transaction: transaction.GetDbTransaction());

await transaction.CommitAsync();
```
