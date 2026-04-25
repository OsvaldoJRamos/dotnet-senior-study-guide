# Microservices vs Monolith

## Monolith

A single application with all the code together:

```
┌─────────────────────────┐
│        Monolith          │
│  ┌─────┐ ┌─────┐ ┌────┐│
│  │Users│ │Order│ │Pay ││
│  └─────┘ └─────┘ └────┘│
│        [1 database]      │
└─────────────────────────┘
```

### Advantages

- Simple to develop, test, deploy
- A single codebase
- Simple ACID transactions
- No network latency between modules

### Disadvantages

- Scales everything together (cannot scale just the payment module)
- Deploys everything together (a small change redeploys everything)
- Large teams step on each other's toes
- Single technology stack

## Microservices

Each module is an independent service with its own database:

```
┌────────┐  ┌────────┐  ┌────────┐
│ Users  │  │ Orders │  │Payment │
│ Service│  │ Service│  │Service │
│ [DB 1] │  │ [DB 2] │  │ [DB 3] │
└───┬────┘  └───┬────┘  └───┬────┘
    │           │            │
────┴───────────┴────────────┴──── Message Bus / API Gateway
```

### Advantages

- Independent scaling per service
- Independent deployment
- Each team "owns" its service
- Technology can vary per service

### Disadvantages

- High operational complexity (monitoring, tracing, deployment)
- Distributed transactions (no simple ACID -- requires SAGA)
- Network latency
- Harder debugging

## Modular Monolith (middle ground)

A monolith with **well-isolated modules**. Each module has its own entities and rules, but runs in the same process:

```
┌─────────────────────────────────┐
│         Modular Monolith         │
│  ┌─────────┐ ┌─────────┐ ┌────┐│
│  │ Users   │ │ Orders  │ │Pay ││
│  │ Module  │ │ Module  │ │Mod ││
│  │(schema1)│ │(schema2)│ │(s3)││
│  └─────────┘ └─────────┘ └────┘│
│         [1 database, N schemas]  │
└─────────────────────────────────┘
```

- Communication between modules via **internal interfaces** or **in-process events**
- If needed in the future, extracting a module into a microservice is easier
- **Best option for the majority of projects**

## Communication between services

### Synchronous (request/response)

```
Service A  ──HTTP/gRPC──→  Service B
           ←──response───
```

- REST APIs or gRPC
- Simple, but creates **temporal coupling** (A depends on B being up)

### Asynchronous (events/messages)

```
Service A  ──publishes event──→  Message Broker  ──delivers──→  Service B
```

- RabbitMQ, Kafka, Azure Service Bus, SQS
- Decouples services in time
- More resilient, but more complex

## API Gateway

Single entry point that routes to microservices:

```
Client → [API Gateway] → Users Service
                        → Orders Service
                        → Payment Service
```

Responsibilities:
- Routing
- Centralized authentication
- Rate limiting
- Response aggregation
- SSL termination

Tools: YARP (.NET), Kong, Ocelot, Azure API Management

## When to choose what

| Scenario | Recommendation |
|---------|-------------|
| New project, small team | Monolith or Modular Monolith |
| Simple domain, CRUD | Monolith |
| Complex domain, medium team | Modular Monolith |
| Multiple teams, high scale | Microservices |
| Parts with very different volumes | Microservices for those parts |

> "If you can't build a well-made monolith, you won't be able to build microservices." -- Simon Brown

## Lewis & Fowler's nine characteristics (2014)

The seminal article (Lewis & Fowler, 2014) defines microservices not by service size but by nine traits. A senior should recognize them by name:

1. **Componentization via Services** — components are independently deployable, not just independently buildable libraries.
2. **Organized around Business Capabilities** — teams own a vertical slice (UI + logic + DB), not a horizontal layer. Reflects Conway's Law.
3. **Products not Projects** — "you build it, you run it" (Werner Vogels, Amazon). The team owns the service across its full lifecycle.
4. **Smart Endpoints, Dumb Pipes** — business logic lives in the service; the bus is just a transport (REST, lightweight messaging). Anti-pattern: the heavy ESB.
5. **Decentralized Governance** — let each team pick the right tool, share via libraries instead of standards committees. Netflix's "polyglot programming."
6. **Decentralized Data Management** — each service owns its database. Cross-service consistency is eventual, with sagas and compensating transactions instead of distributed 2PC.
7. **Infrastructure Automation** — continuous delivery, automated provisioning, immutable deploys. Without it, the operational cost crushes the team.
8. **Design for Failure** — services *will* fail. Circuit breakers, timeouts, bulkheads, fallback paths are mandatory, not optional.
9. **Evolutionary Design** — services are split as the model reveals seams. The article is explicit that good microservice boundaries are *emergent*, not designed up front.

Fowler and Lewis are explicit that this list isn't a checklist — a system that scores 7/9 may be microservice-style enough. But missing the operational pieces (#7, #8) is what separates "microservices" from "distributed monolith."

## The microservice premium

Martin Fowler's explicit warning, captured in his bliki post *MicroservicePremium* (May 2015): microservices introduce complexity on their own account, and that *"adds a premium to a project's cost and risk - one that often gets projects into serious trouble."* The premium covers:

- **Operational maturity** — many services to deploy, monitor, secure, and roll back. Without that capability, the architecture costs more than it saves.
- **Distributed systems failure modes** — network partitions, partial failure, eventual consistency, retries / idempotency.
- **Cross-service tracing and debugging** — logs and breakpoints aren't enough; you need distributed tracing.
- **Testing complexity** — contract tests between services, plus integration environments that mirror production topology.

Fowler's companion bliki *MonolithFirst* (June 2015) is even blunter: *"Almost all the successful microservice stories have started with a monolith that got too big and was broken up,"* and *"almost all the cases where I've heard of a system that was built as a microservice system from scratch, it has ended up in serious trouble."* The interview-safe answer: **monolith-first, then extract** — the seams emerge from real usage, not whiteboard speculation.

---

[← Previous: CQRS](08-cqrs.md) | [Next: Messaging →](10-messaging.md) | [Back to index](README.md)
