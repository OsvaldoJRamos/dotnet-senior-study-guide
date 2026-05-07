# Consistency Models

CAP says you trade C for A during a partition. But "C" is not binary — it's a spectrum of **consistency models**, each with a precise definition and a different cost. Knowing them by name and their guarantees is table stakes at senior level.

## The spectrum (strongest to weakest)

| Model | Informal meaning | Cost |
|---|---|---|
| **Strict / linearizable** | Every read sees the latest write globally, as if there were one copy. | Highest latency; cross-region coordination |
| **Sequential** | All processes see all operations in *some* total order; not necessarily real-time. | Still global coordination, but no wall-clock sync |
| **Causal** | Operations that are causally related are seen in order by everyone; concurrent ops may differ. | Track causal dependencies (vector clocks) |
| **Session guarantees** | Within one client session: read-your-writes, monotonic reads/writes, writes-follow-reads. | Session affinity, sticky routing |
| **Eventual** | In the absence of new writes, replicas eventually converge. | Cheapest; readers see stale and out-of-order data |

## Strict / linearizable consistency

Formally (Herlihy & Wing, 1990): operations appear to take effect atomically at some point between their start and end, in an order consistent with real time.

Practically: a write completes; any subsequent read **anywhere** returns that write or a newer one. Feels like one machine.

**Use when**: money, inventory, leader election, distributed locks. Anything where "stale" equals "wrong."

**.NET / Azure example**: Cosmos DB's `Strong` consistency level. Writes cross a cross-region majority quorum before acknowledging, and reads are guaranteed linearizable.

```csharp
var clientOptions = new CosmosClientOptions
{
    ConsistencyLevel = ConsistencyLevel.Strong
};
```

**Cost**: cross-region latency on every write. In a multi-region deployment, this can be 100+ ms per write.

## Sequential consistency

Lamport (1979): *"the result of any execution is the same as if the (read and write) operations of all processes on the data store were executed in some sequential order, and the operations of each individual process appear in this sequence in the order specified by its program."*

All processes agree on a single order. Weaker than linearizable because it doesn't require that order to match real time — so a write that returned at 10:00:01 might be placed after a write that returned at 10:00:02 in the agreed order, as long as everyone agrees.

Useful in theory, rarely advertised as a product guarantee — most systems are either linearizable or weaker than sequential.

## Causal consistency

Operations that are **causally related** (the "happened-before" relation, see [Clocks and Ordering](08-clocks-and-ordering.md)) must be seen in the same order by every replica. Concurrent operations may be seen in different orders on different replicas.

Classic example: if Alice posts "I lost my dog" and later "Found him, false alarm", a replica must not show the second post without the first. But two unrelated posts (different users, different conversations) can be ordered differently on different replicas — nobody cares.

**.NET / Azure example**: Cosmos DB's `ConsistentPrefix` level — readers never see writes out of order, but they may see stale data. Cheaper than `Strong` or `BoundedStaleness`.

## Session guarantees (Terry et al., Bayou, 1994)

Four guarantees scoped to a single client session. Individually weaker than global consistency, but the **combination** gives a surprisingly usable experience at eventual-consistency cost.

| Guarantee | What it means |
|---|---|
| **Read-your-writes (RYW)** | After you write X, your own reads see X (or newer). |
| **Monotonic reads** | If you read a value, you never later read an older value. |
| **Monotonic writes** | Your writes are applied in the order you issued them. |
| **Writes-follow-reads** | If you read X then write Y, any replica applying Y has first applied X. |

**.NET / Azure example**: Cosmos DB `Session` level (the default) gives exactly these four guarantees for the session. Requires the SDK's session token to be passed across calls:

```csharp
// Pin the session token from a write so subsequent reads see it
var writeResponse = await container.CreateItemAsync(item);
string sessionToken = writeResponse.Headers.Session;

var requestOptions = new ItemRequestOptions { SessionToken = sessionToken };
var readResponse = await container.ReadItemAsync<MyItem>(
    id: item.Id,
    partitionKey: new PartitionKey(item.PartitionKey),
    requestOptions: requestOptions);
```

If reads go through a different client instance (common in scale-out web apps), forward the session token in a cookie or header — otherwise RYW is not guaranteed.

## Eventual consistency

Weakest model with any guarantee: *"if no update takes a very long time, all replicas eventually become consistent."* No ordering guarantees, no staleness bounds, no session semantics.

**Use when**: analytics, counters you can recompute, content caches, search indexes. Never for money.

**.NET / Azure example**: Cosmos DB `Eventual` — lowest latency, highest throughput, weakest guarantees. Good fit for a product catalog replicated to five regions where eventual convergence within seconds is acceptable.

## Bounded staleness — a practical middle ground

Cosmos DB adds **Bounded Staleness**: reads are guaranteed to lag writes by at most *K* versions or *T* time. Outside the bound, it's linearizable; inside the bound, it's consistent-prefix.

```csharp
new CosmosClientOptions
{
    ConsistencyLevel = ConsistencyLevel.BoundedStaleness
    // Configure K (versions) and T (time) at account level in the Azure portal
};
```

This is the honest "I want strong, but only pay for it when I'm close to real time" choice.

## Decision flow

```text
Does stale data corrupt business state?
├── Yes (money, inventory, leases, locks) ──▶ Strong / linearizable
└── No
    ├── Do I need within-session RYW + monotonic reads?
    │   ├── Yes ──▶ Session (cheapest that still feels right to users)
    │   └── No
    │       ├── Causal dependencies between writes? ──▶ Causal / ConsistentPrefix
    │       └── No ──▶ Eventual (cheapest, highest throughput)
```

## Pitfalls seniors get asked about

- **Confusing "strong" with "ACID"**: ACID is about single-DB transactional integrity. "Strong consistency" is about replication. A PostgreSQL single instance is fully ACID but there's no replication topology to reason about. Once you add a read replica, consistency becomes a replication choice.
- **Treating replica reads as free**: reading from an async replica gives eventual semantics even if the primary is fully ACID. You can see committed-but-not-yet-replicated data vanish on failover.
- **Assuming "linearizable" means "fresh"**: linearizable means ordered-with-real-time, but a linearizable read can still return "the value as of 5 ms ago" — that's perfectly fine, because no intervening write completed in those 5 ms.
- **Ignoring clock skew**: Spanner-style external consistency is bounded by clock uncertainty (TrueTime). Without atomic clocks, "strong + low latency across regions" is fundamentally hard — see [Clocks and Ordering](08-clocks-and-ordering.md).

> **Interview signal**: name the model, its formal definition, what it costs, and a concrete example from your stack. "We used session consistency on Cosmos for the shopping cart because the same user's replica pin gives us RYW without the cross-region hop of strong."

## References

- Herlihy, Wing — *Linearizability: A Correctness Condition for Concurrent Objects* (1990)
- Lamport — *How to Make a Multiprocessor Computer that Correctly Executes Multiprocess Programs* (1979, introduces sequential consistency)
- Terry et al. — *Session Guarantees for Weakly Consistent Replicated Data* (1994)
- Azure Cosmos DB consistency levels: https://learn.microsoft.com/azure/cosmos-db/consistency-levels

---

[← Previous: CAP Theorem](01-cap-theorem.md) | [Next: Idempotency →](03-idempotency.md) | [Back to index](README.md)
