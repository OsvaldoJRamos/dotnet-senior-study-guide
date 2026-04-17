# CAP Theorem

## The formal statement

Brewer conjectured it (PODC 2000). **Gilbert and Lynch proved it** ("Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services", 2002). The formal definitions are:

| Property | Formal definition (Gilbert & Lynch) |
|---|---|
| **Consistency (C)** | Every read receives the most recent write or an error. Equivalent to **linearizability** — there exists a total order of operations consistent with real time. |
| **Availability (A)** | Every request received by a non-failing node must result in a (non-error) response. |
| **Partition tolerance (P)** | The system continues to operate despite an arbitrary number of messages being dropped (or delayed) between nodes. |

**Theorem:** in an asynchronous network, no distributed data store can simultaneously provide all three. At most two.

The proof is a short adversary argument: given two nodes that must both be available and a partition between them, a write to one cannot be observed by a read on the other, so either consistency breaks (read returns stale data) or availability breaks (one side refuses the read).

## The "pick 2 of 3" misconception

The popular framing is wrong. Partitions are **not a design choice** — they are a fact of networking. Any distributed system, eventually, will see a partition (a dropped packet, a GC pause, a network card failure, a cross-region link flap).

So the real choice is binary and only matters **during a partition**:

- **CP** — when the network splits, refuse requests (or return errors) on the minority side to preserve consistency. Example: etcd, ZooKeeper, HBase.
- **AP** — when the network splits, keep serving reads/writes on both sides and reconcile later. Example: Cassandra, DynamoDB (default), Riak.

During normal operation (no partition), both CP and AP systems can be fully available and consistent. The trade-off is invisible 99% of the time and decisive the other 1%.

> The question on a design review is never "is this system CP or AP?" but **"what does this system do during a partition, and is that the behaviour the business wants?"**

## PACELC (Abadi, 2012)

Abadi argued in "Consistency Tradeoffs in Modern Distributed Database System Design" that CAP ignores the cost paid in the normal case. His extension:

> **If partition (P), choose A or C. Else (E), choose L (latency) or C.**

Because even without a partition, maintaining strong consistency across replicas requires synchronous coordination (quorum reads, synchronous replication), which costs latency. The "E" side is what you pay every single request.

| Classification | Normal-case choice | Partition-case choice | Example |
|---|---|---|---|
| **PC/EC** | Consistency | Consistency | Traditional RDBMS with sync replication, HBase |
| **PA/EL** | Latency | Availability | Cassandra, Dynamo, Riak |
| **PC/EL** | Latency | Consistency | PNUTS |
| **PA/EC** | Consistency | Availability | MongoDB (default), Hazelcast IMDG |

PACELC is the more honest framing for modern systems — it surfaces the cost you pay **every day**, not only when the network fails.

## What CAP does NOT say

Common misreadings to push back on in an interview:

- **"CAP means you must give up consistency for availability."** No — it says you trade C against A *only during a partition*. During normal operation, a well-designed system is both.
- **"Picking CP means the system is never available."** No — CP systems are available during normal operation. They just refuse minority-side requests during a partition.
- **"CAP is about databases."** No — it's about any system that replicates state across a network. It applies equally to caches, distributed locks, and service meshes.
- **"You can't have both strong consistency and low latency."** True only at scale — at small replica counts with a fast, reliable network, the cost is small. The honest statement is PACELC's "in the E case, consistency costs latency."
- **"Spanner beats CAP."** Spanner uses TrueTime (GPS + atomic clocks) to make the partition window small enough to be negligible, but it still chooses C over A during a partition — it's CP.

## Practical implications for .NET architects

When you pick a data store or messaging layer, its CAP/PACELC profile dictates failure behaviour your code must handle:

- **CP systems (SQL Server Always On with sync commit, PostgreSQL with synchronous replication, Cosmos DB strong consistency)** — expect write timeouts or failover errors during a network blip. Your code needs retries with exponential backoff and must tolerate "I don't know if the write succeeded" ambiguity (hence idempotency keys).
- **AP systems (Cassandra, DynamoDB eventual consistency, Cosmos DB session/eventual)** — expect stale reads. Your code must tolerate reading your own write "a few ms later," handle conflict resolution (last-write-wins, vector clocks, CRDTs), and never assume monotonic reads without explicit session affinity.

> **Interview signal**: "We chose Cosmos DB at session consistency because our read-your-writes use case only cares about the same user's session, and we didn't want to pay the PA/EC latency cost of strong consistency globally."

## References

- Gilbert, Lynch — *Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services* (2002)
- Abadi — *Consistency Tradeoffs in Modern Distributed Database System Design* (IEEE Computer, 2012)
- Jepsen analyses: https://jepsen.io/analyses — real-world failures of systems that claimed stronger guarantees than they delivered

---

[Next: Consistency Models →](02-consistency-models.md) | [Back to index](README.md)
