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
string result = sb.ToString(); // Final result without multiple large allocations
```

## 2. Choose appropriate data structures

Choosing an appropriate data structure is a key aspect of memory optimization. **Instead of using complex objects and collections**, which can consume more memory due to additional metadata, prefer simple data structures like **arrays**, **lists**, and **structs**.

### Arrays vs Lists

```csharp
// Uses more memory
List<string> names = new List<string>();
names.Add("John");
names.Add("Doe");

// Uses less memory
string[] names = new string[2];
names[0] = "John";
names[1] = "Doe";
```

The `string[]` array requires less memory compared to `List<string>` because it **does not have additional data structures to manage dynamic resizing**.

**However**, this does not mean you should always use arrays instead of lists. If you frequently need to add new elements and rebuild the array, or perform heavy lookups already available in the list, it is better to choose the list.

## 3. Use `Span<T>` and `Memory<T>` (high performance)

For high-performance scenarios, avoid allocating new arrays:

```csharp
// Allocates new array
byte[] slice = array[10..20]; // creates a copy

// No allocation — uses a "window" over the original array
Span<byte> slice = array.AsSpan(10, 10);
```

## 4. Object Pooling

Reuse expensive objects instead of repeatedly creating and destroying them:

```csharp
// Microsoft.Extensions.ObjectPool
var pool = new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());

var sb = pool.Get();
try
{
    sb.Append("Hello");
    // uses the StringBuilder
}
finally
{
    pool.Return(sb); // returns to the pool for reuse
}
```

## 5. ArrayPool for temporary arrays

```csharp
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(1024); // borrows
try
{
    // uses the buffer
}
finally
{
    pool.Return(buffer); // returns
}
```

## Optimization techniques (checklist)

1. **Cache** — store results of expensive operations
2. **Async Jobs** — process heavy work in the background
3. **Monitoring** — use tools to measure real metrics in production
4. **CDN on the front-end** — serve static assets closer to the user
5. **Separate read and write databases** — CQRS to scale reads independently
6. **ElasticSearch** — for full-text search and relevance-based search

---

[← Previous: Garbage Collector](02-garbage-collector.md) | [Back to index](README.md) | [Next: Memory Leak →](04-memory-leak.md)
