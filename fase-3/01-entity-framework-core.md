# 01 — Entity Framework Core

EF Core é o ORM oficial do .NET. Mapeia classes C# para tabelas SQL, traduz LINQ em queries SQL e gerencia o ciclo de vida das entidades com `DbContext` + Change Tracker.

---

## DbContext

```csharp
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

// Registro no DI:
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));
```

Prefira **Fluent API** (`OnModelCreating`) sobre Data Annotations — mantém entidades como POCO puro e centraliza a config do banco.

---

## Migrations

```bash
dotnet tool install --global dotnet-ef

dotnet ef migrations add InitialCreate
dotnet ef database update
dotnet ef migrations remove           # desfaz última (se não aplicada)
dotnet ef migrations list             # lista todas
dotnet ef migrations script -o migrate.sql  # script SQL pra revisão de DBA
```

A migration é código C# (`.cs`), não SQL. Pode ser editada manualmente para seeds, índices customizados ou data migration.

---

## Operações CRUD

```csharp
// CREATE
_db.Habits.Add(habit);
await _db.SaveChangesAsync();

// READ com Include (eager loading)
var habit = await _db.Habits
    .Include(h => h.Completions)
    .FirstOrDefaultAsync(h => h.Id == id);

// READ com projeção (SELECT só as colunas necessárias)
var dtos = await _db.Habits
    .Where(h => h.UserId == userId)
    .Select(h => new HabitDto { Id = h.Id, Name = h.Name })
    .ToListAsync();

// UPDATE — entidade tracked (FindAsync carrega e rastreia)
var habit = await _db.Habits.FindAsync(id);
habit.Name = "Novo Nome"; // EF detecta mudança
await _db.SaveChangesAsync();

// UPDATE — detached (via API)
_db.Habits.Update(updatedHabit); // marca TODAS propriedades como modified
await _db.SaveChangesAsync();

// DELETE com carga
var habit = await _db.Habits.FindAsync(id);
_db.Habits.Remove(habit);
await _db.SaveChangesAsync();

// DELETE direto (EF Core 7+)
await _db.Habits.Where(h => h.Id == id).ExecuteDeleteAsync();
```

---

## Change Tracker

EF mantém snapshot das entidades carregadas. No `SaveChangesAsync()`, compara estado atual com snapshot e gera SQL **só pras colunas alteradas**:

```csharp
var habit = await _db.Habits.FindAsync(id); // tracked + snapshot
habit.Name = "Novo";
habit.Description = "Desc";
await _db.SaveChangesAsync();
// SQL: UPDATE Habits SET Name=@p0, Description=@p1 WHERE Id=@p2
// Apenas colunas alteradas, não UPDATE completo
```

**AsNoTracking** — sem snapshot, sem Change Tracker (leve, rápido):

```csharp
var items = await _db.Habits.AsNoTracking().ToListAsync(); // readonly
```

**ExecuteUpdate / ExecuteDelete** — SQL direto (EF Core 7+), zero entidades carregadas:

```csharp
await _db.Habits
    .Where(h => !h.IsActive)
    .ExecuteUpdateAsync(s => s.SetProperty(h => h.EndedAt, DateTime.UtcNow));
```

---

## Relacionamentos

```csharp
// Include — INNER JOIN implícito
var habits = await _db.Habits.Include(h => h.Completions).ToListAsync();

// ThenInclude — 2 níveis
var users = await _db.Users
    .Include(u => u.Habits).ThenInclude(h => h.Completions)
    .ToListAsync();

// Filtered Include (EF Core 5+)
var habits = await _db.Habits
    .Include(h => h.Completions.Where(c => c.Date >= recentDate))
    .ToListAsync();

// LEFT JOIN
var data = await (
    from h in _db.Habits
    join c in _db.HabitCompletions on h.Id equals c.HabitId into completions
    from co in completions.DefaultIfEmpty()
    select new { h.Name, CompletionDate = (DateOnly?)co.Date }
).ToListAsync();
```

---

## N+1 e armadilhas

```csharp
// ❌ N+1: 1 query pra habits + N queries pra completions
var habits = await _db.Habits.ToListAsync();
foreach (var h in habits)
    Console.WriteLine(h.Completions.Count); // query extra!

// ✓ Include — 1 query com JOIN
var habits = await _db.Habits.Include(h => h.Completions).ToListAsync();

// ❌ Entidade detached com mesmo ID = risco de INSERT falso
var habit = new Habit { Id = existingId };
_db.Habits.Update(habit); // EF pode tratar como INSERT

// ✓ Attach primeiro
_db.Habits.Attach(habit);
habit.Name = "Novo"; // detecta mudança de propriedade
await _db.SaveChangesAsync();
```

---

## Tracking vs NoTracking

| Cenário | Modo |
|---|---|
| Exibir dados (readonly) | `AsNoTracking()` — sempre |
| Atualizar entidade carregada | Tracked (padrão) |
| Atualizar entidade detached | `Attach()` + mudanças |
| Bulk delete | `ExecuteDeleteAsync()` |
| Bulk update | `ExecuteUpdateAsync()` |
