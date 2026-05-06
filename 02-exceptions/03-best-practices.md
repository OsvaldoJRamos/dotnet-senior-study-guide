# Best Practices

Most exception bugs come from a small set of bad habits. The official Microsoft guidance lays them out clearly; this file boils them down to the ones that actually move the needle on senior code.

## Catch only what you can recover from

The MS Learn "Best practices for exceptions" guide is direct:

> "When your code can't recover from an exception, don't catch that exception. Enable methods further up the call stack to recover if possible."

Catching exceptions you can't actually do anything about turns a clear failure into a silent bug. Two anti-patterns:

```csharp
// ANTI-PATTERN 1: swallow
try { _service.Save(order); }
catch (Exception) { /* nothing */ }   // crashed quietly; no one will know

// ANTI-PATTERN 2: log-and-continue at every layer
try { _service.Save(order); }
catch (Exception ex)
{
    _logger.LogError(ex, "Save failed");
    return null;     // caller now sees null and has no idea why
}
```

The "swallow" is obvious; the "log-and-continue" is the more insidious one — it looks responsible (we logged it!) but converts an exceptional state into a normal-looking return value, and now every caller has to either trust the result or do its own validation. Worse, the SAME log line repeats at every layer the exception passes through.

**Prefer**: catch at the **outermost boundary** (HTTP middleware, a `BackgroundService` loop, the `Main` method, a message-bus consumer). Inner code lets exceptions propagate. The outer boundary translates the exception into the response shape that boundary needs (HTTP status, log, retry, dead-letter queue, etc.).

## Use `try`/`finally` (or `using`) for cleanup

> "Use `try`/`catch`/`finally` blocks to recover from errors or release resources. ... Code in a `finally` clause is almost always executed even when exceptions are thrown." — MS Learn

For anything `IDisposable`/`IAsyncDisposable`, use `using` — the compiler emits the correct `try`/`finally` for you. Manual `try`/`finally` is for non-disposable resources (counter increments, flags to reset, etc.).

```csharp
// Sync resource cleanup
public async Task ProcessAsync(int id, CancellationToken ct)
{
    Busy = true;
    try
    {
        await DoWorkAsync(id, ct);
    }
    finally
    {
        Busy = false;   // restored even if DoWorkAsync throws
    }
}
```

## Tester-Doer and Try-Parse patterns

Per the Framework Design Guidelines: *"DO NOT use exceptions for the normal flow of control, if possible."* Two patterns:

### Tester-Doer

Provide a "tester" method to check preconditions before calling the "doer":

```csharp
if (queue.Count > 0)         // tester
    var item = queue.Dequeue();   // doer (throws if empty)
```

### Try-Parse

When the test-then-do has a race condition (e.g., a value could change between the check and the operation), use a `Try*` method that returns a `bool` and an `out` parameter:

```csharp
// Bad: throws OverflowException on bad input
int n = int.Parse(input);

// Good: returns false, no allocation, no stack trace
if (!int.TryParse(input, out int n)) return Result.BadRequest();
```

The MS Learn best-practices guide:

> "If the performance cost of exceptions is prohibitive, some .NET library methods provide alternative forms of error handling. For example, `Int32.Parse` throws an `OverflowException` if the value to be parsed is too large to be represented by `Int32`. However, `Int32.TryParse` doesn't throw this exception."

Common Try-Parse pairs in BCL: `int.TryParse`, `Guid.TryParse`, `Dictionary.TryGetValue`, `Queue.TryDequeue`, `ConcurrentQueue.TryDequeue`, `Channel.Reader.TryRead`. Use the `Try*` form on hot paths.

## Use `ArgumentNullException.ThrowIfNull` and friends

.NET 6+ ships built-in `ThrowIf*` helpers that allocate-and-throw with `[CallerArgumentExpression]` for readable parameter names:

```csharp
// Old verbose style
public void Process(Order order)
{
    if (order is null) throw new ArgumentNullException(nameof(order));
    // ...
}

// Modern (.NET 6+)
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    // ...
}
```

The full set, all from the official "Best practices" page:

- `ArgumentNullException.ThrowIfNull`
- `ArgumentException.ThrowIfNullOrEmpty`, `ThrowIfNullOrWhiteSpace`
- `ArgumentOutOfRangeException.ThrowIfNegative`, `ThrowIfNegativeOrZero`, `ThrowIfZero`, `ThrowIfEqual`, `ThrowIfNotEqual`, `ThrowIfLessThan`, `ThrowIfGreaterThan`, `ThrowIfLessThanOrEqual`, `ThrowIfGreaterThanOrEqual`
- `ObjectDisposedException.ThrowIf`
- `CancellationToken.ThrowIfCancellationRequested`

These are equivalent to the manual `if-throw` but shorter, JIT-friendly, and code-analyzer-recommended (CA1510-CA1513).

## Fail fast for unrecoverable state

> "CONSIDER terminating the process by calling `System.Environment.FailFast` ... instead of throwing an exception if your code encounters a situation where it is unsafe for further execution." — Framework Design Guidelines

`FailFast` skips `finally` blocks and immediately terminates the process with an event log entry. Use when state corruption means the next line of code could do real harm (e.g., financial logic detecting an impossible invariant violation). This is rare in application code; more common in critical infrastructure.

```csharp
if (account.Balance < 0)   // an invariant we believe is impossible
    Environment.FailFast("Negative balance detected for account " + account.Id);
```

## Restore state on failure

> "Callers should be able to assume that there are no side effects when an exception is thrown from a method." — MS Learn

If your method partially mutates state and then throws, callers see corruption. Two options:

1. **Validate up front** — check all preconditions before any mutation.
2. **Rollback in `catch`** — if the operation is multi-step, undo earlier steps when a later step fails.

```csharp
public void TransferFunds(Account from, Account to, decimal amount)
{
    string trxId = from.Withdrawal(amount);
    try
    {
        to.Deposit(amount);
    }
    catch
    {
        from.RollbackTransaction(trxId);
        throw;
    }
}
```

This is a value-preserving design pattern — sometimes called "strong exception guarantee" (the operation either succeeds completely or leaves state unchanged).

## Use `throw;` to rethrow, never `throw ex`

```csharp
// CORRECT
catch (Exception ex)
{
    _logger.LogError(ex, "Failed");
    throw;                    // original stack preserved
}

// BUG (CA2200)
catch (Exception ex)
{
    _logger.LogError(ex, "Failed");
    throw ex;                 // stack restarts here — lost frames
}
```

Detailed explanation in [Fundamentals](01-fundamentals.md#throw-semantics). For rethrowing from a different scope (e.g., a different thread), use `ExceptionDispatchInfo` — see [`ExceptionDispatchInfo`](06-dispatch-info.md).

## Throw argument validation synchronously in async methods

> "In task-returning methods, you should validate arguments and throw any corresponding exceptions, such as `ArgumentException` and `ArgumentNullException`, before entering the asynchronous part of the method. Exceptions that are thrown in the asynchronous part of the method are stored in the returned task and don't emerge until, for example, the task is awaited." — MS Learn

```csharp
// BAD — exception goes into the Task, not raised on call
public async Task SaveAsync(Order order)
{
    ArgumentNullException.ThrowIfNull(order);   // OK to validate at top
    await _db.SaveAsync(order);
}
```

Wait — that's actually CORRECT because the validation runs synchronously *before* the first `await`. The trap is splitting the method into a sync wrapper that returns the task. Pattern:

```csharp
public Task<Order> GetAsync(int id)
{
    if (id < 0) throw new ArgumentOutOfRangeException(nameof(id));
    return GetAsyncInner(id);   // returns the actual async task
}

private async Task<Order> GetAsyncInner(int id) { /* ... */ }
```

This way the `ArgumentOutOfRangeException` is thrown **synchronously** at the call site, not stored in the returned task to be observed only at `await`. Useful when validation is cheap and you want callers to see invalid inputs immediately.

## Don't throw from `Equals`, `GetHashCode`, `ToString`, static constructors

Per CA1065. These methods are called by frameworks (collections, debuggers, serializers) and are expected to never throw. A throw from `GetHashCode` can corrupt a `Dictionary`; from `ToString` it breaks debugging; from a static constructor it permanently faults the type with `TypeInitializationException`.

## Don't throw from exception filters or `finally`

Per the Framework Design Guidelines and CA2219:

> "DO NOT throw exceptions from exception filter blocks. When an exception filter raises an exception, the exception is caught by the CLR, and the filter returns false. This behavior is indistinguishable from the filter executing and returning false explicitly and is therefore very difficult to debug."
>
> "AVOID explicitly throwing exceptions from finally blocks. Implicitly thrown exceptions resulting from calling methods that throw are acceptable."

A throw inside `finally` masks the original exception that triggered the unwind. A throw inside an exception filter silently turns the filter into a `false`.

## Senior-interview gotchas

- **`throw;` preserves the stack; `throw ex;` does not.** CA2200.
- **Don't catch what you can't fix.** Let it bubble to a single boundary.
- **`catch (Exception)` is for the outermost boundary, not for sprinkling.**
- **Use `Try*` patterns on hot paths.** Throw cost is real (see [Performance](08-performance.md)).
- **Don't throw `NullReferenceException`, `IndexOutOfRangeException`, etc.** — they're reserved for the runtime (CA2201).
- **Use `ArgumentNullException.ThrowIfNull` and friends** in .NET 6+.
- **Validate arguments synchronously** in `async` methods (or split into sync wrapper that throws + private async core).
- **`StackOverflowException` is uncatchable since .NET 2.0.** Don't try.
- **Don't throw in `finally`, exception filters, `Equals`, `GetHashCode`, `ToString`, or static constructors.**

## Useful Links

- [Best practices for exceptions — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Exception throwing — Framework Design Guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/exception-throwing) — DO/DO NOT/CONSIDER bullets from the Cwalina-Abrams book
- [CA2200: Rethrow to preserve stack details](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2200)
- [CA2201: Don't raise reserved exception types](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2201)
- [CA1065: Don't raise exceptions in unexpected locations](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1065)
- [CA2219: Don't raise exceptions in finally clauses](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2219)

---

[← Previous: Exception Hierarchy](02-exception-hierarchy.md) | [Back to index](README.md) | [Next: Custom Exceptions →](04-custom-exceptions.md)
