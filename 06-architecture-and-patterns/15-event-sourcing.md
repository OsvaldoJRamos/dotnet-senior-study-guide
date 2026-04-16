# Event Sourcing

Event Sourcing persists state as an **append-only log of events**, not as the current row in a table. The current state is derived by replaying the events. Every change is a fact with a name, a payload, and a timestamp — and the log is the source of truth.

> Martin Fowler: *"Capture all changes to an application state as a sequence of events."* (`martinfowler.com/eaaDev/EventSourcing.html`)

> Fowler (continued): *"We can discard the application state completely and rebuild it by re-running the events from the event log on an empty application."*

It's the opposite mental model from CRUD. Instead of `UPDATE orders SET status = 'Confirmed' WHERE id = 42`, you append `OrderConfirmed { OrderId: 42, At: ... }` to an immutable stream.

## When it pays off

| Good fit | Bad fit |
|---|---|
| Business asks *"what happened and when?"* | Nobody cares about history |
| Audit / compliance requirements (finance, healthcare) | Simple CRUD, low criticality |
| Temporal queries (*"what was the balance on 2026-01-15?"*) | Reporting is the only read path |
| Naturally event-driven domain (banking, IoT, logistics) | Data shape changes constantly in unpredictable ways |
| Need to re-run business rules retroactively (fix bug, replay) | Team is learning DDD/CQRS at the same time |

> Event Sourcing is powerful and expensive. Never adopt it because it sounds cool — adopt it because the domain demands an event log.

## The core mechanics

### 1. Commands trigger events

```
Command:  ConfirmOrder { OrderId, At }
                       │
                       ▼
            Order.Confirm()      ← aggregate validates invariants
                       │
                       ▼
Event:    OrderConfirmed { OrderId, At }
                       │
                       ▼
            append to stream
```

The aggregate checks invariants (*"order must be in Draft"*, *"must have at least one line"*); if valid, it returns one or more **events** that describe what happened. The events are appended to the aggregate's stream.

### 2. State is a fold over events

```csharp
public sealed class Order
{
    public OrderId Id { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderLine> _lines = new();

    // Rehydrate from history
    public static Order FromEvents(IEnumerable<IDomainEvent> events)
    {
        var order = new Order();
        foreach (var e in events) order.Apply(e);
        return order;
    }

    private void Apply(IDomainEvent e)
    {
        switch (e)
        {
            case OrderCreated c:   Id = c.OrderId; Status = OrderStatus.Draft; break;
            case OrderLineAdded l: _lines.Add(new OrderLine(l.Product, l.Quantity, l.UnitPrice)); break;
            case OrderConfirmed _: Status = OrderStatus.Confirmed; break;
            case OrderCancelled _: Status = OrderStatus.Cancelled; break;
        }
    }
}
```

Loading = read the stream + fold. Saving = append the new events atomically at the expected stream version.

### 3. Snapshots (optional)

Replaying 100k events on every load is slow. A **snapshot** is a pre-computed state at version N; loading = snapshot + events after N. Don't add snapshots until profiling says you need them — they add complexity.

## Event Sourcing and CQRS

Event Sourcing and CQRS ([CQRS](08-cqrs.md)) are frequently paired, but they are **separate patterns**:

- **CQRS** splits reads and writes into different models.
- **Event Sourcing** stores writes as an event stream.

They combine well because the event stream naturally feeds **read projections**: subscribe to the event log, project into a denormalized read store (SQL view, Elasticsearch index, Redis cache) shaped for queries.

```
Command → Aggregate → Events → Event Store
                         │
                         ├─► Read projection A (orders list, SQL view)
                         ├─► Read projection B (full-text search, Elasticsearch)
                         └─► Read projection C (real-time dashboard, Redis + SignalR)
```

You can adopt CQRS without Event Sourcing. Adopting Event Sourcing without CQRS is unusual — the read side almost always benefits from projections.

## Storage

Event store requirements:

1. **Append-only per stream.**
2. **Optimistic concurrency on append**: "append these events at expected version N" — fail if another writer advanced the stream.
3. **Read a stream from version 0 (or from a snapshot).**
4. **Global ordering** (for projections that subscribe to every event).

Options:

| Store | Notes |
|---|---|
| **EventStoreDB** | Purpose-built, strong guarantees, subscriptions and projections built in. |
| **Marten** (on PostgreSQL) | .NET library; treats PG as an event store + document DB. |
| **SQL Server / PostgreSQL tables** | A manually-designed `events` table with `(stream_id, version)` unique + append-only discipline. Fine for small-scale. |
| **Kafka** | Great as a transport/projection log, but not ideal as the primary event store — per-key compaction, no per-stream transactional appends. |

## Projections and read models

A **projection** subscribes to the event stream and builds a query-optimized view. Three characteristics:

1. **Idempotent**: processing the same event twice produces the same state (store the last event position).
2. **Replayable**: delete the projection, re-read events from position 0, rebuild it. New projections are just "replay with a new handler".
3. **Eventually consistent**: the read side lags the write side by milliseconds to seconds. If the UI needs "read your own writes", either wait for the projection or read from the write side for that request.

## Challenges

### Event schema evolution

Events are immutable. But over years, the shape changes (rename a field, split a concept). Strategies:

- **Additive only**: new fields optional, never remove.
- **Upcasting**: read old events, transform to the current shape in-memory.
- **Event versioning** (`OrderConfirmedV2`): keep handlers for every version.

> A stored event is a **fact**; you can't revise history. You can only add a new fact that corrects or supersedes it.

### Personally Identifiable Information (PII) and GDPR "right to be forgotten"

Events are append-only, but GDPR requires erasure. Common pattern: **crypto-shredding** — encrypt PII with a per-subject key, store the ciphertext in the event, and delete the key on erasure request. The event remains, but the PII is unreadable.

### Eventual consistency

You can't run a `SELECT` on the write side and expect to see a freshly-projected view. If your UI or downstream depends on read-after-write, either:

- Read from the write side for that request.
- Wait for the projection to catch up (with a subscription position).
- Accept staleness and communicate it in the UX.

### Debugging

Replaying events deterministically reproduces any bug — the event log is a time machine. But debugging across services via events is harder than step-through. Invest in **correlation IDs** and **event sourcing-aware observability** (see [Design Docs and C4](17-design-docs-c4-adr.md) for documenting these flows).

## Anti-patterns

- **CRUD dressed up as events.** `CustomerFieldXChanged` for every setter. Events should capture **business intent** (*OrderConfirmed*, *PaymentCaptured*), not property diffs.
- **Using a message queue as an event store.** Kafka topics, RabbitMQ — those are transport, not storage. You lose per-stream concurrency and replay guarantees.
- **Missing idempotency on projections.** One duplicate event → a double-counted balance. Always track the last processed position per projection.
- **Unbounded streams.** A `User` aggregate with years of events and no snapshotting will make loads slow. Snapshot or design smaller aggregates.
- **Leaking infrastructure into events.** `UserUpdated { __etag: "...", ...internalDto }` ties your event schema to the DB. Events are a **published language** — keep them clean.

## Rule of thumb

> Event Sourcing is a commitment. If you don't need temporal queries, auditability, or retroactive rule replays, stick with CRUD + an audit table. If you do, the event log is the most honest way to represent what the system actually is.

---

[← Previous: API Gateway and BFF](14-api-gateway-and-bff.md) | [Back to index](README.md) | [Next: Application Architectural Patterns →](16-app-architectural-patterns.md)
