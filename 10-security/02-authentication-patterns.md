# Authentication Patterns

Authentication answers *"who is this?"*. The three patterns you're expected to know cold in a senior interview are **cookies**, **JWT bearer tokens**, and **OpenID Connect**. Pick the wrong one and you pay for it in session management, logout UX, or refresh-token security for years.

## Session vs stateless at a glance

| Dimension | Cookies (session) | JWT bearer (stateless) |
|---|---|---|
| Who stores state | Server (session store) or signed payload in the cookie | Client holds the token |
| Transport | `Set-Cookie` + browser auto-send | `Authorization: Bearer <jwt>` header |
| Logout / revoke | Delete server session, or rotate key | Hard — need a token deny-list or short TTL |
| CSRF risk | Yes — needs antiforgery | No (header not auto-sent cross-site) — XSS becomes the bigger risk |
| Best fit | Server-rendered apps (MVC, Razor Pages) | APIs, microservice-to-microservice, SPAs via BFF |
| Scaling | Needs sticky sessions or shared session store | Stateless; any node can validate |

> The default ASP.NET Core authentication cookie is encrypted with the [Data Protection API](https://learn.microsoft.com/aspnet/core/security/data-protection/introduction), so the claims live *inside* the cookie — no server-side store is needed unless you want one. That makes "cookies = always stateful" a common myth.

## Cookie authentication in ASP.NET Core

```csharp
using Microsoft.AspNetCore.Authentication.Cookies;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.ExpireTimeSpan = TimeSpan.FromMinutes(20);
        options.SlidingExpiration = true;
        options.AccessDeniedPath = "/forbidden";
    });

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.Run();
```

To sign a user in, construct a `ClaimsPrincipal` and call `HttpContext.SignInAsync`. To sign out, `HttpContext.SignOutAsync`.

### Cookie security checklist

- `Secure = true` (HTTPS only). Enforced automatically when the cookie is essential and the request is HTTPS.
- `HttpOnly = true` (default). Blocks `document.cookie` reads from JS — reduces XSS impact.
- `SameSite = Lax` by default (allows OAuth callbacks); `Strict` breaks cross-site sign-in flows — know the trade-off.
- For multi-server deployments, [configure Data Protection](https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview) with a shared key ring (Redis, Azure Blob, file share) — otherwise each node encrypts with a different key and cookies from node A look invalid on node B.

## JWT bearer in ASP.NET Core

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenant}/v2.0";
        options.Audience  = "api://my-api";
        options.TokenValidationParameters = new()
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
        };
    });

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

> `AddJwtBearer` validates tokens — it does **not issue** them. Issuance is the IdP's job (Entra ID, Auth0, Keycloak, IdentityServer/Duende, OpenIddict). The `dotnet user-jwts` CLI can issue tokens for local dev only.

### JWT pitfalls senior interviewers probe for

- **No revocation.** A stolen JWT is valid until it expires. Mitigation: short access-token lifetime (5–15 min) + refresh tokens.
- **`alg: none`.** Libraries that accept `none` are dangerous. `Microsoft.IdentityModel.Tokens` rejects it by default; verify any third-party JWT library does the same.
- **Audience / issuer validation off.** Turning these off "to make it work in dev" is the most common way a service accepts tokens meant for someone else.
- **Storing JWTs in `localStorage`.** Any XSS = token theft. Prefer cookies with `HttpOnly` + BFF, or browser-isolated storage.
- **Trusting claims from a self-signed token.** If you sign with HS256 and share the secret, anyone with the secret can forge tokens. Prefer RS256/ES256 (public-key verification).

## OpenID Connect (OIDC)

OIDC is "a simple identity layer on top of OAuth 2.0". It adds:

- An **ID Token** (a JWT) that proves *who* the user is — something plain OAuth doesn't give you.
- A standard **`/userinfo`** endpoint.
- Standard **scopes** (`openid`, `profile`, `email`).

Use OIDC when you integrate with an identity provider (Entra ID, Okta, Auth0, Google, Apple) or when you build an SSO story across multiple apps.

```csharp
builder.Services.AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = "oidc";
    })
    .AddCookie()
    .AddOpenIdConnect("oidc", options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenant}/v2.0";
        options.ClientId = "...";
        options.ClientSecret = "..."; // only for confidential clients
        options.ResponseType = "code";     // authorization code flow
        options.UsePkce = true;            // required for public clients; recommended for confidential
        options.SaveTokens = true;
        options.Scope.Add("openid");
        options.Scope.Add("profile");
        options.Scope.Add("email");
    });
```

## OAuth 2.0 Authorization Code flow + PKCE (RFC 7636)

The flow every SPA and mobile app should use today:

1. Client generates a random `code_verifier` (43–128 characters per RFC 7636).
2. Client computes `code_challenge = BASE64URL(SHA256(code_verifier))` and sends it to `/authorize` with `code_challenge_method=S256`.
3. User authenticates; authorization server returns a one-time `code` via redirect.
4. Client POSTs `code` **plus the raw `code_verifier`** to `/token`.
5. Server checks `BASE64URL(SHA256(code_verifier)) == code_challenge`. If yes, issues tokens.

> PKCE closes the authorization-code interception attack that affects public clients (SPAs, mobile, desktop) that cannot keep a `client_secret`. Modern guidance (OAuth 2.1 draft) recommends PKCE for **all** clients, including confidential ones.

### When each flow applies

| Client | Flow | Notes |
|---|---|---|
| Server-side web app with a backend | Authorization Code + PKCE | Confidential client; can use `client_secret` too |
| SPA (no server code) | Authorization Code + PKCE | Never use the deprecated Implicit flow |
| Native / mobile | Authorization Code + PKCE via system browser | Use AppAuth; never embed the IdP in a WebView |
| Service-to-service | Client Credentials | No user involved |
| Legacy / ROPC (username+password) | **Avoid.** Deprecated in OAuth 2.1 draft |

## Refresh tokens

An access token expires in minutes; a refresh token trades for a new access token without prompting the user.

### Rules

- **Rotate on use**: the IdP issues a new refresh token with every refresh and invalidates the old one. Any replay of the old one signals theft → revoke the whole family.
- **Bind to the client**: for public clients, pair with PKCE and a **DPoP** or mTLS binding when the threat model demands it.
- **Server-side only** for confidential clients. Never expose a refresh token to a browser that doesn't strictly need it; prefer a **BFF (Backend-for-Frontend)** that keeps tokens on the server and gives the browser a cookie.
- **Short absolute lifetime**: e.g., 14 days, regardless of sliding activity.

## Patterns compared

| Use case | Recommended |
|---|---|
| MVC / Razor Pages monolith | Cookie auth + ASP.NET Core Identity |
| Public REST API used by a SPA | BFF with cookies ↔ OIDC + JWT on the back |
| Public REST API used by mobile | OIDC Authorization Code + PKCE; short-lived access + rotating refresh |
| Service-to-service inside a cluster | mTLS or OAuth Client Credentials |
| Internal dashboard across teams | SSO via OIDC against your corporate IdP |

## Interview traps

- Claiming JWT is "more secure than cookies" — it isn't. Each has a different attack surface.
- Using the Implicit flow "because it's simpler" — it's deprecated.
- Putting access tokens in `localStorage` — any XSS = full account takeover.
- Same secret for signing JWTs and encrypting other data — rotate independently.
- Forgetting PKCE on mobile/SPA clients — free authorization-code interception.
- Not configuring Data Protection for a multi-instance ASP.NET Core app — cookies / antiforgery tokens fail randomly behind a load balancer.

---

[← Previous: OWASP Top 10 for .NET](01-owasp-top-10-dotnet.md) | [Next: Authorization →](03-authorization.md) | [Back to index](README.md)
