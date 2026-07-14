# 01 — Docker + .NET: containerizando uma API

## Por que Docker no ecossistema .NET

Historicamente, .NET era sinônimo de Windows Server + IIS. Docker mudou isso — a partir do .NET Core, aplicações .NET rodam em containers Linux tão bem quanto qualquer aplicação Node.js ou Go.

A Microsoft mantém imagens oficiais otimizadas no Docker Hub:
- `mcr.microsoft.com/dotnet/sdk:9.0` — SDK completo (build, restore, publish)
- `mcr.microsoft.com/dotnet/aspnet:9.0` — runtime mínimo (só para executar)
- `mcr.microsoft.com/dotnet/runtime:9.0` — runtime sem ASP.NET (workers, console apps)

A imagem de runtime tem ~120MB, comparável a imagens Go compiladas. O SDK é maior (~800MB) mas só é usado no build.

---

## Multi-stage build: o padrão .NET

O Dockerfile idiomático para .NET usa multi-stage build para manter a imagem final enxuta:

```dockerfile
# Dockerfile — eventhub
# Estágio 1: Build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copia apenas os .csproj primeiro (cache de camadas)
COPY src/EventHub.Domain/EventHub.Domain.csproj src/EventHub.Domain/
COPY src/EventHub.Application/EventHub.Application.csproj src/EventHub.Application/
COPY src/EventHub.Infrastructure/EventHub.Infrastructure.csproj src/EventHub.Infrastructure/
COPY src/EventHub.Api/EventHub.Api.csproj src/EventHub.Api/

# Restaura dependências (camada cacheada se .csproj não mudar)
RUN dotnet restore EventHub.Api/EventHub.Api.csproj

# Copia o resto e compila
COPY . .
RUN dotnet publish EventHub.Api/EventHub.Api.csproj -c Release -o /app/publish --no-restore

# Estágio 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

# Cria usuário não-root (segurança)
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --gid 1001 appuser

COPY --from=build /app/publish .
RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "EventHub.Api.dll"]
```

### Por que copiar .csproj primeiro

O Docker cacheia cada instrução `COPY`. Se você copia o código-fonte inteiro de uma vez e depois roda `dotnet restore`, qualquer mudança em qualquer arquivo invalida o cache e força re-download de todos os pacotes NuGet.

Copiando apenas os `.csproj` primeiro e rodando `dotnet restore`, essa camada só é reconstruída quando um `.csproj` muda (ou seja, quando você adiciona/remove pacotes). Mudanças em arquivos `.cs` não invalidam o cache do restore — economia de 20-30 segundos por build.

### Por que usuário não-root

Por padrão, containers rodam como root. Se um invasor explorar uma vulnerabilidade na aplicação, ele tem acesso root ao container. Criar um usuário específico (`appuser`) limita o dano potencial — é uma prática de segurança básica recomendada pela Microsoft e por qualquer scanner de vulnerabilidades.

---

## docker-compose: API + banco de dados

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: eventhub
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      ConnectionStrings__Default: "Host=postgres;Database=eventhub;Username=postgres;Password=postgres"
      ASPNETCORE_ENVIRONMENT: Development
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  pgdata:
```

**Variáveis de ambiente com `__`**: o ASP.NET Core converte `ConnectionStrings__Default` para `ConnectionStrings:Default` (separador `:` vira `__` em variáveis de ambiente, porque `:` não é válido em nomes de variáveis em alguns shells).

**`depends_on` com `condition: service_healthy`**: espera o PostgreSQL estar realmente pronto (não só o container iniciado). Sem o healthcheck, o `depends_on` só espera o processo iniciar — o que não garante que o banco está aceitando conexões.

---

## .dockerignore: o que NÃO vai para o container

```dockerignore
**/bin/
**/obj/
**/.git/
**/.vs/
**/node_modules/
*.user
*.DotSettings
.vscode/
.idea/
*.sln.DotSettings
```

Sem `.dockerignore`, o `COPY . .` envia `bin/` e `obj/` (que podem ter gigabytes de artefatos de build) para o contexto do Docker. Isso torna o build mais lento e pode causar conflitos com os assemblies gerados no estágio de build.

---

## Tamanho da imagem e otimizações

### Imagem mínima: chiseled containers

A partir do .NET 8, a Microsoft oferece imagens "chiseled" — Ubuntu minimizado onde tudo que não é necessário para rodar a aplicação foi removido. A imagem `aspnet:9.0-noble-chiseled` tem ~50MB (vs ~120MB da imagem normal):

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0-noble-chiseled AS runtime
```

O trade-off: sem shell (`/bin/bash`), sem `apt`, sem `curl`. Você não consegue `docker exec` para debug. Use em produção, não em desenvolvimento.

### ReadyToRun (R2R)

```dockerfile
RUN dotnet publish EventHub.Api/EventHub.Api.csproj -c Release \
    -o /app/publish \
    --no-restore \
    -p:PublishReadyToRun=true
```

Compila antecipadamente os assemblies (AOT parcial). Reduz o tempo de startup em ~20-30% em troca de um binário ligeiramente maior. Útil para cenários serverless ou onde cold start importa.

---

## Docker + EF Core: migrations no container

Em produção, as migrations podem rodar como parte do startup:

```csharp
// Program.cs — aplica migrations no startup
using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await db.Database.MigrateAsync();
```

Isso garante que o container da API sempre encontra o banco no schema correto. Mas tem trade-offs:
- **Vantagem**: deploy simples (só subir o container)
- **Desvantagem**: escalar para 3 réplicas causa 3 tentativas simultâneas de aplicar a mesma migration (o EF Core lida com isso via lock, mas é um ponto de contenção na inicialização)

Para ambientes com múltiplas réplicas, a prática recomendada é rodar migrations como um **init container** ou **job** separado no CI/CD.

---

## Executando

```bash
# Build e run com docker-compose
docker compose up --build

# Testar
curl http://localhost:8080/health

# Parar
docker compose down

# Remover volumes (limpa banco)
docker compose down -v
```

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Escrever um Dockerfile multi-stage com SDK → runtime para um projeto .NET
- [ ] Explicar por que copiar .csproj primeiro otimiza o cache de camadas
- [ ] Configurar docker-compose com healthcheck no PostgreSQL e depends_on
- [ ] Diferenciar `aspnet:9.0` (completa) de `aspnet:9.0-noble-chiseled` (mínima, sem shell)
- [ ] Executar `docker compose up --build` e acessar `/health`
- [ ] Explicar o trade-off de rodar `MigrateAsync()` no startup do container
