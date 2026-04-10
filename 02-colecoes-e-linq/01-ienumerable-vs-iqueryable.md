# IEnumerable vs IQueryable

## Diferença fundamental

O **IEnumerable** trabalha com todos os dados da memória, já o **IQueryable** cria uma consulta que será executada posteriormente. Geralmente a consulta é executada quando usamos `.ToList()`.

- **IEnumerable** é mais adequado para consultar dados que já estão na memória (uma lista, por exemplo).
- **IQueryable** é mais adequado para usar junto ao Entity Framework para consulta em bancos de dados, por exemplo.

## IEnumerable x List

O uso do **IEnumerable** é preferível pois está se usando uma interface ao invés de uma classe concreta. Além disso, o IEnumerable posterga as operações até o momento final (execução diferida / deferred execution).

### Exemplo com IEnumerable (execução diferida):

```csharp
private void TesteIEnumerable()
{
    var nomes = new List<string> { "Luis", "João", "Ricardo", "Alexandre" };
    IEnumerable<string> nomesContenhamLetraO = nomes.Where(x => x.Contains("o"));
    nomes[0] = "Marcos";

    foreach (var nome in nomesContenhamLetraO)
    {
        Console.WriteLine(nome);
    }
}
```

**Saída:**
```
Marcos
João
Ricardo
```

Note que mesmo mudando o nome de "Luis" para "Marcos" **após** ter criado o objeto IEnumerable, ele mostrou no console o nome "Marcos". Isso porque o IEnumerable somente foi executado dentro do loop do foreach, e mudamos o nome antes.

### Exemplo com List (execução imediata):

```csharp
private void TesteList()
{
    var nomes = new List<string> { "Luis", "João", "Ricardo", "Alexandre" };
    List<string> nomesContenhamLetraO = nomes.Where(x => x.Contains("o")).ToList();
    nomes[0] = "Marcos";

    foreach (var nome in nomesContenhamLetraO)
    {
        Console.WriteLine(nome);
    }
}
```

**Saída:**
```
João
Ricardo
```

Neste segundo exemplo o objeto List já havia sido criado e armazenado em memória com o nome "Luis". A mudança para "Marcos" não afetou o resultado porque a query já tinha sido materializada pelo `.ToList()`.

## IQueryable — quando usar

```csharp
// IQueryable traduz a expressão LINQ para SQL no banco
IQueryable<Produto> query = context.Produtos
    .Where(p => p.Preco > 100)
    .OrderBy(p => p.Nome);

// A query SQL só é executada aqui:
var resultado = query.ToList();
```

A vantagem é que o **filtro é feito no banco**, não na aplicação. Com IEnumerable, todos os registros seriam trazidos para a memória e filtrados lá — muito menos eficiente.

## Resumo

| Característica | IEnumerable | IQueryable |
|---|---|---|
| Onde executa o filtro | Na memória (C#) | No servidor (SQL) |
| Melhor para | Coleções em memória | Consultas a banco de dados |
| Execução | Diferida (lazy) | Diferida (lazy) |
| Provider | LINQ to Objects | LINQ to SQL/Entities |
| Namespace | `System.Collections.Generic` | `System.Linq` |

---

[Voltar ao índice](README.md) | [Próximo: ICollection e IList →](02-icollection-ilist.md)
