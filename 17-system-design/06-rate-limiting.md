# Rate Limiting

Rate limiting protects you from abuse (malicious or accidental), enforces quotas, and preserves fairness across tenants. A senior can explain the four canonical algorithms, when each fits, and how to implement them in a distributed system.

## Why rate-limit

- **Abuse protection** — credential stuffing, scraping, DoS.
- **Fairness** — a noisy tenant can't starve others.
- **Cost control** — every downstream call (OpenAI, Twilio, Stripe) has a price tag.
- **Back-pressure** — slow failures are better than collapse.

Microsoft: *"While rate limiting can help mitigate the risk of Denial of Service (DoS) attacks by limiting the rate at which requests are processed, it's not a comprehensive solution for Distributed Denial of Service (DDoS) attacks."* ([docs](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)). Combine rate limiting with a WAF / CDN for L7 DDoS.

## The four algorithms

### 1. Fixed window

Count requests per discrete time window (e.g., 100 per minute, reset at each minute boundary).

```text
 Window 10:00-10:01  [#########    ]  9 / 100 used
 Window 10:01-10:02  [             ]  0 / 100
```

| Pros | Cons |
|---|---|
| Trivial — one counter, one expiry | Boundary burst: 100 at 10:00:59 + 100 at 10:01:00 = 200 in 2 seconds |
| Easy to reason about | Not fair within the window |

### 2. Sliding window (log or counter-based)

Track timestamps of recent requests and count those within the last N seconds. Boundary-burst problem disappears.

**Sliding-window log** — keeps every timestamp (exact, but memory grows with rate).

**Sliding-window counter** — approximates by blending current + previous window counts weighted by elapsed fraction. This is what the ASP.NET Core sliding-window limiter does: windows are divided into `SegmentsPerWindow` segments and the window slides one segment at a time ([docs](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)).

| Pros | Cons |
|---|---|
| No boundary burst | Slightly more complex; approximation if counter-based |

### 3. Token bucket

A bucket holds up to N tokens. Tokens are added at a fixed rate (`R` per second). Each request consumes one. If the bucket is empty, reject. Accommodates short bursts while capping sustained throughput.

```text
 Bucket capacity: 10, refill 2/sec
 t=0 [##########]  burst allowed
 t=1 [####------]  6 requests eaten; 2 refilled next second
```

| Pros | Cons |
|---|---|
| Natural burst tolerance | Slightly harder to tune (capacity + rate) |
| Matches real-world quotas ("1 000 requests/min with burst 200") | |

### 4. Leaky bucket

A queue drained at a fixed rate. If the queue is full, new requests are rejected. Smooths output rate regardless of input pattern — classic traffic-shaping.

| Pros | Cons |
|---|---|
| Smooth, predictable downstream rate | No burst tolerance (unlike token bucket) |
| Good for protecting a fragile downstream | Queue adds latency |

> Token bucket vs leaky bucket: **token bucket allows bursts up to capacity**, **leaky bucket enforces strictly constant output**. Pick token bucket for fairness-to-users; leaky bucket for protecting a rate-sensitive downstream.

### Decision table

| Need | Pick |
|---|---|
| Simple per-minute quota, tolerate boundary burst | Fixed window |
| No boundary burst, fairness matters | Sliding window |
| Real-world quota with burst allowance | Token bucket |
| Strict constant output rate to a downstream | Leaky bucket |
| Cap concurrent in-flight (not per-time) | Concurrency limiter |

## `System.Threading.RateLimiting` (.NET 7+)

.NET ships the primitives in `System.Threading.RateLimiting` and the ASP.NET Core integration in `Microsoft.AspNetCore.RateLimiting` ([docs](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)). The rate-limit middleware page is versioned for ASP.NET Core 7.0 through 11.0 (per the official monikers on the page).

### Available limiters

| Class | Algorithm |
|---|---|
| `FixedWindowRateLimiter` | Fixed window |
| `SlidingWindowRateLimiter` | Sliding window (segment-based) |
| `TokenBucketRateLimiter` | Token bucket |
| `ConcurrencyLimiter` | Caps concurrent in-flight requests; the docs describe it as a `RateLimiter` implementation but note it *"limits only the number of concurrent requests and doesn't cap the number of requests in a time period"* |
| `PartitionedRateLimiter<TResource>` | Combines any of the above, keyed by tenant/user/IP |

### Minimal fixed-window example

```csharp
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 10;
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

var app = builder.Build();
app.UseRateLimiter();

app.MapGet("/api/search", () => "ok")
   .RequireRateLimiting("fixed");

app.Run();
```

### Per-user / per-IP partitioning

Straight from the Microsoft docs — a global limiter partitioned by user identity:

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: httpContext.User.Identity?.Name ?? httpContext.Request.Headers.Host.ToString(),
            factory: _ => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 10,
                QueueLimit = 0,
                Window = TimeSpan.FromMinutes(1)
            }));
});
```

> The docs warn explicitly: *"Creating partitions with user input makes the app vulnerable to Denial of Service (DoS) Attacks"* — e.g., an attacker spoofing IPs to blow up memory by creating millions of partitions. Bound partitions or rate-limit partition creation itself.

### Token bucket example

```csharp
builder.Services.AddRateLimiter(_ => _
    .AddTokenBucketLimiter("token", options =>
    {
        options.TokenLimit = 100;
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 10;
        options.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
        options.TokensPerPeriod = 20;
        options.AutoReplenishment = true;
    }));
```

With `AutoReplenishment = true`, an internal timer refills tokens every `ReplenishmentPeriod`. With `false`, call `TryReplenish()` manually — useful in tests.

### Responding well to a rejection

Set `OnRejected` to return a useful response with a `Retry-After` header:

```csharp
options.OnRejected = async (context, cancellationToken) =>
{
    if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
    {
        context.HttpContext.Response.Headers.RetryAfter =
            ((int)retryAfter.TotalSeconds).ToString();
    }

    context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
    await context.HttpContext.Response.WriteAsync("Rate limit exceeded.", cancellationToken);
};
```

## Distributed rate limiting

The `System.Threading.RateLimiting` primitives are **in-process**. With N app instances, each has its own counter — real effective limit is `N × limit`. Options:

### Option 1 — Pin by partition key at the LB

Use consistent hashing at the LB so all requests from one key land on one instance. Now the in-process limiter is authoritative for that key. Trade-off: uneven load and fragile on scale-out (see [Load Balancing](04-load-balancing.md)).

### Option 2 — Centralized counter (Redis)

Every instance increments a shared counter. Common implementations:

- **Fixed-window counter**: `INCR key` then `EXPIRE key 60`. Atomic via Lua to avoid a race between INCR and EXPIRE.
- **Sliding-window counter**: two counters (current window, previous window), weighted.
- **Token bucket**: sorted-set of timestamps or Lua-implemented bucket with token and last-refill fields.

Redis is the de-facto solution — sub-millisecond RTT inside a region makes it cheap enough per request. Battle-tested libraries: `redis-rate-limiter` (Node), `sharding-repository-redis` for .NET, or write your own 15-line Lua script.

### Option 3 — Gateway / sidecar rate limiting

Push the concern to a dedicated tier:

- **Envoy** with a global rate-limit service (Lyft's `ratelimit`).
- **Kong**, **NGINX Plus**, **Azure API Management**, **AWS API Gateway** all have per-key rate limiting built in.
- Service mesh (Istio) can rate-limit at the sidecar.

This keeps rate limiting out of your business code and allows per-tenant SLA configuration centrally.

## Headers and protocol hygiene

Return predictable headers so clients can self-throttle:

| Header | What |
|---|---|
| `X-RateLimit-Limit` | Requests allowed per window |
| `X-RateLimit-Remaining` | Requests remaining in the current window |
| `X-RateLimit-Reset` | Epoch seconds when the window resets |
| `Retry-After` | Seconds (or HTTP date) after which to retry — standard on 429 and 503 |

`Retry-After` is an IETF-standard header defined in [RFC 9110 §10.2.3](https://datatracker.ietf.org/doc/html/rfc9110#section-10.2.3). The `X-RateLimit-*` headers above are **de-facto conventions, not an IETF standard** — the `draft-ietf-httpapi-ratelimit-headers` effort (which proposes standardized `RateLimit` / `RateLimit-Policy` fields, not the `X-` variants) expired in September 2025 without becoming an RFC. Use them, but don't rely on a client library honoring any specific name.

## Pitfalls

- **In-process limiting with many instances** — see above; real cap is N × limit.
- **No jitter on retries** — `Retry-After: 60` returned to 10 k clients causes a thundering herd at t+60. Clients should add jitter, but don't rely on it.
- **Partitioning on unbounded keys** — an attacker spoofing IPs can blow up memory. Cap distinct partitions or use a bounded backing store.
- **Rate-limiting after auth** — DoS-ing the auth service is trivial. Rate-limit at the edge (unauth'd) with a looser limit, then tighter per-user.
- **No back-pressure for bursty clients** — a pure reject wastes effort. Queueing a few (bounded) can smooth legitimate bursts; the ASP.NET Core limiter supports this via `QueueLimit`.
- **Treating rate limiting as DDoS protection** — it isn't. Use a WAF/CDN for L7 DDoS, see Microsoft's explicit note above.

---

[← Previous: Caching Strategies](05-caching-strategies.md) | [Next: Case Study: URL Shortener →](07-case-study-url-shortener.md) | [Back to index](README.md)
