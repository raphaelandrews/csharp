# 03 — CI/CD com GitHub Actions

## Pipeline de CI: build + test a cada push

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: eventhub_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Test
        run: dotnet test --no-build -c Release
        env:
          ConnectionStrings__Default: "Host=localhost;Database=eventhub_test;Username=postgres;Password=postgres"

      - name: Build Docker image
        run: docker build -t eventhub-api:latest .
```

### Services no GitHub Actions

O bloco `services` cria containers auxiliares durante o job. O PostgreSQL é iniciado automaticamente, tem portas mapeadas, e é destruído ao final. Nada de Docker Compose aqui — o GitHub Actions gerencia o ciclo de vida.

### Matrix build: testando em múltiplas versões

```yaml
strategy:
  matrix:
    dotnet: ['8.0.x', '9.0.x']
steps:
  - uses: actions/setup-dotnet@v4
    with:
      dotnet-version: ${{ matrix.dotnet }}
```

Isso roda o pipeline em paralelo para cada versão do .NET. Útil para bibliotecas que precisam garantir compatibilidade — menos relevante para APIs.

---

## Pipeline de CD: deploy automático

```yaml
# .github/workflows/cd.yml
name: CD

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        run: |
          docker build -t ghcr.io/${{ github.repository }}/api:latest .
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}/api:latest

      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: eventhub-api
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          images: ghcr.io/${{ github.repository }}/api:latest
```

### GitHub Container Registry (ghcr.io)

Cada repositório GitHub tem um registry de containers gratuito (para contas públicas). A imagem é empurrada para `ghcr.io/raphaelandrews/eventhub/api:latest` e pode ser puxada por qualquer serviço que suporte OCI containers.

---

## O que colocar no pipeline (checklist mínimo)

| Etapa | O que faz | Por que |
|---|---|---|
| `dotnet restore` | Baixa pacotes NuGet | Cacheável, separado do build |
| `dotnet build` | Compila | Falha rápido se houver erro de compilação |
| `dotnet test` | Roda testes | Unitários + integração |
| `dotnet format --verify` | Verifica formatação | StyleCop/.editorconfig |
| `docker build` | Constrói imagem | Garante que o Dockerfile funciona |
| `docker push` | Publica imagem | Disponibiliza para deploy |

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Criar um workflow que roda em `push` e `pull_request`
- [ ] Configurar um service container (PostgreSQL) para testes de integração
- [ ] Publicar uma imagem Docker no GitHub Container Registry
- [ ] Usar `secrets.GITHUB_TOKEN` para autenticação no registry
