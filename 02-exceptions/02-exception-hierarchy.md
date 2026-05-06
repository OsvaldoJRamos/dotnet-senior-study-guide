# Exception Hierarchy

Every exception in .NET inherits from `System.Exception`. Knowing the standard branches lets you catch at the right level — broad enough to cover related errors, narrow enough to not swallow bugs.

## The base: `System.Exception`

> "Represents errors that occur during application execution." — MS Learn

`Exception` is a regular class (not abstract). It implements `ISerializable` so exceptions can cross AppDomain / serialization boundaries. Direct constructors:

| Constructor | Use |
|---|---|
| `Exception()` | Default values |
| `Exception(string message)` | With a message |
| `Exception(string message, Exception innerException)` | With message + wrapped cause |

The `(message, innerException)` overload is the one that matters for wrapping — it's how you preserve the underlying cause chain.

## Two historical branches

.NET inherits a design split from the original Framework:

| Branch | Meaning |
|---|---|
| `SystemException` | Errors thrown by the runtime / framework (`NullReferenceException`, `IndexOutOfRangeException`, `InvalidOperationException`, etc.) |
| `ApplicationException` | Originally meant for application-defined exceptions |

The intent was: app code derives from `ApplicationException`, runtime/framework code derives from `SystemException`. **In practice, the distinction is dead.** The official Framework Design Guidelines recommend deriving custom exceptions directly from `Exception` (not from `ApplicationException`), and Microsoft's own libraries do not consistently use the split. Treat `ApplicationException` as deprecated by convention.

> Rule for new code: derive your custom exceptions from `System.Exception` directly. See [Custom Exceptions](04-custom-exceptions.md).

## Common exceptions a senior must recognize

### Argument validation

| Type | When |
|---|---|
| `ArgumentException` | Generic argument problem. Has `ParamName`. |
| `ArgumentNullException` | Required argument was `null`. |
| `ArgumentOutOfRangeException` | Argument outside the allowed range. |

These all inherit `ArgumentException → SystemException → Exception`. Use the most specific one. .NET 6+ ships static `ThrowIf*` helpers (`ArgumentNullException.ThrowIfNull`, `ArgumentOutOfRangeException.ThrowIfNegative`, etc.) — see [Best Practices](03-best-practices.md).

### State / contract violations

| Type | When |
|---|---|
| `InvalidOperationException` | The object is in a state where the call is invalid (`List<T>` modified during enumeration, etc.) |
| `NotSupportedException` | Operation isn't supported (e.g., a read-only collection's `Add`) |
| `NotImplementedException` | Stub for unfinished code — should never reach production |
| `ObjectDisposedException` | Method called on a disposed object |

`InvalidOperationException` is the right choice for "you can't do that *right now*"; `NotSupportedException` is for "you can't do that *at all*".

### I/O and infrastructure

| Type | When |
|---|---|
| `IOException` | Base for file/stream errors |
| `FileNotFoundException` | Specific file missing |
| `DirectoryNotFoundException` | Specific directory missing |
| `UnauthorizedAccessException` | Permission denied |
| `System.Net.Http.HttpRequestException` | HTTP-level failure |
| `System.Data.Common.DbException` | Base for database provider errors |
| `TimeoutException` | Generic timeout |
| `TaskCanceledException` (subtype of `OperationCanceledException`) | Async operation cancelled |

For network calls always inspect derived types or status codes — generic `Exception` catches mask real problems.

### Concurrency

| Type | When |
|---|---|
| `OperationCanceledException` | Cancellation requested via a `CancellationToken` |
| `TaskCanceledException` | The task itself was cancelled (subclass of `OperationCanceledException`) |
| `AggregateException` | Wraps multiple exceptions thrown by parallel work; see [Async and AggregateException](07-async-and-aggregate.md) |
| `LockRecursionException`, `SynchronizationLockException` | Lock primitive misuse |

> **Cancellation tip:** catch `OperationCanceledException`, not `TaskCanceledException`. Per the official guidance: *"It's better to catch `OperationCanceledException` instead of `TaskCanceledException`, which derives from `OperationCanceledException`, when you call an asynchronous method."*

### "Reserved" exceptions you should NOT throw

Per code analysis rule **CA2201**, these are reserved for the runtime:

- `Exception` (base — too vague)
- `SystemException` (too vague)
- `NullReferenceException` (only the runtime throws this)
- `IndexOutOfRangeException` (only the runtime throws this)
- `OutOfMemoryException` (the GC throws this)
- `StackOverflowException` (uncatchable since .NET 2.0)
- `AccessViolationException` (memory corruption — bug in the runtime, not in your code)

Throwing any of these in your code muddles diagnostics. Use `ArgumentNullException`, `ArgumentOutOfRangeException`, etc. instead.

### Special: exceptions you can't really catch

- **`StackOverflowException`** — the CLR cannot reliably run a `catch` handler when the stack is gone. As of .NET 2.0+, this exception terminates the process by default; catching it has no effect.
- **`OutOfMemoryException`** — technically catchable but at that point you almost can't allocate anything safely.
- **`ExecutionEngineException`** — runtime corruption, terminates the process.

Treat these as crash-the-process events. The "fail fast" doctrine applies (see [Best Practices](03-best-practices.md)).

## Drawing the catch boundary

A general guide for what level to catch at:

```csharp
try
{
    // operation
}
catch (FileNotFoundException ex)        // most specific you can recover from
{
    return Result.NotFound();
}
catch (IOException ex)                   // umbrella for file/stream issues
{
    _logger.LogError(ex, "I/O error");
    throw;                               // can't recover — let it bubble
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    return Result.Cancelled();
}
// no catch (Exception) — let unknown errors propagate to a global handler
```

`catch (Exception)` is reserved for the *outermost* boundary (a hosting layer like ASP.NET Core's exception middleware, a `BackgroundService` loop, a fire-and-forget guard) where the alternative is letting the process die. In ordinary application code, narrow exception types make bugs surface fast.

## Useful Links

- [`System.Exception` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.exception) — full class hierarchy under the "Derived" section
- [Best practices for exceptions — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions) — official handling and throwing guidance
- [CA2201 — Don't raise reserved exception types](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2201) — list of reserved types

---

[← Previous: Fundamentals](01-fundamentals.md) | [Back to index](README.md) | [Next: Best Practices →](03-best-practices.md)
