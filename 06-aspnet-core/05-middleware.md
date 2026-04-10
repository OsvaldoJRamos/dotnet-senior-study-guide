# Middleware Pipeline

## O que e Middleware

Middleware sao componentes que formam um **pipeline de processamento** de requisicoes HTTP no ASP.NET Core. Cada middleware pode:

- Processar a requisicao **antes** de passar para o proximo
- Processar a resposta **depois** que o proximo retorna
- **Curto-circuitar** o pipeline (nao chamar o proximo)

```
Request → [Middleware 1] → [Middleware 2] → [Middleware 3] → Endpoint
Response ← [Middleware 1] ← [Middleware 2] ← [Middleware 3] ←
```

## Ordem importa

A ordem em que os middlewares sao registrados define a ordem de execucao. **Ordem errada = bugs sutis**.

```csharp
var app = builder.Build();

// Ordem recomendada pela Microsoft:
app.UseExceptionHandler("/error");   // 1. Captura excecoes
app.UseHsts();                        // 2. HSTS
app.UseHttpsRedirection();            // 3. Redireciona HTTP -> HTTPS
app.UseStaticFiles();                 // 4. Arquivos estaticos (curto-circuita aqui)
app.UseRouting();                     // 5. Determina o endpoint
app.UseCors();                        // 6. CORS
app.UseAuthentication();              // 7. Identifica quem e o usuario
app.UseAuthorization();               // 8. Verifica se pode acessar
app.MapControllers();                 // 9. Executa o endpoint
```

> Se `UseAuthentication` vier **depois** de `UseAuthorization`, a autorizacao nao vai ter o usuario identificado = bug.

## Middleware customizado

### Inline (simples)

```csharp
app.Use(async (context, next) =>
{
    // Antes do proximo middleware
    var sw = Stopwatch.StartNew();
    
    await next(context);
    
    // Depois do proximo middleware
    sw.Stop();
    Console.WriteLine($"{context.Request.Path} levou {sw.ElapsedMilliseconds}ms");
});
```

### Classe (reutilizavel)

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        
        await _next(context);
        
        sw.Stop();
        _logger.LogInformation("{Method} {Path} completed in {Elapsed}ms",
            context.Request.Method, context.Request.Path, sw.ElapsedMilliseconds);
    }
}

// Registro
app.UseMiddleware<RequestTimingMiddleware>();
```

## Map e MapWhen (branching)

Permite criar **ramificacoes** no pipeline para rotas especificas:

```csharp
// Middleware so para rotas que comecam com /api
app.MapWhen(
    context => context.Request.Path.StartsWithSegments("/api"),
    appBuilder => appBuilder.UseMiddleware<ApiKeyMiddleware>()
);

// Branch completa para /health
app.Map("/health", appBuilder =>
{
    appBuilder.Run(async context =>
    {
        await context.Response.WriteAsync("OK");
    });
});
```

## Curto-circuito

Middleware pode parar o pipeline sem chamar `next`:

```csharp
app.Use(async (context, next) =>
{
    if (context.Request.Headers["X-Api-Key"] != "minha-chave")
    {
        context.Response.StatusCode = 401;
        await context.Response.WriteAsync("Unauthorized");
        return; // nao chama next = curto-circuito
    }

    await next(context);
});
```

## Middleware vs Filters

| Aspecto | Middleware | Filter |
|---------|-----------|--------|
| Escopo | **Toda** requisicao HTTP | Apenas requisicoes que chegam ao **MVC/Minimal API** |
| Acesso ao endpoint | Nao tem contexto do endpoint | Tem acesso a action, model binding, etc. |
| Ordem | Definida pelo registro | Definida por ordem + tipo (Authorization, Resource, Action, etc.) |
| Uso ideal | Cross-cutting (logging, CORS, auth) | Logica especifica de MVC (validacao, cache de action) |

## Pontos importantes para entrevista

1. **Ordem de registro = ordem de execucao** — errar a ordem e um bug comum
2. **RequestDelegate** e o delegate que representa o proximo middleware
3. Middlewares sao **singletons** — cuidado com injecao de servicos Scoped (use `InvokeAsync` com parametros)
4. `app.Run()` e um middleware **terminal** — nao chama `next`
5. `app.Use()` pode chamar ou nao `next` — decisao do middleware

---

[← Anterior: Resiliência de APIs](04-resiliencia-de-apis.md) | [Próximo: Background Services →](06-background-services.md) | [Voltar ao índice](README.md)
