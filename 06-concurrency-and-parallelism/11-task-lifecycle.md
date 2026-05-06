# Task Lifecycle

Every `Task` moves through a finite set of states from creation to completion. Knowing the states (and the difference between "completed" and "successful") is the difference between code that handles errors correctly and code that throws on the happy path.

## The eight `TaskStatus` values

The `TaskStatus` enum has **exactly 8 values**, split between in-progress and terminal states. From the official docs (`TaskStatus` MS Learn — quotes are verbatim):

| Value | Name | Description |
|---|---|---|
| 0 | `Created` | "The task has been initialized but has not yet been scheduled." |
| 1 | `WaitingForActivation` | "The task is waiting to be activated and scheduled internally by the .NET infrastructure." |
| 2 | `WaitingToRun` | "The task has been scheduled for execution but has not yet begun executing." |
| 3 | `Running` | "The task is running but has not yet completed." |
| 4 | `WaitingForChildrenToComplete` | "The task has finished executing and is implicitly waiting for attached child tasks to complete." |
| 5 | **`RanToCompletion`** | "The task completed execution successfully." |
| 6 | **`Canceled`** | "The task acknowledged cancellation by throwing an `OperationCanceledException` with its own `CancellationToken` while the token was in signaled state, or the task's `CancellationToken` was already signaled before the task started executing." |
| 7 | **`Faulted`** | "The task completed due to an unhandled exception." |

The last three (`RanToCompletion`, `Canceled`, `Faulted`) are **terminal** — once a task reaches one of them, it never moves again.

## How tasks enter the lifecycle

The starting state depends on how the task was created:

| Source | Initial state |
|---|---|
| `new Task(...)` (rare) | `Created` — explicit `Start()` required |
| `Task.Run(...)`, `Task.Factory.StartNew(...)` | `WaitingToRun` (then `Running`) |
| `async`-method tasks, continuations, `TaskCompletionSource` | `WaitingForActivation` |
| `Task.FromResult(value)` | Born in `RanToCompletion` (per `Task.FromResult` MS Learn — "creates a `Task<TResult>` object whose ... `Status` property is `RanToCompletion`") |
| `Task.FromException(ex)` | Born in `Faulted` |
| `Task.FromCanceled(ct)` | Born in `Canceled` |
| `Task.CompletedTask` | A pre-built task in `RanToCompletion` (the docs note: "Repeated attempts to retrieve this property value may not always return the same instance" — it's *a* completed task, not strictly a singleton) |

`WaitingForChildrenToComplete` is rare in modern code — it requires the older `TaskCreationOptions.AttachedToParent` pattern, almost never used today.

## `IsCompleted` ≠ "ran successfully" — the senior gotcha

`Task` exposes several boolean properties for state. The trap is that **`IsCompleted` returns `true` for any terminal state**, including `Faulted` and `Canceled`:

| Property | True when status is... |
|---|---|
| `IsCompleted` | `RanToCompletion` **or** `Canceled` **or** `Faulted` |
| `IsCompletedSuccessfully` | `RanToCompletion` only — "Gets whether the task ran to completion." (MS Learn) |
| `IsFaulted` | `Faulted` only |
| `IsCanceled` | `Canceled` only |

So the classic bug:

```csharp
// BUG — task may be Faulted or Canceled; .Result will rethrow
if (task.IsCompleted)
    UseResult(task.Result);
```

The correct guard is `IsCompletedSuccessfully`:

```csharp
if (task.IsCompletedSuccessfully)
    UseResult(task.Result);
```

`IsCompletedSuccessfully` was introduced in .NET Core 2.0 / .NET Standard 2.1 and is the right check whenever you want to peek without risk of throwing.

## What `await` does for each terminal state

`await` automatically unwraps the terminal state into the natural language equivalent:

| Terminal state | `await` behavior |
|---|---|
| `RanToCompletion` | Returns the `Task<T>` value, or just continues for plain `Task`. |
| `Faulted` | Rethrows the **original** exception (already unwrapped — not wrapped in `AggregateException`). |
| `Canceled` | Throws `TaskCanceledException` (a subclass of `OperationCanceledException`), unwrapped. |

That last detail is what makes `await` consumer-friendly. With `.Wait()` or `.Result`, you get an `AggregateException` instead.

## `Wait()` / `.Result` and the `AggregateException` wrapping

The official `Task.Wait()` docs document this directly:

> "**`AggregateException`**: The task was canceled. The `InnerExceptions` collection contains a `TaskCanceledException` object. -or- An exception was thrown during the execution of the task. The `InnerExceptions` collection contains information about the exception or exceptions."

So `Wait()` / `.Result` always wrap. If your code throws `InvalidOperationException`, the caller catches `AggregateException` and has to drill into `.InnerException`. For cancellation, the `OperationCanceledException` is wrapped too — you can't `catch (OperationCanceledException)` directly when blocking.

Two ways to unblock the unwrapped exception when you genuinely must block:

```csharp
// Option 1 — ugly but unwrapped
try
{
    return task.GetAwaiter().GetResult();   // unwraps like await would
}
catch (OperationCanceledException) { /* cancellation */ }
catch (InvalidOperationException)  { /* domain error */ }

// Option 2 — keep the wrapping but flatten
try { task.Wait(); }
catch (AggregateException ae)
{
    ae = ae.Flatten();
    foreach (var inner in ae.InnerExceptions) { /* handle each */ }
}
```

`task.GetAwaiter().GetResult()` is the **lesser evil** when sync entry points are unavoidable: same blocking cost as `.Wait()`/`.Result`, but exception types match what the caller would have seen with `await`. See [Task, async/await](03-task-async-await.md) for when blocking is justified at all (rare).

## When does cancellation become `Canceled`, not `Faulted`?

Throwing an `OperationCanceledException` inside the task body is **not enough**. The official Task Cancellation guide (`learn.microsoft.com/.../task-cancellation`) is precise:

> "When a task instance observes an `OperationCanceledException` thrown by the user code, it compares the exception's token to its **associated token** (the one that was passed to the API that created the Task). If the tokens are same **and** the token's `IsCancellationRequested` property returns `true`, the task interprets this as acknowledging cancellation and transitions to the `Canceled` state."

> "If the token's `IsCancellationRequested` property returns `false` or if the exception's token doesn't match the Task's token, the `OperationCanceledException` is treated like a normal exception, causing the Task to transition to the `Faulted` state."

In short, three conditions must all be true to land in `Canceled`:

1. An `OperationCanceledException` is thrown from the task body.
2. The exception carries a `CancellationToken` that **equals** the token passed to the task-creating API (e.g., the `Task.Run` overload that takes a token, or to `Task.Factory.StartNew`).
3. That token's `IsCancellationRequested` is `true`.

If any condition fails, the task ends up `Faulted`. This is why the canonical pattern is:

```csharp
var task = Task.Run(() =>
{
    // Periodically check; throw with the right token if cancelled.
    ct.ThrowIfCancellationRequested();
    DoSomeWork();
}, ct);   // ← same token also passed to Task.Run
```

`ct.ThrowIfCancellationRequested()` does the right thing automatically: if `IsCancellationRequested`, it throws an `OperationCanceledException(ct)` carrying the matching token. Just throwing `new OperationCanceledException()` without a token, or with a different token, lands you in `Faulted`.

> Returning normally from a delegate that was cancelled via `ct.IsCancellationRequested` polling without throwing transitions to `RanToCompletion`, **not** `Canceled`. Only the throw-with-matching-token pattern produces `Canceled`. The official cancellation guide calls this out explicitly.

## `task.Exception` after `Faulted`

```csharp
if (task.IsFaulted)
{
    // task.Exception is always an AggregateException, never null in this branch.
    foreach (var inner in task.Exception!.InnerExceptions)
        _logger.LogError(inner, "Task failed");
}
```

`task.Exception` is `AggregateException` even when only one exception was thrown. For `Canceled` tasks, `task.Exception` is `null` — cancellation isn't a fault.

## Quick decision rule

| You want to... | Use |
|---|---|
| Wait for a task and get the value with natural exceptions | `await task` |
| Check whether a task succeeded without throwing | `task.IsCompletedSuccessfully` |
| Check whether a task is in any terminal state (success, fault, or cancel) | `task.IsCompleted` |
| Block synchronously and propagate `AggregateException` | `task.Wait()` / `task.Result` |
| Block synchronously but get unwrapped exceptions | `task.GetAwaiter().GetResult()` (still blocks — only when async-up isn't possible) |
| Inspect the actual exception(s) of a faulted task | `task.Exception.InnerExceptions` (always wrapped) |
| Build an already-completed task without running anything | `Task.FromResult(value)`, `Task.CompletedTask`, `Task.FromException`, `Task.FromCanceled` |

## Senior-interview gotchas

- **8 `TaskStatus` values total**, with **3 terminal**: `RanToCompletion`, `Canceled`, `Faulted`.
- **Most async-method tasks are born in `WaitingForActivation`**, not `Created` or `WaitingToRun`.
- **`IsCompleted` is the wrong check for "succeeded".** Use `IsCompletedSuccessfully`.
- **`await` unwraps; `.Wait()`/`.Result` wraps in `AggregateException`.** `GetAwaiter().GetResult()` is the unwrapped-blocking compromise.
- **Three conditions** turn an OCE into `Canceled` (not `Faulted`): correct exception type + matching token + `IsCancellationRequested == true`. Use `ct.ThrowIfCancellationRequested()` and pass the same `ct` to the task-creating API.
- **`Task.FromResult` and `Task.CompletedTask` are born in `RanToCompletion`.** Useful for implementing async interfaces with synchronous fast paths.
- **`task.Exception` is always `AggregateException`** when set, even for a single inner exception.
- **`Task.FromResult` can return cached singletons** for some `T` and some values (per docs, "Starting in .NET 6, for some `TResult` types and some result values, this method may return a cached singleton object rather than allocating a new object").

## Useful Links

- [`TaskStatus` enum — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskstatus) — verbatim descriptions of all 8 values
- [`Task.IsCompletedSuccessfully` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.iscompletedsuccessfully)
- [`Task.Wait` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.wait) — `AggregateException` wrapping spec
- [`Task.FromResult` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.fromresult) — born `RanToCompletion`, .NET 6 caching note
- [`Task.CompletedTask` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.completedtask)
- [Task Cancellation — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-cancellation) — definitive source for the `Canceled` vs `Faulted` rules

---

[← Previous: Concurrent Collections](10-concurrent-collections.md) | [Back to index](README.md)
