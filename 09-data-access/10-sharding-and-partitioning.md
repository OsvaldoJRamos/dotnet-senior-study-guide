# Sharding and Partitioning

**Partitioning** is splitting a large dataset into smaller pieces. **Sharding** is partitioning across multiple **independent servers** so each holds only a subset of the data. Done right, you scale writes horizontally — something a single primary cannot do. Done wrong, you turn a one-database problem into a multi-database nightmare.

## Vertical vs horizontal

| Strategy | What splits | Typical use |
|---|---|---|
| **Vertical partitioning** | Split **columns** across tables/stores (e.g., hot columns in one table, blobs in another) | Row width / row-lock contention |
| **Horizontal partitioning** | Split **rows** into ranges/buckets inside the **same database** | Archive old data, prune fast, parallel I/O |
| **Sharding** (horizontal, across servers) | Split rows across multiple **independent** servers | Scale writes + storage beyond one node |

> Within a single database, the engine provides partitioning primitives (SQL Server partitioned tables, PostgreSQL declarative partitioning, MySQL partitions). These are local — they do not remove the single-writer bottleneck. Sharding does.

## Routing strategies

Every sharded system needs a **shard key** and a **routing function** that maps key → shard.

### 1. Range-based

Rows with adjacent keys live on the same shard: shard A holds IDs 0–1M, shard B 1M–2M, etc.

- ✅ Range queries are cheap (hit one shard).
- ⚠️ **Hot shard risk** on monotonic keys (auto-increment, timestamps) — all new writes land on the last shard.

### 2. Hash-based

Shard = `hash(key) mod N`. Random distribution.

- ✅ Even spread — no hot shard on monotonic keys.
- ⚠️ Range queries become **scatter-gather** across every shard.
- ⚠️ `mod N` is hostile to rebalancing — adding a shard moves most keys. Use **consistent hashing** instead.

### 3. Directory-based (lookup table)

A metadata table stores `key → shard`. Route through the directory.

- ✅ Full flexibility (rebalance without changing keys).
- ⚠️ Extra hop on every query; directory becomes a bottleneck and SPOF if it's not replicated.

### 4. Geo / tenant-based

Shard by region, tenant ID, or other natural business axis.

- ✅ Strong data locality (regulatory, latency).
- ✅ Easy blast-radius isolation per tenant.
- ⚠️ Skew when one tenant is much larger than the rest.

## Choosing a shard key

The shard key is the most important and hardest-to-change decision in the whole design.

MongoDB docs put it plainly: *"The choice of shard key affects the performance, efficiency, and scalability of a sharded cluster. A cluster with the best possible hardware and infrastructure can be bottlenecked by the choice of shard key."* (`mongodb.com/docs/manual/sharding/`)

A good shard key:

1. **High cardinality** — enough distinct values that the data actually splits.
2. **Low frequency per value** — no single value dominates the write load.
3. **Matches the dominant query pattern** — included in `WHERE` for most reads, so queries hit one shard.
4. **Immutable** (or rarely changes) — mutating a shard key moves the row across servers.

Common good picks: `tenant_id`, `user_id`, `order_id` with a composite partition prefix. Common bad picks: `created_at` alone (hot shard), `is_active` (2 values), anything the app routinely `UPDATE`s.

## Hotspots and skew

A **hotspot** is a shard that takes disproportionate load. Root causes:

- Monotonic shard key on range-based routing (all writes hit the tail shard).
- One tenant/customer generating most traffic (celebrity problem).
- Skewed reads concentrated on a recent time window.

**Mitigations:**

- Compose the shard key: `(tenant_id, order_id)` distributes within a tenant.
- Add a **random salt** prefix for write-hot keys (common in DynamoDB).
- Hash the time component for time-series writes.
- Move the heavy tenant to its own dedicated shard (MongoDB calls this a **zone**; Citus supports **schema-based sharding** for tenant isolation).

## Rebalancing

As data grows or traffic shifts, chunks must move between shards. Two broad shapes:

| Approach | How |
|---|---|
| **Automatic balancer** | MongoDB balancer migrates chunks between shards based on data size thresholds |
| **Manual / operator-driven** | Citus `citus_rebalance_start()`, Vitess reshards via workflows (`MoveTables`, `Reshard`) |

> Rebalancing is never free — it consumes bandwidth and adds write amplification. Plan shard counts with growth in mind; reshard during low-traffic windows; throttle to avoid saturating replication.

## Cross-shard queries — where joins go to die

Anything that isn't scoped by the shard key becomes a **scatter-gather**: the router fans out to every shard, each returns partial results, the router merges. Painful at N=4, catastrophic at N=100.

**Common painful patterns:**

- `JOIN` across two tables when their shard keys differ.
- `ORDER BY ... LIMIT N` globally (needs the top-N from every shard to be right).
- `COUNT(*)` or aggregations without a `GROUP BY` on the shard key.
- Global `UNIQUE` constraints (can only be enforced *per shard* naturally).

**Mitigations:**

- **Co-location**: shard related tables by the same key so joins stay local. Citus calls these **distributed tables** joined on the distribution column.
- **Reference tables**: small, rarely changing tables replicated to every shard. Citus docs: *"Replicated to all nodes for joins and foreign keys from distributed tables and maximum read performance"* (`citusdata.com`).
- **Denormalize** the fields you need into the sharded row to avoid the join entirely.
- **Push aggregates down**: `COUNT` per shard, sum in the coordinator.

## Real-world implementations

### Citus (PostgreSQL extension)

*"Citus is a PostgreSQL extension that transforms Postgres into a distributed database"* (`github.com/citusdata/citus`).

- **Coordinator node** holds metadata and routes queries; **worker nodes** store shards as regular Postgres tables.
- Table types: **distributed** (hash-sharded on a distribution column), **reference** (replicated everywhere), **local** (coordinator only).
- Since Citus 12, also supports **schema-based sharding** (one schema per tenant).
- You still speak standard SQL — the extension rewrites distributed queries.

### Vitess (MySQL)

Originally built at YouTube. *"Vitess currently supports MySQL and Percona Server for MySQL"* (`vitess.io`). It ran all YouTube database traffic for over five years.

- **Keyspace** — logical database. *"If you're using sharding, a keyspace maps to multiple MySQL databases; if you're not using sharding, a keyspace maps directly to a MySQL database name."* (`vitess.io`)
- **Shard** — a slice of the keyspace, backed by a MySQL replica set.
- **vtgate** — the query router (proxy) that parses SQL, consults the vschema, and routes to shards.
- **vttablet** — sidecar in front of each MySQL instance.
- **VIndex** — the sharding function that maps column values to keyspace IDs; Vitess provides primary and secondary vindexes for efficient routing on non-primary keys.

### MongoDB native sharding

- **mongos** is the query router; **config servers** (a replica set) hold shard metadata; each shard is itself a replica set.
- Two shard key strategies: **hashed** (even distribution, scatter-gather for ranges) and **ranged** (locality, hotspot risk on monotonic keys).
- The **balancer** migrates chunks between shards automatically.
- **Zones** pin ranges of shard key values to specific shards — used for data-residency (EU data on EU shards) and tiered storage.
- A shard key is immutable once set, except for the special case of updating the value within the same shard; resharding is possible but heavy (`reshardCollection`).

## When NOT to shard

Sharding is **one-way door** engineering. The cost comes before the benefit.

Before reaching for it, exhaust:

1. **Vertical scaling** — a single modern server with 64+ cores and NVMe handles enormous workloads.
2. **Read replicas** — if the bottleneck is reads, see [Replication and Read Replicas](11-replication-and-read-replicas.md).
3. **Local partitioning** — SQL Server partitioned tables, Postgres declarative partitioning, MySQL partitions. Same server, parallel I/O, prunable old data.
4. **Better indexes, better queries** — most "we need to shard" conversations end after an execution plan review.
5. **Archive hot-from-cold** — move historical data to cheaper storage, keep the OLTP table lean.

> Shard when a single write primary can't sustain your write throughput or storage, and the other options are exhausted. Sharding to "be ready for scale" usually costs more than the scale you're preparing for.

## Pitfalls

- **Wrong shard key.** Changing the shard key later means a full reshard. Model it against your top 5 read/write patterns *before* launch.
- **Distributed transactions.** Two-phase commit across shards is slow and availability-hostile. Prefer per-shard atomic writes + idempotent async propagation (Saga / Outbox — see [Polyglot Persistence](09-polyglot-persistence.md)).
- **"Just one more cross-shard query."** Each one becomes a production incident at scale. Push back; denormalize; rethink the access pattern.
- **Ignoring the operational cost.** Every shard is a server to back up, patch, monitor, and fail over. N shards = ~Nx ops load, not 1x.
- **Auto-increment IDs.** Globally unique IDs must come from the app (ULID, UUIDv7) or a dedicated sequence — not `AUTO_INCREMENT` per shard.
- **Secondary-index uniqueness.** `UNIQUE (email)` across shards requires a centralized lookup or a dedicated "email → user_id" table; the DB can't enforce it.

---

[← Previous: Polyglot Persistence](09-polyglot-persistence.md) | [Next: Replication and Read Replicas →](11-replication-and-read-replicas.md) | [Back to index](README.md)
