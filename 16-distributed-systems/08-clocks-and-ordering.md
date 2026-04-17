# Clocks and Ordering

## Why clocks are hard

In a single process, "before" and "after" are obvious. Across machines, they are not. Two questions drive every distributed ordering decision:

1. **Did event A cause event B, or are they concurrent?**
2. **Can I give every event a globally meaningful timestamp?**

The answer to (1) is "sometimes, if we track dependencies." The answer to (2) is "not without special hardware or coordination."

## Physical clocks and their limits

Every machine has a wall-clock (`DateTime.UtcNow`, `Stopwatch.GetTimestamp()`, `System.currentTimeMillis`). They disagree with each other, and they drift:

- **Clock skew**: two machines can disagree by tens of ms even with NTP; across regions without tight sync, by seconds.
- **Non-monotonicity**: wall-clock time can jump backwards (NTP corrections, leap seconds, admin adjustments).
- **Pauses**: a GC pause or VM live-migration freezes the clock for hundreds of ms relative to others.

Consequences:

- **Do not use wall-clock time to order events across machines** unless you're OK with "close enough."
- **Do not use wall-clock time to decide expiry correctness.** A token that expires "at 12:00:00" can be accepted at 11:59:59 by a client and rejected at 11:59:58 by a server whose clock is fast.
- **For measuring elapsed time within one process**: use a **monotonic** clock (`Stopwatch` in .NET, `CLOCK_MONOTONIC` on Linux). Never subtract two `DateTime.UtcNow` readings if the measurement matters.

### .NET specifics

```csharp
// WRONG — wall clock, can go backwards
var start = DateTime.UtcNow;
DoWork();
var elapsed = DateTime.UtcNow - start;

// CORRECT — monotonic
var sw = Stopwatch.StartNew();
DoWork();
var elapsed = sw.Elapsed;

// For timestamps (ticks) across requests, Stopwatch.GetTimestamp() is monotonic
// on the same process but not synchronised across processes.
```

`DateTimeOffset.UtcNow` is backed by the OS wall clock and shares its weaknesses. `TimeProvider` (introduced in .NET 8) is a testable abstraction over both.

## The happened-before relation (Lamport, 1978)

Leslie Lamport's "Time, Clocks, and the Ordering of Events in a Distributed System" (1978) defines a **partial order** over events called **happened-before** (`→`):

1. If *a* and *b* are events in the same process and *a* comes before *b*, then *a* → *b*.
2. If *a* is the sending of a message and *b* is the receipt of that message, then *a* → *b*.
3. Transitive: if *a* → *b* and *b* → *c*, then *a* → *c*.

Two events *a* and *b* where neither *a* → *b* nor *b* → *a* are **concurrent** (written *a* ∥ *b*). In a distributed system, concurrency is common — most pairs of events across unrelated processes are concurrent.

## Lamport timestamps

A **scalar** logical clock that respects happened-before.

Per the original paper, each process maintains an integer counter `C` and follows these rules:

1. A process **increments its counter** before each local event.
2. When sending a message, **include the counter value** with it.
3. On receiving a message with timestamp *t*, set `C := max(C, t) + 1` before processing it.

**Guarantee (clock consistency condition):** if *a* → *b*, then `C(a) < C(b)`.

**What it does NOT give you:** `C(a) < C(b)` does NOT imply *a* → *b*. Two concurrent events can have different Lamport values. So Lamport timestamps detect "certainly after" but not "certainly before."

```csharp
public sealed class LamportClock
{
    private long _counter;

    public long Tick() => Interlocked.Increment(ref _counter);

    public long ObserveAndTick(long received)
    {
        long next;
        long current;
        do
        {
            current = _counter;
            next = Math.Max(current, received) + 1;
        } while (Interlocked.CompareExchange(ref _counter, next, current) != current);
        return next;
    }
}
```

### Uses

- **Total ordering of events for tie-breaking** (combine Lamport value with a node ID).
- **Distributed mutual exclusion** (the original use case in Lamport's paper).
- Building block for more advanced schemes.

### Limitations

- Cannot distinguish "happened-before" from "concurrent." Two events with Lamport values 5 and 7 may be causally ordered or completely unrelated — you can't tell.

## Vector clocks (Fidge 1988, Mattern 1988)

Independently developed by Colin Fidge and Friedemann Mattern. Each process maintains a **vector** of counters, one entry per process in the system.

### Rules

For process *i* with vector `V[i]`:

1. On a local event: `V[i][i] += 1`.
2. On send: attach a copy of `V[i]` to the message.
3. On receive of message with vector *W*: set `V[i][j] := max(V[i][j], W[j])` for every *j*, then `V[i][i] += 1`.

### Comparison

Given vectors *V* and *W*:

- `V < W` if `V[k] ≤ W[k]` for every *k* and `V[k] < W[k]` for at least one *k*.
- `V = W` if they're equal component-wise.
- Otherwise (`V` has some component greater and `W` has some component greater), the events are **concurrent**.

**Strong guarantee:** `V(a) < V(b)` **if and only if** *a* → *b*. Unlike Lamport, vector clocks detect concurrency exactly.

### Cost

Vector size = number of processes. In a 1000-node cluster each message carries a 1000-entry vector. Pruning and compression schemes exist (interval tree clocks, dotted version vectors), but vector clocks rarely scale past small replica sets.

### Uses

- **Causal consistency**: Riak used dotted version vectors for sibling reconciliation; DynamoDB's predecessor Dynamo used vector clocks.
- **Conflict detection in collaborative editing** (CRDTs use vector-clock-like structures).
- **Distributed snapshot and debugging**.

## Hybrid Logical Clocks (HLC)

Kulkarni, Demirbas, Madappa, Avva, Leone (2014) introduced HLCs as a hybrid of physical and logical time. The idea:

- Keep a timestamp close to physical (wall-clock) time so it's **human-meaningful** and **comparable across disjoint nodes**.
- Enforce a **monotonic logical** component so it respects happened-before even when physical clocks disagree.

Each node tracks `(L, C)` where `L` is the logical component (a physical-time-like value) and `C` is a counter. On a local event or receive, `L` is set to the max of the local physical time, the current `L`, and any received `L`, with `C` bumped to preserve order within the same `L`.

### Why HLC matters

- Fits in 64 or 96 bits — practical to store per row or per event.
- Causal-ordering guaranteed like Lamport.
- Values remain close to wall-clock — useful for range queries, TTLs, and debugging.

Verify against the original paper and the docs of any system you're using before citing specific bit layouts — HLC is implemented slightly differently by each adopter.

## Spanner-style external consistency

Google Spanner uses **TrueTime**: GPS receivers and atomic clocks in every data centre deliver `now()` as an interval `[earliest, latest]` with a bounded uncertainty (typically a few ms). Spanner waits out the uncertainty at commit time so any committed transaction's timestamp is guaranteed to be in the past by the time clients see it — this is **external consistency**, a property stronger than linearizability.

Without atomic clocks you can't replicate TrueTime cheaply. CockroachDB's official architecture docs describe using HLCs explicitly: *"CockroachDB implements hybrid-logical clocks (HLC) which are composed of a physical component (always close to local wall time) and a logical component (used to distinguish between events with the same physical component)."* It combines HLC with a clock-uncertainty window and transaction-restart protocol to approximate strong semantics. Other systems take different tacks (YugabyteDB uses hybrid timestamps; Cosmos DB's replication strategy is internal). Validate specific behaviour against the vendor docs before quoting it — these systems tune their strategy across releases.

## Practical guidance

| Need | Use |
|---|---|
| Measure elapsed time within a process | Monotonic clock (`Stopwatch` / `CLOCK_MONOTONIC`) |
| Human-readable timestamp for logs and audit | Wall clock (`DateTime.UtcNow`), accept drift |
| Order events within one service | Wall clock is usually fine |
| Order events across services where causality matters | Logical clock in message metadata (Lamport or HLC) |
| Detect "did A cause B or are they concurrent?" | Vector clock (or dotted version vector) |
| Strong consistency with low latency globally | Spanner-style (requires hardware) or accept the [PACELC latency cost](01-cap-theorem.md) |

## Pitfalls

- **Using `DateTime.UtcNow` for "happened before"**: it almost works, until a leap second or an NTP correction flips two events' order.
- **Storing wall-clock times as `DateTime.Now` (local time)**: ambiguous across DST transitions, impossible to compare across zones. Always `DateTime.UtcNow` (or better: `DateTimeOffset`).
- **Treating Lamport values as "how many events happened"**: they're not event counts; they're upper bounds on causal depth.
- **Assuming NTP fixes everything**: NTP provides *best-effort* sync to some reference, not bounded skew. It can still step the clock backwards and it quietly fails when a server falls off the reference pool.
- **Emitting events with "now" and ordering downstream by that field**: two services emitting at the same instant can have events reordered arbitrarily. If order matters, pass a logical clock or serialise through a single partition.

## References

- Lamport — *Time, Clocks, and the Ordering of Events in a Distributed System* (CACM, 1978)
- Fidge — *Timestamps in Message-Passing Systems That Preserve the Partial Ordering* (1988)
- Mattern — *Virtual Time and Global States of Distributed Systems* (1988)
- Kulkarni et al. — *Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases* (2014) — Hybrid Logical Clocks
- Corbett et al. — *Spanner: Google's Globally-Distributed Database* (OSDI 2012)

---

[← Previous: Consensus — Raft and Paxos](07-consensus-raft-paxos.md) | [Back to index](README.md)
