# Function as a Service (FaaS)

## What it is

FaaS (AWS Lambda, Azure Functions, Google Cloud Functions) are basically **small containers** that execute isolated functions.

## How it works

1. You configure an **event/trigger** (HTTP call, queue message, pub/sub)
2. When the event arrives, a **container spins up** and executes the function
3. After processing, the container can be **shut down** to save resources

```
Event (HTTP, SQS, etc.) --> Container spins up --> Function executes --> Container may shut down
```

## Characteristics

- Pre-configured containers with runtimes (Node.js, Python, Go, C#, Java, etc.)
- Allows installing external dependencies
- Execution environments are **short-lived** — typically recycled after ~5–15 minutes of idleness (AWS Lambda), so don't count on in-memory state across invocations
- The platform may keep the container warm for a while to serve follow-up events (mitigating cold starts)

## What it is suited for

- **Event processing**: queue messages, webhooks
- **Scheduled tasks**: lightweight cron jobs
- **Lightweight APIs**: simple and independent endpoints
- **File processing**: image resizing, ETL

## What it is NOT suited for

- **Monolithic web frameworks** (ASP.NET MVC, Rails, etc.) - technically possible, but not the intended purpose
- **Stateful applications**: FaaS is stateless by nature
- **Long-running processing**: has time limits (see comparison below)
- **Cold start**: first execution can be slow (container spinning up)

### Execution time limits

| Platform | Plan | Max duration |
|----------|------|--------------|
| AWS Lambda | Any | 15 min |
| Azure Functions | Consumption | 5 min default, 10 min max |
| Azure Functions | Premium / Dedicated (App Service) | 30 min default, unbounded configurable |
| Google Cloud Functions (Gen 2) | Any | 60 min (HTTP) / 9 min (event) |

## Example with Azure Functions (C# — isolated worker)

Azure Functions has two .NET models: the legacy **in-process** model (`[FunctionName]`, method-injected `ILogger`) and the modern **isolated worker** model. The in-process model runs inside the host; the isolated worker runs in its own process with full control over DI, middleware, and .NET version. **Microsoft is ending support for the in-process model in November 2026 — new projects should use the isolated worker model.**

```csharp
// ProcessOrderFunction.cs
public class ProcessOrderFunction
{
    private readonly ILogger<ProcessOrderFunction> _logger;

    public ProcessOrderFunction(ILogger<ProcessOrderFunction> logger)
    {
        _logger = logger;
    }

    [Function("ProcessOrder")]
    public async Task Run(
        [QueueTrigger("orders-queue")] string orderJson)
    {
        var order = JsonSerializer.Deserialize<Order>(orderJson);
        _logger.LogInformation("Processing order {OrderId}", order.Id);

        // process the order...
    }
}
```

```csharp
// Program.cs — host setup
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(services =>
    {
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
        // register your dependencies here
    })
    .Build();

host.Run();
```

> See the [Azure Functions isolated worker guide](https://learn.microsoft.com/azure/azure-functions/dotnet-isolated-process-guide) for the full migration story.

## Durable Functions (Azure)

Durable Functions is an extension of Azure Functions for writing **stateful orchestrations** on top of stateless functions. Senior-relevant because most non-trivial serverless workflows need it.

Key building blocks:

- **Orchestrator function**: coordinates the workflow. Deterministic code (no direct I/O) — state is persisted via event sourcing.
- **Activity function**: the actual work (HTTP call, DB write, etc.). Called from the orchestrator.
- **Entity function**: small, addressable piece of state (think actor model) — useful for counters, aggregates, per-user state.

Common patterns:

| Pattern | Use case |
|---------|----------|
| **Function chaining** | Run A → B → C sequentially, passing results |
| **Fan-out / fan-in** | Execute N activities in parallel, aggregate results |
| **Async HTTP APIs** | Long-running job with a status endpoint |
| **Monitoring** | Poll an external resource on a schedule |
| **Human interaction** | Wait for external event (approval) with timeout |

> Use Durable Functions when you need workflows that outlive a single function execution — otherwise queue + function is simpler.

## Cold Start

When a container is created from scratch, there is a delay (**cold start**). Strategies to mitigate:

- **Provisioned concurrency** (AWS) / **Always Ready** (Azure) - keep containers warm
- Use lightweight runtimes (Node.js, Go) vs heavy ones (Java, .NET)
- **AOT compilation** (.NET 8+) - drastically reduces cold start

## Beware of vendor lock-in

The implementation varies between providers (AWS, Azure, GCP). The framework and triggers are platform-specific.

---

[← Previous: CI/CD](01-ci-cd.md) | [Next: Docker and Kubernetes →](03-docker-and-kubernetes.md) | [Back to index](README.md)
