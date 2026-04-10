# Seguranca Web

## Autenticacao vs Autorizacao

| Conceito | Pergunta | Exemplo |
|----------|----------|---------|
| **Autenticacao** | Quem e voce? | Login com email/senha, JWT |
| **Autorizacao** | Voce pode fazer isso? | Role admin, policy, claim |

## JWT (JSON Web Token)

Token composto de 3 partes (separadas por `.`):

```
header.payload.signature
eyJhbGci...eyJzdWIi...SflKxwRJ...
```

### Estrutura

```json
// Header
{ "alg": "HS256", "typ": "JWT" }

// Payload (claims)
{
  "sub": "user-123",
  "name": "Osvaldo",
  "role": "admin",
  "exp": 1712000000,
  "iss": "minha-api",
  "aud": "minha-app"
}

// Signature
HMACSHA256(base64(header) + "." + base64(payload), secret)
```

### Validacao no ASP.NET Core

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "minha-api",
            ValidateAudience = true,
            ValidAudience = "minha-app",
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(config["Jwt:Secret"]!))
        };
    });
```

### Access Token + Refresh Token

```
1. Login → API retorna Access Token (curto, ~15min) + Refresh Token (longo, ~7dias)
2. Requisicoes usam Access Token no header Authorization
3. Access Token expira → client usa Refresh Token para obter novo par
4. Refresh Token expira → usuario precisa logar novamente
```

> Access Token **curto** limita a janela de ataque se for vazado.

## CORS (Cross-Origin Resource Sharing)

Mecanismo do browser que **bloqueia** requisicoes de origens diferentes por padrao.

```
https://meu-site.com (frontend)  →  https://api.meu-site.com (API)
                                      ↑ Origem diferente = CORS
```

### Configuracao no ASP.NET Core

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("MeuFrontend", policy =>
    {
        policy.WithOrigins("https://meu-site.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

app.UseCors("MeuFrontend");
```

> **Nunca** use `AllowAnyOrigin()` com `AllowCredentials()` em producao.

## OWASP Top 10 (principais vulnerabilidades)

### 1. Injection (SQL, Command)

```csharp
// VULNERAVEL — concatenacao de string
var query = $"SELECT * FROM Users WHERE Name = '{input}'";

// SEGURO — parametros
var query = "SELECT * FROM Users WHERE Name = @name";
command.Parameters.AddWithValue("@name", input);

// SEGURO — EF Core (parametriza automaticamente)
var user = await _context.Users.FirstOrDefaultAsync(u => u.Name == input);
```

### 2. Broken Authentication

- Senhas fracas sem requisitos
- Sem rate limiting no login (brute force)
- Tokens sem expiracao
- Sem MFA

### 3. XSS (Cross-Site Scripting)

```html
<!-- VULNERAVEL: renderizar input do usuario sem sanitizar -->
<p>@Html.Raw(userInput)</p>

<!-- SEGURO: Razor sanitiza automaticamente -->
<p>@userInput</p>
```

### 4. CSRF (Cross-Site Request Forgery)

Alguem forca o browser do usuario a fazer requisicao indesejada:

```csharp
// Protecao em ASP.NET Core (automatico com forms)
[ValidateAntiForgeryToken]
[HttpPost]
public IActionResult Transferir(TransferenciaDto dto) { ... }
```

> APIs REST com JWT no header **nao precisam** de anti-CSRF (o token nao e enviado automaticamente).

## HTTPS

**Sempre** force HTTPS:

```csharp
app.UseHttpsRedirection();
app.UseHsts(); // HTTP Strict Transport Security
```

## Headers de seguranca

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Append("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
    await next();
});
```

## Secrets Management

| Abordagem | Quando usar |
|-----------|-------------|
| **Azure Key Vault** | Producao (Azure) |
| **AWS Secrets Manager** | Producao (AWS) |
| **User Secrets** (.NET) | Desenvolvimento local |
| **Environment variables** | CI/CD, containers |
| **appsettings.json** | Configuracoes **nao sensiveis** apenas |

```csharp
// User Secrets (dev)
dotnet user-secrets set "Jwt:Secret" "minha-chave-super-secreta"

// Azure Key Vault (prod)
builder.Configuration.AddAzureKeyVault(
    new Uri("https://meu-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

> **Nunca** commite secrets no Git. Use `.gitignore` para `appsettings.Development.json`.

## Checklist de seguranca

- [ ] HTTPS forcado
- [ ] JWT com expiracao curta + refresh token
- [ ] CORS configurado (nao `AllowAnyOrigin`)
- [ ] Input validation em todas as entradas
- [ ] Parametrizacao de queries (nunca concatenar SQL)
- [ ] Rate limiting no login e APIs publicas
- [ ] Secrets em vault, nao no codigo
- [ ] Headers de seguranca configurados
- [ ] Logging de tentativas de autenticacao
- [ ] Dependencias atualizadas (vulnerabilidades conhecidas)

---

[← Anterior: REST API Design](03-rest-api-design.md) | [Voltar ao índice](README.md)
