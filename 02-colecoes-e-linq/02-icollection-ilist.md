# ICollection e IList

## Hierarquia de interfaces

```
IEnumerable<T>          → Permite iterar (foreach)
    └── ICollection<T>  → Adiciona Count, Add, Remove, Contains
            └── IList<T> → Adiciona acesso por índice (this[int])
```

## Comparação

| Interface | O que oferece | Quando usar |
|---|---|---|
| `IEnumerable<T>` | Apenas iteração (`foreach`) | Quando só precisa percorrer os dados |
| `ICollection<T>` | Iteração + `Count`, `Add`, `Remove`, `Contains` | Quando precisa saber o tamanho ou modificar a coleção |
| `IList<T>` | Tudo de ICollection + acesso por índice `[i]` | Quando precisa acessar elementos por posição |

## Exemplos de uso

```csharp
// Use IEnumerable quando o consumidor só vai iterar
public IEnumerable<Produto> BuscarAtivos()
{
    return _context.Produtos.Where(p => p.Ativo);
}

// Use ICollection quando precisa expor Add/Remove
public class Pedido
{
    public ICollection<ItemPedido> Itens { get; set; } = new List<ItemPedido>();
}

// Use IList quando precisa de acesso por índice
public void ProcessarPorOrdem(IList<Tarefa> tarefas)
{
    for (int i = 0; i < tarefas.Count; i++)
    {
        tarefas[i].Ordem = i + 1;
    }
}
```

## Regra prática

**Use a interface mais restrita possível:**
- Retorno de métodos? → `IEnumerable<T>` na maioria dos casos
- Propriedade de navegação no EF? → `ICollection<T>`
- Precisa de índice? → `IList<T>` ou `IReadOnlyList<T>`

Evite expor `List<T>` diretamente — expor a implementação concreta acopla o código.

## IReadOnlyCollection e IReadOnlyList

Para coleções que não devem ser modificadas pelo consumidor:

```csharp
public IReadOnlyCollection<Produto> Produtos => _produtos.AsReadOnly();
public IReadOnlyList<Produto> ProdutosOrdenados => _produtos.OrderBy(p => p.Nome).ToList();
```

---

[← Anterior: IEnumerable vs IQueryable](01-ienumerable-vs-iqueryable.md) | [Voltar ao índice](README.md) | [Próximo: Yield Return →](03-yield-return.md)
