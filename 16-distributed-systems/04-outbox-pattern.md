# Outbox Pattern

## The problem: dual writes

A service often needs to do two things atomically:

1. Change its own database (e.g., insert an `Order` row).
2. Publish a message to a broker (e.g., `OrderPlaced` to Kafka / RabbitMQ / Service Bus).

These live in **two different systems**, so a single transaction cannot span them. Every crash-ordering is a bug:

| Failure | Outcome |
|---|---|
| Commit DB, then crash before publish | Order exists, nobody is notified. Inventory never reserves stock. |
| Publish, then crash before DB commit | Event about an order that does not exist. Downstream systems corrupt state. |
| Publish, DB commit fails | Event fires for a nonexistent order. Poison message forever. |

Chris Richardson's canonical statement: *"it is not viable to use a traditional distributed transaction (2PC) that spans the database and the message broker."*

## The solution

In the **same DB transaction** that writes the domain change, also insert a row into an **outbox table** in the same database. A separate **relay** process reads the outbox and publishes to the broker, then marks the row as sent.

```text
┌─────────────────────────────┐
│  Single DB transaction      │
│                             │
│  INSERT INTO orders (...);  │
│  INSERT INTO outbox (...);  │──┐
│  COMMIT;                    │  │
└─────────────────────────────┘  │
                                 ▼
                       ┌──────────────────┐
                       │ Relay / Publisher│
                       └────────┬─────────┘
                                │
                                ▼
                       ┌──────────────────┐
                       │     Broker       │
                       └──────────────────┘
```

Guarantees:

- **Never publish a message for a change that rolled back** — message and change are in the same transaction.
- **Never lose a message after the change committed** — relay retries until the broker acks.
- **At-least-once delivery** — if the relay crashes between publishing and marking sent, the message is published again on retry. Consumers must be idempotent.

## Two ways to run the relay

### 1. Polling publisher

A background worker queries the outbox for unsent rows and publishes them.

```sql
SELECT id, aggregate_type, payload, created_at
  FROM outbox
 WHERE published_at IS NULL
 ORDER BY id
 LIMIT 100
   FOR UPDATE SKIP LOCKED;   -- PostgreSQL; SQL Server uses READPAST
```

Pros: simple, no external infra. Cons: poll interval is a latency floor; at high volume the table gets hot.

### 2. Transaction log tailing (CDC)

Instead of polling, tail the database's write-ahead log (WAL / binlog / change feed) and turn log entries directly into broker messages. Commonly done with **Debezium** (Kafka Connect-based CDC) for PostgreSQL, MySQL, SQL Server.

Pros: near-zero latency, no polling overhead, no second write to mark-as-sent. Cons: requires a CDC pipeline and enough operational maturity to run one.

> Debezium ships an "outbox event router" Single Message Transform that reads a dedicated outbox table via CDC and routes each row to the right Kafka topic.

## .NET implementations

### MassTransit — Transactional Outbox

MassTransit provides two modes:

| Mode | Durability | Use |
|---|---|---|
| **In-Memory Outbox** | Messages live in memory until the consumer finishes | Buffer within a single consume; lost on crash |
| **Transactional (Bus + Consumer) Outbox** | Messages persisted via EF Core / other to a DB table, surviving restarts | Production-grade |

The MassTransit docs describe the in-memory version as holding *"published and sent messages in memory until the message is processed successfully (such as the saga being saved to the database)."* For cross-process durability you need the **EF-backed** outbox:

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UsePostgres();           // or UseSqlServer()
        o.UseBusOutbox();          // captures sends inside the scoped DbContext
    });

    x.AddConsumer<OrderPlacedConsumer>();

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbit://...");
        cfg.ConfigureEndpoints(ctx);
    });
});
```

In the handler, the DbContext that saves the domain change is the same DbContext the outbox writes to:

```csharp
public async Task Handle(PlaceOrderCommand cmd)
{
    _db.Orders.Add(new Order { ... });
    await _publishEndpoint.Publish(new OrderPlaced { ... });   // goes to outbox, not broker
    await _db.SaveChangesAsync();                              // one transaction
    // A hosted service delivers the outbox rows to the broker.
}
```

MassTransit's consumer side also supports an **inbox** (dedup by `MessageId`), closing the loop on at-least-once by making the consumer idempotent.

### Wolverine

Wolverine (Jeremy D. Miller, the author of MassTransit-era Marten) integrates outbox into its core messaging pipeline, with first-class support for EF Core and Marten. Outbox is configured per endpoint/message handler; durability is provided by the same DB that stores state. Verify the current API against the Wolverine docs before wiring it — the project iterates fast.

### NServiceBus

NServiceBus ships an Outbox feature that uses the business DB to store outgoing messages and deduplicate inbound messages. Works with SQL Persistence and NHibernate Persistence. As with MassTransit, the guarantee is at-least-once delivery with consumer-side dedup.

## Schema sketch

```sql
CREATE TABLE outbox (
    id              BIGSERIAL PRIMARY KEY,
    message_id      UUID         NOT NULL UNIQUE,      -- used by consumers to dedupe
    aggregate_type  TEXT         NOT NULL,
    aggregate_id    TEXT         NOT NULL,
    event_type      TEXT         NOT NULL,
    payload         JSONB        NOT NULL,
    headers         JSONB,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    published_at    TIMESTAMPTZ
);

CREATE INDEX outbox_unpublished_idx
    ON outbox (id) WHERE published_at IS NULL;
```

Notes:

- **`message_id` UNIQUE**: the consumer inbox dedupes on this. Without it, duplicates are undetectable.
- **Partial index on unpublished rows**: keeps the poller query fast even as the outbox grows.
- **Archive / truncate published rows** periodically — don't let the table grow unbounded.

## Pitfalls

- **Outbox in a different database from the domain data**: loses the atomicity. The whole point is a single transaction.
- **Skipping the consumer-side dedup (inbox)**: at-least-once means duplicates. Without dedup, you will process the same event twice on a relay restart.
- **Using the outbox as an event store**: an outbox is *transient* — it holds outgoing messages until delivered. Event sourcing is a different pattern (see [Event Sourcing](../06-architecture-and-patterns/15-event-sourcing.md)).
- **No backpressure on the relay**: if the broker is down, outbox rows accumulate. Monitor unpublished count and lag; alert when it crosses a threshold.
- **Forgetting ordering requirements**: an outbox does not guarantee cross-aggregate ordering. Within one aggregate (same partition key) order is preserved; across aggregates, the broker's partitioning strategy decides.
- **Publishing from inside the transaction**: if you publish directly to the broker before committing, you have re-created the dual-write problem. The publish must go to the outbox only.

## When NOT to use it

- Single-database, single-service systems — no network between the write and the "publish," so there's nothing to decouple.
- Cases where "fire and forget, occasional loss is fine" is the explicit business requirement (metrics shipping, non-critical audit).
- When you already have a CDC pipeline and the downstream consumers read the domain tables directly — the outbox is redundant.

## References

- Richardson — *Pattern: Transactional Outbox*: https://microservices.io/patterns/data/transactional-outbox.html
- MassTransit — *Transactional Outbox*: https://masstransit.io/documentation/patterns/transactional-outbox
- Debezium — *Outbox Event Router*: https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html

---

[← Previous: Idempotency](03-idempotency.md) | [Next: Saga Pattern →](05-saga-pattern.md) | [Back to index](README.md)
