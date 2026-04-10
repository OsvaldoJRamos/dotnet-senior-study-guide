# Task, async/await and PLINQ

## 3. Task with async/await

Ideal for **asynchronous concurrency**, but can also be used for parallelism with `Task.WhenAll`:

```csharp
var tarefas = new[]
{
    Task.Run(() => Calcular(1)),
    Task.Run(() => Calcular(2)),
    Task.Run(() => Calcular(3))
};

await Task.WhenAll(tarefas);
```

### Practical example with I/O:

```csharp
var cepTaskList = ceps.Select(cep => new ViaCepService().GetCepAsync(cep));
var cepList = await Task.WhenAll(cepTaskList);
```

## 4. PLINQ (Parallel LINQ)

Allows parallelizing LINQ queries:

```csharp
var resultado = lista
    .AsParallel()
    .Where(x => Processar(x))
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
var result = MinhaOperacaoAsync().Result;

// CORRECT - use await
var result = await MinhaOperacaoAsync();
```

### async void — avoid it

```csharp
// WRONG - exceptions cannot be caught
async void ProcessarDados() { ... }

// CORRECT - use async Task
async Task ProcessarDados() { ... }
```

The only exception for `async void` is UI event handlers.

---

[← Previous: Parallel.ForEach and Invoke](02-parallel-foreach-invoke.md) | [Back to index](README.md) | [Next: Race Conditions →](04-race-conditions.md)
