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

Deep dive: [Stack and Heap](../03-memory-and-performance/01-stack-and-heap.md)

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

Deep dive: [Garbage Collector](../03-memory-and-performance/02-garbage-collector.md)

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

Deep dive: [Memory Leaks](../03-memory-and-performance/04-memory-leak.md)

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

Deep dive: [Structs vs Classes](../03-memory-and-performance/05-structs-vs-classes.md)

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

Deep dive: [Memory Optimization](../03-memory-and-performance/03-memory-optimization.md)

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

Deep dive: [Memory Optimization](../03-memory-and-performance/03-memory-optimization.md)

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

Deep dive: [Memory Leaks](../03-memory-and-performance/04-memory-leak.md)

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

Deep dive: [Garbage Collector](../03-memory-and-performance/02-garbage-collector.md)

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

Deep dive: [Memory Optimization](../03-memory-and-performance/03-memory-optimization.md)

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

Deep dive: [Task, Async/Await](../04-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

[Back to index](README.md)
