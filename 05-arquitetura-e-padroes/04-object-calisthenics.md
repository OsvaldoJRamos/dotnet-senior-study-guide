# Object Calisthenics

Object Calisthenics é um conjunto de **boas práticas de programação orientada a objetos** criado por Jeff Bay. A ideia é treinar os desenvolvedores a escrever código limpo, modular e de fácil manutenção, por meio de **restrições autoimpostas** que promovem um estilo de codificação mais disciplinado.

Essas regras combinam bem com os princípios de **Clean Code**, de Robert C. Martin, pois forçam o código a ser **simples, coeso, legível e testável**.

## As 9 regras

### 1. Only One Level Of Indentation Per Method
**Evite múltiplos níveis de indentação**. Em vez disso, **extraia código para métodos privados** ou use early returns.

### 2. Don't Use The ELSE Keyword
A palavra `else` pode indicar lógica desnecessária. Use **early return** para simplificar o fluxo.

### 3. Wrap All Primitives And Strings
Evite o uso de tipos primitivos diretamente quando eles têm **significado de negócio**. Use objetos de valor (`Value Object`).

### 4. First Class Collections
Evite usar listas, arrays ou dicionários diretamente. Encapsule dentro de uma classe.

**Exemplo ruim:**
```csharp
public class Order
{
    public List<Item> Items { get; set; }
}
```

**Exemplo bom:**
```csharp
public class OrderItems
{
    private readonly List<Item> _items = new();

    public void Add(Item item)
    {
        _items.Add(item);
    }

    public IEnumerable<Item> All() => _items;
}

public class Order
{
    public OrderItems Items { get; }

    public Order(OrderItems items)
    {
        Items = items;
    }
}
```

### 5. One Dot Per Line
Evite encadear muitas chamadas de método (Law of Demeter). Cada linha deve ter no máximo um "ponto" de navegação.

### 6. Don't Abbreviate
Use nomes descritivos. Abreviações dificultam a leitura.

### 7. Keep All Entities Small
Mantenha classes e métodos pequenos e focados.

### 8. No Classes With More Than Two Instance Variables
Limitar variáveis de instância força a decomposição em objetos menores e mais coesos.

**Exemplo ruim:**
```csharp
public class ReportGenerator
{
    private string _title;
    private string _author;
    private DateTime _created;
    private List<string> _sections;
    private string _footer;
}
```

**Exemplo bom — divida em objetos menores e mais coesos:**
```csharp
public class ReportMetadata
{
    public string Title { get; }
    public string Author { get; }
    public DateTime Created { get; }

    public ReportMetadata(string title, string author, DateTime created)
    {
        Title = title;
        Author = author;
        Created = created;
    }
}

public class Report
{
    private readonly ReportMetadata _metadata;
    private readonly List<string> _sections;

    public Report(ReportMetadata metadata, List<string> sections)
    {
        _metadata = metadata;
        _sections = sections;
    }
}
```

### 9. No Getters/Setters/Properties
Evite expor estado interno. Use métodos que executam ações (ver [Tell, Don't Ask](05-tell-dont-ask.md)).

---

[← Anterior: KISS, DRY e YAGNI](03-kiss-dry-yagni.md) | [Voltar ao índice](README.md) | [Próximo: Tell, Don't Ask →](05-tell-dont-ask.md)
