# CQRS - Command Query Responsibility Segregation

## O que e

Separar as operacoes de **leitura** (Query) e **escrita** (Command) em modelos diferentes. Em vez de um unico modelo que faz tudo, voce tem:

- **Command Model**: otimizado para escrita (normalizado, com validacoes)
- **Query Model**: otimizado para leitura (desnormalizado, rapido)

```
┌─────────┐    Command    ┌──────────────┐    ┌──────────────┐
│ Cliente  │ ───────────→ │ Command Side │ ──→│  Write DB    │
│          │              │ (validacao,  │    │ (normalizado)│
│          │    Query     │  regras)     │    └──────────────┘
│          │ ───────────→ ├──────────────┤           │ sync
│          │ ←─────────── │  Query Side  │ ←─ ┌──────────────┐
└─────────┘              │ (projecoes)  │    │  Read DB     │
                          └──────────────┘    │(desnormaliz.)│
                                              └──────────────┘
```

## Por que usar

1. **Performance**: leituras e escritas tem necessidades diferentes
2. **Escalabilidade**: escalar leitura e escrita independentemente
3. **Simplicidade**: cada lado tem um modelo simples em vez de um modelo complexo que tenta fazer tudo
4. **Seguranca**: separar quem pode ler de quem pode escrever

## Implementacao simples (mesmo banco)

Nao precisa de dois bancos. O nivel mais basico e separar **handlers**:

```csharp
// Command
public record CriarPedidoCommand(string ClienteId, List<ItemDto> Itens);

public class CriarPedidoHandler
{
    private readonly IPedidoRepository _repo;

    public async Task<Guid> Handle(CriarPedidoCommand cmd)
    {
        var pedido = new Pedido(cmd.ClienteId, cmd.Itens);
        await _repo.SalvarAsync(pedido);
        return pedido.Id;
    }
}

// Query
public record ObterPedidoQuery(Guid PedidoId);

public class ObterPedidoHandler
{
    private readonly IDbConnection _db; // Dapper, leitura direta

    public async Task<PedidoDto?> Handle(ObterPedidoQuery query)
    {
        return await _db.QueryFirstOrDefaultAsync<PedidoDto>(
            "SELECT Id, ClienteNome, Total, Status FROM vw_Pedidos WHERE Id = @Id",
            new { Id = query.PedidoId });
    }
}
```

## Com MediatR

```csharp
// Command
public record CriarPedidoCommand(string ClienteId) : IRequest<Guid>;

public class CriarPedidoHandler : IRequestHandler<CriarPedidoCommand, Guid>
{
    public async Task<Guid> Handle(CriarPedidoCommand cmd, CancellationToken ct)
    {
        // ... escrita
    }
}

// Query
public record ObterPedidoQuery(Guid Id) : IRequest<PedidoDto?>;

public class ObterPedidoHandler : IRequestHandler<ObterPedidoQuery, PedidoDto?>
{
    public async Task<PedidoDto?> Handle(ObterPedidoQuery query, CancellationToken ct)
    {
        // ... leitura otimizada
    }
}

// Controller
[HttpPost]
public async Task<IActionResult> Criar(CriarPedidoDto dto)
{
    var id = await _mediator.Send(new CriarPedidoCommand(dto.ClienteId));
    return CreatedAtAction(nameof(Obter), new { id }, null);
}
```

## Event Sourcing (frequentemente combinado com CQRS)

Em vez de salvar o **estado atual**, salva todos os **eventos** que levaram ao estado:

```csharp
// Eventos
public record PedidoCriado(Guid PedidoId, string ClienteId, DateTime Data);
public record ItemAdicionado(Guid PedidoId, string Produto, int Quantidade);
public record PedidoAprovado(Guid PedidoId, DateTime Data);

// O estado e reconstruido "replaying" os eventos:
// PedidoCriado → ItemAdicionado → ItemAdicionado → PedidoAprovado
// Resultado: Pedido com 2 itens, status Aprovado
```

### Vantagens do Event Sourcing

- **Auditoria completa** — historico de tudo que aconteceu
- **Debug** — pode reconstruir o estado em qualquer ponto no tempo
- **Event-driven** — eventos podem alimentar outros sistemas

### Desvantagens

- Complexidade alta
- Queries no banco de eventos sao lentas (precisa de projecoes/read models)
- Versionamento de eventos e complicado

## Niveis de CQRS

| Nivel | Descricao | Complexidade |
|-------|-----------|-------------|
| 1 | Separar handlers de leitura e escrita | Baixa |
| 2 | Modelos diferentes para leitura e escrita | Media |
| 3 | Bancos diferentes (write DB + read DB) | Alta |
| 4 | Event Sourcing + projecoes | Muito alta |

> Comece pelo nivel 1. So suba se tiver necessidade real.

## Quando usar

- Dominio complexo com muitas regras de escrita
- Leitura e escrita com volumes muito diferentes
- Necessidade de escalar leitura independente
- Requisitos de auditoria (event sourcing)

## Quando NAO usar

- CRUDs simples
- Dominio com pouca logica de negocio
- Time pequeno sem experiencia com o padrao

---

[← Anterior: Clean Architecture](07-clean-architecture.md) | [Próximo: Microservices →](09-microservices.md) | [Voltar ao índice](README.md)
