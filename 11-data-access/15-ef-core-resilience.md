# EF Core Resilience

EF Core sits on top of an ADO.NET provider, so everything in [Database Resilience](14-database-resilience.md) still applies — pooling, timeouts, transient error codes. What EF Core adds is its own machinery for **retrying units of work**, **detecting lost updates**, and **probing the database** — and a few sharp edges that only exist because the change tracker holds state across retries.

## The building blocks

| Failure mode | EF Core feature |
|---|---|
| Transient fault (network blip, failover, deadlock victim) | **Execution strategy** — `EnableRetryOnFailure()` |
| Connection dropped *during commit* — outcome unknown | `ExecuteInTransactionAsync(..., verifySucceeded)` or idempotent design |
| Two users editing the same row (lost update) | **Concurrency tokens** + `DbUpdateConcurrencyException` |
| Slow query holding a connection | `CommandTimeout` / `SetCommandTimeout` |
| Load balancer / orchestrator needs to know if the DB is reachable | `AddDbContextCheck<TContext>()` |

## Enabling retry on failure

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sql =>
        sql.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(10),
            errorNumbersToAdd: null))); // extra SQL Server error numbers to treat as transient
```

- **Defaults** (parameterless overload): *"Default values of 6 for the maximum retry count and 30 seconds for the maximum default delay are used."*
- The delay grows **exponentially with a small random factor**, capped at `maxRetryDelay` (this comes from the EF Core source code; the docs do not describe it).
- When all attempts fail, EF throws **`RetryLimitExceededException`**. The last transient error is its `InnerException`.

### Which errors are retried

The docs only say the SQL Server strategy is *"pre-configured with error numbers for transient errors that can be retried"* (including Azure SQL). The source (`SqlServerTransientExceptionDetector`, an internal API that can change) shows the details:

- **Retried:** deadlock victim (1205), Azure SQL "database unavailable" and throttling errors (40613, 40501, 40197, ...), network errors, and any `TimeoutException` that is not a `SqlException`.
- **Not retried: a SqlClient command timeout (`SqlException.Number == -2`).** The source explains: *"This exception can be thrown even if the operation completed successfully, so it's safer to let the application fail."*

> **Interview point:** "Does `EnableRetryOnFailure` retry a query that timed out?" — No. A command timeout is ambiguous (the write may have committed), so EF lets it fail. Fix slow queries; don't retry them.

### Azure SQL: retry is on by default with `UseAzureSql`

From the SQL Server provider docs (EF Core 9+): *"When using UseAzureSql and UseAzureSynapse, connection resiliency is automatically set up with the appropriate settings specific for those databases. Otherwise, when using UseSqlServer, configure the provider with EnableRetryOnFailure."*

```csharp
// EF Core 9+: retrying strategy configured automatically for Azure SQL
options.UseAzureSql(connectionString);

// UseSqlServer: no retry unless you ask for it
options.UseSqlServer(connectionString, sql => sql.EnableRetryOnFailure());
```

> **History worth knowing:** EF Core 8 RC2 briefly enabled retry automatically when it *detected* Azure SQL from the connection string. It broke existing `BeginTransaction` code and caused memory build-up on streamed queries (dotnet/efcore #32165, #32052), so it was reverted before GA. EF 9's explicit `UseAzureSql` replaced it. The lesson: **turning on retry changes behavior**, which is the subject of the rest of this file.

### PostgreSQL (Npgsql)

```csharp
options.UseNpgsql(connectionString, npgsql =>
    npgsql.EnableRetryOnFailure(
        maxRetryCount: 5,
        maxRetryDelay: TimeSpan.FromSeconds(10),
        errorCodesToAdd: null)); // SQLSTATE strings, e.g. "57P01", not ints
```

The Npgsql docs describe it as *"a retrying execution strategy, which will attempt to detect most transient PostgreSQL/network errors"* and note that it *"relies on the IsTransient property of NpgsqlException."* It inherits the same 6 / 30 s defaults from the EF Core base class.

## The hidden cost: buffering

From the EF Core docs: *"Enabling retry on failure causes EF to internally buffer the resultset, which may significantly increase memory requirements for queries returning large resultsets."*

The efficient-querying docs explain why: buffering ensures *"the same results are returned if the query is retried later"*, and if you call `ToList()` with a retrying strategy, *"the resultset is loaded into memory twice"*.

> **Consequence:** with retry enabled, `AsAsyncEnumerable()` over a million rows no longer streams. For large exports, use a separate context without a retrying strategy, page the query, or go to raw ADO.NET/Dapper.

## Custom execution strategy

Most of the time `errorNumbersToAdd` is enough. When you need full control, derive from `ExecutionStrategy` and override `ShouldRetryOn`:

```csharp
public sealed class DeadlockOnlyStrategy : ExecutionStrategy
{
    public DeadlockOnlyStrategy(ExecutionStrategyDependencies dependencies)
        : base(dependencies, maxRetryCount: 3, maxRetryDelay: TimeSpan.FromSeconds(2)) { }

    protected override bool ShouldRetryOn(Exception exception)
        => exception is SqlException { Number: 1205 }; // deadlock victim only
}

// Registration — the lambda receives ExecutionStrategyDependencies, not the DbContext
options.UseSqlServer(connectionString, sql =>
    sql.ExecutionStrategy(dependencies => new DeadlockOnlyStrategy(dependencies)));
```

## User-initiated transactions: the exception everyone hits

With a retrying strategy, this code throws. Not on `BeginTransactionAsync`, but on the next operation that goes through the strategy:

```csharp
await using var transaction = await db.Database.BeginTransactionAsync();
db.Orders.Add(order);
await db.SaveChangesAsync(); // throws InvalidOperationException here
```

> *"The configured execution strategy 'SqlServerRetryingExecutionStrategy' does not support user-initiated transactions. Use the execution strategy returned by 'DbContext.Database.CreateExecutionStrategy()' to execute all the operations in the transaction as a retriable unit."*

The reason: EF can retry a single `SaveChanges` on its own, but it cannot replay *your* sequence of operations. You must hand it the **whole unit of work** as a delegate:

```csharp
var strategy = db.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async () =>
{
    // Fresh context per attempt: the delegate runs again from the top on a transient failure
    await using var context = await contextFactory.CreateDbContextAsync();
    await using var transaction = await context.Database.BeginTransactionAsync();

    context.Orders.Add(order);
    await context.SaveChangesAsync();

    context.Payments.Add(payment);
    await context.SaveChangesAsync();

    await transaction.CommitAsync();
});
```

The docs: *"If a transient failure occurs, the execution strategy will invoke the delegate again."* The official sample creates the context **inside** the delegate. Reusing an outer context is risky. After the first `SaveChangesAsync` succeeds, the change tracker marks `order` as `Unchanged`. A retry after a rollback then would not re-insert it.

Rules that follow:

- **Everything the unit needs goes inside the delegate**, including `TransactionScope` (*"This approach can also be used with ambient transactions"*). A `TransactionScope` opened *outside* `ExecuteAsync` throws the same error. A maintainer explains: *"All transactions need to be scoped inside the retryable unit"* (dotnet/efcore #34825).
- **Nested strategies are a no-op.** A strategy called inside an existing user transaction does nothing (#34214). Per the API docs, *"All nested calls will be handled by the outermost strategy."*
- **The exception is a coding bug, not a transient fault.** Don't catch it and retry. A maintainer: *"the exception does not indicate any database error, but rather a coding error"* (#29396).

## Commit failure: the idempotency problem

The worst case for any retry: the connection drops **while `COMMIT` is in flight**. The docs: *"the resulting state of the transaction is unknown"*. A blind retry *"could lead to data corruption … for example when inserting a new row with auto-generated key values"*. The insert ran once, the retry runs it again, and now you have two orders.

The docs list four options:

| Option | Idea | Trade-off |
|---|---|---|
| **1. Do (almost) nothing** | *"Avoid using store-generated keys"*. Use client-generated GUIDs so a duplicate insert fails on the PK instead of creating a second row. | Simplest. Needs a unique key you control. |
| **2. Rebuild application state** | Discard the context, reload from the DB, tell the user. | Fine for interactive UIs. |
| **3. Add state verification** | `ExecuteInTransactionAsync` with a `verifySucceeded` callback. | Precise. Needs a cheap "did it happen?" query. |
| **4. Manually track the transaction** | Insert a row in a tracking table inside the transaction. On commit failure, check if it exists. | Generic. Extra table plus cleanup. |

Option 3, adapted from the docs:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
db.Orders.Add(order); // order.Id is a client-generated Guid

await strategy.ExecuteInTransactionAsync(
    db,
    operation: (context, ct) => context.SaveChangesAsync(acceptAllChangesOnSuccess: false, ct),
    verifySucceeded: (context, ct) => context.Orders.AsNoTracking().AnyAsync(o => o.Id == order.Id, ct));

db.ChangeTracker.AcceptAllChanges(); // only after the commit is confirmed
```

`acceptAllChangesOnSuccess: false` keeps the entity in the `Added` state, so the operation can be retried if the commit fails. You accept the changes yourself only after success. `verifySucceeded` runs only when a transient error happens **during commit**.

> This is the same idea as an HTTP **idempotency key**. See [Idempotency](../19-distributed-systems/03-idempotency.md).

## Optimistic concurrency: lost updates are not transient

A retrying strategy handles *infrastructure* failures. Two users saving the same row is a *business* conflict. It surfaces as `DbUpdateConcurrencyException`, which no execution strategy retries for you.

```csharp
public class Product
{
    public int Id { get; set; }
    public decimal Price { get; set; }

    [Timestamp] // SQL Server rowversion — the database bumps it on every update
    public byte[] Version { get; set; } = default!;
}
```

For an app-managed token (`[ConcurrencyCheck]` / `.IsConcurrencyToken()`, e.g. a `Guid`), the docs warn *"you must assign it in application whenever persisting changes"*.

When the `UPDATE ... WHERE Id = @id AND Version = @originalVersion` affects zero rows, *"SaveChanges() throws a DbUpdateConcurrencyException"*. That happens on updates and deletes; it is *"generally never thrown when adding entities"*.

```csharp
try
{
    await db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        var databaseValues = await entry.GetDatabaseValuesAsync();
        if (databaseValues is null)
        {
            // Row was deleted by someone else — the docs' sample doesn't check this
            throw new ConflictException("The record no longer exists.");
        }

        // Pick a policy: store wins (reload), client wins (refresh originals), or merge.
        // Client wins:
        entry.OriginalValues.SetValues(databaseValues);
    }
    await db.SaveChangesAsync();
}
```

In an API, the most common policy is none of the three. Return **409 Conflict** and let the client re-read. More on pessimistic vs optimistic locking in [Database Locks](07-database-locks.md).

### `ExecuteUpdate` / `ExecuteDelete` skip concurrency checks

*"Since ExecuteUpdate and ExecuteDelete do not interact with the change tracker, they cannot automatically apply concurrency control."* They also *"do not implicitly start a transaction"*. Apply the token yourself:

```csharp
var updated = await db.Products
    .Where(p => p.Id == id && p.Version == originalVersion)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Price, newPrice));

if (updated == 0) return Results.Conflict();
```

## Command timeout

```csharp
// For every command issued by this context type (seconds)
options.UseSqlServer(connectionString, sql => sql.CommandTimeout(15));

// For one context instance — e.g. a long-running report
db.Database.SetCommandTimeout(TimeSpan.FromMinutes(2));
```

*"If this value is not set, then the default value used is defined by the underlying ADO.NET data provider."* See [Database Resilience](14-database-resilience.md) for the provider defaults. Remember: the SQL Server strategy does **not** retry command timeouts.

## DbContext pooling is not connection pooling

| | `AddDbContextPool` | ADO.NET connection pool |
|---|---|---|
| What is pooled | `DbContext` instances (avoids setup cost) | Physical database connections |
| Default size | 1024 (since EF Core 6) | Provider setting (e.g. `Max Pool Size`) |
| When exhausted | *"EF falls back to the non-pooling behavior"* | Callers block, then time out |
| Resilience impact | None directly — it's a performance feature | Exhaustion is a classic outage cause |

Pooling gotchas: *"the context's OnConfiguring is only invoked once"*, and pooling *"generally does not reset state in the underlying database driver"*.

## Health check

```csharp
// Package: Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);
```

*"By default: The DbContextHealthCheck calls EF Core's CanConnectAsync method."* Pass `customTestQuery` to run a real query instead. Tag it as a **readiness** check, not liveness. A database outage should take the pod out of rotation, not restart it. See [API Resilience](../10-aspnet-core/04-api-resilience.md).

## When NOT to turn on retry (or what to change first)

- **Large result sets you stream.** Retry forces buffering.
- **Code that opens transactions outside a strategy.** It throws until you refactor every call site.
- **Inserts with store-generated keys and no verification.** Commit failure plus retry gives duplicates.
- **Retries at several layers.** EF retry 6×, wrapped in a Polly retry 3×, behind an HTTP client retry 3× = up to 54 database attempts for one user click. Pick one layer. See [Retries, Backoff and Jitter](../18-reliability-and-sre/01-retries-backoff-jitter.md).
- **Long transactions.** The whole delegate re-runs. Keep units of work short.

---

[← Previous: Database Resilience](14-database-resilience.md) | [Back to index](README.md)
