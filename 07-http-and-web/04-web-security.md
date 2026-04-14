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
                Encoding.UTF8.GetBytes(config["Jwt:Secret"]!)),
            // Default is 5 minutes — tokens live past their `exp`. Set to zero
            // unless you have clock-drift requirements.
            ClockSkew = TimeSpan.Zero
        };
    });
```

> **`alg` confusion attack:** if your server expects `RS256` (asymmetric) but blindly trusts the token header, an attacker can forge a token signed with `HS256` using your **public key as the HMAC secret**, and the library will accept it. Always pin the expected algorithm (`ValidAlgorithms` / explicit `SecurityTokenValidator`) and never let the token's own `alg` header pick the verification method.

### Access Token + Refresh Token

```
1. Login → API returns Access Token (short-lived, ~15min) + Refresh Token (long-lived, ~7days)
2. Requests use Access Token in the Authorization header
3. Access Token expires → client uses Refresh Token to obtain a new pair
4. Refresh Token expires → user needs to log in again
```

> A **short-lived** Access Token limits the attack window if it is leaked.

### Refresh token rotation

A leaked refresh token is a long-lived credential — rotation limits the damage.

1. Every call to `/token` (refresh) issues a **new** access token **and a new refresh token**.
2. The **old** refresh token is **invalidated** immediately.
3. If an old (already-rotated) refresh token is ever used again, treat it as a **leak**: **invalidate the entire token family** (every refresh descended from the same original login) and force re-authentication.

```
login         → RT₁
refresh(RT₁)  → RT₂  (RT₁ invalidated)
refresh(RT₂)  → RT₃  (RT₂ invalidated)

Attacker tries refresh(RT₁) → reuse detected
  → invalidate RT₂, RT₃, and all descendants. Force login.
```

> This is called **reuse detection**. It is the standard way to bound the blast radius of a stolen refresh token.

## OAuth 2 and OpenID Connect (OIDC)

**OAuth 2** is an **authorization** framework: it lets an app get an **access token** to call an API on the user's behalf. It does **not** define how to identify the user.

**OIDC** is **OAuth 2 + an identity layer**. On top of the access token, it returns an `id_token` (a JWT with user claims like `sub`, `email`, `name`) so the client knows **who** logged in.

### Grant types (pick the right one)

| Grant type | Use case |
|------------|----------|
| **Authorization Code + PKCE** | Default for **public clients** (SPAs, mobile, desktop). PKCE prevents code interception |
| **Authorization Code** (with client secret) | Traditional server-side web apps with a confidential backend |
| **Client Credentials** | **Service-to-service** — no user involved. The service authenticates with its own credentials |
| **Device Code** | Devices with no browser/keyboard (smart TVs, CLI tools). User authorizes from another device |
| **Refresh Token** | Exchanged at `/token` for a new access token when the old one expires |

### Deprecated grants (do not use)

- **Implicit flow** — returned the access token directly in the URL fragment. Leaked via referrer headers and browser history. **Replaced by Authorization Code + PKCE**, which SPAs can now use safely.
- **Resource Owner Password Credentials (ROPC)** — the app collects the user's username/password and sends them to the IdP. Defeats the whole point of federated login (phishing, no MFA support, no SSO). Only acceptable for migrating legacy systems.

### OAuth 2 vs OIDC in one sentence

> OAuth 2 answers **"can this app call this API?"**. OIDC also answers **"who is the user?"** by adding the `id_token`.

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

> REST APIs with JWT in the **`Authorization` header** do not need anti-CSRF — browsers do not attach custom headers automatically across origins, so an attacker's page cannot forge the request.
>
> **Caveat:** if the JWT is stored in a **cookie** (even `HttpOnly`), CSRF **still applies** — the browser sends cookies automatically on cross-site requests. In that case, use anti-CSRF tokens or `SameSite=Lax/Strict` cookies.

### Cookie attributes (core CSRF/session defenses)

| Attribute | Effect |
|-----------|--------|
| `HttpOnly` | JavaScript cannot read the cookie (`document.cookie`). Mitigates XSS token theft |
| `Secure` | Cookie only sent over HTTPS |
| `SameSite=Strict` | Never sent on cross-site requests. Strongest CSRF defense; breaks cross-site logins |
| `SameSite=Lax` | Sent on top-level navigations (GET) but not on cross-site POST/fetch. Modern default |
| `SameSite=None` | Sent on all cross-site requests. **Requires `Secure`**. Needed for third-party embeds |

```csharp
context.Response.Cookies.Append("session", token, new CookieOptions
{
    HttpOnly = true,
    Secure = true,
    SameSite = SameSiteMode.Lax,
    Expires = DateTimeOffset.UtcNow.AddHours(1)
});
```

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
    // X-XSS-Protection is deprecated. Chrome removed the XSS auditor
    // and enabling it has historically introduced new vulnerabilities.
    // Set to "0" to disable, and rely on CSP instead.
    context.Response.Headers.Append("X-XSS-Protection", "0");
    context.Response.Headers.Append("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
    await next();
});
```

> `X-XSS-Protection` is **superseded by Content Security Policy (CSP)**. Modern browsers ignore it; older ones were safer with it off.

### Content Security Policy (CSP)

CSP is the modern defense against XSS. The browser enforces an allow-list of sources for scripts, styles, images, connections, etc. If an attacker injects a `<script>` tag, the browser refuses to execute it because it is not on the allow-list.

```csharp
context.Response.Headers.Append(
    "Content-Security-Policy",
    "default-src 'self'; " +
    "script-src 'self' 'nonce-abc123'; " +
    "style-src 'self' 'unsafe-inline'; " +
    "img-src 'self' data: https:; " +
    "object-src 'none'; " +
    "frame-ancestors 'none'");
```

| Directive | Meaning |
|-----------|---------|
| `default-src 'self'` | Fallback: everything must come from the same origin |
| `script-src` | Where scripts can load from. Use **nonces** (per-request random value) or **hashes** instead of `'unsafe-inline'` |
| `style-src` | Where stylesheets can load from |
| `object-src 'none'` | Blocks `<object>`, `<embed>`, `<applet>` (legacy XSS vectors) |
| `frame-ancestors 'none'` | Modern replacement for `X-Frame-Options: DENY` (clickjacking defense) |

> Prefer **nonces** (`<script nonce="abc123">`) over `'unsafe-inline'`. The nonce is a per-request random value that only your server knows.

> See also: [Middleware Pipeline](../08-aspnet-core/05-middleware.md) for implementation details.

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
