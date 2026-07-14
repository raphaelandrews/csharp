# 01 — Entity Framework Core

## O que é um ORM e por que ele existe

Um ORM (Object-Relational Mapper) resolve o problema de **impedância objeto-relacional**: seu código C# trabalha com objetos, grafos, herança e coleções. O banco de dados trabalha com tabelas, linhas, colunas e chaves estrangeiras. Traduzir entre esses dois mundos manualmente significa escrever SQL para cada operação, mapear colunas para propriedades manualmente, e gerenciar transações — código repetitivo, propenso a erros e difícil de manter.

O EF Core é o ORM oficial do .NET. Ele atua em três frentes:

1. **Mapeamento**: traduz classes C# (`Habit`, `User`) para tabelas SQL e propriedades para colunas. Configurável via Fluent API ou Data Annotations.
2. **Query translation**: traduz LINQ para SQL. `_db.Habits.Where(h => h.IsActive).ToListAsync()` vira `SELECT * FROM Habits WHERE IsActive = true`. O provider (Npgsql, SqlServer, SQLite) gera o dialect SQL correto.
3. **Change tracking**: rastreia o estado das entidades carregadas do banco. Quando você modifica uma propriedade e chama `SaveChangesAsync()`, o EF Core detecta o que mudou e gera apenas os comandos SQL necessários (UPDATE, INSERT, DELETE) — não a entidade inteira.

---

## DbContext: a unidade de trabalho

O `DbContext` é a classe central do EF Core. Ele representa uma **sessão com o banco de dados** — gerencia conexão, rastreia mudanças e persiste tudo em uma transação quando você chama `SaveChangesAsync()`. Cada instância de `DbContext` é projetada para ser usada por uma única unidade de trabalho (em APIs ASP.NET, uma requisição HTTP = um DbContext).

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
```

Registro no DI — `AddDbContext` configura o DbContext como **Scoped** (uma instância por requisição HTTP):

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));
```

### Fluent API vs Data Annotations

O `OnModelCreating` usa **Fluent API** — métodos encadeados que configuram o mapeamento. A alternativa são **Data Annotations** (atributos `[Key]`, `[Required]`, `[MaxLength]` nas classes de entidade).

Por que Fluent API é preferível:
1. **Separação**: as entidades permanecem POCO (Plain Old CLR Objects) — classes C# puras sem dependência de `System.ComponentModel.DataAnnotations`. O mapeamento fica centralizado no `DbContext`.
2. **Poder expressivo**: Fluent API cobre cenários que Data Annotations não cobrem (índices compostos, filtros, sequences, table splitting, owned types).
3. **Manutenção**: todas as regras de banco estão em um lugar. Se você precisar mudar um índice, não precisa caçar a entidade — é só olhar o `OnModelCreating`.

---

## Migrations: evolução do banco como código

Migrations são arquivos C# que descrevem mudanças incrementais no schema do banco. Cada migration tem um método `Up()` (aplica a mudança) e `Down()` (reverte):

```bash
dotnet ef migrations add InitialCreate   # gera migration
dotnet ef database update                # aplica no banco
dotnet ef migrations remove              # desfaz última (se não aplicada)
dotnet ef migrations list                # lista todas
dotnet ef migrations script -o migrate.sql  # gera script SQL para revisão de DBA
```

**Por que migrations são código C#, não SQL**: isso permite que você edite a migration manualmente para adicionar seeds, criar índices customizados, ou fazer data migration (transformar dados existentes durante o schema change). Um arquivo `.cs` é versionado no Git, revisado em PR, e executado de forma determinística.

---

## Operações CRUD

### CREATE

```csharp
_db.Habits.Add(habit);
await _db.SaveChangesAsync(); // INSERT gerado automaticamente
```

`Add()` registra a entidade no Change Tracker com estado `Added`. `SaveChangesAsync()` gera e executa o SQL.

### READ com Include (eager loading)

```csharp
var habit = await _db.Habits
    .Include(h => h.Completions)          // carrega relacionamento (INNER JOIN)
    .FirstOrDefaultAsync(h => h.Id == id);
```

### READ com projeção (Select)

```csharp
var dtos = await _db.Habits
    .Where(h => h.UserId == userId)
    .Select(h => new HabitDto { Id = h.Id, Name = h.Name })
    .ToListAsync();
// SQL: SELECT Id, Name FROM Habits WHERE UserId = @p0
```

`Select()` gera SQL que traz **apenas as colunas projetadas** — mais eficiente que carregar a entidade inteira.

### UPDATE — entidade rastreada

```csharp
var habit = await _db.Habits.FindAsync(id); // carrega e rastreia
habit.Name = "Novo Nome";                    // EF detecta mudança
await _db.SaveChangesAsync();                // UPDATE só da coluna Name
```

### UPDATE — entidade detached (via API)

```csharp
_db.Habits.Update(updatedHabit); // marca TODAS propriedades como Modified
await _db.SaveChangesAsync();    // UPDATE de todas as colunas
```

`Update()` marca a entidade inteira como modificada — gera `UPDATE SET todas_as_colunas`. Prefira carregar a entidade rastreada e modificar apenas o necessário.

### DELETE com carga

```csharp
var habit = await _db.Habits.FindAsync(id);
_db.Habits.Remove(habit);
await _db.SaveChangesAsync();
```

### DELETE direto (EF Core 7+)

```csharp
await _db.Habits.Where(h => h.Id == id).ExecuteDeleteAsync();
// SQL: DELETE FROM Habits WHERE Id = @p0 — sem carregar entidade
```

`ExecuteDelete` e `ExecuteUpdate` executam SQL direto sem carregar entidades no Change Tracker. Use para operações em lote.

---

## Change Tracker

O Change Tracker mantém um snapshot de cada entidade carregada. No `SaveChangesAsync()`, ele compara o estado atual com o snapshot e gera SQL **apenas para as propriedades que mudaram**:

```csharp
var habit = await _db.Habits.FindAsync(id); // carregado + snapshot
habit.Name = "Novo";
habit.Description = "Desc";
await _db.SaveChangesAsync();
// SQL gerado: UPDATE Habits SET Name=@p0, Description=@p1 WHERE Id=@p2
// Colunas não alteradas (Color, WeeklyFrequency) NÃO aparecem no UPDATE
```

**AsNoTracking**: desabilita o Change Tracker para queries somente leitura. Mais rápido, menos memória:

```csharp
var items = await _db.Habits.AsNoTracking().ToListAsync(); // sem snapshot
```

Use `AsNoTracking()` em todas as queries de listagem e exibição. Só use tracking quando você pretende modificar a entidade e salvar depois.

**ExecuteUpdate**: SQL direto sem carregar entidades (EF Core 7+):

```csharp
await _db.Habits
    .Where(h => !h.IsActive)
    .ExecuteUpdateAsync(s => s.SetProperty(h => h.EndedAt, DateTime.UtcNow));
// Uma única query SQL — zero entidades carregadas em memória
```

---

## Relacionamentos

```csharp
// Include — carrega relacionamento (INNER JOIN implícito)
var habits = await _db.Habits.Include(h => h.Completions).ToListAsync();

// ThenInclude — 2 níveis de profundidade
var users = await _db.Users
    .Include(u => u.Habits).ThenInclude(h => h.Completions)
    .ToListAsync();

// Filtered Include (EF Core 5+) — filtra entidades relacionadas
var habits = await _db.Habits
    .Include(h => h.Completions.Where(c => c.Date >= recentDate))
    .ToListAsync();
```

**Cuidado com múltiplos Includes**: cada `Include` gera um JOIN. Com 3+ Includes em coleções, o resultado é um produto cartesiano — muitas linhas duplicadas transferidas do banco. Use `AsSplitQuery()` para evitar isso (abordado no tópico de otimização de queries na Fase 8).

---

## N+1 e armadilhas comuns

O problema N+1 é o erro de performance mais comum com ORMs:

```csharp
// Errado: N+1 queries — 1 para carregar habits + N para completions
var habits = await _db.Habits.ToListAsync();
foreach (var h in habits)
    Console.WriteLine(h.Completions.Count); // cada acesso dispara query extra!
```

Isso acontece porque `Completions` é uma propriedade de navegação que não foi carregada. Quando você acessa, o EF Core faz **lazy loading** (se configurado) ou simplesmente não retorna nada. Para carregar os dados em uma única query, use `Include()` ou projeção com `Select()`.

Outra armadilha: entidades detached com mesmo ID. Se você criar `new Habit { Id = existingId }` e chamar `Update()`, o EF Core pode tratar como INSERT em vez de UPDATE porque não tem o snapshot original:

```csharp
// Correto: Attach primeiro, modificar depois
_db.Habits.Attach(habit); // anexa ao tracker com estado Unchanged
habit.Name = "Novo";      // EF detecta mudança de propriedade
await _db.SaveChangesAsync(); // UPDATE só do Name
```

---

## Tracking vs NoTracking

| Cenário | Modo recomendado |
|---|---|
| Exibir dados (readonly) | `AsNoTracking()` — sempre |
| Atualizar entidade carregada do banco | Tracking (padrão) |
| Atualizar entidade vinda da API (detached) | `Attach()` + modificar propriedades |
| Bulk delete (>1 registro) | `ExecuteDeleteAsync()` |
| Bulk update (>1 registro) | `ExecuteUpdateAsync()` |

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar o que um ORM faz e por que o EF Core é mais que um "gerador de SQL"
- [ ] Configurar um DbContext com Fluent API e explicar por que é preferível a Data Annotations
- [ ] Criar e aplicar migrations e entender que elas são código C# versionável
- [ ] Diferenciar tracking (modifica e salva) de `AsNoTracking()` (readonly, mais rápido)
- [ ] Identificar e corrigir o problema N+1 usando `Include()` ou `Select()`
- [ ] Usar `ExecuteDeleteAsync()` e `ExecuteUpdateAsync()` para operações em lote
