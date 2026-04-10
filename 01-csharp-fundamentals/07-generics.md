# Generics

## What they are

Generics allow you to create classes, interfaces, and methods that work with **any type**, without losing type safety. They avoid code duplication and unnecessary boxing/unboxing.

```csharp
// Without generics: needs cast, no type safety
ArrayList list = new ArrayList();
list.Add(42);
int value = (int)list[0]; // manual cast, risk of InvalidCastException

// With generics: type-safe, no cast
List<int> list = new List<int>();
list.Add(42);
int value = list[0]; // no cast
```

## Generic classes and methods

```csharp
// Generic class
public class Repository<T> where T : class
{
    private readonly List<T> _items = new();

    public void Add(T item) => _items.Add(item);
    public T? GetByIndex(int index) => _items.ElementAtOrDefault(index);
    public IReadOnlyList<T> GetAll() => _items.AsReadOnly();
}

// Generic method
public T? Deserialize<T>(string json)
{
    return JsonSerializer.Deserialize<T>(json);
}

// Usage
var repo = new Repository<Product>();
repo.Add(new Product("Notebook"));
var product = repo.GetByIndex(0);
```

## Constraints (restrictions)

Constraints limit which types can be used as a generic argument:

```csharp
public class Service<T> where T : class, IEntity, new()
//                        ↑         ↑           ↑
//                   reference type  implements   has parameterless
//                                  IEntity       constructor
```

| Constraint | Meaning |
|-----------|---------|
| `where T : class` | Must be a reference type |
| `where T : struct` | Must be a value type |
| `where T : new()` | Must have a parameterless constructor |
| `where T : IEntity` | Must implement the interface |
| `where T : BaseClass` | Must inherit from the class |
| `where T : notnull` | Cannot be null |
| `where T : unmanaged` | Must be an unmanaged type (int, float, struct without refs) |

### Multiple constraints

```csharp
public class Repository<T> where T : class, IEntity, new()
{
    public T CreateNew()
    {
        var item = new T(); // possible because of new()
        item.Id = Guid.NewGuid(); // possible because of IEntity
        return item;
    }
}
```

## Covariance and Contravariance

### Covariance (`out`) -- "can return a more specific type"

```csharp
// IEnumerable<out T> is covariant
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings; // OK! string inherits from object

// Works because IEnumerable only RETURNS T, never receives
```

### Contravariance (`in`) -- "can accept a more generic type"

```csharp
// Action<in T> is contravariant
Action<object> printObject = obj => Console.WriteLine(obj);
Action<string> printString = printObject; // OK!

// Works because Action only RECEIVES T, never returns
```

### Practical rule

- `out T` = T only appears as a **return** (covariant)
- `in T` = T only appears as a **parameter** (contravariant)

```csharp
public interface IConverter<in TInput, out TOutput>
{
    TOutput Convert(TInput input);
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

[← Previous: Equals vs ==](06-equals-vs-operator.md) | [Next: Delegates and Events →](08-delegates-and-events.md) | [Back to index](README.md)
