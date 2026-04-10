# Stack e Heap

Em C# a memória é dividida em duas áreas principais: **Stack** e **Heap**.

## Stack

Usada para armazenar **value types** (ex: `int`, `float`, `struct`) e **referências** a objetos. É rápida e gerenciada automaticamente. Sempre que um método é chamado, um novo bloco de memória é alocado na stack, e quando o método termina, a memória é liberada.

```
int a = 10; // local variable

┌─────────┐
│ int a=10 │
└─────────┘
   Stack
```

## Heap

Usada para armazenar **reference types** (ex: objetos, arrays, strings). Quando objetos são criados usando `new`, eles são armazenados na heap. Essa memória permanece alocada até que o Garbage Collector (GC) decida que não está mais em uso.

```
Test a = new Test();

┌─────┐      ┌────────┐
│  a  │ ──── │ object │
└─────┘      └────────┘
 Stack          Heap
```

## Exemplo em código

```csharp
class Program
{
    static void Main(string[] args)
    {
        int number = 10;             // Stored in the stack
        Person person = new Person(); // Stored in the heap
        person.Name = "Alice";       // The reference to the object is on the stack, but the object is on the heap
    }
}

class Person
{
    public string Name;
}
```

Nesse exemplo, o inteiro `number` é armazenado na stack, mas o objeto `Person` e a string `"Alice"` são armazenados na heap.

## SOH e LOH

Objetos são alocados em duas áreas do heap:

1. **Small Object Heap (SOH):** Objetos menores que 85.000 bytes (cerca de 83 KB)
2. **Large Object Heap (LOH):** Objetos maiores que 85.000 bytes

O LOH é diferente do SOH porque objetos grandes são caros de alocar e desalocar. Alocações no LOH **não são compactadas** pelo GC, ou seja, pode ocorrer fragmentação de memória, levando a uso ineficiente de memória ao longo do tempo.

### Dicas para o LOH:
- Minimize o número de objetos grandes que você aloca
- Em vez de alocar arrays grandes, quebre-os em arrays menores que caibam no SOH
- Strings são imutáveis em C#, então concatenar strings grandes pode causar alocações no LOH. Use `StringBuilder` para reduzir o uso de memória

---

[Voltar ao índice](README.md) | [Próximo: Garbage Collector →](02-garbage-collector.md)
