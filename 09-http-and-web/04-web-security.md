# Web Security

## Authentication vs Authorization

| Concept | Question | Example |
|---------|----------|---------|
| **Authentication** | Who are you? | Login with email/password, JWT |
| **Authorization** | Can you do this? | Admin role, policy, claim |

## JWT (JSON Web Token)

Token composed of 3 parts (separated by `.`):

```
header.payload.signature
eyJhbGci...eyJzdWIi...SflKxwRJ...
```

### Structure

```json
// Header
{ "alg": "HS256", "typ": "JWT" }

// Payload (claims)
{
  "sub": "user-123",
  "name": "Osvaldo",
  "role": "admin",
  "exp": 1712000000,
  "iss": "my-api",
  "aud": "my-app"
}

// Signature
HMACSHA256(base64(header) + "." + base64(payload), secret)
```

### Validation in ASP.NET Core

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "my-api",
            ValidateAudience = true,
            ValidAudience = "my-app",
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(config["Jwt:Secret"]!))
        };
    });
```

### Access Token + Refresh Token

```
1. Login → API returns Access Token (short-lived, ~15min) + Refresh Token (long-lived, ~7days)
2. Requests use Access Token in the Authorization header
3. Access Token expires → client uses Refresh Token to obtain a new pair
4. Refresh Token expires → user needs to log in again
```

> A **short-lived** Access Token limits the attack window if it is leaked.

## CORS (Cross-Origin Resource Sharing)

Browser mechanism that **blocks** requests from different origins by default.

```
https://my-site.com (frontend)  →  https://api.my-site.com (API)
                                      ↑ Different origin = CORS
```

### Configuration in ASP.NET Core

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("MyFrontend", policy =>
    {
        policy.WithOrigins("https://my-site.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

app.UseCors("MyFrontend");
```

> **Never** use `AllowAnyOrigin()` with `AllowCredentials()` in production.

## OWASP Top 10 (main vulnerabilities)

### 1. Injection (SQL, Command)

```csharp
// VULNERABLE — string concatenation
var query = $"SELECT * FROM Users WHERE Name = '{input}'";

// SAFE — parameters
var query = "SELECT * FROM Users WHERE Name = @name";
command.Parameters.AddWithValue("@name", input);

// SAFE — EF Core (parameterizes automatically)
var user = await _context.Users.FirstOrDefaultAsync(u => u.Name == input);
```

### 2. Broken Authentication

- Weak passwords without requirements
- No rate limiting on login (brute force)
- Tokens without expiration
- No MFA

### 3. XSS (Cross-Site Scripting)

```html
<!-- VULNERABLE: rendering user input without sanitizing -->
<p>@Html.Raw(userInput)</p>

<!-- SAFE: Razor sanitizes automatically -->
<p>@userInput</p>
```

### 4. CSRF (Cross-Site Request Forgery)

Someone forces the user's browser to make an unwanted request:

```csharp
// Protection in ASP.NET Core (automatic with forms)
[ValidateAntiForgeryToken]
[HttpPost]
public IActionResult Transfer(TransferDto dto) { ... }
```

> REST APIs with JWT in the header **do not need** anti-CSRF (the token is not sent automatically).

## HTTPS

**Always** enforce HTTPS:

```csharp
app.UseHttpsRedirection();
app.UseHsts(); // HTTP Strict Transport Security
```

## Security headers

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

| Approach | When to use |
|----------|-------------|
| **Azure Key Vault** | Production (Azure) |
| **AWS Secrets Manager** | Production (AWS) |
| **User Secrets** (.NET) | Local development |
| **Environment variables** | CI/CD, containers |
| **appsettings.json** | **Non-sensitive** configurations only |

```csharp
// User Secrets (dev)
dotnet user-secrets set "Jwt:Secret" "my-super-secret-key"

// Azure Key Vault (prod)
builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

> **Never** commit secrets to Git. Use `.gitignore` for `appsettings.Development.json`.

## Security checklist

- [ ] HTTPS enforced
- [ ] JWT with short expiration + refresh token
- [ ] CORS configured (not `AllowAnyOrigin`)
- [ ] Input validation on all inputs
- [ ] Parameterized queries (never concatenate SQL)
- [ ] Rate limiting on login and public APIs
- [ ] Secrets in vault, not in code
- [ ] Security headers configured
- [ ] Logging of authentication attempts
- [ ] Dependencies updated (known vulnerabilities)

---

[← Previous: REST API Design](03-rest-api-design.md) | [Back to index](README.md)
