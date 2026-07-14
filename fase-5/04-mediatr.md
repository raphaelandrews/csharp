# 04 — MediatR: o padrão Mediator no ecossistema .NET

## Por que MediatR é quase onipresente

MediatR é uma biblioteca criada por Jimmy Bogard (mesmo autor do AutoMapper) que implementa o padrão Mediator. Em 2026, é difícil encontrar um projeto .NET corporativo com CQRS que não use MediatR — ele se tornou tão onipresente quanto o Entity Framework.

A biblioteca é pequena (menos de 2000 linhas de código) e faz basicamente duas coisas:
1. **Roteia mensagens para handlers** (`IMediator.Send()` → `IRequestHandler<TRequest, TResponse>`)
2. **Fornece um pipeline de behaviors** (validação, logging, retry, transaction)

O valor real não está no código da biblioteca — está no **padrão arquitetural** que ela impõe: cada caso de uso vira uma classe isolada, com dependências explícitas e responsabilidade única.

---

## O problema que o Mediator resolve

Sem MediatR, um controller/endpoint típico em .NET:

```csharp
// Sem MediatR — o controller depende diretamente de múltiplos serviços
app.MapPost("/api/events", async (
    CreateEventRequest request,
    IEventRepository repo,
    IValidator<CreateEventRequest> validator,
    ILogger<Program> logger,
    CancellationToken ct) =>
{
    var validation = await validator.ValidateAsync(request);
    if (!validation.IsValid)
        return Results.ValidationProblem(...);

    logger.LogInformation("Creating event: {Title}", request.Title);

    var @event = Event.Create(request.Title, ...);
    repo.Add(@event);
    await repo.SaveChangesAsync(ct);

    return Results.Created($"/api/events/{@event.Id}", EventDto.FromEntity(@event));
});
```

Problemas:
1. O endpoint conhece detalhes de validação, logging, domínio e persistência
2. Se amanhã você adicionar cache, retry policy, ou notificação por email, o endpoint cresce
3. Não há como testar o fluxo completo sem mockar 4 dependências
4. Cada endpoint duplica lógica de validação, logging e error handling

Com MediatR:

```csharp
// Com MediatR — o endpoint só sabe enviar um comando
app.MapPost("/api/events", async (
    CreateEventCommand cmd,
    IMediator mediator,
    CancellationToken ct) =>
{
    var result = await mediator.Send(cmd, ct);
    return Results.Created($"/api/events/{result.Id}", result);
});

// Toda a lógica está no handler:
public class CreateEventCommandHandler : IRequestHandler<CreateEventCommand, EventDto>
{
    private readonly IEventRepository _repo;

    public async Task<EventDto> Handle(CreateEventCommand cmd, CancellationToken ct)
    {
        var @event = Event.Create(cmd.Title, cmd.Description, cmd.OrganizerId);
        _repo.Add(@event);
        await _repo.SaveChangesAsync(ct);
        return EventDto.FromEntity(@event);
    }
}
```

O endpoint virou uma casca fina. O handler é uma classe focada, testável isoladamente, com dependências explícitas.

---

## Como o MediatR funciona

### 1. Registro no DI

```csharp
// Program.cs
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(CreateEventCommand).Assembly);
});
```

Isso escaneia o assembly da Application e registra automaticamente todos os `IRequestHandler<T, R>`, `INotificationHandler<T>`, e behaviors.

### 2. Envio de comandos/queries

```csharp
// IMediator.Send() para request/response (commands e queries)
var result = await mediator.Send(new CreateEventCommand(...), ct);

// IMediator.Publish() para notificações (domain events)
await mediator.Publish(new EventPublished(eventId, DateTime.UtcNow), ct);
```

`Send()` é para **1 handler** (command ou query). `Publish()` é para **múltiplos handlers** (notificações/eventos).

### 3. Pipeline de behaviors

Behaviors são o recurso mais poderoso (e menos compreendido) do MediatR. Eles envolvem cada handler em um pipeline de middlewares:

```
Request
    │
    ▼
┌──────────────┐
│ Validation   │ ← ValidationBehavior: valida o request automaticamente
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Logging      │ ← LoggingBehavior: loga entrada/saída/duração
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Transaction  │ ← TransactionBehavior: abre transação, commit/rollback
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Handler      │ ← Seu handler (CreateEventCommandHandler)
└──────────────┘
```

Cada behavior implementa `IPipelineBehavior<TRequest, TResponse>`:

```csharp
// Application/Common/Behaviors/ValidationBehavior.cs
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!_validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);
        var results = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(context, ct)));

        var failures = results
            .Where(r => r.Errors.Any())
            .SelectMany(r => r.Errors)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next(); // chama o próximo behavior (ou o handler)
    }
}
```

Registrado no DI:

```csharp
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

**A mágica**: você escreve o behavior uma vez, e ele se aplica automaticamente a **todo** handler registrado. Adicionar validação, logging ou transação a um caso de uso novo não requer código extra — é automático.

### Behaviors comuns em projetos .NET

| Behavior | O que faz |
|---|---|
| `ValidationBehavior` | Executa FluentValidation antes do handler; lança `ValidationException` se inválido |
| `LoggingBehavior` | Loga o nome do request, parâmetros e duração com Serilog/ILogger |
| `TransactionBehavior` | Abre `BeginTransactionAsync()` antes e `CommitAsync()`/`RollbackAsync()` depois |
| `PerformanceBehavior` | Loga um warning se o handler demorar mais que N ms |
| `RetryBehavior` | Usa Polly para retry em falhas transientes |
| `AuthorizationBehavior` | Verifica claims/permissões antes de executar o handler |

### Exemplo: PerformanceBehavior

```csharp
public class PerformanceBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;
    private const int WarningThresholdMs = 500;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        var response = await next();
        sw.Stop();

        if (sw.ElapsedMilliseconds > WarningThresholdMs)
        {
            _logger.LogWarning(
                "Slow request: {RequestName} took {Elapsed}ms",
                typeof(TRequest).Name, sw.ElapsedMilliseconds);
        }

        return response;
    }
}
```

Isso é aplicado automaticamente a **todos** os handlers sem modificar nenhum handler existente. Esse é o poder do pipeline.

---

## MediatR no contexto do CQRS

A combinação CQRS + MediatR é tão comum que as pessoas confundem os dois. Vamos separar:

| CQRS | MediatR |
|---|---|
| Padrão arquitetural (separar leitura de escrita) | Biblioteca que implementa o padrão Mediator |
| Define que commands e queries são diferentes | Roteia objetos para seus handlers |
| Pode ser implementado sem MediatR | Pode ser usado sem CQRS (ex: só pra notifications) |
| Decisão de design | Decisão de implementação |

Na prática, a maioria dos projetos .NET escolhe os dois juntos porque:
- `IRequest<T>` do MediatR é o contrato natural para commands e queries
- `IRequestHandler<T, R>` é o contrato natural para handlers de caso de uso
- `INotification<T>` é o contrato natural para domain events
- Behaviors resolvem cross-cutting concerns (validação, logging) sem poluir os handlers

---

## Estrutura de arquivos típica com MediatR + CQRS

```
Application/
├── Events/
│   ├── Commands/
│   │   ├── CreateEvent/
│   │   │   ├── CreateEventCommand.cs       ← record : IRequest<EventDto>
│   │   │   ├── CreateEventCommandHandler.cs ← IRequestHandler<,>
│   │   │   └── CreateEventCommandValidator.cs ← AbstractValidator<>
│   │   ├── UpdateEvent/
│   │   └── CancelEvent/
│   └── Queries/
│       ├── GetEventById/
│       │   ├── GetEventByIdQuery.cs
│       │   └── GetEventByIdQueryHandler.cs
│       └── GetEvents/
│           ├── GetEventsQuery.cs
│           └── GetEventsQueryHandler.cs
├── Common/
│   ├── Behaviors/
│   │   ├── ValidationBehavior.cs
│   │   ├── LoggingBehavior.cs
│   │   └── TransactionBehavior.cs
│   ├── Exceptions/
│   └── Interfaces/
└── DependencyInjection.cs
```

Cada feature (CreateEvent, GetEvents) é uma pasta com command/query + handler + validator. Isso facilita encontrar o código relacionado a uma feature sem navegar por dezenas de arquivos.

---

## MediatR vs "Mediator é desnecessário"

Assim como o Repository Pattern em cima de EF Core, o MediatR tem seus críticos. O argumento principal: **você não precisa de uma biblioteca para chamar um método**.

```csharp
// Sem MediatR: chama o handler diretamente
var handler = new CreateEventHandler(repo);
var result = await handler.Handle(cmd, ct);

// Com MediatR: indireção via mediator
var result = await mediator.Send(cmd, ct);
```

A diferença parece cosmética. Mas o MediatR não está ali para substituir uma chamada de método — está ali para:

1. **Pipeline de behaviors**: você ganha validação, logging e transação em todos os handlers sem código repetido. Isso é impossível sem um mediador central.

2. **Desacoplamento real**: o endpoint não conhece o handler. Se você decidir trocar o handler, mover para outro assembly, ou adicionar decorators, o endpoint não muda.

3. **Convenção sobre configuração**: todo dev .NET que já usou MediatR sabe que `IRequest<T>` → `IRequestHandler<T, R>`. É um padrão estabelecido que reduz o tempo de onboarding.

4. **Notificações (pub/sub in-process)**: `mediator.Publish()` é um sistema de eventos in-process poderoso. Múltiplos handlers podem reagir ao mesmo evento (`EventPublished` → enviar email, atualizar cache, log de auditoria).

**Meu take**: em projetos pequenos, MediatR adiciona indireção desnecessária. Em projetos médios/grandes com CQRS, ele se paga pelo pipeline de behaviors e pela convenção. Como o eventhub é um projeto de aprendizado, vamos usar MediatR para entender o padrão — é o que as vagas pedem.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Registrar MediatR via `AddMediatR()` e escanear um assembly
- [ ] Criar um `IRequest<T>` (command ou query) e seu `IRequestHandler<T, R>`
- [ ] Implementar um `IPipelineBehavior<T, R>` (ex: validação automática)
- [ ] Explicar a diferença entre `IMediator.Send()` (1 handler) e `IMediator.Publish()` (N handlers)
- [ ] Diferenciar CQRS (padrão) de MediatR (biblioteca) em uma entrevista
- [ ] Argumentar quando MediatR se paga (pipeline de behaviors, desacoplamento) vs quando é indireção desnecessária
