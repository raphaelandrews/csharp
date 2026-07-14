# 03 — Resiliência: Circuit Breaker, Retry e Polly

## O problema: falhas em cascata

```
Serviço A → Serviço B (lento, timeout 30s)
                ↑
Serviço C → Serviço D (depende do B)
```

Se o Serviço B começa a demorar 30s para responder, o Serviço A acumula threads esperando. Logo o A também fica lento. O C, que depende do A, também degrada. Uma falha local vira falha sistêmica.

A biblioteca **Polly** (mantida pela .NET Foundation) implementa os patterns de resiliência que previnem isso. É a biblioteca padrão do ecossistema .NET para políticas de tolerância a falhas.

---

## Retry: tente de novo (falhas transientes)

Falhas transientes são temporárias: um timeout de rede, um deadlock no banco, um 503 do load balancer. Retry resolve:

```csharp
// Polly v8+
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential, // 200ms → 400ms → 800ms
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<PostgresException>(ex => ex.IsTransient)
    })
    .Build();

await pipeline.ExecuteAsync(async ct =>
{
    await httpClient.PostAsJsonAsync("/api/external/confirm", data, ct);
}, cancellationToken);
```

### Exponential backoff

Sem backoff: tenta em 200ms, depois 200ms, depois 200ms — 3 tentativas em 600ms. Com isso, se o serviço estava com pico, as 3 tentativas batem durante o mesmo pico e todas falham.

Com backoff exponencial: 200ms → 400ms → 800ms = 1.4s total. Dá tempo do serviço remoto se recuperar.

### Quando NÃO usar retry

- Operações não-idempotentes: se a chamada "criar pagamento" falhou mas o pagamento foi criado, o retry duplica o pagamento
- Timeout de negócio: se o usuário espera resposta em 2s, 3 retries com backoff podem exceder esse limite

---

## Circuit Breaker: pare de tentar (falhas persistentes)

Se o serviço externo está fora do ar, retry só piora: cada tentativa consome recursos (threads, conexões) e todas vão falhar. O Circuit Breaker "abre o circuito" depois de N falhas consecutivas e rejeita chamadas imediatamente por um período:

```
Estado: Closed (normal) → 5 falhas consecutivas → Open (rejeita tudo)
                                                      │
                                              30s depois
                                                      │
                                                      ▼
                                            Half-Open (testa 1 chamada)
                                              │                │
                                         sucesso          falha
                                              │                │
                                              ▼                ▼
                                          Closed          Open (de novo)
```

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,              // 50% de falhas abre o circuito
        MinimumThroughput = 10,           // precisa de pelo menos 10 chamadas para avaliar
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(60), // fica aberto por 60s
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
    })
    .Build();
```

O Circuit Breaker **previne o efeito domino**: em vez de 1000 threads bloqueadas esperando timeout de um serviço morto, as chamadas falham imediatamente (com `BrokenCircuitException`) e o serviço sobrevivente continua respondendo.

---

## Timeout: não espere para sempre

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();
```

Sem timeout explícito, uma chamada HTTP pode esperar indefinidamente (ou até o timeout do OS, que pode ser minutos). Timeout é a política mais simples e mais negligenciada.

---

## Compondo políticas

Polly permite compor múltiplas políticas em pipeline:

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(5))                  // 1. Timeout primeiro
    .AddRetry(new RetryStrategyOptions                    // 2. Retry dentro do timeout
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions  // 3. Circuit breaker depois
    {
        FailureRatio = 0.5,
        BreakDuration = TimeSpan.FromSeconds(30)
    })
    .Build();
```

A ordem importa: Timeout limita cada tentativa individual. Retry tenta de novo se timeout ou falha. Circuit Breaker abre se os retries continuam falhando.

---

## Polly + MassTransit (resiliência na mensageria)

MassTransit já integra Polly para retry em consumidores:

```csharp
cfg.ReceiveEndpoint("notificacoes-email", e =>
{
    e.ConfigureConsumer<ReservationConfirmedConsumer>(context);

    e.UseMessageRetry(r => r
        .Exponential(3, TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(1))
    );
});
```

Se o consumidor lançar exceção, MassTransit automaticamente:
1. Retry com backoff exponencial (1s → 3s → 9s)
2. Se esgotar retries, move para a Dead Letter Queue (`_error`)
3. Você pode inspecionar a DLQ pela UI do RabbitMQ e reenfileirar manualmente

---

## Resiliência no eventhub

| Camada | Política | Por que |
|---|---|---|
| HTTP → API externa de pagamento | Retry + Circuit Breaker + Timeout | Pagamento pode falhar transitoriamente |
| Consumidor de notificações | Retry (MassTransit built-in) | Envio de email pode falhar |
| Redis (cache) | Timeout curto (500ms) + fallback para banco | Cache é otimização, não deve bloquear |

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Configurar retry com backoff exponencial usando Polly
- [ ] Explicar os 3 estados do Circuit Breaker (Closed, Open, Half-Open)
- [ ] Compor Timeout → Retry → Circuit Breaker em pipeline
- [ ] Usar retry no MassTransit (`UseMessageRetry`)
- [ ] Saber quando NÃO usar retry (operações não-idempotentes)
- [ ] Explicar a diferença entre falha transiente (retry resolve) e falha persistente (circuit breaker resolve)
