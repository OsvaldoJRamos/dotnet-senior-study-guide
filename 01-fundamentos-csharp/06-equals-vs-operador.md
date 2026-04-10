# Equals() vs ==

## Diferenca fundamental

- `==` compara por **referencia** (para reference types) ou por **valor** (para value types)
- `Equals()` compara por **valor/conteudo** (pode ser sobrescrito)

## Com value types (int, struct, etc.)

Ambos comparam por **valor** - funcionam da mesma forma:

```csharp
int a = 5;
int b = 5;
Console.WriteLine(a == b);        // True
Console.WriteLine(a.Equals(b));   // True
```

## Com reference types (classes)

`==` compara se as **referencias apontam para o mesmo objeto na memoria**:

```csharp
var obj1 = new Pessoa("Joao");
var obj2 = new Pessoa("Joao");

Console.WriteLine(obj1 == obj2);      // False (referencias diferentes)
Console.WriteLine(obj1.Equals(obj2)); // False (Equals padrao tambem compara referencia)
```

Para comparar por conteudo, e preciso sobrescrever `Equals()`:

```csharp
public class Pessoa
{
    public string Nome { get; set; }

    public override bool Equals(object? obj)
    {
        if (obj is not Pessoa outra) return false;
        return Nome == outra.Nome;
    }

    public override int GetHashCode() => Nome.GetHashCode();
}

var obj1 = new Pessoa { Nome = "Joao" };
var obj2 = new Pessoa { Nome = "Joao" };
Console.WriteLine(obj1.Equals(obj2)); // True (agora compara por conteudo)
```

## Caso especial: string

`string` e reference type, mas o `==` foi **sobrescrito** para comparar por conteudo:

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(a == b);      // True (compara conteudo)
Console.WriteLine(a.Equals(b)); // True
```

## Records (C# 9+)

Records implementam automaticamente `Equals()` e `==` por valor:

```csharp
public record Pessoa(string Nome);

var p1 = new Pessoa("Joao");
var p2 = new Pessoa("Joao");
Console.WriteLine(p1 == p2);      // True
Console.WriteLine(p1.Equals(p2)); // True
```

## Cuidado com null

```csharp
Pessoa? p = null;
// p.Equals(outro) -> NullReferenceException!
// p == null        -> True (seguro)

// Forma segura:
Console.WriteLine(Equals(p, outro)); // metodo estatico, seguro com null
```

## Resumo

| Cenario | `==` | `Equals()` |
|---------|------|------------|
| Value types | Valor | Valor |
| Reference types (padrao) | Referencia | Referencia |
| string | Conteudo (sobrescrito) | Conteudo |
| Record | Conteudo (sobrescrito) | Conteudo |
| Com override de Equals | Referencia (a menos que sobrecarregue `==` tambem) | Conteudo customizado |

---

[← Anterior: Modificadores](05-modificadores.md) | [Próximo: Generics →](07-generics.md) | [Voltar ao índice](README.md)
