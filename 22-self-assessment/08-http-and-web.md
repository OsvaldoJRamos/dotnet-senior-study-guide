# HTTP and Web

> Read the questions, think about your answer, then click to reveal.

---

### 1. Which HTTP methods are idempotent, and why does it matter?

<details>
<summary>Reveal answer</summary>

**Idempotent methods**: `GET`, `PUT`, `DELETE`, `HEAD`, `OPTIONS`. Calling them multiple times produces the same result as calling once.

**Not idempotent**: `POST`, `PATCH` (in general).

It matters because:
- **Retries are safe** for idempotent methods -- network failures don't cause duplicates.
- **Caches and proxies** rely on idempotency semantics.
- `PUT` replaces the entire resource (idempotent); `PATCH` applies a partial update (may not be idempotent depending on the operation).

Deep dive: [HTTP Semantics](../08-http-and-web/01-http-semantics.md)

</details>

---

### 2. What is the difference between "safe" and "idempotent" HTTP methods?

<details>
<summary>Reveal answer</summary>

They overlap but are not the same (RFC 9110).

- **Safe** — the method is read-only; it does not change server state. Safe methods are `GET`, `HEAD`, `OPTIONS`, `TRACE`. By definition, every safe method is also idempotent.
- **Idempotent** — calling the method N times has the same effect as calling it once. Idempotent methods include all safe methods plus `PUT` and `DELETE`.

Example: `DELETE /orders/42` is idempotent (after the first call, the resource is gone; subsequent calls still leave it gone), but it is **not** safe — it changes state.

Deep dive: [HTTP Semantics](../08-http-and-web/01-http-semantics.md)

</details>

---

### 3. How should you design REST API URLs? Give examples of good and bad patterns.

<details>
<summary>Reveal answer</summary>

**Good patterns** (nouns, plural, hierarchical):
- `GET /api/orders` -- list orders
- `GET /api/orders/42` -- single order
- `GET /api/orders/42/items` -- items within an order
- `POST /api/orders` -- create an order

**Bad patterns**:
- `GET /api/getOrders` -- verb in URL (redundant with HTTP method)
- `POST /api/orders/create` -- action in URL
- `GET /api/order` -- inconsistent singular/plural

Use **kebab-case** for multi-word resources (`/api/order-items`), and keep URLs **predictable and consistent**.

Deep dive: [REST API Design](../08-http-and-web/03-rest-api-design.md)

</details>

---

### 4. What HTTP status codes should a well-designed API return, and when?

<details>
<summary>Reveal answer</summary>

| Code | Meaning | When to use |
|------|---------|-------------|
| **200** | OK | Successful GET, PUT, PATCH |
| **201** | Created | Successful POST that creates a resource |
| **204** | No Content | Successful DELETE (nothing to return) |
| **400** | Bad Request | Invalid input / validation failure |
| **401** | Unauthorized | Missing or invalid authentication |
| **403** | Forbidden | Authenticated but not authorized |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Duplicate or state conflict |
| **422** | Unprocessable Entity | Well-formed request that fails domain validation |
| **429** | Too Many Requests | Rate limit exceeded |
| **500** | Internal Server Error | Unhandled server-side exception |
| **503** | Service Unavailable | Server overloaded or down for maintenance |

Deep dive: [REST API Design](../08-http-and-web/03-rest-api-design.md)

</details>

---

### 5. What are common approaches to API pagination and versioning?

<details>
<summary>Reveal answer</summary>

**Pagination**:
- **Offset-based**: `?page=2&pageSize=20` -- simple but slow for large offsets (DB must skip rows).
- **Cursor-based**: `?cursor=abc123&limit=20` -- uses an opaque token pointing to the last item. Fast and consistent, best for feeds.

**Versioning**:
- **URL segment**: `/api/v2/orders` -- most common, explicit.
- **Header**: `Api-Version: 2` -- clean URLs, harder to test in a browser.
- **Query string**: `?api-version=2` -- easy to use, less RESTful.

Deep dive: [REST API Design](../08-http-and-web/03-rest-api-design.md)

</details>

---

### 6. What is RFC 7807 Problem Details, and why use it?

<details>
<summary>Reveal answer</summary>

**RFC 7807** defines a standard JSON (or XML) format for machine-readable error responses:

```json
{
  "type": "https://example.com/probs/out-of-credit",
  "title": "You do not have enough credit.",
  "status": 403,
  "detail": "Your current balance is 30, but that costs 50.",
  "instance": "/account/12345/msgs/abc"
}
```

Why use it:
- **Consistent error shape** across your APIs — clients always know where to look.
- **Extensible** — add custom fields alongside the standard ones.
- **Built into ASP.NET Core** — `AddProblemDetails()` and `Results.Problem(...)` emit it automatically for 4xx/5xx.
- Pairs well with Minimal APIs and the `IProblemDetailsService` abstraction.

Deep dive: [REST API Design](../08-http-and-web/03-rest-api-design.md)

</details>

---

### 7. What is CORS, why does it exist, and how do you configure it in ASP.NET Core?

<details>
<summary>Reveal answer</summary>

**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism that blocks requests from a different origin (scheme + host + port) unless the server explicitly allows it. It prevents malicious sites from making API calls on behalf of a user.

Configuration in ASP.NET Core:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("https://myapp.com")
              .AllowAnyHeader()
              .AllowAnyMethod());
});

app.UseCors("AllowFrontend");
```

**Never use `AllowAnyOrigin()` with `AllowCredentials()`** -- the browser will reject it, and it's a security risk.

Deep dive: [Web Security](../08-http-and-web/04-web-security.md)

</details>

---

### 8. What is a CORS preflight request, and when does it happen?

<details>
<summary>Reveal answer</summary>

When a cross-origin request is **not a "simple request"**, the browser first sends an `OPTIONS` preflight with `Access-Control-Request-Method` and `Access-Control-Request-Headers` to ask the server whether the actual request is allowed. Only if the server responds with matching `Access-Control-Allow-*` headers does the browser send the real request.

A request is "simple" (no preflight) only when it uses `GET`/`HEAD`/`POST` with a restricted content type (`text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data`) and no custom headers.

Any `PUT`/`DELETE`/`PATCH`, any `Content-Type: application/json`, and any custom header (e.g. `Authorization`, `X-Correlation-Id`) trigger a preflight.

Tip: cache preflights with `Access-Control-Max-Age` to avoid the extra round-trip on every request.

Deep dive: [Web Security](../08-http-and-web/04-web-security.md)

</details>

---

### 9. What are the three parts of a JWT, and what does each contain?

<details>
<summary>Reveal answer</summary>

A JWT has three Base64URL-encoded parts separated by dots: `header.payload.signature`

1. **Header** -- algorithm (`alg`: HS256, RS256) and token type (`typ`: JWT).
2. **Payload** -- claims: `sub` (subject), `exp` (expiration), `iss` (issuer), `aud` (audience), plus custom claims.
3. **Signature** -- `HMACSHA256(base64(header) + "." + base64(payload), secret)` -- proves the token wasn't tampered with. (For HS256 (symmetric); RS256/ES256 use RSA/ECDSA signatures over the same input. Asymmetric is preferred for distributed services so verifiers don't need the signing key.)

**Important**: the payload is **encoded, not encrypted**. Anyone can read it. Never put secrets in a JWT.

Deep dive: [Web Security](../08-http-and-web/04-web-security.md)

</details>

---

### 10. What is the difference between HTTPS and TLS? Why is HTTPS mandatory for modern APIs?

<details>
<summary>Reveal answer</summary>

**TLS (Transport Layer Security)** is the cryptographic protocol that provides encryption, integrity, and authentication. **HTTPS** is simply HTTP running over TLS.

HTTPS is mandatory because:
- **Encryption** -- prevents eavesdropping on sensitive data (passwords, tokens, PII).
- **Integrity** -- prevents man-in-the-middle tampering.
- **Authentication** -- the server's certificate proves its identity.
- **Browser requirements** -- modern browsers flag HTTP sites as "Not Secure."
- **HTTP/2 and HTTP/3** require TLS in practice.

Deep dive: [Web Security](../08-http-and-web/04-web-security.md)

</details>

---

### 11. What is HSTS, and why is it important?

<details>
<summary>Reveal answer</summary>

**HSTS (HTTP Strict Transport Security)** is a response header (`Strict-Transport-Security: max-age=31536000; includeSubDomains`) that tells browsers to always connect to the site over HTTPS for a given period, even if the user types `http://`.

It defends against **SSL-stripping man-in-the-middle attacks**, where an attacker downgrades the initial HTTP request before the redirect to HTTPS happens.

In ASP.NET Core, `app.UseHsts()` enables it in non-Development environments. The `preload` directive plus submission to the browser HSTS preload list bakes your domain into the browser so even the first request is HTTPS.

Deep dive: [Web Security](../08-http-and-web/04-web-security.md)

</details>

---

### 12. What is the difference between `Cache-Control: no-cache` and `no-store`?

<details>
<summary>Reveal answer</summary>

They look similar but mean different things (RFC 9111):

- **`no-cache`** — caches **may store** the response, but must **revalidate** with the origin before serving it (using `ETag`/`If-None-Match` or `Last-Modified`/`If-Modified-Since`). The response is reusable if the server returns `304 Not Modified`.
- **`no-store`** — caches **must not store** the response at all. Every request goes to the origin. Use for sensitive data (banking statements, health records).

Other common directives:
- `private` — only browsers may cache (not shared CDNs/proxies).
- `public` — shared caches may store.
- `max-age=N` — response is fresh for N seconds.
- `must-revalidate` — once stale, a cache must revalidate before reusing.

Deep dive: [HTTP Semantics](../08-http-and-web/01-http-semantics.md)

</details>

---

### 13. What is the difference between `ETag` and `Last-Modified` for conditional requests?

<details>
<summary>Reveal answer</summary>

Both enable **304 Not Modified** responses, saving bandwidth.

| Header | What it is | Client sends back with |
|--------|-----------|-----------------------|
| `ETag` | Opaque fingerprint of the representation (e.g., hash) | `If-None-Match` |
| `Last-Modified` | Timestamp of last change | `If-Modified-Since` |

`ETag` is more precise (detects content changes even within the same second) but requires computing/storing the tag. `Last-Modified` is cheaper but has 1-second granularity and can miss rapid updates.

ETags also support **optimistic concurrency** for writes via `If-Match` — the server rejects a `PUT` with 412 Precondition Failed if the ETag the client has is stale.

Deep dive: [HTTP Semantics](../08-http-and-web/01-http-semantics.md)

</details>

---

### 14. What's new in HTTP/2 and HTTP/3?

<details>
<summary>Reveal answer</summary>

**HTTP/2** — originally RFC 7540 (2015), revised and obsoleted by **RFC 9113 (June 2022)**:
- **Binary** framing instead of plain text.
- **Multiplexing** — multiple requests/responses on a single TCP connection (no more head-of-line blocking at the HTTP layer).
- **Header compression** (HPACK).
- **Server push** (rarely used, mostly deprecated by browsers).

**HTTP/3** — RFC 9114 (June 2022):
- Runs on **QUIC** over UDP instead of TCP.
- Eliminates TCP head-of-line blocking (each stream has its own loss recovery).
- **0-RTT** connection resumption for return visits.
- Built-in TLS 1.3 — no separate TLS handshake.

Both require TLS in practice. Performance wins are biggest on lossy/mobile networks (HTTP/3) and on pages that fetch many small resources over one domain (HTTP/2).

Deep dive: [HTTP Semantics](../08-http-and-web/01-http-semantics.md)

</details>

---

### 15. A developer says "our API returns 200 for everything and puts the error in the response body." What's wrong with this approach?

<details>
<summary>Reveal answer</summary>

This violates HTTP semantics and causes multiple problems:
- **Proxies, CDNs, and caches** rely on status codes -- a 200 with an error body gets cached as a success.
- **Monitoring tools** count 4xx/5xx for alerting -- all errors become invisible.
- **Client libraries** (HttpClient, Axios) use status codes to branch success/failure logic.
- **API consumers** must parse every response body to check for errors instead of checking the status code.

Use **proper status codes** (400, 404, 409, 500) and a consistent error body format (like RFC 7807 Problem Details).

Deep dive: [REST API Design](../08-http-and-web/03-rest-api-design.md)

</details>

---

### 16. What are the main MIME types you should know for APIs?

<details>
<summary>Reveal answer</summary>

| MIME type | Used for |
|-----------|----------|
| `application/json` | JSON request/response bodies |
| `application/problem+json` | RFC 7807 error responses |
| `application/xml` | XML payloads (SOAP, legacy APIs) |
| `application/x-www-form-urlencoded` | HTML form submissions, OAuth token endpoints |
| `multipart/form-data` | File uploads |
| `application/octet-stream` | Arbitrary binary data |
| `text/event-stream` | Server-Sent Events (SSE) |

Clients use the **`Accept` header** to negotiate which representation they want; servers echo the chosen type in **`Content-Type`**. Returning the wrong `Content-Type` breaks parsers downstream.

Deep dive: [MIME Types](../08-http-and-web/02-mime-types.md)

</details>

---
