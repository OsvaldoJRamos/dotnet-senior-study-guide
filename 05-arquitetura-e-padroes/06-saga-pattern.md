# SAGA Pattern

## Problem

Multiple microservices that can present data inconsistencies.

**Example:** the user places an order and there is stock available, but when the inventory microservice is called later, the quantity is no longer available.

## Solution

Each microservice processes a request, saves to its own database, and then passes the transaction along to the others. If any failure occurs, compensating actions (e.g., rollback) are executed on all services that have already been processed.

## Approaches

### 1. Orchestration
A central service (orchestrator) coordinates all saga steps and decides what to do in case of failure.

```
Orquestrador → Serviço A → Serviço B → Serviço C
                  ↑                        |
                  └──── Compensação ←──────┘ (se falhar)
```

**Advantages:** easy to understand, centralized logic
**Disadvantages:** single point of failure, coupling to the orchestrator

### 2. Choreography
Each service knows what to do when it receives an event. There is no central coordinator.

```
Serviço A → Evento → Serviço B → Evento → Serviço C
                                               |
Serviço A ← Evento compensatório ←────────────┘ (se falhar)
```

**Advantages:** decoupled, each service is autonomous
**Disadvantages:** harder to trace the flow, distributed complexity

## When to use

- Distributed transactions across multiple microservices
- When traditional ACID transactions (single database) are not possible
- Operations that need rollback in case of partial failure

---

[← Previous: Tell, Don't Ask](05-tell-dont-ask.md) | [Next: Clean Architecture →](07-clean-architecture.md) | [Back to index](README.md)
