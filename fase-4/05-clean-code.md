# 05 — Clean Code e convenções da comunidade .NET

## Por que convenções importam em C#

Go tem `gofmt` — o formatador oficial resolve 90% das discussões de estilo. C# não tem um `gofmt` com a mesma autoridade cultural. O que existe é um conjunto de convenções que **a comunidade espera que você siga**, e violá-las passa uma impressão de "não conhece a linguagem" em code reviews e entrevistas.

As convenções .NET são **mais estritas que Go** em nomenclatura e estrutura, e **menos estritas que Java** em verbosidade. O equilibrio é: explícito o suficiente para um novato entender, conciso o suficiente para um sênior não se irritar.

---

## Nomenclatura: a convenção mais visível

| Elemento | Convenção | Exemplo |
|---|---|---|
| Classe, struct, record | PascalCase | `HabitService`, `UserDto` |
| Interface | PascalCase com `I` prefixo | `IHabitRepository`, `IUnitOfWork` |
| Método | PascalCase | `GetByIdAsync`, `CreateAsync` |
| Propriedade | PascalCase | `IsActive`, `WeeklyFrequency` |
| Campo privado | `_camelCase` (underscore prefix) | `_uow`, `_httpClient` |
| Parâmetro | camelCase | `userId`, `cancellationToken` |
| Variável local | camelCase | `var habit = ...` |
| Constante | PascalCase | `public const int MaxRetries = 3;` |
| Enum member | PascalCase | `Status.Active`, `Status.Pending` |
| Método assíncrono | Sufixo `Async` | `GetAllAsync()`, `SaveChangesAsync()` |

### O underscore em campos privados `_camelCase`

Essa é talvez a convenção mais idiomaticamente C#. Em Go, campos privados são `camelCase`. Em C#, o underscore é o marcador visual de "isso é estado interno da classe, não uma propriedade pública":

```csharp
public class HabitService
{
    private readonly IUnitOfWork _uow; // campo privado
    //    ^                            // underscore = "sou privado"

    public HabitService(IUnitOfWork uow) // parâmetro = camelCase normal
    {
        _uow = uow; // this.uow sem precisar de this
    }
}
```

**Por que isso é útil**: sem underscore, você precisa de `this.uow = uow` para diferenciar o campo do parâmetro. Com underscore, `_uow = uow` resolve sem `this`. Menos ruído.

---

## Async/Await conventions

### Sufixo Async é obrigatório por convenção

```csharp
// Correto
public async Task<HabitDto> GetByIdAsync(Guid id) { ... }
public async Task SaveAsync() { ... }

// Incorreto (a menos que seja interface pública de biblioteca que não deve expor implementação)
public async Task<HabitDto> GetById(Guid id) { ... }
```

A exceção: quando você **não quer expor que o método é assíncrono na API pública** de uma biblioteca. Mas em código de aplicação, o sufixo `Async` é esperado.

### CancellationToken sempre como último parâmetro

```csharp
// Correto
Task<HabitDto> GetByIdAsync(Guid id, Guid userId, CancellationToken ct = default);

// Incorreto
Task<HabitDto> GetByIdAsync(CancellationToken ct, Guid id, Guid userId);
```

### Não use async void (exceto em event handlers)

```csharp
// Correto: async Task
public async Task HandleAsync() { ... }

// Incorreto: async void (exceções são perdidas, não pode ser awaited)
public async void HandleAsync() { ... }

// Única exceção válida: event handlers de UI (WinForms/WPF)
button.Click += async (s, e) => { ... }; // async void implícito, mas é o padrão da UI
```

---

## Estrutura de arquivos

### Um tipo público por arquivo

Essa é uma convenção forte em .NET. Em Go, é comum ter múltiplos tipos no mesmo arquivo. Em C#, **cada classe/interface/record público tem seu próprio arquivo**:

```
# Go: models.go contém Habit, HabitCompletion, User
# C#:
Models/Habit.cs
Models/HabitCompletion.cs
Models/User.cs
```

A exceção são tipos pequenos e fortemente relacionados: `Dtos.cs` com múltiplos records de request/response é aceitável. `Models.cs` com múltiplas entidades também (embora em projetos maiores, cada entidade vai para seu arquivo).

### Ordem dos membros na classe

A convenção (seguida pelo StyleCop e pelo analisador do Visual Studio):

```csharp
public class Example
{
    // 1. Campos privados
    private readonly IUnitOfWork _uow;
    private int _counter;

    // 2. Construtor
    public Example(IUnitOfWork uow) { _uow = uow; }

    // 3. Propriedades públicas
    public string Name { get; set; }

    // 4. Métodos públicos
    public async Task DoSomethingAsync() { ... }

    // 5. Métodos privados
    private void Helper() { ... }
}
```

---

## Tratamento de null

Com **nullable reference types** habilitados (nosso projeto tem `<Nullable>enable</Nullable>`), o compilador distingue tipos que podem ser null:

```csharp
string name;       // não-nullable (compilador avisa se pode ser null)
string? name;      // nullable (compilador avisa se você não verificar)
```

### Padrões idiomáticos de null handling

```csharp
// 1. Null-conditional operator (?.)
var length = habit?.Name?.Length;

// 2. Null-coalescing operator (??)
var color = request.Color ?? "#3B82F6";

// 3. Null-coalescing assignment (??=) — C# 8+
_habits ??= new HabitRepository(_db);

// 4. Pattern matching com 'is'
if (habit is null) return Results.NotFound();
if (habit is not null) { ... }

// 5. ArgumentNullException.ThrowIfNull — .NET 6+
ArgumentNullException.ThrowIfNull(request);
```

### Evite: null-forgiving operator (!) sem certeza

```csharp
// Ruim: suprime o warning sem verificar
var name = habit!.Name; // se habit for null, NullReferenceException em produção

// Bom: verifica antes
if (habit is null) throw new NotFoundException();
var name = habit.Name; // compilador sabe que não é null aqui
```

---

## Expression-bodied members

Para métodos e propriedades de uma linha, C# oferece sintaxe concisa com `=>`:

```csharp
// Propriedade calculada
public bool HasCompletions => Completions.Count > 0;

// Método simples
public void Add(Habit habit) => _db.Habits.Add(habit);

// Construtor
public HabitRepository(AppDbContext db) => _db = db;

// Destructor
~HabitRepository() => _db.Dispose();
```

**Quando usar**: para membros de uma linha. Se precisar de duas linhas, use bloco `{ }` normal. A exceção é switch expressions que podem ter múltiplas linhas.

---

## LINQ: preferir método sintaxe sobre query sintaxe

```csharp
// Query syntax (parece SQL, menos idiomático em C#)
var active = from h in habits
             where h.IsActive
             orderby h.CreatedAt descending
             select h;

// Method syntax (mais idiomático, compõe melhor)
var active = habits
    .Where(h => h.IsActive)
    .OrderByDescending(h => h.CreatedAt);
```

A query syntax existe para familiaridade com SQL, mas a method syntax é preferida na comunidade .NET porque compõe melhor e é mais consistente com o resto do código C#.

---

## Implicit typing (var)

```csharp
// Use var quando o tipo é óbvio do lado direito
var habit = new Habit();          // óbvio
var habits = await _db.Habits.ToListAsync(); // óbvio

// Use tipo explícito quando não for óbvio
HabitDto? result = await service.GetByIdAsync(id, ct); // o tipo não é óbvio do nome do método
decimal totalPrice = items.Sum(i => i.Price * i.Quantity); // var seria int? decimal? não óbvio
```

A regra de ouro: se você consegue saber o tipo lendo o lado direito, use `var`. Se precisa pensar, explicite o tipo.

---

## File-scoped namespaces

C# 10 introduziu namespaces com escopo de arquivo (em vez de bloco):

```csharp
// C# tradicional (indentação extra, chave extra)
namespace Habitrack.Api.Services
{
    public class HabitService { }
}

// C# 10+ (file-scoped, menos indentação)
namespace Habitrack.Api.Services;

public class HabitService { }
```

O time do .NET recomendou file-scoped namespaces como padrão para novos projetos. Nosso projeto já usa isso. Toda a indentação que seria gasta no namespace vai para o código de verdade.

---

## Nullable enable e ImplicitUsings

Todo projeto .NET moderno deve ter estas duas flags:

```xml
<Nullable>enable</Nullable>
<ImplicitUsings>enable</ImplicitUsings>
```

**Nullable enable** ativa NRT (Nullable Reference Types). O compilador analisa fluxo de null e emite warnings. Isso é o mais próximo que C# chega de `Option<T>` do Rust ou `nil` safety. É uma feature de compilador, não de runtime — em runtime, referências ainda podem ser null.

**ImplicitUsings** adiciona global usings automaticamente (`System`, `System.Collections.Generic`, `System.Linq`, `System.Threading.Tasks`, etc.) sem poluir cada arquivo com `using System;`. Menos ruído, mais código.

---

## `.editorconfig`: o gofmt do C#

Embora não seja obrigatório como `gofmt`, o `.editorconfig` é o padrão da indústria para definir e aplicar convenções de código em projetos .NET:

```ini
root = true

[*.cs]
# Nomenclatura
dotnet_naming_rule.private_fields_underscore.severity = suggestion
dotnet_naming_rule.private_fields_underscore.style = underscore

# Preferências de linguagem
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_expression_bodied_methods = true:silent
dotnet_style_explicit_tuple_names = true:suggestion
dotnet_style_predefined_type_for_locals_parameters_members = true:suggestion
dotnet_style_null_propagation = true:suggestion

# Formatação
indent_size = 4
indent_style = space
```

O analisador do Roslyn (compilador C#) respeita o `.editorconfig` e emite warnings/suggestions em tempo de compilação. É o mais próximo de `gofmt` que o .NET oferece.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Nomear corretamente: `PascalCase` para público, `_camelCase` para campos privados, `I` prefixo em interfaces
- [ ] Usar sufixo `Async` em todos os métodos que retornam `Task`/`ValueTask`
- [ ] Aplicar null-conditional (`?.`), null-coalescing (`??`) e pattern matching (`is null`) em vez de `if (x == null)`
- [ ] Preferir LINQ method syntax sobre query syntax
- [ ] Saber quando usar `var` vs tipo explícito
- [ ] Manter `<Nullable>enable</Nullable>` e `<ImplicitUsings>enable</ImplicitUsings>` no `.csproj`
- [ ] Estruturar uma classe na ordem: campos → construtor → propriedades → métodos públicos → métodos privados
