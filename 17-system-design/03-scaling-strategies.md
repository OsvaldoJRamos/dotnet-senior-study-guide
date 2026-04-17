# Scaling Strategies

Scaling is the art of removing the next bottleneck — not of building infinitely scalable systems upfront. Senior signal: pick the cheapest strategy that buys you another order of magnitude, then stop.

## Vertical vs horizontal

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| What you do | Bigger box (more CPU/RAM) | More boxes |
| Max ceiling | Hardware limit | ~Unbounded (given the software supports it) |
| Operational complexity | Low — one node | High — LB, service discovery, shared state |
| Failure mode | Single point of failure | Graceful degradation (with enough redundancy) |
| Cost curve | Sharply non-linear above mid-tier | Roughly linear |
| Best fit | Stateful primaries (RDBMS), caches you can't shard | Stateless web/app tier, microservices |

> First rule: **scale up before you scale out**. A c6i.8xlarge gets you a surprising distance before you need a distributed anything. Complexity is a cost that compounds.

## Stateless services

A service is **stateless** when any node can serve any request. State lives in the database, cache, or session store — not in process memory.

Stateless gives you four things:

1. **Horizontal scale** — just add instances behind the LB.
2. **Easy failover** — any instance going down is a non-event.
3. **Rolling deploys** — drain, kill, replace.
4. **Auto-scaling** — spin up on CPU/QPS thresholds.

### In-process state to externalize

| State | Move to |
|---|---|
| User session | Redis / SQL / signed cookie (JWT) |
| In-memory cache of reference data | Distributed cache (Redis) or HybridCache for local+distributed |
| Uploaded file buffers | Object storage (S3 / Blob) |
| WebSocket connection | Sticky LB + back-plane (SignalR Redis back-plane) — see [Chat System](09-case-study-chat-system.md) |

> Sticky sessions are a smell, not a solution. They pin failure to specific nodes. Prefer a shared session store.

## Database read scaling — read replicas

Reads almost always dominate (often 10-100:1). Add **read replicas** before you shard.

```text
 Writes ──► Primary ──async replication──► Replica 1
                            └─────────────► Replica 2
 Reads  ◄───────────── Replicas (round-robin / nearest)
```

Trade-offs:

- **Replication lag**: replicas are *eventually* consistent. A read-your-writes flow (user posts, then sees the post) may read stale if it hits a lagging replica.
- **Read-your-writes fix**: route recent writes' reads to the primary for a short window, or use session-bound replica routing.
- **Failover**: replicas can be promoted to primary on failure. Managed services (RDS, Aurora, Azure SQL) do this for you.

## Database write scaling — sharding

When writes exceed a single primary, **shard** — partition the data across multiple independent primaries.

### Common shard keys

| Strategy | How | Pros | Cons |
|---|---|---|---|
| Hash of key | `shard_id = hash(user_id) % N` | Even distribution | Range queries impossible; resharding is painful |
| Range | `user_id 1-1M → shard 1`, etc. | Range queries cheap | Hot shard if workload skewed |
| Geographic | `region = user.country` | Data locality, compliance | Hot regions overload |
| Consistent hashing | Keys map to a ring, nodes own arcs | Resharding moves ~1/N of data (Karger et al., STOC '97) | Slightly more complex; needs virtual nodes for balance |

See [Load Balancing → consistent hashing](04-load-balancing.md) for the ring-based mechanics.

### What breaks when you shard

- **Cross-shard joins** — don't do them. Denormalize or do application-side joins.
- **Cross-shard transactions** — rare, expensive, use saga / outbox instead. See [Saga Pattern](../06-architecture-and-patterns/06-saga-pattern.md).
- **Unique constraints** — only enforceable within a shard, not across.
- **Schema migrations** — now run N times, ideally online.

> Sharding is a one-way door. Exhaust read replicas, caching, and denormalization first.

## CDN for static content

Static assets (images, JS, CSS, video segments) **should never touch your origin** on a cache hit. A CDN:

- Terminates TLS close to the user (< 10-30 ms RTT globally).
- Caches at edge PoPs, bypassing origin for hot content.
- Absorbs traffic spikes and DDoS.
- Reduces origin egress cost dramatically.

When to use:

- Any public image, JS bundle, video, downloadable file.
- API responses that are **cacheable and public** (e.g., top-10 lists, public profiles) via `Cache-Control: public, max-age=...`.

When NOT to use:

- Private or personalized content (per-user feeds, account pages) unless you cache per-user at the edge with a short TTL — tricky.
- Write traffic. CDNs are read-through, not write-through.

## Database connection pooling

Every request opening a new DB connection is the single most common self-inflicted bottleneck. Postgres' default `max_connections` is 100; at 1 000 rps with 50 ms queries you already need thoughtful pooling.

### Application-side pool (first line)

Most ORMs and drivers pool by default: ADO.NET in .NET pools per connection string. Tune:

- **Max pool size** — cap well below the DB's `max_connections`, and remember N app instances × max = total.
- **Min pool size** — keep a warm floor to avoid cold-connection latency under burst.
- **Timeout** — fail fast when exhausted; don't queue forever.

### External pooler (second line)

When you have many app instances (e.g., serverless, containers, high horizontal count), an external pooler multiplexes many client connections into few DB connections.

| Pooler | DB | Mode | Notes |
|---|---|---|---|
| **PgBouncer** | Postgres | transaction / session / statement | Most common Postgres pooler; transaction-pooling mode blocks session-level features (prepared stmts, SET, advisory locks) unless configured |
| **pgcat** | Postgres | transaction + sharding | Modern PgBouncer alternative with load balancing and sharding |
| **ProxySQL** | MySQL | Query-level | Supports read/write splitting, query rewriting |
| **RDS Proxy / Azure Database Proxy** | Managed | transaction | Managed poolers for serverless-heavy workloads |

> In transaction-pooling mode (PgBouncer), prepared statements and session-scoped state are NOT safe. Check your ORM — EF Core 7+ works, but always validate with your driver.

## The scaling ladder (in order)

When asked *"how would you scale this?"*, walk down this ladder and stop when the math says you're fine:

1. **Scale up** the single box. Cheapest, least complex.
2. **Cache** aggressively (CDN for static, Redis for hot keys). See [Caching Strategies](05-caching-strategies.md).
3. **Stateless app tier + LB** — horizontal scale of the compute layer.
4. **Read replicas** — offload reads.
5. **Async work** — move slow paths to a queue + workers.
6. **Denormalize / CQRS** — separate read and write models.
7. **Shard** — last resort for write-bound systems.
8. **Multi-region** — latency, compliance, or DR; never "because we can".

Each step buys roughly one order of magnitude of capacity. If you need five steps on day one, you either have real scale or you're over-engineering — usually the latter.

---

[← Previous: Capacity Estimation](02-capacity-estimation.md) | [Next: Load Balancing →](04-load-balancing.md) | [Back to index](README.md)
