# 02 — Dapper (Micro-ORM)

## O que é Dapper e onde ele se posiciona

Dapper é um **micro-ORM** mantido pelo Stack Overflow que resolve um problema específico: mapear resultados de queries SQL para objetos C# com o mínimo de overhead possível. Diferente do EF Core, Dapper não tem Change Tracker, não gera SQL, não faz migrations, não traduz LINQ. Ele apenas executa o SQL que você escreve e mapeia as colunas do resultado para propriedades de objetos.

O espectro de abstração de acesso a dados no .NET:

```
ADO.NET puro  ←  Dapper  ←  EF Core
(mais rápido,     (SQL explícito,   (LINQ→SQL,
 mais código)     zero tracking)    Change Tracker)
```

Dapper opera uma camada acima do ADO.NET puro: você ainda escreve SQL manualmente, mas não precisa lidar com `DbDataReader`, `GetString()`, `GetInt32()`, e mapeamento manual de colunas. Dapper usa reflection para mapear colunas → propriedades automaticamente, e seu desempenho é próximo do ADO.NET puro porque faz caching agressivo dos mapeamentos.

---

## Uso básico

```bash
dotnet add package Dapper
```

```csharp
using Dapper;
using Npgsql;

// Configuração — Dapper usa IDbConnection (qualquer provider ADO.NET)
using var conn = new NpgsqlConnection(connectionString);

// Query — múltiplas linhas
var habits = await conn.QueryAsync<Habit>(
    "SELECT * FROM Habits WHERE UserId = @UserId",
    new { UserId = userId });

// QuerySingleOrDefault — 1 linha ou null
var habit = await conn.QuerySingleOrDefaultAsync<Habit>(
    "SELECT * FROM Habits WHERE Id = @Id",
    new { Id = id });

// Execute — INSERT/UPDATE/DELETE (retorna linhas afetadas)
int rows = await conn.ExecuteAsync(
    "INSERT INTO Habits (Id, UserId, Name, WeeklyFrequency) VALUES (@Id, @UserId, @Name, @WeeklyFrequency)",
    new { habit.Id, habit.UserId, habit.Name, habit.WeeklyFrequency });

// ExecuteScalar — valor único (COUNT, MAX, etc.)
int count = await conn.ExecuteScalarAsync<int>(
    "SELECT COUNT(*) FROM Habits WHERE UserId = @UserId",
    new { UserId = userId });
```

**Como Dapper mapeia colunas para propriedades**: por padrão, o nome da coluna no banco deve bater exatamente com o nome da propriedade C# (case-insensitive). Se os nomes diferem, você configura o mapeamento manualmente. Dapper não usa `[Column]` attributes por padrão — ele simplesmente tenta casar nomes.

### Mapeamento customizado

```csharp
// Múltiplos resultados em uma query (Multi Mapping)
var sql = @"
    SELECT * FROM Habits WHERE Id = @Id;
    SELECT * FROM HabitCompletions WHERE HabitId = @Id;";

using var multi = await conn.QueryMultipleAsync(sql, new { Id = habitId });
var habit = await multi.ReadSingleOrDefaultAsync<Habit>();
var completions = await multi.ReadAsync<HabitCompletion>();
```

### Stored Procedures

```csharp
var result = await conn.QueryAsync<Habit>(
    "get_active_habits",
    new { UserId = userId },
    commandType: CommandType.StoredProcedure);
```

### Transações com Dapper

Dapper não gerencia transações — você usa a transação do ADO.NET:

```csharp
using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync();
using var tx = await conn.BeginTransactionAsync();

try
{
    await conn.ExecuteAsync(
        "INSERT INTO Habits (...) VALUES (...)", new { ... }, tx);
    await conn.ExecuteAsync(
        "INSERT INTO HabitCompletions (...) VALUES (...)", new { ... }, tx);

    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```

Note que a transação é passada como parâmetro para cada chamada Dapper. Sem isso, cada `ExecuteAsync` rodaria em uma transão implícita separada.

---

## Dapper vs EF Core: quando usar cada um

| Critério | Dapper | EF Core |
|---|---|---|
| **Performance** | Muito próximo de ADO.NET puro | ~5-15% overhead sobre Dapper (Change Tracker, LINQ→SQL) |
| **Produtividade** | SQL manual — mais código | LINQ — menos código, type-safe |
| **Migrations** | Não tem — use ferramentas externas (DbUp, FluentMigrator) | Nativo via `dotnet ef migrations` |
| **Change Tracking** | Não tem — você controla tudo | Automático — detecta mudanças e gera UPDATEs parciais |
| **Query composition** | Limitada — SQL como string | Expression trees — totalmente composable |
| **Aprendizado** | Só precisa saber SQL e C# | Precisa aprender o EF Core (mapeamento, tracking, migrations) |

### Cenários onde Dapper brilha

1. **Queries complexas com performance crítica**: relatórios com múltiplos JOINs, CTEs, window functions que são difíceis ou impossíveis de expressar em LINQ
2. **Sistemas com SQL já otimizado**: você herda um banco com stored procedures, views e índices complexos — Dapper executa o SQL existente sem tentar "traduzir"
3. **Bulk operations**: inserir 100.000 registros com Dapper é significativamente mais rápido que com EF Core (sem Change Tracker, sem detectar mudanças)
4. **Microsserviços simples**: se o serviço tem 3 queries SQL, Dapper resolve com menos dependências que o EF Core

### Cenários onde EF Core brilha

1. **Aplicações com lógica de negócio complexa**: o Change Tracker detecta o que mudou e gera UPDATEs precisos
2. **Domínios que evoluem rápido**: migrations versionadas mantêm banco e código sincronizados
3. **Equipes que não têm DBA**: escrever LINQ é mais seguro que escrever SQL manual (sem injection, type-safe)
4. **CRUD com muitos relacionamentos**: `Include()` e `ThenInclude()` são mais produtivos que escrever JOINs manualmente

### A estratégia híbrida (usada em produção real)

Muitos projetos .NET usam EF Core para comandos (INSERT, UPDATE, DELETE) e Dapper para queries de leitura complexas. Isso combina o melhor dos dois mundos:

```csharp
// EF Core para comandos (Change Tracker + migrations)
public class EventRepository : IEventRepository
{
    private readonly AppDbContext _db;
    public void Add(Event @event) => _db.Events.Add(@event);
    public async Task SaveChangesAsync() => await _db.SaveChangesAsync();
}

// Dapper para queries complexas (performance)
public class EventReportService
{
    private readonly IDbConnection _conn;
    public async Task<SalesReport> GetSalesReport(DateTime from, DateTime to)
    {
        return await _conn.QuerySingleAsync<SalesReport>(@"
            SELECT
                COUNT(*) AS TotalSales,
                SUM(t.Price) AS TotalRevenue
            FROM Tickets t
            JOIN Sessions s ON t.SessionId = s.Id
            WHERE t.PurchasedAt BETWEEN @From AND @To",
            new { From = from, To = to });
    }
}
```

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar a diferença fundamental entre Dapper (executa seu SQL, mapeia resultados) e EF Core (traduz LINQ para SQL, rastreia mudanças)
- [ ] Usar `QueryAsync<T>`, `QuerySingleOrDefaultAsync<T>`, `ExecuteAsync` e `ExecuteScalarAsync<T>`
- [ ] Implementar uma transação manual com `IDbTransaction` passada para cada chamada Dapper
- [ ] Escolher Dapper para queries complexas/performance crítica e EF Core para comandos com Change Tracker
- [ ] Conhecer a estratégia híbrida (EF Core para escrita, Dapper para leitura)
