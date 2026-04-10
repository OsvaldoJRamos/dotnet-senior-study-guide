# Sorting Algorithms

## Overview

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |

> **Stable vs Unstable:** A stable sort preserves the relative order of equal elements. If you sort students by grade and two have the same grade, a stable sort keeps them in their original order.

## Bubble Sort — O(n²)

Repeatedly swaps adjacent elements if they are in the wrong order. Simple but inefficient.

```csharp
void BubbleSort(int[] arr)
{
    int n = arr.Length;
    for (int i = 0; i < n - 1; i++)
    {
        bool swapped = false;
        for (int j = 0; j < n - i - 1; j++)
        {
            if (arr[j] > arr[j + 1])
            {
                (arr[j], arr[j + 1]) = (arr[j + 1], arr[j]);
                swapped = true;
            }
        }
        if (!swapped) break; // already sorted — best case O(n)
    }
}
```

- **Best case:** O(n) — when already sorted (with early exit)
- **Use case:** Educational only. Never use in production.

## Selection Sort — O(n²)

Finds the minimum element and places it at the beginning. Repeats for each position.

```csharp
void SelectionSort(int[] arr)
{
    int n = arr.Length;
    for (int i = 0; i < n - 1; i++)
    {
        int minIdx = i;
        for (int j = i + 1; j < n; j++)
        {
            if (arr[j] < arr[minIdx])
                minIdx = j;
        }
        (arr[i], arr[minIdx]) = (arr[minIdx], arr[i]);
    }
}
```

- **Always O(n²)** — no best-case optimization
- **Advantage:** Minimal number of swaps (at most n-1)

## Insertion Sort — O(n²)

Builds the sorted array one element at a time by inserting each element into its correct position.

```csharp
void InsertionSort(int[] arr)
{
    for (int i = 1; i < arr.Length; i++)
    {
        int key = arr[i];
        int j = i - 1;

        while (j >= 0 && arr[j] > key)
        {
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = key;
    }
}
```

- **Best case:** O(n) — when nearly sorted
- **Use case:** Small arrays or nearly sorted data. .NET uses insertion sort for small partitions inside IntroSort.

## Merge Sort — O(n log n)

Divide and conquer: split the array in half, sort each half, then merge the sorted halves.

```csharp
void MergeSort(int[] arr, int left, int right)
{
    if (left >= right) return;

    int mid = left + (right - left) / 2;
    MergeSort(arr, left, mid);
    MergeSort(arr, mid + 1, right);
    Merge(arr, left, mid, right);
}

void Merge(int[] arr, int left, int mid, int right)
{
    int[] temp = new int[right - left + 1];
    int i = left, j = mid + 1, k = 0;

    while (i <= mid && j <= right)
    {
        if (arr[i] <= arr[j])
            temp[k++] = arr[i++];
        else
            temp[k++] = arr[j++];
    }

    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];

    Array.Copy(temp, 0, arr, left, temp.Length);
}

// Usage
int[] data = { 38, 27, 43, 3, 9, 82, 10 };
MergeSort(data, 0, data.Length - 1);
```

- **Always O(n log n)** — guaranteed performance
- **Space:** O(n) — needs auxiliary array
- **Stable:** Yes

> **Interview tip:** Merge sort is the go-to when you need guaranteed O(n log n) and stability. It's also the natural choice for sorting linked lists (no random access needed).

## Quick Sort — O(n log n) average

Pick a pivot, partition the array so everything left of the pivot is smaller and everything right is larger. Recurse on each side.

```csharp
void QuickSort(int[] arr, int low, int high)
{
    if (low >= high) return;

    int pivotIndex = Partition(arr, low, high);
    QuickSort(arr, low, pivotIndex - 1);
    QuickSort(arr, pivotIndex + 1, high);
}

int Partition(int[] arr, int low, int high)
{
    int pivot = arr[high]; // last element as pivot
    int i = low - 1;

    for (int j = low; j < high; j++)
    {
        if (arr[j] < pivot)
        {
            i++;
            (arr[i], arr[j]) = (arr[j], arr[i]);
        }
    }

    (arr[i + 1], arr[high]) = (arr[high], arr[i + 1]);
    return i + 1;
}

// Usage
int[] data = { 10, 7, 8, 9, 1, 5 };
QuickSort(data, 0, data.Length - 1);
```

- **Average:** O(n log n)
- **Worst:** O(n²) — when pivot is always the smallest/largest (already sorted array with last-element pivot)
- **Space:** O(log n) — recursive call stack
- **Not stable**

> **Mitigation:** Use median-of-three or random pivot selection to avoid worst case. .NET's IntroSort does this automatically.

## .NET's built-in sorting — IntroSort

`Array.Sort()` and `List<T>.Sort()` use **IntroSort**, a hybrid algorithm:

1. Starts with **Quick Sort**
2. If recursion depth exceeds `2 * log₂(n)`, switches to **Heap Sort** (guarantees O(n log n))
3. For small partitions (≤16 elements), uses **Insertion Sort** (better constant factors)

```csharp
int[] data = { 5, 3, 8, 1, 9, 2 };
Array.Sort(data);  // IntroSort — O(n log n) guaranteed

// Custom comparison
Array.Sort(data, (a, b) => b.CompareTo(a)); // descending

// LINQ — creates a new sorted collection
var sorted = data.OrderBy(x => x).ToList();         // stable sort
var desc = data.OrderByDescending(x => x).ToList();  // stable sort
```

| Method | Algorithm | Stable | In-place |
|--------|-----------|--------|----------|
| `Array.Sort()` | IntroSort | No | Yes |
| `List<T>.Sort()` | IntroSort | No | Yes |
| `LINQ OrderBy()` | Stable QuickSort variant | Yes | No (allocates) |

> **Practical tip:** If you need a stable sort, use LINQ `OrderBy`. If you need maximum performance and don't care about stability, use `Array.Sort()` or `List<T>.Sort()`.

## Choosing the right sort

| Scenario | Recommendation |
|----------|---------------|
| General purpose | `Array.Sort()` / `List<T>.Sort()` (IntroSort) |
| Need stability | LINQ `OrderBy()` |
| Nearly sorted data | Insertion Sort (or TimSort in other platforms) |
| Sorting linked lists | Merge Sort |
| External sorting (huge files) | Merge Sort (sequential access pattern) |
| Guaranteed O(n log n), no extra space | Heap Sort |

---

[← Previous: Hash Tables](03-hash-tables.md) | [Next: Searching Algorithms →](05-searching-algorithms.md) | [Back to index](README.md)
