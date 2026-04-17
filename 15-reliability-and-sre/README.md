# 15 - Reliability and SRE

Reliability is not a bolt-on feature — it is what keeps a system useful under load, partial failure, and bad deploys. This section covers the techniques seniors are expected to apply (retries, circuit breakers, bulkheads, timeouts) and the operational disciplines of SRE (chaos engineering, incident response, postmortems, error budgets). Examples use Polly v8 and `Microsoft.Extensions.Http.Resilience` for .NET, plus Google SRE and AWS references for the operational practices.

## Contents

1. [Retries, Backoff and Jitter](01-retries-backoff-jitter.md) - When retries help vs hurt, exponential backoff, jitter strategies (AWS), idempotency, Polly v8 retry, retry storms
2. [Circuit Breaker](02-circuit-breaker.md) - State machine (Closed/Open/HalfOpen/Isolated), Polly v8 `CircuitBreakerStrategyOptions`, the standard resilience handler, alternatives
3. [Bulkhead Pattern](03-bulkhead-pattern.md) - Failure isolation, concurrency limiting, Polly v8 `AddConcurrencyLimiter`, sizing
4. [Timeouts](04-timeouts.md) - Why every call needs one, `HttpClient.Timeout` vs `CancellationToken`, gRPC deadlines, cascading timeout rule
5. [Chaos Engineering](05-chaos-engineering.md) - Principles of Chaos, steady-state hypothesis, blast radius, game days, AWS FIS
6. [Incident Response](06-incident-response.md) - Incident Command System (IC / Ops / Comms / Planning), severity levels, runbooks, communication cadence
7. [Postmortems](07-postmortems.md) - Blameless culture, root cause vs contributing factors, 5 Whys pitfalls, action items (Google SRE workbook)
8. [Error Budgets](08-error-budgets.md) - Definition, burn rate, relationship with SLOs, feature-freeze policy

---

[Back to index](../README.md)
