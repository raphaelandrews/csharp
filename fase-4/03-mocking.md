# 03 — Mocking com Moq e NSubstitute

## Por que mockar

O problema: testar `HabitService` isoladamente requer um `IUnitOfWork`. Mas `IUnitOfWork` depende de `AppDbContext`, que depende de um banco de dados real. Para testar o serviço **sem banco**, você cria um mock.

Mock = objeto que finge implementar uma interface, permitindo que você:
1. Configure retornos esperados (`Setup`)
2. Verifique interações (`Verify`)
3. Lance exceções sob demanda

Sem mock:

```
HabitService → IUnitOfWork → AppDbContext → PostgreSQL (precisa do banco rodando)
```

Com mock:

```
HabitService → IUnitOfWork (mock: respostas pré-programadas)
```

---

## Moq vs NSubstitute: qual escolher

O ecossistema .NET tem dois frameworks dominantes de mocking. Ambos resolvem o mesmo problema, mas com filosofias diferentes.

### Moq (mais popular, mais verboso)

```csharp
var mock = new Mock<IHabitRepository>();

mock.Setup(r => r.GetByIdAsync(habitId, It.IsAny<CancellationToken>()))
    .ReturnsAsync(new Habit { Id = habitId, Name = "Test" });

var repo = mock.Object;
```

**Filosofia**: explícito e configurável. `Setup`, `Verify`, `It.IsAny<>()`, `It.Is<T>(predicate)`. A sintaxe é mais verbosa, mas deixa claro o que está acontecendo.

### NSubstitute (mais fluente, menos verboso)

```csharp
var repo = Substitute.For<IHabitRepository>();

repo.GetByIdAsync(habitId, Arg.Any<CancellationToken>())
    .Returns(new Habit { Id = habitId, Name = "Test" });
```

**Filosofia**: sintaxe fluente. O mock é configurado chamando o método diretamente e encadeando `.Returns()`. Menos código, mais legível.

| Critério | Moq | NSubstitute |
|---|---|---|
| Curva de aprendizado | Média (sintaxe própria) | Baixa (sintaxe fluente) |
| Verbosidade | Alta | Baixa |
| Mercado | 80%+ dos projetos | Crescendo |
| Performance | Rápida | Comparável |
| Strict mode | `MockBehavior.Strict` | Não tem |
| Protected members | Suporta | Suporta |

**Recomendação para o roadmap**: usar Moq. É o que aparece na maioria das vagas e projetos corporativos. A sintaxe é mais verbosa, mas você vai encontrar Moq em praticamente toda codebase .NET que encontrar.

---

## Moq na prática

### Setup: configurando retornos

```csharp
var uowMock = new Mock<IUnitOfWork>();
var habitRepoMock = new Mock<IHabitRepository>();
var userRepoMock = new Mock<IUserRepository>();

uowMock.Setup(u => u.Habits).Returns(habitRepoMock.Object);
uowMock.Setup(u => u.Users).Returns(userRepoMock.Object);

// Configura o repositório para retornar um hábito específico
habitRepoMock
    .Setup(r => r.GetByIdAsync(habitId, It.IsAny<CancellationToken>()))
    .ReturnsAsync(new Habit { Id = habitId, UserId = userId, Name = "Exercício" });

// Configura SaveChangesAsync para retornar 1 (1 linha afetada)
uowMock
    .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
    .ReturnsAsync(1);
```

**`It.IsAny<T>()`** — casa com qualquer valor do tipo. Use quando o argumento específico não importa.

**`It.Is<T>(predicate)`** — casa com valores que satisfazem um predicado:

```csharp
habitRepoMock
    .Setup(r => r.Add(It.Is<Habit>(h => h.Name == "Exercício")))
    .Verifiable(); // verifica que foi chamado depois
```

### Verify: verificando interações

```csharp
// Verifica que o método foi chamado exatamente 1 vez
habitRepoMock.Verify(
    r => r.Add(It.IsAny<Habit>()),
    Times.Once());

// Verifica que NUNCA foi chamado
habitRepoMock.Verify(
    r => r.Delete(It.IsAny<Habit>()),
    Times.Never());

// Verifica que foi chamado exatamente 2 vezes
uowMock.Verify(
    u => u.SaveChangesAsync(It.IsAny<CancellationToken>()),
    Times.Exactly(2));
```

### Callbacks: executar lógica durante o mock

```csharp
var habits = new List<Habit>();

habitRepoMock
    .Setup(r => r.Add(It.IsAny<Habit>()))
    .Callback<Habit>(h => habits.Add(h)); // captura o que foi adicionado

// Depois do teste:
Assert.Single(habits);
Assert.Equal("Exercício", habits[0].Name);
```

### Exceções: forçando falhas

```csharp
habitRepoMock
    .Setup(r => r.GetByIdAsync(invalidId, It.IsAny<CancellationToken>()))
    .ThrowsAsync(new NotFoundException("Habit not found"));

uowMock
    .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
    .ThrowsAsync(new DbUpdateException("Connection refused"));
```

---

## Exemplo completo: teste unitário do HabitService.CreateAsync

```csharp
public class HabitServiceTests
{
    private readonly Mock<IUnitOfWork> _uowMock;
    private readonly Mock<IHabitRepository> _habitRepoMock;
    private readonly Mock<IUserRepository> _userRepoMock;
    private readonly HabitService _service;

    public HabitServiceTests()
    {
        _uowMock = new Mock<IUnitOfWork>();
        _habitRepoMock = new Mock<IHabitRepository>();
        _userRepoMock = new Mock<IUserRepository>();

        _uowMock.Setup(u => u.Habits).Returns(_habitRepoMock.Object);
        _uowMock.Setup(u => u.Users).Returns(_userRepoMock.Object);
        _uowMock.Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
                .ReturnsAsync(1);

        _service = new HabitService(_uowMock.Object);
    }

    [Fact]
    public async Task CreateAsync_ValidRequest_ReturnsDtoWithGeneratedId()
    {
        // Arrange
        var request = new CreateHabitRequest
        {
            Name = "Exercício",
            Description = "30 min de corrida",
            WeeklyFrequency = 5,
            Color = "#FF0000"
        };
        var userId = Guid.NewGuid();

        Habit? capturedHabit = null;
        _habitRepoMock
            .Setup(r => r.Add(It.IsAny<Habit>()))
            .Callback<Habit>(h => capturedHabit = h);

        // Act
        var result = await _service.CreateAsync(request, userId);

        // Assert
        Assert.NotNull(result);
        Assert.Equal("Exercício", result.Name);
        Assert.Equal(userId, capturedHabit!.UserId);
        Assert.True(result.CurrentStreak == 0);

        _habitRepoMock.Verify(r => r.Add(It.IsAny<Habit>()), Times.Once());
        _uowMock.Verify(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once());
    }

    [Fact]
    public async Task GetByIdAsync_NotFound_ReturnsNull()
    {
        // Arrange
        var habitId = Guid.NewGuid();
        _habitRepoMock
            .Setup(r => r.GetByIdWithCompletionsAsync(habitId, It.IsAny<CancellationToken>()))
            .ReturnsAsync((Habit?)null);

        // Act
        var result = await _service.GetByIdAsync(habitId, Guid.NewGuid());

        // Assert
        Assert.Null(result);
    }

    [Fact]
    public async Task GetByIdAsync_DifferentUser_ReturnsNull()
    {
        // Arrange
        var habitId = Guid.NewGuid();
        var ownerId = Guid.NewGuid();
        var otherUserId = Guid.NewGuid();

        _habitRepoMock
            .Setup(r => r.GetByIdWithCompletionsAsync(habitId, It.IsAny<CancellationToken>()))
            .ReturnsAsync(new Habit
            {
                Id = habitId,
                UserId = ownerId,
                Name = "Test",
                Completions = new List<HabitCompletion>()
            });

        // Act
        var result = await _service.GetByIdAsync(habitId, otherUserId);

        // Assert — usuário diferente, deve retornar null
        Assert.Null(result);
    }
}
```

---

## Strict vs Loose Mocking

Moq tem dois modos:

```csharp
var strict = new Mock<IHabitRepository>(MockBehavior.Strict);
// Qualquer chamada não configurada → exceção
// Útil para detectar chamadas inesperadas

var loose = new Mock<IHabitRepository>(MockBehavior.Loose);
// Comportamento padrão
// Chamadas não configuradas retornam default(T) ou Task vazia
// Menos setup, mas pode esconder chamadas inesperadas
```

**Regra prática**: use `Loose` (padrão) para a maioria dos testes. Use `Strict` quando você suspeita que chamadas indesejadas estão acontecendo ou quando o contrato da interface é crítico para a correção (ex: métodos que disparam efeitos colaterais, como envio de email).

---

## Mock xUnit, não Mock o que você não controla

Uma tentação comum é mockar `DateTime.UtcNow`. **Não faça isso.** Em vez disso, abstraia:

```csharp
// Ruim: mockar DateTime via framework (frágil, reflection hack)
// Bom: abstrair com interface
public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
}

public class SystemDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}

// No teste:
var dateMock = new Mock<IDateTimeProvider>();
dateMock.Setup(d => d.UtcNow).Returns(new DateTime(2026, 1, 1));
```

Mesma lógica para `Guid.NewGuid()`, `Random`, `File`, etc. Se não é sua interface, não mocke — abstraia.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Configurar um mock com `Mock<T>` e injetar com `.Object`
- [ ] Usar `Setup().ReturnsAsync()` para métodos assíncronos
- [ ] Usar `It.IsAny<T>()` e `It.Is<T>(predicate)` para matching de argumentos
- [ ] Verificar interações com `Verify(Times.Once())`
- [ ] Capturar argumentos com `Callback<T>()`
- [ ] Explicar a diferença entre Moq (sintaxe Setup/Verify) e NSubstitute (sintaxe fluente)
- [ ] Saber quando usar `MockBehavior.Strict` vs `Loose`
