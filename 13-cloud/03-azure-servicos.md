# Azure - Servicos Principais

## Compute

### App Service

Plataforma gerenciada para hospedar **web apps, APIs e backends**. Sem gerenciar infra.

- Suporta .NET, Node.js, Python, Java, PHP
- Auto-scale, custom domains, SSL
- Deployment slots (staging → swap → production)
- **Melhor opcao para APIs .NET na maioria dos casos**

### Azure Container Apps

Containers gerenciados com **auto-scaling** (incluindo scale to zero):

- Baseado no Kubernetes (KEDA + Dapr)
- Ideal para microservices e APIs containerizadas
- Mais simples que AKS, mais poderoso que App Service

### AKS (Azure Kubernetes Service)

Kubernetes gerenciado. Controle total sobre orquestracao:

- Control plane gerenciado pela Microsoft
- Voce gerencia worker nodes
- Ideal para cenarios complexos que precisam de K8s nativo

### Azure Functions

FaaS — funcoes serverless (veja [FaaS](../08-devops/02-faas.md)):

- Triggers: HTTP, Queue, Timer, Blob, Event Grid
- Consumption plan: paga por execucao
- Premium plan: sem cold start

### Quando usar o que

| Cenario | Servico |
|---------|---------|
| API .NET simples | App Service |
| Microservices containerizados | Container Apps |
| K8s complexo, controle total | AKS |
| Evento/trigger isolado | Azure Functions |

## Storage

### Blob Storage

Equivalente ao S3. Armazena objetos (arquivos, imagens, backups):

```csharp
var blobClient = new BlobClient(connectionString, "container", "arquivo.pdf");

// Upload
await blobClient.UploadAsync(stream, overwrite: true);

// Download
var response = await blobClient.DownloadContentAsync();
var conteudo = response.Value.Content;

// SAS Token (URL temporaria)
var sasUri = blobClient.GenerateSasUri(BlobSasPermissions.Read,
    DateTimeOffset.UtcNow.AddHours(1));
```

| Tier | Uso | Custo |
|------|-----|-------|
| **Hot** | Acesso frequente | $$ |
| **Cool** | Acesso raro (30+ dias) | $ |
| **Archive** | Arquivo longo prazo | ¢ |

## Database

### Azure SQL

SQL Server gerenciado. Sem gerenciar VM, backups automaticos:

- Compativel com SQL Server on-premises
- Elastic pools para multiplos bancos
- Geo-replication para disaster recovery

### Cosmos DB

Banco NoSQL multi-modelo distribuido globalmente:

- APIs: SQL, MongoDB, Cassandra, Gremlin, Table
- Latencia < 10ms no p99
- Multi-region com consistency levels configuravel
- Caro — use apenas quando precisa de escala global

## Messaging

### Azure Service Bus

Broker de mensagens enterprise:

- **Queues**: ponto a ponto
- **Topics/Subscriptions**: pub/sub
- Suporta transacoes, sessoes, dead-letter queue
- Equivalente ao RabbitMQ gerenciado

```csharp
// Enviar
var sender = serviceBusClient.CreateSender("pedidos-queue");
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.Serialize(pedido)));

// Receber
var processor = serviceBusClient.CreateProcessor("pedidos-queue");
processor.ProcessMessageAsync += async args =>
{
    var pedido = JsonSerializer.Deserialize<Pedido>(
        args.Message.Body.ToString());
    await args.CompleteMessageAsync(args.Message);
};
```

### Azure Event Grid

Roteamento de **eventos** entre servicos Azure:

```
[Blob criado] → [Event Grid] → [Azure Function processa]
[Resource mudou] → [Event Grid] → [Logic App notifica]
```

## Identity

### Azure AD / Entra ID

Identity provider da Microsoft. Gerencia autenticacao e autorizacao:

- SSO (Single Sign-On)
- OAuth 2.0 / OpenID Connect
- Managed Identities (sem gerenciar secrets)

### Managed Identity

Identidade gerenciada para servicos Azure acessarem outros servicos **sem secrets**:

```csharp
// Em vez de connection string com senha:
var credential = new DefaultAzureCredential(); // usa managed identity
var client = new BlobServiceClient(new Uri("https://mystorage.blob.core.windows.net"), credential);
```

> **Sempre prefira Managed Identity** sobre connection strings com senha.

## Monitoring

### Application Insights

APM (Application Performance Monitoring) integrado:

- Request tracking
- Dependency tracking (SQL, HTTP, Redis)
- Exception logging
- Custom metrics e events
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

Plataforma de monitoramento unificada:
- Metricas de todos os servicos Azure
- Log Analytics (queries KQL)
- Alertas e action groups

## Networking

### Azure API Management (APIM)

API Gateway gerenciado:
- Rate limiting, throttling
- Autenticacao centralizada
- Transformacao de request/response
- Developer portal

### Azure Front Door

CDN + WAF + Load Balancer global:
- Distribuicao global de conteudo
- Protecao contra DDoS
- SSL offloading

## Arquitetura tipica no Azure

```
[Azure Front Door / CDN]
         ↓
[API Management]
         ↓
[App Service / Container Apps]     (API)
    ↓            ↓
[Azure SQL]  [Redis Cache]         (dados)
[Service Bus] → [Azure Functions]  (async)
[Blob Storage]                     (arquivos)
[Application Insights]             (monitoramento)
[Key Vault]                        (secrets)
```

---

[← Anterior: AWS Aprofundado](02-aws-aprofundado.md) | [Voltar ao índice](README.md)
