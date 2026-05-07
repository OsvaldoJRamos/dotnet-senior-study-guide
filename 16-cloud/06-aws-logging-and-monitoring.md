# AWS Logging and Monitoring

## CloudWatch Logs

Centralized log management for all AWS services and applications.

### Concepts

```
[Log Group]  →  /ecs/my-api          (one per application/service)
  └── [Log Stream]  →  ecs/api/abc123  (one per container/instance)
       └── [Log Events]  →  individual log lines with timestamps
```

| Concept | Description |
|---------|-------------|
| **Log Group** | Container for log streams (e.g., one per service) |
| **Log Stream** | Sequence of events from a single source |
| **Log Event** | A single log entry (timestamp + message) |
| **Retention** | How long logs are kept (1 day to 10 years, or never expire) |

### Sending logs from .NET

```csharp
// Using Serilog + AWS CloudWatch sink
builder.Host.UseSerilog((context, config) =>
{
    config
        .WriteTo.Console()
        .WriteTo.AmazonCloudWatch(new CloudWatchSinkOptions
        {
            LogGroupName = "/app/my-api",
            CreateLogGroup = true,
            TextFormatter = new JsonFormatter(),
            MinimumLogEventLevel = LogEventLevel.Information
        }, new AmazonCloudWatchLogsClient());
});
```

### Structured logging (JSON)

Always log in JSON format — makes it queryable:

```json
{
  "timestamp": "2026-04-09T14:30:00Z",
  "level": "Information",
  "message": "Order created",
  "orderId": "ORD-12345",
  "customerId": "CUST-678",
  "total": 150.00,
  "traceId": "abc-123-def"
}
```

### CloudWatch Log Insights

SQL-like query language for searching logs:

```
# Find errors in the last hour
fields @timestamp, @message
| filter @message like /Exception/
| sort @timestamp desc
| limit 50

# Count errors per minute
fields @timestamp
| filter level = "Error"
| stats count() as errorCount by bin(5m)

# Find slow requests
fields @timestamp, duration, path
| filter duration > 1000
| sort duration desc
| limit 20

# Group errors by type
fields @timestamp, exceptionType
| filter level = "Error"
| stats count() by exceptionType
| sort count desc
```

### Log subscriptions

Stream logs to other services in real time:

```
[CloudWatch Logs] → [Subscription Filter] → [Kinesis Firehose] → [S3 / Elasticsearch]
                                           → [Lambda] → (custom processing)
```

## CloudWatch Metrics and Alarms

### Default metrics (automatic)

EC2: CPU, network, disk. ECS: CPU, memory. ALB: request count, latency, 5xx count. RDS: connections, read/write IOPS.

### Custom metrics

```csharp
var cloudWatch = new AmazonCloudWatchClient();

await cloudWatch.PutMetricDataAsync(new PutMetricDataRequest
{
    Namespace = "MyApp",
    MetricData = new List<MetricDatum>
    {
        new MetricDatum
        {
            MetricName = "OrdersProcessed",
            Value = 1,
            Unit = StandardUnit.Count,
            Dimensions = new List<Dimension>
            {
                new Dimension { Name = "Environment", Value = "Production" }
            }
        }
    }
});
```

### Alarms

```
CloudWatch Alarm:
  Metric: ALB 5xx error rate
  Threshold: > 5% for 3 consecutive periods (1 min each)
  Action: Send to SNS topic → Email + Slack + PagerDuty
```

## CloudTrail

Records **all API calls** made in your AWS account. Essential for security auditing.

```
Who did what, when, from where:

{
  "eventTime": "2026-04-09T14:30:00Z",
  "userIdentity": { "arn": "arn:aws:iam::123:user/john" },
  "eventName": "DeleteBucket",
  "sourceIPAddress": "203.0.113.50",
  "requestParameters": { "bucketName": "important-data" }
}
```

Use cases:
- **Security investigation**: who deleted that S3 bucket?
- **Compliance**: prove that only authorized users accessed sensitive resources
- **Change tracking**: when was that security group modified?

## X-Ray (Distributed Tracing)

Traces requests across multiple services. Essential for debugging microservices.

```
Client → [ALB] → [API Service] → [Order Service] → [RDS]
                        ↓
                  [Payment Service] → [External API]

X-Ray trace:
├── ALB (5ms)
├── API Service (120ms)
│   ├── Order Service (80ms)
│   │   └── RDS Query (45ms)  ← slow!
│   └── Payment Service (200ms)
│       └── External API (180ms)  ← bottleneck!
└── Total: 325ms
```

### X-Ray in .NET

```csharp
// Add X-Ray SDK
builder.Services.AddXRay();
app.UseXRay("my-api");

// Automatic tracing for:
// - Incoming HTTP requests
// - Outgoing HTTP calls (HttpClient)
// - SQL queries (EF Core, ADO.NET)
// - AWS SDK calls
```

## Kinesis Data Firehose

Fully managed service to **load streaming data** into data stores. Zero administration.

```
[Data Sources] → [Kinesis Firehose] → [Destinations]
                       ↓
                 Optional: transform,
                 compress, encrypt
```

### Common patterns

```
# Logs to S3 (archival)
[CloudWatch Logs] → [Firehose] → [S3 bucket]
                                    └── partitioned by date: /year/month/day/

# Logs to Elasticsearch/OpenSearch (search & dashboards)
[Application] → [Firehose] → [OpenSearch] → [Kibana dashboards]

# Clickstream analytics
[Web/Mobile] → [Kinesis Data Streams] → [Firehose] → [S3] → [Athena queries]
```

### Firehose features

| Feature | Description |
|---------|-------------|
| **Buffering** | Batches data (1-15 min or 1-128 MB) before delivery |
| **Compression** | GZIP, Snappy, ZIP |
| **Encryption** | SSE with KMS |
| **Transformation** | Lambda function to transform records |
| **Format conversion** | JSON → Parquet/ORC (for analytics) |
| **Error handling** | Failed records go to a separate S3 prefix |

### Destinations

| Destination | Use case |
|-------------|----------|
| **S3** | Log archival, data lake, compliance |
| **OpenSearch** | Log search, dashboards, alerting |
| **Redshift** | Data warehouse analytics |
| **Splunk** | Enterprise log management |
| **HTTP endpoint** | Custom destinations |

## Kinesis Data Streams vs Firehose

| Aspect | Data Streams | Firehose |
|--------|-------------|----------|
| Management | You manage shards | Fully managed |
| Latency | Real-time (~200ms) | Near real-time (1-15 min buffer) |
| Consumers | Multiple, custom | Predefined destinations |
| Data retention | 1-365 days | No retention (delivery only) |
| Use case | Real-time processing, multiple consumers | Loading data into stores |
| Pricing | Per shard hour | Per GB processed |

## Observability Stack

### Option 1: AWS Native

```
Logs:    CloudWatch Logs + Log Insights
Metrics: CloudWatch Metrics + Alarms
Traces:  X-Ray
Dashboards: CloudWatch Dashboards
```

### Option 2: Third-party (common in enterprise)

```
Logs:    Datadog / Splunk / ELK (Elasticsearch + Logstash + Kibana)
Metrics: Datadog / Prometheus + Grafana
Traces:  Datadog APM / Jaeger / Zipkin
Alerts:  PagerDuty / OpsGenie
```

### Option 3: Hybrid

```
Logs:    CloudWatch Logs → Firehose → OpenSearch → Kibana
Metrics: CloudWatch Metrics + custom Prometheus
Traces:  X-Ray (AWS) + Application Insights (Azure) depending on cloud
Alerts:  CloudWatch Alarms → SNS → PagerDuty
```

## Best Practices

1. **Structured logging (JSON)** — always. Makes logs queryable and parseable
2. **Correlation IDs** — include a `traceId` in every log entry across services
3. **Log levels** — use appropriately: Debug/Info in dev, Warning/Error in prod
4. **Retention policies** — don't keep logs forever. 30 days for most, 1 year for compliance
5. **Alarms on business metrics** — not just CPU/memory. Alert on error rates, order failures, payment timeouts
6. **Dashboards** — create dashboards for each service: latency, error rate, throughput
7. **Cost** — CloudWatch Logs ingestion can be expensive at scale. Consider Firehose → S3 for archival

---

[← Previous: AWS Load Balancers](05-aws-load-balancers.md) | [Back to index](README.md)
