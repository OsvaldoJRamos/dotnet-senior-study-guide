# Database Resilience

The database is the dependency you can least afford to treat naively. Connections are scarce, state survives failures, and a retried write can **duplicate data**. This file covers resilience at the driver level, independent of any ORM: ADO.NET, `Microsoft.Data.SqlClient`, Npgsql, Dapper. EF Core-specific behavior (execution strategies, concurrency tokens) is in [EF Core Resilience](15-ef-core-resilience.md).

## Classify the failure first

Not every database exception deserves a retry. The Azure Architecture Center is explicit: *"Retry tasks only when the faults are transient."*

| Failure | What you see in .NET | Transient? | Response |
|---|---|---|---|
| Network blip, failover, Azure SQL reconfiguration | `SqlException` with a transient number (e.g. 40197, 40613, 4060); `NpgsqlException.IsTransient == true` | Yes | Retry with backoff on a **new** connection |
| Deadlock victim | SQL Server `1205`; PostgreSQL `40P01` | Yes | Retry the **whole transaction** |
| Serialization failure | PostgreSQL `40001` | Yes | Retry the whole transaction |
| Throttling / resource limits | Azure SQL `40501`, `10928`/`10929`; PostgreSQL `53300` too_many_connections | Yes, slowly | Back off longer; reduce concurrency |
| Pool exhausted | `InvalidOperationException`: *"Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool…"* | No (it's your app) | Fix leaks and slow queries. Retrying adds pressure. |
| Command timeout | `SqlException.Number == -2` | **Ambiguous** | Don't auto-retry writes. The statement may have committed. |
| Constraint violation, syntax error, permission denied | `SqlException` / `PostgresException` | No | Fail fast |

> **Interview point:** a timeout is not a transient fault you can blindly retry. The SqlClient docs: *"A client-side timeout surfaces as error `-2`, which isn't in the built-in transient error list."* The write might have succeeded on the server after your client gave up.

## Timeouts and pool defaults

| Setting | SqlClient | Npgsql |
|---|---|---|
| Connect timeout | `Connect Timeout` = **15 s** | `Timeout` = **15 s** |
| Command timeout | `SqlCommand.CommandTimeout` = **30 s** (`0` = no limit) | `Command Timeout` = **30 s** (`0` = infinite) |
| Max pool size | `Max Pool Size` = **100** | `Maximum Pool Size` = **100** |
| Min pool size | `Min Pool Size` = **0** | `Minimum Pool Size` = **0** |
| Idle connection pruning | — | `Connection Idle Lifetime` = 300 s |
| Max connection age | — | `Connection Lifetime` = 3600 s (since Npgsql 9.0; disabled before) |

Apply the [cascading timeout rule](../18-reliability-and-sre/04-timeouts.md). A 30 s command timeout behind an API with a 10 s request deadline means the database keeps working after the user is gone.

## Connection pool exhaustion

The classic database outage is usually not "the database is down". More often the **app ran out of pooled connections**. When all connections are in use, *"later opens wait… If the wait exceeds `Connect Timeout`, the open fails"* with:

> *"Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool. This may have occurred because all pooled connections were in use and max pool size was reached."*

The SqlClient docs list the usual suspects: *"a connection leak, slow queries, blocked transactions, too much concurrency, pool fragmentation, or a database capacity limit."* Their checklist starts with *"Every connection and reader is disposed on every path."*

```csharp
// Leak: if QueryAsync throws, the connection never returns to the pool
var connection = new SqlConnection(connectionString);
await connection.OpenAsync();
var orders = await connection.QueryAsync<Order>(sql);
```

```csharp
// Correct: disposal on every path returns the connection to the pool
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync(ct);
var orders = await connection.QueryAsync<Order>(sql);
```

> **Don't fix exhaustion by raising `Max Pool Size`.** If queries are slow, more connections just move the queue into the database. Fix the slow query or the leak. Also check the math: pod count × max pool size must stay below the server's connection limit.

> **Blocking period gotcha:** *"A retry provider that opens the same pool during a blocking period receives the cached exception."* Retrying `Open` immediately after a failure may not reach the server at all.

## SQL Server: idle connection resiliency

`ConnectRetryCount` / `ConnectRetryInterval` are connection-string keywords that re-establish a **broken idle connection** transparently.

- `ConnectRetryCount`: 0–255. *"Since version 5.x the default value for non Azure endpoints is 1. For Azure SQL endpoints, the default is 2. For Azure SQL serverless or on demand endpoints, the default is 5."* (The Azure SQL troubleshooting page still says the default is 1. Trust the SqlClient reference.)
- `ConnectRetryInterval`: 1–60 s, default 10.

What it does **not** do: *"If a transient error occurs while your query is being executed, your SqlConnection object doesn't retry the connect operation. It certainly doesn't retry your query."*

## SQL Server: configurable retry logic (SqlClient 3.0+)

`Microsoft.Data.SqlClient` can retry `Open` and command execution itself:

```csharp
var options = new SqlRetryLogicOption
{
    NumberOfTries = 4,                          // TOTAL attempts, including the first (1–60)
    DeltaTime = TimeSpan.FromSeconds(1),
    MaxTimeInterval = TimeSpan.FromSeconds(20), // caps EACH delay, not the total
    // TransientErrors = null → built-in baseline list (1205, 4060, 40197, 40613, ...)
    AuthorizedSqlCondition = commandText =>     // only retry reads
        commandText.TrimStart().StartsWith("SELECT", StringComparison.OrdinalIgnoreCase)
};
var retryProvider = SqlConfigurableRetryFactory.CreateExponentialRetryProvider(options);

await using var connection = new SqlConnection(connectionString) { RetryLogicProvider = retryProvider };
await connection.OpenAsync(ct);

// The connection's provider does NOT flow to commands — set it on each command
await using var command = new SqlCommand("SELECT Id, Total FROM Orders WHERE CustomerId = @id", connection)
{
    RetryLogicProvider = retryProvider
};
```

Facts that trip people up:

- **Off by default.** *"The default is SqlConfigurableRetryFactory.CreateNoneRetryProvider, which doesn't retry."* Also: *"assigning a provider to a connection doesn't apply it to commands created from that connection."*
- **`TransientErrors` replaces, it doesn't append.** When `null`, the providers use 20 baseline numbers: 233, 997, 1204, 1205, 1222, 4060, 4221, 10060, 10928, 10929, 40143, 40197, 40501, 40540, 40613, 42108, 42109, 49918, 49919, 49920. The baseline is exposed as `SqlConfigurableRetryFactory.BaselineTransientErrors` since SqlClient 7.0.
- **Jitter is built in.** *"The fixed, incremental, and exponential providers add random jitter to each interval."*
- **Transactions are skipped.** *"The built-in providers skip retry when a command runs inside an ambient TransactionScope or has a SqlTransaction attached."* The guidance is to *"roll back and retry the entire transaction as one unit."* A bug could still retry a command outside its transaction after a deadlock (dotnet/SqlClient #4309, fix merged September 2026). Check your version's release notes before relying on this.
- **Idempotency is on you.** *"Restrict command retries to idempotent operations."* `AuthorizedSqlCondition` is the hook. When it returns `false`, the command runs once without retry.
- **Old blog posts are stale.** The `Switch.Microsoft.Data.SqlClient.EnableRetryLogic` AppContext switch was a 3.0 preview requirement and was removed in 4.0.

## Azure SQL specifics

The Azure SQL docs list these as transient: **926, 4060, 4221, 615, 40197, 40501, 40613, 49918, 49919, 49920**. Errors 40540 and 40143 are *"Subcodes embedded in a 40197 failover message"*. Treat them like 40197.

Retry guidance from the same docs:

- *"We recommend that you wait for 5 seconds before your first retry… the delay should grow exponentially, up to a maximum of 60 seconds."*
- *"Don't directly retry a `SELECT`… establish a fresh connection."* The old connection is likely dead.

## PostgreSQL / Npgsql

**Npgsql has no built-in command retry.** From its failover docs: *"Npgsql will never implicitly retry a failed command on a failover server, since that command may be in a transaction."* PostgreSQL itself says the same about serialization failures: *"retry the complete transaction… PostgreSQL does not offer an automatic retry facility."*

What Npgsql gives you is classification. `NpgsqlException.IsTransient` indicates *"whether retrying the operation could succeed (e.g. a network error or a timeout)"*. For server errors, `PostgresException.IsTransient` is true for SQLSTATEs such as:

| SQLSTATE | Name |
|---|---|
| `40001` | serialization_failure |
| `40P01` | deadlock_detected |
| `53300` | too_many_connections |
| `55P03` | lock_not_available |
| `57P01` / `57P02` / `57P03` | admin_shutdown / crash_shutdown / cannot_connect_now |
| `08xxx` | connection exceptions (e.g. `08006` connection_failure) |

> **Contrast with SqlClient:** Npgsql treats an inner `TimeoutException` as transient. If your retry layer keys off `IsTransient`, it **will** retry timed-out commands. Make sure those commands are idempotent.

### Multi-host failover

Npgsql 6.0+ accepts several hosts. Since 7.0, `NpgsqlDataSource` lets you pick the server role per data source:

```csharp
var builder = new NpgsqlDataSourceBuilder("Host=pg1,pg2,pg3;Database=orders;Username=app");
await using var multiHost = builder.BuildMultiHost();

var primary = multiHost.WithTargetSession(TargetSessionAttributes.Primary);         // writes
var replicas = multiHost.WithTargetSession(TargetSessionAttributes.PreferStandby);  // reads

await using var connection = await primary.OpenConnectionAsync(ct);
```

Other values: `Any`, `Standby`, `PreferPrimary`, `ReadWrite`, `ReadOnly`. Gotchas from the docs and open issues:

- After a failover *"you may get a broken connection from the pool"*. Your retry policy still has to handle it.
- Pooled connections can linger after a host is demoted (npgsql #5807).
- *"No suitable host was found"* hides the real cause in `InnerException` (#5766). Log the full exception.

### PgBouncer

When an external pooler like PgBouncer sits in front, the Npgsql docs say *"you'll want to turn off Npgsql's internal connection pool by specifying `Pooling=false`"*. If you keep Npgsql pooling with PgBouncer in transaction mode, set `No Reset On Close=true`. An Npgsql maintainer notes that errors like *`prepared statement "_auto0" does not exist`* can happen when a pooler doesn't track prepared statements (npgsql #6178).

## Writing your own retry: Polly v8 around ADO.NET or Dapper

When there is no ORM strategy, retry with a Polly pipeline. Retry the **whole unit of work**, including opening the connection and the transaction:

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        ShouldHandle = new PredicateBuilder()
            .Handle<SqlException>(ex => ex.Number is 1205 or 4060 or 40197 or 40501 or 40613),
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        Delay = TimeSpan.FromSeconds(1)
    })
    .Build();

await pipeline.ExecuteAsync(async ct =>
{
    await using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync(ct);
    await using var transaction = await connection.BeginTransactionAsync(ct);

    await connection.ExecuteAsync(
        "UPDATE Accounts SET Balance = Balance - @Amount WHERE Id = @From",
        new { Amount = amount, From = fromId }, transaction);
    await connection.ExecuteAsync(
        "UPDATE Accounts SET Balance = Balance + @Amount WHERE Id = @To",
        new { Amount = amount, To = toId }, transaction);

    await transaction.CommitAsync(ct);
}, cancellationToken);
```

Polly's default `ShouldHandle` is *"Any exceptions other than `OperationCanceledException`"*. Never use the default for database work, because it would retry constraint violations and syntax errors. Always list the transient errors explicitly. Retry, circuit breaker and backoff details are in [Retries, Backoff and Jitter](../18-reliability-and-sre/01-retries-backoff-jitter.md).

> **The commit problem still applies.** If the connection drops during `CommitAsync`, the transfer may have happened, and the retry runs it again. Make the operation idempotent (a transfer ID with a unique constraint) or verify before retrying. [EF Core Resilience](15-ef-core-resilience.md#commit-failure-the-idempotency-problem) lists the four standard options.

## Idempotent writes

*"Consider whether the operation is idempotent. If so, it's inherently safe to retry."* Ways to get there:

- **Client-generated keys** (GUID / ULID). A duplicate insert fails on the primary key instead of creating a second row.
- **Natural unique constraints**, e.g. `UNIQUE (PaymentId)` on a ledger table.
- **Upserts** instead of blind inserts.
- **An idempotency table** keyed by request ID, written in the same transaction. See [Idempotency](../19-distributed-systems/03-idempotency.md).

## Retry at one layer only

A .NET app can have **five** stacked retry layers against the same database:

| Layer | Scope |
|---|---|
| `ConnectRetryCount` (SqlClient) | Reconnecting a broken idle connection |
| SqlClient configurable retry logic | `Open` and single commands |
| EF Core execution strategy | `SaveChanges` / a unit of work |
| Your Polly pipeline | Any delegate |
| The caller's HTTP retry | The whole API request |

The Azure guidance: *"avoid implementations that include duplicated layers of retry code… A retry with a count of three on both calls adds up to nine retry attempts."* Pick one layer per concern: usually connection-level reconnects plus **one** unit-of-work retry. A SqlClient maintainer notes that `ConnectRetryCount` uses its own transient list and can multiply with configurable retry logic (dotnet/SqlClient #1409).

Logging guidance from the same page: *"Log transient faults as warning entries rather than as error entries."* Log `Error` only when the last attempt fails.

## Failover-aware connection strings (SQL Server Availability Groups)

| Keyword | What it does | What it does NOT do |
|---|---|---|
| `MultiSubnetFailover=True` | *"attempts connections in parallel and uses the first successful connection"* across the listener's IPs | *"doesn't shorten the server's failover or database recovery time"*. Incompatible with `Failover Partner`. |
| `ApplicationIntent=ReadOnly` | *"request a read workload on an AG database"*. With read-only routing configured on the server, lands on a secondary. | Doesn't work without the listener + server-side routing |

None of these *"make an in-flight transaction survive a broken connection"*. You still need the retry logic above. See [Replication and Read Replicas](11-replication-and-read-replicas.md).

## Circuit breaker on the database?

The Azure Circuit Breaker pattern warns that requests piling up on a failing dependency *"might hold critical system resources, such as memory, threads, and database connections"*. It notes that *"in a data store that contains multiple shards, one shard might be fully accessible while another experiences a temporary problem."*

My recommendation:

- **Single primary database:** a breaker turns slow failures into fast failures, which frees threads. But the app is down either way. **Timeouts + a concurrency limit (bulkhead)** on DB work usually give more value.
- **Optional data stores** (read replica, reporting DB, search index) **and shards:** a breaker per store/shard lets you degrade gracefully (serve cached data, hide a widget) instead of failing the request. See [Circuit Breaker](../18-reliability-and-sre/02-circuit-breaker.md) and [Bulkhead Pattern](../18-reliability-and-sre/03-bulkhead-pattern.md).

## Health checks

The ASP.NET Core docs show a database probe using the community `AspNetCore.Diagnostics.HealthChecks` packages. Note: *"Microsoft doesn't maintain or support `AspNetCore.Diagnostics.HealthChecks`."* The docs also add that *"In most cases, running a test query isn't necessary."* With EF Core, use `AddDbContextCheck` (see [EF Core Resilience](15-ef-core-resilience.md#health-check)). Register database checks as **readiness**, not liveness. Restarting the app does not fix the database.

## Checklist

- [ ] Every connection/reader disposed on every path (`await using`)
- [ ] Connect and command timeouts set below the caller's deadline
- [ ] Retry only explicit transient errors, never Polly's catch-all default
- [ ] Retry the whole transaction, never a single statement inside it
- [ ] Writes are idempotent before they are retryable
- [ ] Exactly one unit-of-work retry layer
- [ ] Transient failures logged as `Warning`, final failure as `Error`
- [ ] DB health check registered as readiness

---

[← Previous: Data Modeling](13-data-modeling.md) | [Next: EF Core Resilience →](15-ef-core-resilience.md) | [Back to index](README.md)
