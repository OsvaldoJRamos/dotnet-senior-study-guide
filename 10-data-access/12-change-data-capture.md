# Change Data Capture

**Change Data Capture (CDC)** is a pattern for extracting a stream of row-level changes from a database and making them available to downstream consumers — caches, search indexes, data lakes, other services.

[Polyglot Persistence](09-polyglot-persistence.md) introduced CDC alongside the Transactional Outbox. This file goes deeper: how it actually works, the three implementation styles, the main engines' native features, and Debezium — the de-facto open-source CDC platform in the Java/Kafka world.

## Log-based vs trigger-based vs query-based

Three ways to detect changes. Pick the one whose cost your ops team can absorb.

| Approach | How it works | Impact on source DB | Completeness |
|---|---|---|---|
| **Log-based** | Tail the transaction log / WAL / binlog | Minimal — no extra writes | High — captures every commit including deletes |
| **Trigger-based** | `AFTER INSERT/UPDATE/DELETE` triggers write to a shadow table | Noticeable — every write costs an extra row | High |
| **Query-based (polling)** | Periodic `SELECT WHERE updated_at > ?` | Low if indexed — but extra read load | Misses deletes; misses intra-poll changes |

**Log-based is the modern default.** The database already writes every change to its log for durability — CDC just reads that same log. Zero extra load on the write path, no triggers to maintain, and the log captures operations the app-level code might miss (legacy jobs, manual UPDATEs, replication apply).

> Query-based CDC is the retrofit answer for systems where you cannot enable log reading. It can work, but you will eventually miss a delete and debug a phantom for two days.

## The Debezium / Kafka Connect model

Debezium is an open-source CDC platform built on **Kafka Connect**. Debezium describes itself as *"an open source CDC (change data capture) platform"* that *"provides a low latency data streaming platform for change data capture"* (`github.com/debezium/debezium`).

```
┌───────────┐   log/WAL   ┌─────────────────┐   Kafka topic  ┌──────────┐
│  Source   │───read─────►│ Debezium        │───produce─────►│ Kafka    │
│  DB       │             │ connector       │                │ cluster  │
│ (MySQL,   │             │ (in Kafka       │                └─────┬────┘
│  Postgres,│             │  Connect)       │                      │
│  SQL Srv, │             └─────────────────┘                      │
│  Mongo,   │                                                       ▼
│  Oracle)  │                                               ┌──────────┐
└───────────┘                                               │ Consumer │
                                                            │ (app /   │
                                                            │  sink    │
                                                            │  connector│
                                                            │  → ES,   │
                                                            │  Postgres│
                                                            │  ...)    │
                                                            └──────────┘
```

Key properties from the Debezium README:

- Each connector *"deployed to the distributed Kafka Connect service monitors a database and records changes in Kafka topics"*.
- Supported source connectors: **MySQL/MariaDB, PostgreSQL, Oracle, SQL Server, MongoDB** (plus others like Db2, Cassandra, Vitess, Spanner depending on release). Debezium also ships a **JDBC sink connector** (for writing CDC events from Kafka into a relational target) — it is a sink, not a source.
- For PostgreSQL, Debezium uses logical decoding plugins: `pgoutput` (built into Postgres), and external options like `decoderbufs` and `wal2json`.
- Events are durable because they live in Kafka topics — consumers can be down and catch up later.

### Snapshot then stream

Debezium (and most CDC tools) run in two phases:

1. **Initial snapshot** — read the current table contents as an `r` (read) event stream, so consumers can bootstrap from zero.
2. **Streaming** — from the snapshot's LSN/offset onward, emit every new commit as a `c`/`u`/`d` event.

Interrupted snapshots, incremental snapshots (to backfill without full re-read), and signal tables are advanced features — check the current Debezium docs for your connector before relying on them.

### Event shape

A Debezium event payload typically carries `op` (c/u/d/r), `ts_ms`, `source` (LSN, file, pos, snapshot flag), `before` and `after` row images:

```json
{
  "op": "u",
  "ts_ms": 1713553300123,
  "source": { "db": "shop", "table": "orders", "lsn": 246789 },
  "before": { "id": 42, "status": "Pending" },
  "after":  { "id": 42, "status": "Shipped" }
}
```

## Engine-specific CDC

### SQL Server CDC

Microsoft Learn: *"The source of change data for change data capture is the SQL Server transaction log. As inserts, updates, and deletes are applied to tracked source tables, entries that describe those changes are added to the log. The log serves as input to the capture process."* (`learn.microsoft.com`)

- Enable at the database with `sys.sp_cdc_enable_db`, then per table with `sys.sp_cdc_enable_table`.
- A **capture job** (SQL Server Agent) reads the log and writes rows to **change tables** in the `cdc` schema, named `cdc.<schema>_<table>_CT`.
- Each change-table row includes `__$operation` (1=delete, 2=insert, 3=update-before, 4=update-after), `__$start_lsn`, `__$update_mask`.
- Query via `fn_cdc_get_all_changes_<capture_instance>` and `fn_cdc_get_net_changes_<capture_instance>`.
- The **cleanup job** runs daily at 2 AM by default, retaining change data for 4320 minutes (3 days) and removing up to 5000 rows per delete.
- **Caveat:** CDC requires SQL Server Agent to be running. When a database has CDC enabled, log truncation is held until the capture process has processed the relevant log entries — a stuck capture job can blow up your transaction log.

### SQL Server Change Tracking (not CDC)

Change Tracking is **lighter but less complete**. Microsoft Learn: *"change tracking [is] a lightweight solution that provides an efficient change tracking mechanism for applications."* It tells you *which rows changed* and the operation type — but not the intermediate values.

> **CDC vs Change Tracking:** CDC captures **every change** including before/after column values — good for auditing and replicating to other systems. Change Tracking just records that a row changed — good for sync apps that will re-fetch the latest row. Pick CDC for downstream projections; pick Change Tracking for "give me rows modified since version X".

### PostgreSQL logical decoding

The Postgres feature Debezium's Postgres connector sits on top of.

Postgres docs: *"PostgreSQL provides infrastructure to stream the modifications performed via SQL to external consumers. This functionality can be used for a variety of purposes, including replication solutions and auditing. Changes are sent out in streams identified by logical replication slots."* (`postgresql.org/docs/current/logicaldecoding.html`)

- `wal_level = logical` must be set (restart required).
- A **replication slot** is a named stream position — Postgres holds WAL until the slot consumer catches up. An abandoned slot causes unbounded WAL growth; always monitor `pg_replication_slots`.
- An **output plugin** formats the changes. `pgoutput` is built-in (used by logical replication and Debezium); `wal2json` and `decoderbufs` are external.
- Availability of the `before` image for `UPDATE`/`DELETE` depends on **`REPLICA IDENTITY`** on the table (`DEFAULT`, `USING INDEX`, `FULL`, or `NOTHING`). `FULL` captures the whole old row at the cost of bigger WAL.

### MySQL binlog

- CDC reads the **binary log** (binlog). Requires `binlog_format = ROW` (or `MIXED` that falls back to row for DDL).
- Binlog coordinates (`file`, `pos`) or GTIDs identify the stream position; Debezium tracks them in its offsets topic.
- Binlog retention (`expire_logs_seconds` / `binlog_expire_logs_seconds`) must be long enough that a down consumer can catch up; otherwise you have to re-snapshot.

### MongoDB change streams

- MongoDB exposes `collection.watch()` / `db.watch()` as a first-class API (not a log-reader bolt-on).
- Backed by the oplog; requires a replica set (or sharded cluster — the mongos aggregates).
- Resume tokens let consumers pick up where they left off across restarts and failovers.

## Use cases

### 1. Cache invalidation

Read-through cache (Redis, in-memory) backed by a relational source. CDC publishes every `UPDATE` to `product`, a consumer evicts `product:{id}` from Redis.

```
SQL UPDATE product SET price=... WHERE id=42
        │
        ▼
   Debezium
        │
        ▼
   Kafka topic "dbserver.shop.product"
        │
        ▼
   Consumer → redis.DEL("product:42")
```

Cleaner than dual-writes — the app doesn't know or care about the cache.

### 2. Data sync / projections to another store

Write to SQL, project to Elasticsearch for search, to a data warehouse for analytics, to another service's database for a bounded context that needs a read-model. CDC is the backbone of **the read side of CQRS** at scale.

### 3. Audit / history trail

Append every change to a compliance store (S3, ADLS, Snowflake). CDC captures even rows changed by legacy jobs that bypass the app layer.

### 4. Zero-downtime migration

Start CDC from the legacy DB → new DB. Snapshot once, then stream. Cut over when lag is near zero.

### 5. Event-driven integration (use carefully)

You *can* publish CDC events as business events to other services — but the events are **row-shaped**, not business-shaped. A `customer_address.UPDATE` event doesn't tell another service *why* it changed (address correction vs move vs fraud unwinding).

> For business events that cross service boundaries, prefer the **Transactional Outbox** pattern (see [Polyglot Persistence](09-polyglot-persistence.md)) — the app decides the event shape. Keep CDC for technical projections and migrations.

## Ordering, idempotency, exactly-once

- **Ordering:** per-key ordering is preserved by partitioning the Kafka topic by the primary key. Global ordering across a table is not guaranteed in a partitioned topic.
- **Delivery:** at-least-once. Consumers must dedup. Use the source LSN / binlog position / change token as the idempotency key.
- **"Exactly-once":** Kafka offers transactional writes and `exactly_once_source` / `exactly_once_support` in Connect, but end-to-end exactly-once across the source DB + Kafka + the sink is still an engineering commitment (idempotent sinks, fencing, dedup). Treat any "exactly-once" claim as "at-least-once + idempotent consumer".

## Schema evolution

The source table schema *will* change. CDC consumers will break.

- **Add column** — old consumers ignore the new field; generally safe.
- **Drop column** — consumers with explicit mappings break.
- **Rename column** — appears as drop + add; always breaking.
- **Type change** — almost always breaking; the serialized payload changes.

Use a **schema registry** (Confluent Schema Registry, Apicurio) with Avro/Protobuf/JSON Schema to enforce backward/forward compatibility rules. Without one, a DBA's `ALTER TABLE` at 3 AM is a silent outage of every downstream consumer.

## Pitfalls

- **Unmanaged replication slots.** Forgot a Debezium instance? Postgres WAL grows until the disk fills. Alert on `pg_replication_slots.active = false`.
- **`REPLICA IDENTITY DEFAULT` on a table without a primary key.** Postgres logical decoding produces events with no `before` image, so updates/deletes are effectively useless downstream.
- **Capture job stopped in SQL Server CDC.** Log truncation halts; the transaction log grows. Monitor `sys.dm_cdc_log_scan_sessions`.
- **Schema drift.** Running `ALTER TABLE` with no coordination with downstream consumers is how CDC pipelines go down. Deploy schema changes as part of a reviewed process.
- **Using CDC as a business event bus.** Works until someone refactors the schema and now `UserRegistered` events stop flowing. Use Outbox for business events.
- **Missing deletes.** Query-based CDC never sees them. Log-based CDC sees them only if the tool processes `DELETE` events. DynamoDB Streams / Postgres logical / SQL Server CDC all capture deletes; double-check for any tool you pick.
- **Heavy initial snapshots on a busy primary.** Snapshot reads can contend with production. Schedule in a window, or run against a replica, or use **incremental snapshots**.

## Rule of thumb

> Use **Outbox** for events that represent business decisions another service will act on. Use **CDC** for technical projections (caches, search, warehouses) and migrations. The patterns complement each other; the mistake is using one for the other's job.

---

[← Previous: Replication and Read Replicas](11-replication-and-read-replicas.md) | [Next: Data Modeling →](13-data-modeling.md) | [Back to index](README.md)
