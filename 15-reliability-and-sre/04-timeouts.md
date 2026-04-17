# Timeouts

Every remote call must have a timeout. Without one, a slow peer can hold your thread / connection / socket indefinitely, and the rest of your resilience machinery (retry, circuit breaker, bulkhead) is blind. A default timeout of "hang forever" is a bug.

## The cascading timeout rule

The timeout of a caller must be **shorter** than the timeout of the thing calling it. Otherwise the caller gives up before the callee has a chance to respond — wasted work — or the caller waits after the callee has already failed — wasted time.

```text
user → API gateway → service A → service B → database

timeouts (from user inward):
  gateway      : 30 s
  service A    : 20 s   (< 30 s)
  service B    : 10 s   (< 20 s)
  database     :  3 s   (< 10 s)
```

This is **deadline propagation**: instead of each hop picking an independent timeout, the caller passes the *remaining time* to the callee. gRPC and some RPC frameworks propagate deadlines automatically; plain HTTP does not.

## `HttpClient.Timeout` — what it actually does

From the official .NET API reference:

- **Default: 100 seconds (100,000 ms).** Far too long for an interactive request path.
- Applies to **every request** issued through that `HttpClient` instance.
- Setting it to `Timeout.InfiniteTimeSpan` disables it — never do this in a request-response path.
- A DNS lookup on an unresolvable host can still take up to 15 seconds even if `Timeout` is lower.

```csharp
var http = new HttpClient
{
    Timeout = TimeSpan.FromSeconds(5) // per-request, fail fast
};
```

## `HttpClient.Timeout` vs `CancellationToken`

They are *both* timeouts, and whichever fires first wins. Microsoft's docs state: *"You may also set different timeouts for individual requests using a CancellationTokenSource on a task. Note that only the shorter of the two timeouts will apply."*

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2));
var response = await http.GetAsync(url, cts.Token); // 2 s beats HttpClient.Timeout of 5 s
```

Practical rule:

- Configure a **conservative `HttpClient.Timeout`** as the outer bound (e.g. 30 s — catches forgotten cancellation).
- Pass a **request-scoped `CancellationToken`** derived from the caller's deadline for the actual per-request limit.

When a request is cancelled through the token, you get `TaskCanceledException` (or `OperationCanceledException`). When it hits `HttpClient.Timeout`, you also get `TaskCanceledException` — the inner exception is sometimes `TimeoutException`, sometimes not. Do not rely on exception type alone to distinguish; check `CancellationToken.IsCancellationRequested`.

## Polly v8 timeout strategy

Polly's timeout strategy wraps the inner `CancellationToken` and throws `TimeoutRejectedException` when the limit expires *and* the callback has acknowledged cancellation. Critical: your code must respect the token.

```csharp
using Polly;
using Polly.Timeout;

var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(3) // default 30 s
    })
    .Build();

await pipeline.ExecuteAsync(
    async ct => await http.GetAsync(url, ct), // MUST pass ct to the inner call
    cancellationToken);
```

If you ignore the inner token (e.g. `http.GetAsync(url)` with no `ct`), the strategy cannot cancel anything and `TimeoutRejectedException` will not fire until the HTTP call returns on its own.

## gRPC deadlines

gRPC distinguishes **deadline** (an absolute point in time) from **timeout** (a relative duration). From the official gRPC blog:

> *"Some language APIs work in terms of a deadline, a fixed point in time by which the RPC should complete. Others use a timeout, a duration of time after which the RPC times out."*

In .NET gRPC clients, set the deadline on the call options:

```csharp
var response = await client.GetOrderAsync(
    new GetOrderRequest { Id = orderId },
    deadline: DateTime.UtcNow.AddSeconds(3));
```

Deadlines propagate through gRPC call chains: if service A calls service B with `deadline: now + 3s` and 500 ms elapses before A forwards the call, B sees `deadline: now + 2.5s`. When the deadline expires, the server receives a signal to stop work; the client receives `StatusCode.DeadlineExceeded`.

> Important asymmetry the gRPC blog calls out: *"both the client and server make their own independent and local determination about whether the RPC was successful."* A response sent by the server **can arrive after the client's deadline** — the client sees failure, the server recorded success. Idempotency keys matter even with deadlines.

## Common mistakes

- **Single global timeout.** One `HttpClient.Timeout = 100s` for the whole app. Use per-call cancellation tokens aligned with the caller's deadline.
- **Timeout > caller's deadline.** A service whose SLO is 1 s calling a dependency with a 5 s timeout — guaranteed to exceed SLO when the dependency slows.
- **Retry + timeout mis-ordered.** Attempt timeout must be on the inner call; total timeout must wrap the retries. The standard HTTP resilience handler gets this right by default (outer total timeout of 30 s, inner attempt timeout of 10 s).
- **Silently swallowing `TaskCanceledException`.** A timeout that logs nothing is a timeout that cannot be tuned.

---

[← Previous: Bulkhead Pattern](03-bulkhead-pattern.md) | [Next: Chaos Engineering →](05-chaos-engineering.md) | [Back to index](README.md)
