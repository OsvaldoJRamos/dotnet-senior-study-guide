# Task, async/await and PLINQ

## 3. Task with async/await

Ideal for **asynchronous concurrency**, but can also be used for parallelism with `Task.WhenAll`:

```csharp
var tasks = new[]
{
    Task.Run(() => Calculate(1)),
    Task.Run(() => Calculate(2)),
    Task.Run(() => Calculate(3))
};

await Task.WhenAll(tasks);
```

### Practical example with I/O:

```csharp
var cepTaskList = ceps.Select(cep => new ViaCepService().GetCepAsync(cep));
var cepList = await Task.WhenAll(cepTaskList);
```

## Why `await` frees a thread for I/O but not pure CPU

`await` does not magically suspend anything. It frees the calling thread only when the awaited `Task` depends on something **outside a managed thread** to complete.

- **I/O-bound work** — the wait is handled by the kernel / network stack / disk controller. Once the operation is in flight, no managed thread needs to sit on it. The state machine registers a continuation and the thread returns to the pool. When the OS signals completion, some thread (not necessarily the original) picks up the continuation.
- **CPU-bound work without `Task.Run`** — there is no external wait: executing the code *is* the work. Whatever thread called the method is the thread that runs it. `await` has nothing to release.
- **CPU-bound work inside `Task.Run`** — the work is explicitly moved onto a pool thread, so the caller now has something external to observe and `await` can release it.

One-line rule: `await` frees the caller when the work is done by something other than a managed thread (kernel, hardware, or a different pool thread). Otherwise someone still has to run the code.

> The async docs state it directly: "The `async` and `await` keywords don't cause extra threads to be created. Async methods don't require multithreading because an async method doesn't run on its own thread." Concurrency comes from the wait being delegated — not from spinning up threads.

## 4. PLINQ (Parallel LINQ)

Allows parallelizing LINQ queries:

```csharp
var result = list
    .AsParallel()
    .Where(x => Process(x))
    .ToList();
```

## Task.Run vs async/await — when to use each one

| Scenario | Use |
|---|---|
| Heavy CPU-bound work | `Task.Run` |
| I/O calls (API, database, file) | `async/await` directly (without `Task.Run`) |
| Multiple simultaneous I/O operations | `Task.WhenAll` |
| Heavy query on a large collection | `PLINQ` |

### Library rule: keep `Task.Run` in the caller

Don't hide `Task.Run` inside library methods. It takes the decision away from the consumer:

- If the caller is already on a background thread, you double the hop for nothing.
- If the caller needed UI-thread affinity for what follows, you silently break it.
- If the work turns out to be cheap, you paid for a thread switch with no gain.

Expose synchronous work as synchronous and genuine async work as async. Let the caller decide when to offload with `Task.Run`.

## Common mistakes

### Do not use .Result or .Wait()

```csharp
// WRONG - can cause deadlock on classic ASP.NET / WinForms / WPF
var result = MyOperationAsync().Result;

// CORRECT - use await
var result = await MyOperationAsync();
```

The deadlock happens **only when a `SynchronizationContext` is captured** — i.e., classic ASP.NET (System.Web), WinForms, WPF. It does **not** happen in ASP.NET Core or console apps (neither has a sync context by default).

**Mechanism:** inside `MyOperationAsync`, an `await` captures the current sync context. When the awaited operation completes, the continuation is posted back to that context to resume. But the caller is blocking that same context with `.Result` / `.Wait()` — so the continuation never runs, and the caller waits forever.

### async void — avoid it

```csharp
// WRONG
async void ProcessData() { ... }

// CORRECT - use async Task
async Task ProcessData() { ... }
```

Three real problems with `async void`:

1. **Unhandled exceptions crash the process** — they are raised on the captured `SynchronizationContext` (or `ThreadPool`), bypassing the caller's `try/catch`.
2. **Cannot be `await`ed** — there is no `Task` to observe completion, success, or failure.
3. **Breaks composition** — no way to `Task.WhenAll`, chain, cancel, or test reliably.

The only legitimate use is **UI event handlers** (the framework signature requires `void`).

### ConfigureAwait(false)

By default, `await` captures the current `SynchronizationContext` and resumes the continuation on it. `ConfigureAwait(false)` tells the runtime **not to capture** the context — the continuation runs on any thread pool thread.

```csharp
var data = await httpClient.GetStringAsync(url).ConfigureAwait(false);
```

- **Library code:** use `ConfigureAwait(false)` everywhere. You don't know who calls you, and forcing resumption on a caller's sync context causes perf loss and the deadlock above.
- **ASP.NET Core / console apps:** it's a **no-op** (no sync context to capture), but harmless.
- **App code in WinForms/WPF:** do NOT use it when you need to touch UI after the await.

### Fire-and-forget

An unawaited `Task` is **fire-and-forget** — nobody is watching for its completion or its exceptions.

```csharp
// BAD — if DoWorkAsync throws, no one catches it here
DoWorkAsync();
```

Three problems:

1. **Exceptions are easy to lose.** A faulted task whose exception is never observed raises `TaskScheduler.UnobservedTaskException` only when the task is finalized by the GC — the delay is unpredictable and the event is easy to miss in production.
2. **Shutdown can cut it mid-flight.** No one knows when it finished, so graceful shutdown may terminate the process before the work completes.
3. **`async void` is the worst variant.** Exceptions thrown from an `async void` method are re-raised on the `SynchronizationContext` that was active when the method started, bypassing the caller's `try`/`catch`. On a UI or classic ASP.NET context this can crash the app.

If fire-and-forget is genuinely what you want, wrap it and log:

```csharp
_ = Task.Run(async () =>
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { _logger.LogError(ex, "Background work failed"); }
});
```

Better: enqueue into a `Channel<T>` or a hosted `BackgroundService` that owns lifecycle and error handling.

## Realistic hybrid example

A single request often mixes both workloads. `await` each kind with the tool that fits it:

```csharp
public async Task<Report> GenerateReportAsync(int userId, CancellationToken ct)
{
    var data      = await _db.GetUserDataAsync(userId, ct);         // I/O — thread released
    var processed = await Task.Run(() => HeavyTransform(data), ct); // CPU — runs on a pool thread
    await _storage.SaveAsync(processed, ct);                        // I/O — thread released
    return new Report(processed);
}
```

Three `await`s, three distinct reasons to suspend. Only the middle one actually keeps a thread busy executing — the two I/O calls delegate the wait to the OS and return their thread to the pool in the meantime.

---

[← Previous: Parallel.ForEach and Invoke](02-parallel-foreach-invoke.md) | [Back to index](README.md) | [Next: Race Conditions →](04-race-conditions.md)
