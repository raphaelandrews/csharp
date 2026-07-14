# 06 — Secrets Management: variáveis de ambiente e segurança

## O problema: secrets no código-fonte

```json
// NUNCA comite isso:
{
  "ConnectionStrings": {
    "Default": "Host=prod-db;Password=SuperSecret123"
  },
  "Jwt": {
    "Secret": "my-production-jwt-secret-key-12345678"
  }
}
```

Um único push para um repositório público e suas credenciais de produção estão expostas. Mesmo em repositórios privados, secrets no código são um risco de vazamento, rotatividade difícil, e violação de compliance.

---

## Hierarquia de configuração no ASP.NET Core

O ASP.NET Core carrega configuração de múltiplas fontes, em ordem de precedência (a última sobrescreve):

```csharp
// Program.cs — ordem padrão (implícita no CreateDefaultBuilder)
1. appsettings.json
2. appsettings.{Environment}.json
3. User Secrets (apenas Development)
4. Variáveis de ambiente
5. Argumentos de linha de comando
```

Isso significa que você pode ter um `appsettings.json` com valores padrão para desenvolvimento e sobrescrever em produção via variáveis de ambiente.

```json
// appsettings.json — seguro para commitar
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=eventhub;Username=postgres;Password=postgres"
  }
}
```

```bash
# Produção — variável de ambiente sobrescreve
export ConnectionStrings__Default="Host=prod-db.internal;Password=RealPassword"
```

---

## User Secrets: desenvolvimento local seguro

```bash
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Password=devpass"
dotnet user-secrets set "Jwt:Secret" "dev-secret-with-at-least-32-chars!!"
```

Os segredos são armazenados em `~/.microsoft/usersecrets/<guid>/secrets.json` — fora do repositório, sem risco de commit acidental.

No `Program.cs`, o User Secrets é carregado automaticamente em Development:

```csharp
// Isso já acontece por padrão com WebApplication.CreateBuilder()
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```

---

## Padrão de configuração tipada (IOptions<T>)

Em vez de espalhar `Configuration["Jwt:Secret"]` pelo código:

```csharp
// Definição
public class JwtSettings
{
    public const string SectionName = "Jwt";
    public string Secret { get; init; } = null!;
    public string Issuer { get; init; } = "eventhub";
    public string Audience { get; init; } = "eventhub-client";
    public int ExpirationMinutes { get; init; } = 1440;
}

// Registro
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection(JwtSettings.SectionName));

// Uso
public class AuthService
{
    public AuthService(IOptions<JwtSettings> jwtSettings)
    {
        var settings = jwtSettings.Value;
    }
}
```

**Vantagens**:
- Validação no startup: `services.AddOptions<JwtSettings>().ValidateDataAnnotations().ValidateOnStart()`
- Recarga automática: `IOptionsSnapshot<T>` recarrega a cada requisição, `IOptionsMonitor<T>` notifica mudanças
- Testabilidade: `Options.Create(new JwtSettings { Secret = "test" })`

---

## Azure Key Vault integrado

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri("https://eventhub-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

`DefaultAzureCredential` tenta autenticar na seguinte ordem:
1. `AZURE_CLIENT_ID` + `AZURE_CLIENT_SECRET` (service principal)
2. Managed Identity (automático no App Service)
3. Azure CLI (`az login`)
4. Visual Studio Code credential

Isso significa que o mesmo código funciona localmente (`az login`) e em produção (Managed Identity).

---

## Segredos no Docker/K8s

### Docker Compose — via arquivo .env

```yaml
# docker-compose.yml
environment:
  ConnectionStrings__Default: ${DB_CONNECTION_STRING}
```

```bash
# .env (NÃO commitar)
DB_CONNECTION_STRING=Host=db;Password=secret
```

### Kubernetes — Secrets

```yaml
env:
  - name: ConnectionStrings__Default
    valueFrom:
      secretKeyRef:
        name: eventhub-secrets
        key: connection-string
```

O Secret do K8s é apenas base64 (não criptografado). Em produção, combine com:
- **Sealed Secrets**: criptografa o Secret com chave pública, só o cluster consegue decriptar
- **External Secrets Operator**: sincroniza secrets do Azure Key Vault/AWS Secrets Manager para o K8s

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Usar `dotnet user-secrets` para desenvolvimento local
- [ ] Sobrescrever `appsettings.json` com variáveis de ambiente em produção
- [ ] Implementar `IOptions<T>` com validação no startup
- [ ] Explicar a hierarquia de configuração: JSON → User Secrets → Env Vars → CLI args
- [ ] Diferenciar `IOptions<T>` (singleton), `IOptionsSnapshot<T>` (scoped) e `IOptionsMonitor<T>` (reativo)
