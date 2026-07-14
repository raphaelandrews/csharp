# 02 — Mensageria assíncrona: RabbitMQ e Kafka

## Por que mensageria

REST é síncrono: o serviço A chama o B e espera a resposta. Se o B estiver lento ou fora do ar, o A sofre. Mensageria é assíncrona: A publica uma mensagem e segue a vida. B processa quando puder.

Isso resolve três problemas:
1. **Desacoplamento temporal**: A e B não precisam estar online ao mesmo tempo
2. **Resiliência**: se B cair, as mensagens ficam na fila e são processadas quando B voltar
3. **Escala**: você pode ter N instâncias de B processando a mesma fila em paralelo

---

## RabbitMQ vs Kafka: qual escolher

| Critério | RabbitMQ | Kafka |
|---|---|---|
| Modelo | Fila (broker empurra mensagem) | Log (consumidor puxa do offset) |
| Roteamento | Exchanges + bindings (flexível) | Tópicos (simples) |
| Ordenação | Por fila | Por partição |
| Reprocessamento | Não nativo (mensagem consumida = removida) | Nativo (mensagens persistem, rebobinar offset) |
| Throughput | ~20k msg/s | ~1M msg/s |
| Curva de aprendizado | Baixa | Alta (ZooKeeper/KRaft, partições, offsets) |
| Caso de uso típico | Comandos entre serviços (eventhub) | Event sourcing, analytics, streams |

**Para o eventhub: RabbitMQ.** É mais simples de configurar, o modelo de exchanges encaixa bem com `ReservaConfirmada` → múltiplos consumidores (email, SMS, auditoria), e não precisamos de throughput de milhões de mensagens.

---

## Conceitos fundamentais do RabbitMQ

### Exchange + Queue + Binding

```
Publisher ──▶ Exchange ──▶ Queue ──▶ Consumer
                 │              │
                 │ (binding)    │
                 └──────────────┘
```

- **Exchange**: recebe mensagens do publisher e decide para quais filas enviar
- **Queue**: buffer onde as mensagens ficam até serem consumidas
- **Binding**: regra que conecta uma exchange a uma fila (`routing key = "reserva.confirmada"`)
- **Routing Key**: string que o publisher define e o exchange usa para rotear

### Tipos de Exchange

| Tipo | Comportamento |
|---|---|
| **Direct** | Roteia pela routing key exata. `routing_key = "reserva.confirmada"` → fila com binding `"reserva.confirmada"` |
| **Fanout** | Ignora routing key, envia para TODAS as filas vinculadas. Útil para broadcast. |
| **Topic** | Roteia por padrão com wildcards. `"reserva.*"` → filas com binding `"reserva.#"` |
| **Headers** | Roteia por headers da mensagem (raro). |

Para o eventhub: **Direct Exchange** → `reserva.confirmada` → Queue `notificacoes-email`.

### Conceitos operacionais

| Conceito | O que é |
|---|---|
| **Durable** | Fila/exchange sobrevive a restart do broker |
| **Persistent** | Mensagem é salva em disco (DeliveryMode=2) |
| **Ack** | Consumidor confirma que processou. Sem ack, RabbitMQ re-enfileira. |
| **Nack** | Consumidor rejeita mensagem. Pode re-enfileirar ou descartar. |
| **Prefetch** | Quantas mensagens o consumidor pega de uma vez. Prefetch=1 garante ordem. |
| **DLX (Dead Letter Exchange)** | Para onde vão mensagens rejeitadas ou expiradas |

---

## Implementação no eventhub

### Pacotes NuGet

```bash
dotnet add package RabbitMQ.Client     # cliente oficial (ambos serviços)
dotnet add package MassTransit          # abstração de alto nível (recomendado)
```

**MassTransit vs RabbitMQ.Client cru**: MassTransit é para RabbitMQ o que EF Core é para SQL — abstrai detalhes de conexão, retry, serialização, e adiciona patterns como Consumer, Saga, e Request/Response. Vamos usar MassTransit.

### Publicador (EventHub.Api)

```csharp
// Application/Events/Commands/CreateReservation/
public record ReservationConfirmed(Guid ReservationId, Guid EventId, Guid UserId, string Email)
    : IIntegrationEvent; // MassTransit marker interface

// Handler (simplificado — dentro do handler de reserva)
await _publishEndpoint.Publish(new ReservationConfirmed(
    reservation.Id, @event.Id, userId, userEmail), ct);
```

### Consumidor (EventHub.Notifier)

```csharp
// Notifier/Consumers/ReservationConfirmedConsumer.cs
public class ReservationConfirmedConsumer : IConsumer<ReservationConfirmed>
{
    private readonly ILogger<ReservationConfirmedConsumer> _logger;

    public async Task Consume(ConsumeContext<ReservationConfirmed> context)
    {
        var msg = context.Message;
        _logger.LogInformation(
            "[EMAIL] Enviando confirmação para {Email} — Reserva {ReservationId}",
            msg.Email, msg.ReservationId);

        // Simula envio de email (200ms)
        await Task.Delay(200);
        _logger.LogInformation("[EMAIL] Confirmação enviada para {Email}", msg.Email);

        // Simula envio de SMS
        _logger.LogInformation(
            "[SMS] Enviando SMS de confirmação para reserva {ReservationId}",
            msg.ReservationId);
        await Task.Delay(100);
        _logger.LogInformation("[SMS] SMS enviado");
    }
}
```

### Configuração MassTransit

```csharp
// Program.cs (serviço principal)
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
    });
});

// Program.cs (serviço de notificações)
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<ReservationConfirmedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        cfg.ReceiveEndpoint("notificacoes-email", e =>
        {
            e.ConfigureConsumer<ReservationConfirmedConsumer>(context);
        });
    });
});
```

---

## Docker Compose com RabbitMQ

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI (http://localhost:15672)
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    # ... depende de rabbitmq com condition: service_healthy

  notifier:
    # ... depende de rabbitmq com condition: service_healthy
```

A imagem `rabbitmq:3-management-alpine` inclui o plugin de UI web. Acesse `http://localhost:15672` (guest/guest) para ver exchanges, filas, mensagens e consumidores em tempo real.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar a diferença entre RabbitMQ (filas, broker push) e Kafka (log, consumer pull)
- [ ] Configurar MassTransit com RabbitMQ em dois serviços
- [ ] Publicar um `IIntegrationEvent` e consumir com `IConsumer<T>`
- [ ] Acessar o Management UI do RabbitMQ e ver filas, mensagens e consumidores
- [ ] Entender o que acontece com a mensagem quando o consumidor cai (fica na fila, esperando)
- [ ] Saber a diferença entre Ack, Nack e DLX
