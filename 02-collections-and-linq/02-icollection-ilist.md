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
public IEnumerable<Produto> BuscarAtivos()
{
    return _context.Produtos.Where(p => p.Ativo);
}

// Use ICollection when you need to expose Add/Remove
public class Pedido
{
    public ICollection<ItemPedido> Itens { get; set; } = new List<ItemPedido>();
}

// Use IList when you need index-based access
public void ProcessarPorOrdem(IList<Tarefa> tarefas)
{
    for (int i = 0; i < tarefas.Count; i++)
    {
        tarefas[i].Ordem = i + 1;
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
public IReadOnlyCollection<Produto> Produtos => _produtos.AsReadOnly();
public IReadOnlyList<Produto> ProdutosOrdenados => _produtos.OrderBy(p => p.Nome).ToList();
```

---

[← Previous: IEnumerable vs IQueryable](01-ienumerable-vs-iqueryable.md) | [Back to index](README.md) | [Next: Yield Return →](03-yield-return.md)
