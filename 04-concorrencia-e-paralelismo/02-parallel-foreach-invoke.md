# Parallel.ForEach e Parallel.Invoke

## 1. Parallel.ForEach

Executa iterações de um loop em paralelo.

```csharp
var parallelOptions = new ParallelOptions();
parallelOptions.MaxDegreeOfParallelism = 8;

var stopWatch = new Stopwatch();
stopWatch.Start();

var cepList = new List<CepModel>();

Parallel.ForEach(ceps, parallelOptions, cep =>
{
    cepList.Add(new ViaCepService().GetCep(cep));
});

stopWatch.Stop();
Console.WriteLine($"Elapsed Time {stopWatch.ElapsedMilliseconds} ms");

cepList.ToList().ForEach(cep => Console.WriteLine(cep));
Console.ReadKey();
```

> **Atenção:** o `List<T>` não é thread-safe. No exemplo acima, usar `ConcurrentBag<T>` seria mais seguro.

## 2. Parallel.Invoke

Executa múltiplos métodos simultaneamente:

```csharp
Parallel.Invoke(
    () => Metodo1(),
    () => Metodo2(),
    () => Metodo3()
);
```

Útil quando você tem um número fixo de operações independentes para executar em paralelo.

## Quando usar cada um

| Cenário | Usar |
|---|---|
| Processar uma coleção em paralelo | `Parallel.ForEach` |
| Executar N métodos independentes | `Parallel.Invoke` |
| Operações I/O-bound (API, banco) | `Task` com `async/await` (não `Parallel`) |

---

[← Anterior: Paralelismo vs Concorrência](01-paralelismo-vs-concorrencia.md) | [Voltar ao índice](README.md) | [Próximo: Task, async/await →](03-task-async-await.md)
