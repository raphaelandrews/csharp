# 06 — Idempotência e Consistência Eventual

## O problema: mensagens duplicadas

Sistemas de mensageria garantem **at-least-once delivery** por padrão. Isso significa que uma mensagem pode ser entregue mais de uma vez:

- O consumidor processa a mensagem mas morre antes de enviar o Ack
- O broker reenvia a mensagem para outro consumidor
- Network partition faz o broker achar que a mensagem não foi entregue

Se o consumidor "enviar email de confirmação" duas vezes, o usuário recebe dois emails. Se "confirmar pagamento" duas vezes, o cliente é cobrado em dobro.

**Idempotência** resolve isso: processar a mesma mensagem N vezes tem o mesmo efeito que processar 1 vez.

---

## Estratégias de idempotência

### 1. Idempotency Key (a mais comum)

O publisher inclui um identificador único na mensagem. O consumidor verifica se já processou essa key:

```csharp
public class ReservationConfirmedConsumer : IConsumer<ReservationConfirmed>
{
    private readonly AppDbContext _db;

    public async Task Consume(ConsumeContext<ReservationConfirmed> context)
    {
        var msg = context.Message;

        // Verifica se já processou essa mensagem
        var alreadyProcessed = await _db.ProcessedMessages
            .AnyAsync(m => m.MessageId == msg.ReservationId);

        if (alreadyProcessed)
        {
            _logger.LogWarning("Mensagem duplicada ignorada: {Id}", msg.ReservationId);
            return; // Idempotente: não faz nada
        }

        // Processa
        await SendConfirmationEmail(msg);

        // Registra que processou
        _db.ProcessedMessages.Add(new ProcessedMessage
        {
            MessageId = msg.ReservationId,
            ProcessedAt = DateTime.UtcNow
        });
        await _db.SaveChangesAsync();
    }
}
```

A tabela `ProcessedMessages` funciona como um "deduplication store". O check + insert precisam ser **atômicos** (mesma transação ou constraint unique).

### 2. Unique Constraint no banco (mais simples)

```csharp
// Não precisa de tabela separada se a operação já tem unique constraint
modelBuilder.Entity<Notification>(entity =>
{
    entity.HasIndex(n => n.ReservationId).IsUnique();
});

// No consumidor:
try
{
    _db.Notifications.Add(new Notification
    {
        ReservationId = msg.ReservationId,
        SentAt = DateTime.UtcNow
    });
    await _db.SaveChangesAsync();
}
catch (DbUpdateException) when (IsDuplicateKeyException(ex))
{
    // Já foi processada — idempotente
    _logger.LogWarning("Notificação duplicada ignorada: {Id}", msg.ReservationId);
}
```

A constraint unique no banco garante atomicidade sem código extra. Se dois consumidores tentarem inserir a mesma `ReservationId`, um ganha e o outro recebe exceção de duplicate key.

### 3. Operações naturalmente idempotentes

Algumas operações são idempotentes por natureza:

```csharp
// Idempotente: SET é idempotente (executar 2x = mesmo resultado)
await _cache.SetStringAsync($"seats:{sessionId}", "42");

// Idempotente: upsert
await _db.Sessions
    .Where(s => s.Id == sessionId)
    .ExecuteUpdateAsync(s => s.SetProperty(x => x.AvailableSeats, 42));

// NÃO idempotente: increment/decrement
session.AvailableSeats--;  // executar 2x = -2, ERRADO
session.ReservedSeats++;   // executar 2x = +2, ERRADO
```

Sempre que possível, modele operações como SET (valor absoluto) em vez de INCREMENT (delta). SET é naturalmente idempotente.

---

## Consistência Eventual

Quando o serviço de reservas publica `ReservaConfirmada` e o serviço de notificações consome, há uma janela de inconsistência:

```
T0: Reserva criada no banco (serviço de reservas)
T1: Evento ReservaConfirmada publicado
T2: (janela: notificação ainda não foi enviada)
T3: Consumidor processa, envia email
T4: Usuário recebe email
```

Isso é **consistência eventual**: o sistema converge para o estado correto, mas há uma janela onde os estados dos serviços estão desincronizados.

### Como lidar

1. **Aceitar**: para notificações, uma janela de 2 segundos é aceitável. O email de confirmação pode chegar 2 segundos depois da reserva.

2. **Compensação**: se o pagamento falhar depois que a reserva foi confirmada, publicar um evento `PagamentoRecusado` que reverte a reserva.

3. **Saga**: para fluxos longos (reservar → pagar → emitir ingresso), use o padrão Saga com MassTransit, onde cada passo tem uma compensação definida:

```csharp
// Saga: orquestra o fluxo de compra
public class PurchaseSaga : MassTransitStateMachine<PurchaseState>
{
    public State ReservationCreated { get; set; }
    public State PaymentProcessed { get; set; }
    public State Completed { get; set; }

    public Event<ReservaConfirmada> ReservaConfirmada { get; set; }
    public Event<PagamentoAprovado> PagamentoAprovado { get; set; }
    public Event<PagamentoRecusado> PagamentoRecusado { get; set; }

    public PurchaseSaga()
    {
        InstanceState(x => x.CurrentState);

        Event(() => ReservaConfirmada);
        Event(() => PagamentoAprovado);
        Event(() => PagamentoRecusado);

        Initially(
            When(ReservaConfirmada)
                .Then(ctx => ctx.Saga.ReservationId = ctx.Message.ReservationId)
                .TransitionTo(ReservationCreated)
                .Publish(ctx => new ProcessarPagamento(ctx.Saga.ReservationId))
        );

        During(ReservationCreated,
            When(PagamentoAprovado)
                .TransitionTo(Completed)
                .Publish(ctx => new EmitirIngresso(ctx.Saga.ReservationId)),
            When(PagamentoRecusado)
                .TransitionTo(Completed)
                .Publish(ctx => new CancelarReserva(ctx.Saga.ReservationId)) // compensação
        );
    }
}
```

A Saga garante que, independentemente de falhas, o sistema sempre converge para um estado consistente (reserva confirmada + paga ou reserva cancelada).

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Implementar idempotência via idempotency key + tabela de mensagens processadas
- [ ] Usar unique constraint como mecanismo de deduplication
- [ ] Explicar a diferença entre operação idempotente (SET, upsert) e não-idempotente (INCREMENT, decrement)
- [ ] Desenhar o fluxo de consistência eventual entre Reservas → Notificações
- [ ] Explicar o padrão Saga: cada passo tem uma ação de compensação
- [ ] Saber que at-least-once delivery é o padrão da maioria dos brokers
