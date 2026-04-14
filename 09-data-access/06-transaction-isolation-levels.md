# Transaction Isolation Levels

Isolation levels define **what a transaction can see** when other transactions are running concurrently. Higher isolation = more correctness guarantees, but more locking and lower throughput.

## The four concurrency anomalies

| Anomaly | What happens |
|---|---|
| **Dirty read** | You read rows another transaction has modified but not yet committed. If that transaction rolls back, you saw data that never "really" existed. |
| **Non-repeatable read** | You read the same row twice in the same transaction and get different values because another transaction **updated** it in between. |
| **Phantom read** | You re-run the same query in the same transaction and new rows appear because another transaction **inserted** them. |
| **Lost update** | Two transactions read a row, both update it based on the value they read, and one update silently overwrites the other. |

## The four ANSI isolation levels

Columns show whether each anomaly is **prevented** at that level.

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Typical locking |
|---|---|---|---|---|
| **Read Uncommitted** | No | No | No | Almost no locks |
| **Read Committed** | **Yes** | No | No | Short-lived shared locks on reads |
| **Repeatable Read** | **Yes** | **Yes** | No | Shared locks held until commit |
| **Serializable** | **Yes** | **Yes** | **Yes** | Range locks; transactions effectively serialized |

### Defaults across databases

| Engine | Default level | Notes |
|---|---|---|
| **SQL Server** | Read Committed | With `READ_COMMITTED_SNAPSHOT ON`, reads use row versions (MVCC) instead of shared locks |
| **PostgreSQL** | Read Committed | Always MVCC; `SERIALIZABLE` is actually Serializable Snapshot Isolation (SSI) |
| **MySQL (InnoDB)** | Repeatable Read | Uses next-key locks, which **also prevent phantoms** — stricter than the ANSI definition |
| **Oracle** | Read Committed | Always MVCC; no "dirty read" level exists at all |

## Each level in detail

### Read Uncommitted

Reads ignore locks and can see uncommitted data. Fastest, least safe.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- equivalent to WITH (NOLOCK) in SQL Server
SELECT * FROM Orders WHERE CustomerId = 42;
```

> Avoid in business logic. People still reach for `WITH (NOLOCK)` for "performance", but you risk reading phantoms, missing rows, and even duplicate rows because of page splits mid-scan.

### Read Committed

You only see committed data. Locks on reads are released as soon as the row is read.

- Prevents **dirty reads**.
- Still allows **non-repeatable reads** and **phantoms** inside the same transaction.
- In SQL Server with **`READ_COMMITTED_SNAPSHOT ON`**, reads use row versions instead of shared locks (MVCC-style) — no blocking on readers.

### Repeatable Read

Shared locks acquired during reads are held until the transaction commits, so rows you already read cannot be changed by others.

- Prevents **dirty** and **non-repeatable reads**.
- **Phantoms** are still possible — another transaction can insert new rows matching your predicate.

### Serializable

Strongest level. The database behaves as if transactions ran one after another.

- SQL Server uses **range locks** on the index range covered by your query, blocking inserts that would match.
- Prevents all anomalies but increases blocking and deadlock risk.

### Snapshot (SQL Server, Postgres, Oracle)

A separate model based on **row versioning / MVCC** rather than locking. Each transaction sees a consistent snapshot of the database at its start time.

```sql
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRAN
    SELECT * FROM Orders WHERE CustomerId = 42;
    -- Other transactions' commits are invisible inside this transaction
COMMIT
```

- No reader/writer blocking.
- Writers still conflict: an **update conflict** error is raised if two snapshot transactions try to modify the same row.
- Cost: version store in `tempdb`.

### Write skew — the snapshot gotcha

Snapshot isolation prevents dirty/non-repeatable/phantom reads, **but not write skew**. Write skew happens when two transactions each read a set of rows, make decisions based on that read, and write results that together violate an invariant that neither violated alone.

Classic example: a hospital rule says "at least one doctor must be on call". Two on-call doctors both read `count >= 2`, each decide it's safe to go off call, and both update their status in parallel. Under snapshot isolation, both commits succeed — and now **nobody is on call**.

- **SQL Server Snapshot** → allows write skew.
- **PostgreSQL Serializable (SSI)** → detects and aborts one of the transactions.
- **SQL Server Serializable** → also prevents it (via range locks).

If your invariant spans multiple rows, use `SERIALIZABLE` or add application-level locking (e.g., `SELECT ... FOR UPDATE` on a sentinel row).

## Setting the isolation level

### T-SQL

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRAN
    -- ...
COMMIT
```

### ADO.NET

```csharp
using var tx = connection.BeginTransaction(IsolationLevel.Serializable);
```

### EF Core

```csharp
using var tx = await context.Database
    .BeginTransactionAsync(IsolationLevel.Snapshot);
```

### TransactionScope

```csharp
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted },
    TransactionScopeAsyncFlowOption.Enabled);
```

## Choosing an isolation level

- **Default to Read Committed** (or `READ_COMMITTED_SNAPSHOT`) — fits most OLTP workloads.
- **Repeatable Read** when a single transaction re-reads the same rows and needs them stable (e.g., validate then update).
- **Serializable** when you need strict invariants that no new matching row can appear (e.g., "only one active subscription per user").
- **Snapshot** when read-heavy workloads are blocked by writers — great for reporting inside OLTP databases.
- **Read Uncommitted** only for approximate dashboards where stale/dirty data is acceptable.

## Dealing with lost updates

Higher isolation reduces but doesn't eliminate lost updates. Two common strategies:

### Optimistic concurrency (preferred)

Add a `RowVersion`/`timestamp` column. On update, include it in the `WHERE` clause; if no rows were affected, the row changed under you and you retry.

```csharp
// EF Core
[Timestamp]
public byte[] RowVersion { get; set; }

// SaveChangesAsync throws DbUpdateConcurrencyException on conflict
```

### Pessimistic locking

Lock the row when you read it:

```sql
SELECT * FROM Orders WITH (UPDLOCK, ROWLOCK)
WHERE Id = @id;
```

Use sparingly — causes blocking and is the main source of deadlocks.

## Deadlocks

A deadlock happens when two transactions each hold a lock the other needs. The database detects it and **kills one transaction** (error 1205 in SQL Server).

Mitigations:
- Access tables/rows in the **same order** across transactions.
- Keep transactions **short**.
- Lower isolation where safe, or use snapshot isolation to remove read/write conflicts.
- Add retry logic for transient deadlock errors.

---

[← Previous: SQL Indexes](05-sql-indexes.md) | [Back to index](README.md)
