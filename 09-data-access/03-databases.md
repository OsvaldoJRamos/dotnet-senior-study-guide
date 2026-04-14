# Databases

## General

In an application with heavy write pressure, say an e-commerce on Black Friday, closing several purchases per second, you should program your application to use **queues**. Every new purchase enters the queue and gets written to the database as capacity allows. You should never just increase the number of web application servers and let everyone connect to the same database at the same time — that will only increase contention and in the end nobody can write.

It's better for everyone to enter a queue and let only the queue manage the database. Reads use Redis caches, which takes the search load off the database.

The major bottleneck in a web application is the database, because even with replication and sharding, you will always have contention due to ACID guarantees.

## NoSQL Databases

Unlike a relational database, most are neither a transactional database nor an analytical database where you can run queries like SQL with any criteria at any time.

Searches in NoSQL databases need to be **planned and indexed in advance**. NoSQL databases tend to be databases that run in clusters, with shards, meaning no single server has all the data you need at the same time, and if your search is poorly planned, it will have to scan across all nodes in the network, which costs processing and time — a lot of time.

## Relational Databases

In a relational database, what costs the most in terms of performance is writing, more than reading. Writes need to have ACID guaranteed: **atomicity, consistency, isolation, and durability**. Written data must be guaranteed to be on disk so that an unexpected crash doesn't cause data loss.

Multiple connections writing at the same time cannot step on each other's toes — you need a concurrency scheme like **multiversion concurrency control or MVCC**, which most databases use today. And you need **transaction logs** to be able to rollback or return to the original state of the data if a multi-operation transaction fails midway.

## 1. MongoDB

It is a NoSQL document database. It is **not** an in-memory database — it uses the **WiredTiger** storage engine with memory-mapped files and a working-set cache, so hot data is served from RAM while the full dataset lives on disk.

### Write concerns

What `OK` means on a write is configurable via **write concerns**:

| Write concern | Meaning |
|---|---|
| `w: 0` | Fire-and-forget — no acknowledgement, fastest, weakest durability |
| `w: 1` (default) | **Primary** acknowledged the write in memory |
| `w: "majority"` | A majority of the replica set acknowledged — safe against primary failure |
| `j: true` | Written to the on-disk **journal** before ack (durable across crash) |

Combine them: `{ w: "majority", j: true }` gives strong durability (slower); `{ w: 1 }` is the typical default (fast, primary-only). Stronger concerns trade latency for safety — pick per operation based on criticality.

Since MongoDB 5.0 the replica-set roles are **primary** and **secondary** (the old "master/slave" terminology is gone). You can require reads/writes to go through the primary, or route reads to secondaries with `readPreference`.

## 2. Cassandra

A NoSQL **wide-column store**. Despite surface-level similarities with relational databases — schemas and CQL which looks like SQL — the data model is different. It organizes data into **tables** with a **partition key** (which node owns the row) and **clustering columns** (sort order within the partition). It was built to be highly distributed, with multiple nodes in a cluster, preferably across multiple regions.

Cassandra **was not made to be used on a single instance**, like a blog database. Its sweet spot is running in a cluster, with multiple nodes, and with heavy writes.

### Tunable consistency

Cassandra is eventually consistent by default, but lets you pick a **consistency level per operation**:

| Level | Meaning |
|---|---|
| `ONE` | One replica responds — fastest, weakest |
| `QUORUM` | Majority of replicas across the cluster respond |
| `LOCAL_QUORUM` | Majority within the **local datacenter** — common for multi-region |
| `ALL` | Every replica responds — strongest, slowest, fragile |

Pairing writes and reads at `QUORUM` gives read-your-writes consistency at the cost of latency. There are **no joins and no multi-row transactions** (only lightweight transactions on a single partition via Paxos) — the model is optimized for write-heavy, append-style workloads.

## 3. Redis

In practice most of us will always end up using a **combination of a relational database with a cache**, using something like Redis. In this configuration Redis can be a bit more relaxed because the right approach is to have data queried from Redis and whatever isn't there you load from the relational database and write to Redis as needed.

Therefore Redis can be rebuilt from scratch even if there's a crash and you need to tear everything down. And you use Redis to keep things that are expensive to calculate and write in a relational database like **aggregations**, things with counters, averages, and other metrics that aggregate multiple rows from the database.

### Persistence modes

Redis is in-memory but can persist to disk:

- **RDB (snapshots):** periodic point-in-time dumps of the dataset. Compact, fast restore, but you can lose the seconds between snapshots on crash.
- **AOF (append-only file):** every write is appended to a log. Safer (configurable fsync: `always`, `everysec`, `no`) but larger files and slower restore.
- **Both together:** most durable; Redis prefers the AOF on restart.

### Eviction policies

When `maxmemory` is hit, Redis evicts based on `maxmemory-policy`:

| Policy | Behavior |
|---|---|
| `noeviction` | Return errors on writes (default) |
| `allkeys-lru` | Evict least-recently-used across all keys — typical cache setting |
| `allkeys-lfu` | Evict least-frequently-used |
| `volatile-lru` / `volatile-lfu` | Same, but only on keys with a TTL |
| `volatile-ttl` | Evict keys with the shortest remaining TTL first |
| `allkeys-random` / `volatile-random` | Random eviction |

For a pure cache use `allkeys-lru` (or `allkeys-lfu` for skewed access patterns); for a key-value store where losing untagged keys is unacceptable, stick with `volatile-*` and always set TTLs.

## Performance Tips

1. **Controlled denormalization** — The more normalized the table, the slower it will be as it demands more from the database. E.g.: instead of creating a STATES table (Sao Paulo, Minas Gerais), the state abbreviation and name can be saved directly in the user table. This will reduce joins and increase performance.

2. **Cache** — Add caching to avoid hitting the database as much as possible.

3. **Queues for writes** — Program the application to put database write requests into a queue (like RabbitMQ), to avoid bottlenecks and timeouts for the user.

4. **Full-text search** — When you need to do text search and show results by relevance (like e-commerce search), the most correct and performant approach is to use something like **ElasticSearch**.

5. **Indexes** — Create the necessary indexes, but don't create too many as it can hurt performance (each index needs to be updated on every write).

---

[← Previous: Entity Framework](02-entity-framework.md) | [Next: Query Optimization →](04-query-optimization.md) | [Back to index](README.md)
