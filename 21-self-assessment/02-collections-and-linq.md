# Collections and LINQ

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between IEnumerable, IQueryable, ICollection, and IList?

<details>
<summary>Reveal answer</summary>

| Interface | Key characteristic | Where it runs |
|-----------|--------------------|---------------|
| `IEnumerable<T>` | Forward-only, read-only iteration | In-memory (LINQ to Objects) |
| `IQueryable<T>` | Builds expression trees, translates to SQL | Database (LINQ to Entities) |
| `ICollection<T>` | Adds `Count`, `Add`, `Remove`, `Contains` | In-memory |
| `IList<T>` | Adds index-based access (`list[i]`) | In-memory |

**Rule of thumb**: accept the most general type you need. Return the most specific type the caller needs. Use `IQueryable` only when you want the query to be further composed before hitting the database.

Deep dive: [IEnumerable vs IQueryable](../02-collections-and-linq/01-ienumerable-vs-iqueryable.md)

</details>

---

### 2. What is deferred execution in LINQ? Why does it matter?

<details>
<summary>Reveal answer</summary>

Deferred execution means the query is **not executed when declared** — it runs only when you **enumerate** the result (`foreach`, `ToList()`, `Count()`, etc.).

```csharp
var query = orders.Where(o => o.Total > 100); // no execution yet
var list = query.ToList();                     // executes NOW
```

Why it matters:
- **Multiple enumeration**: iterating the same query twice executes it twice (two DB round-trips or two full scans).
- **Stale data**: the query sees the data at enumeration time, not declaration time.
- **Composability**: you can build queries incrementally before executing.

Use `.ToList()` to materialize when you need to iterate multiple times.

Deep dive: [IEnumerable vs IQueryable](../02-collections-and-linq/01-ienumerable-vs-iqueryable.md)

</details>

---

### 3. How does `yield return` work and when would you use it?

<details>
<summary>Reveal answer</summary>

`yield return` creates an **iterator** — the compiler generates a state machine that produces elements one at a time, lazily.

```csharp
IEnumerable<int> EvenNumbers(int max)
{
    for (int i = 0; i <= max; i++)
        if (i % 2 == 0)
            yield return i;
}

// Only generates values as you consume them
foreach (var n in EvenNumbers(1_000_000))
{
    if (n > 10) break; // stops early — no wasted computation
}
```

Use it when:
- The full collection is expensive to compute or load into memory
- You want **lazy evaluation** and early termination
- You are building custom LINQ-style operators

Deep dive: [Yield Return](../02-collections-and-linq/03-yield-return.md)

</details>

---

### 4. What are the most common LINQ performance pitfalls?

<details>
<summary>Reveal answer</summary>

1. **Multiple enumeration**: calling `.Where(...)` and then iterating twice hits the source twice. Fix: materialize with `.ToList()`.
2. **N+1 queries**: using `.Select(x => x.Orders.Count)` on an `IQueryable` without `.Include()` fires a query per row.
3. **Client-side evaluation**: EF silently pulling data to memory when it cannot translate a LINQ expression to SQL (EF Core 3+ throws instead of silently evaluating).
4. **Unnecessary allocations**: chaining many operators creates intermediate iterators. For hot paths, consider manual loops.
5. **Using `First()` instead of `FirstOrDefault()`**: throws an exception if empty — use `FirstOrDefault` and handle `null`.
6. **`.Count() > 0` instead of `.Any()`**: `Any()` short-circuits after the first match.

</details>

---

### 5. When would you use a Dictionary vs a List? What about lookup performance?

<details>
<summary>Reveal answer</summary>

| Operation | `List<T>` | `Dictionary<TKey, TValue>` |
|-----------|-----------|---------------------------|
| Lookup by key | O(n) — linear scan | O(1) — hash lookup |
| Lookup by index | O(1) | Not supported |
| Ordered | Yes (insertion order) | No order guarantee. In practice insertion order has been preserved since .NET Core 2.0 — but this is implementation detail, not contract. Use `OrderedDictionary<TKey, TValue>` (.NET 9+) or `SortedDictionary` if you need ordering. |
| Memory | Lower overhead | Higher (hash buckets) |

Use `Dictionary` when you need **frequent lookups by key**. Use `List` when you need **ordered access** or **sequential iteration** and lookups are rare.

**Common mistake**: using `list.FirstOrDefault(x => x.Id == id)` inside a loop — this is O(n*m). Convert to a dictionary first: `list.ToDictionary(x => x.Id)`.

</details>

---

### 6. What is a HashSet and when would you use it?

<details>
<summary>Reveal answer</summary>

`HashSet<T>` is an unordered collection of **unique elements** with O(1) `Contains`, `Add`, and `Remove`.

Use it when:
- You need to check membership frequently
- You need to eliminate duplicates
- You need set operations (union, intersection, difference)

```csharp
var allowed = new HashSet<string> { "admin", "editor" };
if (allowed.Contains(userRole)) { /* O(1) check */ }

var setA = new HashSet<int> { 1, 2, 3 };
var setB = new HashSet<int> { 2, 3, 4 };
setA.IntersectWith(setB); // setA = { 2, 3 }
```

**Tip**: `HashSet` requires proper `Equals` and `GetHashCode` implementations on custom types. For case-insensitive strings, pass `StringComparer.OrdinalIgnoreCase` to the constructor.

</details>

---

### 7. How does ConcurrentDictionary differ from a regular Dictionary with a lock?

<details>
<summary>Reveal answer</summary>

`ConcurrentDictionary<TKey, TValue>` uses **fine-grained locking** (striped locks) instead of a single global lock, allowing better throughput under contention.

| Aspect | `Dictionary` + `lock` | `ConcurrentDictionary` |
|--------|----------------------|----------------------|
| Granularity | Entire dictionary locked | Per-bucket locking |
| API | Manual lock around every access | Thread-safe methods built in |
| Atomic operations | You build them | `GetOrAdd`, `AddOrUpdate`, `TryRemove` |
| Performance under contention | Degrades — single bottleneck | Scales — parallel reads, localized writes |

**Gotcha**: the value factory in `GetOrAdd` and `AddOrUpdate` may be called **multiple times** if there is contention — do not put side effects in it. If you need guaranteed-once execution, use `Lazy<T>` as the value.

</details>

---

### 8. What is the difference between `Select` and `SelectMany` in LINQ?

<details>
<summary>Reveal answer</summary>

- **`Select`**: maps each element to **one** result (1-to-1).
- **`SelectMany`**: maps each element to **a collection** and flattens the results (1-to-many).

```csharp
var departments = new[]
{
    new { Name = "IT", Employees = new[] { "Alice", "Bob" } },
    new { Name = "HR", Employees = new[] { "Carol" } },
};

// Select — returns IEnumerable<string[]>
var nested = departments.Select(d => d.Employees);
// [ ["Alice","Bob"], ["Carol"] ]

// SelectMany — returns IEnumerable<string>
var flat = departments.SelectMany(d => d.Employees);
// [ "Alice", "Bob", "Carol" ]
```

Use `SelectMany` whenever you need to flatten nested collections — equivalent to a double `foreach`.

</details>

---

### 9. How would you avoid multiple enumeration of an IEnumerable?

<details>
<summary>Reveal answer</summary>

Multiple enumeration happens when you iterate the same `IEnumerable` more than once. If it is backed by a database query or a `yield` method, each enumeration re-executes the source.

**Solutions:**
1. **Materialize** the sequence: `var items = query.ToList();`
2. **Pass `IReadOnlyList<T>`** or `IReadOnlyCollection<T>` when the consumer needs count + iteration.
3. **ReSharper/Rider warning**: "Possible multiple enumeration of IEnumerable" — take it seriously.

```csharp
// Bad
IEnumerable<Order> orders = GetOrders();
var count = orders.Count();      // 1st enumeration
var first = orders.First();      // 2nd enumeration

// Good
var orders = GetOrders().ToList();
var count = orders.Count;        // no re-enumeration
var first = orders[0];
```

</details>

---

### 10. What is the difference between `ToDictionary`, `ToLookup`, and `GroupBy`?

<details>
<summary>Reveal answer</summary>

| Method | Returns | Key uniqueness | Execution |
|--------|---------|---------------|-----------|
| `ToDictionary` | `Dictionary<TKey, TValue>` | Keys must be unique (throws on duplicate) | Immediate |
| `ToLookup` | `ILookup<TKey, TValue>` | Keys can repeat (groups values) | Immediate |
| `GroupBy` | `IEnumerable<IGrouping<TKey, TValue>>` | Keys can repeat (groups values) | Deferred |

```csharp
// ToDictionary — one value per key
var byId = users.ToDictionary(u => u.Id);

// ToLookup — multiple values per key, materialized
var byRole = users.ToLookup(u => u.Role);
var admins = byRole["Admin"]; // returns empty if not found

// GroupBy — same as ToLookup but lazy
var grouped = users.GroupBy(u => u.Role);
```

Use `ToLookup` when you need repeated random access to groups. Use `GroupBy` when you iterate groups once.

Deep dive: [ICollection, IList](../02-collections-and-linq/02-icollection-ilist.md)

</details>

---

### 11. How does `Span<T>` relate to LINQ? Can you use LINQ with spans?

<details>
<summary>Reveal answer</summary>

You **cannot** use LINQ directly with `Span<T>` because `Span<T>` is a `ref struct` (stack-only) and LINQ extension methods require `IEnumerable<T>` which would box it.

For high-performance scenarios where you work with slices of arrays or memory:
- Use manual `for` loops instead of LINQ
- Use `Memory<T>` if you need to pass the data to async methods or store it on the heap
- In .NET 9+, some LINQ-like methods are available for spans via `MemoryExtensions`

```csharp
Span<int> numbers = stackalloc int[] { 1, 2, 3, 4, 5 };
// numbers.Where(...) — does NOT compile
// Manual loop instead
int sum = 0;
foreach (var n in numbers)
    if (n > 2) sum += n;
```

Deep dive: [Memory Optimization](../03-memory-and-performance/03-memory-optimization.md)

</details>

---

### 12. What is the difference between `List<T>` and `LinkedList<T>`? When would you actually pick the linked list?

<details>
<summary>Reveal answer</summary>

| Operation | `List<T>` | `LinkedList<T>` |
|-----------|-----------|-----------------|
| Memory layout | Contiguous array | Heap-allocated nodes with prev/next pointers |
| Access by index | O(1) | O(n) (no direct indexer) |
| Insert/remove at known node | O(n) — shifts elements | O(1) — pointer updates |
| Iteration speed | Very fast (cache-friendly) | Slow (pointer chasing, cache misses) |
| Memory overhead per item | ~`sizeof(T)` | `sizeof(T)` + 2 pointers + object header |

Even when Big-O suggests `LinkedList<T>`, `List<T>` usually wins in practice because modern CPUs love contiguous memory. Pick `LinkedList<T>` only when you already **hold a node reference** and need O(1) insert/remove — e.g., an LRU cache with a dictionary of nodes.

Deep dive: [LinkedList vs List](../02-collections-and-linq/05-linkedlist-vs-list.md)

</details>

---

### 13. How do you pick the right collection in .NET for a given job?

<details>
<summary>Reveal answer</summary>

Follow the intent, not the default:

| Need | Use |
|------|-----|
| Ordered, index-accessible, mutable | `List<T>` |
| Key → value lookup | `Dictionary<TKey, TValue>` |
| Unique values + fast membership | `HashSet<T>` |
| FIFO | `Queue<T>` · LIFO | `Stack<T>` |
| Ordered by key (sorted) | `SortedDictionary<TKey, TValue>` / `SortedSet<T>` |
| Thread-safe key/value | `ConcurrentDictionary<TKey, TValue>` |
| Thread-safe producer/consumer | `Channel<T>` (async) or `BlockingCollection<T>` |
| Immutable snapshot | `ImmutableList<T>`, `ImmutableDictionary<TKey, TValue>` |
| Read-only view | `IReadOnlyList<T>`, `IReadOnlyDictionary<TKey, TValue>` |
| Priority processing | `PriorityQueue<TElement, TPriority>` (.NET 6+) |
| Hot-path, stack-allocated | `Span<T>` / `stackalloc` |

Avoid the pre-generics `System.Collections` types (`ArrayList`, `Hashtable`) — they exist only for legacy compatibility.

Deep dive: [Collections Overview](../02-collections-and-linq/04-collections-overview.md)

</details>

---

[Back to index](README.md)
