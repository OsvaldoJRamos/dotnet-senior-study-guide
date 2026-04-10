# API Resilience

## What it is

The ability of an API to continue functioning acceptably even when **failures, slowdowns, load spikes, or unstable dependencies** occur.

It's not about avoiding failures — it's about **failing in a controlled manner** and **recovering quickly**.

## Why it matters

In distributed systems (microservices, external integrations, cloud):

- External dependencies go down
- Networks fail
- Services become slow
- Traffic spikes happen
- Deploys cause instability

Without resilience, a small failure becomes a **domino effect**.

## Resilience patterns

### 1. Retry (automatic retries)

Repeat the call when **transient** failures occur (timeout, 5xx, connection reset).

```csharp
// With Polly
builder.Services.AddHttpClient("api")
    .AddTransientHttpErrorPolicy(p => 
        p.WaitAndRetryAsync(3, attempt => 
            TimeSpan.FromSeconds(Math.Pow(2, attempt)))); // exponential backoff
```

> Retry without control **increases load** and worsens the failure. Always use **exponential backoff**.

### 2. Timeout

Don't wait forever. Every external call should have an explicit timeout.

```csharp
builder.Services.AddHttpClient("api")
    .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(
        TimeSpan.FromSeconds(10)));
```

Prevents stuck threads and pool exhaustion.

### 3. Circuit Breaker

"Shuts off" calls to a service that is failing:

- After N failures -> circuit **opens** (blocks calls)
- After some time -> **half-open** (tests with one call)
- If it works -> circuit **closes** (returns to normal)

```csharp
builder.Services.AddHttpClient("api")
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)));
```

### 4. Fallback

An alternative response when something fails:

- Return **cached data**
- Return a **default value**
- **Degrade** functionality
- **Friendly message**

```csharp
var fallbackPolicy = Policy<HttpResponseMessage>
    .Handle<HttpRequestException>()
    .FallbackAsync(new HttpResponseMessage(HttpStatusCode.OK)
    {
        Content = new StringContent("[]") // returns empty list as fallback
    });
```

### 5. Bulkhead (isolation)

Isolates resources so that the failure of one doesn't bring down all of them:

```csharp
var bulkhead = Policy.BulkheadAsync<HttpResponseMessage>(
    maxParallelization: 10,    // max simultaneous calls
    maxQueuingActions: 20);    // max in the wait queue
```

### 6. Rate Limiting

Limits the rate of requests to protect the service:

```csharp
// .NET 7+
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window = TimeSpan.FromSeconds(10);
        opt.PermitLimit = 100;
    });
});
```

## Typical implementation with HttpClientFactory + Polly

```csharp
builder.Services.AddHttpClient("payment", client =>
{
    client.BaseAddress = new Uri("https://api.payment.com");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3, 
    attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))))
.AddTransientHttpErrorPolicy(p => p.CircuitBreakerAsync(5, 
    TimeSpan.FromSeconds(30)));
```

## Summary

The goal is to avoid the domino effect, protect internal resources, and improve the user experience even when something goes wrong.

---

[← Previous: OAuth 2.0](03-oauth2.md) | [Next: Middleware →](05-middleware.md) | [Back to index](README.md)
