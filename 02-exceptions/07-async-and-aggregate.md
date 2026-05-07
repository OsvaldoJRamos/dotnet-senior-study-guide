# Async Exceptions and `AggregateException`

Async code changes how exceptions flow through your program. The two pieces a senior must know: how exceptions traverse `async`/`await` (and where they live in between), and how `AggregateException` wraps multiple failures from parallel work.

> Most of the deep details are in the concurrency section — see [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md) and [Task, async/await](../06-concurrency-and-parallelism/03-task-async-await.md). This file focuses on the exception-handling angle.

## How async exceptions propagate

When an `async` method throws, the exception is **captured into the returned `Task`** (not raised at the call site). The task transitions to the `Faulted` state. The exception emerges only when the task is observed — typically via `await`:

```csharp
async Task<int> ProcessAsync(int n)
{
    if (n < 0) throw new ArgumentException("must be non-negative");
    await Task.Delay(100);
    return n * 2;
}

try
{
    int result = await ProcessAsync(-1);   // ← exception emerges here, not above
}
catch (ArgumentException ex)
{
    // ex is the original exception, unwrapped
}
```

Per the official "Best practices" guide:

> "Asynchronous methods store exceptions that are thrown during execution in the task they return. If an exception is stored into the returned task, that exception is thrown when the task is awaited. Usage exceptions, such as `ArgumentException`, are still thrown synchronously."

The "still thrown synchronously" part: `await` rethrows the **original** exception type — not `AggregateException`. The mechanism is `ExceptionDispatchInfo` under the hood (see [`ExceptionDispatchInfo`](06-dispatch-info.md)). The stack trace shows the original throw site plus a `--- End of stack trace from previous location ---` separator and the `await` site.

### Synchronous validation in async methods

If an exception is "synchronous" (validation that should fail at the call site, not at `await`), throw it before any `await`. Or split into a sync wrapper:

```csharp
public Task<int> GetAsync(int id)
{
    ArgumentOutOfRangeException.ThrowIfNegative(id);
    return GetAsyncCore(id);    // returns a real async task
}

private async Task<int> GetAsyncCore(int id) { /* await */ }
```

Now invalid `id` throws *immediately* at the call, not inside the `await`. See [Best Practices](03-best-practices.md#throw-argument-validation-synchronously-in-async-methods).

### Exceptions in iterator methods

For `IEnumerable<T>` and `IAsyncEnumerable<T>` methods that throw before yielding, the exception propagates only on the first `MoveNext`/`MoveNextAsync`:

> "If an exception occurs in an iterator method, the exception propagates to the caller only when the iterator advances to the next element." — MS Learn

A subtle gotcha when refactoring: code that *creates* the iterator runs synchronously, but the exception at the start of the iterator's body waits until the first `foreach` iteration to surface.

## `AggregateException`

When multiple tasks fail (`Task.WhenAll`, `Parallel.For/ForEach`, `Task.Wait` on a multi-failure scenario), the framework needs a way to surface them all. That's `AggregateException`.

```csharp
var tasks = urls.Select(url => httpClient.GetAsync(url));
try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
    // Surprise: ex is just the FIRST exception, not AggregateException!
    // await unwraps to the first inner exception of the AggregateException.
}
```

That's a famous senior-interview gotcha: `await Task.WhenAll(...)` rethrows only the **first** inner exception. If there are 5 failures, `await` raises 1; the other 4 are still observable via `Task.Exception` on the `WhenAll` result.

To get all of them:

```csharp
Task all = Task.WhenAll(tasks);
try
{
    await all;
}
catch
{
    // all.Exception is the AggregateException with all failures
    foreach (var inner in all.Exception!.InnerExceptions)
        _logger.LogError(inner, "One of many failures");
    throw;
}
```

## `.Wait()` and `.Result` always wrap

In contrast to `await`, the synchronous blocking calls `task.Wait()` and `task.Result` **always** throw `AggregateException`, even for a single inner exception or cancellation.

From the official `Task.Wait` docs:

> "**`AggregateException`**: The task was canceled. The `InnerExceptions` collection contains a `TaskCanceledException` object. -or- An exception was thrown during the execution of the task. The `InnerExceptions` collection contains information about the exception or exceptions."

Two consequences:

1. You **cannot** `catch (OperationCanceledException)` directly when blocking — it'll be wrapped.
2. `catch (InvalidOperationException)` won't match either — it's wrapped.

The unwrap-at-block compromise is `task.GetAwaiter().GetResult()` — same blocking cost as `.Wait()`/`.Result` but exception types are unwrapped (it's what `await` uses internally). See [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md#wait--result-and-the-aggregateexception-wrapping).

## `Flatten` and `Handle`

`AggregateException` can nest (parallel-of-parallel scenarios → an `AggregateException` whose `InnerExceptions` contain other `AggregateException`s). The `Flatten()` method recursively unwraps all nested aggregates into a single flat aggregate:

```csharp
catch (AggregateException ae)
{
    foreach (var inner in ae.Flatten().InnerExceptions)
        _logger.LogError(inner, "Failure");
}
```

`Flatten` returns a NEW `AggregateException` (it doesn't mutate the original). Use it when you have nested aggregates from `Parallel.For` containing parallel sub-work.

The `Handle(Func<Exception, bool>)` method is a less-common API for selectively consuming inner exceptions:

```csharp
catch (AggregateException ae)
{
    ae.Handle(ex =>
    {
        if (ex is OperationCanceledException) return true;   // consumed
        return false;   // re-aggregate and rethrow
    });
}
```

If `Handle` returns `true` for every inner exception, the aggregate is fully consumed (no rethrow). If `false` for any, the unhandled ones are wrapped in a NEW `AggregateException` and thrown. Most codebases prefer the explicit `foreach` over `Handle` because it's clearer.

## Cancellation specifics

The cancellation rules from the [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md#when-does-cancellation-become-canceled-not-faulted) page apply here:

- **`await` on a `Canceled` task** → throws `TaskCanceledException` (a subclass of `OperationCanceledException`), unwrapped.
- **`Wait()` / `.Result` on a `Canceled` task** → throws `AggregateException` wrapping `TaskCanceledException`.
- **`await Task.WhenAll(...)`** when some tasks were cancelled and some faulted → if any faulted, the first faulted exception is rethrown; cancellations are observable on the individual tasks.

The official advice: catch `OperationCanceledException`, not `TaskCanceledException`, since the latter is a subtype and other async APIs may throw the base type:

```csharp
try { await DoAsync(ct); }
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // graceful exit
}
```

## When `AggregateException` is the right shape

Use `AggregateException` when:

- You ran multiple operations and want callers to see **all** failures.
- You're aggregating results across tasks (e.g., a bulk-operation handler that reports per-item failures).

For most async I/O code, `await` + the first-exception-wins behavior is fine. The cases where you need to manually inspect `Task.Exception` and walk `InnerExceptions` are real but rare.

## Senior-interview gotchas

- **`async` methods catch their own exceptions** — they're stored in the returned `Task`, not raised at call.
- **`await` rethrows the original exception**, unwrapped. That's via `ExceptionDispatchInfo`.
- **`.Wait()` / `.Result` always wraps in `AggregateException`** — including for cancellation.
- **`await Task.WhenAll(...)` rethrows only the FIRST inner exception.** The rest are reachable via `Task.Exception.InnerExceptions`.
- **`AggregateException.Flatten()` recursively unwraps nested aggregates.** Useful for `Parallel.For` of `Parallel.For`.
- **Catch `OperationCanceledException`, not `TaskCanceledException`.** It's the broader, more idiomatic type.
- **Synchronous validation should throw before the first `await`** so callers see the error immediately, not via `Task.Exception`.
- **`task.GetAwaiter().GetResult()`** blocks but doesn't wrap — the lesser evil when you must block.

## Useful Links

- [`AggregateException` class — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.aggregateexception) — Flatten, Handle, InnerExceptions
- [Best practices: Catch cancellation and asynchronous exceptions — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions#catch-cancellation-and-asynchronous-exceptions)
- [Task Lifecycle — internal cross-reference](../06-concurrency-and-parallelism/11-task-lifecycle.md)
- [Task, async/await — internal cross-reference](../06-concurrency-and-parallelism/03-task-async-await.md)

---

[← Previous: ExceptionDispatchInfo](06-dispatch-info.md) | [Back to index](README.md) | [Next: Performance →](08-performance.md)
