# Common .NET Interview Questions

## 1. Why can the Singleton pattern be considered an anti-pattern?

Singleton introduces **hidden global state**, increases coupling, and makes automated testing difficult.

- Violates the **Dependency Inversion Principle** (DIP)
- Hides real dependencies
- Rigid lifecycle
- Can cause **concurrency** issues in multithreaded environments
- Can cause **memory leaks** (references that are never released)

**Solution**: use DI with a Singleton lifecycle controlled by the container.

## 2. How to share resources between requests in ASP.NET Core?

ASP.NET Core is **stateless** - data shared between requests must live outside the request scope:

| Mechanism | Use |
|-----------|-----|
| `IMemoryCache` | In-memory cache, thread-safe, with expiration |
| `IDistributedCache` (Redis) | Multiple instances, persistent |
| Database | Data that needs to survive a restart |
| Static variables | **Discouraged** - breaks DI, hinders testing, memory leaks |

## 3. Difference between interface and abstract class

| Aspect | Interface | Abstract Class |
|--------|-----------|---------------|
| Inheritance | Multiple (several interfaces) | Single (one base class) |
| Implementation | Signature only (up to C# 7) | Can have concrete methods |
| Constructor | None | Can have one |
| Fields | None | Can have them |
| Default methods | Yes (C# 8+) | Yes |
| When to use | Contract / capability | Base model / hierarchy |

## 4. What makes a SQL query slow?

- Lack of proper indexes
- Functions on columns (non-sargable)
- Poorly planned JOINs
- Large volume without pagination
- Unnecessary SELECT *
- Outdated statistics
- Locks and concurrency

(See details in [Query Optimization](../07-acesso-a-dados/04-otimizacao-de-queries.md))

## 5. Lazy Loading vs Eager Loading

- **Lazy**: loads on demand (N+1 risk)
- **Eager**: loads everything together with Include (more predictable)
- **Recommendation**: avoid Lazy Loading in most scenarios

(See details in [Entity Framework](../07-acesso-a-dados/02-entity-framework.md))

## 6. Singleton vs Scoped vs Transient

(See details in [Service Lifetimes](../06-aspnet-core/02-service-lifetimes.md))

## 7. What is Parameter Sniffing?

SQL Server caches the execution plan based on the first parameters. If future parameters are very different, the plan can be terrible.

Solution: `OPTION (RECOMPILE)` when necessary.

(See details in [Query Optimization](../07-acesso-a-dados/04-otimizacao-de-queries.md))

## 8. Performance optimization techniques

1. **Cache** (in-memory or distributed)
2. **Asynchronous jobs** (queues for heavy operations)
3. **Monitoring** with production metrics (Application Insights, Grafana)
4. **CDN** on the front-end
5. **CQRS** (separate read and write databases)
6. **ElasticSearch** for text search

## 9. Race Conditions vs Deadlocks

- **Race condition**: threads access a shared resource without synchronization
- **Deadlock**: threads wait forever for each other

(See details in [Concurrency and Parallelism](../04-concorrencia-e-paralelismo/))

## 10. API Resilience

Patterns: Retry, Timeout, Circuit Breaker, Fallback, Bulkhead, Rate Limiting.

Implementation: HttpClientFactory + Polly.

(See details in [API Resilience](../06-aspnet-core/04-resiliencia-de-apis.md))

---

[Back to index](README.md)
