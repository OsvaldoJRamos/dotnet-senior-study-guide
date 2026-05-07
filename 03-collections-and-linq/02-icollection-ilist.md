# ICollection and IList

## Interface hierarchy

```
IEnumerable<T>          → Allows iteration (foreach)
    └── ICollection<T>  → Adds Count, Add, Remove, Contains
            └── IList<T> → Adds index-based access (this[int])
```

## Comparison

| Interface | What it offers | When to use |
|---|---|---|
| `IEnumerable<T>` | Iteration only (`foreach`) | When you only need to traverse the data |
| `ICollection<T>` | Iteration + `Count`, `Add`, `Remove`, `Contains` | When you need to know the size or modify the collection |
| `IList<T>` | Everything from ICollection + index-based access `[i]` | When you need to access elements by position |

## Usage examples

```csharp
// Use IEnumerable when the consumer will only iterate
public IEnumerable<Product> GetActive()
{
    return _context.Products.Where(p => p.Active);
}

// Use ICollection when you need to expose Add/Remove
public class Order
{
    public ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
}

// Use IList when you need index-based access
public void ProcessByOrder(IList<Task> tasks)
{
    for (int i = 0; i < tasks.Count; i++)
    {
        tasks[i].Order = i + 1;
    }
}
```

## Practical rule

**Use the most restrictive interface possible:**
- Method return type? → `IEnumerable<T>` in most cases
- Navigation property in EF? → `ICollection<T>`
- Need index access? → `IList<T>` or `IReadOnlyList<T>`

Avoid exposing `List<T>` directly — exposing the concrete implementation couples the code.

## IReadOnlyCollection and IReadOnlyList

For collections that should not be modified by the consumer:

```csharp
public IReadOnlyCollection<Product> Products => _products.AsReadOnly();
public IReadOnlyList<Product> SortedProducts => _products.OrderBy(p => p.Name).ToList();
```

---

[← Previous: IEnumerable vs IQueryable](01-ienumerable-vs-iqueryable.md) | [Back to index](README.md) | [Next: Yield Return →](03-yield-return.md)
