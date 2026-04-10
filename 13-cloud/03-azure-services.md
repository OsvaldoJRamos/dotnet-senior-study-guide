# Azure - Main Services

## Compute

### App Service

Managed platform for hosting **web apps, APIs, and backends**. No infrastructure management required.

- Supports .NET, Node.js, Python, Java, PHP
- Auto-scale, custom domains, SSL
- Deployment slots (staging → swap → production)
- **Best option for .NET APIs in most cases**

### Azure Container Apps

Managed containers with **auto-scaling** (including scale to zero):

- Based on Kubernetes (KEDA + Dapr)
- Ideal for microservices and containerized APIs
- Simpler than AKS, more powerful than App Service

### AKS (Azure Kubernetes Service)

Managed Kubernetes. Full control over orchestration:

- Control plane managed by Microsoft
- You manage worker nodes
- Ideal for complex scenarios that require native K8s

### Azure Functions

FaaS — serverless functions (see [FaaS](../08-devops/02-faas.md)):

- Triggers: HTTP, Queue, Timer, Blob, Event Grid
- Consumption plan: pay per execution
- Premium plan: no cold start

### When to Use What

| Scenario | Service |
|----------|---------|
| Simple .NET API | App Service |
| Containerized microservices | Container Apps |
| Complex K8s, full control | AKS |
| Isolated event/trigger | Azure Functions |

## Storage

### Blob Storage

Equivalent to S3. Stores objects (files, images, backups):

```csharp
var blobClient = new BlobClient(connectionString, "container", "arquivo.pdf");

// Upload
await blobClient.UploadAsync(stream, overwrite: true);

// Download
var response = await blobClient.DownloadContentAsync();
var conteudo = response.Value.Content;

// SAS Token (temporary URL)
var sasUri = blobClient.GenerateSasUri(BlobSasPermissions.Read,
    DateTimeOffset.UtcNow.AddHours(1));
```

| Tier | Use | Cost |
|------|-----|------|
| **Hot** | Frequent access | $$ |
| **Cool** | Rare access (30+ days) | $ |
| **Archive** | Long-term archive | ¢ |

## Database

### Azure SQL

Managed SQL Server. No VM management, automatic backups:

- Compatible with on-premises SQL Server
- Elastic pools for multiple databases
- Geo-replication for disaster recovery

### Cosmos DB

Globally distributed multi-model NoSQL database:

- APIs: SQL, MongoDB, Cassandra, Gremlin, Table
- Latency < 10ms at p99
- Multi-region with configurable consistency levels
- Expensive — use only when you need global scale

## Messaging

### Azure Service Bus

Enterprise message broker:

- **Queues**: point-to-point
- **Topics/Subscriptions**: pub/sub
- Supports transactions, sessions, dead-letter queue
- Equivalent to managed RabbitMQ

```csharp
// Send
var sender = serviceBusClient.CreateSender("pedidos-queue");
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.Serialize(pedido)));

// Receive
var processor = serviceBusClient.CreateProcessor("pedidos-queue");
processor.ProcessMessageAsync += async args =>
{
    var pedido = JsonSerializer.Deserialize<Pedido>(
        args.Message.Body.ToString());
    await args.CompleteMessageAsync(args.Message);
};
```

### Azure Event Grid

**Event** routing between Azure services:

```
[Blob created] → [Event Grid] → [Azure Function processes]
[Resource changed] → [Event Grid] → [Logic App notifies]
```

## Identity

### Azure AD / Entra ID

Microsoft's identity provider. Manages authentication and authorization:

- SSO (Single Sign-On)
- OAuth 2.0 / OpenID Connect
- Managed Identities (no secret management needed)

### Managed Identity

Managed identity for Azure services to access other services **without secrets**:

```csharp
// Instead of a connection string with password:
var credential = new DefaultAzureCredential(); // uses managed identity
var client = new BlobServiceClient(new Uri("https://mystorage.blob.core.windows.net"), credential);
```

> **Always prefer Managed Identity** over connection strings with passwords.

## Monitoring

### Application Insights

Integrated APM (Application Performance Monitoring):

- Request tracking
- Dependency tracking (SQL, HTTP, Redis)
- Exception logging
- Custom metrics and events
- Live metrics stream

```csharp
builder.Services.AddApplicationInsightsTelemetry();

// Custom tracking
var telemetry = app.Services.GetRequiredService<TelemetryClient>();
telemetry.TrackEvent("PedidoCriado", new Dictionary<string, string>
{
    { "PedidoId", pedido.Id.ToString() },
    { "Valor", pedido.Total.ToString() }
});
```

### Azure Monitor

Unified monitoring platform:
- Metrics from all Azure services
- Log Analytics (KQL queries)
- Alerts and action groups

## Networking

### Azure API Management (APIM)

Managed API Gateway:
- Rate limiting, throttling
- Centralized authentication
- Request/response transformation
- Developer portal

### Azure Front Door

CDN + WAF + Global Load Balancer:
- Global content distribution
- DDoS protection
- SSL offloading

## Typical Azure Architecture

```
[Azure Front Door / CDN]
         ↓
[API Management]
         ↓
[App Service / Container Apps]     (API)
    ↓            ↓
[Azure SQL]  [Redis Cache]         (data)
[Service Bus] → [Azure Functions]  (async)
[Blob Storage]                     (files)
[Application Insights]             (monitoring)
[Key Vault]                        (secrets)
```

---

[← Previous: AWS In Depth](02-aws-in-depth.md) | [Back to index](README.md)
