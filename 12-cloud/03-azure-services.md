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

FaaS — serverless functions (see [FaaS](../11-devops/02-faas.md)):

- Triggers: HTTP, Queue, Timer, Blob, Event Grid
- Consumption plan: pay per execution
- Premium plan: minimal cold starts (pre-warmed instances)

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
var blobClient = new BlobClient(connectionString, "container", "file.pdf");

// Upload
await blobClient.UploadAsync(stream, overwrite: true);

// Download
var response = await blobClient.DownloadContentAsync();
var content = response.Value.Content;

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

#### Cosmos DB consistency levels

From strongest (safest, most expensive/latent) to weakest (cheapest, fastest):

| Level | Guarantee |
|-------|-----------|
| **Strong** | Linearizable — reads always see the latest committed write. Single-region-write only or constrained multi-region setups. |
| **Bounded Staleness** | Reads lag writes by at most K versions **or** T time. Predictable staleness window. |
| **Session** (default) | Within a single client session: read-your-writes, monotonic reads/writes, consistent prefix. The sensible default for most apps. |
| **Consistent Prefix** | Reads never see writes out of order (no gaps), but may lag. |
| **Eventual** | Cheapest, lowest latency, no ordering guarantees — eventually converges. |

> Pick the weakest level that still satisfies your correctness requirements — it directly affects RU cost and latency.

## Messaging

### Azure Service Bus

Enterprise message broker (queues + topics, sessions, dead-letter, scheduled delivery). Different from RabbitMQ in protocol (AMQP 1.0 vs RabbitMQ's AMQP 0.9.1) and feature set — Service Bus has no concept of RabbitMQ's flexible exchange types, while it adds first-class sessions, duplicate detection, and transactions.

- **Queues**: point-to-point
- **Topics/Subscriptions**: pub/sub with filter rules per subscription
- Supports transactions, sessions (FIFO per session), dead-letter queue, scheduled delivery, duplicate detection

```csharp
// Send
var sender = serviceBusClient.CreateSender("orders-queue");
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.Serialize(order)));

// Receive
var processor = serviceBusClient.CreateProcessor("orders-queue");
processor.ProcessMessageAsync += async args =>
{
    var order = JsonSerializer.Deserialize<Order>(
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
telemetry.TrackEvent("OrderCreated", new Dictionary<string, string>
{
    { "OrderId", order.Id.ToString() },
    { "Value", order.Total.ToString() }
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

### Azure Application Gateway

**Regional** Layer 7 load balancer with integrated WAF:
- Path-based and host-based routing, URL rewrite, redirects
- WAF (OWASP rule sets) at the regional edge
- SSL termination, end-to-end TLS, autoscaling
- Scoped to a single region / VNet — use for backend routing inside one region

### Azure Front Door

**Global** Layer 7 entry point with CDN + WAF:
- Global content distribution and anycast routing to the nearest PoP
- Global WAF + DDoS protection
- SSL offloading, caching, global failover between regional backends

#### Application Gateway vs Front Door

| Aspect | Application Gateway | Front Door |
|--------|---------------------|-----------|
| Scope | Regional | Global |
| Layer | L7 (HTTP/HTTPS) | L7 (HTTP/HTTPS) |
| CDN | No | Yes |
| WAF | Yes (regional) | Yes (global edge) |
| Typical use | Route inside a region (to AKS, App Service, VMs) | Global front door for multi-region apps, CDN, DDoS |

> Common pattern: **Front Door** at the edge → **Application Gateway** inside each region → backends.

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

[← Previous: AWS In Depth](02-aws-in-depth.md) | [Next: AWS EC2 and Compute →](04-aws-ec2-and-compute.md) | [Back to index](README.md)
