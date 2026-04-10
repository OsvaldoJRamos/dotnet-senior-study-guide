# Background Services

## O que sao

Servicos que rodam em **background** no ASP.NET Core, sem depender de requisicoes HTTP. Uteis para:

- Jobs agendados
- Processamento de filas
- Health checks periodicos
- Sincronizacao de dados
- Cache warming

## IHostedService

Interface base com dois metodos:

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

## BackgroundService (mais comum)

Classe abstrata que simplifica a criacao de servicos de longa duracao:

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

// Registro
builder.Services.AddHostedService<FilaProcessorService>();
```

## Cuidado com Scoped services

BackgroundService e **singleton**. Para usar servicos **scoped** (como DbContext), crie um scope manualmente:

```csharp
// ERRADO: injetar DbContext direto no construtor
public class MeuServico : BackgroundService
{
    private readonly AppDbContext _context; // ERRO em runtime!
}

// CORRETO: criar scope
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var scope = _provider.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    // usar context...
}
```

## Job agendado (Timer-based)

```csharp
public class RelatorioService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await GerarRelatorioAsync();
            
            // Executa a cada 1 hora
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

## Com PeriodicTimer (.NET 6+)

Mais preciso que `Task.Delay` para intervalos regulares:

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

## Para jobs mais complexos

Para cenarios que precisam de **agendamento avancado** (cron expressions, persistencia, retentativas), considere:

- **Hangfire** — dashboard, cron jobs, retentativas, persistencia em banco
- **Quartz.NET** — port do Quartz Java, scheduler completo

```csharp
// Hangfire
RecurringJob.AddOrUpdate<RelatorioService>(
    "relatorio-diario",
    service => service.GerarAsync(),
    Cron.Daily(8, 0)); // todo dia as 8h
```

---

[← Anterior: Middleware](05-middleware.md) | [Próximo: Caching →](07-caching.md) | [Voltar ao índice](README.md)
