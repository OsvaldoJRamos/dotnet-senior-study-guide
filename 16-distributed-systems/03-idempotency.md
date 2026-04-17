# Idempotency

> An operation is **idempotent** if applying it multiple times produces the same observable result as applying it once.

In a distributed system, idempotency is not a nice-to-have — it's the only way to get correctness out of an unreliable network.

> Section 06 has a deeper operational walkthrough — see [Idempotency and Race Conditions](../06-architecture-and-patterns/11-idempotency-and-race-conditions.md). This file focuses on *why* distributed systems force you into idempotency and how to reason about it.

## Why it's mandatory

The **Two Generals Problem** proves that on an asynchronous network with possible message loss, there is no protocol that can guarantee both parties know a message was delivered. Practical consequence: if a client sends a request and times out, it **cannot know** whether the server received it, processed it, or sent a response that was lost on the way back.

The only safe behaviour for the client is to **retry**. And the only safe behaviour for the server is to make retries **harmless** — that is, idempotent.

### Delivery semantics

| Semantic | What it means | Who provides it |
|---|---|---|
| **At-most-once** | Message delivered zero or one time. Losses possible. | Fire-and-forget UDP, `auto.offset.reset=latest` with no commit |
| **At-least-once** | Message delivered one or more times. No loss, but duplicates possible. | Default for Kafka, RabbitMQ, SQS, Azure Service Bus |
| **Exactly-once** | Message delivered exactly once. | Only achievable end-to-end by combining at-least-once delivery with **idempotent processing**. |

> End-to-end exactly-once *delivery* is impossible across a network (Two Generals). "Exactly-once" in real systems means **at-least-once delivery + idempotent consumer = effectively-once processing**.

Kafka's "exactly-once semantics" (EOS) is scoped: per Confluent's own statement, it applies to the read-process-write pattern within Kafka Streams. "Exactly-once semantics is guaranteed within the scope of Kafka Streams' internal processing only; for example, if the event streaming app...makes an RPC call to update some remote stores...the resulting side effects would not be guaranteed exactly once." (Confluent, "Exactly-Once Semantics Are Possible"). For anything leaving Kafka, you still need idempotent consumers. See [Distributed Transactions](06-distributed-transactions.md).

## HTTP methods and idempotency (RFC 9110)

RFC 9110 classifies methods by their semantics. **Idempotent methods** can be retried safely:

| Method | Safe | Idempotent | Notes |
|---|---|---|---|
| `GET`, `HEAD` | Yes | Yes | Read-only. |
| `PUT` | No | Yes | "Replace resource with this" — same body, same end state. |
| `DELETE` | No | Yes | Second delete returns 404 or 204; state is still "deleted." |
| `OPTIONS`, `TRACE` | Yes | Yes | Metadata. |
| `POST` | No | **No** | "Create new resource"; retrying creates two. |
| `PATCH` | No | **No** | Partial update; retry semantics depend on the patch. |

So `POST` and `PATCH` need **application-level** idempotency — that's what the `Idempotency-Key` header is for.

## The `Idempotency-Key` header (IETF draft)

The IETF `draft-ietf-httpapi-idempotency-key-header` formalises what Stripe popularised:

- **Header**: `Idempotency-Key: <string>` — typically a UUIDv4 chosen by the client.
- **Scope**: per operation, per resource. Do NOT reuse a key across different requests.
- **Methods**: only `POST` and `PATCH` need it; the draft explicitly says `GET`/`PUT`/`DELETE`/`HEAD`/`OPTIONS` are already idempotent per RFC 9110.
- **Replay behaviour**: a second request with the same key returns the original response verbatim (same status code, same body).
- **Same key, different body**: per the draft, the server SHOULD return **HTTP 422 Unprocessable Content**.
- **Concurrent same key** (first still in flight): per the draft, the server SHOULD return **HTTP 409 Conflict**.
- **Missing key on an endpoint that requires it**: per the draft, the server SHOULD return **HTTP 400 Bad Request**.

## Implementation pattern (ASP.NET Core, conceptual)

```csharp
public sealed class IdempotencyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IIdempotencyStore _store;

    public IdempotencyMiddleware(RequestDelegate next, IIdempotencyStore store)
    {
        _next = next;
        _store = store;
    }

    public async Task InvokeAsync(HttpContext ctx)
    {
        if (ctx.Request.Method is not ("POST" or "PATCH"))
        {
            await _next(ctx);
            return;
        }

        if (!ctx.Request.Headers.TryGetValue("Idempotency-Key", out var key))
        {
            // Depending on your contract: 400 if required, or let the request through.
            await _next(ctx);
            return;
        }

        var bodyHash = await HashBodyAsync(ctx.Request);

        var existing = await _store.TryGetAsync(key!, ctx.RequestAborted);
        if (existing is not null)
        {
            if (existing.BodyHash != bodyHash)
            {
                ctx.Response.StatusCode = StatusCodes.Status422UnprocessableEntity;
                return;
            }

            if (existing.InFlight)
            {
                ctx.Response.StatusCode = StatusCodes.Status409Conflict;
                return;
            }

            // Replay the original response verbatim.
            ctx.Response.StatusCode = existing.StatusCode;
            await ctx.Response.Body.WriteAsync(existing.Body, ctx.RequestAborted);
            return;
        }

        await _store.MarkInFlightAsync(key!, bodyHash, ctx.RequestAborted);

        // Capture the response as it's produced, then persist it.
        var originalBody = ctx.Response.Body;
        using var buffer = new MemoryStream();
        ctx.Response.Body = buffer;

        await _next(ctx);

        buffer.Seek(0, SeekOrigin.Begin);
        await _store.CompleteAsync(key!, ctx.Response.StatusCode, buffer.ToArray(), ctx.RequestAborted);

        buffer.Seek(0, SeekOrigin.Begin);
        await buffer.CopyToAsync(originalBody);
    }

    private static Task<string> HashBodyAsync(HttpRequest req) => /* SHA-256 of the body */;
}
```

The store needs two things to be correct under concurrency:

1. A **unique constraint** on the idempotency key in the underlying database (this is the source of truth, not the cache).
2. A way to record "in flight" so a second concurrent request with the same key gets 409.

## Idempotent domain operations

Not every write is naturally idempotent. Design for it:

| Pattern | Idempotent? | How |
|---|---|---|
| `INSERT` by surrogate key | Yes if the key is client-chosen and unique-constrained | `INSERT ... ON CONFLICT DO NOTHING` / `MERGE` |
| `UPDATE x = x + 1` (counter) | **No** — every retry increments again | Switch to `UPDATE x = :absolute_value` or track per-request deltas |
| `UPDATE x = :new_value WHERE version = :expected_version` | Yes (optimistic concurrency) | Stale retries fail cleanly; the "real" one succeeds |
| `DELETE WHERE id = :id` | Yes | Second delete is a no-op |
| Publish an event | Yes if consumers dedupe on message ID (inbox pattern) | Record processed message IDs |

## Idempotency at the message layer

At-least-once brokers (Kafka, RabbitMQ, SQS, Azure Service Bus) will redeliver messages. Consumers must dedupe:

- **Inbox pattern**: the consumer records every processed message ID in a table with a unique constraint. Duplicate delivery → unique violation → skip.
- **Natural key upsert**: if the message carries a natural key (e.g., `OrderId`), upsert by that key instead of insert.
- **Idempotent handlers**: design the handler so re-executing with the same message produces the same state (see the patterns table above).

See [Outbox Pattern](04-outbox-pattern.md) for the producer-side guarantee that completes the picture.

## Pitfalls

- **Storing the key only in Redis**: the cache is a latency optimization; the database unique constraint is the real guarantee. Redis-only is a race waiting to happen.
- **TTL too short**: if keys expire before the client's retry window closes, replays turn into fresh requests. Stripe keeps idempotency results for at least 24 hours — a sensible floor for public APIs.
- **Reusing a key across operations**: a key is per-operation, per-resource. Reusing a key for "charge $10" then "charge $20" will either replay the wrong response or hit the 422 / 409 path.
- **Non-deterministic response bodies**: if your response includes `Guid.NewGuid()` or `DateTime.UtcNow`, replays will return the first value — which is correct behaviour, but the client's logs will look strange. Generate once, persist, replay.
- **Confusing idempotency with exactly-once**: idempotency makes duplicates harmless; it doesn't prevent them from happening. Exactly-once messaging is a design pattern, not a protocol guarantee.

## References

- IETF draft — *The Idempotency-Key HTTP Header Field*: https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header
- RFC 9110 (HTTP Semantics), §9.2.2 Idempotent Methods: https://www.rfc-editor.org/rfc/rfc9110#section-9.2.2
- Stripe API docs — *Idempotent Requests*: https://stripe.com/docs/api/idempotent_requests
- Confluent — *Exactly-Once Semantics Are Possible: Here's How Apache Kafka Does It*

---

[← Previous: Consistency Models](02-consistency-models.md) | [Next: Outbox Pattern →](04-outbox-pattern.md) | [Back to index](README.md)
