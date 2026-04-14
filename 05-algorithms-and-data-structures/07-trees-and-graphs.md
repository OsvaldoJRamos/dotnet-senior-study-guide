# Trees and Graphs

## Trees

A tree is a **hierarchical** data structure with a root node and child nodes. Each node has exactly one parent (except the root).

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13
```

### Binary Tree node

```csharp
public class TreeNode<T>
{
    public T Value { get; set; }
    public TreeNode<T>? Left { get; set; }
    public TreeNode<T>? Right { get; set; }

    public TreeNode(T value) => Value = value;
}
```

## Binary Search Tree (BST)

A binary tree where for every node:
- **Left subtree** contains only values **less than** the node
- **Right subtree** contains only values **greater than** the node

| Operation | Average | Worst (unbalanced) |
|-----------|---------|-------------------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

> **Key insight:** The worst case happens when the tree degenerates into a linked list (e.g., inserting sorted data). Self-balancing trees (AVL, Red-Black) guarantee O(log n).

### BST implementation

```csharp
public class BinarySearchTree
{
    private TreeNode<int>? _root;

    public void Insert(int value)
    {
        _root = Insert(_root, value);
    }

    private TreeNode<int> Insert(TreeNode<int>? node, int value)
    {
        if (node is null)
            return new TreeNode<int>(value);

        if (value < node.Value)
            node.Left = Insert(node.Left, value);
        else if (value > node.Value)
            node.Right = Insert(node.Right, value);

        return node;
    }

    public bool Search(int value)
    {
        return Search(_root, value);
    }

    private bool Search(TreeNode<int>? node, int value)
    {
        if (node is null) return false;
        if (value == node.Value) return true;
        if (value < node.Value) return Search(node.Left, value);
        return Search(node.Right, value);
    }

    public int Height()
    {
        return Height(_root);
    }

    private int Height(TreeNode<int>? node)
    {
        if (node is null) return -1;
        return 1 + Math.Max(Height(node.Left), Height(node.Right));
    }
}
```

## Tree traversals

### Depth-First Traversals (recursive)

```csharp
//         8
//        / \
//       3   10

// In-order (Left → Root → Right): 3, 8, 10 — gives sorted order for BST
void InOrder(TreeNode<int>? node, List<int> result)
{
    if (node is null) return;
    InOrder(node.Left, result);
    result.Add(node.Value);
    InOrder(node.Right, result);
}

// Pre-order (Root → Left → Right): 8, 3, 10 — useful for copying a tree
void PreOrder(TreeNode<int>? node, List<int> result)
{
    if (node is null) return;
    result.Add(node.Value);
    PreOrder(node.Left, result);
    PreOrder(node.Right, result);
}

// Post-order (Left → Right → Root): 3, 10, 8 — useful for deletion
void PostOrder(TreeNode<int>? node, List<int> result)
{
    if (node is null) return;
    PostOrder(node.Left, result);
    PostOrder(node.Right, result);
    result.Add(node.Value);
}
```

### Breadth-First Traversal (BFS) — Level Order

Visits nodes level by level, left to right. Uses a **queue**.

```csharp
List<int> LevelOrder(TreeNode<int>? root)
{
    var result = new List<int>();
    if (root is null) return result;

    var queue = new Queue<TreeNode<int>>();
    queue.Enqueue(root);

    while (queue.Count > 0)
    {
        var node = queue.Dequeue();
        result.Add(node.Value);

        if (node.Left is not null) queue.Enqueue(node.Left);
        if (node.Right is not null) queue.Enqueue(node.Right);
    }

    return result;
}

// For the tree above: [8, 3, 10, 1, 6, 14, 4, 7, 13]
```

### Level-by-level grouping

```csharp
List<List<int>> LevelOrderGrouped(TreeNode<int>? root)
{
    var result = new List<List<int>>();
    if (root is null) return result;

    var queue = new Queue<TreeNode<int>>();
    queue.Enqueue(root);

    while (queue.Count > 0)
    {
        int levelSize = queue.Count;
        var level = new List<int>();

        for (int i = 0; i < levelSize; i++)
        {
            var node = queue.Dequeue();
            level.Add(node.Value);

            if (node.Left is not null) queue.Enqueue(node.Left);
            if (node.Right is not null) queue.Enqueue(node.Right);
        }

        result.Add(level);
    }

    return result;
    // [[8], [3, 10], [1, 6, 14], [4, 7, 13]]
}
```

## Graphs

A graph is a set of **vertices** (nodes) connected by **edges**. Unlike trees, graphs can have cycles, disconnected components, and multiple paths between nodes.

### Types

| Type | Description |
|------|-------------|
| Directed | Edges have direction (A → B) |
| Undirected | Edges go both ways (A — B) |
| Weighted | Edges have costs/distances |
| Unweighted | All edges have equal weight |
| Cyclic | Contains at least one cycle |
| Acyclic | No cycles (DAG = Directed Acyclic Graph) |

### Graph representations

#### Adjacency List (most common)

```csharp
// Using Dictionary — works for any node type
var graph = new Dictionary<string, List<string>>
{
    ["A"] = new() { "B", "C" },
    ["B"] = new() { "A", "D" },
    ["C"] = new() { "A", "D" },
    ["D"] = new() { "B", "C" }
};

// Space: O(V + E)
// Check if edge exists: O(degree of vertex)
// Iterate neighbors: O(degree of vertex)
```

#### Adjacency Matrix

```csharp
// For n vertices (0 to n-1)
int n = 4;
bool[,] matrix = new bool[n, n];
matrix[0, 1] = true; // edge from 0 to 1
matrix[1, 0] = true; // undirected: add both directions

// Space: O(V²)
// Check if edge exists: O(1)
// Iterate all neighbors: O(V)
```

| Representation | Space | Edge check | Iterate neighbors | Best for |
|---------------|-------|-----------|-------------------|----------|
| Adjacency List | O(V + E) | O(degree) | O(degree) | Sparse graphs |
| Adjacency Matrix | O(V²) | O(1) | O(V) | Dense graphs |

> **Practical tip:** Use adjacency list for most real-world graphs (social networks, web pages, road maps) since they are usually sparse.

## BFS — Breadth-First Search

Explores all neighbors at the current depth before moving deeper. Uses a **queue**.

```csharp
List<string> BFS(Dictionary<string, List<string>> graph, string start)
{
    var visited = new HashSet<string>();
    var queue = new Queue<string>();
    var result = new List<string>();

    visited.Add(start);
    queue.Enqueue(start);

    while (queue.Count > 0)
    {
        var node = queue.Dequeue();
        result.Add(node);

        foreach (var neighbor in graph[node])
        {
            if (!visited.Contains(neighbor))
            {
                visited.Add(neighbor);
                queue.Enqueue(neighbor);
            }
        }
    }

    return result;
}
```

- **Time:** O(V + E)
- **Space:** O(V)
- **Use case:** Shortest path in unweighted graphs, level-order traversal

### BFS shortest path (unweighted)

```csharp
int? ShortestPath(Dictionary<string, List<string>> graph, string start, string end)
{
    if (start == end) return 0;

    var visited = new HashSet<string> { start };
    var queue = new Queue<(string node, int distance)>();
    queue.Enqueue((start, 0));

    while (queue.Count > 0)
    {
        var (node, distance) = queue.Dequeue();

        foreach (var neighbor in graph[node])
        {
            if (neighbor == end) return distance + 1;

            if (!visited.Contains(neighbor))
            {
                visited.Add(neighbor);
                queue.Enqueue((neighbor, distance + 1));
            }
        }
    }

    return null; // not reachable
}
```

## DFS — Depth-First Search

Explores as deep as possible along each branch before backtracking. Uses a **stack** (or recursion).

```csharp
// Iterative DFS
List<string> DFS(Dictionary<string, List<string>> graph, string start)
{
    var visited = new HashSet<string>();
    var stack = new Stack<string>();
    var result = new List<string>();

    stack.Push(start);

    while (stack.Count > 0)
    {
        var node = stack.Pop();
        if (visited.Contains(node)) continue;

        visited.Add(node);
        result.Add(node);

        foreach (var neighbor in graph[node])
        {
            if (!visited.Contains(neighbor))
                stack.Push(neighbor);
        }
    }

    return result;
}

// Recursive DFS
void DfsRecursive(Dictionary<string, List<string>> graph, string node, HashSet<string> visited)
{
    visited.Add(node);
    Console.Write($"{node} ");

    foreach (var neighbor in graph[node])
    {
        if (!visited.Contains(neighbor))
            DfsRecursive(graph, neighbor, visited);
    }
}
```

- **Time:** O(V + E)
- **Space:** O(V)
- **Use case:** Cycle detection, topological sort, pathfinding, connected components

## BFS vs DFS

| Aspect | BFS | DFS |
|--------|-----|-----|
| Data structure | Queue | Stack / recursion |
| Explores | Level by level | Branch by branch |
| Shortest path (unweighted) | Yes | No |
| Memory | O(width of graph) | O(depth of graph) |
| Better for | Shortest path, nearest neighbor | Cycle detection, topological sort, exhaustive search |

## Dijkstra's Algorithm — Shortest path in weighted graphs

Finds the shortest path from a source to all other nodes. Uses a **priority queue** (min-heap).

```csharp
Dictionary<string, int> Dijkstra(
    Dictionary<string, List<(string neighbor, int weight)>> graph,
    string start)
{
    var distances = new Dictionary<string, int>();
    var pq = new PriorityQueue<string, int>();

    // Initialize all distances to infinity
    foreach (var node in graph.Keys)
        distances[node] = int.MaxValue;

    distances[start] = 0;
    pq.Enqueue(start, 0);

    while (pq.TryDequeue(out var current, out var dist))
    {
        // Staleness check: skip entries left over from before a shorter path was found
        if (dist > distances[current]) continue;

        // Sink nodes may not appear as keys in the adjacency list
        if (!graph.TryGetValue(current, out var neighbors) || neighbors is null)
            continue;

        foreach (var (neighbor, weight) in neighbors)
        {
            // Guard against overflow when distances[current] is still int.MaxValue
            if (distances[current] == int.MaxValue) continue;

            int newDist = distances[current] + weight;

            if (newDist < distances[neighbor])
            {
                distances[neighbor] = newDist;
                pq.Enqueue(neighbor, newDist);
            }
        }
    }

    return distances;
}

// Usage
var graph = new Dictionary<string, List<(string, int)>>
{
    ["A"] = new() { ("B", 4), ("C", 2) },
    ["B"] = new() { ("A", 4), ("D", 3), ("C", 1) },
    ["C"] = new() { ("A", 2), ("B", 1), ("D", 5) },
    ["D"] = new() { ("B", 3), ("C", 5) }
};

var distances = Dijkstra(graph, "A");
// A→A: 0, A→B: 3, A→C: 2, A→D: 6
```

- **Time:** O((V + E) log V) with a priority queue
- **Space:** O(V)
- **Limitation:** Does not work with negative edge weights (use Bellman-Ford instead)

> **Interview tip:** Dijkstra is greedy — it always processes the node with the smallest known distance next. This works because all edge weights are non-negative, so a shorter path can never be found later through a longer intermediate route.

> **A\*:** A\* is essentially Dijkstra plus a **heuristic estimate** of the remaining distance to the goal; it is optimal as long as the heuristic is **admissible** (never overestimates the true remaining cost).

## Topological Sort — Kahn's algorithm

For a **Directed Acyclic Graph (DAG)**, a topological sort produces a linear ordering of vertices such that for every edge `u → v`, `u` comes before `v`. Useful for build systems, task scheduling, and dependency resolution.

**Kahn's algorithm:**

1. Compute the **in-degree** (number of incoming edges) of every node
2. Enqueue all nodes with **in-degree 0** (no dependencies)
3. Dequeue a node, append it to the result, and **decrement** the in-degree of each neighbor; if a neighbor's in-degree drops to 0, enqueue it
4. Repeat until the queue is empty

```csharp
List<string> TopologicalSort(Dictionary<string, List<string>> graph)
{
    var inDegree = graph.Keys.ToDictionary(k => k, _ => 0);

    foreach (var neighbors in graph.Values)
        foreach (var n in neighbors)
            inDegree[n] = inDegree.GetValueOrDefault(n) + 1;

    var queue = new Queue<string>(inDegree.Where(kv => kv.Value == 0).Select(kv => kv.Key));
    var result = new List<string>();

    while (queue.Count > 0)
    {
        var node = queue.Dequeue();
        result.Add(node);

        foreach (var neighbor in graph.GetValueOrDefault(node, new()))
        {
            if (--inDegree[neighbor] == 0)
                queue.Enqueue(neighbor);
        }
    }

    // Cycle detection: if not every node was visited, a cycle exists
    if (result.Count != inDegree.Count)
        throw new InvalidOperationException("Graph has a cycle — no topological order exists");

    return result;
}
```

- **Time:** O(V + E)
- **Cycle detection:** if the final result count != node count, the graph has a cycle

## .NET collections for tree/graph problems

| Need | Use |
|------|-----|
| Sorted elements with O(log n) operations | `SortedSet<T>` (red-black tree) |
| Sorted key-value with O(log n) | `SortedDictionary<K,V>` (red-black tree) |
| Priority processing | `PriorityQueue<T, P>` (min-heap) |
| Graph adjacency list | `Dictionary<T, List<T>>` |
| Visited tracking | `HashSet<T>` |

---

[← Previous: Recursion](06-recursion.md) | [Next: Dynamic Programming →](08-dynamic-programming.md) | [Back to index](README.md)
