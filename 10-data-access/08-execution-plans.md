# Execution Plans

An **execution plan** (a.k.a. **query plan**) is the recipe the database's query optimizer decides to use for running a SQL statement: which tables to read first, which indexes to use, how to join them, and how to aggregate/sort the output. It is the single most important artifact when debugging a slow query — row counts, costs, and operator choices tell you exactly *why* the engine is spending time where it is.

For the surrounding context, see [Query Optimization](04-query-optimization.md), [SQL Indexes](05-sql-indexes.md), and [Database Locks](07-database-locks.md).

## How the optimizer builds a plan

Every modern relational database uses a **cost-based optimizer (CBO)**. The flow:

1. **Parse** the SQL into a logical tree (relational algebra).
2. **Bind** — resolve object names, permissions, data types.
3. **Rewrite** — apply algebraic equivalences (predicate pushdown, subquery unnesting, join reordering).
4. **Plan / optimize** — enumerate physical alternatives (which index, which join algorithm, which order) and pick the one with the **lowest estimated cost** based on **statistics**.
5. **Execute** — run the chosen plan. Some engines also **cache** the plan for reuse.

> Microsoft Learn on SQL Server: *"The Query Optimizer chooses a query plan using a set of heuristics to balance compilation time and plan optimality in order to find a good query plan."* It does not search exhaustively — it searches until the expected compile cost exceeds the expected gain.

The plan quality depends entirely on **statistics** (histograms of column distributions, densities, cardinalities). Stale statistics = bad plans, even on a well-indexed table.

## Estimated vs actual plans

Every engine distinguishes between the plan the optimizer *thinks* will happen and what *actually* happened at runtime:

| Kind | When produced | What it shows | Executes the query? |
|---|---|---|---|
| **Estimated** (compile-time) | Before execution — just asks the optimizer to plan | Operators, estimated rows, estimated cost | No |
| **Actual** (runtime) | After execution | Estimated values **plus** actual row counts, actual time, actual loops, warnings | Yes |

The **gap between estimated and actual row counts** is the #1 signal of a broken plan. If the optimizer estimated 10 rows but 10 million actually came out of that operator, it picked the wrong algorithm (probably nested loops when it should have been a hash join). Root cause is usually stale statistics, parameter sniffing, or a non-sargable predicate.

## How to capture a plan — by engine

### SQL Server

**SSMS graphical plan:**

- **Query → Display Estimated Execution Plan** — estimated plan, no execution.
- **Query → Include Actual Execution Plan** — toggle on, then run the query; plan appears on the **Execution Plan** tab.
- **Query → Include Live Query Statistics** — in-flight progress, updated every second; useful when a query hangs.

**T-SQL (text/XML):**

```sql
-- Estimated plan only (does not execute).
SET SHOWPLAN_XML ON;
GO
SELECT * FROM Orders WHERE CustomerId = 42;
GO
SET SHOWPLAN_XML OFF;

-- Actual plan — runs the query and returns the plan XML alongside results.
SET STATISTICS XML ON;
GO
SELECT * FROM Orders WHERE CustomerId = 42;
GO
SET STATISTICS XML OFF;

-- Runtime metrics without the plan.
SET STATISTICS IO ON;     -- logical reads, physical reads, read-ahead reads
SET STATISTICS TIME ON;   -- CPU time, elapsed time
```

> Microsoft Learn: the estimated plan is *"the compiled plan as produced by the Query Optimizer, based on estimations. This is the query plan that is stored in the plan cache."* The actual plan *"returns the compiled plan plus its execution context... This plan includes actual runtime information such as execution warnings, and in newer versions of the Database Engine, the elapsed and CPU time used during execution."*

**Query Store** (SQL Server 2016+; enabled by default for new databases in **Azure SQL Database**, **Azure SQL Managed Instance**, and **SQL Server 2022+** in `READ_WRITE` mode) keeps plan history, runtime stats, wait stats, and supports **plan forcing** — query the `sys.query_store_*` catalog views or use the SSMS Query Store reports.

### PostgreSQL

```sql
-- Estimated plan only.
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Actual plan. ACTUALLY RUNS THE QUERY — wrap DML in a transaction you roll back.
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;

-- For INSERT/UPDATE/DELETE, side effects apply; always wrap:
BEGIN;
EXPLAIN (ANALYZE) UPDATE orders SET status = 'shipped' WHERE id = 1;
ROLLBACK;
```

Official docs warning: *"Keep in mind that because `EXPLAIN ANALYZE` actually runs the query, any side-effects will happen as usual, even though whatever results the query might output are discarded."*

Useful options: `ANALYZE`, `BUFFERS` (I/O counts — implicitly enabled with `ANALYZE` in current versions), `VERBOSE`, `SETTINGS`, `FORMAT TEXT|XML|JSON|YAML`.

### MySQL (8.x)

```sql
-- Traditional tabular output (default).
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Tree output — the only format that shows hash joins explicitly.
EXPLAIN FORMAT=TREE SELECT * FROM orders WHERE customer_id = 42;

-- Actual plan with timing, iterator-based. Always uses TREE format.
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

MySQL docs: `EXPLAIN ANALYZE` *"runs a statement and produces EXPLAIN output along with timing and additional, iterator-based, information about how the optimizer's expectations matched the actual execution."*

## Reading a plan — SQL Server

Plans read **right-to-left, top-to-bottom** (data flows from leaves to root). Arrows between operators are **thicker when more rows flow through them** — a fat arrow pointing into a Nested Loops join is almost always a problem.

### Scan vs Seek — the first thing to check

| Operator | What it does | When it's chosen | Health |
|---|---|---|---|
| **Table Scan** | Reads every row of a heap | No usable index, or most of the table matches | Fine for small tables, disaster for large ones |
| **Clustered Index Scan** | Reads every row of the clustered table | Same as above on a clustered table | Fine for small/analytic, red flag for OLTP lookups |
| **Clustered Index Seek** | Navigates the B-tree to the matching keys | Sargable predicate on the clustered key | Ideal for OLTP lookups |
| **Nonclustered Index Seek** | Seeks into a secondary index | Sargable predicate on an indexed column | Ideal — often followed by a Key Lookup |
| **Nonclustered Index Scan** | Reads the entire secondary index | Predicate not matching the index key order | Still better than table scan if it's covering |

> **Seek is not automatically better than Scan.** Scanning a small covering index can beat seeking + lookup. The cost model decides.

### Join operators

| Operator | Good when | Bad when |
|---|---|---|
| **Nested Loops** | Outer side has few rows, inner side has an index on the join key | Outer side turns out to be huge — inner side gets scanned N times |
| **Hash Match** | Large unsorted inputs; no useful index on the join key | Hash table doesn't fit in memory → spills to `tempdb` |
| **Merge Join** | Both inputs already sorted on the join key (clustered indexes on both sides) | Inputs not sorted — the optimizer has to add Sort operators |

A classic bad plan: the optimizer estimated 10 rows on the outer side (so picked Nested Loops) but 10 million actually came out → the inner seek runs 10 million times.

### Operators that scream "fix me"

- **Key Lookup** (on a clustered table) / **RID Lookup** (on a heap) — a nonclustered index matched the predicate but the query needs columns it doesn't cover. Each matching row triggers a separate clustered-index/heap lookup. Fix: add the missing columns as `INCLUDE` in the nonclustered index to make it **covering**.
- **Sort** — expensive and blocking. If the query has `ORDER BY`, consider an index that delivers the rows already sorted.
- **Hash Match (Aggregate)** or **Sort** with **spill warnings** — memory grant was too small and the operator wrote intermediate data to `tempdb`. Shows up as a yellow `⚠` on the operator. Usually driven by bad cardinality estimates.
- **Convert_Implicit** in the predicate (`CONVERT_IMPLICIT(varchar, column, 0) = @param`) — data-type mismatch. SQL Server wraps the column in a conversion, which kills index usage (non-sargable). Fix the parameter type at the app layer or change the column type. The plan XML shows it as a warning on the operator.
- **Missing Index** green hint — a hint, not a prescription. It tells you *a* useful index exists; it does **not** consider existing indexes, write cost, or overlap with what you already have. Validate manually.
- **Columns With No Statistics** warning — the optimizer is flying blind on that column. Create statistics (`CREATE STATISTICS ...`) or an index that covers it.

## Reading a plan — PostgreSQL

PostgreSQL `EXPLAIN` output reads **top-down** (root first) with indentation for children. Each node shows `(cost=startup..total rows=N width=B)` and, with `ANALYZE`, `(actual time=first..total rows=X loops=L)`.

```
Nested Loop  (cost=4.65..118.50 rows=10 width=488) (actual time=0.017..0.051 rows=10 loops=1)
  ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.90 rows=1 width=244)
       (actual time=0.003..0.003 rows=1 loops=10)
```

Key interpretation rules:

- **`cost`** is in arbitrary units (`seq_page_cost = 1.0` by convention), not milliseconds. Don't compare cost to actual time.
- **`loops > 1`** means the node ran N times. `actual time` and `rows` are **per-execution averages** — multiply by `loops` for total time.
- **`rows` (estimate) vs `actual rows`** — the cardinality gap. If estimate = 1 and actual = 10,000, the planner was wrong and likely picked the wrong algorithm.
- **Scan types**: `Seq Scan` (whole table), `Index Scan` (via index), `Index Only Scan` (returns from index alone — no heap fetch, needs the visibility map to be up to date), `Bitmap Heap Scan` (index builds a bitmap, then heap is read in physical order — good for medium selectivity).
- **Join strategies**: `Nested Loop`, `Hash Join`, `Merge Join` — same semantics as SQL Server.

`BUFFERS` adds `Buffers: shared hit=N read=M` — cache hits vs disk reads. A plan that looks fast in dev but slow in prod is often an all-cache-hit plan in dev and a mostly-disk-read plan in prod.

## Reading a plan — MySQL

`EXPLAIN` returns a table whose most important columns are:

| Column | What it tells you |
|---|---|
| `type` | Access method — the single most important column |
| `key` | Which index was actually chosen (`NULL` = none) |
| `possible_keys` | Indexes the optimizer considered |
| `rows` | Estimated rows examined |
| `filtered` | Percentage of rows kept after the `WHERE` |
| `Extra` | `Using index` (covering), `Using filesort` (extra sort), `Using temporary` (temp table) — the last two are usually red flags |

`type` from best to worst (per MySQL docs): `system` → `const` → `eq_ref` → `ref` → `range` → `index` → `ALL`. `ALL` is a full table scan.

For non-trivial queries use `EXPLAIN FORMAT=TREE` or `EXPLAIN ANALYZE` — the tabular format hides how operators feed each other, and only TREE/ANALYZE format shows **hash joins** explicitly.

## The analysis checklist

When a query is slow, walk the plan with these questions in order:

1. **Is there a giant Scan where you expected a Seek?** → check the predicate (sargable?), the index (exists? matches the filter order?), and stats.
2. **Does any operator show Estimated rows ≪ Actual rows?** → stale statistics, bad parameter sniffing, or a misestimated predicate. See [Query Optimization § Parameter Sniffing](04-query-optimization.md#parameter-sniffing-sql-server).
3. **Is the join algorithm reasonable for the data volume?** → Nested Loops with a huge outer side = rewrite or hint.
4. **Is there a Key Lookup / RID Lookup on the hot path?** → make the index covering.
5. **Any spill / tempdb / memory grant warning?** → fix cardinality estimates first; raising memory grants is a band-aid.
6. **Any implicit conversion in the predicate?** → fix the data type at the app or column level.
7. **Is the plan reused from cache with a bad shape for this parameter?** → parameter sniffing; consider `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, or plan guides.

## Statistics — the foundation

No optimizer produces good plans without good statistics. The planner relies on:

- **Histograms** — per-column distribution of values.
- **Densities** — uniqueness estimates for combinations of columns.
- **Cardinality** — row counts per table/partition.

| Engine | Auto-update trigger | Manual refresh |
|---|---|---|
| SQL Server (2014 and earlier, or compat level <130) | `500 + (0.20 * n)` row modifications for tables with `n > 500` rows | `UPDATE STATISTICS <table>` / `sp_updatestats` |
| SQL Server (2016+ with compat level 130+) | `MIN(500 + 0.20*n, SQRT(1000*n))` — a dynamic threshold that triggers updates more often on large tables | same as above |
| PostgreSQL | Background autovacuum runs `ANALYZE` when enough rows change | `ANALYZE <table>` / `VACUUM ANALYZE` |
| MySQL (InnoDB) | `innodb_stats_auto_recalc` ON by default; recalculates when ~10% of rows change | `ANALYZE TABLE <table>` |

If you just bulk-loaded a table and queries are slow, **update statistics first** — it's a 10-second command that usually beats any indexing change.

## Tools beyond the built-ins

- **SentryOne / SolarWinds Plan Explorer** (free, Windows) — much better SQL Server plan visualization than SSMS. Highlights expensive operators, shows deadlocked sections of the tree, inline stats.
- **pgMustard**, **explain.depesz.com**, **explain.dalibo.com** — PostgreSQL plan visualizers that highlight bad cardinality estimates and slow nodes.
- **MySQL Workbench Visual Explain** — graphical view of the traditional `EXPLAIN` output.

## Pitfalls

- **Estimated costs are unitless.** Don't compare "cost 1200" to "cost 600" across different servers or versions — the cost model is relative to hardware-independent constants.
- **The plan you see in SSMS may not be the plan that ran in production.** Different session-level `SET` options (`ARITHABORT`, `ANSI_NULLS`, `QUOTED_IDENTIFIER`) produce separate cache entries. This is the root cause of *"fast in SSMS, slow in app"*.
- **`EXPLAIN ANALYZE` in PostgreSQL and MySQL actually executes the query.** On DML, always wrap in `BEGIN ... ROLLBACK` (Postgres). On a huge `SELECT`, you'll pay the real cost.
- **Green "missing index" hints in SSMS are not reviewed proposals.** They ignore existing indexes and write cost. A table with 14 indexes because a developer accepted every hint is common — and slow on writes.
- **A plan that changed overnight without code changes** is almost always **parameter sniffing**, a statistics update, or auto-plan regression. Use **Query Store** on SQL Server to diff and force a known-good plan.
- **`SET NOCOUNT ON`** does not change the plan but it changes the wire traffic — unrelated, but frequently confused with plan-level settings.

## Rule of thumb

> Every performance investigation starts with the actual plan and the statistics. If you change code, indexes, or hints **without looking at the plan first**, you are guessing — and guessing is how you end up with 14 overlapping indexes and the same slow query.

---

[← Previous: Database Locks](07-database-locks.md) | [Back to index](README.md) | [Next: Polyglot Persistence →](09-polyglot-persistence.md)
