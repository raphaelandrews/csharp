# 04 — Cache distribuído com Redis

## O problema: "quantas vagas ainda tem?"

No eventhub, a pergunta mais frequente é "quantas vagas restam para a sessão X?". Se cada consulta bater no PostgreSQL, em um pico de vendas (1000 usuários tentando comprar o último lote de ingressos), o banco vira gargalo.

Redis resolve isso: armazena a contagem de vagas em memória, com latência de ~1ms (vs ~10-50ms do PostgreSQL).

---

## Redis no ecossistema .NET

### Pacotes

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
dotnet add package StackExchange.Redis
```

### Configuração

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "eventhub:";
});
```

Connection string: `"localhost:6379"` (local) ou `"eventhub.redis.cache.windows.net:6380,password=...,ssl=True"` (Azure).

---

## Estratégias de cache

### 1. Cache-Aside (mais comum)

```
1. Verifica Redis → se existe, retorna
2. Se não existe, busca no PostgreSQL
3. Salva no Redis com TTL
4. Retorna
```

```csharp
public class CachedEventRepository : IEventRepository
{
    private readonly IEventRepository _inner;
    private readonly IDistributedCache _cache;

    public async Task<int> GetAvailableSeatsAsync(Guid sessionId, CancellationToken ct)
    {
        var cacheKey = $"session:{sessionId}:seats";

        var cached = await _cache.GetStringAsync(cacheKey, ct);
        if (cached is not null)
            return int.Parse(cached);

        var seats = await _inner.GetAvailableSeatsAsync(sessionId, ct);

        await _cache.SetStringAsync(cacheKey, seats.ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(30)
            }, ct);

        return seats;
    }
}
```

**TTL curto (30s)**: disponibilidade de vagas muda rápido. Um cache de 30s significa que no pior caso o usuário vê uma informação 30s desatualizada. Aceitável para listagem, mas **não** para o momento da reserva — a reserva SEMPRE bate no banco com `FOR UPDATE`.

### 2. Write-Through (atualiza cache na escrita)

```csharp
public async Task ReserveSeatAsync(Guid sessionId, Guid userId, CancellationToken ct)
{
    // 1. Reserva no banco (source of truth)
    await _inner.ReserveSeatAsync(sessionId, userId, ct);

    // 2. Atualiza cache (ou invalida)
    var cacheKey = $"session:{sessionId}:seats";
    await _cache.RemoveAsync(cacheKey, ct);
    // Ou recalcula: var newCount = ...; await _cache.SetStringAsync(...);
}
```

Invalidar é mais seguro que atualizar: se a atualização do cache falhar, a próxima leitura vai no banco e reconstrói. Se você atualiza o cache com valor errado, o erro persiste até o TTL expirar.

### 3. Cache para lock distribuído

Redis também é usado para coordenação entre serviços:

```csharp
// Impede que o mesmo usuário reserve 2 ingressos simultaneamente
var lockKey = $"lock:reservation:{userId}";
var lockValue = Guid.NewGuid().ToString();

// Tenta adquirir lock por 10 segundos
if (await _cache.GetStringAsync(lockKey) is null)
{
    await _cache.SetStringAsync(lockKey, lockValue, new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(10)
    });

    try
    {
        // Lógica de reserva
    }
    finally
    {
        // Só remove se o lock ainda é nosso (evita remover lock de outro)
        var current = await _cache.GetStringAsync(lockKey);
        if (current == lockValue)
            await _cache.RemoveAsync(lockKey);
    }
}
```

---

## O que NÃO cachear

- **Dados que precisam de consistência forte**: saldo financeiro, status de pagamento
- **Dados que mudam a cada requisição**: contagem de requisições por usuário (use contadores atômicos, não cache)
- **Dados enormes**: Redis é memória RAM. Um objeto JSON de 10MB consome 10MB de RAM por entrada.

---

## Docker Compose com Redis

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
```

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Implementar Cache-Aside com `IDistributedCache` via Redis
- [ ] Escolher um TTL adequado para dados que mudam com frequência (vagas) vs dados estáveis (catálogo de eventos)
- [ ] Explicar por que a reserva deve bater no banco (não no cache)
- [ ] Invalidar cache após write em vez de atualizar (por segurança)
- [ ] Usar Redis como lock distribuído para prevenir race conditions
