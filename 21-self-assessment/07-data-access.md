# Data Access

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between an ORM, a Micro ORM, and raw ADO.NET? When would you pick each?

<details>
<summary>Reveal answer</summary>

| Approach | Example | Strengths | Best for |
|----------|---------|-----------|----------|
| **ORM** | EF Core | Change tracking, migrations, LINQ, relationships | Complex domain models, rapid development |
| **Micro ORM** | Dapper | Fast, maps SQL to objects, minimal overhead | Read-heavy, performance-critical queries |
| **ADO.NET** | SqlConnection | Full control, no abstraction | Bulk operations, stored procs, extreme perf needs |

Choose **EF Core** when productivity matters and the domain is complex. Choose **Dapper** for high-throughput reads or when you want full SQL control. Choose **ADO.NET** for edge cases like bulk inserts or very low-level data access.

Deep dive: [ORM vs Micro ORM vs ADO.NET](../09-data-access/01-orm-vs-microorm-vs-adonet.md)

</details>

---

### 2. What is the N+1 problem in Entity Framework, and how do you fix it?

<details>
<summary>Reveal answer</summary>

The **N+1 problem** occurs when EF loads a parent entity (1 query) and then lazily loads each child entity individually (N queries). For 100 orders with their items, that's 101 queries instead of 1-2.

Fixes:
- **Eager loading**: `.Include(o => o.Items)` -- generates a JOIN in a single query.
- **Explicit loading**: `context.Entry(order).Collection(o => o.Items).LoadAsync()` -- controlled, still extra queries.
- **Projection**: `Select` only the fields you need into a DTO -- avoids loading full entities.
- **Split queries**: `.AsSplitQuery()` -- separate queries per collection, avoids cartesian explosion.

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 3. What is the difference between Lazy Loading and Eager Loading in EF Core?

<details>
<summary>Reveal answer</summary>

- **Lazy Loading** -- related entities are loaded **on first access** via virtual navigation properties and a proxy. Convenient but dangerous: can cause N+1 silently and fails outside the DbContext scope.
- **Eager Loading** -- related entities are loaded **upfront** with `.Include()`. You get all data in one (or a few) queries, making behavior predictable.

**Best practice**: prefer eager loading or projections. If lazy loading is used, be very aware of where data access happens to avoid performance surprises.

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 4. How does the EF Core Change Tracker work, and when should you use `AsNoTracking()`?

<details>
<summary>Reveal answer</summary>

The **Change Tracker** keeps a snapshot of every entity queried from the database. On `SaveChangesAsync()`, it compares current values to the snapshot and generates `INSERT`/`UPDATE`/`DELETE` SQL.

Use **`AsNoTracking()`** for **read-only queries** -- it skips snapshot creation, reducing memory and improving performance. Since there's no tracking, you can't call `SaveChanges()` to persist modifications on those entities.

```csharp
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();
```

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 5. How do EF Core migrations work? What's the difference between `Add-Migration` and `Update-Database`?

<details>
<summary>Reveal answer</summary>

Migrations let you **version-control your database schema** alongside your code.

- **`Add-Migration <Name>`** -- compares the current model to the last snapshot and generates a C# migration file with `Up()` and `Down()` methods.
- **`Update-Database`** -- applies pending migrations to the database by executing the `Up()` methods in order.

In production, you typically generate a **SQL script** (`Script-Migration`) or use a migration bundle instead of running `Update-Database` directly for safer, reviewable deployments.

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 6. When would you choose Dapper over EF Core?

<details>
<summary>Reveal answer</summary>

Choose **Dapper** when:
- You need **raw performance** on read-heavy paths (Dapper maps results ~2-3x faster than EF Core with tracking).
- You already have complex, hand-tuned **SQL or stored procedures**.
- You want a **thin layer** without change tracking, migrations, or LINQ translation.

Choose **EF Core** when you benefit from change tracking, migrations, LINQ, and relationships. Many teams use **both together** -- EF Core for writes and complex domain logic, Dapper for high-performance reads.

Deep dive: [ORM vs Micro ORM vs ADO.NET](../09-data-access/01-orm-vs-microorm-vs-adonet.md)

</details>

---

### 7. What are database indexes and how do they improve query performance? What's the trade-off?

<details>
<summary>Reveal answer</summary>

An **index** is a sorted data structure (typically B-tree) that lets the database find rows without scanning the entire table.

**Benefits**: dramatically faster `WHERE`, `JOIN`, and `ORDER BY` on indexed columns.

**Trade-offs**:
- **Slower writes** -- every `INSERT`/`UPDATE`/`DELETE` must also update the index.
- **Storage overhead** -- indexes consume disk space.
- **Maintenance** -- fragmented indexes need periodic rebuilding.

**Rule of thumb**: index columns you frequently filter or join on; avoid indexing columns with low selectivity (e.g., boolean flags).

Deep dive: [Query Optimization](../09-data-access/04-query-optimization.md)

</details>

---

### 8. What is "parameter sniffing" and when does it become a problem?

<details>
<summary>Reveal answer</summary>

**Parameter sniffing** is when SQL Server creates an execution plan based on the **first parameter values** it sees and reuses that plan for all subsequent calls. This is usually good -- plan reuse is efficient.

It becomes a problem when the first values are **atypical** (e.g., a value that matches 1 row vs. another that matches 1 million rows). The cached plan is optimal for the first case but terrible for the second.

Mitigations: `OPTION (RECOMPILE)`, `OPTIMIZE FOR UNKNOWN`, query hints, or plan guides.

Deep dive: [Query Optimization](../09-data-access/04-query-optimization.md)

</details>

---

### 9. What does "sargable" mean, and why should you avoid wrapping columns in functions in a WHERE clause?

<details>
<summary>Reveal answer</summary>

**SARGable** = "Search ARGument able." A predicate is sargable when the database can use an **index seek** to evaluate it.

Wrapping a column in a function (e.g., `WHERE YEAR(CreatedDate) = 2025`) prevents the optimizer from using the index on `CreatedDate` because it must evaluate the function for every row (index scan).

**Fix**: rewrite to a range predicate:

```sql
-- Non-sargable
WHERE YEAR(CreatedDate) = 2025

-- Sargable
WHERE CreatedDate >= '2025-01-01' AND CreatedDate < '2026-01-01'
```

Deep dive: [Query Optimization](../09-data-access/04-query-optimization.md)

</details>

---

### 10. When would you choose a NoSQL database over a relational (SQL) database?

<details>
<summary>Reveal answer</summary>

| Choose SQL when | Choose NoSQL when |
|-----------------|-------------------|
| Data has clear relationships and needs referential integrity | Data is denormalized or document-shaped |
| Transactions with ACID guarantees are required | Horizontal scalability is a priority |
| Schema is well-known and stable | Schema evolves rapidly |
| Complex joins and aggregations | High write throughput, eventual consistency is OK |

Examples: **MongoDB** (documents), **Redis** (key-value/cache), **Cassandra** (wide-column, massive write throughput). Many systems use **both** -- SQL for transactional data, NoSQL for caching, logs, or search.

Deep dive: [Databases](../09-data-access/03-databases.md)

</details>

---

### 11. What is connection pooling and why is it important?

<details>
<summary>Reveal answer</summary>

**Connection pooling** reuses database connections instead of creating a new one for every request. Opening a connection is expensive (TCP handshake, authentication, TLS). The pool maintains a set of open connections that are checked out and returned.

In .NET, `SqlConnection` pooling is **on by default** (controlled by the connection string). Key settings: `Max Pool Size`, `Min Pool Size`, `Connection Lifetime`.

**Pitfall**: if you don't dispose connections (`using` / `await using`), they aren't returned to the pool, eventually exhausting it and causing timeouts.

Deep dive: [ORM vs Micro ORM vs ADO.NET](../09-data-access/01-orm-vs-microorm-vs-adonet.md)

</details>

---

### 12. What are the standard SQL transaction isolation levels, and what problems does each prevent?

<details>
<summary>Reveal answer</summary>

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
|----------------|-----------|-------------------|-------------|
| Read Uncommitted | Possible | Possible | Possible |
| **Read Committed** | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible |
| Serializable | Prevented | Prevented | Prevented |

- **Dirty read** -- reading uncommitted changes from another transaction.
- **Non-repeatable read** -- re-reading a row yields different values because another transaction modified it.
- **Phantom read** -- re-running a query returns new rows inserted by another transaction.

**Defaults vary by engine**: Read Committed is the default in SQL Server and PostgreSQL. **MySQL InnoDB defaults to Repeatable Read** (with gap locks that also prevent phantoms). Oracle defaults to Read Committed and has no Read Uncommitted level.

Higher isolation = more correctness but more locking and lower throughput.

Deep dive: [Transaction Isolation Levels](../09-data-access/06-transaction-isolation-levels.md)

</details>

---

### 13. What is the difference between a clustered and a non-clustered index?

<details>
<summary>Reveal answer</summary>

- **Clustered index** — defines the **physical order** of rows. The leaf pages of the index *are* the table rows. A table can have **only one**. The primary key becomes the clustered index by default.
- **Non-clustered index** — a separate structure whose leaf pages store the indexed columns and a **pointer to the row** (the clustered key, or the RID if the table is a heap). A table can have **many**.

When a non-clustered index doesn't contain all columns the query needs, a **key lookup** occurs — extra I/O to fetch missing columns from the clustered index. A **covering index** (using `INCLUDE`) eliminates this by storing the extra columns at the leaf level.

Deep dive: [SQL Indexes](../09-data-access/05-sql-indexes.md)

</details>

---

### 14. What's the difference between an Index Seek, an Index Scan, and a Table Scan?

<details>
<summary>Reveal answer</summary>

| Operator | Cost | What happens |
|----------|------|--------------|
| **Index Seek** | Low | B-tree navigation jumps straight to matching rows |
| **Index Scan** | Medium | Reads every leaf page of the index (predicate isn't usable or too many rows estimated) |
| **Table Scan** / **Clustered Index Scan** | High | Reads every row of the table |

Common causes of an unexpected scan despite an index existing:
- **Non-sargable predicate**: `WHERE YEAR(CreatedAt) = 2026` — function on the column disables the seek.
- **Implicit type conversion**: `varchar` column compared against `nvarchar` parameter.
- **Leading column not in the WHERE** for a composite index.
- **Stale statistics** causing bad cardinality estimates.
- `SELECT *` forcing many key lookups, making a scan look cheaper to the optimizer.

Deep dive: [SQL Indexes](../09-data-access/05-sql-indexes.md)

</details>

---

### 15. What is a covering index and when is it worth creating one?

<details>
<summary>Reveal answer</summary>

A **covering index** contains every column the query reads — either as index keys or via `INCLUDE`. The query is served entirely from the index, with no key lookup into the clustered index.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Covering
ON Orders(CustomerId)
INCLUDE (OrderDate, Total);

-- No key lookup needed
SELECT OrderDate, Total FROM Orders WHERE CustomerId = 42;
```

Create one when:
- A hot query runs frequently and shows a **key lookup** as a significant cost in the execution plan.
- The extra included columns aren't huge (covering indexes bloat the index size and slow writes).

Don't add `INCLUDE` blindly — monitor `sys.dm_db_index_usage_stats` and remove unused covering indexes.

Deep dive: [SQL Indexes](../09-data-access/05-sql-indexes.md)

</details>

---

### 16. What is Snapshot isolation and how is it different from Serializable?

<details>
<summary>Reveal answer</summary>

**Snapshot** is a **row-versioning** (MVCC) model, not a locking one. Each transaction sees a consistent picture of the database as of the moment it started. Writers don't block readers and vice versa; writer-vs-writer conflicts surface as **update conflict** errors (3960 in SQL Server).

**Serializable** uses **range locks** to make transactions behave as if they ran one at a time — preventing all anomalies (dirty, non-repeatable, phantoms) but increasing blocking and deadlock risk.

Use snapshot when readers are being blocked by writers and you can tolerate retries on update conflicts. Use serializable when you need strict "nothing new can appear" semantics in short, focused transactions.

Deep dive: [Transaction Isolation Levels](../09-data-access/06-transaction-isolation-levels.md)

</details>

---

### 17. What are the different lock granularities and modes in SQL Server?

<details>
<summary>Reveal answer</summary>

**Granularity** (smaller = more concurrency, more overhead):

`RID` (row in heap) → `KEY` (row in B-tree) → `PAGE` (8 KB) → `EXTENT` (8 pages) → `HoBT` → `TABLE` → `DATABASE`. Plus `XACT` for optimized locking (2022+).

**Modes** and what they mean:

| Mode | Purpose |
|---|---|
| **S** (Shared) | Reads |
| **U** (Update) | "I'll probably update" — blocks other U/X; prevents conversion deadlocks |
| **X** (Exclusive) | Writes; incompatible with everything |
| **IS / IX / SIX** | Intent locks at higher levels — signal lower-level S/X |
| **Sch-S / Sch-M** | Schema stability / modification (DDL) |
| **Range*** | Key-range locks for `SERIALIZABLE` to prevent phantoms |

**Key facts for an interview:**
- X is compatible with nothing.
- U+U is **not** compatible (that's the point — it prevents lost updates).
- Every query — even `NOLOCK` — takes `Sch-S`, which is why DDL can block everything.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 18. What is lock escalation and how do you handle it?

<details>
<summary>Reveal answer</summary>

SQL Server collapses many fine-grained locks into a single higher-level lock when a statement acquires roughly **5,000 locks on one table/partition**, or when lock memory exceeds internal thresholds (~24% of DB engine memory). It's re-checked every ~1,250 additional locks.

**Why it exists:** tracking millions of row locks is expensive.

**Why it hurts:** a big `UPDATE` that should affect 10k rows suddenly locks the whole table.

**Detection:** Extended Events (`lock_escalation`) or `sys.dm_tran_locks`.

**Mitigations, in order of preference:**
1. **Batch big DML** — delete/update a few hundred rows at a time in a loop.
2. **Enable `READ_COMMITTED_SNAPSHOT`** — readers stop taking S locks, shrinking the lock count.
3. **Enable optimized locking** (SQL Server 2022+ with ADR + RCSI) — keeps only a transaction-ID lock and releases row/page locks incrementally.
4. **Cover your predicates with indexes** — a scan locks far more rows than a seek.
5. **Last resort:** `ALTER TABLE ... SET (LOCK_ESCALATION = DISABLE)` on that table.

**PostgreSQL has no escalation** — different trade-off; you worry about tuple bloat instead.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 19. Why is `WITH (NOLOCK)` dangerous beyond just "dirty reads"?

<details>
<summary>Reveal answer</summary>

`NOLOCK` / `READUNCOMMITTED` disables shared locks, so scans can observe the table mid-mutation. It doesn't just allow "reading uncommitted data" — it can return:

- **Duplicate rows** — a row that moves due to a page split during your scan can be read twice.
- **Missing rows** — same mechanism, in reverse.
- **Inconsistent data** — row images from different points in time.

These can happen even when no transaction is rolled back, so retrying doesn't help. Can also surface **error 601**.

The correct modern answer to "we want non-blocking reads" is `ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON`. Readers use row versions instead of locks — same non-blocking behavior, zero correctness risk. A senior interviewer will score you down for defending `NOLOCK` as a performance trick.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 20. Pessimistic vs optimistic concurrency — how do you implement each?

<details>
<summary>Reveal answer</summary>

**Pessimistic** — lock the row at read time.

```sql
-- SQL Server
BEGIN TRAN;
SELECT Balance FROM Accounts WITH (UPDLOCK, ROWLOCK) WHERE Id = @id;
UPDATE Accounts SET Balance = Balance - @amt WHERE Id = @id;
COMMIT;
```

```sql
-- PostgreSQL
BEGIN;
SELECT balance FROM accounts WHERE id = $1 FOR UPDATE;
UPDATE accounts SET balance = balance - $2 WHERE id = $1;
COMMIT;
```

Best for short critical sections with expected contention and expensive retries.

**Optimistic** — no locks on read; detect conflicts on write via a version column.

```sql
CREATE TABLE Orders (
    Id int PRIMARY KEY,
    Total decimal(10,2),
    RowVersion rowversion NOT NULL
);

UPDATE Orders SET Total = @new
WHERE Id = @id AND RowVersion = @expected; -- @@ROWCOUNT = 0 means conflict
```

```csharp
// EF Core
public class Order
{
    [Timestamp] public byte[] RowVersion { get; set; } = default!;
}
// SaveChangesAsync throws DbUpdateConcurrencyException on conflict
```

Best for web apps with many readers, few conflicts, and easy retries.

**Rule of thumb:** start optimistic; switch to pessimistic for hotspots where conflicts dominate.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 21. How do you diagnose lock contention and deadlocks in production?

<details>
<summary>Reveal answer</summary>

**SQL Server:**

```sql
-- Current locks and who holds them
SELECT * FROM sys.dm_tran_locks;

-- Active blocking chains
SELECT blocking_session_id, session_id, wait_type, wait_resource
FROM sys.dm_os_waiting_tasks
WHERE blocking_session_id IS NOT NULL;

-- Server-wide top lock waits
SELECT wait_type, waiting_tasks_count, wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type LIKE 'LCK%'
ORDER BY wait_time_ms DESC;
```

Deadlocks (error 1205) are auto-captured in the `system_health` Extended Events session as an XML deadlock graph — pull it with `sys.dm_xe_session_targets`. Always **retry with backoff** in app code; deadlocks are transient.

**PostgreSQL:**

```sql
SELECT pid, relation::regclass, mode, granted FROM pg_locks;

-- Blocking tree
SELECT blocked.pid, blocking.pid, blocked.query, blocking.query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

Deadlocks in Postgres raise `SQLSTATE 40P01`; serialization failures under `SERIALIZABLE` raise `40001`. Both need retries.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 22. How do you prevent lost updates in a concurrent system?

<details>
<summary>Reveal answer</summary>

Higher isolation levels reduce lost updates but don't eliminate them across transactions. Two proven strategies:

- **Optimistic concurrency** (preferred): add a `RowVersion`/`timestamp` column; on update, include it in the `WHERE`. If 0 rows are affected, the row changed under you — retry or surface a conflict. EF Core throws `DbUpdateConcurrencyException`.
- **Pessimistic locking**: lock the row when reading (`SELECT ... WITH (UPDLOCK, ROWLOCK)`). Safe but causes blocking and is the biggest source of deadlocks — use sparingly.

Optimistic concurrency scales better in web apps; pessimistic is reserved for short critical sections where conflicts are expected and retries are expensive.

Deep dive: [Transaction Isolation Levels](../09-data-access/06-transaction-isolation-levels.md)

</details>

---

### 23. What is an execution plan and what is the difference between estimated and actual?

<details>
<summary>Reveal answer</summary>

An **execution plan** is the concrete strategy the optimizer chose for a query: which indexes, which join algorithms, which order, how to aggregate and sort.

- **Estimated plan**: compiled by the optimizer based on statistics; does **not** execute the query. Useful for a quick sanity check.
- **Actual plan**: the same plan plus runtime data — actual row counts, elapsed time, spill/warning flags. Only available *after* execution.

The single most important thing to compare is **estimated rows vs actual rows** per operator. A large gap means the optimizer was working with bad assumptions (stale statistics, parameter sniffing, non-sargable predicate) and almost always picked the wrong algorithm.

In SQL Server: `Ctrl+L` (estimated) / `Ctrl+M` (actual) in SSMS, or `SET SHOWPLAN_XML ON` / `SET STATISTICS XML ON` in T-SQL.
In PostgreSQL: `EXPLAIN` (estimated) / `EXPLAIN (ANALYZE, BUFFERS)` (actual — runs the query).
In MySQL: `EXPLAIN FORMAT=TREE` / `EXPLAIN ANALYZE`.

Deep dive: [Execution Plans](../09-data-access/08-execution-plans.md)

</details>

---

### 24. What red flags do you look for when reading a SQL Server execution plan?

<details>
<summary>Reveal answer</summary>

In order of "fix me first":

1. **Big Table/Clustered Index Scan** where a Seek was expected — non-sargable predicate, missing index, or implicit conversion.
2. **Estimated rows ≪ Actual rows** on any operator — broken cardinality estimate; the plan shape is probably wrong.
3. **Key Lookup / RID Lookup** on the hot path — the nonclustered index isn't covering. Fix with `INCLUDE` columns.
4. **Nested Loops with a huge outer input** — should probably be Hash Match. Usually a symptom of #2.
5. **Sort or Hash Match with a spill warning (yellow ⚠)** — operator wrote to `tempdb` because the memory grant was too small. Usually downstream of #2.
6. **`CONVERT_IMPLICIT` in the predicate** — data type mismatch kills index usage. Fix the parameter type at the app or column level.
7. **"Columns With No Statistics" warning** — optimizer is guessing. Create statistics or a covering index.

Green "missing index" hints are suggestions, not prescriptions — they ignore existing indexes and write cost.

Deep dive: [Execution Plans](../09-data-access/08-execution-plans.md)

</details>

---

### 25. A query is fast in SSMS but slow in the application. Why?

<details>
<summary>Reveal answer</summary>

Two usual suspects, both plan-related:

- **Parameter sniffing**: the app and SSMS cached different plans because the first-run parameters were very different. The app's cached plan is terrible for the current parameters. Fix with `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, or a plan guide — trade-offs in [Query Optimization](../09-data-access/04-query-optimization.md#parameter-sniffing-sql-server).
- **Different `SET` options**: the app's connection has different `ARITHABORT`, `ANSI_NULLS`, or `QUOTED_IDENTIFIER` from SSMS, so it produces a **separate plan cache entry**. Microsoft Learn explicitly warns: SSMS defaults to `ARITHABORT ON`, and *"client applications setting ARITHABORT to OFF might receive different query plans, making it difficult to troubleshoot poorly performing queries."* On modern databases (compat level 90+ with `ANSI_WARNINGS ON`), `ARITHABORT` is implicitly ON regardless of the client-side value — but if the app is on a legacy compat level or explicitly sets it OFF, plans diverge.

The fix starts the same way both times: capture the **actual** plan the app is using (Extended Events, Query Store), not the one SSMS gives you.

Deep dive: [Execution Plans](../09-data-access/08-execution-plans.md)

</details>

---

### 26. What's the difference between partitioning and sharding?

<details>
<summary>Reveal answer</summary>

- **Partitioning** splits a table into smaller pieces (by range, list, or hash) inside the **same database server** — SQL Server partitioned tables, PostgreSQL declarative partitioning, MySQL partitions. Helps with maintenance (archive, fast prune), parallel I/O, index locality — but does **not** remove the single-writer bottleneck.
- **Sharding** splits data across **multiple independent servers**, each holding a subset. Removes the single-primary write limit; adds enormous operational complexity (routing, rebalancing, cross-shard queries).

Partitioning first; shard only when a single primary truly cannot keep up and the other options (read replicas, better indexes, archiving) are exhausted.

Deep dive: [Sharding and Partitioning](../09-data-access/10-sharding-and-partitioning.md)

</details>

---

### 27. How do you choose a good shard key, and what makes a bad one?

<details>
<summary>Reveal answer</summary>

A good shard key has:

1. **High cardinality** — enough distinct values that data actually splits.
2. **Low frequency per value** — no single value dominates write load.
3. **Alignment with the dominant query pattern** — appears in `WHERE` so reads hit one shard.
4. **Immutability** — mutating a shard key moves the row across servers.

**Bad picks:** `created_at` alone (monotonic → hot shard), boolean flags (low cardinality), any column the app routinely updates.

**Good picks:** `tenant_id`, `user_id`, composite `(tenant_id, order_id)`. MongoDB docs: *"The choice of shard key affects the performance, efficiency, and scalability of a sharded cluster."*

Deep dive: [Sharding and Partitioning](../09-data-access/10-sharding-and-partitioning.md)

</details>

---

### 28. Range-based vs hash-based sharding — trade-offs?

<details>
<summary>Reveal answer</summary>

| | Range | Hash |
|---|---|---|
| Distribution | Adjacent keys co-located | Random |
| Range queries | Cheap (one shard) | Scatter-gather (all shards) |
| Hot shard on monotonic keys | **Yes** (last shard) | No |
| Rebalancing | Split ranges | `mod N` hostile; use **consistent hashing** |

Range is right when queries often read ranges (time series slices, sequential scans). Hash is right when writes are monotonic (auto-increment, timestamps) and range reads are rare. A **directory-based** strategy (metadata table mapping key → shard) is the flexible middle ground at the cost of an extra lookup.

Deep dive: [Sharding and Partitioning](../09-data-access/10-sharding-and-partitioning.md)

</details>

---

### 29. What's the difference between synchronous and asynchronous replication, and when do you pick which?

<details>
<summary>Reveal answer</summary>

- **Asynchronous** — primary commits locally, then ships changes. Low write latency; **non-zero data-loss window** on primary failure. PostgreSQL streaming replication is asynchronous by default (the docs warn: *"some transactions that were committed may not have been replicated to the standby server, causing data loss"*).
- **Synchronous** — primary waits for at least one replica to harden the log before committing. Zero data loss on failover; higher write latency. SQL Server AGs synchronous-commit mode works this way.

Pick **sync** for HA replicas on low-latency networks where losing a committed transaction is unacceptable. Pick **async** for DR replicas across regions or where write latency matters more than absolute durability. In SQL Server AGs you can mix: sync local + async remote.

Deep dive: [Replication and Read Replicas](../09-data-access/11-replication-and-read-replicas.md)

</details>

---

### 30. How do you handle "read-your-writes" consistency with read replicas?

<details>
<summary>Reveal answer</summary>

Replication lag means a user who just wrote may not see their own change on a read replica.

Patterns:

1. **Primary affinity for recent writes** — remember the last write timestamp per user; for N seconds (e.g., 5s) route that user's reads to the primary.
2. **Route critical flows to the primary** — checkout, profile edit. Let browsing and reports use replicas.
3. **Wait for replication position** — poll the replica's LSN/GTID until it has caught up to the user's last write.
4. **Synchronous replication** — pay the latency to eliminate the window entirely.

"Always read from the primary" defeats the purpose of replicas. "Always read from a replica" breaks your users. The right answer is a conscious split.

Deep dive: [Replication and Read Replicas](../09-data-access/11-replication-and-read-replicas.md)

</details>

---

### 31. What are the failover modes in SQL Server Always On Availability Groups?

<details>
<summary>Reveal answer</summary>

Three forms:

- **Automatic failover** — cluster manager detects primary failure and promotes a synchronized secondary. Requires sync-commit mode and failover mode set to `Automatic` on both replicas. No data loss.
- **Planned manual failover** — operator-initiated, sync mode required, no data loss. Used for patching.
- **Forced failover** — only option when no synchronized secondary is available. Microsoft Learn: *"Forced failover is a disaster recovery option. It's the only form of failover that's possible when the target secondary replica isn't synchronized with the primary replica."* **Data loss is possible.**

SQL Server 2019 raised the max synchronous replicas to 5 (primary + 4 sync secondaries). The **availability group listener** routes `ApplicationIntent=ReadOnly` clients to a readable secondary.

Deep dive: [Replication and Read Replicas](../09-data-access/11-replication-and-read-replicas.md)

</details>

---

### 32. What is Change Data Capture (CDC), and how does log-based CDC differ from trigger-based?

<details>
<summary>Reveal answer</summary>

**CDC** extracts a stream of row-level changes from a database so downstream consumers (caches, search indexes, warehouses) can stay in sync.

| | Log-based | Trigger-based | Query-based |
|---|---|---|---|
| Mechanism | Tail transaction log / WAL / binlog | `AFTER` triggers write to a shadow table | `SELECT WHERE updated_at > ?` |
| Source-DB impact | Minimal (log is already written) | Extra write per DML | Extra reads |
| Captures deletes | Yes | Yes | **No** |
| Intra-poll changes | Yes | Yes | Missed |

**Log-based is the modern default** — zero extra load on the write path, no triggers to maintain, captures operations the app layer might miss. SQL Server CDC, PostgreSQL logical decoding, MySQL binlog, and MongoDB change streams are all log-based. Debezium wraps all of these in a Kafka Connect integration.

Deep dive: [Change Data Capture](../09-data-access/12-change-data-capture.md)

</details>

---

### 33. What is Debezium, and what's the difference between CDC and the Transactional Outbox pattern?

<details>
<summary>Reveal answer</summary>

**Debezium** is an open-source CDC platform that runs as **Kafka Connect** source connectors. It supports MySQL/MariaDB, PostgreSQL, Oracle, SQL Server, and MongoDB as sources (plus others like Db2, Cassandra, Vitess, Spanner depending on release). Debezium also ships a **JDBC sink connector** for writing CDC events from Kafka into a relational target — it is a sink, not a source. Debezium runs a **snapshot** of the source tables, then tails the database log and emits `c`/`u`/`d`/`r` events (with `before`/`after` row images) into Kafka topics.

**CDC vs Outbox** — both solve "keep another store in sync with my DB":

| | CDC | Transactional Outbox |
|---|---|---|
| Event shape | Row-shaped (`table.column=value`) | Business-shaped (`OrderConfirmed`) |
| App knowledge | None; DB is the source | App writes the event in the same TX |
| Best for | Technical projections (cache invalidation, search index, warehouse), zero-downtime migrations | Business events consumed by other bounded contexts |

Use both: Outbox for business integration events, CDC for technical projections.

Deep dive: [Change Data Capture](../09-data-access/12-change-data-capture.md)

</details>

---

### 34. In PostgreSQL, what is a replication slot and why must you monitor it?

<details>
<summary>Reveal answer</summary>

A **replication slot** is a named stream position that Postgres uses to know "this consumer has acknowledged up to LSN X; I must retain WAL beyond X until it catches up." Logical decoding (used by Debezium and logical replication) creates a logical slot.

**Why it's dangerous:** if the consumer stops (Debezium crash, forgotten test cluster), Postgres keeps retaining WAL indefinitely. The pg_wal directory grows until the disk fills and the primary stops accepting writes.

**Monitor:**

```sql
SELECT slot_name, active, restart_lsn,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots;
```

Alert when `active = false` or `lag_bytes` exceeds a threshold, and drop abandoned slots with `pg_drop_replication_slot(...)`.

Deep dive: [Change Data Capture](../09-data-access/12-change-data-capture.md)

</details>

---

### 35. Walk through normalizing an orders table from 1NF to 3NF.

<details>
<summary>Reveal answer</summary>

Start with a flat `Orders(OrderId, CustomerId, CustomerName, CustomerCity, ProductId, ProductName, Qty)`:

- **1NF** — atomic values, no repeating groups. If `tags` is a comma-separated string, split it into a child table first.
- **2NF** — no partial dependency on a composite PK. If PK is `(OrderId, ProductId)`, `CustomerName` depends only on `OrderId`, not on `ProductId`. Split into `Orders(OrderId, CustomerId, ...)` and `OrderLines(OrderId, ProductId, Qty)`.
- **3NF** — no transitive dependencies. `CustomerName`/`CustomerCity` depend on `CustomerId`, not on `OrderId`. Extract `Customers(CustomerId, CustomerName, CustomerCity)`.

Denormalize afterwards *only* with a measured reason: duplicate a rarely-changing column into a hot table, or maintain a summary column (`TotalAmount`) to avoid recomputing from lines on every read.

Deep dive: [Data Modeling](../09-data-access/13-data-modeling.md)

</details>

---

### 36. In a document store like MongoDB, when do you embed vs reference?

<details>
<summary>Reveal answer</summary>

**Embed** when:

- Child data is read together with the parent almost always.
- Children are owned by the parent (no independent lifecycle).
- Cardinality is bounded (an order has ~5 lines, not 50,000).
- Total document size stays well under the limit (16 MB in MongoDB).

**Reference** when:

- The child is shared (a product referenced by many orders — embedding would duplicate it).
- The child has its own lifecycle and is queried independently.
- Cardinality is unbounded or very large.

If every read needs `$lookup` across collections, the data is in the wrong shape — embed or denormalize. Joins in document stores are slower and less optimized than in a relational engine.

Deep dive: [Data Modeling](../09-data-access/13-data-modeling.md)

</details>

---

### 37. What is DynamoDB single-table design, and why does AWS recommend it?

<details>
<summary>Reveal answer</summary>

**Single-table design** stores multiple entity types (users, orders, items, ...) in one table with generic `PK` and `SK` keys, then uses key prefixes and begins_with queries to fetch related data in one request.

```
PK            SK                  Attributes
USER#42       PROFILE             name=Ada
USER#42       ORDER#2024-04-16    total=120
ORDER#999     ITEM#p-1            qty=1, price=120
```

AWS explicit guidance: *"You should maintain as few tables as possible in a DynamoDB application. Having fewer tables keeps things more scalable, requires less permissions management, and reduces overhead for your DynamoDB application."*

**Why:** DynamoDB is query-first. You design access patterns up front; single-table lets one query (plus a few GSIs) cover them all. Multiple tables multiply round-trips and lose the pattern's benefit.

Deep dive: [Data Modeling](../09-data-access/13-data-modeling.md)

</details>

---

### 38. In Cassandra, what's the difference between a partition key and a clustering column, and why do they matter?

<details>
<summary>Reveal answer</summary>

Cassandra's primary key is composite: `(partition_key, clustering_column_1, clustering_column_2, ...)`.

- **Partition key** — mandatory; decides which node holds the row. Queries **must** include the partition key for efficient access.
- **Clustering columns** — optional; define the **order** of rows within a partition and enable range queries inside it.

**Query-first modeling:** design one table per top query. The same data is routinely duplicated across multiple tables, each with keys chosen for a specific access pattern.

**Pitfall:** unbounded partitions. A partition that grows forever (one partition per chat, kept for years) becomes a hotspot and breaks compaction. Widely cited community guidance is to keep partitions well under ~100 MB / ~100k rows — bucket by time if needed: `PRIMARY KEY ((chat_id, year_month), sent_at)`.

Deep dive: [Data Modeling](../09-data-access/13-data-modeling.md)

</details>

---
