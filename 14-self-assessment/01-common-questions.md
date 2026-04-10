# Common Questions

> **How to use this file:** read each question, think about your answer, then click to reveal. Try to answer from memory after studying the other sections — this is your self-assessment.

---

### 1. Why can the Singleton pattern be considered an anti-pattern?

<details>
<summary>Reveal answer</summary>

Singleton introduces **hidden global state**, increases coupling, and makes automated testing difficult.

- Violates the **Dependency Inversion Principle** (DIP)
- Hides real dependencies
- Rigid lifecycle
- Can cause **concurrency** issues in multithreaded environments
- Can cause **memory leaks** (references that are never released)

**Solution**: use DI with a Singleton lifecycle controlled by the container.

</details>

---

### 2. How do you share resources between requests in ASP.NET Core?

<details>
<summary>Reveal answer</summary>

ASP.NET Core is **stateless** — data shared between requests must live outside the request scope:

| Mechanism | Use |
|-----------|-----|
| `IMemoryCache` | In-memory cache, thread-safe, with expiration |
| `IDistributedCache` (Redis) | Multiple instances, persistent |
| Database | Data that needs to survive a restart |
| Static variables | **Discouraged** — breaks DI, hinders testing, memory leaks |

</details>

---

### 3. What is the difference between an interface and an abstract class?

<details>
<summary>Reveal answer</summary>

| Aspect | Interface | Abstract Class |
|--------|-----------|---------------|
| Inheritance | Multiple (several interfaces) | Single (one base class) |
| Implementation | Signature only (up to C# 7) | Can have concrete methods |
| Constructor | None | Can have one |
| Fields | None | Can have them |
| Default methods | Yes (C# 8+) | Yes |
| When to use | Contract / capability | Base model / hierarchy |

</details>

---

### 4. What makes a SQL query slow?

<details>
<summary>Reveal answer</summary>

- Lack of proper indexes
- Functions on columns (non-sargable)
- Poorly planned JOINs
- Large volume without pagination
- Unnecessary SELECT *
- Outdated statistics
- Locks and concurrency

Deep dive: [Query Optimization](../07-data-access/04-query-optimization.md)

</details>

---

### 5. What is the difference between Lazy Loading and Eager Loading?

<details>
<summary>Reveal answer</summary>

- **Lazy**: loads on demand (N+1 risk)
- **Eager**: loads everything together with `.Include()` (more predictable)
- **Recommendation**: avoid Lazy Loading in most scenarios

Deep dive: [Entity Framework](../07-data-access/02-entity-framework.md)

</details>

---

### 6. Explain the difference between Singleton, Scoped, and Transient lifetimes.

<details>
<summary>Reveal answer</summary>

| Lifetime | Instance created | Use case |
|----------|-----------------|----------|
| **Singleton** | Once per application | Shared state, caches, configuration |
| **Scoped** | Once per request | DbContext, per-request services |
| **Transient** | Every time it's requested | Lightweight, stateless services |

> Injecting a Scoped service into a Singleton causes a **captive dependency** — the Scoped instance lives forever inside the Singleton.

Deep dive: [Service Lifetimes](../06-aspnet-core/02-service-lifetimes.md)

</details>

---

### 7. What is Parameter Sniffing?

<details>
<summary>Reveal answer</summary>

SQL Server caches the execution plan based on the first parameters. If future parameters are very different, the cached plan can be terrible for the new values.

Classic symptom: query runs fast in SSMS and **hangs** when running from the application.

**Solution**: `OPTION (RECOMPILE)` when necessary.

Deep dive: [Query Optimization](../07-data-access/04-query-optimization.md)

</details>

---

### 8. Name at least 5 techniques to optimize performance in a .NET application.

<details>
<summary>Reveal answer</summary>

1. **Cache** (in-memory or distributed)
2. **Asynchronous jobs** (queues for heavy operations)
3. **Monitoring** with production metrics (Application Insights, Grafana)
4. **CDN** on the front-end
5. **CQRS** (separate read and write databases)
6. **ElasticSearch** for text search
7. **Connection pooling** and query optimization
8. **Output caching** for frequently requested endpoints

</details>

---

### 9. What is the difference between a Race Condition and a Deadlock?

<details>
<summary>Reveal answer</summary>

- **Race condition**: multiple threads access a shared resource without synchronization — the outcome depends on timing
- **Deadlock**: two or more threads wait forever for each other to release a lock

Deep dive: [Concurrency and Parallelism](../04-concurrency-and-parallelism/)

</details>

---

### 10. What patterns would you use to make an API resilient?

<details>
<summary>Reveal answer</summary>

| Pattern | Purpose |
|---------|---------|
| **Retry** | Automatic retries with exponential backoff |
| **Timeout** | Fail fast instead of waiting forever |
| **Circuit Breaker** | Stop calling a failing service temporarily |
| **Fallback** | Return cached/default data when the service is down |
| **Bulkhead** | Isolate resources so one failure doesn't bring down everything |
| **Rate Limiting** | Protect the service from excessive requests |

Implementation: HttpClientFactory + Polly.

Deep dive: [API Resilience](../06-aspnet-core/04-api-resilience.md)

</details>

---

### 11. What is the difference between `Equals()` and `==` in C#?

<details>
<summary>Reveal answer</summary>

| Scenario | `==` | `Equals()` |
|----------|------|------------|
| Value types | Value | Value |
| Reference types (default) | Reference | Reference |
| string | Content (overridden) | Content |
| Record | Content (overridden) | Content |
| With Equals override | Reference (unless you also overload `==`) | Custom content |

Deep dive: [Equals vs ==](../01-csharp-fundamentals/06-equals-vs-operator.md)

</details>

---

### 12. Explain Clean Architecture and its dependency rule.

<details>
<summary>Reveal answer</summary>

Code organized in **concentric layers** where dependencies point inward:

```
Presentation → Application → Domain ← Infrastructure
```

- **Domain** (center): entities, value objects, repository interfaces. Depends on nothing.
- **Application**: use cases, DTOs, orchestration. Depends only on Domain.
- **Infrastructure**: EF Core, external APIs, implementations. Implements Domain interfaces.
- **Presentation**: controllers, API. References Application + Infrastructure for DI.

Deep dive: [Clean Architecture](../05-architecture-and-patterns/07-clean-architecture.md)

</details>

---

### 13. When would you use microservices vs a modular monolith?

<details>
<summary>Reveal answer</summary>

| Scenario | Recommendation |
|----------|----------------|
| New project, small team | Monolith or Modular Monolith |
| Simple domain, CRUD | Monolith |
| Complex domain, medium team | Modular Monolith |
| Multiple teams, high scale | Microservices |

> "If you can't build a well-made monolith, you won't be able to build microservices." — Simon Brown

The **Modular Monolith** is the best starting point for most projects — isolated modules that can be extracted to microservices later if needed.

Deep dive: [Microservices](../05-architecture-and-patterns/09-microservices.md)

</details>

---

### 14. How does the ASP.NET Core middleware pipeline work?

<details>
<summary>Reveal answer</summary>

Middlewares form a **chain of responsibility** for HTTP requests. Each can process the request, pass it to the next, or short-circuit.

```
Request → [ExceptionHandler] → [HTTPS] → [Routing] → [Auth] → [Endpoint]
Response ← [ExceptionHandler] ← [HTTPS] ← [Routing] ← [Auth] ←
```

**Order matters** — `UseAuthentication` must come before `UseAuthorization`. Getting the order wrong is a common bug.

Deep dive: [Middleware](../06-aspnet-core/05-middleware.md)

</details>

---

### 15. What is CQRS and when would you use it?

<details>
<summary>Reveal answer</summary>

Separating **read** (Query) and **write** (Command) operations into different models.

| Level | Description | Complexity |
|-------|-------------|------------|
| 1 | Separate handlers for reads and writes | Low |
| 2 | Different models for read and write | Medium |
| 3 | Different databases (write DB + read DB) | High |
| 4 | Event Sourcing + projections | Very high |

Start at level 1. Only move up if there is a real need. Don't use for simple CRUDs.

Deep dive: [CQRS](../05-architecture-and-patterns/08-cqrs.md)

</details>

---

[Back to index](README.md)
