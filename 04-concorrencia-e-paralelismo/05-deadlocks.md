# Deadlocks

## O que é

**Deadlock** é uma situação em computação concorrente onde duas ou mais threads ou processos estão bloqueados, esperando que o outro libere recursos que cada um precisa para prosseguir, resultando em uma espera circular. Pode ser difícil de detectar e pode levar a degradação severa de performance ou até crashes do sistema.

## Como prevenir

### 1. Evite dependências circulares

Dependências circulares ocorrem quando duas ou mais threads estão esperando por recursos mantidos uma pela outra. Isso pode ser evitado projetando o sistema de forma que cada processo adquira recursos em uma **ordem específica** e os libere na **ordem inversa**.

### 2. Use Lock Hierarchy (hierarquia de locks)

Use uma hierarquia de locks para prevenir dependências circulares. Uma hierarquia de locks é um conjunto de locks organizados em uma ordem específica, e cada thread adquire locks na mesma ordem.

```csharp
private object lock1 = new object();
private object lock2 = new object();

public void Method1()
{
    lock (lock1)
    {
        lock (lock2)
        {
            // Access the resources
        }
    }
}

public void Method2()
{
    lock (lock1)       // mesma ordem que Method1!
    {
        lock (lock2)
        {
            // Access the resources
        }
    }
}
```

Os métodos `Method1` e `Method2` adquirem os locks `lock1` e `lock2` **na mesma ordem**, garantindo que não há espera circular.

### 3. Evite `lock` em recursos externos (ex: I/O, banco de dados)

Não use `lock` em métodos `async`, pois pode bloquear threads do pool:

```csharp
// NÃO faça isso
lock (locker)
{
    await database.SaveAsync(); // perigoso: pode travar
}
```

**Solução:** use mecanismos de sincronização assíncronos (`SemaphoreSlim`).

### 4. Use a Task Parallel Library (TPL)

A TPL é um framework de concorrência poderoso do .NET que pode ajudar a prevenir deadlocks. Ela gerencia concorrência automaticamente e garante que tasks não interfiram umas com as outras.

```csharp
Task.Run(() =>
{
    // Access the resources
});

Task.Run(() =>
{
    // Access the resources
});
```

### 5. Use timeouts

```csharp
bool lockTaken = false;
try
{
    Monitor.TryEnter(lockObj, TimeSpan.FromSeconds(5), ref lockTaken);
    if (lockTaken)
    {
        // recurso adquirido
    }
    else
    {
        // timeout - log e fallback
    }
}
finally
{
    if (lockTaken) Monitor.Exit(lockObj);
}
```

---

[← Anterior: Race Conditions](04-race-conditions.md) | [Voltar ao índice](README.md) | [Próximo: SemaphoreSlim →](06-semaphore-slim.md)
