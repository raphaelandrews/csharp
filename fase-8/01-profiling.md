# 01 — Profiling de aplicações .NET

## As ferramentas que você precisa conhecer

O .NET SDK vem com ferramentas de diagnóstico que rodam no terminal — sem IDE, sem GUI, sem Visual Studio. Isso é particularmente relevante para ambientes Linux e containers onde você não tem acesso a ferramentas gráficas.

### dotnet-counters: monitoramento em tempo real

```bash
# Instalar (vem com o SDK, mas pode precisar instalar separado)
dotnet tool install --global dotnet-counters

# Listar processos .NET rodando
dotnet-counters ps

# Monitorar métricas em tempo real
dotnet-counters monitor -p <PID> --refresh-interval 1

# Métricas específicas
dotnet-counters monitor -p <PID> \
    System.Runtime[gc-heap-size,alloc-rate,cpu-usage] \
    Microsoft.AspNetCore.Hosting[requests-per-second,total-requests,current-requests]
```

Saída típica:
```
[System.Runtime]
    GC Heap Size (MB)        42
    Allocation Rate (B / 1s) 1,048,576
    CPU Usage (%)             12
[Microsoft.AspNetCore.Hosting]
    requests-per-second        85
    total-requests           12,430
    current-requests            3
```

### dotnet-trace: tracing de eventos

```bash
dotnet tool install --global dotnet-trace

# Coletar trace por 30 segundos
dotnet-trace collect -p <PID> --duration 00:00:30 -o trace.nettrace

# Analisar no PerfView (Windows) ou dotnet-trace convert + speedscope (Linux)
dotnet-trace convert trace.nettrace --format Speedscope -o trace.speedscope.json
# Abra trace.speedscope.json em https://www.speedscope.app/
```

### dotnet-dump: análise de crash

```bash
dotnet tool install --global dotnet-dump

# Coletar dump (em produção, sem parar o processo)
dotnet-dump collect -p <PID> -o dump.dmp

# Analisar
dotnet-dump analyze dump.dmp
> clrstack        # stack trace gerenciado
> dumpheap -stat  # objetos no heap por tipo
> gcroot <addr>   # o que segura esse objeto vivo
```

### dotnet-gcdump: análise de memória

```bash
dotnet tool install --global dotnet-gcdump

# Coletar GC dump
dotnet-gcdump collect -p <PID> -o gcdump.gcdump

# Analisar no PerfView ou dotnet-gcdump report
dotnet-gcdump report gcdump.gcdump
```

---

## O que procurar em produção

### 1. GC pressure (dotnet-counters)

Se `GC Heap Size` está crescendo monotonicamente sem nunca cair: memory leak.

Se `Allocation Rate` está alto (GB/s) mesmo com pouco tráfego: código alocando objetos desnecessariamente (LINQ materializando coleções inteiras, strings concatenadas em loop, boxing de value types).

### 2. ThreadPool starvation (dotnet-counters)

Se `threadpool-thread-count` está no máximo e `threadpool-queue-length` está crescendo: código síncrono bloqueando threads (`.Result`, `.Wait()`, `Task.Run()` desnecessário).

```bash
dotnet-counters monitor -p <PID> \
    System.Runtime[threadpool-thread-count,threadpool-queue-length,threadpool-completed-items-count]
```

### 3. High CPU (dotnet-trace)

Colete um trace durante o pico e abra no speedscope. Procure por métodos que consomem >5% do tempo total. Foque nos seus métodos (ignorando `System.Threading.Monitor.Wait` e outros internos do runtime).

---

## Métricas com OpenTelemetry (produção)

As ferramentas CLI são para diagnóstico ad-hoc. Para monitoramento contínuo, exporte para Prometheus + Grafana:

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddAspNetCoreInstrumentation();
        metrics.AddRuntimeInstrumentation(); // GC, CPU, threads
        metrics.AddPrometheusExporter();     // /metrics endpoint
    });

app.MapPrometheusScrapingEndpoint();
```

Acesse `http://localhost:8080/metrics` para ver as métricas em formato Prometheus.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Usar `dotnet-counters` para ver GC heap size, allocation rate e requests/s
- [ ] Coletar um trace com `dotnet-trace` e visualizar no speedscope
- [ ] Identificar memory leak (GC heap size cresce sem parar)
- [ ] Identificar threadpool starvation (queue cresce, threads no máximo)
- [ ] Exportar métricas via OpenTelemetry para Prometheus
