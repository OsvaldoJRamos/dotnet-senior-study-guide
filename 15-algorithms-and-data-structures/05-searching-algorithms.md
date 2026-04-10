# Searching Algorithms

## Overview

| Algorithm | Time Complexity | Requires Sorted | Space |
|-----------|----------------|-----------------|-------|
| Linear Search | O(n) | No | O(1) |
| Binary Search | O(log n) | Yes | O(1) |
| Interpolation Search | O(log log n) avg | Yes + uniform distribution | O(1) |
| Hash-based lookup | O(1) avg | No | O(n) |

## Linear Search — O(n)

Checks each element one by one until it finds the target or reaches the end.

```csharp
int LinearSearch(int[] arr, int target)
{
    for (int i = 0; i < arr.Length; i++)
    {
        if (arr[i] == target)
            return i;
    }
    return -1; // not found
}
```

- **Best case:** O(1) — target is the first element
- **Worst case:** O(n) — target is last or not present
- **No prerequisites** — works on unsorted data

> **When to use:** Small collections, unsorted data, or when the search is done only once (sorting + binary search isn't worth it for a single query).

### LINQ equivalents

```csharp
var items = new List<string> { "apple", "banana", "cherry" };

// All O(n) — linear scan
var found = items.FirstOrDefault(x => x == "banana");
bool exists = items.Contains("cherry");
bool any = items.Any(x => x.StartsWith("a"));
int index = items.IndexOf("banana");
```

## Binary Search — O(log n)

Repeatedly divides the search space in half. **Requires sorted data.**

```csharp
int BinarySearch(int[] sorted, int target)
{
    int left = 0;
    int right = sorted.Length - 1;

    while (left <= right)
    {
        int mid = left + (right - left) / 2; // avoids overflow

        if (sorted[mid] == target)
            return mid;
        else if (sorted[mid] < target)
            left = mid + 1;
        else
            right = mid - 1;
    }

    return -1; // not found
}
```

### How it works — step by step

```
Array: [2, 5, 8, 12, 16, 23, 38, 56, 72, 91]
Target: 23

Step 1: left=0, right=9, mid=4 → arr[4]=16 < 23 → left=5
Step 2: left=5, right=9, mid=7 → arr[7]=56 > 23 → right=6
Step 3: left=5, right=6, mid=5 → arr[5]=23 ✓ Found at index 5!
```

Only 3 comparisons for 10 elements. For 1,000,000 elements, binary search needs at most ~20 comparisons.

> **Key insight:** Each step eliminates half the data. That's why it's O(log n). Doubling the data size adds only one extra step.

### Recursive version

```csharp
int BinarySearchRecursive(int[] sorted, int target, int left, int right)
{
    if (left > right) return -1;

    int mid = left + (right - left) / 2;

    if (sorted[mid] == target) return mid;
    if (sorted[mid] < target)
        return BinarySearchRecursive(sorted, target, mid + 1, right);
    else
        return BinarySearchRecursive(sorted, target, left, mid - 1);
}
```

## .NET built-in: Array.BinarySearch()

```csharp
int[] sorted = { 1, 3, 5, 7, 9, 11, 13 };

int index = Array.BinarySearch(sorted, 7);    // returns 3
int notFound = Array.BinarySearch(sorted, 4); // returns negative value

// The negative value is the bitwise complement of the index
// where the element would be inserted to keep the array sorted
if (notFound < 0)
{
    int insertionPoint = ~notFound;
    Console.WriteLine($"Not found. Would insert at index {insertionPoint}");
}
```

```csharp
// List<T> also has BinarySearch
var list = new List<int> { 1, 3, 5, 7, 9 };
int idx = list.BinarySearch(5); // returns 2
```

> **Pitfall:** `Array.BinarySearch` and `List<T>.BinarySearch` assume the collection is **already sorted**. If it's not sorted, the result is undefined.

## Interpolation Search — O(log log n) average

Like binary search, but instead of always picking the middle, it **estimates** where the target might be based on its value. Works best on **uniformly distributed** data.

```csharp
int InterpolationSearch(int[] sorted, int target)
{
    int low = 0;
    int high = sorted.Length - 1;

    while (low <= high && target >= sorted[low] && target <= sorted[high])
    {
        if (low == high)
        {
            if (sorted[low] == target) return low;
            return -1;
        }

        // Estimate position based on value distribution
        int pos = low + ((target - sorted[low]) * (high - low))
                      / (sorted[high] - sorted[low]);

        if (sorted[pos] == target)
            return pos;
        else if (sorted[pos] < target)
            low = pos + 1;
        else
            high = pos - 1;
    }

    return -1;
}
```

| Case | Complexity |
|------|-----------|
| Best | O(1) |
| Average (uniform data) | O(log log n) |
| Worst (skewed data) | O(n) |

> **When to use:** Large, uniformly distributed, sorted datasets (like searching in a phone book). For most practical cases, binary search is sufficient.

## SortedSet\<T\> and SortedDictionary\<TKey, TValue\>

These collections maintain sorted order using a **red-black tree** (balanced BST), giving O(log n) for all operations.

```csharp
// SortedSet — unique sorted elements
var sortedSet = new SortedSet<int> { 5, 3, 8, 1, 9 };
// Iteration order: 1, 3, 5, 8, 9

bool has = sortedSet.Contains(3);        // O(log n)
sortedSet.Add(4);                         // O(log n)
sortedSet.Remove(8);                      // O(log n)

// Range operations
var range = sortedSet.GetViewBetween(3, 8); // subset [3, 4, 5]

int min = sortedSet.Min; // O(log n) — or O(1) in some implementations
int max = sortedSet.Max;
```

```csharp
// SortedDictionary — sorted key-value pairs
var sortedDict = new SortedDictionary<string, int>
{
    ["banana"] = 2,
    ["apple"] = 5,
    ["cherry"] = 3
};

// Iteration order: apple, banana, cherry (alphabetical)
foreach (var kvp in sortedDict)
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
```

### Comparison

| Collection | Structure | Lookup | Insert | Ordered |
|-----------|-----------|--------|--------|---------|
| `Dictionary<K,V>` | Hash table | O(1) avg | O(1) avg | No |
| `SortedDictionary<K,V>` | Red-black tree | O(log n) | O(log n) | Yes |
| `SortedList<K,V>` | Sorted array | O(log n) | O(n) | Yes |

> **Practical tip:** Use `SortedDictionary` when you need sorted order with frequent inserts/deletes. Use `SortedList` when memory is a concern and inserts are rare (it uses less memory but insertion is O(n) due to shifting).

## Choosing the right search

| Scenario | Approach |
|----------|---------|
| Unsorted, search once | Linear Search |
| Unsorted, search many times | Convert to `HashSet`/`Dictionary`, then O(1) |
| Sorted array, search many times | Binary Search |
| Need sorted order + fast search | `SortedSet<T>` / `SortedDictionary<K,V>` |
| Uniformly distributed sorted data | Interpolation Search |

---

[← Previous: Sorting Algorithms](04-sorting-algorithms.md) | [Next: Recursion →](06-recursion.md) | [Back to index](README.md)
