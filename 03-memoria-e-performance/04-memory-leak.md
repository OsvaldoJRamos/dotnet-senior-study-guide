# Memory Leak em C#

## O que é

Um **memory leak** acontece quando a aplicação aloca memória mas falha em liberá-la quando não é mais necessária. Com o tempo, essa memória residual entope o sistema, causando problemas de performance e, no pior caso, crashes da aplicação.

Um **memory profiler** pode ser usado para identificar memory leaks.

## Causas comuns

### 1. Conexões com o banco não fechadas

Abrir muitas conexões com o banco sem fechá-las (dentro de um loop por exemplo).

```csharp
// ERRADO - conexão nunca é fechada
for (int i = 0; i < 1000; i++)
{
    var conn = new SqlConnection(connectionString);
    conn.Open();
    // usa a conexão mas nunca fecha
}

// CORRETO - using garante que a conexão é fechada
for (int i = 0; i < 1000; i++)
{
    using var conn = new SqlConnection(connectionString);
    conn.Open();
    // conexão fechada automaticamente ao sair do escopo
}
```

### 2. Eventos não desregistrados

Uma das causas mais comuns de memory leak em C# é esquecer de **desregistrar event handlers**. Quando você se inscreve em um evento, o objeto que possui o event handler mantém uma referência ao subscriber, impedindo o garbage collection.

```csharp
public class Publisher
{
    public event EventHandler SomethingHappened;
}

public class Subscriber
{
    public void Subscribe(Publisher publisher)
    {
        publisher.SomethingHappened += HandleEvent;
    }

    public void HandleEvent(object sender, EventArgs e)
    {
        Console.WriteLine("Event handled.");
    }
}
```

**Solução:** sempre desregistre de eventos quando não forem mais necessários.

```csharp
public void Unsubscribe(Publisher publisher)
{
    publisher.SomethingHappened -= HandleEvent;
}
```

### 3. Referências estáticas

Objetos referenciados por variáveis estáticas podem persistir durante toda a vida da aplicação, e se não forem gerenciados cuidadosamente, podem levar a memory leaks.

```csharp
// PROBLEMA: lista estática que só cresce
public class MemoryLeak
{
    public static List<string> CachedData = new List<string>();
}
```

**Solução:** use weak references ou garanta que campos estáticos sejam limpos quando não forem mais necessários.

```csharp
// SOLUÇÃO: método para limpar o cache
public class MemoryLeak
{
    public static List<string> CachedData = new List<string>();

    public static void ClearCache()
    {
        CachedData.Clear();
    }
}
```

### 4. Objetos IDisposable não descartados

Objetos que implementam a interface `IDisposable` precisam de cleanup adequado. Se o método `Dispose` não é chamado explicitamente ou não usamos o bloco `using`, pode levar a resource leaks e memory leaks.

**Exemplo errado:**
```csharp
using System;
using System.IO;

class Program
{
    static void Main()
    {
        FileStream file = new FileStream("example.txt", FileMode.Create);
        // Some work with the file...
        // We forget to call Dispose on 'file.'
    }
}
```

**Forma correta com try/finally:**
```csharp
using System;

class Resource : IDisposable
{
    public void Dispose()
    {
        Console.WriteLine("Resource is disposed.");
    }
}

class ChildResource : Resource
{
    // This class inherits from Resource and should also be disposed.
}

class Program
{
    static void Main()
    {
        ChildResource childResource = new ChildResource();
        try
        {
            // Some work with the childResource...
        }
        finally
        {
            // Ensure Dispose is called to release resources.
            childResource?.Dispose();
        }
    }
}
```

**Forma correta com using (recomendada):**
```csharp
using System;
using System.IO;

class Program
{
    static void Main()
    {
        using (FileStream file = new FileStream("example.txt", FileMode.Create))
        {
            // Some work with the file...
        }
    }
}
```

### Use `IDisposable` corretamente

Recursos não gerenciados (como arquivos, conexões, streams, etc.) **não são liberados automaticamente pelo GC**.

**Prática recomendada:** implemente e use `IDisposable`:

```csharp
using (var stream = new FileStream("data.txt", FileMode.Open))
{
    // uso seguro
}
// Isso garante liberação de recursos com Dispose().
```

---

[← Anterior: Otimização de Memória](03-otimizacao-de-memoria.md) | [Voltar ao índice](README.md) | [Próximo: Structs vs Classes →](05-structs-vs-classes.md)
