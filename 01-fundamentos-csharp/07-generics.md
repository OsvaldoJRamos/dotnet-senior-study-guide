# Generics

## What they are

Generics allow you to create classes, interfaces, and methods that work with **any type**, without losing type safety. They avoid code duplication and unnecessary boxing/unboxing.

```csharp
// Sem generics: precisa de cast, sem type safety
ArrayList lista = new ArrayList();
lista.Add(42);
int valor = (int)lista[0]; // cast manual, risco de InvalidCastException

// Com generics: type-safe, sem cast
List<int> lista = new List<int>();
lista.Add(42);
int valor = lista[0]; // sem cast
```

## Generic classes and methods

```csharp
// Classe generica
public class Repositorio<T> where T : class
{
    private readonly List<T> _items = new();

    public void Adicionar(T item) => _items.Add(item);
    public T? ObterPorIndice(int index) => _items.ElementAtOrDefault(index);
    public IReadOnlyList<T> ObterTodos() => _items.AsReadOnly();
}

// Metodo generico
public T? Deserializar<T>(string json)
{
    return JsonSerializer.Deserialize<T>(json);
}

// Uso
var repo = new Repositorio<Produto>();
repo.Adicionar(new Produto("Notebook"));
var produto = repo.ObterPorIndice(0);
```

## Constraints (restrictions)

Constraints limit which types can be used as a generic argument:

```csharp
public class Servico<T> where T : class, IEntidade, new()
//                        ↑         ↑           ↑
//                   reference type  implements   has parameterless
//                                  IEntidade     constructor
```

| Constraint | Meaning |
|-----------|---------|
| `where T : class` | Must be a reference type |
| `where T : struct` | Must be a value type |
| `where T : new()` | Must have a parameterless constructor |
| `where T : IEntidade` | Must implement the interface |
| `where T : BaseClass` | Must inherit from the class |
| `where T : notnull` | Cannot be null |
| `where T : unmanaged` | Must be an unmanaged type (int, float, struct without refs) |

### Multiple constraints

```csharp
public class Repositorio<T> where T : class, IEntidade, new()
{
    public T CriarNovo()
    {
        var item = new T(); // possivel por causa de new()
        item.Id = Guid.NewGuid(); // possivel por causa de IEntidade
        return item;
    }
}
```

## Covariance and Contravariance

### Covariance (`out`) -- "can return a more specific type"

```csharp
// IEnumerable<out T> e covariante
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings; // OK! string herda de object

// Funciona porque IEnumerable so RETORNA T, nunca recebe
```

### Contravariance (`in`) -- "can accept a more generic type"

```csharp
// Action<in T> e contravariante
Action<object> printObject = obj => Console.WriteLine(obj);
Action<string> printString = printObject; // OK!

// Funciona porque Action so RECEBE T, nunca retorna
```

### Practical rule

- `out T` = T only appears as a **return** (covariant)
- `in T` = T only appears as a **parameter** (contravariant)

```csharp
public interface IConvertedor<in TEntrada, out TSaida>
{
    TSaida Converter(TEntrada entrada);
}
```

## Generic vs object/dynamic

| Aspect | Generic (`T`) | `object` | `dynamic` |
|--------|---------------|----------|-----------|
| Type safety | Compile-time | None (needs cast) | Runtime |
| Performance | No boxing | Boxing for value types | DLR overhead |
| IntelliSense | Yes | No | No |
| When to use | Whenever possible | Interop, reflection | COM interop, ExpandoObject |

## Interview tips

1. Generics avoid **boxing/unboxing** with value types (performance)
2. Constraints guarantee **type safety at compile-time**
3. Covariance/contravariance only work with **interfaces and delegates**, not classes
4. `List<T>` is not covariant -- `IEnumerable<T>` is (because it's read-only)
5. Generics are resolved at **compile-time** (unlike Java which uses type erasure)

---

[← Previous: Equals vs ==](06-equals-vs-operador.md) | [Next: Delegates and Events →](08-delegates-e-eventos.md) | [Back to index](README.md)
