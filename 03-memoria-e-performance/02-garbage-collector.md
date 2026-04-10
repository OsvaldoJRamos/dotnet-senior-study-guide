# Garbage Collector (GC)

## O que é

O **Garbage Collector (GC)** em .NET recupera automaticamente memória que não está mais em uso, liberando os desenvolvedores do gerenciamento manual de memória.

### O que o GC faz:
1. **Rastreia referências de objetos** — monitora todos os objetos na heap e rastreia quais ainda estão em uso
2. **Recupera memória não utilizada** — quando detecta que um objeto não é mais referenciado, marca como "lixo" e recupera sua memória
3. **Melhora a performance** — roda periodicamente para manter o uso de memória otimizado. Também reduz fragmentação compactando a heap

## Gerações (Generations)

O GC usa um conceito de **gerações** para tornar o processo mais eficiente.

### Generation 0 (Gen 0)
- É onde novos objetos são colocados inicialmente
- Objetos na Gen 0 são de vida curta (ex: variáveis temporárias)
- O GC verifica a Gen 0 frequentemente porque a maioria dos objetos tende a se tornar desnecessária rapidamente

### Generation 1 (Gen 1)
- Se um objeto sobrevive a uma coleta da Gen 0 (ou seja, ainda está em uso), ele é promovido para a Gen 1
- Objetos na Gen 1 são considerados de tempo de vida médio
- O GC verifica a Gen 1 com menos frequência que a Gen 0

### Generation 2 (Gen 2)
- Objetos que sobrevivem a uma coleta da Gen 1 vão para a Gen 2
- Esses objetos são de vida longa (ex: dados estáticos, objetos grandes usados durante toda a vida da aplicação)
- A Gen 2 é coletada com menos frequência porque assume-se que esses objetos ficarão por muito tempo

### Por que gerações?

A razão para dividir a memória em gerações é **otimizar o processo de coleta**. Objetos de vida curta (Gen 0) são coletados frequentemente, enquanto objetos de vida longa (Gen 2) são deixados em paz a menos que seja absolutamente necessário.

## Exemplo

```csharp
class Program
{
    static void Main()
    {
        // Generation 0 - Short-lived objects
        for (int i = 0; i < 1000; i++)
        {
            var tempObject = new MyObject(); // Allocated in Gen 0
        }

        // Force garbage collection and print the generation of an object
        GC.Collect(); // Forces the garbage collection to run
        MyObject longLivedObject = new MyObject(); // Created in Gen 0

        Console.WriteLine($"Generation of longLivedObject: {GC.GetGeneration(longLivedObject)}");

        // Promote longLivedObject to Gen 1 by forcing another collection
        GC.Collect();
        Console.WriteLine($"Generation of longLivedObject after GC: {GC.GetGeneration(longLivedObject)}");

        // Promote to Gen 2
        GC.Collect();
        Console.WriteLine($"Generation of longLivedObject after second GC: {GC.GetGeneration(longLivedObject)}");
    }
}

class MyObject
{
    public MyObject()
    {
        // Simulate some memory allocation
        byte[] buffer = new byte[1024];
    }
}
```

## Boas práticas

1. **Evite Memory Leaks** — sempre libere recursos não gerenciados (como conexões com banco de dados) usando o método `Dispose` ou blocos `using`
2. **Use IDisposable** — implemente `IDisposable` para qualquer classe que lide com recursos não gerenciados
3. **Cuidado com objetos grandes** — objetos grandes (arrays, buffers, etc.) podem acabar na Generation 2 e ficar lá por muito tempo. Use-os com cuidado

> A gestão de memória em aplicações C# geralmente é automatizada graças ao **Garbage Collector (GC)**, mas isso **não significa que o desenvolvedor está livre de preocupações**. Há práticas fundamentais para garantir eficiência e **evitar vazamento de memória (memory leaks)**.

---

[← Anterior: Stack e Heap](01-stack-e-heap.md) | [Voltar ao índice](README.md) | [Próximo: Otimização de Memória →](03-otimizacao-de-memoria.md)
