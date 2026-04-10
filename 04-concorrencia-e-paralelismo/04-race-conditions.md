# Race Conditions

## O que são

**Race conditions** ocorrem quando múltiplas threads acessam o mesmo recurso compartilhado, levando a comportamento inesperado devido à falta de sincronização entre as threads. Por exemplo, duas threads executando addClick e removeClick ao mesmo tempo. O total de cliques pode ser diferente dependendo de qual thread executa primeiro.

## Como evitar

### 1. Locks

Uma técnica de sincronização para restringir acesso a um recurso compartilhado. Um objeto que só pode ser mantido por uma thread por vez. A thread deve obter o lock antes de acessar o recurso e liberá-lo quando terminar.

```csharp
object lockObj = new object();
int sharedVar = 0;

// Thread 1
lock (lockObj)
{
    sharedVar++;
}

// Thread 2
lock (lockObj)
{
    sharedVar++;
}
```

Usando locks, apenas uma thread pode acessar `sharedVar` por vez, evitando race conditions.

### 2. Interlocked

Outra forma de evitar race conditions é usar a classe `Interlocked`, que fornece operações atômicas sobre variáveis compartilhadas. Operações atômicas são executadas em um único passo, sem interrupção de outras threads.

```csharp
int sharedVar = 0;

// Thread 1
Interlocked.Increment(ref sharedVar);

// Thread 2
Interlocked.Increment(ref sharedVar);
```

O `Interlocked.Increment` garante que a operação de incremento é executada atomicamente, evitando race conditions.

### 3. Thread Safety

Ao escrever código C#, é importante considerar thread safety no design de classes e métodos. Código thread-safe pode ser acessado por múltiplas threads simultaneamente sem causar race conditions. Você pode garantir thread safety usando técnicas de sincronização como locks ou operações Interlocked, ou usando objetos imutáveis.

```csharp
public class ThreadSafeCounter
{
    private int count;
    private readonly object lockObj = new object();

    public int Increment()
    {
        lock (lockObj)
        {
            return ++count;
        }
    }

    public int Decrement()
    {
        lock (lockObj)
        {
            return --count;
        }
    }
}
```

## Coleções thread-safe

O .NET fornece coleções thread-safe no namespace `System.Collections.Concurrent`:

```csharp
// Em vez de List<T> + lock:
var bag = new ConcurrentBag<string>();
var dict = new ConcurrentDictionary<string, int>();
var queue = new ConcurrentQueue<string>();
```

---

[← Anterior: Task, async/await](03-task-async-await.md) | [Voltar ao índice](README.md) | [Próximo: Deadlocks →](05-deadlocks.md)
