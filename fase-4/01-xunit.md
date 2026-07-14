# 01 — xUnit: o framework de testes do ecossistema .NET

## Por que xUnit (e por que não NUnit ou MSTest)

O .NET tem três frameworks de teste principais, e entender por que xUnit venceu ajuda a entender a cultura da comunidade:

| Framework | Origem | Paradigma | Uso atual |
|---|---|---|---|
| MSTest | Microsoft (2005) | Atributos + classes base | Projetos legados, templates do Visual Studio |
| NUnit | Port do JUnit (2002) | Atributos + [SetUp]/[TearDown] | Ainda usado, especialmente em projetos mais antigos |
| xUnit | Criado pelos autores do NUnit (2007) | Constructor/Dispose, sem atributos de setup | **Padrão de facto hoje**, usado pelo próprio time do ASP.NET Core |

xUnit foi criado por James Newkirk e Brad Wilson — os mesmos que criaram o NUnit 2. Eles escreveram o xUnit para corrigir o que consideravam falhas de design no NUnit:

1. **NUnit usa `[SetUp]`/`[TearDown]`** — métodos mágicos que rodam antes/depois de cada teste. xUnit usa o construtor e `IDisposable` da própria classe, seguindo o princípio de que o framework de teste deve usar os mesmos mecanismos da linguagem em vez de inventar seus próprios.
2. **NUnit permite herança de fixtures** — classes de teste podem herdar de classes base com `[SetUp]`. Isso leva a hierarquias profundas e confusas. xUnit evita herança: cada teste recebe suas dependências via construtor, como qualquer classe C# bem escrita.
3. **xUnit isola cada teste** — cria uma instância nova da classe de teste para cada método `[Fact]`. Isso elimina estado compartilhado acidental entre testes (um problema clássico de NUnit/MSTest quando alguém seta um campo no `[SetUp]` e não limpa).

O time do ASP.NET Core testa o próprio framework com xUnit. Isso diz muito sobre qual é o padrão da casa.

---

## O modelo mental do xUnit

Cada classe de teste é instanciada **uma vez por método de teste**. Esse é o detalhe mais importante para internalizar:

```
Classe: HabitServiceTests
Métodos: CreateAsync_ValidRequest_ReturnsDto
         CreateAsync_NullName_ThrowsException
         GetByIdAsync_NotFound_ReturnsNull

Execução:
  new HabitServiceTests() → roda CreateAsync_ValidRequest_ReturnsDto → descarta
  new HabitServiceTests() → roda CreateAsync_NullName_ThrowsException → descarta
  new HabitServiceTests() → roda GetByIdAsync_NotFound_ReturnsNull → descarta
```

Isso significa que o **construtor é seu `[SetUp]`** e `Dispose()` (via `IDisposable`) é seu **`[TearDown]`**. Sem mágica, sem atributos — só C# idiomático.

### [Fact] vs [Theory]

```csharp
// [Fact]: teste sem parâmetros. Um fato que é sempre verdadeiro.
[Fact]
public async Task GetByIdAsync_ExistingId_ReturnsHabit()
{
    var result = await service.GetByIdAsync(existingId, userId);
    Assert.NotNull(result);
}

// [Theory]: teste parametrizado. Um teorema que deve ser verdadeiro
// para múltiplos conjuntos de dados.
[Theory]
[InlineData("", false)]           // nome vazio → inválido
[InlineData("ab", false)]         // nome muito curto → inválido
[InlineData("Exercício", true)]   // nome válido
public void ValidateName_ReturnsExpected(string name, bool expected)
{
    var result = validator.Validate(name);
    Assert.Equal(expected, result);
}
```

`[Theory]` + `[InlineData]` é o equivalente a *table-driven tests* do Go. Mas o xUnit vai além com `[MemberData]` (dados de um método/propriedade) e `[ClassData]` (dados de uma classe `IEnumerable<object[]>`).

### Tipos de assertions

xUnit usa o modelo `Assert.Expected(actual)` — ao contrário de NUnit que usa `Assert.That(actual, Is.EqualTo(expected))`. As principais:

```csharp
Assert.Equal(expected, actual);          // igualdade via IEquatable<T> ou override Equals
Assert.NotEqual(expected, actual);
Assert.True(condition);
Assert.False(condition);
Assert.Null(obj);
Assert.NotNull(obj);
Assert.Same(expected, actual);           // referência (não igualdade)
Assert.Contains(item, collection);
Assert.DoesNotContain(item, collection);
Assert.Empty(collection);
Assert.NotEmpty(collection);

// Exceções
var ex = await Assert.ThrowsAsync<NotFoundException>(() => service.GetByIdAsync(id, ct));
Assert.Contains("não encontrado", ex.Message);

// Collections (ordem, equivalência)
Assert.Equal(expectedList, actualList);                    // mesma ordem
Assert.Equivalent(expectedList, actualList);               // independe de ordem
Assert.All(collection, item => Assert.True(item.IsValid));

// Strings com diff
Assert.Equal("Habit was not found", ex.Message);
// Se falhar, o xUnit mostra um diff linha a linha, não só "não são iguais"
```

### IAsyncLifetime

Para setup/cleanup **assíncrono** (ex: criar banco, popular dados, dropar tabelas), o `IDisposable` não serve porque `Dispose` é síncrono. xUnit resolve com `IAsyncLifetime`:

```csharp
public class DatabaseTests : IAsyncLifetime
{
    public async Task InitializeAsync()
    {
        // Roda antes de cada teste
        await CreateTestDatabase();
        await SeedData();
    }

    public async Task DisposeAsync()
    {
        // Roda depois de cada teste
        await DropTestDatabase();
    }

    [Fact]
    public async Task SomeTest() { ... }
}
```

---

## Fixtures: compartilhando estado entre testes

xUnit define três níveis de compartilhamento:

### 1. Por classe (Class Fixture)

Compartilha uma instância entre **todos os testes de uma classe**. Útil para conexões de banco, HttpClient, etc.

```csharp
// O fixture — cria uma vez, compartilha entre todos os testes da classe
public class DatabaseFixture : IDisposable
{
    public AppDbContext DbContext { get; }

    public DatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql("Host=localhost;Database=habitrack_test;...")
            .Options;
        DbContext = new AppDbContext(options);
    }

    public void Dispose()
    {
        DbContext.Dispose();
    }
}

// A classe de teste
public class HabitRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public HabitRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture; // mesma instância para todos os testes
    }

    [Fact]
    public async Task GetActiveByUser_ReturnsOnlyActiveHabits()
    {
        var repo = new HabitRepository(_fixture.DbContext);
        var habits = await repo.GetActiveByUserAsync(userId);
        Assert.All(habits, h => Assert.True(h.IsActive));
    }
}
```

O ciclo de vida: `DatabaseFixture()` → todos os testes da classe → `DatabaseFixture.Dispose()`.

### 2. Por coleção (Collection Fixture)

Compartilha uma instância entre **múltiplas classes de teste**. Útil quando todos os testes compartilham um banco e você quer criar/dropar uma vez só.

```csharp
[CollectionDefinition("Database collection")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database collection")]
public class HabitRepositoryTests { ... }

[Collection("Database collection")]
public class UserRepositoryTests { ... }
```

**Atenção**: testes na mesma collection **não rodam em paralelo** entre si. Isso é intencional — se ambos tocam o mesmo banco, paralelismo causaria interferência.

### 3. Assembly Fixture (via IAsyncLifetime na collection)

Para setup que roda uma vez por assembly inteiro. Não é nativo do xUnit, mas implementável com uma collection fixture que contém todos os testes.

---

## Test output e logging

xUnit suprime `Console.WriteLine` por padrão. Para ver output durante o teste:

```csharp
// Opção 1: Injetar ITestOutputHelper (só funciona em testes xUnit, não no código de produção)
public class HabitServiceTests
{
    private readonly ITestOutputHelper _output;

    public HabitServiceTests(ITestOutputHelper output)
    {
        _output = output;
    }

    [Fact]
    public void TestWithOutput()
    {
        _output.WriteLine("Debugging info: user={0}", userId);
    }
}
```

```bash
# Rodar com output visível
dotnet test --logger "console;verbosity=detailed"
```

---

## Rodando testes no dotnet CLI

```bash
dotnet test                           # roda todos os testes
dotnet test --filter "Category=Unit"  # filtra por trait
dotnet test --filter "FullyQualifiedName~HabitService"  # filtra por nome
dotnet test -l "console;verbosity=detailed"  # output detalhado
dotnet test --collect:"XPlat Code Coverage"  # cobertura
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=lcov
dotnet watch test                     # re-roda ao salvar (útil durante TDD)
```

### Traits (categorização)

```csharp
[Fact]
[Trait("Category", "Unit")]
public void SomeUnitTest() { }

[Fact]
[Trait("Category", "Integration")]
public void SomeIntegrationTest() { }
```

```bash
dotnet test --filter "Category=Unit"       # só unitários
dotnet test --filter "Category!=Integration"  # tudo menos integração
```

---

## Estrutura de projeto de testes

O padrão da comunidade .NET:

```
habitrack-api/
├── Habitrack.Api.csproj
├── habitrack-api.sln
└── ...
habitrack-api.Tests/
├── Habitrack.Api.Tests.csproj    # referência ao projeto principal
├── Usings.cs                     # global usings (opcional)
├── Services/
│   ├── HabitServiceTests.cs
│   └── AuthServiceTests.cs
├── Repositories/
│   ├── HabitRepositoryTests.cs
│   └── UserRepositoryTests.cs
└── Integration/
    └── AuthFlowTests.cs
```

O projeto de teste **referencia o projeto principal** e os pacotes de teste:

```xml
<!-- Habitrack.Api.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
    <PackageReference Include="xunit" Version="2.9.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="3.0.2" />
    <PackageReference Include="coverlet.collector" Version="6.0.4" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\habitrack-api\Habitrack.Api.csproj" />
  </ItemGroup>
</Project>
```

Os três pacotes essenciais:
- `Microsoft.NET.Test.Sdk` — infraestrutura de teste do .NET (roda os testes)
- `xunit` — o framework
- `xunit.runner.visualstudio` — integração com VSTest (permite `dotnet test` + IDEs)
- `coverlet.collector` — cobertura de código

---

## xUnit vs Go testing (para referência)

| Aspecto | Go | xUnit |
|---|---|---|
| Função de teste | `func TestXxx(t *testing.T)` | `[Fact] void Xxx()` |
| Setup | antes do loop principal | construtor da classe |
| Teardown | `t.Cleanup(func())` ou `defer` | `IDisposable.Dispose()` |
| Table-driven | slice de structs + loop | `[Theory]` + `[MemberData]` |
| Assertions | `if` + `t.Errorf()` | `Assert.Equal(...)` |
| Paralelismo | `t.Parallel()` | `dotnet test` roda em paralelo por padrão |
| Sub-testes | `t.Run("name", func(t *testing.T))` | `[Theory]` com nomes |

A diferença mais impactante: em Go, testes falham com `t.Error/t.Fatal` e continuam rodando. Em xUnit, uma assertion que falha **interrompe aquele teste imediatamente** (fail fast). Se você quer testar múltiplas condições independentes, use `[Theory]` com múltiplos `[InlineData]` em vez de múltiplos `Assert` no mesmo `[Fact]`.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar por que o xUnit instancia a classe de teste a cada `[Fact]` e por que isso é melhor que `[SetUp]`/`[TearDown]`
- [ ] Usar `[Theory]` + `[InlineData]` para testes parametrizados
- [ ] Implementar `IClassFixture<T>` para compartilhar dependências caras (DbContext, HttpClient)
- [ ] Usar `ITestOutputHelper` para log durante testes
- [ ] Rodar `dotnet test --filter` para executar testes específicos
- [ ] Diferenciar `Assert.Equal` (ordem importa), `Assert.Equivalent` (ordem não importa) e `Assert.Same` (referência)
