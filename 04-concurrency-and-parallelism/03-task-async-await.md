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
// WRONG - can cause deadlock
var result = MyOperationAsync().Result;

// CORRECT - use await
var result = await MyOperationAsync();
```

### async void — avoid it

```csharp
// WRONG - exceptions cannot be caught
async void ProcessData() { ... }

// CORRECT - use async Task
async Task ProcessData() { ... }
```

The only exception for `async void` is UI event handlers.

---

[← Previous: Parallel.ForEach and Invoke](02-parallel-foreach-invoke.md) | [Back to index](README.md) | [Next: Race Conditions →](04-race-conditions.md)
