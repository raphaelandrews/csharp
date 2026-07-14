# 04 — Azure: fundamentos para dev .NET

## Por que Azure (e não AWS/GCP)

O mercado .NET tem uma afinidade histórica com Azure. Isso não significa que você precise ser certificado AZ-900 — mas saber o básico te diferencia em entrevistas onde a empresa usa stack Microsoft.

Os serviços que mais aparecem em vagas .NET:

| Serviço | Equivalente AWS | Para que serve |
|---|---|---|
| **App Service** | Elastic Beanstalk | Hospedar API sem gerenciar VMs |
| **Azure SQL** | RDS | SQL Server gerenciado como serviço |
| **Key Vault** | Secrets Manager | Armazenar connection strings, chaves |
| **Application Insights** | CloudWatch | Logs, métricas, tracing distribuído |
| **Azure Container Registry** | ECR | Registry privado de imagens Docker |
| **Azure DevOps** | CodeBuild+CodePipeline | CI/CD integrado (alternativa ao GitHub Actions) |

---

## App Service: deploy simplificado

App Service é um PaaS — você sobe o código e a Microsoft gerencia servidor, SO, runtime, scaling e certificados SSL.

```bash
# Criar um App Service
az webapp up \
  --name eventhub-api \
  --runtime "DOTNET:9.0" \
  --os-type Linux \
  --sku F1  # Free tier (não paga nada, mas tem limitações)
```

O App Service suporta deploy via:
- **Zip Deploy**: `az webapp deploy --src-path ./publish.zip`
- **Docker**: aponta para `ghcr.io/usuario/eventhub/api:latest`
- **GitHub Actions**: integração nativa com o workflow de CI

### Configuration via portal/CLI

As variáveis de ambiente no App Service são gerenciadas como "Application Settings":

```bash
az webapp config appsettings set \
  --name eventhub-api \
  --settings \
    ConnectionStrings__Default="Server=tcp:myserver.database.windows.net..." \
    ASPNETCORE_ENVIRONMENT="Production"
```

---

## Azure SQL: SQL Server como serviço

```bash
# Criar servidor
az sql server create \
  --name eventhub-db-server \
  --admin-user sqladmin \
  --admin-password "SuperSecret123!" \
  --location brazilsouth

# Criar banco
az sql db create \
  --server eventhub-db-server \
  --name eventhub \
  --service-objective Basic  # $5/mês, para dev
```

Connection string típica:
```
Server=tcp:eventhub-db-server.database.windows.net,1433;
Database=eventhub;
User Id=sqladmin;
Password=SuperSecret123!;
TrustServerCertificate=false;
Encrypt=true;
```

**Atenção**: por padrão, o Azure SQL bloqueia todos os IPs. É preciso configurar firewall rules:
```bash
az sql server firewall-rule create \
  --server eventhub-db-server \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0  # permite acesso de outros serviços Azure
```

---

## Key Vault: segredos não vão no código

```bash
# Criar vault
az keyvault create --name eventhub-vault --resource-group eventhub-rg

# Armazenar segredo
az keyvault secret set \
  --vault-name eventhub-vault \
  --name "ConnectionString" \
  --value "Server=...;Database=...;Password=..."

# A aplicação lê via Managed Identity (sem senha no código):
// Program.cs
builder.Configuration.AddAzureKeyVault(
    new Uri("https://eventhub-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

**Managed Identity**: a aplicação no App Service tem uma identidade gerenciada pelo Azure. Ela pode acessar Key Vault, SQL Database e outros serviços Azure sem nenhuma senha — a autenticação é feita via Azure AD automaticamente. Esse é o "jeito Azure" de fazer segurança e é um diferencial real vs AWS IAM.

---

## Custos realistas (para portfólio)

| Serviço | Tier gratuito | Custo mínimo |
|---|---|---|
| App Service | F1 (60 min CPU/dia) | $0 — suficiente para demo |
| Azure SQL | Não tem | $5/mês (Basic, 2GB) |
| Key Vault | Não tem | $0.03/10k operações — irrisório |
| Container Registry | Não tem | $5/mês (Basic) |

Para portfólio: App Service Free + PostgreSQL local no Docker (em vez de Azure SQL pago). Ou usar SQLite.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar a diferença entre App Service (PaaS) e VM (IaaS)
- [ ] Criar um Key Vault e ler segredos via `AddAzureKeyVault()` + Managed Identity
- [ ] Saber que Azure SQL bloqueia IPs por padrão (precisa de firewall rule)
- [ ] Diferenciar Application Insights (APM) de Log Analytics (logs centralizados)
- [ ] Argumentar quando usar Azure vs manter on-premise/Docker simples
