# 05 — Health Checks e Observabilidade

## Por que health checks vão além de "200 OK"

Um endpoint `/health` que só retorna 200 não diz nada sobre a saúde real da aplicação. Se o banco de dados caiu, o container está rodando mas não consegue servir requisições — e o Kubernetes não sabe disso.

O ASP.NET Core tem um sistema de health checks que verifica dependências reais:

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddNpgSql(
        builder.Configuration.GetConnectionString("Default")!,
        name: "postgres",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "database" })
    .AddUrlGroup(
        new Uri("https://api.externa.com/health"),
        name: "external-api",
        tags: new[] { "external" });

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

Resposta do `/health`:
```json
{
  "status": "Healthy",
  "results": {
    "postgres": { "status": "Healthy", "duration": "00:00:00.015" },
    "external-api": { "status": "Healthy", "duration": "00:00:00.142" }
  }
}
```

Se o PostgreSQL cair:
```json
{
  "status": "Unhealthy",
  "results": {
    "postgres": { "status": "Unhealthy", "description": "Npgsql.PostgresException: ..." },
    "external-api": { "status": "Healthy", "duration": "00:00:00.140" }
  }
}
```

O K8s lê o `status` do JSON e decide se deve reiniciar o pod (liveness) ou remover do load balancer (readiness).

---

## Health checks customizados

```csharp
public class TicketAvailabilityHealthCheck : IHealthCheck
{
    private readonly AppDbContext _db;

    public TicketAvailabilityHealthCheck(AppDbContext db) => _db = db;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        var publishedCount = await _db.Events
            .CountAsync(e => e.Status == EventStatus.Published, ct);

        return publishedCount > 0
            ? HealthCheckResult.Healthy($"{publishedCount} published events")
            : HealthCheckResult.Degraded("No published events found");
    }
}

// Registro:
builder.Services.AddHealthChecks()
    .AddCheck<TicketAvailabilityHealthCheck>("ticket-availability");
```

---

## Endpoints separados para liveness e readiness

```csharp
// Liveness: só verifica se o processo está respondendo (rápido, sem I/O)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // nenhum check — só retorna 200 se o processo está vivo
});

// Readiness: verifica dependências (banco, APIs externas)
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("database")
});
```

No K8s:
```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
```

---

## Observabilidade: Application Insights + OpenTelemetry

O Application Insights é o APM (Application Performance Monitoring) do Azure. Mas o .NET tem suporte nativo ao OpenTelemetry, que é vendor-neutral:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddAspNetCoreInstrumentation();
        tracing.AddEntityFrameworkCoreInstrumentation();
        tracing.AddNpgsql();
        tracing.AddConsoleExporter(); // dev
        // tracing.AddAzureMonitorTraceExporter(); // produção
    })
    .WithMetrics(metrics =>
    {
        metrics.AddAspNetCoreInstrumentation();
        metrics.AddRuntimeInstrumentation();
        metrics.AddConsoleExporter();
    });
```

Isso automaticamente:
- **Tracing**: gera trace ID para cada requisição, propagando entre serviços
- **Metrics**: expõe contadores de GC, uso de memória, threads, taxa de requests
- **Logs**: correlaciona logs com traces via `TraceId`/`SpanId`

### O que você ganha com tracing

```
Requisição HTTP POST /api/events
  TraceId: abc123
  ├── Span: POST /api/events (100ms)
  │   ├── Span: MediatR Send CreateEventCommand (90ms)
  │   │   ├── Span: EventRepository.Add (2ms)
  │   │   └── Span: SaveChangesAsync (80ms)
  │   │       └── Span: Npgsql Execute (75ms)
  │   └── Span: Serilog LogInformation (1ms)
```

Se algo está lento, você sabe exatamente qual span é o gargalo. Isso é o que ferramentas como Application Insights, Dynatrace e Jaeger mostram.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Configurar `AddHealthChecks()` com `AddNpgSql()` para verificar o banco
- [ ] Separar `/health/live` (liveness) de `/health/ready` (readiness)
- [ ] Criar um `IHealthCheck` customizado
- [ ] Explicar o que OpenTelemetry resolve (tracing distribuído, métricas, correlação de logs)
- [ ] Interpretar um trace com múltiplos spans e identificar gargalos
