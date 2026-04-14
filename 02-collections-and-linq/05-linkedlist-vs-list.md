# LinkedList vs List

`List<T>` and `LinkedList<T>` look similar — both are ordered, generic, mutable sequences — but they have very different memory layouts and performance profiles.

## TL;DR

- **`List<T>`** → dynamic array. Fast random access, cheap iteration, slow insert/remove in the middle.
- **`LinkedList<T>`** → doubly-linked list. Constant-time insert/remove at a known node, but slow lookups and poor cache locality.

**99% of the time, `List<T>` wins**, even for scenarios that textbooks suggest a linked list. Use `LinkedList<T>` only when you have a measured need for O(1) insertion/removal in the middle and you already hold a reference to the node.

## How they are stored

### `List<T>` — contiguous array

```
[ A ][ B ][ C ][ D ][ E ][   ][   ][   ]
 ^---------- Count ----^  ^-- Capacity --^
```

Items live next to each other in memory. Adding at the end is O(1) amortized; when capacity is reached, the backing array is copied into a larger one (usually 2x).

### `LinkedList<T>` — nodes + pointers

```
[prev|A|next] <-> [prev|B|next] <-> [prev|C|next]
```

Each element is a `LinkedListNode<T>` allocated on the heap, with pointers to the previous and next nodes.

## Complexity comparison

| Operation | `List<T>` | `LinkedList<T>` |
|---|---|---|
| Access by index | **O(1)** | O(n) (not supported directly) |
| Search by value | O(n) | O(n) |
| Add at end | O(1) amortized | O(1) |
| Add at beginning | O(n) | **O(1)** |
| Insert at middle (known position) | O(n) (shift items) | **O(1)** at a known node |
| Insert at middle (by value) | O(n) | O(n) (find) + O(1) (insert) |
| Remove at end | O(1) | O(1) |
| Remove at beginning | O(n) | **O(1)** |
| Remove at known node | O(n) | **O(1)** |
| Iteration | **Very fast** (cache-friendly) | Slower (pointer chasing) |
| Memory per element | ~`sizeof(T)` | `sizeof(T)` + 2 pointers + object header |

## Why `List<T>` usually wins even when Big-O suggests otherwise

Modern CPUs love contiguous memory. When you iterate a `List<T>`, the prefetcher loads chunks into L1/L2 cache and the loop flies. Iterating a `LinkedList<T>` **chases pointers across the heap**, missing cache lines on almost every step. Each element is also a separate heap allocation, meaning more work for the GC.

In practice:

- **Iterating a million ints** in a `List<int>` is dramatically faster than in a `LinkedList<int>`.
- **Inserting in the middle** of a `List<T>` with a few thousand items is often faster than walking a `LinkedList<T>` to find the insertion node.

## When `LinkedList<T>` is a good fit

- You **already have a `LinkedListNode<T>` reference** (e.g., an LRU cache where you track the node) and need O(1) insert/remove.
- You constantly add/remove at **both ends** of a long sequence (though `Queue<T>` or `Deque`-style structures usually beat it).
- You are implementing a data structure that explicitly relies on node pointers (e.g., a custom LRU, an intrusive list).

```csharp
// LRU cache sketch — LinkedList shines here because we hold node references
var list = new LinkedList<(string Key, string Value)>();
var index = new Dictionary<string, LinkedListNode<(string, string)>>();

void Access(string key, string value)
{
    if (index.TryGetValue(key, out var node))
    {
        list.Remove(node);          // O(1): we have the node
        list.AddFirst(node);        // O(1)
    }
    else
    {
        var node = list.AddFirst((key, value)); // O(1)
        index[key] = node;
    }
}
```

## When NOT to use `LinkedList<T>`

- Random access by index — use `List<T>` or an array.
- Searching by value — use `HashSet<T>` or `Dictionary<TKey, TValue>`.
- Hot iteration loops — `List<T>`/`T[]` is cache-friendly and dramatically faster.
- "I need a queue" — use `Queue<T>`. "I need a stack" — use `Stack<T>`.

## Practical rule

Default to `List<T>`. Reach for `LinkedList<T>` only when profiling shows that shifting elements in a list is a real bottleneck **and** you can operate on node references directly.

---

[← Previous: Collections Overview](04-collections-overview.md) | [Back to index](README.md)
