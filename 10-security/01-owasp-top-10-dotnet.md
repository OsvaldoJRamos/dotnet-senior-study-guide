# OWASP Top 10 for .NET

The OWASP Top 10 is the baseline every web engineer is expected to know. `owasp.org/Top10/` now publishes the **2025** edition as the current release, but the **2021** list is still the one you will see cited in most production guidance, compliance tooling, and interview material — the categories below use the official `AXX:2021` codes. For each, the senior signal is naming the concrete .NET API or built-in mitigation, not repeating the OWASP description.

## The categories (2021)

| Code | Title |
|---|---|
| **A01:2021** | Broken Access Control |
| **A02:2021** | Cryptographic Failures |
| **A03:2021** | Injection |
| **A04:2021** | Insecure Design |
| **A05:2021** | Security Misconfiguration |
| **A06:2021** | Vulnerable and Outdated Components |
| **A07:2021** | Identification and Authentication Failures |
| **A08:2021** | Software and Data Integrity Failures |
| **A09:2021** | Security Logging and Monitoring Failures |
| **A10:2021** | Server-Side Request Forgery (SSRF) |

> The 2025 edition is now the released version on `owasp.org/Top10/`; most production guidance and interview questions still reference 2021. Verify which edition your target company uses before the interview.

## A01:2021 — Broken Access Control

Most common failure: authentication works but the server doesn't check **whether *this* user may access *this* resource** (IDOR — Insecure Direct Object Reference).

### .NET mitigation

Use **policy-based authorization** (not just roles). Check ownership in the handler, not only role membership.

```csharp
// Register the policy
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("OrderOwner", policy =>
        policy.Requirements.Add(new OrderOwnerRequirement()));
});

// Handler checks ownership against the resource
public class OrderOwnerHandler : AuthorizationHandler<OrderOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderOwnerRequirement requirement,
        Order resource)
    {
        if (resource.CustomerId == context.User.FindFirst("sub")?.Value)
            context.Succeed(requirement);
        return Task.CompletedTask;
    }
}

// Call from the endpoint
var result = await _authorizationService.AuthorizeAsync(User, order, "OrderOwner");
if (!result.Succeeded) return Forbid();
```

> Do NOT rely on hidden UI elements or obscure URLs — every endpoint must enforce access on the server.

## A02:2021 — Cryptographic Failures

Formerly "Sensitive Data Exposure". Covers missing encryption, weak algorithms (MD5, SHA-1 for passwords, DES, 3DES), hard-coded keys, and missing TLS.

### .NET mitigation

- **Passwords**: never MD5/SHA-1/SHA-256 alone. Use ASP.NET Core Identity's `PasswordHasher<TUser>` (PBKDF2) or a purpose-built hasher (Argon2id). Details in [Cryptography Basics](07-cryptography-basics.md).
- **Symmetric encryption**: prefer `AesGcm` (authenticated encryption) over raw `Aes` + HMAC.
- **Transport**: `app.UseHttpsRedirection()` + `app.UseHsts()`. The default `max-age` produced by `UseHsts` is **30 days**; set `AddHsts` with a longer value for production.
- **Trusted state (cookies, tokens)**: use the [ASP.NET Core Data Protection API](https://learn.microsoft.com/aspnet/core/security/data-protection/introduction) instead of home-grown AES.

```csharp
// Data Protection — built-in, correctly configured, rotates keys automatically
public class TokenService(IDataProtectionProvider provider)
{
    private readonly IDataProtector _protector =
        provider.CreateProtector("MyApp.Tokens.v1");

    public string Protect(string payload) => _protector.Protect(payload);
    public string Unprotect(string cipher) => _protector.Unprotect(cipher);
}
```

## A03:2021 — Injection

SQL injection, LDAP injection, OS command injection, XSS (now folded into Injection in 2021).

### .NET mitigation

- **SQL**: use parameters (`SqlParameter`, Dapper's `@name`, EF Core `FromSqlInterpolated`). Never concatenate user input.
- **LINQ over EF Core**: inherently parameterized.
- **XSS in Razor**: output is HTML-encoded by default. `@Html.Raw()` and `MarkupString` are opt-in escape hatches — review every use.
- **Process execution**: prefer `ProcessStartInfo.ArgumentList` (encodes each argument) over `Arguments` (single string; shell-escaping pitfalls).

```csharp
// WRONG — string interpolation into raw SQL
var users = db.Users.FromSqlRaw($"SELECT * FROM Users WHERE Email = '{email}'");

// CORRECT — FromSqlInterpolated parameterizes {email}
var users = db.Users.FromSqlInterpolated(
    $"SELECT * FROM Users WHERE Email = {email}");
```

## A04:2021 — Insecure Design

New in 2021. Covers design-level flaws that can't be fixed by a library — missing rate limits, missing business-logic checks, trust boundaries drawn in the wrong place.

### .NET mitigation

- Do [Threat Modeling](05-threat-modeling.md) during design, not after launch.
- Apply the **rate limiter middleware** (`AddRateLimiter`, `UseRateLimiter`) on login, signup, password reset, and expensive endpoints.
- Separate **read** and **write** paths; require stronger auth on writes.
- Define NFRs for abuse (max orders per customer per minute, max failed logins).

## A05:2021 — Security Misconfiguration

Default credentials, verbose error pages in production, unnecessary services exposed, missing security headers.

### .NET mitigation

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // ONLY in Development
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
```

- Strip server headers: `Kestrel.AddServerHeader = false`.
- Set Content-Security-Policy, X-Content-Type-Options, Referrer-Policy via a small middleware or a library like `NetEscapades.AspNetCore.SecurityHeaders`.
- Never ship with `ASPNETCORE_ENVIRONMENT=Development` in production — it changes the exception handler and disables HSTS.

## A06:2021 — Vulnerable and Outdated Components

Shipping a library with a known CVE. For .NET this means out-of-date NuGet packages, unpatched runtime, unmaintained transitive dependencies.

### .NET mitigation

- `dotnet package list --vulnerable --include-transitive` (noun-first form in .NET 10 SDK; `dotnet list package` in .NET 9 and earlier). The `--vulnerable` option is available starting in .NET SDK 9.0.300.
- Enable Dependabot on the repo (`package-ecosystem: "nuget"`).
- Stay on an LTS .NET version; keep a calendar reminder for the runtime EOL.
- Full treatment: [Supply Chain Security](06-supply-chain-security.md).

## A07:2021 — Identification and Authentication Failures

Credential stuffing, weak passwords, missing MFA, broken session management, exposed session IDs.

### .NET mitigation

- Prefer **ASP.NET Core Identity** over rolling your own — it handles hashing, lockout, and two-factor flow.
- Configure lockout: `IdentityOptions.Lockout.DefaultLockoutTimeSpan`, `MaxFailedAccessAttempts`.
- Use the **antiforgery** middleware for cookie-based auth (see A08).
- For APIs, prefer **PKCE** for public clients (SPAs, mobile). Details in [Authentication Patterns](02-authentication-patterns.md).

## A08:2021 — Software and Data Integrity Failures

Deserialization of untrusted data, unsigned update mechanisms, CI/CD compromise, CSRF (now lives here).

### .NET mitigation

- **CSRF**: ASP.NET Core's antiforgery middleware. Per the official doc, antiforgery services are **auto-registered** only when `AddMvc`, `MapRazorPages`, `MapControllerRoute`, or `AddRazorComponents` (ASP.NET Core 8+) is called; the `FormTagHelper` then injects the hidden `__RequestVerificationToken` field. `AddControllers()` alone does NOT enable antiforgery — use `AddControllersWithViews()` if you render forms. For Minimal APIs, call `builder.Services.AddAntiforgery()` and `app.UseAntiforgery()` explicitly.
- **Deserialization**: prefer `System.Text.Json` over `BinaryFormatter` (which is obsolete and removed in .NET 9+). Never deserialize untrusted input into polymorphic types without `TypeInfoResolver` / a type allow-list.
- **CI/CD**: sign your own NuGet packages (`dotnet nuget sign`) and verify consumed packages.

```csharp
// Minimal API — antiforgery is NOT automatic; opt in explicitly
builder.Services.AddAntiforgery();
app.UseAntiforgery();
```

## A09:2021 — Security Logging and Monitoring Failures

The attack happened; you didn't notice. Missing auth-failure logs, no alerts on spikes, logs stored where the attacker has access.

### .NET mitigation

- Log auth events at `Information` or higher, auth failures at `Warning` with **structured properties** (user id, IP, correlation id) — not interpolated strings.
- Ship logs off-box (Seq, Application Insights, CloudWatch, Datadog). A root-owned local file is not enough.
- **Never log secrets, tokens, passwords, or PII**. Use a log scrubber if in doubt.
- Full treatment: the [Observability](../14-observability/README.md) section.

## A10:2021 — Server-Side Request Forgery (SSRF)

Your server fetches a URL an attacker controls (`https://api.myapp.com/fetch?url=http://169.254.169.254/...`), exposing cloud metadata endpoints or internal services.

### .NET mitigation

- **Allow-list** host names; do not accept arbitrary `HttpClient` targets from user input.
- Resolve the host with `Dns.GetHostAddressesAsync` and reject private / link-local ranges (`169.254.0.0/16`, `10.0.0.0/8`, `127.0.0.0/8`) — re-check after redirects.
- Disable auto-redirects (`HttpClientHandler.AllowAutoRedirect = false`) and validate each redirect target.
- On cloud hosts, require **IMDSv2** (AWS) or put the instance behind Azure Policy that blocks metadata access.

```csharp
// Rough sketch — reject non-allowlisted hosts
private static readonly HashSet<string> AllowedHosts = new(["api.partner.com"]);

public async Task<string> FetchAsync(Uri target)
{
    if (!AllowedHosts.Contains(target.Host))
        throw new InvalidOperationException("Host not allowed");
    return await _httpClient.GetStringAsync(target);
}
```

## Rule of thumb

A senior interviewer listening to OWASP answers wants three things: the **name of the category**, the **concrete .NET API** that mitigates it, and the **class of attack** you've personally seen cause real damage. If you can do that for all ten, you've cleared the bar.

---

[Next: Authentication Patterns →](02-authentication-patterns.md) | [Back to index](README.md)
