# Security

> Read the questions, think about your answer, then click to reveal.

---

### 1. Name the OWASP Top 10 2021 categories from memory. For any three, give the concrete .NET mitigation you'd reach for.

<details>
<summary>Reveal answer</summary>

The 2021 list:

- **A01** Broken Access Control
- **A02** Cryptographic Failures
- **A03** Injection
- **A04** Insecure Design
- **A05** Security Misconfiguration
- **A06** Vulnerable and Outdated Components
- **A07** Identification and Authentication Failures
- **A08** Software and Data Integrity Failures
- **A09** Security Logging and Monitoring Failures
- **A10** Server-Side Request Forgery (SSRF)

Example mitigations:

- **A01** — resource-based `AuthorizeAsync(user, resource, "CanEditOrder")` with an `AuthorizationHandler<TRequirement, Order>` so the server enforces ownership per request.
- **A03** — EF Core `FromSqlInterpolated($"... WHERE Email = {email}")`; never `FromSqlRaw` with string concatenation.
- **A06** — `dotnet package list --vulnerable --include-transitive` (requires .NET SDK 9.0.300+), plus Dependabot `package-ecosystem: nuget` on the repo.

Deep dive: [OWASP Top 10 for .NET](../10-security/01-owasp-top-10-dotnet.md)

</details>

---

### 2. When should you pick cookies vs JWT bearer tokens vs OIDC?

<details>
<summary>Reveal answer</summary>

- **Cookies** — server-rendered apps (MVC, Razor Pages). Server controls the session; logout is trivial; antiforgery handles CSRF. Scales with either sticky sessions or Data Protection keys shared across nodes.
- **JWT bearer** — APIs, service-to-service, SPAs via a BFF. Stateless validation; no session store needed. Revocation is the hard part — keep access tokens short (5–15 min) and rotate refresh tokens.
- **OIDC** — any time you integrate with an external identity provider or need SSO across multiple apps. Built on top of OAuth 2.0; adds an ID Token (JWT) that actually identifies the user.

A common senior answer combines them: *"We run cookie auth on the BFF, exchange for a JWT to call internal services, and federate login via OIDC against the corporate IdP."*

Deep dive: [Authentication Patterns](../10-security/02-authentication-patterns.md)

</details>

---

### 3. Walk me through the OAuth 2.0 authorization code flow with PKCE. Why does PKCE exist?

<details>
<summary>Reveal answer</summary>

Per RFC 7636:

1. Client generates a cryptographically random `code_verifier` (43–128 chars).
2. Client computes `code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))` and sends it (with `code_challenge_method=S256`) to `/authorize`.
3. User authenticates; the authorization server redirects back with a one-time `code`.
4. Client POSTs the `code` **plus the raw `code_verifier`** to `/token`.
5. Server checks `BASE64URL(SHA256(code_verifier)) == code_challenge`. If yes, issues tokens.

PKCE exists because public clients (SPAs, mobile, desktop) can't keep a `client_secret`. If an attacker intercepts the authorization code (through OS URL handler hijacking, or a log), they still can't redeem it without the verifier — which never left the client. OAuth 2.1 recommends PKCE for **all** clients, including confidential ones.

Deep dive: [Authentication Patterns](../10-security/02-authentication-patterns.md)

</details>

---

### 4. In ASP.NET Core, what's the difference between `[Authorize(Roles = "...")]` and policy-based authorization? When does each shine?

<details>
<summary>Reveal answer</summary>

- `[Authorize(Roles = "Admin, Manager")]` — declarative, fast, good for coarse-grained checks. Comma-separated = OR. Stacking two `[Authorize]` attributes = AND.
- **Policy-based** — a policy is one or more `IAuthorizationRequirement`s, each evaluated by one or more `AuthorizationHandler<T>` (or `IAuthorizationHandler`). Multiple handlers on the same requirement = OR; multiple requirements in a policy = AND.

Policies win when:
- The rule depends on more than one claim, attribute, or the resource itself (resource-based authorization for IDOR prevention).
- The rule needs DI services (time, database, feature flag).
- You want the rule reusable and testable.

Use `AuthorizationPolicyBuilder` shortcuts (`RequireRole`, `RequireClaim`, `RequireAssertion`) for one-off rules that won't grow.

Deep dive: [Authorization](../10-security/03-authorization.md)

</details>

---

### 5. A dev wants to store the production DB password in `appsettings.json` "just temporarily while we debug." What do you say?

<details>
<summary>Reveal answer</summary>

No. Three problems:

1. `appsettings.json` is checked into source control — a leak is permanent in git history.
2. The file ships with the app — any container registry pull or backup becomes a credential leak.
3. "Temporarily" lasts six months. Every incident review I've seen started with a file labeled *temporary*.

The alternatives by environment:

- **Local dev** — `dotnet user-secrets set "ConnectionStrings:Default" "..."`. Stored in the user profile outside the repo.
- **Prod on Azure** — Azure Key Vault + Managed Identity; read via the Key Vault configuration provider so `builder.Configuration["ConnectionStrings:Default"]` just works.
- **Prod on AWS** — Secrets Manager + IAM role; rotate automatically with the alternating-users pattern for RDS.

And if the secret already landed in git: **rotate it immediately**, don't debate whether the repo is private.

Deep dive: [Secrets Management](../10-security/04-secrets-management.md)

</details>

---

### 6. What does STRIDE stand for and how do you use it in a session?

<details>
<summary>Reveal answer</summary>

Microsoft's threat taxonomy:

- **S**poofing — authentication violated (stolen cred, forged token).
- **T**ampering — integrity violated (modifying data, SQL injection).
- **R**epudiation — non-repudiation violated (no audit trail).
- **I**nformation disclosure — confidentiality violated (stack trace, cloud metadata endpoint).
- **D**enial of service — availability violated (unbounded query, unthrottled login).
- **E**levation of privilege — authorization violated (IDOR, broken access control).

Running a session:

1. Draw the DFD. Mark trust boundaries (dashed lines).
2. Walk each element and each boundary-crossing arrow. Apply the subset of STRIDE that applies to that element type (STRIDE-per-element).
3. For each identified threat, capture *existing control*, *gap*, *owner*. Prioritize with DREAD or a simple Likelihood×Impact matrix.
4. Output is **tickets**, not just the diagram. A beautiful TMT file with no follow-up is waste.

The Microsoft Threat Modeling Tool auto-enumerates STRIDE per element; OWASP Threat Dragon is a cross-platform alternative.

Deep dive: [Threat Modeling](../10-security/05-threat-modeling.md)

</details>

---

### 7. How does CSRF protection work in ASP.NET Core? When do you need to opt in explicitly?

<details>
<summary>Reveal answer</summary>

ASP.NET Core uses the Synchronizer Token Pattern — the server issues a unique hidden token (field name `__RequestVerificationToken`) that the client echoes back with every unsafe request.

- **MVC / Razor Pages** — `AddControllersWithViews()` and `AddRazorPages()` register antiforgery automatically; the `FormTagHelper` injects the token into any `<form method="post">`. `[AutoValidateAntiforgeryToken]` validates on POST/PUT/PATCH/DELETE only. For strict validation on every request (including GETs that mutate), use `[ValidateAntiForgeryToken]`.
- **Minimal APIs** — antiforgery is **not** automatic. You must call `builder.Services.AddAntiforgery()` and `app.UseAntiforgery()`.
- **Plain `AddControllers()`** — does NOT enable antiforgery by itself. Use `AddControllersWithViews()` if you render forms.

Tokens are encrypted with the ASP.NET Core Data Protection API, which must be configured with a shared key ring for multi-instance deployments — otherwise tokens from node A look invalid on node B and users see random `400 Bad Request` failures.

OWASP A08:2021 is where CSRF lives in the 2021 taxonomy.

Deep dive: [OWASP Top 10 for .NET](../10-security/01-owasp-top-10-dotnet.md)

</details>

---

### 8. Explain SBOM. What's the difference between SPDX and CycloneDX?

<details>
<summary>Reveal answer</summary>

An **SBOM (Software Bill of Materials)** is the inventory of components in a build — names, versions, licenses, relationships. It's the artifact you reach for during an incident (*"are we affected by CVE-X?"*) and is increasingly a regulatory requirement (US EO 14028, EU CRA).

Two standards interviewers expect you to name:

- **SPDX** — Linux Foundation; standardized as **ISO/IEC 5962:2021**. Strong adoption in enterprise / legal / license-compliance contexts.
- **CycloneDX** — OWASP Foundation + Ecma International (TC54); standardized as **ECMA-424**. Security-focused, with extensions for SaaSBOM, VEX (Vulnerability Exploitability Exchange), CBOM (cryptography), HBOM (hardware), AI/ML-BOM.

For .NET: generate with the community `dotnet CycloneDX` tool (CycloneDX) or Microsoft's `sbom-tool` (SPDX). Attach the SBOM to the release artifact — an SBOM that only lives on your laptop has no value.

Deep dive: [Supply Chain Security](../10-security/06-supply-chain-security.md)

</details>

---

### 9. Someone asks you to "just SHA-256 the password before storing it." What do you push back with?

<details>
<summary>Reveal answer</summary>

SHA-256 is a fast general-purpose hash. A single modern GPU tries billions of passwords per second against it. It's not designed for passwords.

Password hashing needs four properties:

1. **Salt** — unique per user, prevents rainbow-table and cross-account precomputation.
2. **Slow / work factor** — tunable so we can make attackers spend real time per guess.
3. **Memory-hardness** — forces attackers to spend RAM per guess, which GPUs and ASICs lack.
4. **Constant-time verify** — via `CryptographicOperations.FixedTimeEquals`.

The options:

- **Argon2id** — recommended by OWASP and by RFC 9106 *"if a uniformly safe option that is not tailored to your application or hardware is acceptable."* Memory-hard.
- **bcrypt**, **scrypt** — fine; bcrypt isn't memory-hard, scrypt is.
- **PBKDF2** — built into .NET, FIPS-approved, not memory-hard. Acceptable especially when regulatory constraints forbid Argon2.

Don't write it yourself. ASP.NET Core Identity's `PasswordHasher<TUser>` uses PBKDF2-HMAC-SHA512, a 128-bit salt, a 256-bit subkey, and 100 000 iterations by default (per the source at `dotnet/aspnetcore/src/Identity/Extensions.Core/src/PasswordHasher.cs`) — using it is the right answer for 95% of apps.

Deep dive: [Cryptography Basics](../10-security/07-cryptography-basics.md)

</details>

---

### 10. What's wrong with this call? `aes.Encrypt(nonce: new byte[12], plaintext, ciphertext, tag);`

<details>
<summary>Reveal answer</summary>

The nonce is all-zeros (a fresh `byte[12]` default-initialized). That's already bad — an attacker who sees two ciphertexts under the same key with the same nonce can recover the plaintext and the authentication key. In AES-GCM, nonce reuse is catastrophic.

Rules:

- Generate the nonce with `RandomNumberGenerator.GetBytes(12)` (NIST SP 800-38D recommends 96-bit random nonces).
- A single key + nonce pair must **never** encrypt two different messages. Rotate the key well before 2^32 messages per key.
- Store or transmit the nonce alongside the ciphertext — it isn't secret, but the receiver needs it to decrypt.
- Verify the tag on decrypt (`AesGcm.Decrypt` does this and throws `AuthenticationTagMismatchException` if it fails — never catch-and-ignore).

Also worth flagging: the parameterless `AesGcm(byte[])` and `AesGcm(ReadOnlySpan<byte>)` constructors are **obsolete** in .NET 8+. Use `new AesGcm(key, tagSizeInBytes: 16)` so the code keeps working on new runtimes.

Deep dive: [Cryptography Basics](../10-security/07-cryptography-basics.md)

</details>

---

### 11. What's the default `max-age` on `UseHsts()` in ASP.NET Core? Is that enough for production?

<details>
<summary>Reveal answer</summary>

The default `max-age` emitted by `UseHsts()` is **30 days**. That's shorter than most developers assume and usually **not enough** for production — the HSTS preload list requires `max-age` of at least one year.

Override it explicitly:

```csharp
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});
```

Related facts worth knowing:

- `UseHttpsRedirection()` uses **HTTP 307 Temporary Redirect** by default.
- HSTS has no effect on the **first** HTTP request a browser makes — preload via `hstspreload.org` fixes this for subsequent users but requires `preload`, `includeSubDomains`, and `max-age >= 1 year`.
- Microsoft explicitly warns **not** to use `[RequireHttps]` on Web APIs that receive sensitive data — API clients may not follow HTTP→HTTPS redirects, so the sensitive body goes over HTTP before the redirect arrives. Bind the API to HTTPS only instead.

Deep dive: [Cryptography Basics](../10-security/07-cryptography-basics.md)

</details>

---

### 12. What is Sigstore and why does it matter for supply-chain security?

<details>
<summary>Reveal answer</summary>

Sigstore is an open-source, Linux Foundation / CNCF project for **keyless signing** of software artifacts (container images, binaries, SBOMs). Three components:

- **Cosign** — the client tool developers / CI systems run to sign and verify.
- **Fulcio** — a code-signing CA that issues **short-lived certificates** bound to an OIDC identity (GitHub, Google, Microsoft).
- **Rekor** — an append-only, publicly auditable **transparency log** of every signing event.

Why it matters: traditional code signing needs long-lived private keys that must be stored, rotated, and protected — often badly. Sigstore replaces that with **ephemeral keys + verifiable OIDC identity + public transparency log**. A GitHub Actions workflow authenticates as itself via OIDC, gets a one-minute cert from Fulcio, signs the artifact, records the event in Rekor. There's nothing long-lived to steal.

For .NET, Sigstore's strongest current integration is around **container images** (via `cosign sign`) and admission policies in Kubernetes. NuGet package signing still uses classic X.509 certificates (`dotnet nuget sign`), and the author-signing UX predates Sigstore — NuGet-specific Sigstore integration is on the NuGet team's backlog but not first-class as of .NET 10.

Deep dive: [Supply Chain Security](../10-security/06-supply-chain-security.md)

</details>

---

[Back to index](README.md)
