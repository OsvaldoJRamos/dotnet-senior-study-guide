# Algorithms and Data Structures

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is Big O notation and how do you analyze the time complexity of an algorithm?

<details>
<summary>Reveal answer</summary>

**Big O** describes the **upper bound** of how an algorithm's time (or space) grows relative to input size `n`. You analyze it by:

1. Counting the dominant operations (loops, recursive calls).
2. Dropping constants and lower-order terms.

Common complexities (fast to slow):

| Big O | Name | Example |
|-------|------|---------|
| O(1) | Constant | Hash table lookup |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Single loop through array |
| O(n log n) | Linearithmic | Merge sort, quicksort (avg) |
| O(n^2) | Quadratic | Nested loops, bubble sort |
| O(2^n) | Exponential | Brute-force subsets |

</details>

---

### 2. When would you use an Array (List) vs a LinkedList?

<details>
<summary>Reveal answer</summary>

| Operation | Array / List | LinkedList |
|-----------|-------------|------------|
| Access by index | **O(1)** | O(n) |
| Append to end | **O(1)** amortized | **O(1)** |
| Insert at beginning | O(n) (shift all elements) | **O(1)** |
| Memory | Contiguous, cache-friendly | Scattered, pointer overhead |

**Use arrays / `List<T>`** in almost all cases -- they have better cache locality and lower overhead. Use `LinkedList<T>` only when you frequently insert/remove at arbitrary positions **and** you already have a reference to the node.

In practice, `List<T>` wins for the vast majority of .NET workloads.

</details>

---

### 3. How do hash tables handle collisions? Name two common strategies.

<details>
<summary>Reveal answer</summary>

A **collision** occurs when two different keys hash to the same bucket.

1. **Chaining (separate chaining)** -- each bucket holds a linked list (or similar collection) of all entries that hash to that index. Lookup is O(1) average, O(n) worst case if all keys collide.

2. **Open addressing (probing)** -- if a bucket is occupied, probe the next slot (linear probing, quadratic probing, double hashing). All entries live in the array itself.

.NET's `Dictionary<TKey, TValue>` uses **chaining**. A good hash function and appropriate load factor keep collisions rare and lookups fast.

</details>

---

### 4. What are the requirements for binary search, and what is its time complexity?

<details>
<summary>Reveal answer</summary>

**Requirements**:
- The collection must be **sorted**.
- You need **random access** (index-based, like an array).

**How it works**: compare the target to the middle element. If smaller, search the left half; if larger, search the right half. Repeat.

**Time complexity**: **O(log n)** -- each step halves the search space. For 1 million elements, binary search needs at most ~20 comparisons vs. 1 million for linear search.

```csharp
int index = Array.BinarySearch(sortedArray, target);
```

</details>

---

### 5. When would you choose Merge Sort over Quick Sort?

<details>
<summary>Reveal answer</summary>

| Aspect | Merge Sort | Quick Sort |
|--------|-----------|------------|
| Worst case | **O(n log n)** always | O(n^2) (bad pivot) |
| Average case | O(n log n) | **O(n log n)** |
| Space | O(n) extra | O(log n) (in-place) |
| Stability | **Stable** | Not stable |
| Cache | Less cache-friendly | More cache-friendly |

Choose **Merge Sort** when you need a **stable sort** (preserves order of equal elements) or **guaranteed O(n log n)** (e.g., linked lists, external sorting). Choose **Quick Sort** for general-purpose in-memory sorting -- it's faster in practice due to cache locality. .NET's `Array.Sort` uses an **Introsort** (quicksort + heapsort fallback).

</details>

---

### 6. What is the difference between a stable and an unstable sorting algorithm?

<details>
<summary>Reveal answer</summary>

A **stable** sort preserves the relative order of elements that compare as equal. An **unstable** sort may rearrange them.

Example: sorting employees by department. If two employees are both in "Engineering", a stable sort keeps them in their original order (e.g., alphabetical by name if that was the prior sort).

| Stable | Unstable |
|--------|----------|
| Merge Sort | Quick Sort |
| Insertion Sort | Heap Sort |
| Bubble Sort | Selection Sort |

Stability matters when you sort by multiple criteria (sort by name, then by department -- the name order is preserved within each department).

</details>

---

### 7. When should you prefer recursion over iteration, and vice versa?

<details>
<summary>Reveal answer</summary>

**Prefer recursion** when:
- The problem is naturally recursive (tree traversals, divide-and-conquer, backtracking).
- Code clarity outweighs performance (recursive DFS is simpler to read).

**Prefer iteration** when:
- Deep recursion risks **stack overflow** (e.g., processing 100K+ elements).
- Performance matters -- iteration avoids function-call overhead.
- The problem maps naturally to a loop (summing an array, linear scan).

**Tip**: any recursive algorithm can be converted to iterative using an explicit stack. Many interview solutions start recursive and optimize to iterative if stack depth is a concern.

</details>

---

### 8. What is the difference between BFS and DFS? When would you use each?

<details>
<summary>Reveal answer</summary>

| Aspect | BFS (Breadth-First) | DFS (Depth-First) |
|--------|---------------------|-------------------|
| Data structure | Queue | Stack (or recursion) |
| Explores | Level by level | As deep as possible first |
| Shortest path? | **Yes** (unweighted graphs) | No |
| Memory | O(width) -- can be large | O(depth) -- usually smaller |

**Use BFS** when you need the **shortest path** in an unweighted graph (e.g., minimum hops, social network distance). **Use DFS** for exhaustive search, cycle detection, topological sort, or when memory is constrained and the graph is deep but narrow.

</details>

---

### 9. What is the difference between dynamic programming and a greedy algorithm?

<details>
<summary>Reveal answer</summary>

- **Dynamic Programming (DP)** -- breaks a problem into overlapping subproblems, solves each **once**, and stores results (memoization or tabulation). Guarantees an optimal solution when the problem has **optimal substructure** and **overlapping subproblems**. Example: Fibonacci, knapsack, longest common subsequence.

- **Greedy** -- makes the **locally optimal choice** at each step, hoping it leads to a globally optimal solution. Faster and simpler but only correct for problems with the **greedy choice property**. Example: coin change (with standard denominations), Huffman coding, Dijkstra's algorithm.

**Key difference**: DP explores all combinations (via caching); greedy commits to one choice per step.

</details>

---

### 10. When would you use a Stack vs a Queue?

<details>
<summary>Reveal answer</summary>

- **Stack (LIFO -- Last In, First Out)**: undo operations, expression parsing, DFS, call stack simulation, balanced parentheses checking.
- **Queue (FIFO -- First In, First Out)**: BFS, task scheduling, message processing, print queues, producer-consumer patterns.

```csharp
var stack = new Stack<int>();  // Push, Pop, Peek
var queue = new Queue<int>();  // Enqueue, Dequeue, Peek
```

**Rule of thumb**: if processing order matters and should be **fair** (first come first served), use a queue. If you need to **reverse** or **backtrack**, use a stack.

</details>

---

### 11. What are the three main tree traversal orders, and when is each useful?

<details>
<summary>Reveal answer</summary>

For a binary tree:

| Traversal | Order | Use case |
|-----------|-------|----------|
| **In-order** | Left -> Root -> Right | BST: produces sorted output |
| **Pre-order** | Root -> Left -> Right | Copying/serializing a tree |
| **Post-order** | Left -> Right -> Root | Deleting a tree, evaluating expressions |
| **Level-order** | BFS, level by level | Finding shortest depth, printing by level |

```
       1
      / \
     2   3

In-order:   2, 1, 3
Pre-order:  1, 2, 3
Post-order: 2, 3, 1
```

</details>

---

### 12. You have a sorted array of 1 billion integers. How many comparisons does binary search need in the worst case?

<details>
<summary>Reveal answer</summary>

Binary search needs **log2(n)** comparisons in the worst case.

log2(1,000,000,000) ≈ **30 comparisons**.

That's the power of logarithmic algorithms: even with a billion elements, you find the answer in about 30 steps. A linear search would need up to 1 billion comparisons.

This is why keeping data sorted (or using balanced BSTs / B-trees) enables extremely fast lookups.

</details>

---
