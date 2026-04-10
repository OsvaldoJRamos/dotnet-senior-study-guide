# Recursion

## What it is

A function that **calls itself** to solve a problem by breaking it down into smaller subproblems. Every recursive function needs:

1. **Base case** — the condition that stops the recursion
2. **Recursive case** — the function calling itself with a smaller input

```csharp
void Countdown(int n)
{
    if (n <= 0)              // base case
    {
        Console.WriteLine("Done!");
        return;
    }
    Console.WriteLine(n);
    Countdown(n - 1);        // recursive case
}
```

> **Key rule:** Every recursive call must move toward the base case. Without it, you get a `StackOverflowException`.

## The call stack

Each recursive call creates a new **stack frame** with its own local variables. The frames pile up until the base case is reached, then unwind.

```
Factorial(4)
├── Factorial(3)
│   ├── Factorial(2)
│   │   ├── Factorial(1)
│   │   │   └── returns 1        ← base case
│   │   └── returns 2 * 1 = 2
│   └── returns 3 * 2 = 6
└── returns 4 * 6 = 24
```

## Classic examples

### Factorial

```csharp
int Factorial(int n)
{
    if (n <= 1) return 1;        // base case
    return n * Factorial(n - 1); // recursive case
}

// Factorial(5) = 5 * 4 * 3 * 2 * 1 = 120
```

- **Time:** O(n)
- **Space:** O(n) — n stack frames

### Fibonacci

```csharp
// Naive recursive — O(2^n) time, O(n) space
int Fibonacci(int n)
{
    if (n <= 1) return n;
    return Fibonacci(n - 1) + Fibonacci(n - 2);
}

// Fibonacci(6) = 8
// Call tree explodes: Fibonacci(5) + Fibonacci(4)
//                     each of those calls two more...
```

> **Pitfall:** Naive recursive Fibonacci is O(2^n) because it recalculates the same subproblems. See [Dynamic Programming](08-dynamic-programming.md) for the O(n) solution with memoization.

### Sum of an array

```csharp
int Sum(int[] arr, int index = 0)
{
    if (index >= arr.Length) return 0;          // base case
    return arr[index] + Sum(arr, index + 1);   // recursive case
}
```

### Power (exponentiation)

```csharp
// O(n) — linear recursion
double Power(double baseNum, int exponent)
{
    if (exponent == 0) return 1;
    return baseNum * Power(baseNum, exponent - 1);
}

// O(log n) — fast exponentiation
double FastPower(double baseNum, int exponent)
{
    if (exponent == 0) return 1;
    double half = FastPower(baseNum, exponent / 2);
    if (exponent % 2 == 0)
        return half * half;
    else
        return baseNum * half * half;
}
```

## Tree traversal — recursion's natural domain

Trees are inherently recursive structures. Each node is the root of a subtree.

```csharp
public class TreeNode
{
    public int Value { get; set; }
    public TreeNode? Left { get; set; }
    public TreeNode? Right { get; set; }
}

// In-order traversal: Left → Root → Right
void InOrder(TreeNode? node)
{
    if (node is null) return;     // base case
    InOrder(node.Left);
    Console.Write($"{node.Value} ");
    InOrder(node.Right);
}

// Pre-order: Root → Left → Right
void PreOrder(TreeNode? node)
{
    if (node is null) return;
    Console.Write($"{node.Value} ");
    PreOrder(node.Left);
    PreOrder(node.Right);
}

// Post-order: Left → Right → Root
void PostOrder(TreeNode? node)
{
    if (node is null) return;
    PostOrder(node.Left);
    PostOrder(node.Right);
    Console.Write($"{node.Value} ");
}
```

## Recursion vs iteration

Any recursive algorithm can be rewritten iteratively (and vice versa). The question is which is **clearer** and **more efficient**.

### Factorial — iterative version

```csharp
int FactorialIterative(int n)
{
    int result = 1;
    for (int i = 2; i <= n; i++)
        result *= i;
    return result;
}
// O(n) time, O(1) space — no stack frames
```

### Fibonacci — iterative version

```csharp
int FibonacciIterative(int n)
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
// O(n) time, O(1) space
```

### Comparison

| Aspect | Recursion | Iteration |
|--------|-----------|-----------|
| Readability | Often cleaner for trees, graphs, divide-and-conquer | Better for simple loops |
| Space | O(n) stack frames (risk of overflow) | O(1) typically |
| Performance | Function call overhead | Faster (no call overhead) |
| Stack overflow risk | Yes — default stack ~1MB in .NET | No |
| Natural fit | Trees, graphs, backtracking, divide-and-conquer | Linear traversal, accumulation |

> **Practical tip:** Use recursion when the problem is naturally recursive (trees, divide-and-conquer). Use iteration for simple linear problems. If recursion depth could be large (>1000), consider converting to iterative with an explicit stack.

## Converting recursion to iteration with a stack

```csharp
// Recursive DFS
void DfsRecursive(TreeNode? node)
{
    if (node is null) return;
    Console.Write($"{node.Value} ");
    DfsRecursive(node.Left);
    DfsRecursive(node.Right);
}

// Iterative DFS — using explicit stack
void DfsIterative(TreeNode? root)
{
    if (root is null) return;

    var stack = new Stack<TreeNode>();
    stack.Push(root);

    while (stack.Count > 0)
    {
        var node = stack.Pop();
        Console.Write($"{node.Value} ");

        // Push right first so left is processed first (LIFO)
        if (node.Right is not null) stack.Push(node.Right);
        if (node.Left is not null) stack.Push(node.Left);
    }
}
```

## Tail recursion

A recursive call is **tail recursive** if the recursive call is the **last operation** in the function — nothing happens after it returns.

```csharp
// NOT tail recursive — multiplication happens AFTER the recursive call
int Factorial(int n)
{
    if (n <= 1) return 1;
    return n * Factorial(n - 1); // must multiply after return
}

// Tail recursive — accumulator carries the result
int FactorialTail(int n, int accumulator = 1)
{
    if (n <= 1) return accumulator;
    return FactorialTail(n - 1, n * accumulator); // nothing after this call
}
```

> **Important for .NET:** The C# compiler and JIT do **not** reliably optimize tail recursion (unlike F#). You still risk `StackOverflowException` with deep tail recursion in C#. If depth is a concern, convert to iteration.

## Common recursion patterns

| Pattern | Example |
|---------|---------|
| **Linear recursion** | Factorial, linked list traversal |
| **Binary recursion** | Fibonacci (two calls per step) |
| **Divide and conquer** | Merge sort, quick sort, binary search |
| **Backtracking** | N-Queens, maze solving, Sudoku |
| **Tree recursion** | Tree traversal, tree height, subtree operations |

## Backtracking example — generate permutations

```csharp
List<List<int>> Permutations(int[] nums)
{
    var result = new List<List<int>>();
    var current = new List<int>();
    var used = new bool[nums.Length];
    Backtrack(nums, current, used, result);
    return result;
}

void Backtrack(int[] nums, List<int> current, bool[] used, List<List<int>> result)
{
    if (current.Count == nums.Length)
    {
        result.Add(new List<int>(current)); // base case — found a complete permutation
        return;
    }

    for (int i = 0; i < nums.Length; i++)
    {
        if (used[i]) continue;
        used[i] = true;
        current.Add(nums[i]);
        Backtrack(nums, current, used, result);
        current.RemoveAt(current.Count - 1); // backtrack
        used[i] = false;
    }
}
```

---

[← Previous: Searching Algorithms](05-searching-algorithms.md) | [Next: Trees and Graphs →](07-trees-and-graphs.md) | [Back to index](README.md)
