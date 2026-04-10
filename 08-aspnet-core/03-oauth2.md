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

## Practical example

Let's say you want to build a calendar application that uses Google Calendar:

1. The third-party application obtains an access token from Google, using OAuth 2.0 to authenticate with Google
2. Google issues an access token to the application
3. The application takes that access token and generates its own JWT
4. The application sends the JWT to Google, allowing Google to validate the JWT and grant access to the application

When we log in with Google on a third-party site, when a third-party site can manipulate Google Calendar, all of this is done through an access token. This way, the user's password and login are not needed.

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
                Encoding.UTF8.GetBytes("your-secret-key"))
        };
    });
```

---

[← Previous: Service Lifetimes](02-service-lifetimes.md) | [Next: API Resilience →](04-api-resilience.md) | [Back to index](README.md)
