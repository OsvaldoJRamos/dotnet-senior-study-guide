# Minimal APIs

## What they are

A simplified way to create APIs in ASP.NET Core (.NET 6+) **without controllers**, with less boilerplate. All setup goes in `Program.cs`.

## Basic example

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

## Organization with Route Groups

To avoid polluting Program.cs:

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

## Validation with Filters

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

## Auth, OpenAPI and Rate Limiting

```csharp
app.MapGet("/admin/config", () => Results.Ok(config))
    .RequireAuthorization("AdminPolicy")
    .WithName("ObterConfig")
    .WithOpenApi()
    .RequireRateLimiting("fixed");
```

## Minimal APIs vs Controllers

| Aspect | Minimal APIs | Controllers |
|---------|-------------|-------------|
| Boilerplate | Minimal | More verbose |
| Performance | Slightly better | Slightly worse |
| Organization | Route groups, extension methods | ControllerBase inheritance |
| Filters | Endpoint filters | Action filters (more types) |
| Model binding | Method parameters | `[FromBody]`, `[FromQuery]`, etc. |
| Best for | Simple APIs, microservices | Complex APIs, large projects |

## When to use

- **Minimal APIs**: microservices, simple APIs, when you want less boilerplate
- **Controllers**: complex APIs with many MVC features (model validation, formatters, etc.)
- **Both**: they can coexist in the same project

---

[← Previous: Caching](07-caching.md) | [Next: SignalR →](09-signalr.md) | [Back to index](README.md)
