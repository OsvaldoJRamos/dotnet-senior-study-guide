# Async and Memory

`async`/`await` looks simple in source, but the compiler emits a state machine and the runtime allocates objects under the hood. Understanding what costs what — and when `ValueTask` is worth the extra rules — is part of the senior bar.

## What `async` actually allocates

Every `async` method becomes a compiler-generated state machine struct that implements `IAsyncStateMachine`. Behavior:

1. The state machine starts on the stack.
2. If every `await` completes synchronously, the method returns without ever boxing the state machine. **Cheap path.**
3. If any `await` does not complete synchronously, the state machine is **boxed onto the heap** so it can survive while the operation is pending. The continuation is registered, the thread is released. Then a `Task<T>` (or `ValueTask<T>`) is allocated to represent the pending work.

So a method that frequently awaits things that already completed (cache hit, data already in buffer) pays nothing extra. A method that takes the slow path pays for: state-machine boxing + `Task<T>` allocation + possible `ExecutionContext` capture.

## `Task` vs `ValueTask` — the trade-off

`Task<T>` is a class — every async method that takes the slow path allocates one on the heap. `ValueTask<T>` is a struct that can wrap either a synchronously-available value (no allocation) or a `Task<T>` (when the slow path is unavoidable).

```csharp
public ValueTask<Product?> GetByIdAsync(int id)
{
    if (_cache.TryGetValue(id, out Product? cached))
        return new ValueTask<Product?>(cached);    // ZERO alloc on cache hit

    return LoadAndCacheAsync(id);                  // Task<Product?> only on miss
}

private async ValueTask<Product?> LoadAndCacheAsync(int id)
{
    Product product = await _db.LoadAsync(id);
    _cache.Set(id, product);
    return product;
}
```

The outer method is **not `async`** — it returns a `ValueTask<T>` directly. That's how you get the zero-allocation cache hit path. Only the slow-path helper is `async`.

### The fine print (per the official `ValueTask<TResult>` docs)

The Microsoft API docs are explicit about what you must not do:

- **Awaiting the instance multiple times.**
- **Calling `AsTask` multiple times.**
- **Using `.Result` or `.GetAwaiter().GetResult()` when the operation hasn't yet completed, or using them multiple times.**
- **Using more than one of these techniques to consume the instance.**

> "If you do any of the above, the results are undefined." — MS Learn

### `ValueTask` is bigger than `Task`

A subtle point worth remembering: `ValueTask<T>` is a struct with **multiple fields**, while `Task<T>` is a reference type passed by **a single pointer**. From the official docs:

> "while a `ValueTask<TResult>` can help avoid an allocation in the case where the successful result is available synchronously, it also contains multiple fields, whereas a `Task<TResult>` as a reference type is a single field. This means that returning a `ValueTask<TResult>` from a method results in copying more data."

In other words: `ValueTask` saves **heap allocation** at the cost of **larger value-type copies**. If the slow path dominates, returning a regular `Task<T>` is cheaper overall.

### Microsoft's official recommendation

> "the default choice for any asynchronous method should be to return a `Task` or `Task<TResult>`. Only if performance analysis proves it worthwhile should a `ValueTask<TResult>` be used instead of a `Task<TResult>`." — MS Learn

Don't reach for `ValueTask` everywhere. Use it where profiling shows hot synchronous-completion paths.

## `Memory<T>` and async

`Span<T>` cannot cross `await` (it's a `ref struct`). When you need a buffer that flows across an asynchronous boundary, use `Memory<T>` and convert to `Span<T>` for the synchronous slices of work.

```csharp
public async Task ProcessAsync(Stream stream, CancellationToken ct)
{
    using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(8192);
    Memory<byte> buffer = owner.Memory;

    int read = await stream.ReadAsync(buffer, ct);   // Memory crosses await fine

    // For the synchronous chunk, switch to Span:
    Span<byte> span = buffer.Span[..read];
    Process(span);

    await stream.WriteAsync(buffer[..read], ct);
}
```

Modern stream and socket APIs accept `Memory<byte>` and `ReadOnlyMemory<byte>` directly:

- `Stream.ReadAsync(Memory<byte>, CancellationToken)`
- `Stream.WriteAsync(ReadOnlyMemory<byte>, CancellationToken)`
- `Socket.ReceiveAsync(Memory<byte>, ...)`
- `PipeReader.ReadAsync(...)`

Combining `MemoryPool<T>` with `using` gives you automatic buffer return and zero-allocation parsing in one idiomatic block.

## `ConfigureAwait(false)`

Every `await` captures the current `SynchronizationContext` (or, if null, the current `TaskScheduler`) and resumes the continuation there. `ConfigureAwait(false)` skips that capture-and-restore step.

When it matters:

- **WPF / WinForms:** there *is* a `SynchronizationContext` that pins continuations to the UI thread. Without `ConfigureAwait(false)`, every `await` round-trips through the UI thread, which is bad for throughput and a deadlock risk for code that blocks on tasks.
- **ASP.NET Core:** there is **no** `SynchronizationContext` by default. The behavioral effect of `ConfigureAwait(false)` is therefore minimal — but as Stephen Toub clarified in the official ConfigureAwait FAQ, the runtime still has to read `SynchronizationContext.Current` and `TaskScheduler.Current` on every await, so a tiny per-call cost remains. For application code, leave the call out and keep the noise down. For library code consumed in unknown contexts, keep `ConfigureAwait(false)`.

## `async void`

`async void` exists for one reason: event handlers (button click, etc.) where the framework expects a `void`-returning delegate. Outside that single case, **prefer `Task` or `ValueTask`**.

Why:

- Exceptions thrown from `async void` are re-raised on the captured `SynchronizationContext`, **bypassing the caller's `try/catch`**. In ASP.NET Core or a console app this typically tears down the process.
- The caller cannot `await` the result, so cannot know when the work finished or whether it failed.
- Hard to test (no `Task` to await in unit tests).

## Don't block on async with `.Result` / `.Wait()`

```csharp
public string GetData()
    => GetDataAsync().Result;   // ❌ deadlock-prone, thread-pool exhaustion risk
```

In any environment with a `SynchronizationContext` (WPF, classic ASP.NET, etc.), the await wants to resume on the captured context — but the calling thread is blocked on `.Result`, so the resume can never happen and the call deadlocks. In ASP.NET Core there is no SyncContext, but you still pay a thread-pool thread to sit and block.

Solution: be `async` all the way up. If you genuinely have a sync entry point you cannot change, the lesser evil is `GetAwaiter().GetResult()` (preserves original exception type) — but recognize you've taken on the deadlock risk.

---

[← Previous: Structs vs Classes](05-structs-vs-classes.md) | [Back to index](README.md) | [Next: JIT and Runtime →](07-jit-and-runtime.md)
