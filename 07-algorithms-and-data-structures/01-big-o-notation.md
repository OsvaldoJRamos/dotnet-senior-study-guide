# Big O Notation

## What it is

Big O notation describes the **upper bound** of an algorithm's growth rate as the input size increases. It tells you how an algorithm **scales**, not how fast it runs in absolute terms.

> **Interview tip:** Big O measures the worst-case scenario. When someone asks "what's the complexity?", they usually mean the worst case unless stated otherwise.

## Common complexities

| Big O | Name | Example |
|-------|------|---------|
| O(1) | Constant | Dictionary lookup, array index access |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Loop through a list |
| O(n log n) | Linearithmic | Merge sort, .NET `Array.Sort()` |
| O(n²) | Quadratic | Nested loops, bubble sort |
| O(n³) | Cubic | Triple nested loops, naive matrix multiplication |
| O(2ⁿ) | Exponential | Recursive Fibonacci (naive), power set |
| O(n!) | Factorial | Generating all permutations |

## Growth comparison (n = 1,000)

| Big O | Operations |
|-------|-----------|
| O(1) | 1 |
| O(log n) | ~10 |
| O(n) | 1,000 |
| O(n log n) | ~10,000 |
| O(n²) | 1,000,000 |
| O(2ⁿ) | ∞ (not feasible) |

## How to analyze complexity

### Rule 1: Drop constants

```
O(2n) → O(n)
O(500) → O(1)
```

### Rule 2: Drop non-dominant terms

```
O(n² + n) → O(n²)
O(n + log n) → O(n)
```

### Rule 3: Different inputs = different variables

```csharp
// This is O(a * b), NOT O(n²)
void PrintPairs(int[] a, int[] b)
{
    foreach (var x in a)        // O(a)
        foreach (var y in b)    // O(b)
            Console.WriteLine($"{x},{y}");
}
```

### Rule 4: Sequential steps add, nested steps multiply

```csharp
// Sequential: O(n + m)
foreach (var item in listA) { /* ... */ }  // O(n)
foreach (var item in listB) { /* ... */ }  // O(m)

// Nested: O(n * m)
foreach (var a in listA)           // O(n)
    foreach (var b in listB)       // O(m)
        Console.WriteLine(a + b);
```

## Space complexity

Space complexity measures the **additional memory** an algorithm uses (excluding the input).

```csharp
// O(1) space — only a counter variable
int Sum(int[] arr)
{
    int total = 0;
    foreach (var n in arr) total += n;
    return total;
}

// O(n) space — creates a new array of size n
int[] Double(int[] arr)
{
    int[] result = new int[arr.Length];
    for (int i = 0; i < arr.Length; i++)
        result[i] = arr[i] * 2;
    return result;
}
```

## .NET pitfalls — hidden complexity

### LINQ `Contains()` on a List is O(n)

```csharp
var list = new List<int> { 1, 2, 3, 4, 5 };

// O(n) — scans the entire list
if (list.Contains(3)) { /* ... */ }

// O(1) — use HashSet instead
var set = new HashSet<int> { 1, 2, 3, 4, 5 };
if (set.Contains(3)) { /* ... */ }
```

> **Pitfall:** If you call `Contains()` inside a loop over another collection, you get O(n * m). Switching to a `HashSet` drops it to O(n).

### String concatenation in a loop is O(n²)

```csharp
// O(n²) — each concatenation creates a new string, copying all previous characters
string result = "";
for (int i = 0; i < 10_000; i++)
    result += i.ToString();  // copies the entire string each time

// O(n) — StringBuilder uses a resizable buffer
var sb = new StringBuilder();
for (int i = 0; i < 10_000; i++)
    sb.Append(i);
string result = sb.ToString();
```

### LINQ `OrderBy()` is O(n log n)

```csharp
// This is O(n log n) — not free just because it's one line
var sorted = items.OrderBy(x => x.Name).ToList();
```

### `List<T>.Insert(0, item)` is O(n)

```csharp
var list = new List<int>();
list.Insert(0, 42);  // O(n) — shifts all elements right

// If you need frequent front-insertion, use LinkedList<T> — O(1)
var linked = new LinkedList<int>();
linked.AddFirst(42);  // O(1)
```

## Amortized complexity

Some operations are **usually** O(1) but occasionally O(n). `List<T>.Add()` is an example:

- Most calls: O(1) — just appends to the internal array
- When the array is full: O(n) — allocates a new array (2x size) and copies everything

Over `n` insertions, the total cost is O(n), so the **amortized** cost per insertion is O(1).

> **Interview tip:** When asked about `List<T>.Add()`, say "O(1) amortized" — this shows you understand the resizing behavior.

## Quick reference — .NET collection operations

| Operation | `List<T>` | `LinkedList<T>` | `Dictionary<K,V>` | `HashSet<T>` | `SortedSet<T>` |
|-----------|-----------|------------------|--------------------|--------------|-----------------|
| Access by index | O(1) | O(n) | — | — | — |
| Search | O(n) | O(n) | O(1) avg | O(1) avg | O(log n) |
| Insert at end | O(1)* | O(1) | O(1) avg | O(1) avg | O(log n) |
| Insert at start | O(n) | O(1) | — | — | — |
| Remove | O(n) | O(1)† | O(1) avg | O(1) avg | O(log n) |

*Amortized. †If you have the node reference.

---

[Next: Arrays and Linked Lists →](02-arrays-and-linked-lists.md) | [Back to index](README.md)
