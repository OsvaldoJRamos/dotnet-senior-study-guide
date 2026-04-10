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
- Container tends to be **small** and stay up for **short** periods (hours, at most days)
- The platform keeps the container active for a while longer in case other events are waiting

## What it is suited for

- **Event processing**: queue messages, webhooks
- **Scheduled tasks**: lightweight cron jobs
- **Lightweight APIs**: simple and independent endpoints
- **File processing**: image resizing, ETL

## What it is NOT suited for

- **Monolithic web frameworks** (ASP.NET MVC, Rails, etc.) - technically possible, but not the intended purpose
- **Stateful applications**: FaaS is stateless by nature
- **Long-running processing**: has time limits (e.g.: Lambda = 15min max)
- **Cold start**: first execution can be slow (container spinning up)

## Example with Azure Functions (C#)

```csharp
public class ProcessarPedidoFunction
{
    [FunctionName("ProcessarPedido")]
    public async Task Run(
        [QueueTrigger("pedidos-queue")] string pedidoJson,
        ILogger log)
    {
        var pedido = JsonSerializer.Deserialize<Pedido>(pedidoJson);
        log.LogInformation($"Processando pedido {pedido.Id}");
        
        // processa o pedido...
    }
}
```

## Cold Start

When a container is created from scratch, there is a delay (**cold start**). Strategies to mitigate:

- **Provisioned concurrency** (AWS) / **Always Ready** (Azure) - keep containers warm
- Use lightweight runtimes (Node.js, Go) vs heavy ones (Java, .NET)
- **AOT compilation** (.NET 8+) - drastically reduces cold start

## Beware of vendor lock-in

The implementation varies between providers (AWS, Azure, GCP). The framework and triggers are platform-specific.

---

[← Previous: CI/CD](01-ci-cd.md) | [Next: Docker and Kubernetes →](03-docker-e-kubernetes.md) | [Back to index](README.md)
