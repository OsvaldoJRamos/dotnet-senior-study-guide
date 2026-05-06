# Memory and Performance

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between stack and heap memory?

<details>
<summary>Reveal answer</summary>

| Aspect | Stack | Heap |
|--------|-------|------|
| Allocation | Automatic (push/pop) | Manual (`new`) — managed by GC |
| Speed | Very fast (pointer increment) | Slower (fragmentation, GC) |
| Lifetime | Scoped to method call | Until no references remain |
| Size | Small (default ~1MB per thread) | Large (limited by RAM) |
| What lives here | Value types (locals), return addresses | Reference types, boxed values |

**Important nuance**: value types declared as fields of a class live on the heap (inside the class object). "Value types live on the stack" is a simplification.

Deep dive: [Stack and Heap](../05-memory-and-performance/01-stack-and-heap.md)

</details>

---

### 2. How does the .NET Garbage Collector work? What are generations?

<details>
<summary>Reveal answer</summary>

The GC uses a **generational model** based on the observation that most objects die young:

| Generation | Contains | Collection frequency |
|------------|----------|---------------------|
| **Gen 0** | Newly allocated objects | Very frequent |
| **Gen 1** | Survived one Gen 0 collection | Moderate |
| **Gen 2** | Long-lived objects | Rare (full GC) |

Collection steps: **Mark** (find live objects via root references) → **Sweep** (reclaim dead objects) → **Compact** (defragment memory).

The **Large Object Heap (LOH)** holds objects >= 85KB. It is collected with Gen 2 but **not compacted by default** (can cause fragmentation). Enable compaction with `GCSettings.LargeObjectHeapCompactionMode`.

Deep dive: [Garbage Collector](../05-memory-and-performance/02-garbage-collector.md)

</details>

---

### 3. What causes memory leaks in a managed language like C#?

<details>
<summary>Reveal answer</summary>

Even though .NET has a GC, memory leaks occur when objects **remain referenced** longer than intended:

1. **Event handlers not unsubscribed**: the publisher holds a reference to the subscriber.
2. **Static collections**: objects added to a `static List<T>` live forever.
3. **Closures capturing variables**: a lambda captures a local variable, keeping the entire containing object alive.
4. **Undisposed resources**: `HttpClient`, `DbConnection`, `Timer` not disposed.
5. **Caching without eviction**: unbounded `Dictionary` acting as a cache.
6. **Timers**: `System.Timers.Timer` roots itself via GCHandle while active.

**How to diagnose**: dotnet-dump, dotnet-counters, Visual Studio Diagnostic Tools, or JetBrains dotMemory.

Deep dive: [Memory Leaks](../05-memory-and-performance/04-memory-leak.md)

</details>

---

### 4. When should you use a struct instead of a class?

<details>
<summary>Reveal answer</summary>

Use a **struct** when ALL of these conditions are met:
- The type is **small** (Microsoft guideline: under 16 bytes)
- It is **immutable** (or logically represents a single value)
- It is **short-lived** and allocated frequently
- It does **not** need to participate in inheritance

Examples: `Point`, `DateTime`, `Guid`, `Color`, `Range`.

**Why not always?** Structs are copied on assignment and method calls. Large structs cause performance problems due to excessive copying. They also cannot be `null` (without `Nullable<T>`) and cannot participate in polymorphism.

Deep dive: [Structs vs Classes](../05-memory-and-performance/05-structs-vs-classes.md)

</details>

---

### 5. What are `Span<T>` and `Memory<T>`? When would you use each?

<details>
<summary>Reveal answer</summary>

Both provide **safe, bounds-checked slicing** of contiguous memory without allocating new arrays.

| Type | Stack-safe | Can be stored on heap | Async-friendly |
|------|-----------|----------------------|---------------|
| `Span<T>` | Yes (`ref struct`) | No | No |
| `Memory<T>` | Yes | Yes | Yes |

```csharp
// Span — zero-allocation substring
ReadOnlySpan<char> span = "Hello, World!".AsSpan();
ReadOnlySpan<char> hello = span.Slice(0, 5); // no new string allocated

// Memory — when you need to pass to async methods
Memory<byte> buffer = new byte[1024];
await stream.ReadAsync(buffer);
```

Use `Span<T>` in synchronous, hot-path code. Use `Memory<T>` when the data must survive across `await` boundaries or be stored in a field.

Deep dive: [Memory Optimization](../05-memory-and-performance/03-memory-optimization.md)

</details>

---

### 6. What is object pooling and when does it help?

<details>
<summary>Reveal answer</summary>

Object pooling **reuses objects** instead of creating and garbage-collecting them repeatedly. It reduces GC pressure for objects that are expensive to create or allocated at high frequency.

```csharp
// Built-in pool for arrays
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(1024);
try
{
    // use buffer
}
finally
{
    pool.Return(buffer);
}

// ObjectPool<T> from Microsoft.Extensions.ObjectPool
var policy = new DefaultPooledObjectPolicy<StringBuilder>();
var pool = new DefaultObjectPool<StringBuilder>(policy);
var sb = pool.Get();
// use sb
pool.Return(sb);
```

Common use cases: byte buffers, `StringBuilder`, database connections (connection pooling), `HttpClient` (via `IHttpClientFactory`).

**Caution**: pooling adds complexity. Only use it when profiling shows GC pressure or allocation cost is a bottleneck.

Deep dive: [Memory Optimization](../05-memory-and-performance/03-memory-optimization.md)

</details>

---

### 7. What is string interning? How does it affect memory?

<details>
<summary>Reveal answer</summary>

String interning is a process where the runtime maintains a **pool of unique strings**. All string **literals** in your code are automatically interned — if two literals have the same content, they share the same object in memory.

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // True — same interned object

string c = new string("hello".ToCharArray());
Console.WriteLine(ReferenceEquals(a, c)); // False — different object

string d = string.Intern(c);
Console.WriteLine(ReferenceEquals(a, d)); // True — now interned
```

**Benefits**: reduces memory when the same string values appear many times.

**Risks**: interned strings live **forever** (they are rooted by the intern table). Do not intern user-generated or dynamic strings — this is a memory leak.

</details>

---

### 8. How do you identify and fix a memory leak in a .NET application?

<details>
<summary>Reveal answer</summary>

**Identify:**
1. Monitor with `dotnet-counters` — watch GC heap size, Gen 2 collection count
2. Take heap snapshots with `dotnet-dump` or Visual Studio Diagnostic Tools
3. Compare two snapshots — what types are growing?
4. Use `dotnet-gcdump` for lightweight GC heap analysis

**Common patterns and fixes:**

| Pattern | Fix |
|---------|-----|
| Event handler leak | Unsubscribe in `Dispose()` or use weak events |
| Static collection growth | Add eviction (LRU cache) or use `ConditionalWeakTable` |
| Undisposed `IDisposable` | Use `using` statement; enable CA2000 analyzer |
| Timer keeping objects alive | Dispose timer when done; use `PeriodicTimer` in .NET 6+ |
| Closure capturing `this` | Extract the needed value into a local variable |

Deep dive: [Memory Leaks](../05-memory-and-performance/04-memory-leak.md)

</details>

---

### 9. What is the Large Object Heap (LOH) and why does it matter?

<details>
<summary>Reveal answer</summary>

Objects >= **85,000 bytes** (roughly 85KB) are allocated on the **Large Object Heap** instead of the Small Object Heap (SOH).

Key differences:
- LOH is collected only during **Gen 2 collections** (expensive, infrequent)
- LOH is **not compacted by default** — leads to fragmentation
- Frequent large allocations cause **LOH fragmentation** and eventual `OutOfMemoryException` even with available RAM

**Mitigations:**
- Use `ArrayPool<T>` to rent large arrays instead of allocating
- Enable LOH compaction: `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce`
- Avoid repeated allocation of large byte arrays (use pooling)

Deep dive: [Garbage Collector](../05-memory-and-performance/02-garbage-collector.md)

</details>

---

### 10. What are some practical techniques to reduce GC pressure in a hot path?

<details>
<summary>Reveal answer</summary>

1. **Use `Span<T>` / `stackalloc`** — avoid heap allocations for temporary buffers
2. **Object pooling** (`ArrayPool<T>`, `ObjectPool<T>`) — reuse instead of allocate
3. **Use structs** for small, short-lived data — stays on the stack
4. **Avoid boxing** — use generic collections, not `object`-based ones
5. **`StringBuilder`** instead of string concatenation in loops
6. **Cache delegates** — `static` lambdas (C# 9+) to avoid closure allocations
7. **Avoid LINQ in hot paths** — LINQ allocates iterators and delegates
8. **Use `ValueTask<T>`** instead of `Task<T>` for methods that often complete synchronously
9. **String interpolation handlers** (C# 10+) — avoid intermediate string allocations

```csharp
// Closure allocation
list.ForEach(x => Process(x, threshold)); // captures threshold

// No allocation — static lambda (cannot capture outer variables)
list.ForEach(static x => Console.WriteLine(x));
```

Deep dive: [Memory Optimization](../05-memory-and-performance/03-memory-optimization.md)

</details>

---

### 11. How does `ValueTask<T>` differ from `Task<T>` and when should you use it?

<details>
<summary>Reveal answer</summary>

`Task<T>` always allocates an object on the heap. `ValueTask<T>` is a **struct** that wraps either a `T` value (synchronous completion) or a `Task<T>` (asynchronous completion), avoiding allocation in the synchronous case.

```csharp
// Good candidate for ValueTask — often returns from cache
public ValueTask<User> GetUserAsync(int id)
{
    if (_cache.TryGetValue(id, out var user))
        return new ValueTask<User>(user); // no allocation

    return new ValueTask<User>(LoadFromDbAsync(id)); // wraps Task
}
```

**Rules:**
- Never await a `ValueTask` more than once
- Never use `.Result` or `.GetAwaiter().GetResult()` before it completes
- Use `ValueTask` when the method frequently completes synchronously

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 12. What is the Large Object Heap (LOH) and why is it different?

<details>
<summary>Reveal answer</summary>

Objects **≥ 85,000 bytes** (arrays, strings, large buffers) go on the **Large Object Heap** instead of the regular generational heap. LOH rules differ:

- LOH is **not compacted by default** — GC marks free segments in place, which fragments the heap over time.
- LOH collections are **Gen2 collections**, so they trigger full GCs (expensive).
- You can force one-time compaction by setting `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce` before the next GC, but this is a hammer.

Consequences:
- Frequent allocation of large arrays leads to LOH fragmentation and Gen2 pressure.
- Prefer **`ArrayPool<T>.Shared`** or **`MemoryPool<T>`** to reuse large buffers instead of allocating fresh ones.
- Pre-size collections when you know the final size to avoid doubling reallocations that graduate to LOH.

Deep dive: [Garbage Collector](../05-memory-and-performance/02-garbage-collector.md)

</details>

---

### 13. When would you use `ArrayPool<T>`, and what's the risk?

<details>
<summary>Reveal answer</summary>

`ArrayPool<T>.Shared.Rent(minLength)` returns a reusable array from a thread-safe pool, avoiding allocation on hot paths. Call `Return(array)` when done.

Use it for:
- Short-lived buffers in hot paths (parsers, serializers, network I/O).
- Buffers that would otherwise hit the LOH (≥ 85,000 bytes).

Risks:
- **Forgetting to return** — the array leaks from the pool (GC still collects it, but pool efficiency drops).
- **Returning the wrong array** — caller keeps a reference after `Return`, then reads/writes it. Another caller rents it and sees corrupted data. Always clear your local reference after return.
- **Rented arrays can be longer than requested** — the pool gives you the next power-of-two size. Track the logical length separately.
- **Clear-on-return**: call `Return(array, clearArray: true)` if the buffer held sensitive data (tokens, PII) to avoid leaking it to the next renter.

Deep dive: [Memory Optimization](../05-memory-and-performance/03-memory-optimization.md)

</details>

---

### 14. What is the difference between Server GC and Workstation GC?

<details>
<summary>Reveal answer</summary>

The .NET runtime offers two GC flavors:

| Aspect | Workstation GC | Server GC |
|--------|----------------|-----------|
| Default | Client apps, short-running processes | ASP.NET Core, server apps (default in many templates) |
| Heap layout | One heap | One heap **per logical core** |
| Pause model | Optimized for responsiveness | Optimized for throughput; pauses can be longer but rarer |
| Threads | Single GC thread (background) | One GC thread per core |
| Memory | Lower footprint | Higher footprint (one heap per core) |

Toggle via `<ServerGarbageCollection>true</ServerGarbageCollection>` in the `.csproj` (or `DOTNET_gcServer=1`). Pair with `<ConcurrentGarbageCollection>` (background GC) for lower pause times.

Rule of thumb: use Server GC for backend services with many cores; Workstation for desktop apps or small processes where memory matters more than throughput.

Deep dive: [Garbage Collector](../05-memory-and-performance/02-garbage-collector.md)

</details>

---

### 15. What is boxing and unboxing? When does it happen and why should you care?

<details>
<summary>Reveal answer</summary>

- **Boxing** — wrapping a value type in an `object` reference. The value is copied onto the heap and a reference returned.
- **Unboxing** — extracting the value back from the `object`. Requires a type check; fails with `InvalidCastException` if the target type doesn't match.

```csharp
int x = 42;
object boxed = x;        // boxing — heap allocation
int y = (int)boxed;      // unboxing — type check + copy
```

Why you care: boxing causes **heap allocations** and therefore **GC pressure**. Common traps:

- Value types stored in non-generic collections (`ArrayList`, `Hashtable`) — every insert boxes.
- Calling `Equals(object)` or `ToString()` on a struct that doesn't override them — the struct is boxed to satisfy the signature.
- Capturing a value type in a closure that promotes it to a heap-allocated field.
- `string.Format(...)` / interpolation with value types in older targets (modern interpolated strings avoid boxing for built-in types).

Fixes: use generic collections (`List<int>`, `Dictionary<int, string>`), override `Equals`/`GetHashCode` on performance-critical structs, use `Span<T>` and ref struct types where appropriate.

Deep dive: [Stack and Heap](../05-memory-and-performance/01-stack-and-heap.md)

</details>

---

### 16. Where do POH (Pinned Object Heap) objects live, what version of .NET introduced it, and what's the type restriction?

<details>
<summary>Reveal answer</summary>

The POH was introduced in .NET 5. Allocate via `GC.AllocateArray<T>(length, pinned: true)`. Pinned objects sit on this dedicated heap rather than the regular SOH, so they don't disrupt heap compaction.

Per the official MS Learn `GC.AllocateArray<T>` docs: in **.NET 7 and earlier versions**, when `pinned` is `true`, `T` must not be a reference type or a type that contains object references. The reason is design-level — POH objects are not scanned for cross-generation references because by definition they cannot hold them.

POH and LOH are physically separate from SOH but **logically collected together with Gen 2**.

Deep dive: [Garbage Collector](../05-memory-and-performance/02-garbage-collector.md)

</details>

---

### 17. Workstation vs Server GC: what differs, and which is the default?

<details>
<summary>Reveal answer</summary>

| | Workstation | Server |
|---|---|---|
| GC threads | One | One per logical processor |
| Heaps | One | One per logical processor |
| Pause profile | Lower latency, lower throughput | Higher throughput, larger pauses |
| Hardware requirement | Any | Requires 2+ logical processors |

Per the official runtime config docs, the **default is Workstation GC** at the raw runtime level. ASP.NET Core templates flip it to Server GC via `<ServerGarbageCollection>true</ServerGarbageCollection>` in the project file.

**Background GC** (concurrent) is a separate axis, **enabled by default**. It does most of the Gen 2 collection work in parallel with application threads, with only short stop-the-world pauses at the start and end. Gen 0 and Gen 1 collections are still stop-the-world (they're already short).

Deep dive: [Garbage Collector](../05-memory-and-performance/02-garbage-collector.md)

</details>

---

### 18. What is DATAS, and when did it become the default?

<details>
<summary>Reveal answer</summary>

DATAS (Dynamic Adaptation To Application Sizes) sizes the GC heap based on the application's actual workload rather than the machine's core count. Per the official runtime config docs:

- **.NET 8**: introduced as opt-in (`DOTNET_GCDynamicAdaptationMode=1`)
- **.NET 9**: enabled by default

The motivation is containerized workloads where memory has a real cost. Without DATAS, Server GC on a 48-core machine could grow a large heap even with light traffic. DATAS targets a **Throughput Cost Percentage** (default 2%) that combines GC pause time and allocation wait time, and adapts heap size accordingly.

Disable via `System.GC.DynamicAdaptationMode: 0` if you have a hot-from-first-request workload that can't tolerate the heap-growth ramp.

Deep dive: [Garbage Collector](../05-memory-and-performance/02-garbage-collector.md)

</details>

---

### 19. ValueTask vs Task — when does ValueTask actually pay off, and what restrictions does it carry?

<details>
<summary>Reveal answer</summary>

`ValueTask<T>` is a struct that can wrap either an already-available value (no heap allocation) or a `Task<T>` (when the slow path is unavoidable). It pays off in methods where the **synchronous-completion path is common** (cache hits, data already in buffer, etc.).

The official MS Learn docs are explicit about what you must not do:

- Awaiting the instance multiple times
- Calling `AsTask` multiple times
- Using `.Result` or `.GetAwaiter().GetResult()` when the operation hasn't yet completed, or using them multiple times
- Using more than one of these techniques to consume the instance

> "If you do any of the above, the results are undefined." — MS Learn

Subtle nuance: `ValueTask<T>` is a struct with **multiple fields**, while `Task<T>` is a reference type passed by **a single pointer**. Returning a `ValueTask` copies more bytes than returning a `Task`. So if the slow path dominates, `Task` is cheaper overall — `ValueTask` only wins when sync completion is common.

Microsoft's official recommendation: *"the default choice for any asynchronous method should be to return a Task or Task<TResult>. Only if performance analysis proves it worthwhile should a ValueTask<TResult> be used."*

Deep dive: [Async and Memory](../05-memory-and-performance/06-async-and-memory.md)

</details>

---

### 20. WeakReference vs ConditionalWeakTable — when do you reach for each?

<details>
<summary>Reveal answer</summary>

| | `WeakReference<T>` | `ConditionalWeakTable<TKey, TValue>` |
|---|---|---|
| What it does | Holds a reference to a single object without preventing GC | Associates extra data with an object you don't own; entry disappears when the key is collected |
| Type constraints | Any reference type | Both `TKey` and `TValue` must be reference types |
| Equality | N/A (single target) | **Reference identity only** — overrides of `Equals`/`GetHashCode` on the key are ignored |
| Typical use | Hand-rolled cache that mustn't pin memory; breaking observer cycles | Attached properties, framework instrumentation, anywhere you need to "tag" external objects |

The official `ConditionalWeakTable` docs are explicit: *"Two keys are equal if passing them to the `Object.ReferenceEquals` method returns `true`."* This is **not** what `Dictionary<K,V>` does — and is the key to its leak-free behavior.

For "real" caches with TTL and size limits, prefer `IMemoryCache` over either of these. Both `WeakReference` and `ConditionalWeakTable` are tools for specific problems, not a substitute for proper cache infrastructure.

Deep dive: [Memory Leak](../05-memory-and-performance/04-memory-leak.md)

</details>

---

### 21. Tiered compilation and Dynamic PGO — when did each become default, and what do they do?

<details>
<summary>Reveal answer</summary>

**Tiered compilation** (per MS Learn `compilation` runtime config docs):

- .NET Core 2.1 / 2.2: disabled by default
- .NET Core 3.0 and later: enabled by default

It compiles methods twice — Tier 0 (Quick JIT) for fast startup, Tier 1 (full optimization) once a method is observed to be hot. By default, Quick JIT does **not** apply to methods containing loops, since a long-running loop has no method-call boundary at which to swap in optimized code.

**Dynamic PGO** (per the official .NET 8 performance announcement):

- .NET 6: previewed, off by default
- .NET 7: improved, off by default
- .NET 8: *"now on by default"*

PGO instruments running code, observes which types and branches dominate, and feeds that info into Tier-1 recompilation. It enables guarded devirtualization of interface/virtual calls, smarter inlining decisions, block reordering for cache locality, and (in .NET 10) more aggressive escape analysis.

Practical takeaway: idiomatic patterns (`foreach` over `IEnumerable<T>`, lambdas, virtual calls) pay less abstraction tax in modern .NET than they did in .NET Framework. Re-benchmark before assuming an old "perf rule" still holds.

Deep dive: [JIT and Runtime](../05-memory-and-performance/07-jit-and-runtime.md)

</details>

---

### 22. You suspect a memory leak in production. What's the toolchain and workflow?

<details>
<summary>Reveal answer</summary>

The standard sequence:

1. **Live metrics first** — `dotnet-counters monitor -p <PID>` shows GC activity, allocation rate, working set, and Gen 2 collection frequency. Rising Gen 2 + growing working set strongly suggests a managed leak.

2. **Capture two snapshots** with a workload between them:
   ```bash
   dotnet-gcdump collect -p <PID>   # snapshot 1, after warm-up
   # run the suspected workload (e.g., 1000 requests)
   dotnet-gcdump collect -p <PID>   # snapshot 2
   ```

3. **Compare the snapshots** in JetBrains dotMemory (best UI for this) or PerfView. Types whose count grew unexpectedly are the candidates. For each candidate, look at the **retention path** — what chain of references is keeping it alive.

4. **The usual suspects, in order of frequency:**
   - Event handlers not unsubscribed (`+= handler` without matching `-=`)
   - Static collections that grow without bounds (caches without TTL/size limits)
   - Timers not disposed (capture `this` and freeze the whole graph)
   - `AsyncLocal` values captured by long-lived `ExecutionContext` (e.g., timer created while a heavy `AsyncLocal` was set)
   - Closures capturing more than they need (lambdas pulling in `this` implicitly)

5. **Validate locally** — write a focused BenchmarkDotNet test with `[MemoryDiagnoser]` confirming the fix removes the allocation; deploy and check `dotnet-counters` confirms the regression is gone in production.

For production processes you can't restart, `dotnet-gcdump` is significantly cheaper than `dotnet-dump` (managed heap only, smaller files, no unmanaged state). Use the full `dotnet-dump` only when you need access to native state too.

Deep dive: [Diagnostics](../05-memory-and-performance/08-diagnostics.md)

</details>

---

### 23. Why does System.IO.Pipelines exist, and what does `AdvanceTo(consumed, examined)` actually mean?

<details>
<summary>Reveal answer</summary>

Pipelines exists because the naïve `stream.Read(buffer)` pattern can't cleanly express "I got 1.5 messages, please remember the half-message and give me more bytes when you have them" — you end up reinventing buffering, framing, and backpressure badly. Kestrel uses Pipelines internally; for any custom binary protocol server (gRPC, RESP/Redis-compatible, MQTT, MessagePack), Pipelines is the modern answer.

`AdvanceTo(consumed, examined)` is the trick that makes it work:

- **`consumed`** — bytes up to this point are processed; never give them to me again
- **`examined`** — I looked at everything up to here but didn't have enough to make progress; if and only if more data arrives beyond `examined`, return from the next `ReadAsync`

If you collapse them (`AdvanceTo(consumed)` single-arg overload, or pass the same position twice), the pipeline assumes consumed = examined, which can deadlock if you didn't have enough data to commit a message. The two-position form is what enables zero-copy buffered parsing across arbitrary chunk boundaries.

The buffer returned by `ReadResult.Buffer` is a `ReadOnlySequence<byte>` — a logical view over multiple non-contiguous memory segments. Use `IsSingleSegment` for the fast path and `SequenceReader<byte>` for cross-segment parsing.

Deep dive: [System.IO.Pipelines](../05-memory-and-performance/09-pipelines.md)

</details>

---

### 24. Visibility vs ordering vs atomicity — what's the difference, and when do you need each?

<details>
<summary>Reveal answer</summary>

These are three distinct concurrency concerns:

| Concern | Definition | Example |
|---|---|---|
| **Atomicity** | A single operation can't be observed half-done | `int` reads/writes are atomic; `int++` is not (read-modify-write) |
| **Visibility** | A write on thread A becomes observable to thread B | A worker spinning on `_stopRequested` may never see `Stop()` without a barrier |
| **Ordering** | Operations on thread A appear to thread B in the order A wrote them | `_data = ...; _isReady = true;` — B can see `_isReady == true` while `_data` is still null |

Tools per concern:

- **Atomicity for read-modify-write**: `Interlocked.Increment`, `Interlocked.CompareExchange`. Use a `lock` for anything more complex than a single field.
- **Visibility**: `volatile` keyword (field-level) or `Volatile.Read`/`Volatile.Write` (call-site). A `lock` enter implicitly inserts an acquire barrier, exit a release barrier.
- **Ordering**: same tools as visibility — they implement **acquire-release** semantics. `Volatile.Read` is acquire (later ops can't move before it), `Volatile.Write` is release (earlier ops can't move past it).

The `lock` block is the simplest correct answer because it's a **full barrier** at both entry and exit and gives mutual exclusion as a bonus. Reach for `Interlocked` and `Volatile` only when profiling shows the lock is the bottleneck, or for stop-flag-style single-field signaling.

Deep dive: [Memory Model](../05-memory-and-performance/10-memory-model.md)

</details>

---

### 25. SearchValues, string.Create, u8 literals — when does each pay off?

<details>
<summary>Reveal answer</summary>

These are three modern .NET APIs that replace older patterns with allocation-free or allocation-reduced equivalents.

**`SearchValues<T>` (.NET 8)** — replaces `IndexOfAny(char[])` for repeated searches with a known set:

```csharp
private static readonly SearchValues<char> _vowels = SearchValues.Create("aeiouAEIOU");
public bool ContainsVowel(ReadOnlySpan<char> text) => text.IndexOfAny(_vowels) != -1;
```

The `SearchValues` instance pre-computes the optimal strategy (bitmap, range check, vectorized scan, etc.) once. Call sites use that pre-computed structure. Pays off with **3+ values, large inputs, and reuse** — not for one-off lookups.

**`string.Create<TState>`** (since .NET Core 2.1) — write directly into the string's underlying buffer when you know the final length:

```csharp
public string FormatId(long n) => string.Create(16, n, (Span<char> span, long v) =>
{
    "ID-".CopyTo(span);
    v.TryFormat(span[3..], out _, "D13");
});
```

Per the official docs, **the buffer is undefined initially** — you must fill every char or end up with random heap data. Use when the length is known and a `StringBuilder` would be wasted overhead.

**`u8` string literals (C# 11)** — produce a `ReadOnlySpan<byte>` containing the UTF-8 encoding, computed at compile time:

```csharp
private static ReadOnlySpan<byte> AuthBytes => "AUTH "u8;
reader.ValueTextEquals("name"u8);  // no UTF-16 → UTF-8 conversion at runtime
```

The bytes live in the assembly's `.data` section — no allocation at the call site. Replaces hand-rolled `byte[]` arrays and `Encoding.UTF8.GetBytes(...)` calls in hot HTTP/protocol parsing paths. Note that `static readonly` fields can't hold `ReadOnlySpan<byte>` (it's a `ref struct`); use a static expression-bodied property instead.

In C# 10+, `$"..."` interpolation also got dramatically faster via `DefaultInterpolatedStringHandler` — value types are no longer boxed into `object[]` for `string.Format`. You don't write the handler directly; the compiler emits it for you.

Deep dive: [Modern String APIs](../05-memory-and-performance/11-modern-string-apis.md)

</details>

---

[Back to index](README.md)
