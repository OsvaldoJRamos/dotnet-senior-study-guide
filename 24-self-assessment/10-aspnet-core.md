# ASP.NET Core

> Read the questions, think about your answer, then click to reveal.

---

### 1. What are the three DI service lifetimes in ASP.NET Core, and when would you choose each?

<details>
<summary>Reveal answer</summary>

- **Singleton** -- one instance for the app's entire lifetime. Use for stateless services, caches, configuration wrappers.
- **Scoped** -- one instance per HTTP request (or per scope). Use for `DbContext`, unit-of-work, per-request state.
- **Transient** -- new instance every time it's requested. Use for lightweight, stateless helpers.

Choose the **narrowest lifetime that still meets your needs** to avoid unintentional shared state.

Deep dive: [Service Lifetimes](../10-aspnet-core/02-service-lifetimes.md)

</details>

---

### 2. What is the "captive dependency" problem, and how does ASP.NET Core help prevent it?

<details>
<summary>Reveal answer</summary>

A **captive dependency** happens when a longer-lived service (e.g., Singleton) depends on a shorter-lived one (e.g., Scoped). The scoped service gets "captured" and effectively becomes a singleton, which can cause stale data or concurrency bugs.

ASP.NET Core provides **scope validation** (enabled by default in Development): it throws an `InvalidOperationException` at startup if a singleton tries to resolve a scoped service.

```csharp
// This would throw at startup in Development
builder.Services.AddSingleton<MySingleton>(); // depends on IMyScoped
builder.Services.AddScoped<IMyScoped, MyScoped>();
```

Deep dive: [Service Lifetimes](../10-aspnet-core/02-service-lifetimes.md)

</details>

---

### 3. Why does the order of middleware registration matter? What happens if you put `UseAuthentication` after `UseAuthorization`?

<details>
<summary>Reveal answer</summary>

Middleware executes in the **order it is registered**. Each middleware can short-circuit the pipeline or pass the request to the next one. If `UseAuthentication()` comes **after** `UseAuthorization()`, the authorization middleware runs before the user identity is established -- so every request appears unauthenticated and authorization checks fail.

Correct order:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Deep dive: [Middleware](../10-aspnet-core/05-middleware.md)

</details>

---

### 4. How do you create custom middleware, and when would you short-circuit the pipeline?

<details>
<summary>Reveal answer</summary>

Custom middleware is a class with an `InvokeAsync` method that receives `HttpContext` and a `RequestDelegate next`:

```csharp
public class ApiKeyMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.ContainsKey("X-Api-Key"))
        {
            context.Response.StatusCode = 401;
            return; // short-circuit -- don't call next
        }
        await next(context);
    }
}
```

**Short-circuiting** is useful for validation, caching, or health-check responses where you want to stop processing early and return immediately.

Deep dive: [Middleware](../10-aspnet-core/05-middleware.md)

</details>

---

### 5. What are the main differences between Minimal APIs and Controllers? When would you pick one over the other?

<details>
<summary>Reveal answer</summary>

| Aspect | Minimal APIs | Controllers |
|--------|-------------|-------------|
| Boilerplate | Very little, lambda-based | More ceremony (classes, attributes) |
| Filters | Endpoint filters | Action filters, model validation |
| API versioning | Manual or via packages | Built-in attribute support |
| Best for | Microservices, small APIs | Large APIs, complex routing |

Choose **Minimal APIs** for simple, fast-to-build services. Choose **Controllers** when you need rich model binding, content negotiation, or a large surface area with shared filters.

Deep dive: [Minimal APIs](../10-aspnet-core/08-minimal-apis.md)

</details>

---

### 6. How do you consume a scoped service (like DbContext) inside an `IHostedService`?

<details>
<summary>Reveal answer</summary>

`IHostedService` is registered as a **singleton**, so you cannot inject scoped services directly (captive dependency). Instead, inject `IServiceScopeFactory` and create a scope manually:

```csharp
public class MyWorker(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use db...
    }
}
```

Deep dive: [Background Services](../10-aspnet-core/06-background-services.md)

</details>

---

### 7. What happens if `ExecuteAsync` in a `BackgroundService` throws an unhandled exception?

<details>
<summary>Reveal answer</summary>

In .NET 6+, an unhandled exception in `ExecuteAsync` will **stop the host** by default (the app crashes). In earlier versions, the exception was silently swallowed and the service just stopped running.

You can control this behavior with `HostOptions.BackgroundServiceExceptionBehavior`:
- `StopHost` (default in .NET 6+) -- crash the app.
- `Ignore` -- swallow the exception and stop the service silently.

**Best practice**: always wrap your work in a `try/catch` with proper logging.

Deep dive: [Background Services](../10-aspnet-core/06-background-services.md)

</details>

---

### 8. What is the difference between `IMemoryCache` and `IDistributedCache`? When would you use each?

<details>
<summary>Reveal answer</summary>

- **`IMemoryCache`** -- stores data in the **process memory** of a single server. Fast, but not shared across instances. Good for single-server apps or data that's OK to be duplicated per instance.
- **`IDistributedCache`** -- abstraction over an external store (Redis, SQL Server). Shared across all instances. Necessary for horizontally scaled apps.

Use `IMemoryCache` for hot, per-server lookups. Use `IDistributedCache` when consistency across instances matters (e.g., session state, feature flags).

Deep dive: [Caching](../10-aspnet-core/07-caching.md)

</details>

---

### 9. What is a "cache stampede" and how can you prevent it?

<details>
<summary>Reveal answer</summary>

A **cache stampede** (thundering herd) happens when a popular cache entry expires and many concurrent requests all try to regenerate it at the same time, overloading the data source.

Prevention strategies:
- **Lock / semaphore**: only one thread regenerates; others wait.
- **`GetOrCreateAsync` with `SemaphoreSlim`**: serialize cache population.
- **Stale-while-revalidate**: serve the stale value while refreshing in the background.
- **Randomized expiration**: add jitter to TTLs so entries don't all expire at once.

Deep dive: [Caching](../10-aspnet-core/07-caching.md)

</details>

---

### 10. How does SignalR scale across multiple server instances, and what role does a Redis backplane play?

<details>
<summary>Reveal answer</summary>

By default, SignalR keeps connection state **in-process**, so a message sent on Server A won't reach clients connected to Server B. A **Redis backplane** acts as a pub/sub broker: every server publishes messages to Redis, and every server subscribes to them, ensuring all clients receive broadcasts regardless of which server they're connected to.

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis(connectionString);
```

Deep dive: [SignalR](../10-aspnet-core/09-signalr.md)

</details>

---

### 11. What transport protocols does SignalR support, and how does it negotiate them?

<details>
<summary>Reveal answer</summary>

SignalR supports three transports, tried in order of preference:
1. **WebSockets** -- full-duplex, best performance.
2. **Server-Sent Events (SSE)** -- server-to-client only, uses HTTP.
3. **Long Polling** -- fallback, higher latency.

The client sends a **negotiate** request; the server responds with available transports. The client picks the best one the environment supports. You can force a specific transport if needed.

Deep dive: [SignalR](../10-aspnet-core/09-signalr.md)

</details>

---

### 12. What is the difference between an access token and a refresh token in OAuth 2.0?

<details>
<summary>Reveal answer</summary>

- **Access token** -- short-lived (minutes), sent with every API request in the `Authorization` header. Grants access to resources. If compromised, limited damage due to short TTL.
- **Refresh token** -- long-lived (days/weeks), stored securely on the client. Used to obtain a **new access token** without re-authenticating the user. Should be rotated on use and revocable.

**Never send the refresh token to a resource server** -- it is only sent to the authorization server's token endpoint.

Deep dive: [OAuth 2.0](../10-aspnet-core/03-oauth2.md)

</details>

---

### 13. How does ASP.NET Core validate a JWT? What happens if you forget to set `ValidateIssuer` or `ValidateAudience`?

<details>
<summary>Reveal answer</summary>

ASP.NET Core's `JwtBearerHandler` validates: **signature** (using the issuer's key), **expiration**, **issuer**, and **audience**. If you disable `ValidateIssuer` or `ValidateAudience`, any token from any identity provider or intended for any application would be accepted -- a serious security hole.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,   // always keep true
    ValidateAudience = true, // always keep true
    ValidateLifetime = true,
    ValidIssuer = "https://my-idp.com",
    ValidAudience = "my-api"
};
```

Deep dive: [OAuth 2.0](../10-aspnet-core/03-oauth2.md)

</details>

---

### 14. How does the built-in Rate Limiting middleware work in .NET 7+? Name two common algorithms.

<details>
<summary>Reveal answer</summary>

The `Microsoft.AspNetCore.RateLimiting` middleware applies rate limits using a **policy-based model**. You configure policies in `Program.cs` and apply them globally, per-endpoint, or per-controller.

Common algorithms:
- **Fixed Window** -- limits requests in fixed time intervals (e.g., 100 req/min).
- **Sliding Window** -- smoother than fixed; divides the window into segments.
- **Token Bucket** -- allows bursts up to a bucket capacity, refills at a steady rate.
- **Concurrency** -- limits how many requests are processed simultaneously.

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 15. What is the difference between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`?

<details>
<summary>Reveal answer</summary>

| Interface | Lifetime | Reloads on change? | Use case |
|-----------|----------|-------------------|----------|
| `IOptions<T>` | Singleton | No | Static config, read once at startup |
| `IOptionsSnapshot<T>` | Scoped | Yes, per request | Config that may change; accessed per request |
| `IOptionsMonitor<T>` | Singleton | Yes, via callback | Singleton services that need live config updates |

**Key rule**: if your service is a singleton and needs fresh config, use `IOptionsMonitor<T>`. If scoped, use `IOptionsSnapshot<T>`.

Deep dive: [Dependency Injection](../10-aspnet-core/01-dependency-injection.md)

</details>

---

### 16. Your API calls an external service that occasionally times out. How would you apply retry + circuit breaker using Polly in .NET 8?

<details>
<summary>Reveal answer</summary>

In .NET 8, use `Microsoft.Extensions.Http.Resilience` which integrates Polly v8:

```csharp
builder.Services.AddHttpClient("catalog")
    .AddStandardResilienceHandler(); // retry + circuit breaker + timeout out of the box
```

Or customize:

```csharp
builder.Services.AddHttpClient("catalog")
    .AddResilienceHandler("custom", builder =>
    {
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential
        });
        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(10),
            FailureRatio = 0.5
        });
    });
```

**Order matters**: retry should wrap circuit breaker (retry is outer, circuit breaker is inner).

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 17. What are health checks in ASP.NET Core, and how do they integrate with container orchestrators?

<details>
<summary>Reveal answer</summary>

Health checks expose HTTP endpoints that report the app's status. You register checks, tag them, and map **separate** endpoints for liveness and readiness:

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);

app.MapHealthChecks("/healthz/live", new HealthCheckOptions { Predicate = _ => false });
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

Orchestrators such as Kubernetes poll these endpoints. A failing **liveness** probe makes the kubelet restart the container. A failing **readiness** probe removes the pod from the Service's endpoints (the load balancer).

**Key rule:** dependencies (database, Redis, external APIs) go in **readiness only**. If the database check is in liveness, a database blip restarts every pod at once, and restarting doesn't fix the database. By default `Healthy`/`Degraded` return 200 and `Unhealthy` returns 503.

Checks such as `.AddSqlServer()` / `.AddRedis()` come from the community `AspNetCore.Diagnostics.HealthChecks` packages, which Microsoft doesn't maintain. `AddDbContextCheck` is Microsoft's EF Core check.

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 18. How do refresh tokens work, and why not just use a long-lived access token?

<details>
<summary>Reveal answer</summary>

A long-lived access token is risky: if stolen, the attacker has extended access, and JWTs can't be revoked cheaply (they're stateless).

**Refresh token flow**:
1. User authenticates → gets a short-lived access token (5–15 min) + refresh token (days/weeks).
2. Access token expires → client sends refresh token to the auth server.
3. Auth server validates, **rotates** the refresh token (old one is invalidated), and issues a new access token.

Benefits: **access tokens expire quickly** (limited damage if stolen), and **refresh tokens can be revoked** server-side (stored in DB, indexed by user/session).

Important: treat refresh tokens like passwords — store them HttpOnly/Secure, detect reuse (if an already-rotated refresh token is presented, invalidate the entire chain).

Deep dive: [OAuth 2.0](../10-aspnet-core/03-oauth2.md)

</details>

---

### 19. What is the difference between OAuth 2.0 and OpenID Connect?

<details>
<summary>Reveal answer</summary>

- **OAuth 2.0** is an **authorization** framework (RFC 6749). It answers "can this app access this resource on behalf of the user?" and issues **access tokens**. It says nothing about who the user is.
- **OpenID Connect (OIDC)** is an **authentication** layer built on top of OAuth 2.0. It adds an **ID token** (a JWT) with standardized claims (`sub`, `name`, `email`) so the client can know who logged in, plus a `UserInfo` endpoint and discovery metadata (`/.well-known/openid-configuration`).

Rule of thumb:
- "Let this app post to my Twitter" → OAuth 2.0.
- "Log in with Google" → OIDC.

In ASP.NET Core, `AddOpenIdConnect(...)` handles the OIDC dance and populates `HttpContext.User`.

Deep dive: [OAuth 2.0](../10-aspnet-core/03-oauth2.md)

</details>

---

### 20. What is the difference between `IHttpClientFactory` and `new HttpClient()`?

<details>
<summary>Reveal answer</summary>

Creating `new HttpClient()` per request leaks **sockets** (SocketException / port exhaustion under load) because each client holds its own connection pool. Reusing a static `HttpClient` avoids that but locks DNS entries — if an IP changes, you keep hitting the old one.

`IHttpClientFactory` (from `Microsoft.Extensions.Http`) solves both: it manages a pool of `HttpMessageHandler` instances with a configurable lifetime (default 2 minutes), so sockets are reused *and* DNS is refreshed.

Three usage patterns:
- **Named clients**: `factory.CreateClient("github")` with a preconfigured base address + headers.
- **Typed clients**: `builder.Services.AddHttpClient<IGitHubApi, GitHubApi>()` — DI injects `GitHubApi` directly.
- **Polly handlers**: add `.AddStandardResilienceHandler()` (Microsoft.Extensions.Http.Resilience) for retries, circuit breaker, and timeouts.

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 21. What does `AddStandardResilienceHandler()` configure, in which order, and what is the dangerous default for `POST`?

<details>
<summary>Reveal answer</summary>

Five strategies, from **outermost to innermost**:

| # | Strategy | Default |
|---|---|---|
| 1 | Rate limiter | 1,000 permits, queue 0 |
| 2 | Total request timeout | 30 s |
| 3 | Retry | 3 retries, exponential, jitter, 2 s base delay |
| 4 | Circuit breaker | 10% failure ratio, min throughput 100, 30 s sampling, 5 s break |
| 5 | Attempt timeout | 10 s |

Retry and circuit breaker handle HTTP 500+, 408, 429, `HttpRequestException` and `TimeoutRejectedException`.

**The trap:** *"By default, the standard resilience handler is configured to make retries for all HTTP methods"*, including `POST`. A retried payment after a timeout can charge twice. Fix with `options.Retry.DisableForUnsafeHttpMethods()` (POST, PATCH, PUT, DELETE, CONNECT), or make the call idempotent with an idempotency key.

Also: add **one** handler per client. Stacking handlers multiplies retries.

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 22. What is hedging, and when would you pick `AddStandardHedgingHandler` over retry?

<details>
<summary>Reveal answer</summary>

*"Hedging retries slow requests in parallel."* If the first attempt fails or hasn't answered after a delay (2 s by default), another attempt is sent, and the fastest successful response wins. Retry only fires **after a failure**. Hedging also fires on **slowness**, without waiting for the first attempt to finish.

- **Use it for:** latency-sensitive, **idempotent reads**, ideally across multiple replicas/regions, to cut tail latency (p99).
- **Don't use it for:** writes. Hedging sends duplicate requests by design.
- The standard hedging handler uses per-endpoint circuit breakers keyed by URL authority (scheme + host + port). Polly's `MaxHedgedAttempts` defaults to 1 extra attempt (max 10).

Polly's own advice: *"If low latency is not a critical requirement, you may find the retry strategy more appropriate."*

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 23. You add `UseRequestTimeouts()` with a 10-second default policy. What actually happens when a request exceeds 10 seconds?

<details>
<summary>Reveal answer</summary>

The middleware (.NET 8+) **cancels `HttpContext.RequestAborted`**. It does not kill the request: *"`Abort()` isn't automatically called on the request"*. So:

- If your handler passes the `CancellationToken` to EF Core / `HttpClient`, the work stops and an `OperationCanceledException` propagates.
- If the handler doesn't produce a response, the middleware returns **504** (configurable via `TimeoutStatusCode` / `WriteTimeoutResponse`).
- If the handler **ignores the token**, the work keeps running. The timeout is cooperative.

Other gotchas: `UseRequestTimeouts` must come after `UseRouting`; adding the middleware alone doesn't trigger timeouts without a policy; and *"When an app is running in debug mode, the timeout middleware doesn't trigger."*

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 24. The rate limiting middleware rejects requests with which status code by default? How should you choose the partition key?

<details>
<summary>Reveal answer</summary>

`RejectionStatusCode` **defaults to 503 Service Unavailable**. For per-client throttling, set it to **429 Too Many Requests**. 503 tells clients and load balancers the server itself is temporarily unavailable (overload or maintenance). A status code set in `OnRejected` wins over the default; also set `Retry-After` there when the lease exposes it.

Partition key:
- **Prefer an authenticated client/tenant ID.**
- **Client IP is weak.** The docs warn it makes the app *"vulnerable to Denial of Service attacks that use IP Source Address Spoofing"*. Behind a proxy, every request looks like the proxy's IP unless Forwarded Headers middleware is configured with trusted proxies.
- **Never partition on unbounded raw input** (arbitrary header values). Each value creates a limiter, which exhausts memory.

The middleware is not a DDoS defense. That belongs at the edge (WAF, gateway, CDN).

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 25. What happens to in-flight requests when a pod receives SIGTERM during a deploy?

<details>
<summary>Reveal answer</summary>

The host triggers `IHostApplicationLifetime.ApplicationStopping` and the server stops accepting new connections. For HTTP/2 and HTTP/3, Kestrel sends `GOAWAY` first. Existing requests continue *"for as long as the shutdown timeout allows"*. When `HostOptions.ShutdownTimeout` expires, *"the pending connections and requests are aborted forcefully."*

- Default `ShutdownTimeout`: **30 seconds** (it was 5 s in .NET 5 and earlier).
- Keep it **below the orchestrator's grace period**, so draining finishes before the process is killed.
- `BackgroundService`s must observe `stoppingToken`, or they get cut mid-work.

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---

### 26. You're migrating Polly v7 code to v8. What behaviors don't carry over 1:1?

<details>
<summary>Reveal answer</summary>

| v7 | v8 |
|---|---|
| `CircuitBreakerAsync(5, 30s)`: opens after N **consecutive** failures | `AddCircuitBreaker(...)` uses a **failure ratio** over a sampling window. *"Polly V8 does not support the standard ("classic") circuit breaker with consecutive failure counting."* |
| `TimeoutAsync(..., TimeoutStrategy.Pessimistic)` | Not supported. v8 timeouts are cooperative only: your delegate **must** honor the `CancellationToken`. |
| `BulkheadAsync` | `AddConcurrencyLimiter` (package `Polly.RateLimiting`) |
| `Policy.WrapAsync` | Chain `Add*` calls on `ResiliencePipelineBuilder`. The first added is the outermost. |
| `PolicyRegistry` | `ResiliencePipelineRegistry<TKey>` / `ResiliencePipelineProvider<TKey>` |

For `HttpClient`, don't port v7 code to `Microsoft.Extensions.Http.Polly`, which is deprecated. Use `Microsoft.Extensions.Http.Resilience` (`AddStandardResilienceHandler` / `AddResilienceHandler`). For non-HTTP code, use `AddResiliencePipeline` (defined in `Polly.Extensions`, which `Microsoft.Extensions.Resilience` references).

Deep dive: [API Resilience](../10-aspnet-core/04-api-resilience.md)

</details>

---
