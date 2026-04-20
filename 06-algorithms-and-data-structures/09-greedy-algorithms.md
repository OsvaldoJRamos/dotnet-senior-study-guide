# Greedy Algorithms

## What they are

A greedy algorithm makes the **locally optimal choice** at each step, hoping to reach a globally optimal solution. It never reconsiders past choices.

> **Key insight:** Greedy algorithms are simple and fast, but they don't always produce the optimal solution. They work only when the problem has the **greedy choice property** — a locally optimal choice leads to a globally optimal solution.

## When greedy works

For a greedy approach to be correct, the problem must have:

1. **Greedy choice property** — a local optimum leads to a global optimum
2. **Optimal substructure** — the optimal solution contains optimal solutions to subproblems

## Activity Selection

Given a set of activities with start and end times, select the maximum number of non-overlapping activities.

**Greedy strategy:** Always pick the activity that **finishes earliest** (leaves the most room for remaining activities).

```csharp
List<(int start, int end)> ActivitySelection(List<(int start, int end)> activities)
{
    // Sort by end time
    var sorted = activities.OrderBy(a => a.end).ToList();

    var selected = new List<(int start, int end)>();
    selected.Add(sorted[0]);

    int lastEnd = sorted[0].end;

    for (int i = 1; i < sorted.Count; i++)
    {
        // Convention: half-open intervals [start, end) — an activity starting exactly
        // at lastEnd is considered non-overlapping (`>=`).
        if (sorted[i].start >= lastEnd)
        {
            selected.Add(sorted[i]);
            lastEnd = sorted[i].end;
        }
    }

    return selected;
}

// Example
var activities = new List<(int, int)>
{
    (1, 4), (3, 5), (0, 6), (5, 7), (3, 9), (5, 9),
    (6, 10), (8, 11), (8, 12), (2, 14), (12, 16)
};

var result = ActivitySelection(activities);
// Selected: (1,4), (5,7), (8,11), (12,16) → 4 activities
```

- **Time:** O(n log n) — dominated by sorting
- **Space:** O(n)

> **Why it works:** By always choosing the earliest-finishing activity, we maximize the remaining time for other activities. This can be formally proven by contradiction.

## Coin Change — Greedy (when it works)

For standard coin denominations (25, 10, 5, 1), the greedy approach works: always pick the largest coin that fits.

```csharp
List<int> GreedyCoinChange(int[] coins, int amount)
{
    // Sort coins descending
    Array.Sort(coins);
    Array.Reverse(coins);

    var result = new List<int>();

    foreach (int coin in coins)
    {
        while (amount >= coin)
        {
            result.Add(coin);
            amount -= coin;
        }
    }

    return result;
}

// Standard US coins: works perfectly
var coins = GreedyCoinChange(new[] { 1, 5, 10, 25 }, 36);
// [25, 10, 1] → 3 coins ✓
```

### When greedy fails for coin change

```csharp
// Coins: {1, 3, 4}, Amount: 6
// Greedy: 4 + 1 + 1 = 3 coins
// Optimal: 3 + 3 = 2 coins ← greedy is WRONG!
```

> **Pitfall:** Greedy coin change only works for "canonical" coin systems (like US coins). For arbitrary denominations, use [Dynamic Programming](08-dynamic-programming.md).

## Fractional Knapsack

Unlike the 0/1 knapsack (DP), here you can take **fractions** of items. Greedy works perfectly.

**Greedy strategy:** Sort by value-to-weight ratio (value/weight) descending. Take as much of the best ratio item as possible.

```csharp
double FractionalKnapsack(int[] weights, int[] values, int capacity)
{
    int n = weights.Length;

    // Create items with value/weight ratio
    var items = Enumerable.Range(0, n)
        .Select(i => new { Weight = weights[i], Value = values[i], Ratio = (double)values[i] / weights[i] })
        .OrderByDescending(x => x.Ratio)
        .ToList();

    double totalValue = 0;
    int remaining = capacity;

    foreach (var item in items)
    {
        if (remaining <= 0) break;

        if (item.Weight <= remaining)
        {
            // Take the whole item
            totalValue += item.Value;
            remaining -= item.Weight;
        }
        else
        {
            // Take a fraction
            totalValue += item.Ratio * remaining;
            remaining = 0;
        }
    }

    return totalValue;
}
```

- **Time:** O(n log n)
- **Space:** O(n)

## Huffman Coding

A greedy algorithm for optimal prefix-free encoding. Used in data compression (ZIP, JPEG).

**Greedy strategy:** Always merge the two nodes with the **lowest frequency**.

```csharp
public class HuffmanNode : IComparable<HuffmanNode>
{
    public char? Character { get; set; }
    public int Frequency { get; set; }
    public HuffmanNode? Left { get; set; }
    public HuffmanNode? Right { get; set; }

    public int CompareTo(HuffmanNode? other)
    {
        // IComparable.CompareTo contract: throw on null, do NOT substitute a default.
        // Returning a comparison against 0 here would break transitivity
        // (e.g. nodes with Frequency == 0 would compare equal to null).
        if (other is null) throw new ArgumentNullException(nameof(other));
        return Frequency.CompareTo(other.Frequency);
    }
}

HuffmanNode BuildHuffmanTree(Dictionary<char, int> frequencies)
{
    var pq = new PriorityQueue<HuffmanNode, int>();

    foreach (var kvp in frequencies)
        pq.Enqueue(new HuffmanNode { Character = kvp.Key, Frequency = kvp.Value }, kvp.Value);

    while (pq.Count > 1)
    {
        var left = pq.Dequeue();
        var right = pq.Dequeue();

        var parent = new HuffmanNode
        {
            Frequency = left.Frequency + right.Frequency,
            Left = left,
            Right = right
        };

        pq.Enqueue(parent, parent.Frequency);
    }

    return pq.Dequeue();
}

void GenerateCodes(HuffmanNode? node, string code, Dictionary<char, string> codes)
{
    if (node is null) return;

    if (node.Character.HasValue)
    {
        codes[node.Character.Value] = code;
        return;
    }

    GenerateCodes(node.Left, code + "0", codes);
    GenerateCodes(node.Right, code + "1", codes);
}
```

## Interval Scheduling / Merge Intervals

A common variation: merge overlapping intervals.

```csharp
List<(int start, int end)> MergeIntervals(List<(int start, int end)> intervals)
{
    if (intervals.Count <= 1) return intervals;

    var sorted = intervals.OrderBy(i => i.start).ToList();
    var merged = new List<(int start, int end)> { sorted[0] };

    for (int i = 1; i < sorted.Count; i++)
    {
        var last = merged[^1];

        // Convention: closed intervals [start, end] — intervals that touch at a point
        // (e.g. [1,3] and [3,5]) are considered overlapping and are merged (`<=`).
        if (sorted[i].start <= last.end)
        {
            // Overlapping — merge
            merged[^1] = (last.start, Math.Max(last.end, sorted[i].end));
        }
        else
        {
            merged.Add(sorted[i]);
        }
    }

    return merged;
}

// [(1,3), (2,6), (8,10), (15,18)] → [(1,6), (8,10), (15,18)]
```

## Greedy vs Dynamic Programming

| Aspect | Greedy | Dynamic Programming |
|--------|--------|-------------------|
| Approach | Local optimal choice | Explore all subproblems |
| Reconsiders choices | Never | Compares alternatives |
| Speed | Usually faster | Slower (but thorough) |
| Correctness | Only for specific problems | Always correct (if applicable) |
| Implementation | Simpler | More complex |
| Proof needed | Must prove greedy choice property | Must prove optimal substructure + overlapping |

### Same problem, different approaches

| Problem | Greedy works? | DP needed? |
|---------|--------------|-----------|
| Activity selection | Yes | Also works, but greedy is simpler |
| Fractional knapsack | Yes | Overkill |
| 0/1 knapsack | No | Yes |
| Coin change (standard coins) | Yes | Also works |
| Coin change (arbitrary coins) | No | Yes |
| Shortest path (non-negative weights) | Yes (Dijkstra, single-source) | Floyd-Warshall solves a different problem: **all-pairs shortest path**. Use it when you need every pairwise distance, not as a drop-in replacement for Dijkstra. |
| Shortest path (negative weights) | No | Yes (Bellman-Ford) |
| Huffman coding | Yes | Overkill |

> **Interview tip:** When faced with an optimization problem, first check if greedy works (it's simpler and faster). If you can't prove the greedy choice property, fall back to DP.

## How to approach greedy problems

1. **Sort** — most greedy algorithms start by sorting the input
2. **Define the greedy choice** — what's the local optimum at each step?
3. **Prove correctness** — can you argue (even informally) that local optimum leads to global optimum?
4. **If unsure** — try a counterexample. If greedy fails, switch to DP

---

[← Previous: Dynamic Programming](08-dynamic-programming.md) | [Back to index](README.md)
