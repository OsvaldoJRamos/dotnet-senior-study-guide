# Design Patterns (Padrões de Projeto)

Os padrões de projeto do GoF (Gang of Four) são soluções reutilizáveis para problemas comuns em design de software.

**Referência completa:** [Refactoring Guru - Design Patterns](https://refactoring.guru/pt-br/design-patterns/what-is-pattern)

## Categorias

### Criacionais (como criar objetos)

| Padrão | Propósito | Exemplo de uso |
|---|---|---|
| **Singleton** | Garante uma única instância de uma classe | Logger, configuração |
| **Factory Method** | Delega a criação de objetos para subclasses | Criar notificações (email, SMS, push) |
| **Abstract Factory** | Cria famílias de objetos relacionados | UI cross-platform |
| **Builder** | Constrói objetos complexos passo a passo | Construir queries, DTOs complexos |
| **Prototype** | Clona objetos existentes | Cópia de configurações |

### Estruturais (como compor objetos)

| Padrão | Propósito | Exemplo de uso |
|---|---|---|
| **Adapter** | Converte a interface de uma classe em outra | Integrar bibliotecas de terceiros |
| **Decorator** | Adiciona responsabilidades dinamicamente | Adicionar logging, caching a serviços |
| **Facade** | Interface simplificada para um subsistema complexo | API gateway, serviço de fachada |
| **Proxy** | Controlador de acesso a um objeto | Lazy loading, caching, controle de acesso |
| **Composite** | Trata objetos individuais e composições uniformemente | Menus, estruturas de árvore |

### Comportamentais (como objetos se comunicam)

| Padrão | Propósito | Exemplo de uso |
|---|---|---|
| **Strategy** | Define família de algoritmos intercambiáveis | Cálculo de frete, validações |
| **Observer** | Notifica objetos sobre mudanças de estado | Eventos, pub/sub |
| **Command** | Encapsula uma ação como objeto | Undo/redo, filas de comandos |
| **Template Method** | Define esqueleto de algoritmo, subclasses definem passos | Processamento de dados |
| **Chain of Responsibility** | Passa pedido por uma cadeia de handlers | Middleware, pipeline de validação |
| **Mediator** | Centraliza comunicação entre objetos | MediatR, event bus |

## Padrões mais usados no dia a dia .NET

### Singleton
```csharp
// No .NET moderno, use DI com AddSingleton
builder.Services.AddSingleton<IMeuServico, MeuServico>();
```

### Strategy
```csharp
public interface ICalculoFrete
{
    decimal Calcular(Pedido pedido);
}

public class FreteCorreios : ICalculoFrete { ... }
public class FreteSedex : ICalculoFrete { ... }

// Uso via DI - a estratégia é injetada
public class PedidoService
{
    private readonly ICalculoFrete _frete;
    public PedidoService(ICalculoFrete frete) => _frete = frete;
}
```

### Repository
```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(Guid id);
}
```

---

[← Anterior: SOLID](01-solid.md) | [Voltar ao índice](README.md) | [Próximo: KISS, DRY e YAGNI →](03-kiss-dry-yagni.md)
