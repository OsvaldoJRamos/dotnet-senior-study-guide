# SemaphoreSlim

## What is a semaphore?

A **semaphore** is a synchronization mechanism that **controls how many threads can access a shared resource at the same time**.

> Imagine a parking lot with 3 spaces: only 3 cars (threads) can enter at a time. Anyone who arrives after that must wait for someone to leave.

## What is SemaphoreSlim?

`SemaphoreSlim` is a lightweight, modern version of the semaphore in C#.
- Supports asynchronous operations with `await`, without blocking the thread like the traditional `lock`.

## How it works

### Creation:
```csharp
var semaforo = new SemaphoreSlim(1); // only 1 thread can enter at a time (like a lock)
```

### Basic usage:
```csharp
await semaforo.WaitAsync(); // waits for permission to enter
try
{
    // Critical region: only 1 thread at a time
}
finally
{
    semaforo.Release(); // releases permission
}
```

### Example with asynchronous parallelism:

```csharp
private static readonly SemaphoreSlim _semaforo = new(1, 1);

public async Task ProcessarAsync()
{
    await _semaforo.WaitAsync();
    try
    {
        Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} entrou.");
        await Task.Delay(1000); // simula operação
    }
    finally
    {
        Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} saiu.");
        _semaforo.Release();
    }
}
```

If you start multiple `ProcessarAsync()` calls at the same time, only **one** will enter the critical region at a time.

## When to use SemaphoreSlim

| Situation | Use SemaphoreSlim? |
|---|---|
| Multiple async tasks accessing a shared resource (e.g., file, cache, list) | Yes |
| You need to **limit concurrency** (e.g., maximum 3 simultaneous API calls) | Yes |
| You need asynchronous synchronization (avoid `lock` with `await`) | Yes |
| You are in synchronous code (no `async`) | Prefer `lock` |
| You want to avoid deadlocks and freezes caused by `.Result` / `.Wait()` | Yes, because SemaphoreSlim with `await` does not block the thread |

## What to avoid

```csharp
// WRONG - lock with await causes deadlock
lock (locker)
{
    await MetodoAsync(); // DO NOT do this
}
```

This causes a **deadlock**, because `lock` blocks the thread while `await` waits.

### Correct with SemaphoreSlim:

```csharp
await _semaforo.WaitAsync();
try
{
    await MetodoAsync();
}
finally
{
    _semaforo.Release();
}
```

## Conclusion

`SemaphoreSlim` is ideal for:
- **Avoiding race conditions** in asynchronous code
- **Controlling concurrency** without freezing the application
- **Avoiding deadlocks** by replacing `lock` in `async` methods

---

[← Previous: Deadlocks](05-deadlocks.md) | [Back to index](README.md)
