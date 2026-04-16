# Database Locks

Database locks are what the engine actually does **under the hood** when you set an isolation level. Understanding them is the difference between "the app is slow" and "the app is slow because of lock escalation on a non-covering index". This file covers SQL Server deeply (because most .NET shops use it) with PostgreSQL contrasts throughout.

For the application-level counterpart (`lock`, `Monitor`, `SemaphoreSlim`), see [Locks in Depth](../04-concurrency-and-parallelism/07-locks-in-depth.md). For isolation-level semantics, see [Transaction Isolation Levels](06-transaction-isolation-levels.md).

## Lock granularity

SQL Server can lock at many levels. Smaller granularity = more concurrency, but more lock structures in memory.

| Resource | What it covers |
|---|---|
| `RID` | A single row in a heap (no clustered index). |
| `KEY` | A single row identified by an index key (B-tree). |
| `PAGE` | 8 KB data or index page. |
| `EXTENT` | 8 contiguous pages. |
| `HoBT` | Entire heap or B-tree. |
| `TABLE` | Whole table including all its indexes. |
| `FILE` / `DATABASE` | Rare; used for admin operations. |
| `XACT` | Transaction ID lock — used by **optimized locking** (SQL Server 2022+). |

SQL Server chooses the granularity automatically; you usually accept that choice. The hints `ROWLOCK`, `PAGLOCK`, `TABLOCK` / `TABLOCKX` override it.

PostgreSQL has a similar hierarchy but no lock escalation (see below).

## Lock modes (SQL Server)

| Mode | Purpose | Compatible with |
|---|---|---|
| **S** (Shared) | Reads | S, U, IS |
| **U** (Update) | "I intend to update" — pre-modification read; prevents conversion deadlocks | S, IS |
| **X** (Exclusive) | Writes (`INSERT`/`UPDATE`/`DELETE`) | Nothing |
| **IS** (Intent Shared) | Signals "a lower-level S is taken somewhere in this object" | S, U, IS, IX, SIX |
| **IX** (Intent Exclusive) | Signals a lower-level X is coming | IS, IX |
| **SIX** (Shared with Intent Exclusive) | Full S at this level + will take IX below | IS |
| **Sch-S / Sch-M** | Schema Stability / Schema Modification (DDL) | See below |
| **BU** | Bulk Update for parallel bulk loads with `TABLOCK` | BU |
| **RangeS-S / RangeS-U / RangeI-N / RangeX-X** | Key-range locks used by `SERIALIZABLE` to prevent phantoms | varies |

### Compatibility matrix (simplified)

Rows = lock currently held. Columns = lock requested. `Y` = granted immediately; `N` = blocks.

| Held \ Requested | S | U | X | IS | IX | SIX |
|---|---|---|---|---|---|---|
| **S**   | Y | Y | N | Y | N | N |
| **U**   | Y | N | N | Y | N | N |
| **X**   | N | N | N | N | N | N |
| **IS**  | Y | Y | N | Y | Y | Y |
| **IX**  | N | N | N | Y | Y | N |
| **SIX** | N | N | N | Y | N | N |

Two takeaways:
- **X is compatible with nothing.** Any writer blocks all other access to the same row.
- **U with U is NOT compatible** — that's what stops two readers from both upgrading to X and deadlocking.

### Schema locks

- `Sch-S` (Schema Stability) — held by any query while it compiles/runs. Even `READUNCOMMITTED` / `NOLOCK` takes this.
- `Sch-M` (Schema Modification) — taken by DDL (`ALTER TABLE`, `DROP INDEX`, ...). Incompatible with every other lock. This is why an `ALTER TABLE ADD COLUMN` on a busy table can freeze the system.

## How isolation levels translate to locks

See [Transaction Isolation Levels](06-transaction-isolation-levels.md) for the anomaly semantics. Here is the locking behavior underneath:

| Level | Read behavior | Lock lifetime |
|---|---|---|
| `READ UNCOMMITTED` / `NOLOCK` | No S locks; Sch-S only | Sch-S for the statement |
| `READ COMMITTED` (default, `READ_COMMITTED_SNAPSHOT OFF`) | S taken row-by-row | S released as soon as the row is read |
| `READ COMMITTED SNAPSHOT` (RCSI) | Row versions — **no S locks** | Writer versioning in `tempdb` |
| `REPEATABLE READ` | S on every row read | S held until commit — prevents non-repeatable reads |
| `SERIALIZABLE` | Key-range locks (`RangeS-S`, etc.) | Held until commit — prevents phantoms |
| `SNAPSHOT` | Row versions, transaction-level snapshot | No read locks at all |

Writes always take X locks regardless of isolation level. Isolation level only affects **reads**.

```sql
-- Enable RCSI database-wide (very common production setting)
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON;
-- Enable SNAPSHOT isolation (opt-in per transaction)
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;
```

## Lock escalation

When a single statement acquires **~5,000 locks on one table/partition**, or server-wide lock memory exceeds internal thresholds (roughly 24% of DB engine memory), SQL Server **escalates** many fine-grained locks to a single `HoBT` or table lock. The trigger is re-checked every additional ~1,250 locks.

Why it happens: tracking millions of row locks is expensive. Collapsing them saves memory at the cost of concurrency.

Why it hurts: a single `UPDATE` that should affect 10,000 rows can lock the entire table, blocking everyone else.

### Detecting it

```sql
-- Extended Event for escalation
CREATE EVENT SESSION [Track_lock_escalation] ON SERVER
ADD EVENT sqlserver.lock_escalation (
    ACTION (sqlserver.sql_text, sqlserver.database_name)
);

-- Live locks
SELECT resource_type, request_mode, request_status, COUNT(*) AS n
FROM sys.dm_tran_locks
GROUP BY resource_type, request_mode, request_status;
```

### Preventing it

```sql
-- Most targeted: disable escalation on a specific table
ALTER TABLE dbo.LogEvents SET (LOCK_ESCALATION = DISABLE);

-- Or escalate to partition level on partitioned tables
ALTER TABLE dbo.Orders SET (LOCK_ESCALATION = AUTO);
```

Better approaches than disabling escalation:

- **Batch big DML**: delete/update in chunks of a few hundred rows inside a loop.
- **Enable RCSI / Snapshot** so readers don't take S locks at all.
- **Use optimized locking** (SQL Server 2022+, requires Accelerated Database Recovery + RCSI) — keeps only a transaction-ID lock for the duration, releasing row/page locks incrementally.

> **PostgreSQL has no lock escalation.** Row locks are stored in the tuple header and in `pg_locks` on demand; there is no memory threshold that collapses them. Instead, long-running transactions bloat the row-version chain — a different problem with a different fix (`VACUUM`).

## Deadlocks (DB side)

Two transactions each hold a lock the other wants. SQL Server's **deadlock monitor** wakes roughly every 5 seconds, detects the cycle, and kills the **deadlock victim** (the transaction estimated to be cheapest to roll back), raising **error 1205** in the client:

```
Transaction (Process ID N) was deadlocked on lock resources with another process
and has been chosen as the deadlock victim. Rerun the transaction.
```

### Mitigation

- **Access objects in the same order** across all transactions that touch them.
- **Keep transactions short** — don't wait for user input inside one.
- **Use RCSI / Snapshot** to remove read/write conflicts.
- **Cover your predicates with indexes** — a scan locks far more rows than a seek.
- **Set `DEADLOCK_PRIORITY`** on background jobs you'd rather sacrifice.
- **Retry with backoff** on 1205 — deadlocks are transient; your retry policy should catch them.

PostgreSQL behaves the same: auto-detects deadlocks and aborts one transaction with `SQLSTATE 40P01`.

### Capturing deadlock graphs

```sql
-- System Health session captures deadlock XML by default
SELECT XEvent.query('.') AS DeadlockGraph
FROM (
    SELECT CAST(target_data AS XML) AS TargetData
    FROM sys.dm_xe_session_targets st
    JOIN sys.dm_xe_sessions s ON s.address = st.event_session_address
    WHERE s.name = 'system_health'
) AS Data
CROSS APPLY TargetData.nodes('RingBufferTarget/event[@name="xml_deadlock_report"]') AS X(XEvent);
```

## Pessimistic vs optimistic concurrency

| Strategy | How | When to use |
|---|---|---|
| **Pessimistic** | Lock the row at read time (`SELECT ... WITH (UPDLOCK, ROWLOCK)` in SQL Server, `SELECT ... FOR UPDATE` in Postgres). Other writers block. | Short critical sections, expected contention, conflict retries are expensive. |
| **Optimistic** | Read without locking, track a version, check the version on write; conflict = retry. | Web apps, high read/write ratio, conflicts are rare. |

### Optimistic in SQL Server

```sql
CREATE TABLE Orders (
    Id         int PRIMARY KEY,
    Total      decimal(10,2),
    RowVersion rowversion NOT NULL
);

-- The update
UPDATE Orders
SET Total = @newTotal
WHERE Id = @id AND RowVersion = @expectedRowVersion;
-- @@ROWCOUNT = 0 means someone else updated first → retry
```

### Optimistic in EF Core

```csharp
public class Order
{
    public int Id { get; set; }
    public decimal Total { get; set; }

    [Timestamp] // maps to rowversion
    public byte[] RowVersion { get; set; } = default!;
}

// SaveChangesAsync throws DbUpdateConcurrencyException when RowVersion doesn't match
```

### Pessimistic in SQL Server

```sql
BEGIN TRAN;

SELECT Total
FROM Orders WITH (UPDLOCK, ROWLOCK)
WHERE Id = @id;

-- compute new total in app

UPDATE Orders SET Total = @newTotal WHERE Id = @id;

COMMIT;
```

`UPDLOCK` prevents two transactions from both reading and then both updating — a classic lost-update pattern.

### Pessimistic in PostgreSQL

```sql
BEGIN;
SELECT total FROM orders WHERE id = $1 FOR UPDATE;
UPDATE orders SET total = $2 WHERE id = $1;
COMMIT;
```

`FOR UPDATE` acquires a row-level exclusive lock; other writers and other `FOR UPDATE` / `FOR SHARE` callers block.

## Row versioning / MVCC

SQL Server and PostgreSQL both implement **multiversion concurrency control**, though with very different mechanics.

### SQL Server

Opt-in via database options:
- `READ_COMMITTED_SNAPSHOT ON` — read committed uses row versions; writers don't block readers. **Strongly recommended** for most OLTP workloads.
- `ALLOW_SNAPSHOT_ISOLATION ON` — additionally enables `SET TRANSACTION ISOLATION LEVEL SNAPSHOT`.

Version store lives in `tempdb`. Writer-writer conflicts raise error 3960.

### PostgreSQL

**Always MVCC**, from the ground up. Every row (tuple) has `xmin` (created-by transaction) and `xmax` (deleted-by transaction); each transaction sees a snapshot defined by the set of committed transactions at its start.

Consequences:
- Readers never block writers, writers never block readers — always.
- `UPDATE` physically **inserts a new row version** and marks the old one dead. `VACUUM` (and autovacuum) reclaim dead tuples.
- Long-running transactions prevent vacuum from reclaiming space → **bloat**.
- PostgreSQL's `SERIALIZABLE` is **Serializable Snapshot Isolation (SSI)** — detects serialization anomalies at commit time and aborts one transaction with `SQLSTATE 40001`. No range locks.

## Table hints (SQL Server)

Hints override the optimizer's default locking. Use sparingly.

| Hint | Meaning | Danger |
|---|---|---|
| `NOLOCK` / `READUNCOMMITTED` | No S locks taken; allow dirty reads | Can return uncommitted data, **missing rows, duplicate rows** due to page splits during a scan. Never use in business logic. |
| `READPAST` | Skip rows locked by others | Good for work-queue readers; silently hides rows in other contexts |
| `UPDLOCK` | Take U locks on reads | Correct for read-then-update patterns |
| `HOLDLOCK` / `SERIALIZABLE` | Hold S (and range) locks to end of transaction | Heavy blocking |
| `ROWLOCK` | Force row-level granularity | May increase lock count and trigger escalation |
| `TABLOCK` / `TABLOCKX` | Take a table-level S/X lock | Massive blast radius; useful for bulk loads with minimal logging |
| `XLOCK` | Take X locks on reads | Exclusive for the whole transaction |
| `NOWAIT` | Fail immediately instead of waiting | Combine with error handling for fast-fail |

```sql
-- Read-then-update without race
BEGIN TRAN;
SELECT @balance = Balance FROM Account WITH (UPDLOCK, ROWLOCK) WHERE Id = @id;
-- validate
UPDATE Account SET Balance = @balance - @debit WHERE Id = @id;
COMMIT;

-- Work-queue dequeue
WITH next AS (
    SELECT TOP(1) * FROM Jobs WITH (READPAST, ROWLOCK, UPDLOCK)
    WHERE Status = 'Pending' ORDER BY EnqueuedAt
)
UPDATE next SET Status = 'InFlight'
OUTPUT inserted.*;
```

> **The `NOLOCK` trap.** Developers use it for "performance", but it can return rows that don't exist, skip rows mid-query, or return a row **twice** because of page splits. A senior interviewer will call this out. The modern answer is `READ_COMMITTED_SNAPSHOT ON` — same non-blocking behavior, zero correctness risk.

## PostgreSQL-specific locking

### Row-level

```sql
SELECT ... FOR UPDATE;         -- strongest row lock; blocks updaters and other lockers
SELECT ... FOR NO KEY UPDATE;  -- like FOR UPDATE but doesn't conflict with FOR KEY SHARE
SELECT ... FOR SHARE;          -- shared row lock; blocks updates
SELECT ... FOR KEY SHARE;      -- weakest; only blocks key-changing updates (used by FKs)

-- Non-blocking variants
SELECT ... FOR UPDATE NOWAIT;      -- fail immediately if locked
SELECT ... FOR UPDATE SKIP LOCKED; -- Postgres equivalent of READPAST — perfect for work queues
```

### Table-level (8 modes)

Most-used:
- `ACCESS SHARE` — taken by plain `SELECT`. Conflicts only with `ACCESS EXCLUSIVE`.
- `ROW EXCLUSIVE` — taken by `INSERT`/`UPDATE`/`DELETE`/`MERGE`.
- `ACCESS EXCLUSIVE` — taken by `DROP TABLE`, `TRUNCATE`, most `ALTER TABLE`. Blocks everything, including `SELECT`. This is the PostgreSQL equivalent of `Sch-M`.

### Advisory locks

Application-controlled locks identified by an integer (or a pair). The database doesn't attach meaning; your code does.

```sql
-- Session-scoped advisory lock (manual release)
SELECT pg_advisory_lock(12345);
-- do work
SELECT pg_advisory_unlock(12345);

-- Try-lock (non-blocking)
SELECT pg_try_advisory_lock(12345);

-- Transaction-scoped (auto-released at commit/rollback) — usually the right choice
SELECT pg_advisory_xact_lock(12345);
```

Great for "at most one worker processes tenant X at a time" without touching a real row. No SQL Server direct equivalent — you'd use `sp_getapplock` / `sp_releaseapplock`.

## Diagnostics

### SQL Server

```sql
-- Current locks
SELECT request_session_id, resource_type, resource_description,
       request_mode, request_status
FROM sys.dm_tran_locks
WHERE request_session_id <> @@SPID;

-- Who is blocking whom
SELECT blocking_session_id, session_id, wait_type, wait_time, wait_resource,
       last_wait_type
FROM sys.dm_os_waiting_tasks
WHERE blocking_session_id IS NOT NULL;

-- Top contended locks (server-level counters)
SELECT wait_type, waiting_tasks_count, wait_time_ms, max_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type LIKE 'LCK%'
ORDER BY wait_time_ms DESC;
```

### PostgreSQL

```sql
-- Current locks
SELECT pid, relation::regclass, mode, granted, query_start
FROM pg_locks
LEFT JOIN pg_stat_activity USING (pid);

-- Blocking tree (Postgres 9.6+)
SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query,
       blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

## Rule-of-thumb table

| Situation | Preferred approach |
|---|---|
| OLTP reads shouldn't block OLTP writes | SQL Server: `READ_COMMITTED_SNAPSHOT ON`. Postgres: already MVCC. |
| Read-then-update on a single row | `UPDLOCK, ROWLOCK` (SQL Server) or `SELECT ... FOR UPDATE` (Postgres) |
| Work queue | `READPAST` + `ROWLOCK` + `UPDLOCK` (SQL Server) or `FOR UPDATE SKIP LOCKED` (Postgres) |
| Web-app concurrency for forms / grids | Optimistic (`RowVersion` + EF Core concurrency token) |
| "Only one worker per tenant at a time" | Advisory lock (Postgres) or `sp_getapplock` (SQL Server) |
| Reporting on a busy OLTP DB | `SNAPSHOT` isolation (SQL Server) or Postgres replica |
| Big batch `DELETE` / `UPDATE` | Loop in chunks; avoid lock escalation |
| "My `ALTER TABLE` hangs" | Use `ONLINE` operations (SQL Server Enterprise) or short maintenance windows; avoid long transactions around DDL |
| Cross-transaction invariants across multiple rows | `SERIALIZABLE` (SQL Server) or `SERIALIZABLE` (Postgres SSI) — retry on 40001 |

## Common senior-interview gotchas

- **`NOLOCK` can return duplicate, missing, or phantom rows** — not just uncommitted ones. The fix is RCSI, not `NOLOCK`.
- **Postgres has no lock escalation**, but long-running transactions cause bloat.
- **`READ_COMMITTED_SNAPSHOT ON` vs `ALLOW_SNAPSHOT_ISOLATION ON`** are **two different** options in SQL Server — easy to confuse.
- **Writer-writer conflicts under snapshot isolation raise error 3960** in SQL Server; under `SERIALIZABLE` in Postgres you get `SQLSTATE 40001`. Both require retry logic.
- **MySQL InnoDB's default is `REPEATABLE READ`**, and its next-key locks also prevent phantoms — stricter than the ANSI definition.
- **A non-covering index turns a seek into a scan**, which locks far more rows and can trigger escalation.
- **Deadlock victim selection is based on rollback cost**, not on "who started first". You can influence it with `SET DEADLOCK_PRIORITY LOW/HIGH`.
- **Advisory locks (Postgres) don't protect actual data** — they're purely cooperative. A client that doesn't take the lock sees nothing.
- **See [Locks in Depth](../04-concurrency-and-parallelism/07-locks-in-depth.md)** for the application-side companion (`lock`, `Monitor`, `SemaphoreSlim`, `Interlocked`).

---

[← Previous: Transaction Isolation Levels](06-transaction-isolation-levels.md) | [Back to index](README.md) | [Next: Execution Plans →](08-execution-plans.md)
