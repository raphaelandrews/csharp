# 02 — Testes de integração com `WebApplicationFactory`

## O problema: testar uma API de ponta a ponta

Testes unitários são ótimos para verificar lógica isolada. Mas um endpoint `POST /api/auth/register` envolve múltiplas camadas reais que não são testadas unitariamente:

```
HTTP Request → Middleware Pipeline → Endpoint → FluentValidation → Service → Repository → DbContext → PostgreSQL
```

O `WebApplicationFactory<T>` do ASP.NET Core resolve isso criando uma **instância real da sua aplicação em memória**, com um `HttpClient` que envia requisições HTTP sem precisar de um servidor de rede. Nada de `dotnet run` em background, nada de `Process.Start`, nada de portas. A aplicação inteira roda no mesmo processo do teste.

Diferente de outros frameworks onde "teste de integração" significa subir containers Docker e fazer chamadas HTTP reais, o ASP.NET Core tem essa infraestrutura built-in. Isso torna o ciclo de feedback muito mais rápido: um teste de integração com `WebApplicationFactory` leva ~100ms, enquanto um teste com HTTP real sobre TCP leva segundos.

---

## Como funciona (a "mágica")

O `WebApplicationFactory<T>` onde `T` é a classe `Program` da sua API (ou qualquer classe do assembly de entrada) faz o seguinte:

1. **Carrega o assembly da aplicação** e encontra o entry point
2. **Cria um `IHost` real** (mesmo pipeline do `Program.cs`, mesmo DI container)
3. **Substitui o servidor Kestrel** por um `TestServer` in-memory (implementa `IServer` mas não abre socket)
4. **Expõe um `HttpClient`** que envia requisições diretamente para o middleware pipeline, sem rede

```
Teste xUnit
    │
    ▼
HttpClient (in-memory)
    │
    ▼
TestServer (IServer in-memory)
    │
    ▼
Middleware Pipeline (authentication, serilog, cors, etc.)
    │
    ▼
Endpoint → Service → Repository → DbContext
    │
    ▼
PostgreSQL real (ou provider in-memory para testes)
```

O resultado é que você testa o sistema inteiro — middleware, autenticação, validação, serviços, banco de dados — com uma única chamada de método.

---

## Configuração básica

### 1. Instalar os pacotes

```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="9.0.4" />
```

### 2. Criar uma WebApplicationFactory customizada

A factory customizada existe para dois propósitos: configurar o banco de dados de teste e substituir serviços do DI.

```csharp
public class HabitrackApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove o DbContext real (que aponta pro Postgres de dev)
            var dbDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));

            if (dbDescriptor is not null)
                services.Remove(dbDescriptor);

            // Substitui pelo Postgres de teste (banco separado)
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql("Host=localhost;Database=habitrack_test;Username=postgres;Password=postgres"));
        });
    }
}
```

### 3. Escrever o teste

```csharp
public class AuthFlowTests : IClassFixture<HabitrackApiFactory>
{
    private readonly HttpClient _client;

    public AuthFlowTests(HabitrackApiFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Register_Login_AccessProtectedEndpoint()
    {
        // Arrange — registra usuário
        var registerResponse = await _client.PostAsJsonAsync("/api/auth/register", new
        {
            Email = "test@example.com",
            Password = "Senha@123"
        });

        registerResponse.EnsureSuccessStatusCode();
        var auth = await registerResponse.Content
            .ReadFromJsonAsync<AuthResponse>();

        Assert.NotNull(auth?.Token);

        // Act — acessa endpoint protegido com o token
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", auth!.Token);

        var habitsResponse = await _client.GetAsync("/api/habits");

        // Assert
        habitsResponse.EnsureSuccessStatusCode();
        var habits = await habitsResponse.Content
            .ReadFromJsonAsync<HabitDto[]>();

        Assert.NotNull(habits);
        Assert.Empty(habits); // usuário novo, sem hábitos
    }
}
```

Esse teste cobre: registro, geração de JWT, login, middleware de autenticação, endpoint protegido — tudo com uma factory e um `HttpClient`.

---

## Substituindo serviços do DI

A maior vantagem da factory é a capacidade de substituir serviços reais por stubs/mocks — sem precisar de interfaces especiais em produção.

### Cenário 1: Substituir banco real por in-memory

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureServices(services =>
    {
        RemoveService<DbContextOptions<AppDbContext>>(services);

        // EF Core in-memory (rápido, mas não é SQL real — não testa constraints,
        // unique indexes, transações com comportamento real)
        services.AddDbContext<AppDbContext>(options =>
            options.UseInMemoryDatabase("TestDb"));

        // Garante que o banco é criado
        var sp = services.BuildServiceProvider();
        using var scope = sp.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        db.Database.EnsureCreated();
    });
}
```

**Atenção**: `UseInMemoryDatabase` não é um banco relacional real. Ele não aplica constraints de FK, unique indexes, nem suporta transações com isolation levels. Para testes de integração que precisam de comportamento SQL real, use um container de teste (Testcontainers) ou um banco dedicado.

### Cenário 2: Mockar serviço externo

Se sua aplicação chama uma API de terceiros (ex: envio de email), você substitui o serviço na factory:

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureServices(services =>
    {
        RemoveService<IEmailService>(services);

        var mockEmail = new Mock<IEmailService>();
        mockEmail.Setup(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>()))
                 .ReturnsAsync(true);

        services.AddSingleton(mockEmail.Object);
    });
}
```

---

## Estratégia de banco de dados em testes de integração

Três opções, em ordem de maturidade:

### 1. EF Core InMemory (mais simples, menos realista)

```csharp
options.UseInMemoryDatabase(Guid.NewGuid().ToString());
```

**Prós**: zero setup, velocidade máxima. **Contras**: não é SQL — não testa constraints, transações, ou queries LINQ complexas que geram SQL específico.

### 2. SQLite in-memory (equilíbrio)

```csharp
var conn = new SqliteConnection("DataSource=:memory:");
conn.Open();
options.UseSqlite(conn);
```

**Prós**: é um banco relacional real, testa constraints e transações. **Contras**: dialect SQLite ≠ PostgreSQL/SQL Server. `EF.Functions` e algumas queries complexas podem ter comportamento diferente.

### 3. PostgreSQL de teste (mais realista, mais lento)

Banco dedicado com migrations aplicadas a cada execução. Estratégia: cria/dropa o banco inteiro no `IClassFixture<T>`:

```csharp
public class TestDatabaseFixture : IAsyncLifetime
{
    public AppDbContext Db { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql("Host=localhost;Database=habitrack_test;Username=postgres;Password=postgres")
            .Options;

        Db = new AppDbContext(options);
        await Db.Database.EnsureDeletedAsync();   // limpa execução anterior
        await Db.Database.MigrateAsync();         // aplica migrations do zero
    }

    public async Task DisposeAsync()
    {
        await Db.Database.EnsureDeletedAsync();
        await Db.DisposeAsync();
    }
}
```

**Prós**: comportamento SQL idêntico à produção. **Contras**: requer PostgreSQL rodando, cada teste leva ~200ms+.

**Recomendação para o habitrack-api**: usar SQLite in-memory para a maioria dos testes de integração. Manter 1-2 testes críticos (ex: fluxo de autenticação completo) com PostgreSQL real. Isso dá feedback rápido no dia a dia e confiança de que o SQL real funciona no CI/CD.

---

## Resolvendo o EntryPoint do Program.cs

O `WebApplicationFactory<Program>` precisa que `Program` seja acessível do projeto de testes. Com **top-level statements** (nosso `Program.cs` sem `namespace` e sem classe), a classe `Program` é gerada implicitamente como `internal`.

A solução padrão: adicionar uma partial class no final do `Program.cs`:

```csharp
// Última linha do Program.cs
public partial class Program { }
```

Isso expõe `Program` para o assembly de testes via `InternalsVisibleTo`:

```xml
<!-- Habitrack.Api.csproj -->
<ItemGroup>
    <InternalsVisibleTo Include="Habitrack.Api.Tests" />
</ItemGroup>
```

---

## Testando erros e edge cases

```csharp
[Fact]
public async Task Register_DuplicateEmail_ReturnsConflict()
{
    // Primeiro registro
    await _client.PostAsJsonAsync("/api/auth/register", new
    {
        Email = "dupe@example.com",
        Password = "Senha@123"
    });

    // Segundo registro com mesmo email
    var response = await _client.PostAsJsonAsync("/api/auth/register", new
    {
        Email = "dupe@example.com",
        Password = "OutraSenha@456"
    });

    Assert.Equal(HttpStatusCode.Conflict, response.StatusCode);
    var content = await response.Content.ReadFromJsonAsync<JsonElement>();
    Assert.Equal("Email already registered", content.GetProperty("error").GetString());
}

[Fact]
public async Task GetHabits_WithoutToken_Returns401()
{
    var response = await _client.GetAsync("/api/habits");
    Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
}

[Fact]
public async Task CreateHabit_InvalidRequest_Returns400()
{
    _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", validToken);

    var response = await _client.PostAsJsonAsync("/api/habits", new
    {
        Name = "ab",                    // muito curto (< 3 chars)
        WeeklyFrequency = 10            // fora do range 1-7
    });

    Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    var errors = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
    Assert.NotNull(errors);
    Assert.Contains("Name", errors!.Errors.Keys);
    Assert.Contains("WeeklyFrequency", errors.Errors.Keys);
}
```

---

## WebApplicationFactory vs containers Docker

Uma dúvida comum: por que não usar Docker Compose com a aplicação real + Postgres de verdade?

| | WebApplicationFactory | Docker Compose |
|---|---|---|
| Velocidade | ~100ms por teste | ~5s+ por suite (startup) |
| Debug | Breakpoint no próprio código | Breakpoint no código, mas mais complexo |
| Cobertura | Mesma | Mesma |
| Fidelidade | Sem rede real, sem serialização HTTP real | HTTP real, serialização/deserialização reais |
| Setup | Nenhum — roda no processo | docker-compose up, esperar healthy |
| CI/CD | Trivial (só dotnet test) | Precisa de Docker no runner |

A recomendação padrão: `WebApplicationFactory` para a maioria dos testes de integração, com um ou dois smoke tests via Docker em CI para validar a aplicação real em ambiente de produção.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar como `WebApplicationFactory<T>` substitui o Kestrel por um `TestServer` in-memory
- [ ] Criar uma factory customizada que substitui a connection string por um banco de teste
- [ ] Escrever um teste end-to-end que registra, loga e acessa um endpoint protegido
- [ ] Testar edge cases: 401 sem token, 409 conflito, 400 validação
- [ ] Escolher entre InMemory, SQLite ou PostgreSQL real para testes de integração, com argumentos técnicos
- [ ] Saber que `public partial class Program { }` resolve o entry point com top-level statements
