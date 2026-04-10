# Background Services

## What they are

Services that run in the **background** in ASP.NET Core, without depending on HTTP requests. Useful for:

- Scheduled jobs
- Queue processing
- Periodic health checks
- Data synchronization
- Cache warming

## IHostedService

Base interface with two methods:

```csharp
public class MeuServico : IHostedService
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("Servico iniciado");
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("Servico parado");
        return Task.CompletedTask;
    }
}
```

## BackgroundService (most common)

Abstract class that simplifies the creation of long-running services:

```csharp
public class FilaProcessorService : BackgroundService
{
    private readonly ILogger<FilaProcessorService> _logger;
    private readonly IServiceProvider _provider;

    public FilaProcessorService(ILogger<FilaProcessorService> logger, IServiceProvider provider)
    {
        _logger = logger;
        _provider = provider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _provider.CreateScope();
                var servico = scope.ServiceProvider.GetRequiredService<IProcessadorFila>();
                
                await servico.ProcessarProximoAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Erro ao processar fila");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

// Registration
builder.Services.AddHostedService<FilaProcessorService>();
```

## Be careful with Scoped services

BackgroundService is a **singleton**. To use **scoped** services (like DbContext), create a scope manually:

```csharp
// WRONG: injecting DbContext directly in the constructor
public class MeuServico : BackgroundService
{
    private readonly AppDbContext _context; // ERRO em runtime!
}

// CORRECT: create a scope
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var scope = _provider.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    // usar context...
}
```

## Scheduled job (Timer-based)

```csharp
public class RelatorioService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await GerarRelatorioAsync();
            
            // Runs every 1 hour
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

## With PeriodicTimer (.NET 6+)

More precise than `Task.Delay` for regular intervals:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));

    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        await ProcessarAsync();
    }
}
```

## For more complex jobs

For scenarios that need **advanced scheduling** (cron expressions, persistence, retries), consider:

- **Hangfire** — dashboard, cron jobs, retries, database persistence
- **Quartz.NET** — port of Quartz Java, full scheduler

```csharp
// Hangfire
RecurringJob.AddOrUpdate<RelatorioService>(
    "relatorio-diario",
    service => service.GerarAsync(),
    Cron.Daily(8, 0)); // todo dia as 8h
```

---

[← Previous: Middleware](05-middleware.md) | [Next: Caching →](07-caching.md) | [Back to index](README.md)
