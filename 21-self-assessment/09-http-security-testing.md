# HTTP, Security, and Testing

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

Deep dive: [HTTP Semantics](../07-http-and-web/01-http-semantics.md)

</details>

---

### 2. How should you design REST API URLs? Give examples of good and bad patterns.

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

Deep dive: [REST API Design](../07-http-and-web/03-rest-api-design.md)

</details>

---

### 3. What HTTP status codes should a well-designed API return, and when?

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
| **429** | Too Many Requests | Rate limit exceeded |
| **500** | Internal Server Error | Unhandled server-side exception |

Deep dive: [REST API Design](../07-http-and-web/03-rest-api-design.md)

</details>

---

### 4. What are common approaches to API pagination and versioning?

<details>
<summary>Reveal answer</summary>

**Pagination**:
- **Offset-based**: `?page=2&pageSize=20` -- simple but slow for large offsets (DB must skip rows).
- **Cursor-based**: `?cursor=abc123&limit=20` -- uses an opaque token pointing to the last item. Fast and consistent, best for feeds.

**Versioning**:
- **URL segment**: `/api/v2/orders` -- most common, explicit.
- **Header**: `Api-Version: 2` -- clean URLs, harder to test in a browser.
- **Query string**: `?api-version=2` -- easy to use, less RESTful.

Deep dive: [REST API Design](../07-http-and-web/03-rest-api-design.md)

</details>

---

### 5. What is CORS, why does it exist, and how do you configure it in ASP.NET Core?

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

Deep dive: [Web Security](../07-http-and-web/04-web-security.md)

</details>

---

### 6. What are the three parts of a JWT, and what does each contain?

<details>
<summary>Reveal answer</summary>

A JWT has three Base64URL-encoded parts separated by dots: `header.payload.signature`

1. **Header** -- algorithm (`alg`: HS256, RS256) and token type (`typ`: JWT).
2. **Payload** -- claims: `sub` (subject), `exp` (expiration), `iss` (issuer), `aud` (audience), plus custom claims.
3. **Signature** -- `HMACSHA256(base64(header) + "." + base64(payload), secret)` -- proves the token wasn't tampered with. (For HS256 (symmetric); RS256/ES256 use RSA/ECDSA signatures over the same input. Asymmetric is preferred for distributed services so verifiers don't need the signing key.)

**Important**: the payload is **encoded, not encrypted**. Anyone can read it. Never put secrets in a JWT.

Deep dive: [Web Security](../07-http-and-web/04-web-security.md)

</details>

---

### 7. How do refresh tokens work, and why not just use a long-lived access token?

<details>
<summary>Reveal answer</summary>

A long-lived access token is risky: if stolen, the attacker has extended access and JWTs can't be revoked (they're stateless).

**Refresh token flow**:
1. User authenticates -> gets a short-lived access token (5-15 min) + refresh token (days/weeks).
2. Access token expires -> client sends refresh token to the auth server.
3. Auth server validates, rotates the refresh token, and issues a new access token.

Benefits: **access tokens expire quickly** (limited damage if stolen), and **refresh tokens can be revoked** server-side (stored in DB).

Deep dive: [OAuth 2.0](../08-aspnet-core/03-oauth2.md)

</details>

---

### 8. Name three vulnerabilities from the OWASP Top 10 and how to prevent them.

<details>
<summary>Reveal answer</summary>

1. **SQL Injection** -- attacker injects SQL via user input. **Prevention**: always use parameterized queries or ORM; never concatenate user input into SQL.

2. **XSS (Cross-Site Scripting)** -- attacker injects malicious JavaScript. **Prevention**: encode output (Razor does this by default), use Content Security Policy headers, sanitize user-generated HTML.

3. **CSRF (Cross-Site Request Forgery)** -- attacker tricks a user's browser into making unwanted requests. **Prevention**: anti-forgery tokens (`[ValidateAntiForgeryToken]`), SameSite cookies, require custom headers for API calls.

Deep dive: [Web Security](../07-http-and-web/04-web-security.md)

</details>

---

### 9. What is the difference between HTTPS and TLS? Why is HTTPS mandatory for modern APIs?

<details>
<summary>Reveal answer</summary>

**TLS (Transport Layer Security)** is the cryptographic protocol that provides encryption, integrity, and authentication. **HTTPS** is simply HTTP running over TLS.

HTTPS is mandatory because:
- **Encryption** -- prevents eavesdropping on sensitive data (passwords, tokens, PII).
- **Integrity** -- prevents man-in-the-middle tampering.
- **Authentication** -- the server's certificate proves its identity.
- **Browser requirements** -- modern browsers flag HTTP sites as "Not Secure."
- **HTTP/2 and HTTP/3** require TLS in practice.

Deep dive: [Web Security](../07-http-and-web/04-web-security.md)

</details>

---

### 10. What is the testing pyramid, and why should the base be unit tests?

<details>
<summary>Reveal answer</summary>

The **testing pyramid** has three layers:
- **Unit tests** (base, most tests) -- test individual methods/classes in isolation. Fast, cheap, stable.
- **Integration tests** (middle) -- test multiple components together (API + DB, HTTP calls). Slower, more realistic.
- **E2E tests** (top, fewest tests) -- test the entire system through the UI or API. Slow, brittle, expensive.

The base is unit tests because they give the **fastest feedback loop**, are **cheapest to maintain**, and catch most logic bugs. Integration and E2E tests cover gaps that unit tests can't (wiring, infrastructure, real behavior).

Deep dive: [Testing Pyramid](../11-testing/01-testing-pyramid.md)

</details>

---

### 11. What is the difference between a mock, a stub, and a fake?

<details>
<summary>Reveal answer</summary>

- **Stub** -- returns pre-configured data. Used to control indirect inputs. You don't verify calls on it.
- **Mock** -- records interactions. You **verify** that specific methods were called with expected arguments.
- **Fake** -- a working implementation unsuitable for production (e.g., in-memory database, fake SMTP server).

```csharp
// Stub: just returns data
var stub = new Mock<IOrderRepo>();
stub.Setup(r => r.GetById(1)).Returns(order);

// Mock: verify interaction
mock.Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
```

**Rule of thumb**: prefer stubs for most tests; use mocks sparingly to verify critical side effects.

Deep dive: [Testing Pyramid](../11-testing/01-testing-pyramid.md)

</details>

---

### 12. When should you mock and when should you NOT mock?

<details>
<summary>Reveal answer</summary>

**Mock when**:
- The dependency is slow (database, HTTP, file system).
- You need to test error paths (simulate exceptions, timeouts).
- You want to verify a side effect (email sent, event published).

**Don't mock when**:
- The dependency is a simple in-memory object (no I/O).
- You're mocking the thing you're testing (tests prove nothing).
- The mock setup is more complex than the real implementation.
- You're testing integration behavior -- use a real database or `WebApplicationFactory`.

Over-mocking leads to **tests that pass but don't prove your code works**.

Deep dive: [Mocking and Best Practices](../11-testing/02-mocking-and-best-practices.md)

</details>

---

### 13. What is `WebApplicationFactory` and when do you use it?

<details>
<summary>Reveal answer</summary>

`WebApplicationFactory<TEntryPoint>` is a test host from `Microsoft.AspNetCore.Mvc.Testing` that boots your entire ASP.NET Core app **in-memory** for integration testing. It creates an `HttpClient` that sends requests to the in-memory server.

```csharp
public class OrdersTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Get_Returns200()
    {
        var client = factory.CreateClient();
        var response = await client.GetAsync("/api/orders");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

You can override services (swap the real DB for an in-memory one) using `WithWebHostBuilder`.

Deep dive: [Mocking and Best Practices](../11-testing/02-mocking-and-best-practices.md)

</details>

---

### 14. What is the Arrange-Act-Assert (AAA) pattern?

<details>
<summary>Reveal answer</summary>

AAA structures every test into three clear phases:

1. **Arrange** -- set up the test data, dependencies, and the system under test.
2. **Act** -- call the method or perform the action being tested.
3. **Assert** -- verify the outcome (return value, state change, or interaction).

```csharp
[Fact]
public void Discount_AppliesTenPercent()
{
    // Arrange
    var calculator = new PriceCalculator();
    var price = 100m;

    // Act
    var result = calculator.ApplyDiscount(price, 0.10m);

    // Assert
    result.Should().Be(90m);
}
```

Keep each section **short and focused**. One logical assertion per test.

Deep dive: [Mocking and Best Practices](../11-testing/02-mocking-and-best-practices.md)

</details>

---

### 15. What is stress testing and what metrics should you watch?

<details>
<summary>Reveal answer</summary>

**Stress testing** pushes the system beyond normal load to find its breaking point and observe degradation behavior. Tools like **k6** or **NBomber** simulate hundreds/thousands of concurrent users.

Key metrics:
- **Response time** (p50, p95, p99) -- how fast are responses under load?
- **Throughput** (requests/second) -- how many requests can the system handle?
- **Error rate** -- percentage of failed requests.
- **CPU / Memory** -- resource saturation.
- **Connection pool exhaustion** -- database or HTTP connections running out.

**Important**: always test against a production-like environment, not localhost.

Deep dive: [Stress and Load Testing](../11-testing/03-stress-and-load-testing.md)

</details>

---

### 16. A developer says "our API returns 200 for everything and puts the error in the response body." What's wrong with this approach?

<details>
<summary>Reveal answer</summary>

This violates HTTP semantics and causes multiple problems:
- **Proxies, CDNs, and caches** rely on status codes -- a 200 with an error body gets cached as a success.
- **Monitoring tools** count 4xx/5xx for alerting -- all errors become invisible.
- **Client libraries** (HttpClient, Axios) use status codes to branch success/failure logic.
- **API consumers** must parse every response body to check for errors instead of checking the status code.

Use **proper status codes** (400, 404, 409, 500) and a consistent error body format (like RFC 7807 Problem Details).

Deep dive: [REST API Design](../07-http-and-web/03-rest-api-design.md)

</details>

---
