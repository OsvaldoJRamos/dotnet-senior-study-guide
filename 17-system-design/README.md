# 17 - System Design

The system design round is where a senior candidate is separated from a mid. It's not about picking the "right" tech — it's about driving the hour: clarify requirements, estimate scale, sketch a defensible high-level, then deep-dive where the interviewer pushes. This section gives you a reusable framework, the math you need to do out loud, and three canonical case studies.

## Contents

1. [System Design Interview Framework](01-system-design-interview-framework.md) - How to structure the hour and common failure modes
2. [Capacity Estimation](02-capacity-estimation.md) - Back-of-envelope math, powers of 2, latency numbers every programmer should know
3. [Scaling Strategies](03-scaling-strategies.md) - Vertical vs horizontal, stateless services, read replicas, sharding, CDN, connection pooling
4. [Load Balancing](04-load-balancing.md) - L4 vs L7, algorithms, health checks, global LB, sticky sessions
5. [Caching Strategies](05-caching-strategies.md) - Cache-aside, read/write-through, write-back, TTL, stampede mitigation, Redis vs Memcached
6. [Rate Limiting](06-rate-limiting.md) - Token bucket, leaky bucket, windows, distributed rate limiting, `System.Threading.RateLimiting`
7. [Case Study: URL Shortener](07-case-study-url-shortener.md) - Requirements, ID generation (base62, counter, Snowflake), schema, caching
8. [Case Study: News Feed](08-case-study-news-feed.md) - Fan-out on write vs read, hybrid, feed generation, storage
9. [Case Study: Chat System](09-case-study-chat-system.md) - WebSocket delivery, presence, message store, read receipts, group chat

---

[Back to index](../README.md)
