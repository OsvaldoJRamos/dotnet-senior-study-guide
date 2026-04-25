# 07 - Architecture and Patterns

## Contents

1. [SOLID](01-solid.md) - The 5 object-oriented design principles
2. [Design Patterns](02-design-patterns.md) - GoF design patterns
3. [KISS, DRY and YAGNI](03-kiss-dry-yagni.md) - Simplicity principles
4. [Object Calisthenics](04-object-calisthenics.md) - 9 rules for clean code
5. [Tell, Don't Ask](05-tell-dont-ask.md) - Encapsulation principle
6. [SAGA Pattern](06-saga-pattern.md) - Consistency in microservices
7. [Clean Architecture](07-clean-architecture.md) - Layers, dependency rule, solution structure
8. [CQRS](08-cqrs.md) - Command Query Responsibility Segregation, Event Sourcing
9. [Microservices](09-microservices.md) - Microservices vs Monolith vs Modular Monolith
10. [Messaging](10-messaging.md) - RabbitMQ, Kafka, Azure Service Bus
11. [Idempotency and Race Conditions](11-idempotency-and-race-conditions.md) - Preventing duplicate payments with layered defenses
12. [Anti-Patterns](12-anti-patterns.md) - God Class, Anemic Domain Model, Distributed Monolith, Golden Hammer, Big Ball of Mud
13. [Domain-Driven Design](13-ddd.md) - Strategic (bounded contexts, context mapping, event storming) and tactical (entities, value objects, aggregates, domain events)
14. [API Gateway and BFF](14-api-gateway-and-bff.md) - Single entry point vs backend-per-frontend, vs service mesh
15. [Event Sourcing](15-event-sourcing.md) - State as event stream, projections, schema evolution, pairing with CQRS
16. [Application Architectural Patterns](16-app-architectural-patterns.md) - MVC, MVVM, MVP, Layered, Hexagonal (Ports and Adapters)
17. [Design Docs, C4 and ADRs](17-design-docs-c4-adr.md) - C4 model (Simon Brown), ADR template (Nygard), RFC structure
18. [NFR-Driven Architecture](18-nfr-driven-architecture.md) - ISO/IEC 25010, how NFRs drive architecture, PACELC
19. [UML Diagrams](19-uml-diagrams.md) - Use case, sequence, activity, class, state, ER, deployment

## Useful Links

- [Refactoring (refactoring.guru)](https://refactoring.guru/refactoring) — catalog of refactorings, code smells, before/after examples.
- [What is a Design Pattern? (refactoring.guru)](https://refactoring.guru/design-patterns/what-is-pattern) — primer on what patterns are and when to use them.
- [DDD Starter Modelling Process (GitHub)](https://github.com/RenatoAugustoFS/ddd-starter-modelling-process) — practical DDD modeling steps (event storming → context mapping → tactical design).
- [Microservices (Martin Fowler)](https://martinfowler.com/microservices/) — Fowler's hub on microservices, including the original article and follow-ups.
- [Microservices article — Lewis & Fowler 2014](https://martinfowler.com/articles/microservices.html) — the seminal piece defining microservices via nine characteristics; mandatory reading for the architecture interview.
- [Cloud Design Patterns (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/architecture/patterns/) — full Azure catalog of 41 cloud design patterns (Ambassador, Bulkhead, Cache-Aside, CQRS, Saga, Sidecar, Strangler Fig, Valet Key, etc.) mapped to the Well-Architected Framework pillars.
- [Cache-Aside pattern (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) — load-on-demand cache pattern, when it fits, common pitfalls.
- [CQRS pattern (AWS Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/cqrs-pattern.html) — AWS take on CQRS, including pairing with Event Sourcing.
- [Antipatterns (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/) — Azure Architecture Center catalog of architectural antipatterns.

---

[Back to index](../README.md)
