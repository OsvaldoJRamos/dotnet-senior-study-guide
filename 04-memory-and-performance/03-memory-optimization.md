# Memory Optimization in .NET

## 1. Use StringBuilder for concatenation

Strings are immutable in C#, so each concatenation creates a new string object.

```csharp
// Inefficient: can cause multiple large allocations on the LOH
string result = "Hello" + largeString1 + largeString2;

// Efficient: uses StringBuilder to avoid large allocations
StringBuilder sb = new StringBuilder();
sb.Append("Hello");
sb.Append(largeString1);
sb.Append(largeString2);
string result = sb.ToString();
```

For hot paths, recycle the `StringBuilder` itself with `ObjectPool<StringBuilder>` (see section 4).

## 2. Choose appropriate data structures

Choosing an appropriate data structure is a key aspect of memory optimization. **Instead of using complex objects and collections** that consume more memory due to additional metadata, prefer simple data structures like **arrays**, **lists**, and **structs**.

### Arrays vs Lists

```csharp
// Uses more memory (capacity overhead, dynamic-resize bookkeeping)
List<string> names = new List<string>();
names.Add("John");
names.Add("Doe");

// Uses less memory
string[] names = new string[2];
names[0] = "John";
names[1] = "Doe";
```

The `string[]` array requires less memory than `List<string>` because it does not have the dynamic-resizing overhead. **However**, this does not mean you should always use arrays. If you frequently need to add new elements or perform operations only the list provides, `List<T>` is the right choice.

## 3. `Span<T>` and `Memory<T>` (high performance)

For high-performance scenarios, avoid allocating new arrays:

```csharp
// Allocates a new array
byte[] slice = array[10..20]; // creates a copy

// No allocation — a "view" over the original array
Span<byte> slice = array.AsSpan(10, 10);
```

### `Span<T>` is a `ref struct`

`Span<T>` is conceptually `(ref T pointer, int length)` — a reference plus a count. Because it can point at stack memory, the runtime forbids it from escaping the stack. Practical consequences:

- **Cannot be a field of a regular class** (only of another `ref struct`)
- **Cannot cross `await` or `yield`**
- **Cannot be captured in a lambda**
- **Cannot be boxed**
- **Cannot be an array element**

C# 13 relaxed some of these restrictions via the `where T : allows ref struct` anti-constraint and added support for `ref struct` types implementing interfaces (only callable through generic constraints, not casts, since boxing is still forbidden).

### `ReadOnlySpan<T>`

Use `ReadOnlySpan<T>` whenever you don't need to mutate. It's strictly more flexible:

```csharp
// Restrictive: only accepts char[] (or stackalloc / Span)
public int Parse(Span<char> input) { ... }

// Flexible: also accepts string (implicit conversion) and ReadOnlySpan sources
public int Parse(ReadOnlySpan<char> input) { ... }
```

`string` implicitly converts to `ReadOnlySpan<char>`, but **not** to `Span<char>` — strings are immutable, so a writable view would break the type system.

### `Memory<T>` for async / fields

`Memory<T>` is the regular-struct counterpart of `Span<T>`. It can be a field, parameter to async methods, captured in lambdas, etc. — but you can't read or write through `Memory<T>` directly. To access elements, convert to `Span<T>`:

```csharp
Memory<byte> memory = new byte[1024];
Span<byte> span = memory.Span; // small lookup; cache outside hot loops
span[0] = 42;
```

`Memory<T>` cannot wrap stack memory — only arrays, strings (for `ReadOnlyMemory<char>`), or custom `MemoryManager<T>` sources.

## 4. `stackalloc` for small temporary buffers

Allocate inside the current stack frame — practically free, scoped to the method:

```csharp
Span<byte> buffer = stackalloc byte[256];
```

Constraints:

- **Type must be unmanaged** (no references inside)
- **Keep it small** — a stack overflow is a process-killing exception, not a recoverable error
- **Contents are not zero-initialized** by default (call `.Clear()` if you need zeroes)

Common pattern when size is variable: stack for small, pool for large.

```csharp
const int StackThreshold = 256;
byte[]? rented = null;
Span<byte> buffer = size <= StackThreshold
    ? stackalloc byte[StackThreshold]
    : (rented = ArrayPool<byte>.Shared.Rent(size));

try
{
    Span<byte> data = buffer[..size];
    // use data
}
finally
{
    if (rented is not null) ArrayPool<byte>.Shared.Return(rented);
}
```

## 5. Object Pooling

Reuse expensive objects instead of repeatedly creating and destroying them:

```csharp
// Microsoft.Extensions.ObjectPool
var pool = new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());

var sb = pool.Get();
try
{
    sb.Append("Hello");
    return sb.ToString();
}
finally
{
    pool.Return(sb); // returns to the pool for reuse
}
```

The `StringBuilderPooledObjectPolicy` clears the builder on return and rejects builders whose `Capacity` exceeds a threshold, so the pool doesn't end up retaining oversized buffers.

## 6. ArrayPool for temporary arrays

```csharp
var buffer = ArrayPool<byte>.Shared.Rent(1024);
try
{
    // The returned array can be LARGER than what you asked for —
    // the pool buckets by power of two. Always work with a slice of the
    // exact size you need:
    Span<byte> actual = buffer.AsSpan(0, 1024);
    // use actual
}
finally
{
    // clearArray: true if the buffer held PII / secrets / object references.
    // For pooled arrays of reference types, NOT clearing keeps the references
    // alive in the pooled array, blocking GC of the targets.
    ArrayPool<byte>.Shared.Return(buffer, clearArray: false);
}
```

`ArrayPool<T>.Shared` is especially valuable for buffers that would otherwise land on the LOH (≥ 85,000 bytes) — every fresh allocation pressures Gen 2 with fragmentation. The pool removes that pressure entirely on the steady state.

### Pitfalls

- **Double return** — calling `Return` twice on the same buffer means another caller could rent it while you still hold a reference. Memory corruption / data races follow.
- **Use after return** — the next renter will clobber your data.
- **Buffer escape** — only hold the rented buffer inside the `try`/`finally` of the renting method.

## 7. Binary parsing with `BinaryPrimitives`

For zero-allocation binary parsing, prefer `System.Buffers.Binary.BinaryPrimitives` over `BitConverter` — it's portable across endianness and works on `Span<byte>`:

```csharp
ReadOnlySpan<byte> buf = stackalloc byte[] { 0x78, 0x56, 0x34, 0x12 };
int value = BinaryPrimitives.ReadInt32LittleEndian(buf);
// value == 0x12345678

// Big-endian (network byte order):
ushort port = BinaryPrimitives.ReadUInt16BigEndian(tcpHeader);

// Try variants for parsers that may fail
if (BinaryPrimitives.TryReadInt32LittleEndian(buf, out int parsed)) { ... }

// Write back
BinaryPrimitives.WriteInt32LittleEndian(dest, 12345);
```

The endianness is **explicit in the method name**. `BitConverter` uses the CPU's native endianness, which silently produces different results on big-endian hardware — `BinaryPrimitives` does not.

## Optimization techniques (checklist)

1. **Cache** — store results of expensive operations
2. **Async Jobs** — process heavy work in the background
3. **Monitoring** — use tools to measure real metrics in production
4. **CDN on the front-end** — serve static assets closer to the user
5. **Separate read and write databases** — CQRS to scale reads independently
6. **ElasticSearch** — for full-text search and relevance-based search

---

[← Previous: Garbage Collector](02-garbage-collector.md) | [Back to index](README.md) | [Next: Memory Leak →](04-memory-leak.md)
