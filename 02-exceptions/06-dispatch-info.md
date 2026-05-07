# `ExceptionDispatchInfo`

The standard `throw;` statement preserves stack trace, but it only works **inside the same `catch` block**. If you need to capture an exception in one place and rethrow it elsewhere — different method, different thread, different `await` — `throw;` isn't available. That's what `ExceptionDispatchInfo` is for.

## The problem it solves

Per the official `ExceptionDispatchInfo` docs:

> "Represents an exception whose state is captured at a certain point in code."
>
> "An `ExceptionDispatchInfo` object stores the stack trace information and Watson information that an exception contains at the point where it's captured. The exception can then be thrown at another time and possibly on another thread by calling the `ExceptionDispatchInfo.Throw` method. The exception is thrown as if it had flowed from the point where it was captured to the point where the `Throw` method is called."

A common case: an aggregator that runs N tasks, captures any failures, and after all complete, replays the first failure to the caller — but with the **original** stack from where it was thrown, not from where the aggregator decided to rethrow.

## The naive approach loses information

Without `ExceptionDispatchInfo`:

```csharp
Exception captured = null;
try { await Operation1Async(); }
catch (Exception ex) { captured = ex; }   // capture for later

await Operation2Async();   // do other things

if (captured != null)
    throw captured;        // ⚠️ stack trace now starts here
```

The `throw captured;` is equivalent to `throw e;` — the runtime resets the stack trace's "current location" to this line. The original throw site at `Operation1Async` is lost.

## With `ExceptionDispatchInfo`

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo? edi = null;

try { await Operation1Async(); }
catch (Exception ex) { edi = ExceptionDispatchInfo.Capture(ex); }

await Operation2Async();

edi?.Throw();   // ✓ stack preserved as if thrown from inside Operation1Async
```

The throw is rendered with both the original stack frames AND a separator showing where the rethrow happened, so you see the full causal chain in the trace:

```
System.IO.FileNotFoundException: Could not find file 'C:\temp\file.txt'.
   at System.IO.File.ReadAllText(String path)
   at MyApp.Operation1Async() in Program.cs:line 12
--- End of stack trace from previous location ---
   at MyApp.Aggregator() in Program.cs:line 24
```

The `--- End of stack trace from previous location ---` marker is the runtime's signal that this is a captured-and-rethrown exception (`ExceptionDispatchInfo.Throw` or async/await re-raise).

## Key API

From the docs:

| Member | Purpose |
|---|---|
| `Capture(Exception)` | Static factory: package an exception's state for later rethrow |
| `SourceException` | The wrapped exception (read-only access without rethrowing) |
| `Throw()` | Rethrow the captured exception, preserving original stack |
| `Throw(Exception)` | Static helper added in .NET 5+: throw any exception while preserving its existing stack |
| `SetCurrentStackTrace(Exception)` | Stamp the current stack trace into an exception (rare; for captured-without-throwing scenarios) |
| `SetRemoteStackTrace(Exception, string)` | Inject an external stack trace into an exception (for cross-process serialization) |

> "`ExceptionDispatchInfo` cannot be serialized and is not intended to cross application domain boundaries." — MS Learn

## Where the runtime uses it for you

You don't always have to reach for `ExceptionDispatchInfo` — `async`/`await` uses it internally:

- When an `async` method throws, the exception is captured into the returned `Task`.
- When you `await` a faulted task, the runtime calls (effectively) `ExceptionDispatchInfo.Throw()` on the stored exception. That's why `await` rethrows the **original** exception with the **original** stack, not an `AggregateException` wrapper.

This is the mechanism behind "the right exception at the right `await`" without the `AggregateException` wrapping — see [Async Exceptions and AggregateException](07-async-and-aggregate.md) and the [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md) coverage.

## When to actually reach for it

Most application code doesn't need `ExceptionDispatchInfo` — `async`/`await` and `throw;` cover the common cases. Reach for it when:

1. **You capture in one method and rethrow in another.** A retry/circuit-breaker layer that captures and replays an exception after some delay, deferring throw until policy decides.

   ```csharp
   public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
   {
       var failures = new List<ExceptionDispatchInfo>();
       for (int attempt = 0; attempt < 3; attempt++)
       {
           try { return await operation(); }
           catch (Exception ex) when (IsTransient(ex))
           {
               failures.Add(ExceptionDispatchInfo.Capture(ex));
               await Task.Delay(_backoff[attempt]);
           }
       }
       failures[^1].Throw();   // rethrow last failure with original stack
       throw new UnreachableException();   // unreachable, satisfy compiler
   }
   ```

2. **You aggregate from many tasks and want to replay one specific failure.** Sometimes preferable to `AggregateException`-then-`Flatten` when callers expect the original exception type.

3. **You implement a custom awaitable / synchronization primitive** that needs to surface a stored exception to whoever awaits it.

## Capturing without throwing

`ExceptionDispatchInfo.Capture` does **not** throw — it just records the exception. The throw happens on `Throw()`. If you never call `Throw`, the captured info is just an object the GC can collect. Useful for "fire and forget but log the first error" patterns.

```csharp
ExceptionDispatchInfo? firstError = null;
foreach (var task in tasks)
{
    try { await task; }
    catch (Exception ex) { firstError ??= ExceptionDispatchInfo.Capture(ex); }
}
firstError?.Throw();   // rethrow the first one we saw, with its original stack
```

## Senior-interview gotchas

- **`throw;` only works inside `catch`.** For cross-method/cross-thread rethrow with stack preservation, use `ExceptionDispatchInfo`.
- **`Capture` does not throw** — it only records. `Throw` is the rethrow.
- **`Throw` stamps a "previous location" marker** in the stack trace — that's how you recognize a captured rethrow.
- **`async`/`await` uses this internally** — that's why `await` doesn't wrap exceptions in `AggregateException`.
- **Not serializable; doesn't cross AppDomain boundaries.** Cross-process exception flow uses message serialization, not `ExceptionDispatchInfo`.
- **`ExceptionDispatchInfo.Throw(Exception)` (static, .NET 5+)** is a convenient one-liner for "throw this exception preserving its stack" — works without explicitly calling `Capture` first.

## Useful Links

- [`ExceptionDispatchInfo` class — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo) — capture, throw, source, set-stack-trace
- [Best practices: Capture and rethrow exceptions properly — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions#capture-and-rethrow-exceptions-properly) — official guidance with example output

---

[← Previous: Exception Filters](05-exception-filters.md) | [Back to index](README.md) | [Next: Async Exceptions and AggregateException →](07-async-and-aggregate.md)
