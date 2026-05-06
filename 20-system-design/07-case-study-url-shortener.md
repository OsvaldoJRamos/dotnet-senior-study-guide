# Case Study: URL Shortener

A classic interview warm-up — deceptively simple, with real nuances in ID generation, consistency, and analytics. Think "bit.ly".

## Requirements

### Functional

- `POST /urls` — create a short URL from a long URL; optionally custom alias, expiration.
- `GET /{short}` — redirect to the long URL.
- `GET /urls/{short}/stats` — click count, referrers, timestamps.

### Non-functional (assumptions — declare them)

- 100 M new URLs/day at peak (1.2 k writes/sec; peak ~3 k).
- 10 B redirects/day → ~115 k reads/sec; peak ~250 k.
- Read:write ratio ~100:1 (typical link use).
- p95 redirect latency < 50 ms globally.
- Links survive ~5 years unless explicitly expired.
- No strong consistency between regions needed for redirects; eventually-consistent is fine (a just-created link being unavailable in another region for a few seconds is acceptable).

### Capacity napkin-math

```text
writes = 100e6 / 86 400 ≈ 1 160/s (avg),  peak ~3 k/s
reads  = 10e9 / 86 400 ≈ 115 k/s (avg), peak ~250 k/s

storage/yr = 100e6 × 365 × 500 bytes × 3 replicas ≈ 55 TB/yr
```

Metadata fits in any modern database; redirects are read-dominated → caching + replicas are the levers.

## API

```http
POST /urls
Content-Type: application/json
Authorization: Bearer ...

{
  "longUrl": "https://example.com/very/long/path?x=1",
  "customAlias": "promo-2026",     // optional
  "expiresAt": "2026-12-31T00:00:00Z"
}

→ 201 Created
{
  "shortUrl": "https://sho.rt/promo-2026",
  "shortCode": "promo-2026",
  "expiresAt": "2026-12-31T00:00:00Z"
}
```

```http
GET /promo-2026
→ 301 Moved Permanently
Location: https://example.com/very/long/path?x=1
Cache-Control: private, max-age=60
```

> 301 is cacheable ~forever by browsers/proxies, which is great for throughput but blocks analytics. Use **302 Found** (or **307 Temporary Redirect**) when you want to count each click. Most production shorteners use 302.

## Data model

One table; denormalized deliberately.

```sql
CREATE TABLE short_urls (
    short_code   VARCHAR(10) PRIMARY KEY,  -- or a binary key
    long_url     TEXT NOT NULL,
    owner_id     BIGINT,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at   TIMESTAMPTZ,
    click_count  BIGINT NOT NULL DEFAULT 0  -- denormalized for hot reads
);

CREATE INDEX ix_short_urls_owner ON short_urls(owner_id);
```

Click events go to a **separate append-only store** (see [analytics](#analytics-pipeline)) — not to this table on the critical path.

## ID generation — the interesting part

The short code is the primary design decision. Three viable approaches.

### Option A — Counter + base62

A global 64-bit counter. Encode to base62 (`0-9a-zA-Z`) for a compact, URL-safe string.

```csharp
// Base62 alphabet — no ambiguous characters, URL-safe
private static readonly char[] Alphabet =
    "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ".ToCharArray();

public static string ToBase62(long n)
{
    if (n == 0) return "0";
    var sb = new StringBuilder();
    while (n > 0)
    {
        sb.Insert(0, Alphabet[n % 62]);
        n /= 62;
    }
    return sb.ToString();
}
```

Space: 62^7 ≈ 3.5 trillion codes in 7 chars, 62^8 ≈ 218 trillion in 8 chars. Comfortable for the lifetime of any product.

| Pros | Cons |
|---|---|
| Sequential, compact, nicely compressible | Centralized counter is a bottleneck at scale |
| Easy to reason about | Leaks total URL count (sequential IDs) |

**How to avoid the counter bottleneck**: each app instance **pre-allocates a range** (e.g., reserve 10 000 IDs from a central service, burn through them locally). Central service just does `UPDATE counters SET value = value + 10000`. This pattern is called an **ID range allocator** — essentially the Flickr ticket-server / HiLo approach.

### Option B — Snowflake-style ID

Twitter Snowflake: 64-bit ID composed of `timestamp | machine_id | sequence`. Layout of Twitter's original design (see the [archived `IdWorker.scala`](https://github.com/twitter-archive/snowflake/blob/snowflake-2010/src/main/scala/com/twitter/service/snowflake/IdWorker.scala) and the [Wikipedia article](https://en.wikipedia.org/wiki/Snowflake_ID)):

- **1 bit** sign (unused, zero).
- **41 bits** timestamp in milliseconds since a custom epoch — the archived source sets `twepoch = 1288834974657L`, which is 2010-11-04 01:42:54 UTC; 41 bits gives roughly 69 years of range.
- **10 bits** machine id (up to 1 024 workers) — Twitter's `IdWorker` splits this 5/5 into `datacenterIdBits = 5` and `workerIdBits = 5`.
- **12 bits** sequence (up to 4 096 IDs per ms per worker).

Then base62-encode to get a short string (~11 chars).

| Pros | Cons |
|---|---|
| No central coordination once workers have unique IDs | Each ID is 11+ chars in base62 — longer than needed for URL-shortener scale |
| Roughly time-ordered | Slight clock-skew risk across workers |

Snowflake is overkill for a URL shortener; it shines for ordered distributed IDs (tweets, orders). Good to mention; rarely picked in this case study.

### Option C — Random short code with collision check

Generate a random 7-char base62 string; insert with a unique constraint; retry on collision.

| Pros | Cons |
|---|---|
| Stateless, no coordination | Collision rate rises as the keyspace fills — becomes slow |
| Doesn't leak URL count | Requires a uniqueness check (index hit) per write |

At 100 M codes out of 62^7 ≈ 3.5 T, collision probability per write is ~3 × 10^-5 — essentially nil. Viable in early life; degrades as the namespace fills.

### Recommendation

**Pre-allocated counter + base62** is the senior default — simple, no collision checks, sequential writes are cache-friendly. Mention Snowflake as the distributed alternative; mention random when you want non-guessable links (but note an attacker can still enumerate the keyspace).

## High-level architecture

```text
     ┌──────┐                         ┌──────────────┐
     │ CDN  │── cache hit (302 rare) │ Edge cache   │
     └──┬───┘                         │ (per PoP)    │
        │                             └──────────────┘
        ▼
   ┌─────────┐   hot key?   ┌────────┐
   │ Gateway │────────────► │ Redis  │
   │  + LB   │              └────┬───┘
   └────┬────┘                   │ miss
        │                        ▼
        ▼                  ┌───────────┐
   ┌─────────┐  write path │ Postgres  │  (sharded by short_code hash
   │ Writes  │────────────►│ (primary) │   once one primary isn't enough)
   └─────────┘             └─────┬─────┘
                                 │ async
                                 ▼
                           ┌───────────┐
                           │ Replicas  │ (reads)
                           └───────────┘

   Redirect side-effect:  fire-and-forget to Kafka → analytics pipeline
```

## Caching strategy

The redirect path is the hot path. Cache aggressively:

- **Edge cache (CDN)** — for popular codes, a PoP can serve the 302 without touching origin. Short TTL (60-300 s) so expiration/deletion is timely. Uses `Cache-Control: public, max-age=300`.
- **Redis** in front of the DB — cache-aside, `short_code → long_url`. TTL proportional to expected access pattern (e.g., 1 hour with jitter).
- **Negative caching** — cache *"this code does not exist"* for a short TTL (e.g., 10 s) to protect the DB from 404-spam attacks.

Hot-set math: top 20% of codes ≈ 20 M entries × ~500 bytes ≈ 10 GB in Redis. Fits on a single node; scale out only when needed.

### Stampede protection

A single viral link can suddenly produce 100 k RPS. The CDN absorbs most, but on CDN expiry Redis sees the shockwave. Use **single-flight locking** on the cache-aside path (see [Caching Strategies](05-caching-strategies.md#cache-stampede-thundering-herd)).

## Write path

1. Validate input (reject malformed URLs, known-bad domains).
2. Allocate a short code (from the pre-allocated range, or use custom alias after uniqueness check).
3. `INSERT INTO short_urls ...`.
4. Don't pre-populate the cache on write — let the first read do it. Pre-populating on every write is wasted work for cold URLs.
5. Return 201 with the short URL.

## Read path

1. Normalize incoming path to `short_code`.
2. Check Redis; on hit, return 302.
3. On miss, take the single-flight lock, query the DB (prefer replica for redirects — a ~seconds-old replica is fine).
4. Populate cache with TTL + jitter.
5. Return 302.
6. Enqueue the click event (Kafka or Kinesis) — non-blocking.

## Analytics pipeline

Do NOT update `click_count` on the hot path — a per-click UPDATE at 250 k RPS will kill the DB.

```text
   Redirect handler  ──enqueue──►  Kafka (topic: click_events)
                                         │
                                         ▼
                                  ┌──────────────┐
                                  │  Consumer    │ batches rows
                                  └──────┬───────┘
                                         ▼
                                  ┌──────────────┐
                                  │  OLAP store  │ (ClickHouse, BigQuery,
                                  │              │  Redshift, DuckDB, etc.)
                                  └──────────────┘
```

Event shape — denormalized for analytics (don't normalize across dimensions):

```json
{
  "shortCode": "promo-2026",
  "timestamp": "2026-04-16T12:34:56Z",
  "ip": "203.0.113.7",
  "country": "BR",
  "userAgent": "...",
  "referrer": "https://twitter.com/...",
  "clickId": "01HXABC..."
}
```

Aggregate stats via scheduled rollups (hourly/daily) into `click_stats` tables for fast `/stats` reads, or query the OLAP store directly.

## Deletion and expiration

- Expired URLs: either a nightly job to clean up, or filter on read (`WHERE expires_at > NOW()`). Filter-on-read avoids write amplification; sweep-and-delete keeps storage bounded.
- When deleting from DB, also **delete from cache** (or let TTL expire) — preferably delete, because a 302 to a revoked URL is a serious bug.

## Failure modes to raise in the interview

| Failure | Impact | Mitigation |
|---|---|---|
| Redis down | Every read hits the DB; DB may collapse | Run DB with enough headroom to serve ~30% of cached reads; circuit-breaker to shed load if unhealthy |
| DB primary down | Writes fail; reads on replicas still work | Managed HA (RDS / Azure SQL) with auto-failover; accept a few minutes of write unavailability |
| Hot link (viral) | Cache + DB overloaded for that one key | CDN absorbs; single-flight lock on origin; serve stale if recent |
| Counter allocator down | New URL creation fails | Fall back to random code with collision check; each worker keeps a pre-fetched range of several thousand IDs |
| Abuse — someone shortens malware | Reputation and legal | URL reputation checks (Google Safe Browsing), abuse-report endpoint, block-list |

## What to NOT over-engineer

- Don't reach for Cassandra on day one — Postgres + read replicas + Redis handles this workload easily. Sharding is a later concern.
- Don't invent your own ID algorithm. Base62 over a counter is boring and correct.
- Don't put the analytics pipeline on the critical path. It's a side-effect.

---

[← Previous: Rate Limiting](06-rate-limiting.md) | [Next: Case Study: News Feed →](08-case-study-news-feed.md) | [Back to index](README.md)
