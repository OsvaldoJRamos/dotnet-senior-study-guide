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
public class MyService : IHostedService
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("Service started");
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("Service stopped");
        return Task.CompletedTask;
    }
}
```

## BackgroundService (most common)

Abstract class that simplifies the creation of long-running services:

```csharp
public class QueueProcessorService : BackgroundService
{
    private readonly ILogger<QueueProcessorService> _logger;
    private readonly IServiceProvider _provider;

    public QueueProcessorService(ILogger<QueueProcessorService> logger, IServiceProvider provider)
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
                var service = scope.ServiceProvider.GetRequiredService<IQueueProcessor>();
                
                await service.ProcessNextAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing queue");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

// Registration
builder.Services.AddHostedService<QueueProcessorService>();
```

## Be careful with Scoped services

BackgroundService is a **singleton**. To use **scoped** services (like DbContext), create a scope manually:

```csharp
// WRONG: injecting DbContext directly in the constructor
public class MyService : BackgroundService
{
    private readonly AppDbContext _context; // ERROR at runtime!
}

// CORRECT: create a scope
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var scope = _provider.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    // use context...
}
```

## Scheduled job (Timer-based)

```csharp
public class ReportService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await GenerateReportAsync();
            
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
        await ProcessAsync();
    }
}
```

## For more complex jobs

For scenarios that need **advanced scheduling** (cron expressions, persistence, retries), consider:

- **Hangfire** — dashboard, cron jobs, retries, database persistence
- **Quartz.NET** — port of Quartz Java, full scheduler

```csharp
// Hangfire
RecurringJob.AddOrUpdate<ReportService>(
    "daily-report",
    service => service.GenerateAsync(),
    Cron.Daily(8, 0)); // every day at 8am
```

---

[← Previous: Middleware](05-middleware.md) | [Next: Caching →](07-caching.md) | [Back to index](README.md)
