# Bulkhead Pattern

Named after the watertight compartments in a ship's hull: if one fills with water, the rest stay afloat. Applied to software: **isolate resources so that a failure in one dependency cannot consume all of the caller's capacity**. Without a bulkhead, one slow downstream backs up your thread pool or HTTP connections and your whole service degrades — even for callers talking to completely healthy peers.

## The failure mode without a bulkhead

A single ASP.NET Core host has finite resources: thread-pool threads, HTTP connection slots, sockets, memory. Suppose your service calls two downstreams, `A` and `B`:

1. `B` goes slow — p99 climbs from 50 ms to 10 s.
2. Requests waiting on `B` hold their threads and connections.
3. New requests arrive — to `A` *and* `B`.
4. The thread pool saturates with requests stuck on `B`.
5. Requests to `A` now also queue behind nothing, starved for threads.
6. The whole service times out, not just the slice that hit `B`.

A bulkhead breaks this by capping the concurrent calls allowed *to each downstream*. When `B`'s bulkhead is full, new calls to `B` fail fast — leaving room for `A`.

## Two flavors

| Type | Isolation | Cost | Used for |
|---|---|---|---|
| **Thread-pool bulkhead** | Dedicated thread pool per dependency | Threads, context switching | CPU-bound or legacy blocking I/O |
| **Semaphore bulkhead** | Semaphore with permit + queue limits | Almost free | Async I/O (the .NET default) |

In modern async .NET, thread-pool bulkheads are rarely the right tool — async calls do not hold a thread while awaiting I/O. Semaphore bulkheads are the standard choice.

## Polly v8: `AddConcurrencyLimiter`

Polly v7 had a distinct `Bulkhead` policy. **Polly v8 removed the named bulkhead strategy** and replaced it with the concurrency limiter from `System.Threading.RateLimiting`, exposed via the `Polly.RateLimiting` package.

From the Polly v8 migration guide: *"In v8, it's not separately exposed because it's essentially a specialized type of rate limiter: the `ConcurrencyLimiter`."*

```csharp
using Polly;
using Polly.RateLimiting;
using System.Threading.RateLimiting;

var pipeline = new ResiliencePipelineBuilder()
    .AddConcurrencyLimiter(
        permitLimit: 20,   // at most 20 concurrent executions
        queueLimit: 5)     // up to 5 more can wait; beyond that, reject
    .Build();

// Or with full options for partitioned / sliding-window variants:
var advanced = new ResiliencePipelineBuilder()
    .AddRateLimiter(new RateLimiterStrategyOptions
    {
        RateLimiter = args => new ValueTask<RateLimitLease>(
            new ConcurrencyLimiter(new ConcurrencyLimiterOptions
            {
                PermitLimit = 20,
                QueueLimit = 5,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst
            }).AcquireAsync(1, args.Context.CancellationToken).AsTask().Result)
    })
    .Build();
```

When the limit is reached and the queue is full, the strategy throws `RateLimiterRejectedException`.

## Sizing considerations

There is no universal "right" permit count. Rough method:

1. Measure the downstream's **p99 latency** under expected load.
2. Multiply by your target **requests per second** to that peer.
3. Add 20–30% headroom — this is your bulkhead size.

Example: you expect 100 rps to `payments-api` with p99 of 200 ms → ~20 concurrent calls at steady state → set `PermitLimit` around 25–30.

Too small: healthy traffic gets rejected. Too large: bulkhead does not actually protect you during an incident. Revisit the number after every incident that hit a rate-limiter rejection.

## In the standard HTTP resilience handler

`AddStandardResilienceHandler` already includes a rate limiter as its outermost strategy, with default `PermitLimit = 1000`, `QueueLimit = 0` (verified against Microsoft Learn). For most per-client isolation needs, the standard handler is enough; reach for a custom pipeline only when you need different limits per downstream route.

## When NOT to use a bulkhead

- **Inside a tight retry loop that the bulkhead cannot see.** A loop retrying 100 times inside a single `ExecuteAsync` still holds one permit the whole time — the bulkhead only counts outer executions.
- **In a pure pub/sub consumer where back-pressure comes from the broker.** Kafka / Service Bus already bound concurrency via partitions and prefetch.
- **As a substitute for a real queue.** If requests are legitimately bursty, a bulkhead will reject them — you probably want a background worker with a persistent queue instead.

> Bulkhead, circuit breaker, and timeout are complementary. Bulkhead caps *how many* concurrent calls; timeout caps *how long* each one runs; circuit breaker caps *for how long* we stop calling at all.

---

[← Previous: Circuit Breaker](02-circuit-breaker.md) | [Next: Timeouts →](04-timeouts.md) | [Back to index](README.md)
