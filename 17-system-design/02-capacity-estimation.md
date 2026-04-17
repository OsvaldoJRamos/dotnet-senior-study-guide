# Capacity Estimation

Back-of-envelope math you do in 3-5 minutes on the whiteboard. The interviewer isn't grading the number — they're grading whether you can reason about scale before choosing tech.

## Useful constants

### Powers of 2

| Power | Approx | Nickname |
|---|---|---|
| 2^10 | ~10^3 | thousand (K, 1 KB) |
| 2^20 | ~10^6 | million (M, 1 MB) |
| 2^30 | ~10^9 | billion (B, 1 GB) |
| 2^32 | ~4 × 10^9 | 4 billion (IPv4 space, unsigned int range) |
| 2^40 | ~10^12 | trillion (T, 1 TB) |
| 2^50 | ~10^15 | quadrillion (1 PB) |
| 2^63 | ~9.2 × 10^18 | signed long max |

### Time

| Unit | Seconds |
|---|---|
| 1 day | 86 400 (~10^5) |
| 1 month (30 d) | ~2.5 × 10^6 |
| 1 year | ~3.15 × 10^7 |

> A useful shortcut: **1 day ≈ 100 000 seconds** (it's 86 400; round up). Makes *"10 M events/day"* become *"~100 events/sec"* instantly.

### Latency numbers every programmer should know

Originally popularized by Jeff Dean; values below are from Colin Scott's interactive extrapolation for ~2020 (same site tracks drift year-by-year). Order of magnitude is what matters — memorize the *shape*, not the exact digits.

| Operation | Latency (2020) |
|---|---|
| L1 cache reference | ~1 ns |
| Branch mispredict | ~3 ns |
| L2 cache reference | ~4 ns |
| Mutex lock/unlock | ~17 ns |
| Main memory reference | ~100 ns |
| Compress 1 KB with Zippy/Snappy | ~2 µs |
| Send 1 KB over 1 Gbps network | ~10 µs |
| SSD random read | ~16 µs |
| Read 1 MB sequentially from memory | ~3 µs |
| Round trip within same datacenter | ~500 µs |
| Read 1 MB sequentially from SSD | ~49 µs |
| Disk seek | ~3 ms |
| Read 1 MB sequentially from disk | ~825 µs |
| Packet CA → Netherlands → CA | ~150 ms |

Source: [Colin Scott — Interactive Latency](https://colin-scott.github.io/personal_website/research/interactive_latency.html). Note: many often-quoted lists are Jeff Dean's original ~2012 figures; SSD, memory, and network numbers drift every year. Quote the general shape and cite the year.

### Rules of thumb you'll reuse

| Thing | Order of magnitude |
|---|---|
| HTTP hop (same region, healthy service) | 1-10 ms |
| Cross-region RTT (same continent) | 30-80 ms |
| Cross-continent RTT | 100-300 ms |
| Redis GET (same DC) | < 1 ms |
| Postgres point read (warm) | 1-5 ms |
| S3 / blob storage GET | 20-100 ms |
| Kafka producer ack (acks=all) | 5-20 ms |

## The four-step estimation

Do these in order. Write each on the board.

### 1. QPS (reads and writes)

```text
DAU × actions-per-user-per-day / seconds-per-day = avg QPS
```

Example — Twitter-like feed, 300 M DAU, each reading 100 tweets/day:

```text
reads/sec = 300e6 × 100 / 86 400 ≈ 347 k reads/sec  (avg)
peak ≈ 2× avg ≈ 700 k reads/sec
```

**Always compute reads and writes separately** — they often differ by 10-100×, and they pick different architectures (read replicas vs sharding).

### 2. Storage (with replication)

```text
events-per-day × bytes-per-event × retention-days × replication = total storage
```

Example — chat system, 300 M messages/day, 500 bytes each, 5-year retention, 3× replication:

```text
= 3e8 × 500 × 365 × 5 × 3
≈ 0.8 PB
```

Add overhead for indexes (20-50%), metadata, and hot-vs-cold tiering.

### 3. Bandwidth (ingress/egress)

```text
writes/sec × avg-write-size = ingress
reads/sec × avg-read-size = egress
```

Egress is usually what you pay for; it dominates the bill on AWS. A 300-KB image served at 100 k req/s is 30 GB/s of egress — CDN territory, not origin.

### 4. Cache size

Target the hot set — typically the top 10-20% of keys serving 80% of traffic (Pareto / power-law).

```text
cache_size = items_in_hot_set × item_size
```

For 1 B total tweets where the top 20% drive 80% of reads, 500 bytes each:

```text
= 1e9 × 0.2 × 500 = 100 GB
```

That fits in a handful of Redis nodes.

## A worked example (Instagram-like)

Interviewer: *"Design an Instagram-like photo service."*

Assumptions (say them out loud, write them):

- 500 M DAU
- Each user posts 1 photo/day (writes)
- Each user views 50 photos/day (reads)
- Photo = 500 KB after compression
- Retention: forever
- Replication factor: 3

**QPS**:

```text
writes/sec = 5e8 / 86 400 ≈ 5 800 writes/sec  → peak ~12 k
reads/sec  = 5e8 × 50 / 86 400 ≈ 290 k reads/sec → peak ~600 k
```

**Storage (year 1)**:

```text
photos/yr = 5e8 × 365 = 1.8e11
bytes/yr = 1.8e11 × 500 KB × 3 replicas ≈ 270 PB/yr
```

Obvious conclusion: **object storage (S3/Blob) for photos, not a database**. The database stores metadata only.

**Bandwidth (egress)**:

```text
egress = 290 k × 500 KB ≈ 145 GB/s  (avg)
```

Single-origin cannot serve 145 GB/s — this is **CDN territory**, full stop. The DB and app layer never serve the photo bytes.

From 5 minutes of math you've already justified: object storage, metadata DB, CDN, and a read replica topology. That's the interviewer's cue to deep-dive.

## What NOT to do

- Don't over-precision the arithmetic. *"Roughly 300 k"* beats *"293 752"*.
- Don't compute what you won't use. If you never bring up bandwidth, don't estimate it.
- Don't skip peak. Real traffic is spiky; 2-3× avg is the usual multiplier.
- Don't forget replication and indexes in storage — a 1 TB dataset is rarely 1 TB on disk.
- Don't pretend you memorized the latency table. Say *"roughly a millisecond for RAM-MB-read, ~500 µs for same-DC RTT, ~150 ms CA-to-EU"* — interviewers know these drift.

---

[← Previous: System Design Interview Framework](01-system-design-interview-framework.md) | [Next: Scaling Strategies →](03-scaling-strategies.md) | [Back to index](README.md)
