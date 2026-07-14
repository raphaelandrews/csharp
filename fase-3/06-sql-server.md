# 06 — SQL Server no ecossistema .NET

## Por que este tópico existe

Se você abrir 10 vagas de C# .NET no Brasil, 7 vão mencionar SQL Server. Não é uma questão técnica — é uma questão histórica e de mercado. O SQL Server é o banco de dados "padrão de fábrica" do ecossistema Microsoft desde os anos 90, e grande parte do mercado corporativo brasileiro roda sistemas legados ou atuais em cima dele.

Você não precisa virar DBA de SQL Server. Mas precisa saber o suficiente para:
1. Entender por que ele aparece em tantas vagas
2. Reconhecer as diferenças práticas em relação ao PostgreSQL (que você usa no dia a dia)
3. Saber subir uma instância localmente (via Docker, já que não existe instalação nativa no Linux)
4. Configurar o EF Core com o provider do SQL Server
5. Conhecer os recursos exclusivos que aparecem em projetos corporativos

---

## O que é SQL Server e por que ele domina o mercado .NET

### Contexto histórico

SQL Server nasceu em 1989 como uma parceria Microsoft + Sybase. Em 1994, a Microsoft tomou controle total do código e desde então o SQL Server evoluiu como **o** banco relacional da stack Microsoft.

A integração sempre foi o diferencial: Visual Studio + SQL Server Management Studio (SSMS) + .NET Framework formavam um ecossistema integrado onde tudo "conversava" nativamente. Enquanto o PostgreSQL era uma opção "de nicho" no mundo Windows, o SQL Server vinha pré-instalado em servidores Windows, com suporte de primeiro nível da Microsoft.

Esse legado de 30+ anos significa que:
- Empresas que adotaram Windows Server + SQL Server nos anos 2000 ainda mantêm esses sistemas
- O ecossistema de ferramentas (SSMS, SSIS, SSRS, SQL Profiler) é maduro e familiar para equipes .NET
- A documentação, comunidade e suporte corporativo são vastos — você sempre acha alguém que já resolveu seu problema
- Muitas certificações Microsoft (MCSE, MCSA) exigiam conhecimento de SQL Server, criando uma base instalada de profissionais certificados

### A mudança com .NET Core/Linux

A partir de 2017, a Microsoft começou a oferecer SQL Server no Linux (via `mssql-server` package). Mas a instalação nativa só funciona em Ubuntu, RHEL e SUSE — **não tem pacote para Arch Linux**. A forma prática de rodar em qualquer Linux é via Docker:

```bash
docker run -e 'ACCEPT_EULA=Y' \
  -e 'MSSQL_SA_PASSWORD=Senha@123' \
  -p 1433:1433 \
  -d mcr.microsoft.com/mssql/server:2022-latest
```

Isso é relevante porque mostra uma mudança de filosofia: a Microsoft entendeu que o futuro é multi-plataforma. Mas o SQL Server ainda carrega o legado de ser "o banco do Windows".

### SQL Server ≠ T-SQL ≠ Windows

Um ponto importante de esclarecer: SQL Server é o **produto** (o SGBD). T-SQL (Transact-SQL) é o **dialeto SQL** que ele usa. E Windows é o **sistema operacional** onde historicamente ele rodava. Hoje, SQL Server roda em Linux e containers, e você pode se conectar a ele de qualquer plataforma via `Microsoft.Data.SqlClient`.

---

## Diferenças práticas SQL Server vs PostgreSQL

Como você já conhece PostgreSQL, o jeito mais eficiente de entender SQL Server é por contraste:

### Tipos de dados

| Conceito | PostgreSQL | SQL Server |
|---|---|---|
| UUID | `UUID` (nativo) | `UNIQUEIDENTIFIER` |
| Booleano | `BOOLEAN` (true/false) | `BIT` (1/0) |
| Auto-incremento | `SERIAL` / `BIGSERIAL` | `IDENTITY(1,1)` |
| Texto ilimitado | `TEXT` | `NVARCHAR(MAX)` |
| Data/hora com timezone | `TIMESTAMPTZ` | `DATETIMEOFFSET` |
| Enum | `CREATE TYPE ... AS ENUM` | Não tem enum nativo (usa lookup table ou `VARCHAR` + constraint) |
| Array | `INTEGER[]`, `TEXT[]` | Não tem array (normaliza em tabela separada) |
| JSON | `JSONB` (indexável, binário) | `NVARCHAR(MAX)` com `ISJSON()` + `JSON_VALUE()` — não é indexável como tipo nativo |
| UUID | `UUID` | `UNIQUEIDENTIFIER` |
| GUID padrão | Precisa de extensão (`uuid-ossp`) | `NEWID()` (aleatório) ou `NEWSEQUENTIALID()` (sequencial) |

**Impacto no EF Core**: ao trocar o provider de `Npgsql` para `SqlServer`, o EF Core ajusta automaticamente o mapeamento de tipos. `Guid` vira `UNIQUEIDENTIFIER`, `bool` vira `BIT`, `DateTime` vira `DATETIME2`. O único cuidado extra é com `DateTimeOffset` (PostgreSQL usa `TIMESTAMPTZ`, SQL Server usa `DATETIMEOFFSET`) — mas o EF Core resolve isso no provider.

### Sintaxe SQL (T-SQL vs PL/pgSQL)

```sql
-- ─── Limite de linhas ───
-- PostgreSQL
SELECT * FROM Habits LIMIT 10;

-- SQL Server (T-SQL)
SELECT TOP 10 * FROM Habits;
-- Ou (SQL Server 2012+)
SELECT * FROM Habits OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;

-- ─── GUID ───
-- PostgreSQL
SELECT uuid_generate_v4();  -- precisa de extensão
INSERT INTO Users (Id) VALUES (gen_random_uuid());

-- SQL Server
SELECT NEWID();
INSERT INTO Users (Id) VALUES (NEWID());

-- ─── Concatenação de string ───
-- PostgreSQL
SELECT 'Hello' || ' ' || 'World';

-- SQL Server (T-SQL)
SELECT 'Hello' + ' ' + 'World';
-- Ou
SELECT CONCAT('Hello', ' ', 'World');

-- ─── Bool ───
-- PostgreSQL
SELECT * FROM Habits WHERE IsActive = true;

-- SQL Server
SELECT * FROM Habits WHERE IsActive = 1;

-- ─── ILIKE (case-insensitive) ───
-- PostgreSQL: ILIKE é nativo
SELECT * FROM Habits WHERE Name ILIKE '%exercício%';

-- SQL Server: collation define case sensitivity
SELECT * FROM Habits WHERE Name LIKE '%exercício%';
-- (case-insensitive é o padrão na maioria das collations)
```

### Comportamento transacional

| Aspecto | PostgreSQL | SQL Server |
|---|---|---|
| Autocommit | `ON` por padrão no psql | `ON` por padrão |
| Isolation level padrão | `READ COMMITTED` | `READ COMMITTED` |
| MVCC | Sim (visibilidade por tupla) | Sim (desde 2005, via tempdb + version store) |
| `SERIALIZABLE` | Previne phantoms, serialização real | Previne phantoms, mas implementação diferente |
| Deadlock detection | Timeout + abort | Detecção automática, escolhe uma vítima |

**Na prática**: o comportamento é bem similar. A principal diferença que afeta seu código é que o SQL Server por padrão **não deixa ler dados não commitados em transações concorrentes dentro do mesmo isolation level** — algo que o PostgreSQL também faz no `READ COMMITTED`. A divergência maior aparece em cenários de alta concorrência com `SERIALIZABLE`.

### Schemas

PostgreSQL e SQL Server organizam tabelas em schemas, mas a semântica é diferente:

```
-- PostgreSQL: schema = namespace lógico (público por padrão)
SELECT * FROM public.Habits;

-- SQL Server: schema = namespace + unidade de segurança
-- Padrão é 'dbo' (database owner)
SELECT * FROM dbo.Habits;
```

No EF Core ambos mapeiam para o mesmo conceito: `.ToTable("Habits", "dbo")` ou `.ToTable("Habits", "public")`.

---

## Concorrência: Row Version no SQL Server

O SQL Server tem um recurso chamado `ROWVERSION` (antigo `TIMESTAMP` — nome infeliz, não tem nada a ver com data/hora). É um número binário de 8 bytes que incrementa automaticamente a cada insert/update na linha. O EF Core mapeia isso com `IsRowVersion()`:

```csharp
// Model
public class Habit
{
    public Guid Id { get; init; }
    public string Name { get; set; }

    [Timestamp]
    public byte[] Version { get; set; } = Array.Empty<byte>();
    // PostgreSQL: usa xmin interno (uint)
    // SQL Server: mapeia para ROWVERSION (byte[])
}

// Fluent API
modelBuilder.Entity<Habit>()
    .Property(h => h.Version)
    .IsRowVersion();
```

O comportamento é idêntico ao que vimos no tópico de concorrência otimista: o EF Core inclui a coluna `Version` na cláusula `WHERE` do `UPDATE`, e se o valor não bater (outra transação já modificou a linha), lança `DbUpdateConcurrencyException`.

**Diferença de tipo entre providers**: com PostgreSQL/Npgsql, `IsRowVersion()` usa `xmin` (tipo `uint`). Com SQL Server, usa `ROWVERSION` (tipo `byte[]`). Na prática, você tipifica a propriedade de acordo com o provider que está usando, ou usa uma abstração genérica.

---

## EF Core com SQL Server

### Provider

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

### Configuração

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

### Connection string

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost,1433;Database=habitrack;User Id=sa;Password=Senha@123;TrustServerCertificate=true"
  }
}
```

A diferença principal para PostgreSQL:
- `Host=localhost` → `Server=localhost,1433`
- `Username=postgres` → `User Id=sa` (system administrator)
- `Password=postgres` → `Password=Senha@123` (precisa ter maiúscula, minúscula, número e caractere especial — é uma restrição do SQL Server, não do Docker)
- `TrustServerCertificate=true` — necessário para conexões locais/dev porque o SQL Server usa certificado auto-assinado por padrão. Em produção, você configura um certificado real.

### Diferenças no DbContext

```csharp
// Providers diferentes, mesma interface
if (useSqlServer)
    options.UseSqlServer(connectionString);
else
    options.UseNpgsql(connectionString);

// O resto é IDÊNTICO:
// - Migrations
// - LINQ queries
// - SaveChangesAsync
// - Change Tracker
// - Transaction API
```

**Essa é a beleza do EF Core**: o provider é um detalhe de configuração. Exceto por recursos específicos de cada banco (como `JSONB` no Postgres ou `ROWVERSION` no SQL Server), o código de acesso a dados é o mesmo.

### Migrations com múltiplos providers

Se você quiser testar o mesmo projeto com PostgreSQL e SQL Server, o EF Core suporta múltiplos providers:

```bash
# Migrations isoladas por provider
dotnet ef migrations add InitialCreate --context AppDbContext \
  --output-dir Migrations/Postgres
dotnet ef migrations add InitialCreate --context SqlServerDbContext \
  --output-dir Migrations/SqlServer
```

Mas na prática, a maioria dos projetos escolhe um banco e fica com ele. A flexibilidade é útil para testes: rodar SQLite em memória nos testes de integração e PostgreSQL/SQL Server em produção.

---

## Recursos exclusivos que aparecem em vagas

### 1. SQL Server Agent

Sistema de agendamento de jobs nativo. Equivalente ao `pg_cron` + `pgAgent` do PostgreSQL. Muito usado para:
- Backups automáticos
- Rebuild de índices
- ETL/importação de dados
- Envio de relatórios por email

É um dos recursos mais citados em vagas que pedem "conhecimento de administração de SQL Server".

### 2. Full-Text Search

Busca textual indexada, mais poderosa que `LIKE '%termo%'`. Equivalente ao `tsvector`/`tsquery` do PostgreSQL.

### 3. SSIS (SQL Server Integration Services)

Ferramenta de ETL (Extract, Transform, Load) — extrai dados de fontes heterogêneas (Excel, CSV, Oracle, etc.), transforma e carrega no SQL Server. Em projetos corporativos, é comum encontrar pacotes SSIS que fazem integração com sistemas legados.

### 4. SSRS (SQL Server Reporting Services)

Servidor de relatórios. Gera relatórios paginados (PDF, Excel) a partir de queries SQL. Muito usado em intranets corporativas.

### 5. Filestream / FileTable

Armazenamento de arquivos binários diretamente no banco, com acesso via sistema de arquivos Windows. Usado para cenários como "anexos de processo judicial" ou "imagens de produtos" onde as regras de compliance exigem que o arquivo esteja junto com o registro no banco.

### 6. Always Encrypted

Criptografia de colunas onde **o banco nunca vê os dados em texto claro**. A aplicação envia os dados já criptografados e o SQL Server só armazena/recupera os bytes cifrados. Útil para dados sensíveis (CPF, cartão de crédito) em ambientes regulados (LGPD, PCI-DSS).

---

## Rodando SQL Server via Docker no Arch Linux

Como não existe instalação nativa no Arch, o Docker é o caminho padrão:

```bash
# Subir container
docker run -d \
  --name sqlserver \
  -e 'ACCEPT_EULA=Y' \
  -e 'MSSQL_SA_PASSWORD=Senha@123456' \
  -p 1433:1433 \
  mcr.microsoft.com/mssql/server:2022-latest

# Verificar se subiu
docker logs sqlserver

# Conectar via CLI
docker exec -it sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P 'Senha@123456' -No

# Dentro do sqlcmd:
# 1> SELECT @@VERSION;
# 2> GO
# 1> CREATE DATABASE habitrack;
# 2> GO
# 1> quit

# Parar/remover quando terminar
docker stop sqlserver
docker rm sqlserver
```

**Restrições da senha SA**: a senha do usuário `sa` (system administrator) precisa ter pelo menos 8 caracteres, com maiúsculas, minúsculas, números e caracteres especiais. Se não atender a esses critérios, o container recusa iniciar.

**Recursos do container**: a imagem Docker do SQL Server **não** inclui SQL Agent, SSIS, SSRS ou Full-Text Search. É uma instância mínima do engine, suficiente para desenvolvimento. Para recursos completos, você precisaria de uma instalação no Windows ou de uma licença Developer Edition.

### Performance no Docker

O SQL Server em container no Linux tem performance comparável à instalação nativa para desenvolvimento. Em produção, a Microsoft recomenda configurar `--memory` e `--cpus` adequados:

```bash
docker run -d \
  --name sqlserver \
  -e 'ACCEPT_EULA=Y' \
  -e 'MSSQL_SA_PASSWORD=Senha@123456' \
  -e 'MSSQL_MEMORY_LIMIT_MB=2048' \
  --memory=4g \
  --cpus=2 \
  -p 1433:1433 \
  mcr.microsoft.com/mssql/server:2022-latest
```

---

## PostgreSQL vs SQL Server: quando usar qual?

A pergunta que vai aparecer na entrevista não é técnica — é de decisão arquitetural.

| Cenário | Recomendação |
|---|---|
| Startup nova, stack livre, orçamento apertado | PostgreSQL. É gratuito, tem features modernas (JSONB, arrays, full-text search nativo), e funciona nativamente em qualquer Linux. |
| Empresa já roda infra Microsoft (Windows Server, Active Directory, Azure) | SQL Server. A integração com o ecossistema reduz custo operacional: backups, monitoring, HA/DR já são conhecidos pela equipe. |
| Sistema precisa de ETL/reporting corporativo | SQL Server. SSIS + SSRS resolvem isso com menos esforço do que montar um pipeline com ferramentas externas. |
| Microsserviços com containers Kubernetes | PostgreSQL. Imagem menor, inicia mais rápido, não tem restrição de licenciamento por core. |
| Compliance exige criptografia de colunas sem expor ao DBA | SQL Server. Always Encrypted é um diferencial real — no PostgreSQL você implementaria isso na camada de aplicação. |
| Alta concorrência com muitas escritas simultâneas | PostgreSQL. O MVCC do PostgreSQL é superior em workloads write-heavy. O SQL Server usa tempdb para version store, que pode virar gargalo. |
| Você quer aprender algo que cobre 70% das vagas .NET | SQL Server. É o que mais aparece. |
| Você quer usar o banco mais moderno e flexível | PostgreSQL. JSONB, arrays, extensões (PostGIS, pgvector) são features que o SQL Server não tem equivalente direto. |

### A resposta de entrevista

"Eu uso PostgreSQL no dia a dia por ser nativo do Linux e ter features que eu valorizo, como JSONB para dados semiestruturados. Mas tenho experiência com SQL Server via Docker, entendo as diferenças de T-SQL, sei configurar o EF Core com os dois providers, e conheço os recursos corporativos como SQL Agent e SSIS — mesmo que eu não seja DBA. Na prática, a escolha depende do ecossistema da empresa: se já usam infra Microsoft, SQL Server faz sentido. Se é um projeto novo com stack aberta, PostgreSQL."

---

## Como testar o habitrack-api com SQL Server

Para alternar entre PostgreSQL e SQL Server, basta trocar o provider e a connection string:

```bash
# Instalar provider SQL Server
dotnet add package Microsoft.EntityFrameworkCore.SqlServer

# Atualizar appsettings.json
# "Default": "Server=localhost,1433;Database=habitrack;User Id=sa;Password=Senha@123456;TrustServerCertificate=true"

# Atualizar Program.cs
options.UseSqlServer(connectionString) // em vez de UseNpgsql

# Rodar migrations
dotnet ef migrations add InitialCreate_SqlServer
dotnet ef database update
```

O resto do código — DbContext, repositórios, Unit of Work, serviços, endpoints — **permanece idêntico**. Essa é a prova prática de que o EF Core cumpre sua promessa de abstrair o banco.

**Atenção**: `Guid` com `init` + `= Guid.NewGuid()` funciona no SQL Server, mas o EF Core não vai delegar a geração do GUID para o banco. Se você quiser usar `NEWSEQUENTIALID()` (GUIDs sequenciais que performam melhor em índices clusterizados), precisa configurar:

```csharp
modelBuilder.Entity<Habit>()
    .Property(h => h.Id)
    .HasDefaultValueSql("NEWSEQUENTIALID()");
```

Isso é uma otimização específica de SQL Server — no PostgreSQL, `uuid_generate_v4()` seria o equivalente, mas o EF Core já gera o GUID em memória por padrão.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar por que SQL Server aparece em tantas vagas .NET (histórico, ecossistema, legado)
- [ ] Subir um container SQL Server via Docker no Arch Linux e conectar via `sqlcmd`
- [ ] Listar pelo menos 3 diferenças práticas entre T-SQL e PL/pgSQL
- [ ] Configurar o EF Core com `UseSqlServer()` e rodar migrations
- [ ] Explicar o que é `ROWVERSION` e como ele se compara ao `xmin` do PostgreSQL
- [ ] Citar 2 recursos exclusivos do SQL Server que são diferenciais em ambientes corporativos
- [ ] Argumentar quando usar PostgreSQL vs SQL Server numa decisão arquitetural
- [ ] Saber que `TrustServerCertificate=true` é necessário para dev local e por que isso não é aceitável em produção
