# SemaphoreSlim

## O que é um semáforo?

Um **semáforo** é um mecanismo de sincronização que **controla quantas threads podem acessar um recurso compartilhado ao mesmo tempo**.

> Imagine um estacionamento com 3 vagas: só 3 carros (threads) podem entrar ao mesmo tempo. Quem chegar depois, espera alguém sair.

## O que é SemaphoreSlim?

`SemaphoreSlim` é uma versão leve e moderna do semáforo em C#.
- Suporta operações assíncronas com `await`, sem bloquear a thread como o `lock` tradicional.

## Como funciona

### Criação:
```csharp
var semaforo = new SemaphoreSlim(1); // só 1 thread pode entrar por vez (como um lock)
```

### Uso básico:
```csharp
await semaforo.WaitAsync(); // aguarda permissão para entrar
try
{
    // Região crítica: apenas 1 thread por vez
}
finally
{
    semaforo.Release(); // libera permissão
}
```

### Exemplo com paralelismo assíncrono:

```csharp
private static readonly SemaphoreSlim _semaforo = new(1, 1);

public async Task ProcessarAsync()
{
    await _semaforo.WaitAsync();
    try
    {
        Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} entrou.");
        await Task.Delay(1000); // simula operação
    }
    finally
    {
        Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} saiu.");
        _semaforo.Release();
    }
}
```

Se você iniciar várias chamadas `ProcessarAsync()` ao mesmo tempo, apenas **uma** entrará na região crítica por vez.

## Quando usar SemaphoreSlim

| Situação | Usar SemaphoreSlim? |
|---|---|
| Várias tarefas assíncronas acessando recurso compartilhado (ex: arquivo, cache, lista) | Sim |
| Você precisa **limitar concorrência** (ex: máximo 3 chamadas simultâneas a uma API) | Sim |
| Você precisa de sincronização assíncrona (evitar `lock` com `await`) | Sim |
| Você está em código sincronizado (sem `async`) | Prefira `lock` |
| Quer evitar deadlocks e travamentos causados por `.Result` / `.Wait()` | Sim, pois SemaphoreSlim com `await` não bloqueia thread |

## O que evitar

```csharp
// ERRADO - lock com await causa deadlock
lock (locker)
{
    await MetodoAsync(); // NÃO faça isso
}
```

Isso causa **deadlock**, pois `lock` bloqueia a thread enquanto o `await` espera.

### Correto com SemaphoreSlim:

```csharp
await _semaforo.WaitAsync();
try
{
    await MetodoAsync();
}
finally
{
    _semaforo.Release();
}
```

## Conclusão

`SemaphoreSlim` é ideal para:
- **Evitar race conditions** em código assíncrono
- **Controlar concorrência** sem travar a aplicação
- **Evitar deadlocks** ao substituir `lock` em métodos `async`

---

[← Anterior: Deadlocks](05-deadlocks.md) | [Voltar ao índice](README.md)
