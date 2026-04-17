# System Design

> Read the questions, think about your answer, then click to reveal.

---

### 1. You get "design Twitter" with 10 minutes on the clock before wrap-up. How do you structure the hour?

<details>
<summary>Reveal answer</summary>

Five phases, roughly: **clarify (5 min) → estimate (5) → high-level (10) → deep dive (25) → trade-offs and wrap-up (10-15)**. The shape matters more than any individual box on the diagram.

Phase 1 pulls out **functional + non-functional + out-of-scope**. Pin scale numbers ("300 M DAU, 100 M posts/day"), latency budgets, and the read:write ratio. Phase 2 does QPS, storage, bandwidth — write them on the board so you reference them later. Phase 3 is a boxes-and-arrows diagram: client → gateway → service → storage, with cache/queue/CDN added only as NFRs justify. Phase 4 is wherever the interviewer pushes: feed generation, storage choice, sharding, failure mode. Phase 5 is bottlenecks, alternatives considered, and what you'd build next.

The bread-and-butter senior mistake: jumping to Kafka-and-Cassandra before requirements. Always pull requirements first.

Deep dive: [System Design Interview Framework](../17-system-design/01-system-design-interview-framework.md)

</details>

---

### 2. The interviewer says "100 M DAU". Walk me through the napkin math you do out loud.

<details>
<summary>Reveal answer</summary>

Four numbers, in order:

1. **QPS reads and writes separately.** `DAU × actions/day / 86 400`. Often reads dominate writes 10-100×; they scale differently (read replicas vs sharding).
2. **Storage** including replication factor and retention. `events × size × days × replicas`.
3. **Bandwidth** especially egress — it's what you pay for. 300 KB responses × 100 k RPS = 30 GB/s, which is CDN territory.
4. **Cache size** for the hot set — Pareto: top 20% of keys is 80% of reads. `hot_items × item_size`.

Useful shortcuts: **1 day ≈ 100 k seconds**, so "10 M events/day ≈ 100/sec". Peak ≈ 2-3× average.

Memorize the *shape* of Jeff Dean's latency numbers: L1 ~1 ns, memory ~100 ns, same-DC RTT ~500 µs, cross-continent ~150 ms. Cite that they drift annually.

Deep dive: [Capacity Estimation](../17-system-design/02-capacity-estimation.md)

</details>

---

### 3. Explain the scaling ladder. When do you shard?

<details>
<summary>Reveal answer</summary>

The ladder, in order of cheapest / least complex: (1) scale up the box, (2) cache (CDN for static, Redis for hot), (3) stateless app tier + LB, (4) read replicas, (5) async work via queues, (6) denormalize / CQRS, (7) shard, (8) multi-region.

Each step buys roughly one order of magnitude. Stop at the first one that works.

Shard **only** when writes exceed what a single primary can handle — reads can usually be solved with replicas and caching. Shard key options: hash (even but no range queries, painful resharding), range (range-friendly but hot shards), geographic (locality but hot regions), consistent hashing (minimal data movement on scale-out). Once you shard you lose cross-shard joins, transactions, and unique constraints across shards.

Sharding is a one-way door. If you shard on day one, you're probably over-engineering.

Deep dive: [Scaling Strategies](../17-system-design/03-scaling-strategies.md)

</details>

---

### 4. L4 vs L7 load balancer — when do you pick each? What changes with gRPC?

<details>
<summary>Reveal answer</summary>

**L4** operates on the 5-tuple (source/dest IP + port + protocol). Pass-through, very fast, supports any TCP/UDP. AWS NLB and Azure Load Balancer are L4.

**L7** parses HTTP — path, host, headers, cookies. Routes per-request, can do WAF, auth, HTTP health checks, TLS termination. AWS ALB and Azure Application Gateway / Front Door are L7.

Pick L4 for non-HTTP (Redis, PostgreSQL, WebSocket at huge scale) or when you need pass-through and ultra-low latency. Pick L7 for HTTP APIs that need path/host/header routing and richer observability.

**gRPC/HTTP/2 gotcha**: HTTP/2 multiplexes many requests over one long-lived TCP connection. An L4 LB hashes on the 5-tuple and will pin **all** requests from a client to one backend — no load balancing. Use an L7 LB that balances per-request (Envoy, ALB, NGINX in HTTP/2 mode).

Deep dive: [Load Balancing](../17-system-design/04-load-balancing.md)

</details>

---

### 5. A popular product page is getting slammed. Cache-aside, read-through, write-through, write-back — which one fits, and what's the stampede risk?

<details>
<summary>Reveal answer</summary>

**Cache-aside** is the default: app reads cache, on miss reads the source and populates. Simple, degrades to direct reads if cache dies.

- **Read-through** puts the fetch inside the cache library — cleaner code, needs a cache that supports loaders.
- **Write-through** writes to both cache and DB synchronously — no staleness but write-amplification.
- **Write-back** writes to the cache; async flush to DB — fastest writes, but **losing the cache loses writes** (use only for recomputable data like counters).

For a hot product page: cache-aside with TTL + jitter.

**Stampede** happens when a hot key expires: N concurrent misses all hit the source. Mitigations: **single-flight locking** (first miss takes a Redis `SET NX` lock with a short TTL, others wait or serve stale), **probabilistic early expiration** (XFetch-style), **stale-while-revalidate** (serve stale while background refresh runs).

Always jitter TTLs to avoid synchronized expiries. Prefer **delete-on-write** over update-on-write; update races with concurrent cache-aside re-populations.

Deep dive: [Caching Strategies](../17-system-design/05-caching-strategies.md)

</details>

---

### 6. Token bucket vs leaky bucket vs sliding window — when each, and what does `System.Threading.RateLimiting` ship?

<details>
<summary>Reveal answer</summary>

- **Fixed window**: trivial, boundary-burst problem (100 at 10:00:59 + 100 at 10:01:00).
- **Sliding window**: no boundary burst; counter-based is an approximation.
- **Token bucket**: refills at rate R, bucket of capacity C — burst up to C, sustained rate R. Matches real quotas.
- **Leaky bucket**: queue drained at fixed rate — strictly smooths output, no burst.

Token bucket → user-facing quotas with burst tolerance. Leaky bucket → protecting a rate-sensitive downstream. Sliding window → fairness without the boundary-burst headache.

.NET ships `System.Threading.RateLimiting` (available in ASP.NET Core 7+) with `FixedWindowRateLimiter`, `SlidingWindowRateLimiter`, `TokenBucketRateLimiter`, `ConcurrencyLimiter`, plus `PartitionedRateLimiter<TResource>` to partition by user/IP/tenant.

**In-process** though — with N app instances the effective cap is N × limit. For a global cap, push to Redis (atomic INCR + EXPIRE, or a Lua token-bucket) or to an edge/gateway that rate-limits before your code runs. Creating partitions on user-controlled input (raw IP) is a DoS vector; cap partition cardinality.

Deep dive: [Rate Limiting](../17-system-design/06-rate-limiting.md)

</details>

---

### 7. Design a URL shortener. How do you generate short codes, and why?

<details>
<summary>Reveal answer</summary>

Three viable approaches:

- **Counter + base62**: 64-bit counter, base62-encoded (7 chars = 62^7 ≈ 3.5 T codes). Simple, sequential, cache-friendly. Central counter is the bottleneck — fix with **pre-allocated ranges**: each app instance grabs 10 000 IDs from a central allocator and burns them locally.
- **Snowflake**: 64-bit ID with `timestamp | machine_id | sequence` (Twitter's original design: 41 + 10 + 12 bits + 1 sign bit, per the Wikipedia description). No coordination needed but longer codes (~11 chars in base62). Overkill here; shines for ordered distributed IDs.
- **Random with collision check**: unique constraint + retry. Stateless but collision rate grows as the keyspace fills.

Senior default: **counter + base62 with pre-allocated ranges**. Mention Snowflake as the distributed alternative. Note random if unguessable URLs are a security requirement (but the namespace is still enumerable).

Architecture: Redis in front of Postgres for redirects, CDN for popular codes, click events fan out async to Kafka → OLAP (ClickHouse/BigQuery) for analytics — do NOT update `click_count` on the hot path. Use 302 (not 301) if you need to count every click.

Deep dive: [Case Study: URL Shortener](../17-system-design/07-case-study-url-shortener.md)

</details>

---

### 8. News feed: fan-out on write vs read. Why is production usually hybrid?

<details>
<summary>Reveal answer</summary>

**Fan-out on write (push)**: on post, push the post ID into every follower's pre-computed timeline. Read is O(1) — one lookup per reader. Write amplification kills you for celebrities: a post to 10 M followers = 10 M inserts.

**Fan-out on read (pull)**: post is stored once. At read time, merge recent posts from everyone the reader follows. Cheap writes, but reads scale with following count — painful for users who follow thousands.

**Hybrid (commonly described for Twitter, Instagram, Facebook — don't quote internal thresholds you can't cite)**: fan-out on write for the long tail, fan-out on read for celebrities (threshold around 1 M followers is cited). The reader's feed = (pre-computed timeline) ∪ (on-demand merge of recent celebrity posts for people they follow).

Important: do fan-out **off** the write path (enqueue to Kafka; worker writes to follower timelines). Store only post IDs in per-user timelines; hydrate bodies and author info on read. Handle deletes/edits at hydration time — don't chase them through every inbox.

Deep dive: [Case Study: News Feed](../17-system-design/08-case-study-news-feed.md)

</details>

---

### 9. Design a chat system. How do you deliver messages reliably in order with 150 M concurrent WebSockets?

<details>
<summary>Reveal answer</summary>

- **Transport**: WebSocket (RFC 6455 — full-duplex over TLS via `wss://`). In .NET, SignalR is the sane default; raw `System.Net.WebSockets.WebSocket` for perf-critical paths.
- **Connection gateway** is stateful — a user's socket lives on exactly one node. Route pushes to that node via a back-plane: Redis Pub/Sub for small scale, Kafka / Service Bus for large, or direct gRPC between gateways with a shared user-to-node registry.
- **At-least-once delivery**: the Chat Service persists the message (the write is the commit) before ACKing the sender. Client includes a `clientMessageId` (UUID/ULID) so the server dedups on retries — otherwise network flaps produce duplicates.
- **Per-conversation ordering only** — partition Kafka and the message store by `conversationId`. Global ordering is expensive and unnecessary.
- **Message store**: wide-column DB (Cassandra / ScyllaDB / HBase / DynamoDB). Partition key = conversationId, clustering key = messageId (time-sortable). "Last 50 messages" is a trivial range scan. RDBMS works up to hundreds of thousands of writes/sec with sharding; beyond that switch.
- **Presence**: Redis with TTL heartbeats — `presence:userId → expires_at`. Fetch lazily when a user opens a chat; don't proactively broadcast every status change or the fan-out firestorm will drown you.
- **Multi-device**: echo each message back to the sender's other devices in the fan-out. Sync `read_up_to` markers across them.

150 M concurrent WebSockets → roughly 1 500 gateway nodes at 100 k sockets each. Reconnection with back-off + jitter; consistent hashing on `(userId, deviceId)` so reconnects usually land on the same node.

Deep dive: [Case Study: Chat System](../17-system-design/09-case-study-chat-system.md)

</details>

---

### 10. What are deep vs shallow health checks, and why does it matter for the LB?

<details>
<summary>Reveal answer</summary>

**Shallow (liveness)**: is the process running and bound to the port? Cheap. Misses bad downstream deps.

**Deep (readiness)**: can the instance reach its DB, cache, downstream services? Comprehensive. Coupling this to your LB health probe means **one flaky DB marks every replica unhealthy** — the LB empties the pool and you 5xx everything.

The right split: **LB checks shallow**, **readiness deep for deploy-gating and pod-ready signals**. Kubernetes encodes this split explicitly — liveness restarts the pod, readiness gates traffic. Modern LBs (Envoy, ALB) also support outlier ejection (boot a host out for a cool-down after N consecutive 5xx) and circuit breakers at the LB tier, which blunt cascading failures without over-relying on the health probe.

Deep dive: [Load Balancing](../17-system-design/04-load-balancing.md)

</details>

---

### 11. What is consistent hashing, and where does it show up in real systems?

<details>
<summary>Reveal answer</summary>

Introduced by Karger et al., *"Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web"* (STOC 1997).

Map both keys and nodes onto a hash ring. Each key is owned by the next node clockwise. Adding or removing a node relocates only ~1/N of the keys (vs (N-1)/N with naive modulo hashing). Use **virtual nodes** (each physical node occupies many ring positions) to smooth distribution — without them, a ring with few nodes is unfair.

Shows up everywhere:

- **CDN edge routing** (map client IP → PoP).
- **Sharded caches** (Memcached clients with Ketama hashing, Redis Cluster's 16 384 hash slots which are a bounded variant).
- **Distributed DBs** (DynamoDB partitioning, Cassandra's token ring).
- **Load balancers** doing session-aware routing without sticky cookies.
- **Rate limiter partitioning** — route all requests for a given user-key to one instance so the in-process counter is authoritative.

The cost: resharding is cheaper but not free, and hot keys still overload their owning node — you need replicas for popular keys.

Deep dive: [Load Balancing](../17-system-design/04-load-balancing.md) and [Scaling Strategies](../17-system-design/03-scaling-strategies.md)

</details>

---

### 12. What are the most common mistakes candidates make in a system design interview?

<details>
<summary>Reveal answer</summary>

- **Jumping to tech before requirements.** "I'd use Kafka and Cassandra" with no scale number is cargo culting.
- **Skipping NFRs.** NFRs *are* the architecture — latency, availability, consistency, geography. Without them you're guessing.
- **Boiling the ocean.** Trying to design every component equally means you run out of time before deep-diving anything.
- **Not using the capacity math.** If phase-2 numbers never inform a decision, why did you do them?
- **Over-engineering day one.** A feed for 10 k users doesn't need a Lambda architecture with CQRS, event sourcing, and multi-region active-active.
- **Ignoring the interviewer's steer.** If they ask about sharding, they want to see sharding. Don't keep redirecting.
- **Silence during deep dive.** Narrate the reasoning. The interviewer grades *reasoning*, not whiteboard aesthetics.
- **Treating stickiness as a fix.** Sticky sessions hide a state problem — externalize state and kill stickiness.
- **Building rate limiting in-process and calling it done.** N instances → real cap is N × limit. Push to Redis or the edge.
- **Promising strong consistency everywhere.** It's expensive. Call out what *can* be eventually consistent (view counts, feed order, presence) and save strong consistency for money / identity / critical invariants.

Deep dive: [System Design Interview Framework](../17-system-design/01-system-design-interview-framework.md)

</details>

---

[Back to index](README.md)
