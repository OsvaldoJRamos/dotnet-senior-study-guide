# Idempotency and Race Conditions

## The scenario

> A client calls `POST /payments` **four times in a row**. The user double-clicked, the mobile network retried on timeout, the load balancer replayed a 5xx, and the webhook got redelivered. The account balance must be debited **exactly once**.

This file walks through the layered defense that makes that guarantee real.

## 1. Why duplicates happen

Duplicates are not edge cases — they are the default in any networked system:

| Source | Example |
|---|---|
| User behavior | Double-click the "Pay" button, browser back + resubmit |
| Network timeout | Client times out, retries — but the original request **did commit** on the server (the response packet was lost, not the work) |
| Mobile / flaky networks | TCP retransmits, app-level retries on connection reset |
| Load balancer / gateway | Retries on 502/504 when the upstream actually succeeded |
| Message redelivery | At-least-once is the default in Kafka, RabbitMQ, SQS, Service Bus. Consumers see the same message twice after a crash or nack |

> End-to-end **exactly-once delivery is impossible** in a distributed system (a consequence of the Two Generals Problem). The practical substitute is **at-least-once delivery + idempotent processing = effectively-once**.

## 2. Layered defense — no single layer is enough

Each layer catches a different class of race. You need all of them on a money-moving endpoint.

| Layer | Mechanism | Catches |
|---|---|---|
| HTTP boundary | `Idempotency-Key` header (Stripe / IETF draft) | Client-side retries sending the same key |
| Application | Dedup cache (e.g., Redis with TTL) | Concurrent duplicate requests in-flight |
| Database | `UNIQUE` constraint on `idempotency_key` | Race that slipped past the cache (two workers, near-simultaneous) |
| Business | Optimistic concurrency (`row_version` / `xmin`) | Two **different** legitimate debits racing on the same account |
| Orchestration | Saga + Outbox | Partial failure across services (cross-link: [SAGA Pattern](06-saga-pattern.md)) |

> The DB unique constraint is the **source of truth**. The Redis cache is an optimization. If you had to pick one, pick the constraint.

## 3. The `Idempotency-Key` header (Stripe / IETF)

Both Stripe and the IETF `httpapi-idempotency-key-header` draft converge on the same contract:

- Header name: **`Idempotency-Key`**.
- Value: **UUID v4** (or any random string with enough entropy; Stripe allows up to 255 chars).
- Scope: **per operation**, not per session or per user. A key is meaningful only for one specific request.
- TTL: Stripe retains results for **at least 24 hours**; the IETF draft says the server SHOULD publish its expiration policy.
- Replay: subsequent requests with the same key return **the same response body and status code** as the first — including 5xx errors (Stripe).
- Different body, same key: the IETF draft says the server SHOULD return **HTTP 422 Unprocessable Entity**; Stripe "compares incoming parameters to those of the original request and errors if they're not the same."
- Concurrent requests with the same key (first still in flight): the IETF draft recommends **HTTP 409 Conflict**.

> Only `POST` / `PATCH` need this. `GET` and `DELETE` are already idempotent by HTTP semantics; Stripe explicitly says sending the header on `GET`/`DELETE` "has no effect."

## 4. Full walkthrough — payment flow

### Schema

```sql
CREATE TABLE payment_attempts (
    idempotency_key  UUID        NOT NULL,
    customer_id      BIGINT      NOT NULL,
    request_hash     CHAR(64)    NOT NULL,       -- SHA-256 of the request body
    response_status  INT         NOT NULL,
    response_body    JSONB       NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT pk_payment_attempts PRIMARY KEY (idempotency_key)
);

-- The UNIQUE/PK on idempotency_key is the real guarantee.

CREATE TABLE accounts (
    id           BIGINT       PRIMARY KEY,
    balance      NUMERIC(18,2) NOT NULL,
    row_version  BIGINT       NOT NULL DEFAULT 0   -- optimistic concurrency token
);
```

### Handler (ASP.NET Core, simplified)

```csharp
public sealed class ChargePaymentHandler
{
    private readonly IDbContextFactory<PaymentDbContext> _dbFactory;
    private readonly IDistributedCache _cache;

    public async Task<PaymentResponse> HandleAsync(
        ChargeRequest request,
        string idempotencyKey,
        CancellationToken ct)
    {
        // 1. Fast path: response already cached (Redis, 24h TTL)
        var cached = await _cache.GetStringAsync($"idemp:{idempotencyKey}", ct);
        if (cached is not null)
            return JsonSerializer.Deserialize<PaymentResponse>(cached)!;

        var requestHash = Sha256(request);

        await using var db = await _dbFactory.CreateDbContextAsync(ct);
        await using var tx = await db.Database.BeginTransactionAsync(ct);

        try
        {
            // 2. Insert the attempt. UNIQUE constraint is the real guard.
            db.PaymentAttempts.Add(new PaymentAttempt
            {
                IdempotencyKey = Guid.Parse(idempotencyKey),
                CustomerId     = request.CustomerId,
                RequestHash    = requestHash,
                ResponseStatus = 200,
                ResponseBody   = "{}"                  // placeholder, updated below
            });
            await db.SaveChangesAsync(ct);
        }
        catch (DbUpdateException ex) when (IsUniqueViolation(ex))
        {
            // 3. Another request with the same key won the race. Replay its response.
            await tx.RollbackAsync(ct);
            var existing = await db.PaymentAttempts
                .AsNoTracking()
                .SingleAsync(x => x.IdempotencyKey == Guid.Parse(idempotencyKey), ct);

            // Reject same-key-different-body per IETF draft (HTTP 422)
            if (existing.RequestHash != requestHash)
                throw new IdempotencyConflictException();   // -> maps to 422

            return JsonSerializer.Deserialize<PaymentResponse>(existing.ResponseBody)!;
        }

        // 4. Debit with optimistic concurrency on the account row.
        var account = await db.Accounts.SingleAsync(a => a.Id == request.AccountId, ct);
        var rows = await db.Database.ExecuteSqlInterpolatedAsync($@"
            UPDATE accounts
               SET balance     = balance - {request.Amount},
                   row_version = row_version + 1
             WHERE id          = {account.Id}
               AND row_version = {account.RowVersion}
               AND balance    >= {request.Amount}", ct);

        if (rows == 0)
            throw new ConcurrencyOrInsufficientFundsException();

        // 5. Persist the final response on the same attempt row.
        var response = new PaymentResponse(Guid.NewGuid(), "captured", request.Amount);
        await db.Database.ExecuteSqlInterpolatedAsync($@"
            UPDATE payment_attempts
               SET response_status = 200,
                   response_body   = {JsonSerializer.Serialize(response)}::jsonb
             WHERE idempotency_key = {Guid.Parse(idempotencyKey)}", ct);

        await tx.CommitAsync(ct);

        // 6. Cache for fast replay (24h TTL matches Stripe's retention).
        await _cache.SetStringAsync(
            $"idemp:{idempotencyKey}",
            JsonSerializer.Serialize(response),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) },
            ct);

        return response;
    }

    private static bool IsUniqueViolation(DbUpdateException ex) =>
        ex.InnerException is Npgsql.PostgresException { SqlState: "23505" };
}
```

### Why the 4 duplicate calls land as 1 debit

| Call | Reaches cache | Reaches DB insert | Outcome |
|---|---|---|---|
| 1 | Miss | Succeeds | Debits account, caches response |
| 2 (in-flight overlap) | Miss | Unique violation | Replays response from DB |
| 3 | Hit | — | Replays from cache |
| 4 | Hit | — | Replays from cache |

Account is debited exactly once. All four callers see identical `200 OK` with the same `PaymentResponse`.

## 5. Pitfalls

- **Scope**: the key is **per operation**, not per user session. Reusing a key for "charge $10" and later "charge $20" is a bug, not a feature.
- **TTL too short**: if you drop the key before the client's retry window closes, you lose the replay guarantee. Stripe uses 24h as a practical floor.
- **Crash AFTER commit, BEFORE response written**: client retries with the same key, sees the cached/DB-stored response. Safe.
- **Crash BEFORE commit**: nothing persisted, client retries, goes down the fresh path. Safe.
- **Header without DB constraint**: two workers processing the same key concurrently can both miss the cache and both write. The `UNIQUE` constraint is non-negotiable — the cache alone is a race.
- **Replay must be byte-identical**: return the same status code and response body, not a "you already did this" message. Clients rely on this to recover seamlessly.
- **Same key + different body**: return `422 Unprocessable Entity` (IETF draft) — do NOT silently process it.
- **Concurrent same-key**: while the first is still in flight, a second attempt should get `409 Conflict` (IETF draft).

## 6. When idempotency keys are not enough: distributed locks

Idempotency handles **duplicates of the same logical request**. It does NOT handle **two different legitimate requests** that must be serialized (e.g., two admins editing the same config, two workers claiming the same job).

### SQL Server — `sp_getapplock`

```sql
BEGIN TRANSACTION;

DECLARE @result INT;
EXEC @result = sp_getapplock
    @Resource  = 'account:12345',
    @LockMode  = 'Exclusive',
    @LockOwner = 'Transaction',
    @LockTimeout = 5000;              -- milliseconds; -1 = wait forever, 0 = fail fast

IF @result < 0
BEGIN
    ROLLBACK TRANSACTION;
    THROW 51000, 'Could not acquire app lock', 1;
END

-- ... work ...

COMMIT TRANSACTION;   -- Transaction-owned lock releases automatically
```

Return codes per Microsoft Learn: `0` granted, `1` granted after wait, `-1` timeout, `-2` canceled, `-3` deadlock victim, `-999` parameter error. With `@LockOwner = 'Transaction'` (the default), `sp_getapplock` **must** be executed inside a transaction.

### PostgreSQL — advisory locks

```sql
BEGIN;
SELECT pg_advisory_xact_lock(hashtext('account:12345'));
-- ... work ...
COMMIT;   -- transaction-scoped lock releases on commit/rollback
```

Use `pg_advisory_xact_lock` (transaction-scoped, auto-released on `COMMIT`/`ROLLBACK`) over `pg_advisory_lock` (session-scoped, must be explicitly released) unless you need cross-transaction holding. The PostgreSQL docs warn: advisory locks "do not honor transaction semantics" in the session variant — a rollback does NOT release them.

### Redis — single-instance

```
SET lock:account:12345 <unique-token> NX PX 30000
```

- `NX` — only set if absent.
- `PX 30000` — expire after 30s (TTL is mandatory: if the holder crashes, the lock frees itself).
- `<unique-token>` — random per-caller; release only deletes if the token still matches (Lua CAS):

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

### Redlock and the correctness debate

Redlock is Redis's multi-instance lock algorithm (acquire on a majority of N independent nodes). Martin Kleppmann's 2016 critique argues Redlock is unsafe for correctness-critical work because:

1. It relies on **timing assumptions** (bounded clock drift, bounded execution time) that don't hold under GC pauses or network partitions.
2. It has **no fencing tokens** — a monotonically increasing number that the downstream storage can reject when a stale lock-holder tries to write after its lease expired.

Kleppmann's recommendation:
- For **efficiency locks** (avoid duplicate work, duplicates are annoying but not fatal): a single Redis instance is fine.
- For **correctness locks** (duplicates cause data corruption): use a proper consensus system (ZooKeeper, etcd) **and** enforce fencing tokens on every write.

Antirez (Redis's creator) published a rebuttal defending Redlock under stated assumptions. The practical takeaway: **don't reach for Redlock for money.** Use the DB unique constraint + optimistic concurrency from Section 3.

> **Rule of thumb**: locks don't prevent duplicates of the *same* request — that's what idempotency keys are for. Locks serialize *different* requests competing for the same resource.

## 7. Cross-service scenarios

When a payment spans multiple services (charge + inventory + shipping), a single DB transaction is no longer available:

- Use the [Saga Pattern](06-saga-pattern.md) to sequence steps with compensating actions.
- Use the **Transactional Outbox** to atomically persist "I owe a message to Kafka" together with the DB change — then a relay publishes it with at-least-once semantics.
- Every consumer must be **idempotent** (dedup on message ID, or upsert on natural key). See [Messaging](10-messaging.md) for the "effectively-once" pattern.

## 8. Testing idempotency

```csharp
[Fact]
public async Task Same_key_twice_debits_once()
{
    var key = Guid.NewGuid().ToString();
    var req = new ChargeRequest(CustomerId: 1, AccountId: 42, Amount: 100m);

    var first  = await _handler.HandleAsync(req, key, default);
    var second = await _handler.HandleAsync(req, key, default);

    first.Should().BeEquivalentTo(second);                 // byte-identical replay
    (await CountAttempts(key)).Should().Be(1);             // one DB row
    (await GetBalance(42)).Should().Be(StartingBalance - 100m);  // debited once
}
```

Additional test layers:
- **Concurrent test**: fire N parallel calls with the same key using `Task.WhenAll`; assert one row, one debit.
- **Chaos test**: inject latency / kill the process mid-commit; replay with the same key; assert consistency.
- **Load test**: k6 or NBomber sending duplicate keys at high concurrency to catch race conditions that only appear under contention.

## 9. Rule of thumb

> If the operation has side effects and can be retried by any party (client, LB, broker, network), it **MUST** accept an `Idempotency-Key` AND be backed by a DB `UNIQUE` constraint. The cache is optional; the constraint is not.

---

[← Previous: Messaging](10-messaging.md) | [Back to index](README.md)
