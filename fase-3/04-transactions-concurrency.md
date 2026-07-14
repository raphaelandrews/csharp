# 04 — Transações e Concorrência

Transações garantem atomicidade (tudo ou nada). Concorrência controla acesso simultâneo ao mesmo dado.

---

## Transações com EF Core

`SaveChangesAsync()` já é transacional por padrão. Tudo rastreado pelo Change Tracker é enviado em uma transação implícita:

```csharp
_db.Habits.Add(habit);
_db.HabitCompletions.Add(completion);
await _db.SaveChangesAsync(); // INSERT de ambos com COMMIT automático
```

### Transação explícita

```csharp
using var transaction = await _db.Database.BeginTransactionAsync();

try
{
    _db.Habits.Add(habit);
    await _db.SaveChangesAsync();

    _db.HabitCompletions.Add(completion);
    await _db.SaveChangesAsync();

    await transaction.CommitAsync(); // AMBOS atomicamente
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### Múltiplos DbContexts na mesma transação

```csharp
using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync();
using var transaction = await conn.BeginTransactionAsync();

await using var db1 = new AppDbContext(
    new DbContextOptionsBuilder<AppDbContext>().UseNpgsql(conn).Options);
db1.Database.UseTransaction(transaction);

await using var db2 = new AuditDbContext(
    new DbContextOptionsBuilder<AuditDbContext>().UseNpgsql(conn).Options);
db2.Database.UseTransaction(transaction);

db1.Habits.Add(habit); await db1.SaveChangesAsync();
db2.AuditLogs.Add(log); await db2.SaveChangesAsync();
await transaction.CommitAsync();
```

---

## Concorrência Otimista (recomendada)

Presume que conflitos são raros. Detecta modificação concorrente via token:

```csharp
public class Habit
{
    public Guid Id { get; init; }
    public string Name { get; set; }

    [Timestamp]
    public uint Version { get; set; } // Postgres: xmin, SQL Server: rowversion
}

modelBuilder.Entity<Habit>()
    .Property(h => h.Version)
    .IsRowVersion();
```
luxo:
1. User A lê habit (Version=1), User B lê habit (Version=1)
2. User A salva → `UPDATE ... WHERE Id=X AND Version=1` → sucesso, Version=2
3. User B tenta salvar → `UPDATE ... WHERE Id=X AND Version=1` → **0 linhas** → `DbUpdateConcurrencyException`

```csharp
try
{
    await _db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Três resoluções:

    // 1. Database wins — recarrega, perde mudanças locais
    foreach (var entry in ex.Entries)
        await entry.ReloadAsync();

    // 2. Client wins — força sobrescrita
    foreach (var entry in ex.Entries)
    {
        entry.OriginalValues.SetValues(entry.GetDatabaseValues()!);
    }
    await _db.SaveChangesAsync();

    // 3. Merge customizado
    var dbValues = await ex.Entries.First().GetDatabaseValuesAsync();
    throw new ConcurrencyConflictException("Hábito foi modificado por outro usuário");
}
```

### Token customizado (sem row version)

```csharp
public class Habit
{
    public Guid ConcurrencyToken { get; set; } = Guid.NewGuid();

    public void UpdateName(string name)
    {
        Name = name;
        ConcurrencyToken = Guid.NewGuid();
    }
}

modelBuilder.Entity<Habit>()
    .Property(h => h.ConcurrencyToken).IsConcurrencyToken();
```

EF inclui o token na cláusula WHERE:
```sql
UPDATE Habits SET Name=@p0, ConcurrencyToken=@p1
WHERE Id=@p2 AND ConcurrencyToken=@p3
```

---

## Concorrência Pessimista

Trava a linha explicitamente. Use com moderação (deadlocks):

```csharp
// Postgres: FOR UPDATE
var habit = await _db.Habits
    .FromSqlRaw("SELECT * FROM \"Habits\" WHERE \"Id\" = {0} FOR UPDATE", id)
    .SingleAsync();
// Linha travada até commit/rollback da transação

// SQL Server: WITH (UPDLOCK)
var habit = await _db.Habits
    .FromSqlRaw("SELECT * FROM Habits WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}", id)
    .SingleAsync();
```

**Quando usar**: alta contenção onde custo de retry > custo de lock. Ex: última vaga de evento — travar por 50ms é melhor que 50 usuários receberem erro de concorrência.

---

## Isolation Levels

```csharp
using var tx = await _db.Database
    .BeginTransactionAsync(IsolationLevel.Serializable);
```

| Nível | Dirty Read | Non-Repeatable Read | Phantom | Performance |
|---|---|---|---|---|
| `ReadUncommitted` | Sim | Sim | Sim | Máxima |
| `ReadCommitted` | Não | Sim | Sim | Alta (padrão) |
| `RepeatableRead` | Não | Não | Sim | Média |
| `Serializable` | Não | Não | Não | Baixa |

**Postgres**: `ReadCommitted` é o padrão. `RepeatableRead` no Postgres já previne phantoms.

---

## Resumo de decisões

| Cenário | Estratégia |
|---|---|
| API REST (comum) | Otimista + catch `DbUpdateConcurrencyException` |
| Reserva de vaga limitada | Pessimista `FOR UPDATE` |
| Relatório de soma | `Snapshot` ou `RepeatableRead` |
| Transferência entre contas | Transação explícita + `Serializable` |
| Salvar formulário | Otimista — "alguém já editou" |
