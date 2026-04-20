# Circuit Breaker

A circuit breaker stops a caller from hammering a failing dependency. Retries assume the problem is transient; a circuit breaker assumes *the problem is persistent and sending more traffic makes it worse*. It is the companion to retry — never the alternative.

## The state machine

Polly v8's circuit breaker exposes four states (`CircuitBreakerStrategyOptions`):

| State | Behavior |
|---|---|
| **Closed** | Normal — calls pass through; failures counted in the sampling window |
| **Open** | Short-circuited — calls fail fast with `BrokenCircuitException`, no downstream traffic |
| **HalfOpen** | After `BreakDuration`, one probe is allowed; if it succeeds the circuit closes, if it fails it re-opens |
| **Isolated** | Manually held open (operator action) — calls fail fast with `IsolatedCircuitException` |

```text
       success                       success
  ┌──────────────┐               ┌──────────────┐
  │              ▼               │              ▼
Closed ──failure ratio──▶ Open ──BreakDuration──▶ HalfOpen
  ▲                        ▲                       │
  └────────────────────────┴───────────────────────┘
                        failure
```

## When to trip

Tripping on "N failures in a row" is naive — a single slow endpoint behind a 1,000 rps client would trip on legitimate blips. The senior approach: **failure ratio over a sampling window, gated by minimum throughput**.

Polly v8's default thresholds (verified against the official docs):

| Property | Default |
|---|---|
| `FailureRatio` | 0.1 (10%) |
| `MinimumThroughput` | 100 executions |
| `SamplingDuration` | 30 seconds |
| `BreakDuration` | 5 seconds |

Meaning: the circuit only opens if **both** conditions hold — at least 100 calls landed in the last 30 s *and* more than 10% failed. This prevents a low-traffic path from tripping on a single exception.

## Polly v8 circuit breaker

```csharp
using Polly;
using Polly.CircuitBreaker;

var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,                          // 50% failures
        MinimumThroughput = 10,                      // at least 10 calls
        SamplingDuration = TimeSpan.FromSeconds(10),
        BreakDuration = TimeSpan.FromSeconds(30),
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>(),
        OnOpened = args =>
        {
            // alert, increment metric
            return default;
        },
        OnClosed = args => default,
        OnHalfOpened = args => default
    })
    .Build();
```

Strategy ordering inside a pipeline matters. The usual pattern, outermost → innermost:
**Timeout → Retry → Circuit Breaker → the call**. The breaker should see the *result of* retries, not short-circuit them.

## With the standard HTTP resilience handler

`AddStandardResilienceHandler` already includes a circuit breaker. For a custom stack:

```csharp
services.AddHttpClient<PaymentsClient>()
    .AddResilienceHandler("payments", builder =>
    {
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        });

        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.2,
            MinimumThroughput = 8,
            SamplingDuration = TimeSpan.FromSeconds(10),
            BreakDuration = TimeSpan.FromSeconds(30)
        });

        builder.AddTimeout(TimeSpan.FromSeconds(5));
    });
```

`HttpCircuitBreakerStrategyOptions` and `HttpRetryStrategyOptions` are HTTP-specific variants that understand HTTP status codes (500+, 408, 429) out of the box, per Microsoft's documented behavior.

## When NOT to use a circuit breaker

- **A call you cannot live without.** If the path has no fallback and no sensible degraded mode, an open circuit just converts slow failures into fast failures — the user still sees an error.
- **Shared breaker across unrelated peers.** One slow Kafka broker should not open the breaker for all Kafka calls. Scope by destination.
- **As a replacement for retries.** They solve different problems.

## Alternatives: adaptive concurrency

Circuit breakers are a step function — closed, then open, then closed again. **Adaptive concurrency** (Netflix's concurrency-limits library, Envoy's adaptive concurrency filter) continuously adjusts how many concurrent calls a client allows, based on observed latency. Load drops smoothly as the peer degrades, rather than collapsing to zero.

.NET does not ship an adaptive-concurrency primitive out of the box. The closest analog is `AddConcurrencyLimiter` with tuned permits, or hedging (`AddStandardHedgingHandler`), which issues additional parallel requests (optionally to alternative endpoints) when the original is slow.

> Rule of thumb: circuit breaker protects you from *hammering* a dead peer; concurrency limit protects you from *overwhelming* a slow one. They compose.

---

[← Previous: Retries, Backoff and Jitter](01-retries-backoff-jitter.md) | [Next: Bulkhead Pattern →](03-bulkhead-pattern.md) | [Back to index](README.md)
