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
var semaphore = new SemaphoreSlim(1); // only 1 thread can enter at a time (like a lock)
```

### Basic usage:
```csharp
await semaphore.WaitAsync(); // waits for permission to enter
try
{
    // Critical region: only 1 thread at a time
}
finally
{
    semaphore.Release(); // releases permission
}
```

### Example with asynchronous parallelism:

```csharp
private static readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task ProcessAsync()
{
    await _semaphore.WaitAsync();
    try
    {
        Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} entered.");
        await Task.Delay(1000); // simulates operation
    }
    finally
    {
        Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} exited.");
        _semaphore.Release();
    }
}
```

If you start multiple `ProcessAsync()` calls at the same time, only **one** will enter the critical region at a time.

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
// DOES NOT COMPILE - CS1996
lock (locker)
{
    await MethodAsync();
}
```

`await` inside a `lock` block is a **C# compile error (CS1996)** — not a runtime deadlock. The reason the compiler forbids it: `lock` uses `Monitor`, which is **thread-affine** (only the acquiring thread may release it). After an `await`, the continuation could resume on a different thread and fail to release the lock. `SemaphoreSlim` is the async-friendly alternative because it is **not** thread-affine — any thread may call `Release`.

### Correct with SemaphoreSlim:

```csharp
await _semaphore.WaitAsync();
try
{
    await MethodAsync();
}
finally
{
    _semaphore.Release();
}
```

## Pitfalls

1. **Not reentrant.** Unlike `lock` (which a thread can re-enter on the same object), `SemaphoreSlim` will **deadlock** if the same logical flow calls `WaitAsync()` twice without releasing in between. Design your code so the critical section is entered once.
2. **Always release in `finally`.** If the critical section throws, you must still `Release()` — otherwise the semaphore is leaked and subsequent callers block forever. Use `try/finally`, or an `IDisposable` wrapper that releases on dispose.
3. **`WaitAsync` accepts a `CancellationToken` and a timeout.** Prefer `await semaphore.WaitAsync(timeout, cancellationToken)` in request-scoped code so a cancelled request (or a stuck holder) does not pile up waiters indefinitely.

## Conclusion

`SemaphoreSlim` is ideal for:
- **Avoiding race conditions** in asynchronous code
- **Controlling concurrency** without freezing the application
- **Avoiding deadlocks** by replacing `lock` in `async` methods

---

[← Previous: Deadlocks](05-deadlocks.md) | [Back to index](README.md) | [Next: Locks in Depth →](07-locks-in-depth.md)
