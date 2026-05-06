# .NET Senior Study Guide

A comprehensive, open-source study guide covering the core topics a **senior .NET / C# developer** should know — from language fundamentals to distributed systems and production reliability.

## Who is this for?

- Developers preparing for **.NET senior/lead positions**
- Mid-level developers looking to **level up** their knowledge
- Anyone who wants a structured, no-fluff reference on the .NET ecosystem

## What's inside?

Every topic includes **concise explanations**, **code examples**, and **practical tips** — written the way you'd explain it to a colleague, not like a textbook. Sections are ordered from the most basic and essential topics to the most advanced, so the guide reads like a learning path.

> All content is written in **English** with code examples in C#/.NET.

## Topics

### [01 - C# Fundamentals](01-csharp-fundamentals/README.md)
.NET ecosystem, namespaces, CLR/IL, numeric types, modifiers, generics, delegates, and events.

### [02 - Exceptions](02-exceptions/README.md)
Exception hierarchy, try/catch/finally, custom exceptions, exception filters (`when`), `ExceptionDispatchInfo`, `AggregateException`, async exceptions, performance.

### [03 - Collections and LINQ](03-collections-and-linq/README.md)
IEnumerable, IQueryable, ICollection, IList, yield return.

### [04 - Computer Architecture](04-computer-architecture/README.md)
CPU internals (fetch-decode-execute, pipeline, branch prediction), cores and threads (SMT/Hyper-Threading, OS scheduling, .NET mapping), memory hierarchy (L1/L2/L3 cache, locality, cache lines).

### [05 - Memory and Performance](05-memory-and-performance/README.md)
Stack/Heap, Garbage Collector, memory optimization, memory leaks, structs vs classes.

### [06 - Concurrency and Parallelism](06-concurrency-and-parallelism/README.md)
Parallel, Task, async/await, race conditions, deadlocks, SemaphoreSlim, locks in depth, the thread pool (work stealing, hill climbing, starvation), Channels, concurrent collections, `TaskStatus` lifecycle.

### [07 - Algorithms and Data Structures](07-algorithms-and-data-structures/README.md)
Big O, sorting, searching, recursion, trees, graphs, dynamic programming, greedy — all with C# examples and complexity analysis.

### [08 - Architecture and Patterns](08-architecture-and-patterns/README.md)
SOLID, Design Patterns, Clean Architecture, CQRS, Microservices, Messaging.

### [09 - HTTP and Web](09-http-and-web/README.md)
HTTP semantics, MIME types, REST API design, web security.

### [10 - ASP.NET Core](10-aspnet-core/README.md)
DI, lifetimes, OAuth, resilience, middleware, caching, Minimal APIs, SignalR.

### [11 - Data Access](11-data-access/README.md)
ORM vs Micro ORM vs ADO.NET, Entity Framework, databases, query optimization, sharding, replication, CDC, data modeling.

### [12 - Security](12-security/README.md)
OWASP Top 10, authentication (Cookies/JWT/OIDC), authorization (RBAC/ABAC/policy), secrets management, threat modeling, supply chain, cryptography.

### [13 - Testing](13-testing/README.md)
Testing pyramid, mocking (Moq), best practices, stress/load testing (k6, NBomber).

### [14 - Git](14-git/README.md)
Object model (DAG, blobs/trees/commits), branching strategies, rebase vs merge, worktree, stash/cherry-pick/reflog, code review workflows, hooks, recovery from dangerous operations.

### [15 - DevOps](15-devops/README.md)
CI/CD, FaaS, Docker, Kubernetes, Terraform, Azure Pipelines.

### [16 - Cloud](16-cloud/README.md)
AWS (IAM, VPC, SQS, S3), Azure (App Service, AKS, Service Bus, Key Vault).

### [17 - Observability](17-observability/README.md)
Three pillars (logs/metrics/traces), structured logging, .NET Metrics API, OpenTelemetry, correlation IDs, SLI/SLO/SLA, alerting.

### [18 - Reliability and SRE](18-reliability-and-sre/README.md)
Retries/backoff/jitter, circuit breaker, bulkhead, timeouts, chaos engineering, incident response, postmortems, error budgets.

### [19 - Distributed Systems](19-distributed-systems/README.md)
CAP theorem, consistency models, idempotency, outbox pattern, saga, distributed transactions, consensus (Raft/Paxos), clocks and ordering.

### [20 - System Design](20-system-design/README.md)
Interview framework, capacity estimation, scaling, load balancing, caching, rate limiting, case studies (URL shortener, news feed, chat).

### [21 - Frontend](21-frontend/README.md)
Browser rendering, DOM, CDN, iframes, Web Storage, Core Web Vitals, PWA/Service Workers, SPA/SSR/SSG, accessibility, plus Angular (Promises/Observables, RxJS, performance, SignalR).

### [22 - AI](22-ai/README.md)
Tensors, embeddings, OpenAI API, prompt engineering, Semantic Kernel, RAG, MCP, architecture scenarios.

### [23 - Soft Skills](23-soft-skills/README.md)
Requirements, stakeholders, elicitation, user stories, communication, technology selection.

### [24 - Self-Assessment](24-self-assessment/README.md)
Test your knowledge — questions with hidden answers.

## How to use

Pick any section and start reading — each topic is self-contained. Use the navigation links at the bottom of each file to move between related topics.

You can also clone the repo and browse it locally, or read it directly on GitHub.

## Contributing

Found an error or want to add a topic? Feel free to open an issue or submit a PR.

## License

MIT
