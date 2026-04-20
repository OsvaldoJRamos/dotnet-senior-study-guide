# Replication and Read Replicas

**Replication** is keeping the same data on more than one server. It's the backbone of high availability, disaster recovery, and read scaling. The interview questions come down to three things: **what topology**, **sync or async**, and **what happens on failover**.

## Topologies

| Shape | Who accepts writes | Where it fits |
|---|---|---|
| **Primary–Replica** (single-primary) | One primary; replicas are read-only | Default for OLTP. SQL Server AGs, PostgreSQL streaming, MySQL async replication |
| **Multi-primary** | Multiple primaries accept writes | Geo-distributed writes, global low latency. MySQL Group Replication multi-primary mode, Postgres via external tools (BDR, pgEdge), Cassandra (natively masterless) |
| **Chain / cascading** | Primary → replica → sub-replica | Reduce WAN load on the primary when shipping to many secondaries |

> For most systems, **single-primary is the right answer**. Multi-primary looks attractive on the slide, then write conflicts turn into silent data corruption. Only pick it when you truly need cross-region writes and have a conflict-resolution strategy you trust.

## Synchronous vs asynchronous

### Asynchronous

Primary commits locally, then sends changes to replicas. Fast writes; replicas lag.

- ✅ Minimal write-latency impact.
- ⚠️ **Data loss window** on primary failure — everything not yet shipped is gone.
- ⚠️ Read replicas can serve stale data.

PostgreSQL streaming replication is asynchronous by default. The docs explicitly warn: *"PostgreSQL streaming replication is asynchronous by default. If the primary server crashes then some transactions that were committed may not have been replicated to the standby server, causing data loss."* (`postgresql.org/docs/current/warm-standby.html`)

### Synchronous

Primary waits for at least one replica to acknowledge before the commit returns.

- ✅ No data loss on primary failure — the acknowledged replica has the transaction.
- ⚠️ Write latency = primary + network round-trip + replica fsync.
- ⚠️ If the sync replica is down/slow, writes stall unless you have a fallback.

SQL Server Always On Availability Groups offers both modes. Microsoft Learn: *"Under synchronous-commit mode, before committing transactions, a synchronous-commit primary replica waits for a synchronous-commit secondary replica to acknowledge that it finished hardening the log. Synchronous-commit mode ensures that once a given secondary database is synchronized with the primary database, committed transactions are fully protected. This protection comes at the cost of increased transaction latency."* (`learn.microsoft.com`)

### Semi-sync (MySQL)

MySQL offers semi-synchronous replication: the primary waits for at least one replica to **receive** (not apply) the event, then commits. Middle-ground durability.

## Replication lag

The time between a commit on the primary and its visibility on a replica. Causes:

- Async replication over WAN — network round-trip + bandwidth.
- Replica applying changes single-threaded while primary committed them in parallel.
- Replica running heavy reports blocking apply.
- Long-running transactions on the primary holding WAL/log segments.

**Impact on the application:**

- Reports from the replica show stale data.
- **Read-your-writes violation**: a user updates their profile, hits a read replica on the next request, sees the old value.
- Eventual consistency everywhere — not automatically a problem, but the team must know.

## Read-your-writes consistency

The common failure mode with read replicas. After a write, the user expects their own reads to see the change. Patterns:

1. **Read from the primary for N seconds after a write** — app remembers the last write timestamp per user and routes recent reads to the primary.
2. **Sticky session / primary affinity for the critical path** — checkout writes and reads go to the primary; browsing reads go to replicas.
3. **Wait for a replication position** — some engines expose the replication LSN/GTID so the app can poll until the replica has caught up.
4. **Synchronous replication for critical flows** — if you can't tolerate staleness, pay the latency.

> "Just always read from the primary" defeats the purpose of replicas. The right shape: writes to primary, bulk reads to replicas, *user's own data* reads to primary for a short window.

## Failover

What happens when the primary dies.

| Kind | Trigger | Data loss |
|---|---|---|
| **Automatic failover** | Cluster manager detects primary failure, promotes a sync replica | None if sync replica was caught up |
| **Planned manual failover** | Operator issues failover for maintenance | None (sync mode required) |
| **Forced failover** | Primary is down and no sync replica is current; promote an async replica | **Possible** — everything not shipped is lost |

SQL Server AGs' forced failover: *"Forced failover is a form of manual failover because you must initiate it manually. Forced failover is a disaster recovery option. It's the only form of failover that's possible when the target secondary replica isn't synchronized with the primary replica."* (`learn.microsoft.com`)

**Runbook essentials:**

- Fencing the old primary (STONITH) so it can't accept writes after demotion.
- Client reconnection strategy — connection pools must re-resolve the listener/VIP.
- Catching up the old primary before reintroducing it (or reseeding from scratch).

## Engine-specific notes

### SQL Server Always On Availability Groups

- *"An availability group supports a set of read-write primary databases and one to eight sets of corresponding secondary databases."* (`learn.microsoft.com`)
- SQL Server 2019 raised the max synchronous replicas to 5 (primary + 4 sync secondaries).
- Synchronous or asynchronous **per replica** — mix for HA local + DR remote.
- Read-only routing via the **Availability Group Listener** sends `ApplicationIntent=ReadOnly` connections to a readable secondary.
- Windows: requires WSFC. Linux: Pacemaker. Read-scale AGs (no cluster manager) support read replicas without automatic failover.

### PostgreSQL streaming replication

- Physical streaming (binary WAL shipping) — the default HA mechanism.
- Async by default; synchronous via `synchronous_standby_names` + `synchronous_commit = on` (or stronger: `remote_apply`).
- A **standby** continuously applies WAL; **hot standby** serves read-only queries while applying.
- `pg_stat_replication` on the primary shows lag in bytes and time.
- **Logical replication** (separate feature) streams row-level changes using the `pgoutput` plugin, letting you selectively replicate tables or cross-version.

### MySQL replication and Group Replication

- Classic **async binlog replication** — default for decades; easy to scale reads, no automatic failover.
- **Semi-sync plugin** — primary waits for at least one replica to acknowledge receipt.
- **Group Replication** — MySQL's built-in HA/fault-tolerant cluster: *"MySQL Group Replication enables you to create elastic, highly-available, fault-tolerant replication topologies."* (`dev.mysql.com`)
- Two modes: **single-primary** (the default, with automatic primary election) and **multi-primary** (*"all servers can accept updates, even if they are issued concurrently"*). Multi-primary uses an **optimistic**, certification-based approach: the docs describe it as *"based on an optimistic replication paradigm, where statements are optimistically executed and rolled back later if necessary"* — conflicting concurrent updates to the same row are detected at certification time and only one wins.

## Connection routing patterns

```csharp
// Primary for writes, replica for reads — naive split
public class OrderReader
{
    private readonly IDbConnectionFactory _replica;
    public Task<Order?> GetAsync(int id) =>
        _replica.Query(...);  // stale reads are OK here
}

public class OrderWriter
{
    private readonly IDbConnectionFactory _primary;
    public Task CreateAsync(Order o) =>
        _primary.Execute(...);
}
```

**Route "user reads their own write" to primary:**

```csharp
// Simplified: remember last write time per user
if (DateTime.UtcNow - _recentWrites.Get(userId) < TimeSpan.FromSeconds(5))
    return await _primary.QueryAsync(...);
return await _replica.QueryAsync(...);
```

Cloud managed services (Azure SQL, RDS, Aurora) expose separate read-only endpoints — use them instead of picking a replica by hand.

## Pitfalls

- **Assuming "the replica is current".** It isn't. Monitor lag (seconds **and** bytes) and alert when it crosses SLO.
- **Running heavy analytics on a replica used for read scale.** The apply process slows down; lag grows; app starts reading stale data.
- **Forgotten failover path.** Tested failover ≠ production failover. Drill it — chaos day, real DNS/VIP failover, real app reconnect.
- **Split-brain.** A partitioned primary that still accepts writes is the classic disaster. Fencing + quorum-based cluster manager prevents it.
- **Multi-primary without conflict strategy.** CRDTs, LWW timestamps, per-key ownership — pick one *before* enabling it, not after you find duplicated rows.
- **Replica as a backup.** It isn't. A `DROP TABLE` replicates instantly. Point-in-time restore + offsite backups are separate disciplines.

---

[← Previous: Sharding and Partitioning](10-sharding-and-partitioning.md) | [Next: Change Data Capture →](12-change-data-capture.md) | [Back to index](README.md)
