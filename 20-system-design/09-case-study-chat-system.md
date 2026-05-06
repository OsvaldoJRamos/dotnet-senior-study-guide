# Case Study: Chat System

Think WhatsApp / Slack / Messenger. The hard parts are **real-time delivery**, **guaranteed message ordering per conversation**, **presence at scale**, and **multi-device sync**.

## Requirements

### Functional

- 1:1 and group chat.
- Send/receive messages in near real time.
- Delivery and read receipts.
- Presence (online / away / offline).
- Message history — infinite scroll.
- Multi-device (same user on phone + laptop).

### Non-functional (declare assumptions)

- 1 B MAU, 300 M DAU.
- 100 B messages/day (avg 30 per active user).
- Median p95 delivery latency < 200 ms (sender → recipient online).
- Messages **must** be delivered (at-least-once), ordered per conversation, durable for years.
- Groups up to 250 members typical; "broadcast" groups up to 100 k members.

### Capacity napkin-math

```text
messages/sec avg = 1e11 / 86 400 ≈ 1.15 M/sec; peak 2-3× ≈ 3 M/sec
storage/yr = 1e11 × 365 × 500 bytes × 3 replicas ≈ 55 PB/yr
concurrent connections = 300 M DAU × 0.5 online ≈ 150 M concurrent WebSockets
```

150 M persistent connections drives the fleet sizing: even at 100 k connections per node (a push), that's ~1 500 gateway nodes.

## High-level architecture

```text
 Client ──WebSocket──► Connection Gateway ──► Chat Service ──► Message Store
  (iOS, Android,       (one of N pods)           │              (Cassandra /
   Web, Desktop)         (stateful)              │               HBase / Scylla)
                                 │               ▼
                                 │         Queue (Kafka)
                                 │               │
                                 ▼               ▼
                        Presence Service     Fan-out Workers
                        (Redis / KV)         (push to other clients)
                                 │
                                 ▼
                          Push Notification
                          (APNs / FCM) for offline devices
```

Key separation:

- **Connection gateway** — owns the WebSocket, stateful, horizontally scaled.
- **Chat service** — stateless HTTP/gRPC worker that persists and forwards.
- **Message store** — the durable log, optimized for per-conversation time-ordered reads.
- **Presence** — fast, ephemeral, eventually consistent.

## Real-time transport

### WebSocket

WebSocket (RFC 6455) upgrades an HTTP/1.1 connection to a full-duplex TCP channel: `ws://` or `wss://` (TLS). One TCP connection, bidirectional frames, no per-message HTTP overhead.

MDN: *"The WebSocket API makes it possible to open a two-way interactive communication session between the user's browser and a server."* ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API))

In .NET, use:

- **SignalR** — higher-level (connection management, groups, reconnection, Redis back-plane). Most new .NET chat apps start here.
- **Raw `System.Net.WebSockets.WebSocket`** — if you need the last bit of perf or custom framing.

### Alternatives

| Transport | When |
|---|---|
| **WebSocket** | Default for browsers, mobile with native libs |
| **MQTT** | Low-bandwidth / IoT devices; many chat apps use MQTT over TLS (Facebook famously did) |
| **Long polling / SSE** | Restricted networks / legacy fallback; SignalR negotiates down to these if WebSocket fails |
| **HTTP/2 server push** | Not a real option — deprecated in browsers |
| **QUIC / HTTP/3** | Emerging; mobile benefits from 0-RTT reconnection |

### Connection gateway

Stateful: each user's device is pinned to one gateway node (the one holding its WebSocket). Load-balanced with sticky routing (consistent hashing on `userId+deviceId`) so reconnects usually land on the same node.

When node X wants to push a message to user Y whose connection lives on node Z, X can't send it directly over a WebSocket it doesn't own. Use a **routing back-plane**:

- **Redis Pub/Sub** — small scale, simple (SignalR's built-in Redis back-plane works this way).
- **Kafka / Service Bus** — large scale, durable, replayable.
- **Direct gRPC between gateways** — each gateway registers its owned user IDs in a shared KV; the sender looks up and RPCs directly.

## Message delivery guarantees

### At-least-once

Client SENDs `{clientMessageId, conversationId, text}`.

1. Gateway receives, forwards to Chat Service.
2. Chat Service writes to the Message Store (the write is the commit).
3. ACK back to sender with the server-assigned `messageId` and persisted timestamp.
4. Fan out to other participants via the Kafka topic.
5. Consumers push to online recipients' gateways; mark offline recipients for push notification.

### Idempotency

Client includes a `clientMessageId` (UUID/ULID). If the client times out and resends, the Chat Service **deduplicates** on `(userId, clientMessageId)` — same message-id means same message, even if the network hiccuped. Without this, at-least-once becomes "duplicated messages when the network flaps".

### Ordering

Per-conversation ordering matters; global ordering does not. Partition the Kafka topic and the message-store rows by `conversationId` so everything within a conversation goes through one partition. Within one partition, ordering is preserved; across conversations, it can be arbitrary.

### Read receipts and delivery receipts

Two levels:

- **Delivered** — the recipient's device has received the message (ack from client over the WebSocket).
- **Read** — the user actually opened the chat (client sends a `read_up_to` marker).

Both are stored per-recipient in a lightweight store (Redis hash, or a `read_markers` table keyed by `(conversationId, userId) → lastReadMessageId`). Updating on every opened message is too chatty — send a marker for the latest ID on screen.

## Message store

The message history is the big storage driver. Requirements:

- Append-mostly.
- Reads are "give me the last N messages in conversation X" or "messages older than this one".
- Billions of rows per large conversation over years.
- Per-conversation ordered; cross-conversation queries are rare.

### Schema (Cassandra / HBase / Scylla style)

```text
Partition key:  conversationId
Clustering key: messageId (time-sortable — ULID or Snowflake)
Columns: senderId, body, mediaRef, createdAt, meta
```

This maps perfectly to a wide-column DB: one partition = one conversation, clustered by time → "give me recent 50 messages" is a trivial range scan. Cassandra, HBase, ScyllaDB, and DynamoDB all fit.

### Why not a relational DB?

At ~3 M writes/sec peak and multi-PB storage, RDBMS sharding becomes the dominant operational cost. A wide-column store is natively partitioned by conversationId with no per-shard operator burden. RDBMS works fine up to a few hundred thousand writes/sec with sharding; past that, switch.

## Presence

"Online / away / offline" — billions of users, updates many times a minute, read constantly.

- Backed by a distributed KV, typically Redis with TTLs.
- Each client heartbeats every 30-60 s; Redis stores `presence:userId → expires_at` with TTL.
- On WebSocket disconnect, the gateway immediately writes `offline`.
- Other users subscribe to presence changes via pub/sub (many chat apps don't — they query on demand to keep it cheap).

> Pushing every presence change to every friend is a fan-out firestorm. Most production chat systems update presence **lazily** — show "online" when you open a chat, not proactively on every tick.

## Group chat

For a 250-member group, fan-out on write is easy (250 message deliveries per send).

For a 100 k-member broadcast group, at peak rates that's millions of deliveries per message — you push the work to dedicated fan-out workers with batching and back-pressure, and accept that for very large groups, delivery is **best-effort** with a delay budget.

## Multi-device sync

One user on phone + laptop + tablet. Each device is its own WebSocket. Strategy:

- Each device is identified by `(userId, deviceId)`.
- A message sent from the phone must echo to the laptop as a "my own message" — typically via the same fan-out that delivers to the recipient, but routed back to the sender's other devices too.
- Read markers sync across devices: when I read on my phone, my laptop should clear the unread badge → send `read_up_to` to all my devices.

## End-to-end encryption (if mentioned)

Serious chat products encrypt client-to-client (Signal Protocol / double ratchet). The server stores only ciphertext. That changes:

- Server can't do rich text search or server-side translation.
- Backup/restore has to be user-key-wrapped.
- Read receipts / typing are metadata — some of that still leaks.

In the interview, if E2EE is in scope, flag the protocol (Signal Protocol is the de-facto choice) and acknowledge the constraint on server-side features.

## Failure modes

| Failure | Impact | Mitigation |
|---|---|---|
| Gateway node crash | Connected users disconnect | Clients auto-reconnect with back-off + jitter; routing layer re-pins to a new node |
| Message store write latency spike | Acks delayed; senders see "sending..." | Acknowledge once durable; if slow, queue and surface per-device retry |
| Kafka consumer lag | Delivery delay to offline-then-online users | Scale consumers; alert on lag |
| Push service (APNs/FCM) down | Offline users miss notifications | Retry with exponential back-off; message still stored and delivered on next app open |
| Duplicate delivery | User sees message twice | Client dedup on `messageId` |
| Network partition of a region | Users in that region can still chat locally but can't reach others | Multi-region replication of the message store; eventual consistency across regions |

## Things to NOT over-complicate

- Don't design global ordering — per-conversation is enough and much cheaper.
- Don't E2E encrypt "by default" in the first pass unless it's in scope; it interacts with everything else (search, backup, metadata).
- Don't broadcast presence eagerly to every friend; fetch on demand.
- Don't put the message store behind a strict ACID RDBMS at planet-scale. Wide-column is the canonical choice for this workload.

---

[← Previous: Case Study: News Feed](08-case-study-news-feed.md) | [Back to index](README.md)
