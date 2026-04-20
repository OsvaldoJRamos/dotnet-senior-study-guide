# Minimal APIs

## What they are

A simplified way to create APIs in ASP.NET Core (.NET 6+) **without controllers**, with less boilerplate. All setup goes in `Program.cs`.

## Basic example

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

app.MapGet("/products", async (IProductService service) =>
    Results.Ok(await service.ListAsync()));

app.MapGet("/products/{id:int}", async (int id, IProductService service) =>
    await service.GetAsync(id) is Product p
        ? Results.Ok(p)
        : Results.NotFound());

app.MapPost("/products", async (CreateProductDto dto, IProductService service) =>
{
    var product = await service.CreateAsync(dto);
    return Results.Created($"/products/{product.Id}", product);
});

app.MapPut("/products/{id:int}", async (int id, UpdateProductDto dto, IProductService service) =>
{
    await service.UpdateAsync(id, dto);
    return Results.NoContent();
});

app.MapDelete("/products/{id:int}", async (int id, IProductService service) =>
{
    await service.RemoveAsync(id);
    return Results.NoContent();
});

app.Run();
```

## Organization with Route Groups

To avoid polluting Program.cs:

```csharp
// Program.cs
app.MapGroup("/api/products").MapProductEndpoints();
app.MapGroup("/api/customers").MapCustomerEndpoints();

// Endpoints/ProductEndpoints.cs
public static class ProductEndpoints
{
    public static RouteGroupBuilder MapProductEndpoints(this RouteGroupBuilder group)
    {
        group.MapGet("/", List);
        group.MapGet("/{id:int}", GetById);
        group.MapPost("/", Create);
        return group;
    }

    private static async Task<IResult> List(IProductService service)
        => Results.Ok(await service.ListAsync());

    private static async Task<IResult> GetById(int id, IProductService service)
        => await service.GetAsync(id) is Product p
            ? Results.Ok(p) : Results.NotFound();

    private static async Task<IResult> Create(CreateProductDto dto, IProductService service)
    {
        var product = await service.CreateAsync(dto);
        return Results.Created($"/products/{product.Id}", product);
    }
}
```

## Validation with Filters

```csharp
app.MapPost("/products", async (CreateProductDto dto, IProductService service) =>
{
    var product = await service.CreateAsync(dto);
    return Results.Created($"/products/{product.Id}", product);
})
.AddEndpointFilter(async (context, next) =>
{
    var dto = context.GetArgument<CreateProductDto>(0);
    if (string.IsNullOrWhiteSpace(dto.Name))
        return Results.BadRequest("Name is required");
    return await next(context);
});
```

## Auth, OpenAPI and Rate Limiting

```csharp
app.MapGet("/admin/config", () => Results.Ok(config))
    .RequireAuthorization("AdminPolicy")
    .WithName("GetConfig")
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
