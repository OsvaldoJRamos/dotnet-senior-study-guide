# IEnumerable vs IQueryable

## Fundamental difference

**IEnumerable** works with all data in memory, while **IQueryable** creates a query that will be executed later. Generally, the query is executed when we use `.ToList()`.

- **IEnumerable** is more suitable for querying data that is already in memory (a list, for example).
- **IQueryable** is more suitable for use with Entity Framework for database queries, for example.

## IEnumerable vs List

Using **IEnumerable** is preferable because you are using an interface instead of a concrete class. Additionally, IEnumerable defers operations until the final moment (deferred execution).

### Example with IEnumerable (deferred execution):

```csharp
private void TesteIEnumerable()
{
    var nomes = new List<string> { "Luis", "João", "Ricardo", "Alexandre" };
    IEnumerable<string> nomesContenhamLetraO = nomes.Where(x => x.Contains("o"));
    nomes[0] = "Marcos";

    foreach (var nome in nomesContenhamLetraO)
    {
        Console.WriteLine(nome);
    }
}
```

**Output:**
```
Marcos
João
Ricardo
```

Note that even though we changed the name from "Luis" to "Marcos" **after** creating the IEnumerable object, the console displayed "Marcos". This is because the IEnumerable was only executed inside the foreach loop, and we changed the name before that.

### Example with List (immediate execution):

```csharp
private void TesteList()
{
    var nomes = new List<string> { "Luis", "João", "Ricardo", "Alexandre" };
    List<string> nomesContenhamLetraO = nomes.Where(x => x.Contains("o")).ToList();
    nomes[0] = "Marcos";

    foreach (var nome in nomesContenhamLetraO)
    {
        Console.WriteLine(nome);
    }
}
```

**Output:**
```
João
Ricardo
```

In this second example, the List object had already been created and stored in memory with the name "Luis". The change to "Marcos" did not affect the result because the query had already been materialized by `.ToList()`.

## IQueryable — when to use

```csharp
// IQueryable translates the LINQ expression to SQL in the database
IQueryable<Produto> query = context.Produtos
    .Where(p => p.Preco > 100)
    .OrderBy(p => p.Nome);

// The SQL query is only executed here:
var resultado = query.ToList();
```

The advantage is that the **filter is applied in the database**, not in the application. With IEnumerable, all records would be brought into memory and filtered there — much less efficient.

## Summary

| Feature | IEnumerable | IQueryable |
|---|---|---|
| Where the filter executes | In memory (C#) | On the server (SQL) |
| Best for | In-memory collections | Database queries |
| Execution | Deferred (lazy) | Deferred (lazy) |
| Provider | LINQ to Objects | LINQ to SQL/Entities |
| Namespace | `System.Collections.Generic` | `System.Linq` |

---

[Back to index](README.md) | [Next: ICollection and IList →](02-icollection-ilist.md)
