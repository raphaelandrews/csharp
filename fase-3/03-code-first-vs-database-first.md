# 03 — Code First vs Database First

Duas estratégias opostas de alinhar código C# com banco de dados.

---

## Code First — o código é a verdade

```
Classes C# → Migration → SQL → Banco
```

```csharp
// Você escreve:
public class Habit
{
    public Guid Id { get; init; }
    public string Name { get; set; }
}

// EF gera:
// CREATE TABLE "Habits" ("Id" uuid NOT NULL PRIMARY KEY, "Name" text NOT NULL);
```

**Vantagens**: banco evolui junto com código (mesma PR), zero SQL manual, portabilidade de banco, migrations são código versionado.

**Desvantagens**: EF pode gerar SQL não-otimizado, difícil otimizar índices sem conhecimento de SQL, não funciona com schema pré-existente.

---

## Database First — o banco é a verdade

```
Banco existente → dotnet ef dbcontext scaffold → Classes C# geradas
```

```bash
dotnet ef dbcontext scaffold \
    "Host=localhost;Database=habitrack" \
    Npgsql.EntityFrameworkCore.PostgreSQL \
    -o Models --context AppDbContext --force
```

Gera entidades + DbContext automaticamente a partir do schema existente.

**Vantagens**: banco desenhado por especialista (índices, partitions, stored procs), ideal pra sistemas legados e bancos compartilhados.

**Desvantagens**: scaffold unidirecional (mudar banco não atualiza código automaticamente), classes geradas são verbosas, risco de dessincronia.

---

## Qual usar?

| Cenário | Estratégia |
|---|---|
| Projeto greenfield | **Code First** |
| Banco legado migrando pra .NET | **Database First** → scaffold inicial, depois Code First |
| Banco gerenciado por DBA | **Database First** |
| Microsserviço com schema próprio | **Code First** |
| Banco compartilhado entre serviços | **Database First** |

**Mercado brasileiro**: Code First é padrão em projetos novos. Database First aparece em sistemas legados SQL Server + DBA. Saber os dois é diferencial.

---

## Estratégia híbrida: Scaffold → Code First

Quando você herda um banco existente mas quer usar migrations daqui pra frente:

```bash
# 1. Scaffold do banco atual
dotnet ef dbcontext scaffold "..." Npgsql.EntityFrameworkCore.PostgreSQL -o Models

# 2. Criar migration inicial vazia (snapshot do estado atual)
dotnet ef migrations add InitialExistingSchema
# → editar arquivo .cs da migration, deixar Up() e Down() vazios

# 3. Daqui pra frente: Code First normal
#    alterar classes C# → migration → database update
```
