# API Resilience

## What it is

The ability of an API to keep functioning acceptably when **failures, slowdowns, load spikes, or unstable dependencies** occur. It's not about avoiding failures. It's about **failing in a controlled manner** and **recovering quickly**, so that one slow dependency or one abusive client doesn't become a domino effect.

An API needs resilience in **two directions**:

| | Outbound: your API calls others | Inbound: others call your API |
|---|---|---|
| Threat | A slow or failing dependency drags your API down (threads, sockets, connections pile up) | Traffic spikes, abusive clients, slow handlers, deploys and restarts |
| .NET tools | `Microsoft.Extensions.Http.Resilience` / Polly v8: retry, timeouts, circuit breaker, hedging, fallback | Rate limiter, request timeouts, concurrency limiter, health checks, graceful shutdown |
| What the caller sees | Nothing, if the fallback works. Otherwise an error your code handles. | `429` / `503` / `504`, fast and explicit, instead of a hang |

The individual patterns (retry, circuit breaker, bulkhead, timeouts) are covered in depth in [Reliability and SRE](../18-reliability-and-sre/README.md). This file is the **ASP.NET Core wiring**.

---

## Outbound: the standard resilience handler

On .NET 8+ the recommended API is `Microsoft.Extensions.Http.Resilience`, built on Polly v8. The old `Microsoft.Extensions.Http.Polly` package is officially deprecated: *"The Microsoft.Extensions.Http.Polly NuGet package is deprecated."*

```csharp
// dotnet add package Microsoft.Extensions.Http.Resilience
builder.Services
    .AddHttpClient<PaymentClient>(c => c.BaseAddress = new Uri("https://api.payment.example"))
    .AddStandardResilienceHandler();
```

That one call chains **five strategies, outermost to innermost**:

| # | Strategy | Default |
|---|---|---|
| 1 | Rate limiter | 1,000 permits, queue 0 |
| 2 | Total request timeout | 30 s, for the whole operation including retries |
| 3 | Retry | 3 retries, exponential backoff, jitter on, 2 s base delay |
| 4 | Circuit breaker | Opens at a 10% failure ratio, min throughput 100, 30 s sampling, 5 s break |
| 5 | Attempt timeout | 10 s per attempt |

Retry and circuit breaker react to **HTTP 500+, 408, 429**, `HttpRequestException` and `TimeoutRejectedException`. Retry also honors the `Retry-After` header by default (`HttpRetryStrategyOptions.ShouldRetryAfterHeader` *"Defaults to `true`."*).

> When a timeout strategy fires, *"Polly throws a `TimeoutRejectedException` … not the standard TimeoutException"*. Catch the right type.

> **`HttpClient.Timeout` (100 s default) used to override the total timeout.** Since `Microsoft.Extensions.Http.Resilience` 8.10.0 the standard resilience and hedging handlers set `HttpClient.Timeout` to infinite *"to allow the timeout strategies to control the timeout"* (source comment from dotnet/extensions PR #5363, which fixed issue #4770). On older versions, a lower `HttpClient.Timeout` silently wins.

### Customizing the standard handler

```csharp
builder.Services
    .AddHttpClient<PaymentClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.MaxRetryAttempts = 2;
        options.Retry.DisableForUnsafeHttpMethods();                  // see below
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(3);     // per attempt
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10); // whole operation
    });
```

Size these from the outside in. The total timeout must fit inside **your own** request deadline. See [Timeouts](../18-reliability-and-sre/04-timeouts.md#the-cascading-timeout-rule).

### POST is retried by default

From the docs: *"By default, the standard resilience handler is configured to make retries for all HTTP methods."* A retried `POST /payments` after a timeout can charge the customer twice.

```csharp
options.Retry.DisableForUnsafeHttpMethods();               // POST, PATCH, PUT, DELETE, CONNECT
// or, explicitly:
options.Retry.DisableFor(HttpMethod.Post, HttpMethod.Delete);
```

The alternative is to make the call idempotent with an `Idempotency-Key` header. See [Idempotency](../19-distributed-systems/03-idempotency.md).

### Don't stack handlers

The docs: *"you should only add one resilience handler and avoid stacking handlers."* Two handlers on the same client multiply retries.

A common setup applies a default handler to every client, then swaps it for one client:

```csharp
builder.Services.ConfigureHttpClientDefaults(http => http.AddStandardResilienceHandler());

#pragma warning disable EXTEXP0001 // RemoveAllResilienceHandlers is still [Experimental]
builder.Services.AddHttpClient<SearchClient>()
    .RemoveAllResilienceHandlers()
    .AddStandardHedgingHandler();
#pragma warning restore EXTEXP0001
```

`RemoveAllResilienceHandlers` shipped in `Microsoft.Extensions.Http.Resilience` 9.2.0 and is still marked experimental (`EXTEXP0001`).

### Hedging: for tail latency

*"Hedging retries slow requests in parallel."* If the first attempt fails or hasn't answered after a delay (2 s by default), another one is sent, and the fastest successful response wins.

```csharp
builder.Services.AddHttpClient<CatalogClient>()
    .AddStandardHedgingHandler();
```

- The pipeline is total timeout (30 s) → hedging → **per-endpoint** rate limiter, circuit breaker and attempt timeout. Circuit breakers come from *"a pool of circuit breakers … based on the URL authority (scheme + host + port)"*.
- Polly's `MaxHedgedAttempts` defaults to **1** extra attempt. The allowed maximum is 10.
- Polly's own advice: *"If low latency is not a critical requirement, you may find the retry strategy more appropriate."*

> **Hedging sends duplicate requests by design.** Use it only for idempotent reads, ideally across several replicas or regions. Never use it for a payment `POST`.

### Custom pipeline: `AddResilienceHandler`

When the standard defaults don't fit, build the pipeline yourself. **The first strategy added is the outermost.**

```csharp
builder.Services.AddHttpClient<InventoryClient>()
    .AddResilienceHandler("inventory", pipeline =>
    {
        pipeline.AddTimeout(TimeSpan.FromSeconds(10));        // total (outermost)
        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 2,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        });
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions());
        pipeline.AddTimeout(TimeSpan.FromSeconds(3));         // per attempt (innermost)
    });
```

The final pipeline name combines the client name and `"inventory"`.

### Non-HTTP calls: `AddResiliencePipeline`

The same Polly v8 pipelines work for anything that isn't `HttpClient`: SDK calls, message brokers, storage clients. (gRPC clients registered with `AddGrpcClient` are `HttpClient`-based and can use `AddStandardResilienceHandler` directly, with `Grpc.Net.ClientFactory` 2.64.0 or later.) Register them in DI:

```csharp
// dotnet add package Microsoft.Extensions.Resilience
builder.Services.AddResiliencePipeline("storage-upload", pipeline =>
{
    pipeline.AddRetry(new RetryStrategyOptions
    {
        // A custom ShouldHandle replaces the default (all exceptions except OperationCanceledException).
        // AddTimeout below throws TimeoutRejectedException, not TimeoutException, so list it explicitly.
        ShouldHandle = new PredicateBuilder().Handle<TimeoutRejectedException>().Handle<TimeoutException>(),
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true
    });
    pipeline.AddTimeout(TimeSpan.FromSeconds(30));
});

public sealed class ReportExporter(ResiliencePipelineProvider<string> pipelines)
{
    private readonly ResiliencePipeline _pipeline = pipelines.GetPipeline("storage-upload");

    public Task ExportAsync(Report report, CancellationToken ct) =>
        _pipeline.ExecuteAsync(async token => await UploadAsync(report, token), ct).AsTask();
}
```

Keyed injection also works: `[FromKeyedServices("storage-upload")] ResiliencePipeline pipeline`. For database calls specifically, see [Database Resilience](../11-data-access/14-database-resilience.md).

### Fallback

Return something useful when everything else failed: cached data, a default, a degraded response.

```csharp
var pipeline = new ResiliencePipelineBuilder<IReadOnlyList<Product>>()
    .AddFallback(new FallbackStrategyOptions<IReadOnlyList<Product>>
    {
        ShouldHandle = new PredicateBuilder<IReadOnlyList<Product>>()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>(),
        FallbackAction = _ => Outcome.FromResultAsValueTask<IReadOnlyList<Product>>(cachedProducts)
    })
    .Build();
```

### Migrating from Polly v7

Many older blog posts and Stack Overflow answers show v7. Map them like this:

| Polly v7 | Polly v8 | Note |
|---|---|---|
| `WaitAndRetryAsync(...)` | `AddRetry(new RetryStrategyOptions { ... })` | |
| `CircuitBreakerAsync(n, duration)` | `AddCircuitBreaker(...)` | *"Polly V8 does not support the standard ("classic") circuit breaker with consecutive failure counting."* v8 uses a failure **ratio**. |
| `TimeoutAsync(...)` | `AddTimeout(...)` | v8 *"does not support pessimistic timeouts"*. Your code must honor the token. |
| `BulkheadAsync(...)` | `AddConcurrencyLimiter(permitLimit, queueLimit)` | Needs `Polly.RateLimiting` |
| `FallbackAsync(...)` | `AddFallback(...)` | |
| `Policy.WrapAsync(...)` | Chain `Add*` calls | First added = outermost |
| `PolicyRegistry` | `ResiliencePipelineRegistry<TKey>` / `ResiliencePipelineProvider<TKey>` | |

### Chaos testing your resilience

Since Polly 8.3.0, the former Simmy chaos strategies are part of `Polly.Core`: `AddChaosLatency`, `AddChaosFault`, `AddChaosOutcome`, `AddChaosBehavior`. Inject latency into a test environment and check that timeouts and fallbacks behave as designed. See [Chaos Engineering](../18-reliability-and-sre/05-chaos-engineering.md).

---

## Inbound: protecting your own API

### Request timeouts (.NET 8+)

A slow handler holds a thread, a DB connection and memory long after the client gave up. The request timeouts middleware puts a deadline on each endpoint:

```csharp
builder.Services.AddRequestTimeouts(options =>
{
    options.DefaultPolicy = new RequestTimeoutPolicy { Timeout = TimeSpan.FromSeconds(10) };
    options.AddPolicy("reports", TimeSpan.FromSeconds(60));
});

var app = builder.Build();
app.UseRouting();
app.UseRequestTimeouts(); // must come after UseRouting

app.MapGet("/orders/{id}", async (int id, AppDbContext db, CancellationToken ct) =>
    await db.Orders.FindAsync([id], ct));                        // default policy: 10 s

app.MapGet("/reports/monthly", BuildMonthlyReport).WithRequestTimeout("reports");
app.MapGet("/exports/full", StreamFullExport).DisableRequestTimeout();
```

How it actually works:

- It **signals**; it doesn't kill. The timeout cancels `HttpContext.RequestAborted`, but *"`Abort()` isn't automatically called on the request"*. Your handler must pass the `CancellationToken` down (DB, HttpClient) for anything to stop.
- If the handler doesn't produce a response, the middleware returns **504**. Change it with `TimeoutStatusCode` or `WriteTimeoutResponse`.
- Adding the middleware alone *"doesn't automatically start triggering timeouts"*. You need a default policy or per-endpoint policies.
- *"When an app is running in debug mode, the timeout middleware doesn't trigger."* Don't conclude it's broken from a debugging session.

### Rate limiting (.NET 7+)

Limit how much any single client can consume, so one noisy tenant can't starve the rest:

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests; // default is 503!

    options.AddPolicy("per-client", httpContext =>
        RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: httpContext.User.FindFirst("client_id")?.Value ?? "anonymous",
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100,                         // burst capacity
                TokensPerPeriod = 50,
                ReplenishmentPeriod = TimeSpan.FromSeconds(10),
                QueueLimit = 0
            }));

    options.OnRejected = (context, ct) =>
    {
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            context.HttpContext.Response.Headers.RetryAfter =
                ((long)Math.Ceiling(retryAfter.TotalSeconds)).ToString(NumberFormatInfo.InvariantInfo);
        return ValueTask.CompletedTask;
    };
});

app.UseRouting();
app.UseRateLimiter(); // after UseRouting when using endpoint-specific policies
app.MapControllers().RequireRateLimiting("per-client");
```

| Algorithm | Behavior | Good for |
|---|---|---|
| Fixed window | N requests per window | Simple quotas |
| Sliding window | Window split into segments, smoother edges | Avoiding bursts at window boundaries |
| Token bucket | Burst up to capacity, steady refill | APIs with bursty but bounded clients |
| Concurrency | Caps **in-flight** requests. It *"doesn't cap the number of requests in a time period."* | Load shedding (below) |

Gotchas:

- **`RejectionStatusCode` defaults to 503.** 503 tells callers and load balancers the *server* is unavailable. Use **429** for per-client throttling. A status set in `OnRejected` wins over this default.
- **Partition key choice.** The docs warn that partitioning by client IP makes the app *"vulnerable to Denial of Service attacks that use IP Source Address Spoofing"*. Behind a proxy, `RemoteIpAddress` is the proxy unless Forwarded Headers middleware is configured with trusted proxies only. Prefer an authenticated client/tenant ID.
- **Unbounded keys exhaust memory.** Partitioning on raw user-controlled input (any header value) creates one limiter per value. That is itself a DoS vector.
- It is *"not a comprehensive solution for Distributed Denial of Service (DDoS) attacks."* That job belongs to the edge (WAF, API gateway, CDN). See [API Gateway and BFF](../08-architecture-and-patterns/14-api-gateway-and-bff.md).

### Load shedding with a concurrency limiter

Under overload, **fast rejection beats slow death**. If the app can comfortably process 200 concurrent requests, keep at most a short queue and reject everything beyond it immediately, instead of leaving requests in a 30-second wait that times out anyway:

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(_ =>
        RateLimitPartition.GetConcurrencyLimiter("global", _ => new ConcurrencyLimiterOptions
        {
            PermitLimit = 200, // in-flight requests
            QueueLimit = 50    // waiting requests; beyond this, reject immediately
            // QueueProcessingOrder defaults to OldestFirst
        }));
});
```

A global-only limiter can run before `UseRouting`. `RejectionStatusCode` is a single default shared by every limiter in `AddRateLimiter`. If you set it to 429 for per-client throttling, load-shed requests get 429 too. The exception is a per-client policy written as an `IRateLimiterPolicy<T>`: it can set 429 in its own `OnRejected` while the global limiter keeps 503. Size `PermitLimit` from load tests, not guesses. The same idea applied to outbound calls is the [Bulkhead Pattern](../18-reliability-and-sre/03-bulkhead-pattern.md).

### Health checks: liveness vs readiness

The docs: *"Readiness indicates if the app is running normally but isn't ready to receive requests. Liveness indicates if an app has crashed and must be restarted."*

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);

// Liveness: is the process alive? Run NO dependency checks.
app.MapHealthChecks("/healthz/live", new HealthCheckOptions { Predicate = _ => false });

// Readiness: can it serve traffic right now? Check dependencies.
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

By default, `Healthy` and `Degraded` return **200** and `Unhealthy` returns **503**.

> **Classic mistake:** putting the database check in the liveness probe. When the database blips, every pod fails liveness and gets **restarted** at the same time. Restarts don't fix the database, and the cold starts make recovery slower. Dependencies belong in readiness, which only takes the pod out of the load balancer.

### Graceful shutdown

Every deploy is a planned failure. On `SIGTERM`, the host triggers `ApplicationStopping`, stops accepting new connections, and *"waits for requests on existing connections to complete, for as long as the shutdown timeout allows."* For HTTP/2 and HTTP/3, Kestrel sends `GOAWAY` first. When the timeout expires, *"the pending connections and requests are aborted forcefully."*

```csharp
// Default: 30 s (it was 5 s in .NET 5 and earlier)
builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromSeconds(25));
```

Keep `ShutdownTimeout` below the orchestrator's grace period, so the app finishes draining before it gets killed. Background workers should observe the `stoppingToken`. See [Background Services](06-background-services.md).

---

## Putting it together

```text
client ──► [edge: WAF / gateway / DDoS protection]
         ──► [rate limiter: per-client 429, global concurrency 503]
         ──► [request timeout: 504 after N s]
         ──► handler ──► HttpClient + standard resilience handler ──► dependency
                     └─► DB with one retry layer (see Data Access)
health: /live (process only)   /ready (dependencies)
deploy: readiness fails → drain → ShutdownTimeout → exit
```

## When NOT to add more resilience

- **Retrying inside an API that callers already retry.** 3 retries at each layer means 4 attempts × 4 attempts = 16 calls to the dependency. Retry at one layer. See [Retries, Backoff and Jitter](../18-reliability-and-sre/01-retries-backoff-jitter.md).
- **Retrying non-idempotent calls.** Disable retries for unsafe methods, or add idempotency keys first.
- **Stacking resilience handlers** on one client.
- **Long timeouts "to be safe".** A 100 s timeout on an endpoint with a 2 s SLO is a resource leak, not resilience.

---

[← Previous: OAuth 2.0](03-oauth2.md) | [Next: Middleware →](05-middleware.md) | [Back to index](README.md)
