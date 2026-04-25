# High Availability and Redundancy

The previous topics in this section (retries, circuit breakers, bulkheads, timeouts) keep a single instance behaving sanely under failure. **High availability** is the architectural decision sitting on top: how many instances, where, and what happens when one dies.

## Availability and the "nines"

Availability is `Uptime / (Uptime + Downtime)`. Industry shorthand is the number of nines.

| Target | Allowed downtime / year | Allowed downtime / month |
|---|---|---|
| 99% (two nines) | ~3.65 days | ~7.20 hours |
| 99.9% (three nines) | ~8.77 hours | ~43.83 minutes |
| 99.95% | ~4.38 hours | ~21.92 minutes |
| 99.99% (four nines) | ~52.60 minutes | ~4.38 minutes |
| 99.999% (five nines) | ~5.26 minutes | ~26.30 seconds |

> **Senior reality:** four nines is hard. Five nines requires multi-region, automated failover, no human in the loop, and a budget to match. Quoting "five nines" without owning the architectural cost is a red flag.

### MTBF and MTTR

- **MTBF (Mean Time Between Failures)** — how long a component runs before failing on average.
- **MTTR (Mean Time To Recovery)** — how long it takes to restore service after a failure.
- `Availability ≈ MTBF / (MTBF + MTTR)`.

The lever is usually MTTR. You don't make hardware fail less; you make recovery faster (automated failover, fast deploys, runbooks, observability).

## Redundancy

Redundancy is the **mechanism** that makes HA possible: extra components so a single failure is not a service-wide outage.

| Type | Example |
|---|---|
| **Hardware** | RAID, dual PSUs, multiple NICs |
| **Software / instance** | Multiple app instances behind a load balancer |
| **Data** | Database replicas, multi-AZ storage, CRDB / Spanner-style replication |
| **Network** | Multiple ISPs, BGP multipath, redundant load balancers |
| **Geographic** | Multi-AZ within a region; multi-region across continents |

### N+1 vs 2N

- **N+1** — `N` instances handle peak load; one extra exists to absorb a single failure. Cheap, but a second simultaneous failure exhausts capacity.
- **2N** — full duplicate set. Expensive, used when capacity loss during a failure is unacceptable (e.g., financial trading, medical telemetry).
- **2N+1** — full duplicate plus a spare; rare outside critical infrastructure.

## Active-passive vs active-active

| | Active-passive (hot-cold / hot-warm) | Active-active (hot-hot) |
|---|---|---|
| Traffic | Only the primary serves; standby idle or replicating | All nodes serve simultaneously |
| Failover | Promote standby (manual or automatic, takes seconds-to-minutes) | None needed — surviving nodes absorb the load |
| Cost | Lower — standby underutilized | Higher — full capacity always running |
| Data sync | Replication lag exists; risk of data loss on failover | Conflict resolution required (write quorum, last-writer-wins, CRDTs) |
| When to use | RTO measured in minutes is acceptable; cost-sensitive | Sub-second RTO; large customer base; cannot afford even a brief blip |

> **Trade-off you'll be quizzed on:** active-active gives you both HA and capacity, but it forces you to solve consistency. Active-passive is simpler but you accept a failover window.

## Failover

The act of moving traffic from a failed primary to a healthy replacement.

- **Detection** — heartbeat, health check, observed error rate. Tune thresholds: too sensitive → flapping; too loose → long outages.
- **Promotion** — replica becomes primary (e.g., PostgreSQL `pg_ctl promote`, SQL Server AG manual failover, Redis Sentinel auto-promote).
- **Traffic redirection** — DNS update (slow, TTL-bounded), load balancer reconfig (faster), or anycast withdrawal (fastest).
- **Pitfall: split-brain** — both nodes believe they are primary after a network partition. Quorum-based consensus is the standard fix: etcd and Consul use **Raft**, ZooKeeper uses **Zab** (ZooKeeper Atomic Broadcast). You need 3+ nodes so a split produces a clear minority and majority — with only 2 nodes, neither side has a quorum and you're stuck.

## Geographic redundancy

| Scope | Failure domain handled |
|---|---|
| **Multi-AZ (within a region)** | Datacenter / power / network outage in one AZ |
| **Multi-region** | Region-wide outage, regulator-imposed failover, large-scale natural events |

Multi-region is significantly more complex: data replication latency, regulatory data-residency constraints, DNS-based traffic steering (Route 53, Azure Front Door, Cloudflare Load Balancing), and the cost of running the second region.

## What "high availability" is **not**

- Not a substitute for circuit breakers, retries, or timeouts. Those handle in-process failure modes; HA handles instance / region failure modes.
- Not free. Quoting an SLO without quoting the cost (extra instances, replicas, cross-AZ data transfer) is naive.
- Not the same as **disaster recovery (DR)**. HA = continuous availability under failures the design anticipates. DR = recovery from a catastrophic event the design did not. Both have RTO and RPO targets, but DR's are usually larger and the recovery process usually involves human decisions.

---

[← Previous: Error Budgets](08-error-budgets.md) | [Back to index](README.md)
