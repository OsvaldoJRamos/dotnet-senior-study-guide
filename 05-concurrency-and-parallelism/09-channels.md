# Channels

`System.Threading.Channels` is the modern producer/consumer primitive in .NET. Conceptually a queue with a write end and a read end, designed around `await` and `IAsyncEnumerable<T>`. It replaces ad-hoc combinations of `ConcurrentQueue<T>`, manual `SemaphoreSlim` signaling, or `BlockingCollection<T>` for in-process pipelines.

## Why a new primitive

| Need | Pre-Channels | Problem |
|---|---|---|
| Thread-safe FIFO queue | `ConcurrentQueue<T>` | No "wait for an item" semantics — consumers must poll. |
| Wait until item available | `BlockingCollection<T>` | Synchronous — `Take()` **blocks the thread**, hostile to async pipelines. |
| Bounded queue with backpressure | `BlockingCollection<T>(capacity)` | Same blocking problem. |
| Async-friendly producer/consumer | (build it yourself) | Easy to get wrong. |

Channels give you **all of the above, async-first**. `WriteAsync` returns a `ValueTask` that completes when space is available; `ReadAsync` returns a `ValueTask<T>` that completes when an item is available; `ReadAllAsync` returns an `IAsyncEnumerable<T>` you can drive with `await foreach`.

## The basic shape

```csharp
using System.Threading.Channels;

Channel<Order> channel = Channel.CreateUnbounded<Order>();

// Producer
await channel.Writer.WriteAsync(order, ct);
channel.Writer.Complete();   // signal: no more items

// Consumer
await foreach (Order order in channel.Reader.ReadAllAsync(ct))
{
    await ProcessAsync(order, ct);
}
```

A channel is created from the static factory `Channel.CreateBounded<T>(...)` or `Channel.CreateUnbounded<T>(...)`. The instance exposes a `Writer` (`ChannelWriter<T>`) and a `Reader` (`ChannelReader<T>`). Producers and consumers each work against one end without seeing the other.

## Bounded vs Unbounded — and backpressure

**`CreateUnbounded<T>()`** — capacity is unlimited. Producers never wait. **Risk:** a slow consumer lets the queue grow without bound; memory blows up. Acceptable when production is naturally bounded (a finite enumeration) or when you're certain consumers keep up.

**`CreateBounded<T>(int)`** — fixed maximum capacity. The interesting question is **what happens when a producer writes to a full channel**, controlled by `BoundedChannelFullMode`:

| Mode | Behavior |
|---|---|
| `Wait` (default) | "Waits for space to be available in order to complete the write operation." |
| `DropNewest` | "Removes and ignores the **newest** item in the channel in order to make room for the item being written." |
| `DropOldest` | "Removes and ignores the **oldest** item in the channel in order to make room for the item being written." |
| `DropWrite` | "Drops the **item being written**." |

(Quotes from `BoundedChannelFullMode` MS Learn docs.)

`Wait` is what gives you **backpressure**: the producer's `WriteAsync` simply doesn't complete until there's space. The producer is naturally throttled to the consumer's rate, and the queue size is bounded by capacity. Without backpressure, a slow consumer becomes a dam that builds memory pressure upstream until something OOMs.

The `Drop*` modes are for telemetry-style flows where losing data is preferable to blocking the producer (e.g., metric samples, log batching).

```csharp
var channel = Channel.CreateBounded<Telemetry>(new BoundedChannelOptions(capacity: 1000)
{
    FullMode = BoundedChannelFullMode.DropOldest,  // prefer fresh data
    SingleReader = true,
    SingleWriter = false
});
```

## Single-reader / single-writer optimizations

`ChannelOptions` exposes `SingleReader` and `SingleWriter` flags:

> "`SingleReader`: `true` readers from the channel guarantee that there will only ever be at most one read operation at a time; `false` if no such constraint is guaranteed."
>
> "`SingleWriter`: `true` if writers to the channel guarantee that there will only ever be at most one write operation at a time; `false` if no such constraint is guaranteed." — `ChannelOptions` MS Learn

These are **promises you make to the runtime**, not guards it enforces. When you set them, the implementation switches to specialized fast paths. From the official Stephen Toub article introducing channels:

> "When `SingleReader` is true, the implementation not only avoids locks when reading, it also avoids interlocked operations when reading, significantly reducing the overheads involved in consuming from the channel."

Set them whenever you can — most pipelines have one consumer and a small fixed set of producers. Lying about it (multiple readers despite `SingleReader = true`) is undefined behavior.

## `AllowSynchronousContinuations` — power tool, sharp edge

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    AllowSynchronousContinuations = true   // ⚠️
});
```

By default, when a producer writes data and there's a waiting consumer, the consumer's continuation is queued for asynchronous execution. With `AllowSynchronousContinuations = true`, the producer can run the consumer's continuation **on its own thread, inline** — saving a thread hop.

The trade-off, from the official docs:

> "if you were holding a lock while calling `TryWrite`, with `AllowSynchronousContinuations` set to `true`, you might end up invoking the callback while holding your lock, which ... could end up observing some broken invariants your lock was trying to maintain."

In practice: leave it `false` unless you have measured a bottleneck and know exactly what code the consumer runs. The latency win is small; the foot-gun is large.

## Reading patterns

Three idiomatic ways to consume:

```csharp
// 1. Streaming — most common
await foreach (var item in channel.Reader.ReadAllAsync(ct))
    await Process(item, ct);

// 2. Pull one at a time
while (await channel.Reader.WaitToReadAsync(ct))
{
    while (channel.Reader.TryRead(out var item))
        await Process(item, ct);
}

// 3. Single read with explicit handling
T item = await channel.Reader.ReadAsync(ct);
```

Pattern 1 is the default. Pattern 2 is useful when you want batch behavior (drain everything available, then process). Pattern 3 is for one-shot scenarios.

`channel.Reader.Completion` is a `Task` that completes when the writer has called `Complete()` *and* the reader has consumed everything — the right thing to `await` if you want to wait for an entire pipeline to drain.

## Closing the channel

From the official `ChannelWriter.Complete` docs: *"Mark the channel as being complete, meaning no more items will be written to it."* Existing consumers drain the queue, then `ReadAllAsync` exits its loop and `Reader.Completion` completes successfully. Calling `Complete()` again throws `InvalidOperationException` ("The channel has already been marked as complete").

`channel.Writer.Complete(exception)` propagates an exception — *"Optional Exception indicating a failure that's causing the channel to complete"* — so consumers see it on the next read. Use it when the producer hit a fatal error and the pipeline should fault.

Once the channel is complete, further write attempts fail. The `System.Threading.Channels` namespace ships a `ChannelClosedException` for this case (the MS Learn `WriteAsync` page itself only documents `OperationCanceledException`, so wrap producer code defensively if writing can race with shutdown).

## Pairing with `IAsyncEnumerable<T>`

`ReadAllAsync` returns `IAsyncEnumerable<T>`, which means a channel reader composes seamlessly with LINQ-like async streams:

```csharp
async IAsyncEnumerable<EnrichedOrder> EnrichAsync(
    Channel<Order> source,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var order in source.Reader.ReadAllAsync(ct))
        yield return await EnrichAsync(order, ct);
}
```

Each `MoveNextAsync` returns a `ValueTask<bool>` (no allocation on synchronous completion), and the pattern composes naturally with channels acting as the transport between stages.

## Prioritized channels (.NET 9+)

`Channel.CreateUnboundedPrioritized<T>()` returns a channel where reads come back in order determined by `IComparer<T>`. Useful when the consumer should preempt low-priority work whenever a high-priority item arrives. Bounded prioritized channels are not provided.

## When to use what

| Workload | Pick |
|---|---|
| In-process producer/consumer with possible async waiting | `Channel<T>` |
| Sync-only producer/consumer in legacy code | `BlockingCollection<T>` |
| Thread-safe queue with **no** wait semantics | `ConcurrentQueue<T>` |
| Cross-process / cross-machine messaging | broker (RabbitMQ, Kafka, Service Bus) — channels are in-process only |
| Complex multi-stage pipelines with batching, broadcasting | TPL Dataflow (`ActionBlock`, `TransformBlock`, etc.) |

## Senior-interview gotchas

- **`Wait` mode is what gives you backpressure** — not "magic", just `WriteAsync` parking the producer until space frees up.
- **Drop modes can silently lose data.** Use them for telemetry, never for orders.
- **`SingleReader`/`SingleWriter` are promises, not guards.** They unlock a faster code path with no locks/interlocked on the hot path. Lying corrupts state.
- **`AllowSynchronousContinuations` runs the consumer on the producer's thread.** Don't enable it while holding a lock or doing UI work.
- **`Reader.Completion` is the "pipeline drained" signal.** `Writer.Complete()` is "no more input"; the pipeline is done only when `Reader.Completion` finishes.
- **Channels are in-process.** They don't survive process restart. If you need durability, you need a real broker.
- **Writing after `Complete()` fails** — the runtime ships `ChannelClosedException` for this in `System.Threading.Channels`. The official `WriteAsync` page only documents `OperationCanceledException`, so wrap producer code defensively if writing can race with shutdown.

## Useful Links

- [`Channel` class — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.channels.channel) — factory methods (`CreateBounded`, `CreateUnbounded`, `CreateUnboundedPrioritized`)
- [`BoundedChannelFullMode` enum — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.channels.boundedchannelfullmode) — `Wait` / `DropNewest` / `DropOldest` / `DropWrite`
- [`ChannelOptions` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.channels.channeloptions) — `SingleReader`, `SingleWriter`, `AllowSynchronousContinuations`
- [An introduction to System.Threading.Channels — .NET Blog (Stephen Toub)](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/) — design rationale, performance characteristics, idioms

---

[← Previous: The Thread Pool](08-thread-pool.md) | [Back to index](README.md) | [Next: Concurrent Collections →](10-concurrent-collections.md)
