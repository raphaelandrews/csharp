# 05 — Segurança: OWASP Top 10, JWT Refresh e CORS

## OWASP Top 10 aplicado a APIs .NET

### 1. Broken Access Control (#1 no OWASP)

O erro mais comum: confiar que o frontend esconde opções. Sempre verifique permissões no backend:

```csharp
// RUIM: confia que o frontend só mostra eventos do usuário
app.MapDelete("/api/events/{id}", async (Guid id, AppDbContext db) =>
{
    var @event = await db.Events.FindAsync(id);
    db.Events.Remove(@event); // qualquer usuário pode deletar qualquer evento!
    await db.SaveChangesAsync();
});

// BOM: verifica ownership no backend
app.MapDelete("/api/events/{id}", async (Guid id, AppDbContext db, HttpContext http) =>
{
    var userId = GetUserId(http);
    var @event = await db.Events.FirstOrDefaultAsync(e => e.Id == id && e.OrganizerId == userId);
    if (@event is null) return Results.NotFound();
    db.Events.Remove(@event);
    await db.SaveChangesAsync();
});
```

### 2. Cryptographic Failures (#2)

```csharp
// RUIM: hash fraco
user.PasswordHash = MD5.Create().ComputeHash(Encoding.UTF8.GetBytes(password));

// BOM: PBKDF2 com salt + iterações (já implementamos no habitrack)
byte[] salt = RandomNumberGenerator.GetBytes(16);
byte[] hash = KeyDerivation.Pbkdf2(password, salt, prf: HMACSHA256, iterationCount: 100_000, numBytes: 32);
```

### 3. Injection (#3)

EF Core com LINQ parametrizado já protege contra SQL injection:

```csharp
// BOM: LINQ parametrizado — nunca concatena strings para SQL
var events = await _db.Events
    .Where(e => e.Title.Contains(searchTerm))
    .ToListAsync();

// RUIM: SQL cru com concatenação
var events = _db.Events.FromSqlRaw($"SELECT * FROM Events WHERE Title LIKE '%{searchTerm}%'");
```

### 4. Security Misconfiguration (#5)

- **HTTPS em produção**: `app.UseHttpsRedirection()`
- **Headers de segurança**: `app.UseHsts()` (HTTP Strict Transport Security)
- **Desabilitar Server header**: `builder.WebHost.ConfigureKestrel(o => o.AddServerHeader = false)`

---

## JWT Refresh Token

O access token JWT deve ter vida curta (15-30min). Se for roubado, expira rápido. O refresh token tem vida longa (7-30 dias) e é usado para obter novos access tokens sem re-login:

```csharp
// AuthService — geração de tokens
public record Tokens(string AccessToken, string RefreshToken);

public async Task<Tokens> GenerateTokens(User user)
{
    var accessToken = GenerateAccessToken(user);     // expira em 15 min
    var refreshToken = GenerateRefreshToken();         // expira em 7 dias

    // Salva refresh token no banco (hash, não plain text)
    user.RefreshTokenHash = HashToken(refreshToken);
    user.RefreshTokenExpiresAt = DateTime.UtcNow.AddDays(7);
    await _db.SaveChangesAsync();

    return new Tokens(accessToken, refreshToken);
}

public async Task<Tokens?> RefreshAccessToken(string refreshToken)
{
    var hashed = HashToken(refreshToken);

    var user = await _db.Users.FirstOrDefaultAsync(u =>
        u.RefreshTokenHash == hashed &&
        u.RefreshTokenExpiresAt > DateTime.UtcNow);

    if (user is null) return null;

    return await GenerateTokens(user); // rotate: gera novo par
}

// Rotação de refresh token: ao usar um refresh token, ele é invalidado
// e um novo é gerado. Isso limita o dano se um refresh token for roubado.
```

### Por que rotação de refresh token

Sem rotação: se um atacante rouba um refresh token, ele tem acesso por 7 dias. Com rotação: o refresh token é usado uma vez e substituído por um novo. Se o atacante usa o token roubado, o usuário legítimo percebe (o token dele foi invalidado) e o atacante perde acesso na próxima rotação.

---

## CORS: não use `AllowAnyOrigin` em produção

```csharp
// Desenvolvimento — ok
builder.Services.AddCors(options =>
{
    options.AddPolicy("Dev", policy =>
        policy.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod());
});

// Produção — explícito
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
        policy.WithOrigins("https://meuapp.com", "https://admin.meuapp.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Authorization", "Content-Type")
              .WithExposedHeaders("X-RateLimit-Remaining"));
});

app.UseCors("Production");
```

`AllowAnyOrigin()` + `AllowCredentials()` não podem coexistir em produção — é uma violação de segurança que qualquer navegador moderno rejeita.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Implementar verificação de ownership no backend (não confiar no frontend)
- [ ] Explicar por que o access token deve ter vida curta (15-30min) e o refresh token vida longa
- [ ] Implementar rotação de refresh token (invalida o antigo ao gerar novo)
- [ ] Configurar CORS com origens explícitas (não `AllowAnyOrigin()`)
- [ ] Saber que EF Core + LINQ parametrizado previne SQL injection automaticamente
- [ ] Usar PBKDF2 (ou Argon2/bcrypt) para hash de senhas, nunca MD5/SHA1 puro
