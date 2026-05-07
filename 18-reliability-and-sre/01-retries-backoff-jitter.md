# Retries, Backoff and Jitter

Retries are the simplest resilience pattern — and the most dangerous when misapplied. A retry helps with transient faults (a dropped TCP packet, a momentary GC pause on the peer) and actively harms during overload, because every retry adds load to a system that already cannot cope. The senior answer is never *"just add retries"* — it's *"retries, but with backoff, jitter, idempotency, and a bound on how many."*

## When retries help vs hurt

| Situation | Retry? |
|---|---|
| Network blip, TCP reset, timeout on a healthy server | Yes — transient |
| 503 with `Retry-After` header | Yes — server is asking |
| 502 / 504 behind a load balancer | Usually yes — a different replica may answer |
| 429 Too Many Requests | Respect `Retry-After`; otherwise back off aggressively |
| 400 / 401 / 403 / 404 / 422 | No — semantically wrong, will fail again |
| 500 from a business logic bug | No — not transient |
| Peer is in overload (p99 spiking, queue filling) | **No** — retries make it worse |

> AWS's Builders' Library puts it bluntly: *"When failures are caused by overload, retries that increase load can make matters significantly worse."* A 5-layer stack that each retries 3 times amplifies the load on the bottom tier by 3^5 = 243 times.

## Exponential backoff

Fixed-interval retries synchronize clients into waves. Exponential backoff spreads them out:

```text
delay(attempt) = base * 2^attempt
```

With `base = 100 ms`: attempts sleep 100 ms, 200 ms, 400 ms, 800 ms, ... A `MaxDelay` cap prevents unbounded growth.

Exponential alone is not enough. If 1,000 clients all hit a shared service at the same instant and all start backing off exponentially, they *still* retry in lockstep — the waves just get further apart.

## Jitter

Jitter adds randomness to the delay so clients desynchronize. The AWS Architecture Blog benchmarked three variants:

| Strategy | Formula | Notes |
|---|---|---|
| **Full jitter** | `sleep = random(0, min(cap, base * 2^attempt))` | Lowest total work in AWS's test vs Equal jitter — common default |
| **Equal jitter** | `temp = min(cap, base * 2^attempt); sleep = temp/2 + random(0, temp/2)` | Worst performer among jitter variants; still better than no jitter |
| **Decorrelated jitter** | `sleep = min(cap, random(base, 3 * last_sleep))` | Self-correlates with recent history; close to full jitter |

> Source: AWS, *Exponential Backoff And Jitter* (Marc Brooker). Full jitter is the usual choice unless you have a specific reason to keep a minimum delay.

## Idempotency is a prerequisite

A retry is only safe if the operation is **idempotent** — executing it N times has the same effect as once.

- `GET`, `PUT`, `DELETE` — idempotent by HTTP spec
- `POST` — **not** idempotent; retrying a "create payment" call risks double-charging
- Make `POST` safe with an **idempotency key** (client-generated UUID in a header; server dedupes)

Polly / `Microsoft.Extensions.Http.Resilience` lets you disable retries for unsafe methods:

```csharp
httpClientBuilder.AddStandardResilienceHandler(options =>
{
    // POST, PATCH, PUT, DELETE, CONNECT per RFC 7231 section 4.2.1
    options.Retry.DisableForUnsafeHttpMethods();
});
```

## Polly v8 retry

Polly v8 is a rewrite — the v7 `Policy.Handle<>()` fluent API is a separate legacy package. Native v8 uses options-based `ResiliencePipelineBuilder`:

```csharp
using Polly;
using Polly.Retry;

var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
        MaxRetryAttempts = 3,            // default 3
        BackoffType = DelayBackoffType.Exponential, // default Constant
        Delay = TimeSpan.FromMilliseconds(200),     // default 2s
        UseJitter = true,                // default false — turn on
        MaxDelay = TimeSpan.FromSeconds(10),
        OnRetry = args =>
        {
            // log attempt number, outcome, retry delay
            return default;
        }
    })
    .Build();

await pipeline.ExecuteAsync(async ct => await client.GetAsync(url, ct), ct);
```

Defaults (from the official Polly docs): `MaxRetryAttempts = 3`, `Delay = 2s`, `BackoffType = Constant`, `UseJitter = false`. You almost always want to flip `UseJitter = true` and `BackoffType = Exponential`.

### With the standard HTTP resilience handler

For HTTP specifically, prefer `Microsoft.Extensions.Http.Resilience`'s `AddStandardResilienceHandler` — it wires retry, circuit breaker, total timeout, attempt timeout, and a rate limiter with sensible defaults:

```csharp
services.AddHttpClient<PaymentsClient>()
    .AddStandardResilienceHandler();
```

Microsoft's documented defaults for the retry stage: `MaxRetryAttempts = 3`, `BackoffType = Exponential`, `UseJitter = true`, `Delay = 2s`.

## Retry budgets (token bucket)

Per-call retry limits do not prevent a **retry storm** — the flood of retries when many clients fail simultaneously. AWS's recommended control is a **token bucket**:

- Every successful call refills tokens
- Every retry consumes a token
- When the bucket is empty, retries are rejected locally — no network call

Effect: during a healthy period, retries are free; during an outage they self-throttle. The AWS SDKs have had this built in since 2016. Polly does not ship a named "retry budget" strategy — implement it yourself as a custom strategy, compose with `AddConcurrencyLimiter`, or rely on the AWS SDK's built-in retry mode.

> **When NOT to retry at all:** write operations without idempotency keys, calls inside a transaction where the outer system is already retrying, or when the response carries definitive failure semantics (`422 Unprocessable Entity`). Adding retries "just in case" is how data duplication bugs are born.

---

[Next: Circuit Breaker →](02-circuit-breaker.md) | [Back to index](README.md)
