# Consensus — Raft and Paxos

## Why consensus matters

Every distributed system eventually needs a set of nodes to **agree on a single value** in the face of failures: who is the leader, what's the next entry in a replicated log, which node owns which shard, whether a lock is held.

This is the **consensus problem**. The FLP impossibility result (Fischer, Lynch, Paterson, 1985) proved it's **impossible to guarantee consensus in an asynchronous system with even one crash failure**. Practical protocols (Paxos, Raft) circumvent FLP by assuming partial synchrony — messages eventually arrive, clocks are eventually close — and by accepting that under pathological conditions progress can stall (but **safety is never violated**).

You don't implement Raft yourself. You use a system that implements it — etcd, Consul, ZooKeeper (ZAB), CockroachDB, MongoDB's replica set election, Kafka's KRaft controller — and you need to understand the trade-offs.

## Paxos (Lamport, 1998)

Leslie Lamport's seminal "The Part-Time Parliament" (1998) and the more accessible "Paxos Made Simple" (2001).

### Roles

- **Proposer** — proposes a value.
- **Acceptor** — votes on proposals; a quorum of acceptors must agree for consensus.
- **Learner** — learns the chosen value once agreement is reached.

### Phases (Basic Paxos, single decision)

1. **Prepare / Promise** — proposer sends a prepare message with a unique, monotonically increasing number *n*. Acceptors that have not seen a higher number promise not to accept any proposal numbered less than *n*.
2. **Accept / Accepted** — if the proposer got promises from a majority, it sends an accept request with its value (or the most recent value any acceptor reported). Acceptors accept unless they've since promised to a higher number.

### Multi-Paxos

Running Basic Paxos for each log entry is expensive. Lamport explains: *"if the leader is relatively stable, phase 1 becomes unnecessary. Thus, it is possible to skip phase 1 for future instances of the protocol with the same leader."* This optimisation — Multi-Paxos — reduces steady-state latency to one round trip per entry.

### Why Paxos is hard

Paxos is notoriously difficult to understand and correctly implement. The paper is short; turning it into a production system requires many unwritten decisions (leader election, log compaction, membership changes), and bugs in those are hard to find.

## Raft (Ongaro & Ousterhout, 2014)

Designed explicitly for **understandability**. The paper's title: "In Search of an Understandable Consensus Algorithm (Extended Version)". Raft decomposes consensus into three sub-problems:

1. **Leader election** — at any time there is one leader.
2. **Log replication** — the leader accepts client commands and replicates them to followers.
3. **Safety** — properties that hold despite arbitrary failures.

### Server states

At any time, each server is in exactly one state:

```text
         (no heartbeat)
Follower ────────────────▶ Candidate ────▶ Leader
     ▲                          │             │
     │                          │ (timeout)   │ (discovers higher term)
     └──────────────────────────┴─────────────┘
```

### Leader election

- Every follower has a randomized **election timeout** — typically **150–300 ms** per the Raft paper.
- If a follower hears no heartbeat from the leader within its timeout, it becomes a **candidate**, increments the term, votes for itself, and requests votes from others.
- A candidate wins if a **majority** grants its vote. On tie (split vote), another randomised timeout triggers a new election.
- The winner becomes leader and immediately starts sending heartbeats (empty AppendEntries) to suppress new elections.

### Log replication

1. The leader accepts client commands, appends each as a log entry, and sends `AppendEntries` RPCs to followers in parallel.
2. Once a majority of the cluster has replicated the entry, the leader marks it **committed** and applies it to its state machine.
3. The leader notifies followers (via the next AppendEntries) of the advanced commit index; they apply in order.

### Safety properties

The Raft paper proves five safety properties:

| Property | Statement |
|---|---|
| **Election Safety** | At most one leader can be elected in a given term. |
| **Leader Append-Only** | A leader never overwrites or deletes its own log entries. |
| **Log Matching** | If two logs contain an entry with the same index and term, all entries up to that index are identical. |
| **Leader Completeness** | If an entry is committed in a term, every leader of all higher terms has that entry. |
| **State Machine Safety** | Once an entry is applied to a state machine at a given index, no other state machine ever applies a different value for that index. |

Together they guarantee the cluster agrees on a single, ever-growing log.

### Quorum math

For a cluster of *N* nodes, the majority is `floor(N/2) + 1`. Failure tolerance is:

| Cluster size | Majority | Failures tolerated |
|---|---|---|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

An **even-sized cluster offers no extra failure tolerance** — a 4-node cluster tolerates 1 failure, same as 3 — and doubles the chance of a split vote. **Always use odd-sized Raft clusters.**

## Raft vs Paxos

| Dimension | Paxos (classical / Multi-Paxos) | Raft |
|---|---|---|
| Primary goal | Provability and theoretical minimality | Understandability |
| Leader concept | Optional optimisation in Multi-Paxos | First-class, always present in steady state |
| Log structure | Not prescribed by Basic Paxos | Built-in (indexed, term-tagged) |
| Membership changes | Underspecified in the original paper | Joint consensus is part of the spec |
| Implementations | Chubby (Google), Spanner variants | etcd, Consul, CockroachDB, TiKV, Kafka KRaft, MongoDB replica sets use a Raft-inspired protocol |

Both have equivalent safety guarantees. The industry has largely standardised on Raft for new systems because it's easier to implement correctly.

## Where you encounter Raft in practice

### etcd

The distributed key-value store Kubernetes uses for cluster state. etcd's docs describe its design as prioritising **consistency and fault-tolerance over availability**. Key characteristics:

- Strongly consistent reads (linearizable) and writes, backed by Raft.
- Watch API for reactive configuration (used heavily by Kubernetes controllers).
- Primitives: leases, elections, locks, distributed transactions.
- Recommended cluster size: 3, 5, or 7 nodes.

Typical use cases: Kubernetes cluster state, service discovery, leader election for your own services, distributed feature flags.

### Consul

HashiCorp's service mesh / service discovery tool. Uses Raft for its server cluster. In addition to Raft, Consul's gossip protocol (based on SWIM) handles membership and failure detection at a larger scale.

Use cases: service registration and discovery, health checking, KV store, Connect (mTLS mesh).

### Kafka KRaft

Since KIP-500, Apache Kafka has been removing its ZooKeeper dependency in favour of a Raft-based metadata quorum (KRaft). In modern Kafka releases, the cluster controller uses a Raft-inspired protocol for cluster metadata. Verify the exact version behaviour against the current Kafka release notes; the ZooKeeper-to-KRaft migration story has evolved across releases and the "production-ready" and "ZooKeeper-removed" milestones are in different versions.

### MongoDB replica sets

MongoDB's replica set election protocol is Raft-inspired (protocol version 1). Primary election and log replication follow Raft's approach; consistency guarantees are configured per-operation via read/write concerns.

### CockroachDB and TiKV

Raft is the per-range replication protocol. Each data "range" is a Raft group of 3 or 5 replicas. Transactions coordinate across Raft groups for cross-range atomicity.

## When to run your own Raft cluster

Honestly: **rarely**.

- Use an existing store (etcd, Consul, ZooKeeper) as the consensus primitive, and let it handle leader election / locks / config for your services.
- Implementing Raft correctly is a 12–18-month project with ongoing maintenance. Hashicorp, etcd-io, CoreOS, and Azure teams have spent years ironing out edge cases (log compaction, snapshotting, membership changes, pre-vote).
- If you genuinely need state-machine replication in your own service (not just distributed locks), look at libraries like `hashicorp/raft` (Go) — there is no mainstream, production-grade embeddable Raft library for .NET at the time of writing; validate the current landscape before committing.

## Pitfalls

- **Even-sized clusters.** 2 nodes cannot elect a leader if one is down. 4 tolerates the same failures as 3 at higher cost. **Always odd.**
- **Running Raft across WAN with no tuning.** Default 150–300 ms election timeouts are for LAN. Across regions, latency easily crosses 100 ms one-way and you get spurious elections. Either tune timeouts up or keep the cluster regional and use async replication for DR.
- **Mistaking quorum-reads for linearizable reads.** etcd needs a confirmed leader to return linearizable reads — a follower read without a round trip to the leader can be stale. Check your client's read mode.
- **Confusing Raft with Byzantine tolerance.** Raft assumes crash-stop failures, not malicious actors. For Byzantine scenarios (blockchain, untrusted participants), you need BFT protocols like PBFT.
- **Assuming the cluster is always available.** If you lose the majority (2 of 3, or 3 of 5), the cluster refuses writes. This is correct behaviour — it's the C in CP — but it means ops must plan for it.

## References

- Ongaro, Ousterhout — *In Search of an Understandable Consensus Algorithm (Extended Version)*: https://raft.github.io/raft.pdf
- Raft visualisation and reference implementations: https://raft.github.io
- Lamport — *Paxos Made Simple* (2001)
- etcd — *Why etcd*: https://etcd.io/docs/v3.5/learning/why/
- Fischer, Lynch, Paterson — *Impossibility of Distributed Consensus with One Faulty Process* (1985)
- Jepsen analyses of consensus systems: https://jepsen.io/analyses

---

[← Previous: Distributed Transactions](06-distributed-transactions.md) | [Next: Clocks and Ordering →](08-clocks-and-ordering.md) | [Back to index](README.md)
