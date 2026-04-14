# Collections Overview

.NET ships with a large set of collections — picking the right one is one of the cheapest performance wins available. This file groups the main collections by purpose and shows when to use each.

## Quick decision table

| You need... | Use |
|---|---|
| Ordered list, index access | `List<T>` |
| Key → value lookup | `Dictionary<TKey, TValue>` |
| Unique values, fast membership check | `HashSet<T>` |
| FIFO processing | `Queue<T>` |
| LIFO processing | `Stack<T>` |
| Ordered by key (sorted) | `SortedDictionary<TKey, TValue>` / `SortedSet<T>` |
| Frequent insert/remove in the middle | `LinkedList<T>` |
| Thread-safe key/value | `ConcurrentDictionary<TKey, TValue>` |
| Thread-safe producer/consumer | `BlockingCollection<T>` / `Channel<T>` |
| Immutable snapshot | `ImmutableList<T>`, `ImmutableDictionary<TKey, TValue>` |
| Read-only view of an existing collection | `IReadOnlyList<T>`, `ReadOnlyCollection<T>` |
| Small, stack-allocated, hot path | `Span<T>`, `stackalloc` |

## 1. Generic collections (`System.Collections.Generic`)

The default choice for most code. Type-safe and allocation-friendly.

| Collection | Characteristics | Typical complexity |
|---|---|---|
| `List<T>` | Dynamic array, ordered, allows duplicates | Access O(1), Add amortized O(1), Insert/Remove O(n) |
| `Dictionary<TKey, TValue>` | Hash table, unordered | Lookup/Add/Remove O(1) average |
| `HashSet<T>` | Hash-based set of unique values | Contains/Add/Remove O(1) average |
| `Queue<T>` | FIFO | Enqueue/Dequeue O(1) |
| `Stack<T>` | LIFO | Push/Pop O(1) |
| `LinkedList<T>` | Doubly-linked list | Insert/Remove at known node O(1), Search O(n) |
| `SortedDictionary<TKey, TValue>` | Tree-based, sorted by key | O(log n) |
| `SortedList<TKey, TValue>` | Array-based, sorted; lower memory than SortedDictionary | O(log n) lookup, O(n) insert |
| `SortedSet<T>` | Tree-based unique sorted values | O(log n) |

```csharp
var orders = new List<Order>();                   // ordered, index access
var byId   = new Dictionary<int, Order>();        // fast lookup by id
var tags   = new HashSet<string>();               // unique values
var jobs   = new Queue<Job>();                    // FIFO pipeline
var undo   = new Stack<Command>();                // LIFO history
```

## 2. Concurrent collections (`System.Collections.Concurrent`)

Thread-safe by design. Use these instead of wrapping a non-concurrent collection with `lock`.

| Collection | Purpose |
|---|---|
| `ConcurrentDictionary<TKey, TValue>` | Thread-safe dictionary with atomic `GetOrAdd`, `AddOrUpdate`, `TryRemove` |
| `ConcurrentQueue<T>` | Thread-safe FIFO |
| `ConcurrentStack<T>` | Thread-safe LIFO |
| `ConcurrentBag<T>` | Unordered, optimized for "produce and consume by the same thread" |
| `BlockingCollection<T>` | Producer/consumer with bounded capacity and blocking `Take`/`Add` |

> For high-throughput async producer/consumer, prefer `System.Threading.Channels.Channel<T>` over `BlockingCollection<T>` — it is async-first and allocates less.

```csharp
var cache = new ConcurrentDictionary<string, User>();
var user = cache.GetOrAdd(id, id => LoadUser(id));
```

## 3. Immutable collections (`System.Collections.Immutable`)

Every mutation returns a **new** collection; the original is never changed. Great for shared state across threads and for avoiding defensive copies.

- `ImmutableList<T>`, `ImmutableArray<T>`
- `ImmutableDictionary<TKey, TValue>`, `ImmutableHashSet<T>`
- `ImmutableQueue<T>`, `ImmutableStack<T>`

```csharp
ImmutableList<string> tags = ImmutableList.Create("prod", "eu");
ImmutableList<string> more = tags.Add("beta"); // tags unchanged
```

> `ImmutableArray<T>` is a struct wrapping a `T[]` — very fast reads, cheap to pass around. Use it for read-heavy snapshots.

## 4. Read-only wrappers

Expose an existing collection without letting callers mutate it.

- `IReadOnlyCollection<T>` — count + iteration
- `IReadOnlyList<T>` — adds indexer
- `IReadOnlyDictionary<TKey, TValue>` — read-only dictionary
- `ReadOnlyCollection<T>` / `AsReadOnly()` — wrapper around a `List<T>`

These are **views**, not copies — if the underlying list changes, the view reflects it.

## 5. Specialized / niche

- `Memory<T>` / `ReadOnlyMemory<T>` — heap-friendly slice of contiguous memory, usable in async code
- `Span<T>` / `ReadOnlySpan<T>` — stack-only slice, hot-path allocation-free
- `ArraySegment<T>` — older API for slicing arrays
- `PriorityQueue<TElement, TPriority>` (.NET 6+) — min-heap priority queue
- `Collection<T>` / `KeyedCollection<TKey, TItem>` — extensible base classes used in frameworks

## Non-generic collections

The `System.Collections` namespace (`ArrayList`, `Hashtable`, `Queue`, `Stack`, etc.) is the **pre-generics** API from .NET 1.x. It stores `object`, which causes boxing for value types and removes type safety.

> Do not use non-generic collections in new code. They exist only for compatibility with legacy APIs.

## Picking the right collection — rule of thumb

1. **Iterating or returning from a method?** → `IEnumerable<T>` or `IReadOnlyList<T>`
2. **Reading by key?** → `Dictionary<TKey, TValue>`
3. **Checking membership / deduplicating?** → `HashSet<T>`
4. **Ordered by index, mutable?** → `List<T>`
5. **Multi-threaded writes?** → `ConcurrentDictionary<TKey, TValue>` or `Channel<T>`
6. **Need sorted iteration?** → `SortedDictionary<TKey, TValue>` or `SortedSet<T>`
7. **Need ultra-fast reads on a snapshot?** → `ImmutableArray<T>` or `FrozenDictionary<TKey, TValue>` (.NET 8+)

---

[← Previous: Yield Return](03-yield-return.md) | [Back to index](README.md) | [Next: LinkedList vs List →](05-linkedlist-vs-list.md)
