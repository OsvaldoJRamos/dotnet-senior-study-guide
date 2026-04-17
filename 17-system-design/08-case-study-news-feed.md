# Case Study: News Feed

The canonical "social network home timeline" problem — Twitter/X, Facebook, Instagram, LinkedIn all solved variants. The interesting decision is **when to do the work**: at write time (fan-out on write), at read time (fan-out on read), or both (hybrid).

## Requirements

### Functional

- `POST /posts` — user publishes a short post.
- `GET /feed` — user sees a reverse-chronological (or ranked) feed of posts from people they follow.
- `POST /follow/{userId}`, `DELETE /follow/{userId}` — follow graph maintenance.

### Non-functional (declare assumptions)

- 300 M DAU, 100 M posts/day.
- Average user follows ~200; p99 follower count >> 10 M (celebrities).
- Feed load is the hot path: p95 < 200 ms.
- Reads dominate writes ~100:1 (users refresh much more than they post).
- Freshness: posts should appear in followers' feeds within seconds; strict ordering not required.

### Capacity napkin-math

```text
posts/sec avg = 1e8 / 86 400 ≈ 1 160/s; peak ~3 k/s
feed reads/sec avg = 3e8 × (~10 feed loads/day) / 86 400 ≈ 35 k/s; peak ~80 k/s

avg followers × writes = 200 × 1 160 ≈ 230 k fan-outs/sec (naive)
```

That 230 k fan-outs/sec drives the whole design.

## API

```http
POST /posts
{
  "text": "Hello world",
  "mediaId": "..."
}

GET /feed?limit=20&cursor=<opaque>
{
  "items": [ { "postId":"...", "authorId":"...", "text":"...", "createdAt":"..." } ],
  "nextCursor": "..."
}
```

## Data model

Three core entities, stored in different best-fit storage:

| Entity | Storage | Why |
|---|---|---|
| **Posts** | Sharded SQL or KV (Cassandra, DynamoDB) keyed by `postId` | Immutable, high write throughput, point reads |
| **Follow graph** | SQL or a graph-friendly KV (Redis sorted sets, Cassandra) | Frequently queried: "who follows X?", "who does X follow?" |
| **Feed timeline per user** | Redis sorted set (or ZSET per user) / Cassandra wide row | Bounded recent window; sorted by time; fast push/pop |

## The central design choice

### Fan-out on write (push)

When a user posts, **write the post's ID into each follower's pre-computed timeline**.

```text
  Post from user A (followers: B, C, D)
  ──► Post ID stored once in Posts
  ──► Post ID appended to B.timeline, C.timeline, D.timeline
  Read:  GET C.timeline → list of IDs → hydrate from Posts
```

| Pros | Cons |
|---|---|
| Feed read is O(1): one lookup per user | Write amplification: a post by a celebrity = 10 M fan-outs |
| Works great for the long tail of normal users | Wasted work for inactive followers |
| | Delete/edit propagation is expensive |

### Fan-out on read (pull)

Timeline is computed at read time by merging recent posts from all users the reader follows.

```text
  Post from A → stored in Posts only
  C opens feed → query "posts by all users C follows, in last 7 days, top 20"
             → merge-sort the results
```

| Pros | Cons |
|---|---|
| Cheap writes — one insert per post | Read is O(users followed × posts each) |
| Deletes/edits trivially consistent | Slow for users who follow thousands |
| No wasted work for inactive followers | Hard to hit sub-second p95 at scale |

### Hybrid — the production answer

Commonly described (cf. highscalability.com write-ups of Twitter/Instagram/Facebook architectures) as a mix:

- **Fan-out on write** for most users. Pre-compute timelines for the long tail.
- **Fan-out on read** for **celebrities** (users with >> average followers — threshold around 1 M is a commonly cited cutoff). Don't fan out celebrity posts to 10 M inboxes; at read time, merge-in recent celebrity posts from users-you-follow-who-are-celebrities.

> "Commonly described" because the exact thresholds and implementations are internal. The architecture pattern is public (engineering blogs, talks) but don't quote internal numbers unless you can cite them.

Pseudocode for the hybrid read:

```csharp
public async Task<Feed> GetFeedAsync(string userId, int limit = 20)
{
    // 1. Pre-computed timeline (fan-out on write) — covers normal followed users
    var precomputed = await _redis.SortedSetRangeByScoreAsync($"timeline:{userId}", take: limit);

    // 2. Celebrities the user follows — fan-out on read
    var celebs = await _followGraph.GetCelebrityFollowsAsync(userId);
    var celebPosts = await Task.WhenAll(
        celebs.Select(c => _posts.GetRecentAsync(c, take: limit)));

    // 3. Merge, sort by time (or ranked score), take top N
    return Merge(precomputed, celebPosts.SelectMany(p => p))
        .OrderByDescending(p => p.CreatedAt)
        .Take(limit)
        .ToFeed();
}
```

## Storage layout

### Per-user timeline (Redis sorted set)

```text
key:   timeline:<userId>
score: post created_at (epoch ms)
value: postId

ZADD timeline:C 1713225600000 postId123
ZREVRANGE timeline:C 0 19   -- top 20 newest
```

Cap the timeline to a bounded window (e.g., latest 1 000 IDs): `ZREMRANGEBYRANK timeline:C 0 -1001` after each insert. Storing unbounded per-user timelines blows out memory.

### Posts (Cassandra-style wide table or sharded SQL)

```text
Partition key: postId (or authorId for list-by-author queries)
Columns: authorId, text, mediaId, createdAt, flags
```

Accessed primarily as point reads (`SELECT * WHERE postId = ?`), so any KV/wide-column store works.

### Follow graph

Two views needed:

- `following(userId) → set of user IDs` — read at feed time.
- `followers(userId) → set of user IDs` — read at post time for fan-out.

Bi-directional edges stored as two rows (denormalized for read speed).

## Fan-out pipeline

Fan-out must NOT be on the synchronous write path. That would make a celebrity's tweet take seconds.

```text
  POST /posts
  ─► write to Posts store
  ─► write to author's own timeline
  ─► enqueue FanOutJob(postId, authorId)  <── async
  ─► return 201 to user (fast)

  FanOutWorker (Kafka consumer):
  ─► load follower list for authorId
  ─► if author is celebrity → skip bulk fan-out (hybrid)
  ─► otherwise push postId onto each follower's timeline sorted set
  ─► throttle / chunk to avoid Redis / DB hot-spot
```

At peak 230 k fan-outs/sec average (ignoring celebrities), the fan-out workers are what you scale horizontally. Each worker writes to Redis in pipelines.

## Feed hydration

Timelines store only **post IDs**, not post bodies. On read:

1. Fetch top-N IDs from timeline ZSET.
2. Parallel fetch posts from Posts store (`mget` / batched reads).
3. Hydrate author info (name/avatar) from a cache of users.
4. Apply per-user filters (muted users, blocked users).
5. Optionally, rank.

## Ranking

If the feed is not strictly reverse-chronological (Instagram, Facebook style), a ranking step re-scores recent candidates using an ML model (engagement probability). Two-stage:

1. **Candidate generation** — pull ~500 recent posts via the timeline system.
2. **Ranking** — score them with a lightweight model (gradient-boosted trees or a small NN) and return the top 20.

Ranking is a whole additional topic; in an interview, **mention it exists** and acknowledge trade-offs (freshness vs relevance, cold-start users).

## Failure modes

| Failure | Impact | Mitigation |
|---|---|---|
| Fan-out queue backlog | Feeds get stale for new posts | Scale consumers; auto-scale on queue depth |
| Celebrity mega-post | Huge fan-out fights for Redis capacity | Hybrid routing skips bulk fan-out; fold in at read |
| Follow graph hot partition | A celebrity's followers live in one partition | Vertical partitioning: split followers lists across ranges |
| Redis instance loss | Some users' timelines disappear | Rebuild from Posts (replay recent posts for that user set) — slow but recoverable; keep replicas |
| Post edit / delete | Must propagate to many inboxes | Usually **lazy**: check post validity at hydration time; if deleted, skip it in the response (avoid touching every timeline) |

## Deletes and edits

Key insight: **don't update every follower's timeline when a post is edited or deleted**. That would cost another fan-out. Instead, **filter at hydration**: if the Posts store says "deleted" or "edited-at version 3", apply that at feed build. The post ID stays in timelines but doesn't render.

## What to NOT do

- Don't build fan-out on the write critical path — it makes posts slow and unreliable.
- Don't fan out to celebrities' 10 M followers — it's write amplification you'll regret.
- Don't keep full post bodies in the per-user timeline — memory explodes and edits become impossible.
- Don't ignore ranking if the interviewer brings it up; at least mention candidate generation + ranker.
- Don't promise strict ordering across the global feed — concurrent writes from different shards make that expensive. Reverse-chronological *approximately* is what the spec actually needs.

---

[← Previous: Case Study: URL Shortener](07-case-study-url-shortener.md) | [Next: Case Study: Chat System →](09-case-study-chat-system.md) | [Back to index](README.md)
