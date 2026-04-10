# Generics

## O que sao

Generics permitem criar classes, interfaces e metodos que trabalham com **qualquer tipo**, sem perder type safety. Evitam duplicacao de codigo e boxing/unboxing desnecessario.

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

## Classes e metodos genericos

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

## Constraints (restricoes)

Constraints limitam quais tipos podem ser usados como argumento generico:

```csharp
public class Servico<T> where T : class, IEntidade, new()
//                        ↑         ↑           ↑
//                   reference type  implementa   tem construtor
//                                  IEntidade     sem parametros
```

| Constraint | Significado |
|-----------|-------------|
| `where T : class` | Deve ser reference type |
| `where T : struct` | Deve ser value type |
| `where T : new()` | Deve ter construtor sem parametros |
| `where T : IEntidade` | Deve implementar a interface |
| `where T : BaseClass` | Deve herdar da classe |
| `where T : notnull` | Nao pode ser null |
| `where T : unmanaged` | Deve ser tipo unmanaged (int, float, struct sem refs) |

### Multiplas constraints

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

## Covariancia e Contravariancia

### Covariancia (`out`) — "pode retornar tipo mais especifico"

```csharp
// IEnumerable<out T> e covariante
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings; // OK! string herda de object

// Funciona porque IEnumerable so RETORNA T, nunca recebe
```

### Contravariancia (`in`) — "pode aceitar tipo mais generico"

```csharp
// Action<in T> e contravariante
Action<object> printObject = obj => Console.WriteLine(obj);
Action<string> printString = printObject; // OK!

// Funciona porque Action so RECEBE T, nunca retorna
```

### Regra pratica

- `out T` = T so aparece como **retorno** (covariante)
- `in T` = T so aparece como **parametro** (contravariante)

```csharp
public interface IConvertedor<in TEntrada, out TSaida>
{
    TSaida Converter(TEntrada entrada);
}
```

## Generic vs object/dynamic

| Aspecto | Generic (`T`) | `object` | `dynamic` |
|---------|---------------|----------|-----------|
| Type safety | Compile-time | Nenhum (precisa cast) | Runtime |
| Performance | Sem boxing | Boxing para value types | Overhead DLR |
| IntelliSense | Sim | Nao | Nao |
| Quando usar | Sempre que possivel | Interop, reflection | COM interop, ExpandoObject |

## Dicas para entrevista

1. Generics evitam **boxing/unboxing** com value types (performance)
2. Constraints garantem **type safety em compile-time**
3. Covariancia/contravariancia so funcionam com **interfaces e delegates**, nao classes
4. `List<T>` nao e covariante — `IEnumerable<T>` e (porque e read-only)
5. Generics sao resolvidos em **compile-time** (diferente de Java que usa type erasure)

---

[← Anterior: Equals vs ==](06-equals-vs-operador.md) | [Próximo: Delegates e Eventos →](08-delegates-e-eventos.md) | [Voltar ao índice](README.md)
