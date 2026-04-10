# Paralelismo vs Concorrência

## Diferença

- **Concorrência:** várias tarefas *em progresso* ao mesmo tempo (pode ser intercalado em uma CPU)
- **Paralelismo:** várias tarefas *executadas literalmente ao mesmo tempo* (requer múltiplos núcleos)

## Quando NÃO usar paralelismo

- **Em tarefas I/O-bound** (como chamadas de API ou banco de dados): prefira `async/await`
- **Em tarefas curtas demais**: o overhead do paralelismo pode deixar tudo mais lento
- **Em códigos com muitos acessos a recursos compartilhados** (como listas): o paralelismo pode gerar problemas de concorrência (race conditions)

## Como o C# decide quantas threads usar?

`Parallel` e `Task` usam o **ThreadPool**, que é gerenciado automaticamente.

Você pode limitar com `ParallelOptions`:

```csharp
var options = new ParallelOptions { MaxDegreeOfParallelism = 4 };
Parallel.ForEach(lista, options, item => Processar(item));
```

---

[Voltar ao índice](README.md) | [Próximo: Parallel.ForEach e Invoke →](02-parallel-foreach-invoke.md)
