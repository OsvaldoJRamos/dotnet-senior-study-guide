# OAuth 2.0

## What it is

**OAuth (Open Authorization)** allows third-party websites or apps to access user data **without the user needing to share their credentials**.

OAuth 2.0 is about **AUTHORIZATION**, not AUTHENTICATION. It is an authorization framework that can be used to authenticate users and grant them access to protected resources.

## Key concepts

- **Access Token** — token that grants access to protected resources. They usually expire quickly.
- **Refresh Token** — used to renew the access token. Not all flows use refresh tokens.
- **Authorization Server** — responsible for issuing access tokens.
- **Identity Provider** — responsible for authenticating users.
- **Resource Server** — server that holds the protected resources.

> Unlike Google, in some cases the Authorization Server and the Identity Provider can be different. In OAuth 2.0, although in some cases they can be the same thing (as with Google), the AUTHORIZATION server is responsible for issuing ACCESS TOKENS and the IDENTITY PROVIDER is responsible for AUTHENTICATING users.

## OAuth 2.0 Flow

```
1. Client ──── Authorization request ────→ Resource Owner
2. Client ←─── Authorization grant ──────  Resource Owner

3. Client ──── Authorization grant ────→ Authorization Server
4. Client ←─── Access token ────────── Authorization Server

5. Client ──── Access token ────────→ Resource Server
6. Client ←─── Protected resource ── Resource Server
```

## Practical example: Authorization Code + PKCE with Google

Suppose you're building a calendar app that reads Google Calendar on behalf of the user. The client **never generates its own token for Google** — it obtains one issued by Google via the Authorization Code + PKCE flow:

1. **Authorization request** — the client generates a random `code_verifier`, derives `code_challenge = SHA256(code_verifier)`, and redirects the user's browser to Google's authorization endpoint:

   ```
   GET https://accounts.google.com/o/oauth2/v2/auth
     ?client_id=...
     &redirect_uri=https://myapp.com/callback
     &response_type=code
     &scope=openid%20email%20https://www.googleapis.com/auth/calendar.readonly
     &code_challenge=<derived>
     &code_challenge_method=S256
     &state=<csrf-token>
   ```

2. **User authenticates** — Google authenticates the user and asks them to consent to the requested scopes.

3. **Authorization code returned** — Google redirects back to `redirect_uri` with a short-lived `code` (and the original `state`):

   ```
   https://myapp.com/callback?code=<auth-code>&state=<csrf-token>
   ```

4. **Token exchange** — the client's **backend** POSTs the code and the original `code_verifier` to Google's token endpoint:

   ```
   POST https://oauth2.googleapis.com/token
     grant_type=authorization_code
     code=<auth-code>
     code_verifier=<original-verifier>
     client_id=...
     redirect_uri=https://myapp.com/callback
   ```

5. **Tokens returned** — Google responds with an `access_token`, optional `id_token` (if the `openid` scope was requested — OIDC), and a `refresh_token` (if `access_type=offline` / `offline_access` was requested).

6. **Calling Google APIs** — the client uses the **Google-issued** `access_token` as a bearer token on every Google API call:

   ```
   GET https://www.googleapis.com/calendar/v3/calendars/primary/events
   Authorization: Bearer <access_token>
   ```

> PKCE (RFC 7636) prevents authorization code interception — even if an attacker steals the `code`, they can't exchange it without the original `code_verifier`. PKCE is now required for **all** clients (public and confidential).

## OIDC vs OAuth 2.0

| Aspect | OAuth 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| Purpose | **Authorization** — delegated API access | **Authentication** — identifying the user |
| Token | `access_token` (opaque or JWT) | `id_token` (always a JWT with user claims) |
| Answers | "Can this client call this API?" | "Who is the end user?" |
| Built on | — | OAuth 2.0 (it's a layer on top) |

Use **OIDC** when your app needs to **know who the user is** (sign-in). Use **OAuth 2.0 alone** when you only need delegated access to a resource API. In practice most "Sign in with Google/Microsoft/…" flows are OIDC (request the `openid` scope).

## Grant types (flows)

| Flow | Use for | Notes |
|---|---|---|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile, desktop | The default for any interactive user. PKCE mandatory. |
| **Client Credentials** | Service-to-service (no user) | Daemon/backend calling another API under its own identity. |
| **Device Code** | TVs, CLIs, IoT without a browser | User completes auth on a second device. |
| **Refresh Token** | Renewing access tokens | Paired with other flows; never used standalone. |
| ~~Implicit~~ | — | **Deprecated** (OAuth 2.1). Use Auth Code + PKCE. |
| ~~Resource Owner Password (ROPC)~~ | — | **Deprecated**. Client handles the user's password — avoid. |

## JWT (JSON Web Token)

JWT can be used to represent the access token. It is an open standard (RFC 7519) that defines a compact and self-contained way of transmitting information between parties as a JSON object.

```
Header.Payload.Signature
```

```csharp
// In ASP.NET Core, configure JWT authentication:
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidIssuer = "your-api",
            ValidAudience = "your-client",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SigningKey"]!))
        };
    });
```

> **Key management matters.** HS256 (symmetric) requires a key of at least **256 bits** (32 bytes) of real entropy — short passwords will be rejected by `Microsoft.IdentityModel` on recent versions. For any **distributed** system (multiple services validating tokens), prefer **asymmetric** algorithms like **RS256** or **ES256**: sign with a private key, distribute only the public key (typically via a JWKS endpoint). Never hardcode keys — load them from **configuration**, **environment variables**, or a secret store like **Azure Key Vault** / **AWS Secrets Manager**, and rotate them.

---

[← Previous: Service Lifetimes](02-service-lifetimes.md) | [Next: API Resilience →](04-api-resilience.md) | [Back to index](README.md)
