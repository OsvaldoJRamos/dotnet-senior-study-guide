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

---

[← Previous: Parallel.ForEach and Invoke](02-parallel-foreach-invoke.md) | [Back to index](README.md) | [Next: Race Conditions →](04-race-conditions.md)
