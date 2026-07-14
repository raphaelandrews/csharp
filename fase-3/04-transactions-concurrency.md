# 04 — Transações e Concorrência

## O problema que transações resolvem

Uma transação garante **atomicidade**: ou todas as operações são persistidas juntas, ou nenhuma é. Sem transações, uma falha no meio do caminho deixa o banco em estado inconsistente — metade dos dados salvos, metade perdidos.

O EF Core gerencia transações automaticamente em dois níveis:

1. **Transação implícita**: `SaveChangesAsync()` por si só é transacional. Tudo que o Change Tracker registrou (Adds, Updates, Deletes) é enviado em uma única transação. Se qualquer comando falhar, tudo é desfeito.
2. **Transação explícita**: você controla o escopo manualmente quando precisa de múltiplos `SaveChangesAsync()` ou múltiplos DbContexts na mesma transação.

---

## Transação implícita (cobre 90% dos casos)

```csharp
_db.Habits.Add(habit);
_db.HabitCompletions.Add(completion);
await _db.SaveChangesAsync(); // INSERT de ambos com COMMIT automático
// Se o segundo INSERT falhar, o primeiro é desfeito automaticamente
```

Isso funciona porque `SaveChangesAsync()` abre uma transação no banco, executa todos os comandos gerados pelo Change Tracker, e dá COMMIT. Se qualquer comando falhar, o driver do banco (Npgsql, SqlClient) faz ROLLBACK automático.

---

## Transação explícita (múltiplos SaveChanges)

Quando você precisa de controle fino — dois `SaveChangesAsync()` que devem ser atômicos:

```csharp
using var transaction = await _db.Database.BeginTransactionAsync();

try
{
    _db.Habits.Add(habit);
    await _db.SaveChangesAsync();    // INSERT 1

    _db.HabitCompletions.Add(completion);
    await _db.SaveChangesAsync();    // INSERT 2

    await transaction.CommitAsync(); // AMBOS confirmados atomicamente
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

Por que dois `SaveChangesAsync()` separados? Cenário real: você cria um hábito, precisa do `Id` gerado pelo banco para criar a completion. O primeiro `SaveChangesAsync()` gera o Id (via `RETURNING` ou `IDENTITY`), e o segundo usa esse Id. Mas ambos precisam ser atômicos — se o segundo falhar, o primeiro deve ser desfeito.

### Múltiplos DbContexts na mesma transação

```csharp
using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync();
using var transaction = await conn.BeginTransactionAsync();

// DbContext 1 — catálogo de eventos
await using var db1 = new AppDbContext(
    new DbContextOptionsBuilder<AppDbContext>().UseNpgsql(conn).Options);
db1.Database.UseTransaction(transaction);

// DbContext 2 — log de auditoria
await using var db2 = new AuditDbContext(
    new DbContextOptionsBuilder<AuditDbContext>().UseNpgsql(conn).Options);
db2.Database.UseTransaction(transaction);

db1.Habits.Add(habit);
await db1.SaveChangesAsync();
db2.AuditLogs.Add(log);
await db2.SaveChangesAsync();

await transaction.CommitAsync(); // AMBOS contexts na mesma transação
```

`UseTransaction()` compartilha a transação do ADO.NET entre múltiplos DbContexts. O segredo: todos compartilham a mesma `NpgsqlConnection`, e a transação é associada a essa conexão.

---

## Concorrência: o problema dos updates simultâneos

Dois usuários carregam o mesmo hábito, ambos modificam, ambos salvam. Sem controle de concorrência, o último sobrescreve o primeiro silenciosamente (last write wins).

Existem duas estratégias: **otimista** (presume que conflitos são raros, detecta no save) e **pessimista** (trava o registro, impede acesso simultâneo).

---

## Concorrência Otimista (recomendada para APIs REST)

### Como funciona

Cada entidade tem um token de versão. Quando você carrega a entidade, o token é X. Quando você salva, o EF Core adiciona `WHERE Version = X` na cláusula do UPDATE. Se outro usuário salvou antes, o token no banco agora é X+1, e seu `WHERE Version = X` não encontra a linha — `DbUpdateConcurrencyException` é lançada.

### Implementação com Timestamp (PostgreSQL: xmin)

```csharp
public class Habit
{
    public Guid Id { get; init; }

    [Timestamp]
    public uint Version { get; set; } // PostgreSQL: mapeia para xmin (oculto, automático)
}

modelBuilder.Entity<Habit>()
    .Property(h => h.Version)
    .IsRowVersion();
```

O `[Timestamp]` + `IsRowVersion()` fazem o EF Core incluir automaticamente a coluna de versão na cláusula WHERE de todo UPDATE e DELETE.

Fluxo de concorrência:
```
1. Usuário A lê Habit (Version=1), Usuário B lê Habit (Version=1)
2. Usuário A salva → UPDATE ... WHERE Id=X AND Version=1 → sucesso, Version=2
3. Usuário B tenta salvar → UPDATE ... WHERE Id=X AND Version=1 → 0 linhas afetadas
   → EF Core lança DbUpdateConcurrencyException
```

### Tratamento da exceção de concorrência

```csharp
try
{
    await _db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Três estratégias de resolução:

    // 1. Database wins — recarrega do banco, perde mudanças locais
    foreach (var entry in ex.Entries)
        await entry.ReloadAsync();

    // 2. Client wins — força sobrescrita (ignora versão do banco)
    foreach (var entry in ex.Entries)
    {
        entry.OriginalValues.SetValues(entry.GetDatabaseValues()!);
    }
    await _db.SaveChangesAsync(); // tenta de novo

    // 3. Merge customizado — notifica o usuário do conflito
    var dbValues = await ex.Entries.First().GetDatabaseValuesAsync();
    throw new ConcurrencyConflictException("Este registro foi modificado por outro usuário");
}
```

### Token customizado (sem dependência de rowversion do banco)

```csharp
public class Habit
{
    public Guid ConcurrencyToken { get; set; } = Guid.NewGuid();

    public void UpdateName(string name)
    {
        Name = name;
        ConcurrencyToken = Guid.NewGuid(); // novo token a cada mudança
    }
}

modelBuilder.Entity<Habit>()
    .Property(h => h.ConcurrencyToken).IsConcurrencyToken();
```

O EF Core inclui o token na cláusula WHERE:
```sql
UPDATE Habits SET Name=@p0, ConcurrencyToken=@p1
WHERE Id=@p2 AND ConcurrencyToken=@p3
```

O token customizado é útil quando você não pode usar `xmin` (PostgreSQL) ou `ROWVERSION` (SQL Server), ou quando precisa de controle explícito sobre quando a versão muda.

---

## Concorrência Pessimista (trava a linha)

Em vez de detectar o conflito depois, a concorrência pessimista **previne** o conflito travando a linha no banco. Enquanto a transação estiver aberta, ninguém mais pode modificar aquela linha:

```csharp
// PostgreSQL: SELECT ... FOR UPDATE
var habit = await _db.Habits
    .FromSqlRaw("SELECT * FROM \"Habits\" WHERE \"Id\" = {0} FOR UPDATE", id)
    .SingleAsync();
// Linha travada até o COMMIT/ROLLBACK da transação

// SQL Server: SELECT ... WITH (UPDLOCK, ROWLOCK)
var habit = await _db.Habits
    .FromSqlRaw("SELECT * FROM Habits WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}", id)
    .SingleAsync();
```

**Quando usar pessimista**: cenários de alta contenção onde o custo de retry (otimista) é maior que o custo do lock. Exemplo: reservar a última vaga de um evento — é melhor travar a linha por 50ms e dar a vaga para o primeiro que chegou, do que 50 usuários tentarem reservar, 49 receberem erro de concorrência, e todos terem que tentar de novo.

---

## Isolation Levels

O isolation level controla como transações concorrentes interagem entre si:

```csharp
using var tx = await _db.Database
    .BeginTransactionAsync(IsolationLevel.Serializable);
```

| Nível | Dirty Read | Non-Repeatable | Phantom | Performance |
|---|---|---|---|---|
| `ReadUncommitted` | Sim | Sim | Sim | Máxima |
| `ReadCommitted` | Não | Sim | Sim | Alta (padrão no PostgreSQL) |
| `RepeatableRead` | Não | Não | Sim | Média |
| `Serializable` | Não | Não | Não | Baixa |

**Dirty Read**: ler dados não commitados de outra transação. `ReadUncommitted` permite — os outros níveis bloqueiam.

**Non-Repeatable Read**: ler a mesma linha duas vezes na mesma transação e obter valores diferentes (outra transação modificou entre as leituras). `RepeatableRead` previne isso.

**Phantom**: uma query retorna linhas diferentes em execuções subsequentes (outra transação inseriu/deletou). Apenas `Serializable` previne completamente.

No PostgreSQL, `ReadCommitted` é o padrão e cobre 95% dos casos. `RepeatableRead` no PostgreSQL já previne phantoms (diferente do padrão SQL) — é uma implementação mais forte que o spec exige.

---

## Resumo de decisões

| Cenário | Estratégia |
|---|---|
| API REST (comum) | Otimista + catch `DbUpdateConcurrencyException` |
| Reserva de vaga limitada | Pessimista `FOR UPDATE` |
| Relatório de soma consistente | `RepeatableRead` |
| Transferência entre contas | Transação explícita + `Serializable` |
| Salvar formulário (vários usuários) | Otimista — "alguém já editou este registro" |

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar que `SaveChangesAsync()` é transacional por padrão — sem BEGIN/COMMIT explícito
- [ ] Implementar uma transação com múltiplos DbContexts compartilhando a mesma conexão
- [ ] Configurar um token de concorrência (`[Timestamp]` / `IsRowVersion()`) e tratar `DbUpdateConcurrencyException`
- [ ] Diferenciar concorrência otimista (detecta conflito, retry) de pessimista (trava linha, previne conflito)
- [ ] Escolher o isolation level correto baseado nas anomalias que você quer prevenir
- [ ] Saber que `RepeatableRead` no PostgreSQL já previne phantoms (comportamento acima do padrão SQL)
