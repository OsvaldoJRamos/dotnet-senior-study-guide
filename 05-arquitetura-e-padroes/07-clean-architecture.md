# Clean Architecture

## O que e

Arquitetura proposta por Robert C. Martin (Uncle Bob) que organiza o codigo em **camadas concentricas**, onde a **dependencia aponta para dentro** — camadas externas dependem das internas, nunca o contrario.

```
┌─────────────────────────────────────┐
│          Infrastructure             │  Frameworks, DB, APIs externas
│  ┌───────────────────────────────┐  │
│  │        Application            │  │  Use Cases, DTOs, Services
│  │  ┌─────────────────────────┐  │  │
│  │  │        Domain            │  │  │  Entidades, Value Objects, Interfaces
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

## Camadas

### 1. Domain (centro)

A camada mais interna. **Nao depende de nada externo**. Contem:

- Entidades e Aggregates
- Value Objects
- Domain Events
- Interfaces de repositorio (contratos, nao implementacao)
- Domain Services
- Enums e Exceptions do dominio

```csharp
// Domain/Entities/Pedido.cs
public class Pedido
{
    public Guid Id { get; private set; }
    public decimal Total { get; private set; }
    public StatusPedido Status { get; private set; }

    public void Aprovar()
    {
        if (Status != StatusPedido.Pendente)
            throw new DomainException("Só é possível aprovar pedidos pendentes");
        Status = StatusPedido.Aprovado;
    }
}

// Domain/Interfaces/IPedidoRepository.cs
public interface IPedidoRepository
{
    Task<Pedido?> ObterPorIdAsync(Guid id);
    Task SalvarAsync(Pedido pedido);
}
```

### 2. Application (use cases)

Orquestra o fluxo. Contem:

- Use Cases / Application Services
- DTOs (entrada e saida)
- Interfaces de servicos externos
- Validacao de input
- Mapeamento entre DTO e entidade

```csharp
// Application/UseCases/AprovarPedidoUseCase.cs
public class AprovarPedidoUseCase
{
    private readonly IPedidoRepository _repo;
    private readonly INotificacaoService _notificacao;

    public AprovarPedidoUseCase(IPedidoRepository repo, INotificacaoService notificacao)
    {
        _repo = repo;
        _notificacao = notificacao;
    }

    public async Task ExecutarAsync(Guid pedidoId)
    {
        var pedido = await _repo.ObterPorIdAsync(pedidoId)
            ?? throw new NotFoundException("Pedido não encontrado");

        pedido.Aprovar(); // regra de negocio no dominio

        await _repo.SalvarAsync(pedido);
        await _notificacao.EnviarAsync($"Pedido {pedidoId} aprovado");
    }
}
```

### 3. Infrastructure (borda)

Implementacoes concretas. Contem:

- Repositorios (EF Core, Dapper)
- Servicos externos (email, storage, APIs)
- Configuracao de banco de dados
- Mapeamentos ORM

```csharp
// Infrastructure/Repositories/PedidoRepository.cs
public class PedidoRepository : IPedidoRepository
{
    private readonly AppDbContext _context;

    public async Task<Pedido?> ObterPorIdAsync(Guid id)
        => await _context.Pedidos.FindAsync(id);

    public async Task SalvarAsync(Pedido pedido)
    {
        _context.Pedidos.Update(pedido);
        await _context.SaveChangesAsync();
    }
}
```

### 4. Presentation (API/UI)

- Controllers / Minimal APIs
- View Models
- Configuracao de DI
- Middlewares

## Regra de dependencia

```
Presentation → Application → Domain ← Infrastructure
                                ↑
                    Infrastructure implementa interfaces do Domain
```

- **Domain** nao referencia nenhum outro projeto
- **Application** referencia apenas Domain
- **Infrastructure** referencia Domain (para implementar interfaces)
- **Presentation** referencia Application e Infrastructure (para DI)

## Estrutura tipica de Solution

```
src/
├── MyApp.Domain/
├── MyApp.Application/
├── MyApp.Infrastructure/
└── MyApp.Api/
tests/
├── MyApp.Domain.Tests/
├── MyApp.Application.Tests/
└── MyApp.Integration.Tests/
```

## Clean Architecture vs outras

| Aspecto | Clean Architecture | N-Layer tradicional | Vertical Slices |
|---------|-------------------|--------------------|--------------------|
| Dependencia | De fora pra dentro | De cima pra baixo | Por feature |
| Domain | Centro, isolado | Meio, acoplado ao DB | Dentro da feature |
| Testabilidade | Alta | Media | Alta |
| Complexidade | Media-Alta | Baixa | Media |
| Quando usar | Dominio complexo | CRUD simples | Dominio medio |

## Quando NAO usar

- Apps CRUD simples — overengineering
- Prototipos ou MVPs — muito overhead inicial
- Times pequenos com dominio simples — adiciona camadas desnecessarias

---

[← Anterior: SAGA Pattern](06-saga-pattern.md) | [Próximo: CQRS →](08-cqrs.md) | [Voltar ao índice](README.md)
