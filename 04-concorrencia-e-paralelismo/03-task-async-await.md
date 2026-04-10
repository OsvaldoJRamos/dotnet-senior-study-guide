# Task, async/await e PLINQ

## 3. Task com async/await

Ideal para **concorrência assíncrona**, mas também pode ser usado para paralelismo com `Task.WhenAll`:

```csharp
var tarefas = new[]
{
    Task.Run(() => Calcular(1)),
    Task.Run(() => Calcular(2)),
    Task.Run(() => Calcular(3))
};

await Task.WhenAll(tarefas);
```

### Exemplo prático com I/O:

```csharp
var cepTaskList = ceps.Select(cep => new ViaCepService().GetCepAsync(cep));
var cepList = await Task.WhenAll(cepTaskList);
```

## 4. PLINQ (Parallel LINQ)

Permite paralelizar queries LINQ:

```csharp
var resultado = lista
    .AsParallel()
    .Where(x => Processar(x))
    .ToList();
```

## Task.Run vs async/await — quando usar cada um

| Cenário | Usar |
|---|---|
| Trabalho CPU-bound pesado | `Task.Run` |
| Chamadas I/O (API, banco, arquivo) | `async/await` direto (sem `Task.Run`) |
| Múltiplas operações I/O simultâneas | `Task.WhenAll` |
| Query pesada em coleção grande | `PLINQ` |

## Erros comuns

### Não use .Result ou .Wait()

```csharp
// ERRADO - pode causar deadlock
var result = MinhaOperacaoAsync().Result;

// CORRETO - use await
var result = await MinhaOperacaoAsync();
```

### async void — evite

```csharp
// ERRADO - exceções não são capturáveis
async void ProcessarDados() { ... }

// CORRETO - use async Task
async Task ProcessarDados() { ... }
```

A única exceção para `async void` são event handlers de UI.

---

[← Anterior: Parallel.ForEach e Invoke](02-parallel-foreach-invoke.md) | [Voltar ao índice](README.md) | [Próximo: Race Conditions →](04-race-conditions.md)
