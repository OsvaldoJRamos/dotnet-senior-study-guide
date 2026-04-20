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

### [02 - Collections and LINQ](02-collections-and-linq/README.md)
IEnumerable, IQueryable, ICollection, IList, yield return.

### [03 - Computer Architecture](03-computer-architecture/README.md)
CPU internals (fetch-decode-execute, pipeline, branch prediction), cores and threads (SMT/Hyper-Threading, OS scheduling, .NET mapping), memory hierarchy (L1/L2/L3 cache, locality, cache lines).

### [04 - Memory and Performance](04-memory-and-performance/README.md)
Stack/Heap, Garbage Collector, memory optimization, memory leaks, structs vs classes.

### [05 - Concurrency and Parallelism](05-concurrency-and-parallelism/README.md)
Parallel, Task, async/await, race conditions, deadlocks, SemaphoreSlim.

### [06 - Algorithms and Data Structures](06-algorithms-and-data-structures/README.md)
Big O, sorting, searching, recursion, trees, graphs, dynamic programming, greedy — all with C# examples and complexity analysis.

### [07 - Architecture and Patterns](07-architecture-and-patterns/README.md)
SOLID, Design Patterns, Clean Architecture, CQRS, Microservices, Messaging.

### [08 - HTTP and Web](08-http-and-web/README.md)
HTTP semantics, MIME types, REST API design, web security.

### [09 - ASP.NET Core](09-aspnet-core/README.md)
DI, lifetimes, OAuth, resilience, middleware, caching, Minimal APIs, SignalR.

### [10 - Data Access](10-data-access/README.md)
ORM vs Micro ORM vs ADO.NET, Entity Framework, databases, query optimization, sharding, replication, CDC, data modeling.

### [11 - Security](11-security/README.md)
OWASP Top 10, authentication (Cookies/JWT/OIDC), authorization (RBAC/ABAC/policy), secrets management, threat modeling, supply chain, cryptography.

### [12 - Testing](12-testing/README.md)
Testing pyramid, mocking (Moq), best practices, stress/load testing (k6, NBomber).

### [13 - DevOps](13-devops/README.md)
CI/CD, FaaS, Docker, Kubernetes, Terraform, Azure Pipelines.

### [14 - Cloud](14-cloud/README.md)
AWS (IAM, VPC, SQS, S3), Azure (App Service, AKS, Service Bus, Key Vault).

### [15 - Observability](15-observability/README.md)
Three pillars (logs/metrics/traces), structured logging, .NET Metrics API, OpenTelemetry, correlation IDs, SLI/SLO/SLA, alerting.

### [16 - Reliability and SRE](16-reliability-and-sre/README.md)
Retries/backoff/jitter, circuit breaker, bulkhead, timeouts, chaos engineering, incident response, postmortems, error budgets.

### [17 - Distributed Systems](17-distributed-systems/README.md)
CAP theorem, consistency models, idempotency, outbox pattern, saga, distributed transactions, consensus (Raft/Paxos), clocks and ordering.

### [18 - System Design](18-system-design/README.md)
Interview framework, capacity estimation, scaling, load balancing, caching, rate limiting, case studies (URL shortener, news feed, chat).

### [19 - Frontend](19-frontend/README.md)
Browser rendering, DOM, CDN, iframes, Web Storage, Core Web Vitals, PWA/Service Workers, SPA/SSR/SSG, accessibility, plus Angular (Promises/Observables, RxJS, performance, SignalR).

### [20 - AI](20-ai/README.md)
Tensors, embeddings, OpenAI API, prompt engineering, Semantic Kernel, RAG, MCP, architecture scenarios.

### [21 - Soft Skills](21-soft-skills/README.md)
Requirements, stakeholders, elicitation, user stories, communication, technology selection.

### [22 - Self-Assessment](22-self-assessment/README.md)
Test your knowledge — questions with hidden answers.

## How to use

Pick any section and start reading — each topic is self-contained. Use the navigation links at the bottom of each file to move between related topics.

You can also clone the repo and browse it locally, or read it directly on GitHub.

## Contributing

Found an error or want to add a topic? Feel free to open an issue or submit a PR.

## License

MIT
