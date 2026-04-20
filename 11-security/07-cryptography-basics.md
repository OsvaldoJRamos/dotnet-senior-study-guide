# Cryptography Basics

You do not need to invent a cipher. You need to **pick the right one**, use it **correctly**, and recognize when someone else didn't. The senior bar is knowing which .NET class maps to which primitive and which RFC backs it.

## The golden rule

> **Don't roll your own crypto.** Use `System.Security.Cryptography`, `Microsoft.AspNetCore.Cryptography.KeyDerivation`, `Microsoft.AspNetCore.DataProtection`, or a vetted library (libsodium, BouncyCastle). Anything else and you will get it wrong in production.

## Password hashing (not encryption)

Passwords must be **hashed with a slow, salted, memory-hard function**. Plain `SHA256(password)` is broken — GPUs can try billions per second.

### Algorithms, ranked

| Algorithm | RFC | Status | When to use |
|---|---|---|---|
| **Argon2id** | RFC 9106 | Recommended by OWASP; winner of the 2015 PHC | New systems; requires FIPS constraints to consider otherwise |
| **bcrypt** | — (Provos/Mazières 1999) | Fine; lacks memory-hardness | Acceptable if Argon2 unavailable |
| **scrypt** | RFC 7914 | Fine; memory-hard | Rare in .NET |
| **PBKDF2** | RFC 8018 | Still acceptable; not memory-hard | Built into .NET; required for FIPS |

### PBKDF2 in .NET

`Microsoft.AspNetCore.Cryptography.KeyDerivation` gives you PBKDF2 directly; ASP.NET Core Identity's `PasswordHasher<TUser>` also uses it under the hood.

```csharp
using Microsoft.AspNetCore.Cryptography.KeyDerivation;
using System.Security.Cryptography;

// 128-bit salt
byte[] salt = RandomNumberGenerator.GetBytes(16);

byte[] hash = KeyDerivation.Pbkdf2(
    password: plainText,
    salt: salt,
    prf: KeyDerivationPrf.HMACSHA512,
    iterationCount: 100_000,
    numBytesRequested: 32); // 256-bit subkey
```

ASP.NET Core Identity's default `PasswordHasher<TUser>` (the `IdentityV3` format) uses **PBKDF2 with HMAC-SHA512, a 128-bit salt, a 256-bit subkey, and 100 000 iterations** per the source at `dotnet/aspnetcore` (`src/Identity/Extensions.Core/src/PasswordHasher.cs`). In most apps, just using `PasswordHasher<TUser>` is the right answer.

### Argon2id — the recommended default

RFC 9106: *"Argon2id MUST be supported by any implementation of this document"* and *"If a uniformly safe option that is not tailored to your application or hardware is acceptable, select Argon2id with t=1 iteration, p=4 lanes, m=2^(21) (2 GiB of RAM)."*

.NET has no first-party Argon2; use `Konscious.Security.Cryptography.Argon2` or libsodium bindings. Validate against the RFC test vectors.

## Symmetric encryption

For encrypting data at rest or in a message, use **AEAD** (authenticated encryption with associated data) — never plain CBC with a separate HMAC you bolt on yourself.

### AES-GCM in .NET

`System.Security.Cryptography.AesGcm`:

```csharp
using System.Security.Cryptography;

byte[] key = RandomNumberGenerator.GetBytes(32);         // 256-bit key
byte[] nonce = RandomNumberGenerator.GetBytes(12);       // 96-bit nonce, per NIST SP 800-38D
byte[] plaintext = Encoding.UTF8.GetBytes("hello world");
byte[] ciphertext = new byte[plaintext.Length];
byte[] tag = new byte[16];                               // 128-bit auth tag

using var aes = new AesGcm(key, tagSizeInBytes: 16);
aes.Encrypt(nonce, plaintext, ciphertext, tag);

// Decrypt — reuse the same nonce and tag
byte[] decrypted = new byte[ciphertext.Length];
aes.Decrypt(nonce, ciphertext, tag, decrypted);
```

> The parameterless `AesGcm(byte[])` and `AesGcm(ReadOnlySpan<byte>)` constructors were marked `[Obsolete]` with `DiagnosticId="SYSLIB0053"` starting in **.NET 8**. Always pass `tagSizeInBytes` explicitly so the code is still valid on newer runtimes.

Rules:

- Key size: AES-128 or AES-256. Most builds use 256.
- Nonce must be **unique per key per message** — never reuse. 96-bit random nonces are safe up to ~2^32 messages per key; rotate the key past that.
- Tag must be verified by `Decrypt`; don't use low-level APIs that skip it.

### When not to use `AesGcm`

For ASP.NET Core round-tripping trusted state (cookies, antiforgery tokens, bearer state), use the **Data Protection API**. It handles key rotation, algorithm agility, and multi-machine key ring sharing.

```csharp
public class PayloadService(IDataProtectionProvider provider)
{
    private readonly IDataProtector _protector =
        provider.CreateProtector("MyApp.Payloads.v1");

    public string Protect(string s)   => _protector.Protect(s);
    public string Unprotect(string s) => _protector.Unprotect(s);
}
```

From the Microsoft overview: *"The ASP.NET Core data protection stack was designed to: Provide a built in solution for most Web scenarios. Address many of the deficiencies of the previous encryption system. Serve as the replacement for the `<machineKey>` element in ASP.NET 1.x - 4.x."* The same doc notes the APIs *"aren't primarily intended for indefinite persistence of confidential payloads"* and points to Windows CNG DPAPI or Azure Rights Management for that scenario — for truly long-term key custody, reach for Key Vault / HSMs.

## Asymmetric (public-key) cryptography

Used for signing, key exchange, and trust establishment (TLS, JWT, OIDC).

| Algorithm | .NET class | Typical use |
|---|---|---|
| **RSA** | `RSA` (often via `RSACng`) | Legacy signing, RSA-OAEP key wrap, RSA-PSS signing; minimum 2048-bit modulus in 2026, 3072-bit preferred |
| **ECDSA** | `ECDsa` | Signatures — JWT ES256 (P-256), ES384 (P-384). Smaller keys, faster |
| **Ed25519** | Not first-party in `System.Security.Cryptography` as of .NET 10 — use `BouncyCastle.Cryptography` or libsodium bindings | Modern signature; fixed curve; no parameter-choice footguns |
| **ECDH** | `ECDiffieHellman` | Key agreement (TLS, E2EE) |
| **X25519** | via libsodium / BouncyCastle | Modern key agreement; fixed curve |

> Ed25519 and X25519 have no BER/DER parameter surprises and aren't vulnerable to the curve-parameter attacks that keep biting custom NIST-curve implementations. Prefer them where available.

### Sign / verify with ECDSA

```csharp
using System.Security.Cryptography;

using ECDsa ecdsa = ECDsa.Create(ECCurve.NamedCurves.nistP256);
byte[] data = Encoding.UTF8.GetBytes("message");

byte[] signature = ecdsa.SignData(data, HashAlgorithmName.SHA256);
bool ok = ecdsa.VerifyData(data, signature, HashAlgorithmName.SHA256);
```

Never MD5 or SHA-1 for signing — collisions are practical. SHA-256 minimum; SHA-384 for ES384.

## TLS 1.3 (RFC 8446)

RFC 8446 (August 2018, Proposed Standard) *"specifies version 1.3 of the Transport Layer Security (TLS) protocol"* and obsoletes TLS 1.2's RFC 5246.

Key differences from 1.2 worth knowing at a senior level:

- All handshake messages **after ServerHello are encrypted**.
- Key exchange **must provide forward secrecy** (ECDHE or DHE; static-RSA key exchange is removed).
- Cipher suites are stripped down — only AEAD ciphers (AES-GCM, ChaCha20-Poly1305).
- **0-RTT mode** allows clients to send data on the first flight with a pre-shared key (saves latency; replayable, so use carefully).
- Handshake is simplified; legacy messages like `ChangeCipherSpec` are removed.

In .NET, `SslStream` negotiates TLS 1.3 automatically on supported OSes (Windows 11 / Windows Server 2022+, recent Linux). For `HttpClient`, TLS 1.3 comes from the underlying platform.

## HTTPS and HSTS in ASP.NET Core

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);  // production — override the default
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();                // emit Strict-Transport-Security header
}
app.UseHttpsRedirection();        // 307 redirect HTTP -> HTTPS by default
```

Facts to memorize:

- **`UseHsts` default `max-age` is 30 days.** Shorter than most people expect; bump it for production.
- `UseHttpsRedirection` uses **HTTP 307 (Temporary Redirect)** by default.
- `IncludeSubDomains` + `Preload` are required for inclusion in the browser HSTS preload list (`hstspreload.org`).
- `UseHsts` only emits the header over HTTPS — it has no effect on the first HTTP request. Preload fixes that for subsequent users.
- Per the docs: **do not use `RequireHttpsAttribute` on Web APIs receiving sensitive data**; API clients may not follow redirects. Instead, bind the API to HTTPS only.

## Random numbers

For **cryptographic** use, always `RandomNumberGenerator` — never `System.Random`:

```csharp
byte[] token = RandomNumberGenerator.GetBytes(32);           // 256 bits
int withinRange = RandomNumberGenerator.GetInt32(1, 1_000);
```

`System.Random` is a pseudo-random source seeded from the system clock — predictable enough to be dangerous for tokens, IDs, or anything an attacker could guess.

## Constant-time comparison

A naive `a == b` on a byte array can leak timing information. For MAC / HMAC / hash comparisons, use:

```csharp
CryptographicOperations.FixedTimeEquals(expected, actual);
```

## .NET FIPS mode

Regulated environments (US government, banking) often require FIPS 140 validation. Per Microsoft's FIPS compliance doc, .NET Core *"Passes cryptographic primitives calls through to the standard modules the underlying operating system provides"* and *"Does not enforce the use of FIPS Approved algorithms or key sizes"* — the OS-level FIPS configuration is what restricts algorithms. Practical consequences:

- Cryptographic operations route through OS-provided validated modules on Windows (CNG) / Linux (OpenSSL in FIPS mode).
- In FIPS-configured operating systems, calls into non-approved algorithms such as `MD5` or certain cipher / curve combinations typically fail at the OS layer (observed as `CryptographicException`); the exact surface depends on the OS policy.
- Argon2 and scrypt aren't FIPS-validated; PBKDF2-HMAC-SHA256 / SHA512 is — which is why ASP.NET Core Identity defaults there.

## Anti-patterns

- `SHA256(password)` or `MD5(password)` — no salt, no work factor; cracks in seconds on a GPU.
- Encrypting with `Aes` + hand-rolled CBC and no MAC — classic padding oracle.
- Reusing an AES-GCM nonce across two messages with the same key — catastrophic (recovers the auth key and plaintext).
- Storing JWT signing keys in `appsettings.json`. Put them in Key Vault / Secrets Manager; load at startup.
- Writing your own certificate chain validation — use `SslStream` / `HttpClientHandler` defaults and only override with eyes wide open.
- MD5 or SHA-1 for signatures — collision attacks are practical.
- Long-lived static symmetric keys with no rotation plan. Rotate, or use the Data Protection API which rotates for you.
- Logging the ciphertext, IV, or key "for debugging." Treat them as secret.

---

[← Previous: Supply Chain Security](06-supply-chain-security.md) | [Back to index](README.md)
