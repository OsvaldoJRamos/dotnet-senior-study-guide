# Arrays and Linked Lists

## Arrays

An array is a **contiguous block of memory** with a fixed size. Each element is accessed by its index in O(1) because the runtime calculates the exact memory address: `base + index * elementSize`.

```csharp
int[] numbers = new int[5];     // fixed size, default values (0)
int[] filled = { 1, 2, 3, 4 }; // initializer syntax

numbers[0] = 10;        // O(1) — direct access
int val = numbers[2];   // O(1) — direct access
```

> **Key point:** Arrays cannot grow. Once allocated, the size is fixed. If you need dynamic sizing, use `List<T>`.

## List\<T\> — dynamic array

`List<T>` wraps an internal array and **resizes automatically** (doubles capacity) when full.

```csharp
var list = new List<int>();   // default capacity: 0, grows to 4 on first add
list.Add(1);                  // O(1) amortized
list.Add(2);
list[0] = 10;                // O(1) — index access

list.Insert(0, 99);          // O(n) — shifts all elements right
list.RemoveAt(0);            // O(n) — shifts all elements left

// Pre-allocate if you know the size
var preAllocated = new List<int>(capacity: 10_000);
```

| Operation | Complexity |
|-----------|-----------|
| `Add` (end) | O(1) amortized |
| `Insert(index)` | O(n) |
| `RemoveAt(index)` | O(n) |
| `Remove(item)` | O(n) — linear search + shift |
| Index access `[i]` | O(1) |
| `Contains` | O(n) |
| `Count` | O(1) |

> **Practical tip:** If you're building a list inside a loop and know the expected size, always set the capacity. This avoids unnecessary array copies.

## LinkedList\<T\> — doubly linked list

Each node stores its value plus pointers to the **next** and **previous** nodes. No contiguous memory — each node is a separate heap allocation.

```csharp
var linked = new LinkedList<int>();
linked.AddLast(1);       // O(1)
linked.AddFirst(0);      // O(1)
linked.AddLast(2);       // O(1)

// Insert after a specific node — O(1) if you have the node reference
var node = linked.Find(1);              // O(n) — must traverse
linked.AddAfter(node!, 99);            // O(1) — just rewire pointers

linked.Remove(node!);                  // O(1) — with node reference
linked.Remove(99);                     // O(n) — must find first
```

| Operation | Complexity |
|-----------|-----------|
| `AddFirst` / `AddLast` | O(1) |
| `AddBefore` / `AddAfter` (with node) | O(1) |
| `Remove` (with node) | O(1) |
| `Remove` (by value) | O(n) |
| `Find` | O(n) |
| Index access | Not supported (must traverse) |

## Array vs List\<T\> vs LinkedList\<T\>

| Feature | `Array` | `List<T>` | `LinkedList<T>` |
|---------|---------|-----------|-----------------|
| Fixed size | Yes | No | No |
| Index access | O(1) | O(1) | O(n) |
| Insert at end | — | O(1)* | O(1) |
| Insert at start | O(n)† | O(n) | O(1) |
| Insert in middle | O(n)† | O(n) | O(1)‡ |
| Memory | Compact | Compact + buffer | Fragmented (per-node overhead) |
| Cache friendly | Yes | Yes | No |

*Amortized. †Requires manual shifting. ‡With node reference.

> **When to use what:** Use `List<T>` in 95% of cases. Use arrays when size is fixed and performance is critical. Use `LinkedList<T>` only when you need frequent insertions/removals at both ends or at known positions — but be aware of the cache-miss penalty.

## Stack\<T\> — LIFO (Last In, First Out)

Think of a stack of plates — you add and remove from the **top**.

```csharp
var stack = new Stack<int>();
stack.Push(1);    // [1]
stack.Push(2);    // [2, 1]
stack.Push(3);    // [3, 2, 1]

int top = stack.Peek();   // 3 — look without removing
int popped = stack.Pop(); // 3 — removes and returns top

Console.WriteLine(stack.Count); // 2
```

| Operation | Complexity |
|-----------|-----------|
| `Push` | O(1) amortized |
| `Pop` | O(1) |
| `Peek` | O(1) |
| `Contains` | O(n) |

**Use cases:** undo/redo, expression evaluation, DFS traversal, call stack simulation, bracket matching.

```csharp
// Classic example: validate balanced parentheses
bool IsBalanced(string expression)
{
    var stack = new Stack<char>();
    var pairs = new Dictionary<char, char>
    {
        { ')', '(' }, { ']', '[' }, { '}', '{' }
    };

    foreach (char c in expression)
    {
        if (c is '(' or '[' or '{')
            stack.Push(c);
        else if (pairs.ContainsKey(c))
        {
            if (stack.Count == 0 || stack.Pop() != pairs[c])
                return false;
        }
    }

    return stack.Count == 0;
}
```

## Queue\<T\> — FIFO (First In, First Out)

Like a real queue — first person in line gets served first.

```csharp
var queue = new Queue<int>();
queue.Enqueue(1);  // [1]
queue.Enqueue(2);  // [1, 2]
queue.Enqueue(3);  // [1, 2, 3]

int front = queue.Peek();      // 1
int dequeued = queue.Dequeue(); // 1 — removes first element
```

| Operation | Complexity |
|-----------|-----------|
| `Enqueue` | O(1) amortized |
| `Dequeue` | O(1) |
| `Peek` | O(1) |
| `Contains` | O(n) |

**Use cases:** BFS traversal, task scheduling, message processing, producer-consumer patterns.

## PriorityQueue\<TElement, TPriority\> (.NET 6+)

Elements are dequeued by **priority**, not insertion order. Lower priority value = higher priority (min-heap).

```csharp
var pq = new PriorityQueue<string, int>();
pq.Enqueue("Low priority task", 10);
pq.Enqueue("Critical task", 1);
pq.Enqueue("Medium task", 5);

while (pq.Count > 0)
{
    var task = pq.Dequeue();
    Console.WriteLine(task);
}
// Output:
// Critical task
// Medium task
// Low priority task
```

| Operation | Complexity |
|-----------|-----------|
| `Enqueue` | O(log n) |
| `Dequeue` | O(log n) |
| `Peek` | O(1) |

> **Practical tip:** `PriorityQueue` does not support updating priorities. If you need that, dequeue and re-enqueue, or use a custom heap implementation.

**Use cases:** Dijkstra's algorithm, task scheduling by priority, event-driven simulation, merge k sorted lists.

## When to use each

| Scenario | Best choice |
|----------|------------|
| Random access by index | `List<T>` or `Array` |
| Frequent add/remove at ends | `LinkedList<T>` |
| LIFO processing | `Stack<T>` |
| FIFO processing | `Queue<T>` |
| Priority-based processing | `PriorityQueue<T, P>` |
| Fixed-size, known at compile time | `Array` |
| Building a collection incrementally | `List<T>` |

---

[← Previous: Big O Notation](01-big-o-notation.md) | [Next: Hash Tables →](03-hash-tables.md) | [Back to index](README.md)
