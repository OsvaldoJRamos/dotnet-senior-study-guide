# Minimal APIs

## O que sao

Forma simplificada de criar APIs no ASP.NET Core (.NET 6+) **sem controllers**, com menos boilerplate. Todo o setup fica no `Program.cs`.

## Exemplo basico

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProdutoService, ProdutoService>();

var app = builder.Build();

app.MapGet("/produtos", async (IProdutoService service) =>
    Results.Ok(await service.ListarAsync()));

app.MapGet("/produtos/{id:int}", async (int id, IProdutoService service) =>
    await service.ObterAsync(id) is Produto p
        ? Results.Ok(p)
        : Results.NotFound());

app.MapPost("/produtos", async (CriarProdutoDto dto, IProdutoService service) =>
{
    var produto = await service.CriarAsync(dto);
    return Results.Created($"/produtos/{produto.Id}", produto);
});

app.MapPut("/produtos/{id:int}", async (int id, AtualizarProdutoDto dto, IProdutoService service) =>
{
    await service.AtualizarAsync(id, dto);
    return Results.NoContent();
});

app.MapDelete("/produtos/{id:int}", async (int id, IProdutoService service) =>
{
    await service.RemoverAsync(id);
    return Results.NoContent();
});

app.Run();
```

## Organizacao com Route Groups

Para nao poluir o Program.cs:

```csharp
// Program.cs
app.MapGroup("/api/produtos").MapProdutoEndpoints();
app.MapGroup("/api/clientes").MapClienteEndpoints();

// Endpoints/ProdutoEndpoints.cs
public static class ProdutoEndpoints
{
    public static RouteGroupBuilder MapProdutoEndpoints(this RouteGroupBuilder group)
    {
        group.MapGet("/", Listar);
        group.MapGet("/{id:int}", ObterPorId);
        group.MapPost("/", Criar);
        return group;
    }

    private static async Task<IResult> Listar(IProdutoService service)
        => Results.Ok(await service.ListarAsync());

    private static async Task<IResult> ObterPorId(int id, IProdutoService service)
        => await service.ObterAsync(id) is Produto p
            ? Results.Ok(p) : Results.NotFound();

    private static async Task<IResult> Criar(CriarProdutoDto dto, IProdutoService service)
    {
        var produto = await service.CriarAsync(dto);
        return Results.Created($"/produtos/{produto.Id}", produto);
    }
}
```

## Validacao com Filters

```csharp
app.MapPost("/produtos", async (CriarProdutoDto dto, IProdutoService service) =>
{
    var produto = await service.CriarAsync(dto);
    return Results.Created($"/produtos/{produto.Id}", produto);
})
.AddEndpointFilter(async (context, next) =>
{
    var dto = context.GetArgument<CriarProdutoDto>(0);
    if (string.IsNullOrWhiteSpace(dto.Nome))
        return Results.BadRequest("Nome é obrigatório");
    return await next(context);
});
```

## Auth, OpenAPI e Rate Limiting

```csharp
app.MapGet("/admin/config", () => Results.Ok(config))
    .RequireAuthorization("AdminPolicy")
    .WithName("ObterConfig")
    .WithOpenApi()
    .RequireRateLimiting("fixed");
```

## Minimal APIs vs Controllers

| Aspecto | Minimal APIs | Controllers |
|---------|-------------|-------------|
| Boilerplate | Minimo | Mais verbose |
| Performance | Ligeiramente melhor | Ligeiramente pior |
| Organizacao | Route groups, extension methods | Herança de ControllerBase |
| Filters | Endpoint filters | Action filters (mais tipos) |
| Model binding | Parametros do metodo | `[FromBody]`, `[FromQuery]`, etc. |
| Melhor para | APIs simples, microservices | APIs complexas, projetos grandes |

## Quando usar

- **Minimal APIs**: microservices, APIs simples, quando quer menos boilerplate
- **Controllers**: APIs complexas com muitas features MVC (model validation, formatters, etc.)
- **Ambos**: podem coexistir no mesmo projeto

---

[← Anterior: Caching](07-caching.md) | [Próximo: SignalR →](09-signalr.md) | [Voltar ao índice](README.md)
