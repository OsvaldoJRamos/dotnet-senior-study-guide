# Polyglot Persistence

**Polyglot persistence** is the practice of using **multiple database technologies inside the same system**, each chosen for the kind of data it handles best — a relational database for the transactional core, Redis for cache and sessions, Elasticsearch for search, a document store for flexible payloads, a time-series DB for metrics.

> Martin Fowler: *"any decent sized enterprise will have a variety of different data storage technologies for different kinds of data."* And: *"increasingly we'll be first asking how we want to manipulate the data and only then figuring out what technology is the best bet for it."* (`martinfowler.com/bliki/PolyglotPersistence.html`)

It's powerful — and it's how most real systems drift into operational pain. This file covers when it pays off, how to integrate the stores safely, and the anti-patterns that show up.

## Why combine stores at all

Relational databases are good at a lot of things and great at *some* things. They are **not** the best answer for:

| Workload | Better fit |
|---|---|
| Hot read path (< 1 ms p95) | In-memory store (Redis, Memcached) |
| Full-text / fuzzy search, faceting, relevance ranking | Elasticsearch, OpenSearch, Postgres `tsvector` at smaller scale |
| Semi-structured data with frequent schema change | Document store (MongoDB, Cosmos DB, Postgres JSONB) |
| Massive write throughput with eventual consistency | Column-family / wide-column (Cassandra, ScyllaDB) |
| Time-series metrics with retention/rollups | InfluxDB, TimescaleDB |
| Graph traversals over deep relationships | Neo4j, Neptune, Postgres + recursive CTEs at smaller scale |

> None of these replaces the relational DB for the transactional core. They **supplement** it. A checkout that crosses three stores on the hot path is a design smell; a checkout that writes to SQL Server and projects to Elasticsearch for the catalog page is the usual shape.

## A realistic architecture

A common pattern in a mid-sized e-commerce:

```
Web / API
   │
   ├── Writes & reads on the transactional aggregate ──► SQL Server / PostgreSQL
   │
   ├── Hot reads (product page, cart) ─────────────────► Redis (cache-aside)
   │
   ├── Search (catalog, faceted filters, typo-tolerant) ► Elasticsearch
   │
   ├── Session / auth tokens ──────────────────────────► Redis (TTL)
   │
   └── Audit log / event history ──────────────────────► Blob storage / S3 / event store
```

- **SQL** is the **source of truth** for money-moving entities.
- **Redis** is derived — can be rebuilt by re-reading from SQL.
- **Elasticsearch** is derived — indexed from the write-side events.
- **Blob / event log** is append-only compliance storage.

## Keeping derived stores in sync

This is the hard part. You have a write in one store and you need it to land (eventually) in another. The three patterns you'll see:

### 1. Transactional Outbox

Write the business data and an **outbox row** in the same local transaction; a background relay publishes the outbox row as a message and marks it processed. This is the most common pattern in .NET stacks.

```
BEGIN TX
  INSERT INTO orders (...)
  INSERT INTO outbox (aggregate_id, event_type, payload, status='pending')
COMMIT
```

A worker reads `status='pending'` rows, publishes them (RabbitMQ / Service Bus / Kafka), and marks them `'published'`. Consumers read from the bus and update Elasticsearch / Redis / the next service's DB.

- ✅ Atomic with the write. No lost events if the broker is down at write time.
- ✅ Idempotent consumers handle at-least-once delivery safely (see [Idempotency and Race Conditions](../06-architecture-and-patterns/11-idempotency-and-race-conditions.md)).
- ⚠️ Extra table to maintain; extra worker to run.

### 2. Change Data Capture (CDC)

The database itself emits a stream of row-level changes. The system reads that stream (no application code change needed) and feeds it to downstream stores.

| Source | CDC tool |
|---|---|
| SQL Server | **CDC feature** + Debezium / Azure Data Factory / Striim |
| PostgreSQL | **Logical replication** (`pgoutput`) + Debezium |
| MySQL | **Binlog** + Debezium / Maxwell |
| MongoDB | **Change streams** |

- ✅ No application code to maintain — the DB is the source.
- ✅ Catches changes from *any* writer (legacy jobs, manual UPDATE, other services).
- ⚠️ Events are row-shaped, not business-shaped. You usually still need a translator to turn `row_changed` into `OrderConfirmed`.
- ⚠️ Schema evolution in the source DB becomes a cross-system breaking change.

> CDC is great for **migrations** (replicating a legacy system into a new read store) and for **analytics feeds**. For business events that drive other services, Outbox is almost always cleaner.

### 3. Dual-write (anti-pattern)

Application code writes to the DB *and* publishes a message, in two separate steps:

```csharp
db.SaveChanges();          // 1
await bus.PublishAsync(e); // 2 — what if the process dies here?
```

- ❌ If step 1 succeeds and step 2 fails, the two stores diverge. There is no local transaction that spans them.
- ❌ Distributed transactions (MSDTC / 2PC) are technically possible but operationally costly and often unsupported by modern brokers.

**Don't do this.** Use Outbox.

## Cache consistency

Caching is the easiest polyglot store to add and the easiest to get wrong. Two patterns dominate:

### Cache-aside (lazy loading)

```
GET ──► Redis MISS ──► SQL ──► write to Redis ──► return
WRITE ──► SQL ──► INVALIDATE Redis key
```

- Simple; the cache is just an optimization.
- **Pitfall**: invalidation race — reader gets stale value between SQL commit and cache delete. For tight consistency, invalidate before commit and refill on the next read.

### Write-through

Writes go through the cache; the cache writes to the DB synchronously.

- Cache and DB always in sync.
- Writes pay the cost of both stores.

### What to cache

- Read-heavy, rarely changing: product catalog, reference data, permissions.
- Expensive to compute: aggregations, reports, formatted DTOs.
- Per-session state: auth tokens, cart — with TTL as a safety net.

### What NOT to cache

- Data you need strongly consistent (balances).
- Data that changes on almost every read (counters — use atomic operations on the DB).
- Large blobs — use a CDN, not Redis.

## The Outbox + Projection shape (end-to-end)

```
┌──────────┐  INSERT order + outbox  ┌──────────┐
│   API    │──────────────────────► │   SQL    │
└──────────┘                        └─────┬────┘
                                          │ outbox relay
                                          ▼
                                    ┌──────────┐
                                    │   Bus    │ (RabbitMQ / Service Bus / Kafka)
                                    └─────┬────┘
                      ┌───────────────────┼───────────────────┐
                      ▼                   ▼                   ▼
               ┌──────────┐        ┌─────────────┐      ┌──────────┐
               │  Redis   │        │Elasticsearch│      │ Another  │
               │ (cache)  │        │ (search idx)│      │ service  │
               └──────────┘        └─────────────┘      └──────────┘
```

Each consumer is **idempotent** (dedup by event ID / business key) and maintains its own progress cursor. If one consumer is down, the others keep going. Rebuilding a store = replay from event ID 0.

## When polyglot is a mistake

Every new store costs:

- A new language/driver to maintain.
- A new set of ops concerns (backup, scaling, failover, observability).
- A new consistency boundary.
- More cognitive load on every engineer joining.

Don't reach for polyglot persistence because a store sounds fashionable. Ask:

1. **What access pattern does the relational DB not support efficiently?** If none, stop.
2. **Can Postgres or SQL Server solve it?** JSONB, GIN, full-text search, hstore, partitioning cover a surprising amount. A fully-supported extension beats a new product.
3. **Who will run the new store in production?** "We'll figure it out" means an outage at 2 AM.
4. **What's the rollback plan if the new store underdelivers?** If there's no plan, you've committed to it forever.

## Pitfalls

- **Search index as source of truth.** Elasticsearch is a read replica, never the primary. Back it with SQL + a rebuild pipeline.
- **Redis as a database.** Redis is fast because it's not durable by default. Use it for caches, sessions, rate-limiting, queues — not for records you can't lose.
- **Dual writes.** Already covered above. Use outbox.
- **No idempotency on consumers.** At-least-once delivery + non-idempotent consumer = duplicates in your read store. Always dedup (see [Idempotency and Race Conditions](../06-architecture-and-patterns/11-idempotency-and-race-conditions.md)).
- **"One DB per microservice" taken to extreme.** A microservice architecture with 12 different database products is an operational nightmare. Standardize on one or two per organization.
- **Rebuild path untested.** If you cannot re-create Redis and Elasticsearch from SQL on demand, you don't own them — they own you.

## Rule of thumb

> Start with **one** relational database, do not add a second store until the access pattern *proves* it can't — and even then, pay the integration cost up front (Outbox + idempotent projections), not as a retrofit after the data has drifted.

---

[← Previous: Execution Plans](08-execution-plans.md) | [Next: Sharding and Partitioning →](10-sharding-and-partitioning.md) | [Back to index](README.md)
