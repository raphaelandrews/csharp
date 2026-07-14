# 05 — gRPC: alternativa ao REST para comunicação interna

## O problema: REST entre serviços

Quando dois serviços se comunicam via REST:

```csharp
// Serviço de Reservas → Serviço de Pagamentos
var response = await httpClient.PostAsJsonAsync(
    "http://pagamentos-api/api/payments",
    new { amount = 150.00m, method = "credit_card" });

var payment = await response.Content.ReadFromJsonAsync<PaymentDto>();
```

Problemas:
1. **Serialização JSON**: texto verboso para dados estruturados (decimal → string → decimal, perda de precisão)
2. **Sem contrato**: se o endpoint mudar (`/api/payments` → `/api/v2/payments`), descobre em runtime
3. **HTTP/1.1**: uma requisição por conexão TCP (ou limited multiplexing com HTTP/2, se configurado)
4. **Latência**: JSON parse + HTTP overhead

gRPC resolve esses problemas para comunicação **entre serviços internos**. Não é substituto para REST em APIs públicas — é para service-to-service.

---

## Por que gRPC

| Critério | REST/JSON | gRPC |
|---|---|---|
| Serialização | JSON (texto) | Protobuf (binário) |
| Contrato | Swagger/OpenAPI (opcional) | `.proto` (obrigatório, compilado) |
| Tipagem | Fraca (`decimal` vira `number`) | Forte (tipos Protobuf → tipos C#) |
| Streaming | Não nativo (WebSocket separado) | Nativo (unário, server, client, bidirectional) |
| Performance | ~2-5ms por chamada | ~0.1-0.5ms por chamada |
| Navegador | Nativo (fetch) | Precisa de gRPC-Web |
| Ecossistema | Universal | Crescendo |

**Regra prática**: REST para APIs públicas (clientes web/mobile), gRPC para comunicação entre seus próprios serviços.

---

## Contrato .proto (o coração do gRPC)

```protobuf
// Protos/payment.proto
syntax = "proto3";

package eventhub.payments;

service PaymentService {
    rpc ProcessPayment (PaymentRequest) returns (PaymentResponse);
    rpc GetPaymentStatus (PaymentStatusRequest) returns (PaymentStatusResponse);
}

message PaymentRequest {
    string reservation_id = 1;
    string payment_method = 2;  // "credit_card" | "pix"
    Money amount = 3;
}

message Money {
    int64 amount_cents = 1;      // 15000 = R$ 150,00
    string currency = 2;         // "BRL"
}

message PaymentResponse {
    string payment_id = 1;
    string status = 2;           // "confirmed" | "refused"
}
```

Os números (`= 1`, `= 2`) são tags de campo no formato binário Protobuf — não são posições, são identificadores que permitem evolução do schema sem quebrar compatibilidade.

---

## Gerando código C# a partir do .proto

```xml
<!-- EventHub.Payments.csproj -->
<ItemGroup>
    <Protobuf Include="Protos\payment.proto" GrpcServices="Server" />
</ItemGroup>
```

O build gera automaticamente:
- `PaymentServiceBase` — classe base para implementar no servidor
- `PaymentServiceClient` — cliente tipado para chamar do outro serviço

### Servidor gRPC

```csharp
// Payments/Services/PaymentGrpcService.cs
public class PaymentGrpcService : PaymentService.PaymentServiceBase
{
    public override async Task<PaymentResponse> ProcessPayment(
        PaymentRequest request, ServerCallContext context)
    {
        // Lógica de pagamento
        return new PaymentResponse
        {
            PaymentId = Guid.NewGuid().ToString(),
            Status = "confirmed"
        };
    }
}

// Program.cs
builder.Services.AddGrpc();
app.MapGrpcService<PaymentGrpcService>();
```

### Cliente gRPC

```csharp
// Program.cs (serviço de reservas)
builder.Services.AddGrpcClient<PaymentService.PaymentServiceClient>(options =>
{
    options.Address = new Uri("https://pagamentos-api:5001");
});

// Uso no handler
public class CreateReservationHandler
{
    private readonly PaymentService.PaymentServiceClient _paymentClient;

    public async Task Handle(CreateReservationCommand cmd, CancellationToken ct)
    {
        var paymentResult = await _paymentClient.ProcessPaymentAsync(
            new PaymentRequest
            {
                ReservationId = reservation.Id.ToString(),
                PaymentMethod = "credit_card",
                Amount = new Money { AmountCents = 15000, Currency = "BRL" }
            });
    }
}
```

---

## Streaming (o diferencial do gRPC)

REST é request/response apenas. gRPC tem 4 modos:

| Modo | Exemplo |
|---|---|
| **Unário** | 1 request → 1 response (como REST) |
| **Server streaming** | 1 request → N responses | Relatório de vendas em tempo real |
| **Client streaming** | N requests → 1 response | Upload de arquivo em chunks |
| **Bidirectional** | N requests → N responses | Chat em tempo real |

```protobuf
// Server streaming
rpc StreamSalesReport (SalesReportRequest) returns (stream SalesReportRow);
```

---

## Quando usar gRPC vs REST vs mensageria

| Comunicação | Usar quando | Exemplo no eventhub |
|---|---|---|
| **REST** | API pública, clientes web/mobile | Listagem de eventos, criação de conta |
| **gRPC** | Service-to-service síncrono, baixa latência | Reservas → Pagamentos (precisa de resposta imediata) |
| **Mensageria** | Service-to-service assíncrono, desacoplado | Reserva confirmada → Notificação (pode esperar) |

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Escrever um arquivo `.proto` com service, rpc e messages
- [ ] Gerar código C# via `<Protobuf Include="..." GrpcServices="Server" />`
- [ ] Implementar um serviço gRPC no ASP.NET Core (`AddGrpc()`, `MapGrpcService()`)
- [ ] Configurar um cliente gRPC via `AddGrpcClient<T>()` com endereço fixo
- [ ] Explicar quando usar gRPC (service-to-service) vs REST (API pública) vs mensageria (assíncrono)
- [ ] Diferenciar os 4 modos de streaming do gRPC
