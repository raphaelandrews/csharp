# 04 — Rate Limiting e Throttling

## O problema: abuso de API

Sem rate limiting, um único IP pode fazer 10.000 requisições/segundo no endpoint de busca de eventos, derrubando o banco. Ou pior: um ataque de brute force no endpoint de login.

O ASP.NET Core 7+ tem rate limiting nativo — sem bibliotecas externas.

---

## Fixed Window (o mais simples)

```csharp
// Program.cs
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", config =>
    {
        config.PermitLimit = 100;          // 100 requisições
        config.Window = TimeSpan.FromMinutes(1); // por minuto
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 10;            // +10 na fila de espera
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();
```

```csharp
// Aplicar a endpoints específicos
app.MapPost("/api/auth/login", ...)
    .RequireRateLimiting("fixed");
```

---

## Sliding Window (mais justo que fixed)

Fixed window tem o problema do "reset": se o limite é 100/min, um atacante pode fazer 100 requisições no último segundo do minuto N e 100 no primeiro segundo do minuto N+1 = 200 requisições em 2 segundos.

Sliding window resolve isso considerando uma janela móvel:

```csharp
options.AddSlidingWindowLimiter("sliding", config =>
{
    config.PermitLimit = 100;
    config.Window = TimeSpan.FromMinutes(1);
    config.SegmentsPerWindow = 6; // 6 segmentos de 10s
});
```

---

## Token Bucket (mais flexível)

Permite bursts controlados: você acumula tokens ao longo do tempo e pode gastá-los de uma vez.

```csharp
options.AddTokenBucketLimiter("token", config =>
{
    config.TokenLimit = 100;             // capacidade máxima do bucket
    config.TokensPerPeriod = 10;         // regenera 10 tokens por período
    config.ReplenishmentPeriod = TimeSpan.FromSeconds(1); // por segundo
    config.QueueLimit = 20;
});
```

Útil para APIs que têm picos eventuais (ex: abertura de vendas de ingressos) mas tráfego médio baixo.

---

## Concurrency Limiter

Limita requisições **simultâneas**, não total no período. Útil para endpoints que consomem muitos recursos (ex: geração de relatório).

```csharp
options.AddConcurrencyLimiter("concurrency", config =>
{
    config.PermitLimit = 5; // no máximo 5 requisições simultâneas
    config.QueueLimit = 10;
});
```

---

## Rate limiting por chave (usuário, IP, tenant)

```csharp
options.AddFixedWindowLimiter("per-user", config =>
{
    config.PermitLimit = 1000;
    config.Window = TimeSpan.FromHours(1);
});

// Por IP
app.MapGet("/api/events", ...)
    .RequireRateLimiting("per-user");

// Configurar a chave como IP
options.AddPolicy("per-ip", context =>
    RateLimitPartition.GetFixedWindowLimiter(
        context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
        _ => new FixedWindowRateLimiterOptions { ... }));
```

---

## Headers de rate limit (boas práticas)

O middleware de rate limiting adiciona headers automaticamente:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1690000000
```

Isso permite que o cliente saiba quando pode voltar a fazer requisições sem tomar 429.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Configurar Fixed Window rate limiting no ASP.NET Core
- [ ] Explicar por que Sliding Window é mais justo que Fixed Window
- [ ] Diferenciar Token Bucket (burst controlado) de Concurrency (limite simultâneo)
- [ ] Configurar rate limiting por IP ou por usuário autenticado
- [ ] Interpretar os headers `X-RateLimit-Limit`, `Remaining`, `Reset`
