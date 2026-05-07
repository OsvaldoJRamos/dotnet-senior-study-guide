# SAGA Pattern

## Problem

Multiple microservices that can present data inconsistencies.

**Example:** the user places an order and there is stock available, but when the inventory microservice is called later, the quantity is no longer available.

## Solution

A saga is a sequence of local transactions: each step is a local transaction in its own service; subsequent steps are triggered via messages (choreography) or commands from an orchestrator. Sagas do **not** propagate a distributed transaction — there is no global commit. If any step fails, previously completed steps are undone by running **compensating transactions**, which are business-level reversals (e.g., "issue refund", "release reserved stock"), not raw database rollbacks. Compensations must be **idempotent**, since messages may be retried or redelivered.

## Approaches

### 1. Orchestration
A central service (orchestrator) coordinates all saga steps and decides what to do in case of failure.

```
Orchestrator → Service A → Service B → Service C
                  ↑                        |
                  └──── Compensation ←─────┘ (if it fails)
```

**Advantages:** easy to understand, centralized logic
**Disadvantages:** single point of failure, coupling to the orchestrator

### 2. Choreography
Each service knows what to do when it receives an event. There is no central coordinator.

```
Service A → Event → Service B → Event → Service C
                                               |
Service A ← Compensating event ←───────────────┘ (if it fails)
```

**Advantages:** decoupled, each service is autonomous
**Disadvantages:** harder to trace the flow, distributed complexity

## When to use

- Distributed transactions across multiple microservices
- When traditional ACID transactions (single database) are not possible
- Operations that need rollback in case of partial failure

## Outbox Pattern (companion to sagas)

Sagas depend on reliable messaging between steps, which exposes the **dual-write problem**: a service must both persist its local change and publish a message, but writing to the DB and writing to the broker cannot share a single transaction. A crash between the two leaves the system inconsistent.

The **Outbox pattern** solves this:

1. In the same DB transaction that persists the domain change, also insert the outgoing message into an `Outbox` table in the same database.
2. A separate publisher process (a background worker or CDC/log-tail tool like Debezium) reads unpublished rows from the `Outbox` and dispatches them to the broker.
3. Once the broker acknowledges, the publisher marks the row as sent (or deletes it).

```
[Service] ── single DB transaction ──▶ [Domain table] + [Outbox table]
                                                           │
                                                           ▼
                                               [Publisher] ──▶ [Broker]
```

This guarantees **at-least-once** delivery of the message whenever the domain change is committed, and never delivers a message for a change that was rolled back. Consumers must therefore be idempotent (dedup by message ID).

A symmetric **Inbox** pattern on the consumer side records processed message IDs to deduplicate redeliveries.

---

[← Previous: Tell, Don't Ask](05-tell-dont-ask.md) | [Next: Clean Architecture →](07-clean-architecture.md) | [Back to index](README.md)
