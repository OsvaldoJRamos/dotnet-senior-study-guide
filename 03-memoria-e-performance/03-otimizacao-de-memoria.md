# Otimização de Memória em .NET

## 1. Use StringBuilder para concatenação

Strings são imutáveis em C#, então cada concatenação cria um novo objeto string.

```csharp
// Ineficiente: pode causar múltiplas alocações grandes no LOH
string result = "Hello" + largeString1 + largeString2;

// Eficiente: usa StringBuilder para evitar alocações grandes
StringBuilder sb = new StringBuilder();
sb.Append("Hello");
sb.Append(largeString1);
sb.Append(largeString2);
string result = sb.ToString(); // Final result without multiple large allocations
```

## 2. Escolha estruturas de dados adequadas

Escolher uma estrutura de dados adequada é um aspecto chave da otimização de memória. **Ao invés de usar objetos e coleções complexas**, que podem consumir mais memória devido a metadados adicionais, prefira estruturas de dados simples como **arrays**, **lists** e **structs**.

### Arrays vs Lists

```csharp
// Uses more memory
List<string> names = new List<string>();
names.Add("John");
names.Add("Doe");

// Uses less memory
string[] names = new string[2];
names[0] = "John";
names[1] = "Doe";
```

O `string[]` array requer menos memória comparado ao `List<string>` porque **não possui estrutura de dados adicional para gerenciar redimensionamento dinâmico**.

**Porém**, isso não significa que você deva sempre usar arrays em vez de lists. Se você precisa frequentemente adicionar novos elementos e reconstruir o array, ou fazer buscas pesadas já disponíveis na list, é melhor escolher a list.

## 3. Use `Span<T>` e `Memory<T>` (alto desempenho)

Para cenários de alta performance, evite alocar arrays novos:

```csharp
// Aloca novo array
byte[] slice = array[10..20]; // cria cópia

// Sem alocação — usa uma "janela" sobre o array original
Span<byte> slice = array.AsSpan(10, 10);
```

## 4. Object Pooling

Reutilize objetos caros em vez de criar e destruir repetidamente:

```csharp
// Microsoft.Extensions.ObjectPool
var pool = new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());

var sb = pool.Get();
try
{
    sb.Append("Hello");
    // usa o StringBuilder
}
finally
{
    pool.Return(sb); // devolve ao pool para reutilização
}
```

## 5. ArrayPool para arrays temporários

```csharp
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(1024); // pega emprestado
try
{
    // usa o buffer
}
finally
{
    pool.Return(buffer); // devolve
}
```

## Técnicas de otimização (checklist)

1. **Cache** — armazene resultados de operações caras
2. **Jobs Assíncronos** — processe trabalho pesado em background
3. **Monitoramento** — use ferramentas para medir métricas reais em produção
4. **CDN no front-end** — sirva assets estáticos mais perto do usuário
5. **Separação de bancos de leitura e escrita** — CQRS para escalar leitura independentemente
6. **ElasticSearch** — para full-text search e busca por relevância

---

[← Anterior: Garbage Collector](02-garbage-collector.md) | [Voltar ao índice](README.md) | [Próximo: Memory Leak →](04-memory-leak.md)
