# Dynamic Programming

## What it is

Dynamic Programming (DP) solves problems by breaking them into **overlapping subproblems** and storing the results to avoid redundant computation. It applies when a problem has:

1. **Optimal substructure** — the optimal solution contains optimal solutions to subproblems
2. **Overlapping subproblems** — the same subproblems are solved multiple times

> **Key insight:** If you find yourself solving the same subproblem repeatedly in a recursive solution, DP can optimize it from exponential to polynomial time.

## Two approaches

| Approach | Direction | How it works | Also called |
|----------|-----------|-------------|-------------|
| **Memoization** | Top-down | Recursion + cache results | Top-down DP |
| **Tabulation** | Bottom-up | Fill a table iteratively | Bottom-up DP |

## Fibonacci — the classic example

### Naive recursion — O(2^n)

```csharp
int Fib(int n)
{
    if (n <= 1) return n;
    return Fib(n - 1) + Fib(n - 2);
}

// Call tree for Fib(5):
//                Fib(5)
//              /        \
//          Fib(4)      Fib(3)
//         /    \       /    \
//      Fib(3) Fib(2) Fib(2) Fib(1)
//      /  \
//  Fib(2) Fib(1)
// ... Fib(2) is computed 3 times, Fib(3) is computed 2 times
```

### Memoization (top-down) — O(n)

```csharp
int Fib(int n, Dictionary<int, int>? memo = null)
{
    memo ??= new Dictionary<int, int>();

    if (n <= 1) return n;
    if (memo.ContainsKey(n)) return memo[n];

    memo[n] = Fib(n - 1, memo) + Fib(n - 2, memo);
    return memo[n];
}
```

- **Time:** O(n) — each subproblem computed once
- **Space:** O(n) — cache + call stack

### Tabulation (bottom-up) — O(n)

```csharp
int Fib(int n)
{
    if (n <= 1) return n;

    int[] dp = new int[n + 1];
    dp[0] = 0;
    dp[1] = 1;

    for (int i = 2; i <= n; i++)
        dp[i] = dp[i - 1] + dp[i - 2];

    return dp[n];
}
```

- **Time:** O(n)
- **Space:** O(n) — the table

### Space-optimized — O(1) space

```csharp
int Fib(int n)
{
    if (n <= 1) return n;

    int prev = 0, curr = 1;
    for (int i = 2; i <= n; i++)
    {
        int next = prev + curr;
        prev = curr;
        curr = next;
    }
    return curr;
}
```

> **Practical tip:** Always check if you can reduce space by keeping only the last few values instead of the entire table.

## 0/1 Knapsack problem

Given `n` items with weights and values, and a knapsack with capacity `W`, maximize the total value without exceeding the weight limit. Each item can be used **at most once**.

```csharp
int Knapsack(int[] weights, int[] values, int capacity)
{
    int n = weights.Length;
    int[,] dp = new int[n + 1, capacity + 1];

    for (int i = 1; i <= n; i++)
    {
        for (int w = 0; w <= capacity; w++)
        {
            // Don't take item i
            dp[i, w] = dp[i - 1, w];

            // Take item i (if it fits)
            if (weights[i - 1] <= w)
            {
                int valueWithItem = values[i - 1] + dp[i - 1, w - weights[i - 1]];
                dp[i, w] = Math.Max(dp[i, w], valueWithItem);
            }
        }
    }

    return dp[n, capacity];
}

// Usage
int[] weights = { 2, 3, 4, 5 };
int[] values  = { 3, 4, 5, 6 };
int capacity  = 8;
int maxValue  = Knapsack(weights, values, capacity); // 10 (items with weight 3+5)
```

- **Time:** O(n * W)
- **Space:** O(n * W) — can be optimized to O(W) using a 1D array

### Space-optimized knapsack

```csharp
int KnapsackOptimized(int[] weights, int[] values, int capacity)
{
    int n = weights.Length;
    int[] dp = new int[capacity + 1];

    for (int i = 0; i < n; i++)
    {
        // Traverse right to left to avoid using the same item twice
        for (int w = capacity; w >= weights[i]; w--)
        {
            dp[w] = Math.Max(dp[w], values[i] + dp[w - weights[i]]);
        }
    }

    return dp[capacity];
}
```

> **Interview tip:** The right-to-left traversal in the 1D version is crucial. Going left-to-right would allow using the same item multiple times (which is the "unbounded knapsack" variant).

## Longest Common Subsequence (LCS)

Find the length of the longest subsequence common to two strings.

```csharp
int LCS(string s1, string s2)
{
    int m = s1.Length, n = s2.Length;
    int[,] dp = new int[m + 1, n + 1];

    for (int i = 1; i <= m; i++)
    {
        for (int j = 1; j <= n; j++)
        {
            if (s1[i - 1] == s2[j - 1])
                dp[i, j] = dp[i - 1, j - 1] + 1;
            else
                dp[i, j] = Math.Max(dp[i - 1, j], dp[i, j - 1]);
        }
    }

    return dp[m, n];
}

// Example
int result = LCS("ABCBDAB", "BDCAB"); // 4 → "BCAB"
```

### Reconstruct the LCS

```csharp
string GetLCS(string s1, string s2)
{
    int m = s1.Length, n = s2.Length;
    int[,] dp = new int[m + 1, n + 1];

    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            if (s1[i - 1] == s2[j - 1])
                dp[i, j] = dp[i - 1, j - 1] + 1;
            else
                dp[i, j] = Math.Max(dp[i - 1, j], dp[i, j - 1]);

    // Backtrack to find the actual subsequence
    var sb = new StringBuilder();
    int x = m, y = n;
    while (x > 0 && y > 0)
    {
        if (s1[x - 1] == s2[y - 1])
        {
            sb.Insert(0, s1[x - 1]);
            x--; y--;
        }
        else if (dp[x - 1, y] > dp[x, y - 1])
            x--;
        else
            y--;
    }

    return sb.ToString();
}
```

- **Time:** O(m * n)
- **Space:** O(m * n)

## Coin Change

Given a set of coin denominations and a target amount, find the **minimum number of coins** needed.

```csharp
int CoinChange(int[] coins, int amount)
{
    int[] dp = new int[amount + 1];
    Array.Fill(dp, amount + 1); // fill with "impossible" value
    dp[0] = 0;

    for (int i = 1; i <= amount; i++)
    {
        foreach (int coin in coins)
        {
            if (coin <= i)
                dp[i] = Math.Min(dp[i], dp[i - coin] + 1);
        }
    }

    return dp[amount] > amount ? -1 : dp[amount];
}

// Example
int result = CoinChange(new[] { 1, 5, 10, 25 }, 36); // 3 → 25 + 10 + 1
```

- **Time:** O(amount * coins.Length)
- **Space:** O(amount)

### Count all ways to make change

```csharp
int CoinChangeWays(int[] coins, int amount)
{
    int[] dp = new int[amount + 1];
    dp[0] = 1; // one way to make 0: use no coins

    foreach (int coin in coins)
    {
        for (int i = coin; i <= amount; i++)
        {
            dp[i] += dp[i - coin];
        }
    }

    return dp[amount];
}

// CoinChangeWays(new[] { 1, 2, 5 }, 5) → 4 ways:
// [5], [2+2+1], [2+1+1+1], [1+1+1+1+1]
```

## How to identify DP problems

| Signal | Example |
|--------|---------|
| "Find the minimum/maximum..." | Minimum coins, maximum profit |
| "Count the number of ways..." | Coin change ways, stair climbing |
| "Is it possible to..." | Subset sum, can jump to end |
| "Longest/shortest..." | LCS, longest increasing subsequence |
| Choices at each step | Take or skip an item, go left or right |
| Overlapping recursive calls | Same arguments appear multiple times |

## DP problem-solving framework

1. **Define the state** — what does `dp[i]` (or `dp[i][j]`) represent?
2. **Find the recurrence** — how does `dp[i]` relate to smaller subproblems?
3. **Set base cases** — what are the trivial answers?
4. **Determine traversal order** — which subproblems must be solved first?
5. **Optimize space** — can you reduce from 2D to 1D?

> **Interview tip:** Start by writing the recursive solution, identify overlapping subproblems, then add memoization. Converting to tabulation is optional but shows deeper understanding.

## Memoization vs Tabulation — when to use each

| Aspect | Memoization (top-down) | Tabulation (bottom-up) |
|--------|----------------------|----------------------|
| Approach | Recursive + cache | Iterative + table |
| Computes | Only needed subproblems | All subproblems |
| Stack overflow risk | Yes (deep recursion) | No |
| Easier to write | Yes — just add cache to recursion | Requires thinking about order |
| Space optimization | Harder | Easier (rolling arrays) |
| Better when | Not all subproblems needed | All subproblems needed, want best performance |

---

[← Previous: Trees and Graphs](07-trees-and-graphs.md) | [Next: Greedy Algorithms →](09-greedy-algorithms.md) | [Back to index](README.md)
