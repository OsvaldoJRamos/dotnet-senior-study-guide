# Yield Return

## O que é

O `yield return` permite criar iteradores que retornam elementos **um por vez**, sob demanda, sem precisar criar uma lista completa na memória.

## Comparação: sem yield vs com yield

### Sem yield (cria lista inteira na memória):
```csharp
List<int> tempResult = new();
foreach(var item in collection)
{
    tempResult.Add(item);
}
return tempResult;
```

### Com yield (retorna elemento por elemento):
```csharp
foreach(var item in collection)
{
    yield return item;
}
```

## Quando usar

O uso de `yield return` é indicado quando **não será necessário utilizar todo o retorno da lista**. Caso o consumidor da função necessite de usar todos os itens da lista, seu uso pode não ser recomendado, pois pode piorar a performance.

## Exemplo 1: Economia de recursos

### Sem yield — carrega tudo na memória:
```csharp
// Retorna um milhão de itens que podem ser iterados
List<object> GetAllItems()
{
    List<object> millionCustomers;
    database.LoadMillionCustomerRecords(millionCustomers);
    return millionCustomers;
}

// Chamada
int num = 0;
foreach(var itm in GetAllItems())
{
    num++;
    if (num == 5)
        break;
}
// Nota: um milhão de itens são retornados do banco, mas somente 5 são usados.
```

### Com yield — carrega sob demanda:
```csharp
// Retorna cada item em cada chamada
IEnumerable<object> IterateOverItems()
{
    for (int i = 0; i < database.Customers.Count(); ++i)
        yield return database.Customers[i];
}

// Chamada
int num = 0;
foreach(var itm in IterateOverItems())
{
    num++;
    if (num == 5)
        break;
}
// Somente executa os 5 itens dentre um milhão existente
```

Para esse cenário o uso do yield seria **muito mais performático**.

## Exemplo 2: Interface responsiva

Imagine que uma lista está sendo retornada por um programa com interface com usuário. Sem o yield, todos os dados precisam ser retornados para serem mostrados na interface. Isso pode levar alguns segundos, quem sabe até em casos extremos, minutos. O usuário vai achar que deu "pau".

Com o yield é possível pegar elemento por elemento e já ir mostrando para o usuário sem bloquear a interface.

## yield break

Usado para encerrar a iteração antecipadamente:

```csharp
IEnumerable<int> NumerosPares(int max)
{
    for (int i = 0; i <= max; i++)
    {
        if (i > 100)
            yield break; // para de gerar elementos
        
        if (i % 2 == 0)
            yield return i;
    }
}
```

---

[← Anterior: ICollection e IList](02-icollection-ilist.md) | [Voltar ao índice](README.md)
