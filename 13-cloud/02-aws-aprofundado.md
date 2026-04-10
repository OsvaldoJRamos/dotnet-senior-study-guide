# AWS - In-Depth Concepts

## IAM (Identity and Access Management)

Controls **who can do what** in AWS.

### Concepts

| Concept | Description |
|---------|-------------|
| **User** | Person or application with credentials |
| **Group** | Set of users with the same permissions |
| **Role** | Temporary identity assumed by services (EC2, Lambda) |
| **Policy** | JSON document that defines permissions |

### Principle of Least Privilege

Always grant **only** the necessary permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::meu-bucket/*"
    }
  ]
}
```

### Roles for Services

Instead of putting access keys in code, assign a **role** to the service:

```
EC2 Instance ──assumes──→ Role "api-role" ──allows──→ S3, SQS, DynamoDB
```

> **Never** put access keys in code or in environment variables in production. Use IAM Roles.

## VPC (Virtual Private Cloud)

Isolated virtual network in AWS:

```
┌─────────────────────── VPC (10.0.0.0/16) ───────────────────────┐
│                                                                   │
│  ┌──── Public Subnet (10.0.1.0/24) ────┐  ┌── Public Subnet ──┐│
│  │  ALB, NAT Gateway, Bastion Host      │  │  (10.0.2.0/24)    ││
│  └──────────────────────────────────────┘  └────────────────────┘│
│                                                                   │
│  ┌──── Private Subnet (10.0.3.0/24) ───┐  ┌── Private Subnet ─┐│
│  │  EC2 (API), ECS Tasks, Lambda        │  │  (10.0.4.0/24)    ││
│  └──────────────────────────────────────┘  └────────────────────┘│
│                                                                   │
│  ┌──── Private Subnet (10.0.5.0/24) ───┐                        │
│  │  RDS, ElastiCache                     │                        │
│  └──────────────────────────────────────┘                        │
└──────────────────────────────────────────────────────────────────┘
```

### Concepts

| Concept | Description |
|---------|-------------|
| **Public Subnet** | Has direct internet access (via Internet Gateway) |
| **Private Subnet** | No direct access — uses NAT Gateway to reach the internet |
| **Security Group** | Stateful firewall at the instance level |
| **NACL** | Stateless firewall at the subnet level |
| **Internet Gateway** | Entry/exit point for internet traffic |
| **NAT Gateway** | Allows private subnet to access the internet (outbound only) |

### Practical Rule

- **ALB** in the public subnet (receives external traffic)
- **Application** in the private subnet (not directly exposed)
- **Database** in the private subnet (isolated)

## S3 (Simple Storage Service)

### Storage Classes

| Class | Use | Cost |
|-------|-----|------|
| **S3 Standard** | Frequent access | $$ |
| **S3 Infrequent Access** | Rare access, but fast retrieval | $ |
| **S3 Glacier** | Archive, retrieval in hours | ¢ |
| **S3 Glacier Deep Archive** | Long-term archive | ¢¢ |

### Bucket Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456789:role/api-role" },
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::meu-bucket/*"
    }
  ]
}
```

### Pre-signed URLs

Temporary URLs for upload/download without exposing the bucket:

```csharp
var request = new GetPreSignedUrlRequest
{
    BucketName = "meu-bucket",
    Key = "relatorio.pdf",
    Expires = DateTime.UtcNow.AddMinutes(15)
};
string url = s3Client.GetPreSignedURL(request);
// URL valid for 15 minutes
```

## SQS (Simple Queue Service)

Managed message queue:

```csharp
// Send
var sendRequest = new SendMessageRequest
{
    QueueUrl = "https://sqs.us-east-1.amazonaws.com/123/minha-fila",
    MessageBody = JsonSerializer.Serialize(pedido)
};
await sqsClient.SendMessageAsync(sendRequest);

// Receive
var receiveRequest = new ReceiveMessageRequest
{
    QueueUrl = queueUrl,
    MaxNumberOfMessages = 10,
    WaitTimeSeconds = 20  // long polling
};
var response = await sqsClient.ReceiveMessageAsync(receiveRequest);
```

### SQS Standard vs FIFO

| Aspect | Standard | FIFO |
|--------|----------|------|
| Order | Best effort | Guaranteed |
| Duplicates | May occur | Exactly-once |
| Throughput | Unlimited | 300 msgs/s (3000 with batching) |
| Use case | Most cases | When order matters |

## SNS (Simple Notification Service)

Managed Pub/Sub — one message to **multiple destinations**:

```
[Publisher] → [SNS Topic] → [SQS Queue 1] (processing)
                           → [SQS Queue 2] (analytics)
                           → [Lambda] (notification)
                           → [Email]
```

## CloudWatch

Monitoring and observability:

- **Metrics**: CPU, memory, latency, errors
- **Logs**: centralized logs from all services
- **Alarms**: notification when a metric exceeds a threshold
- **Dashboards**: real-time visualization

```
CloudWatch Alarm (CPU > 80%) → SNS Topic → Email/Slack/PagerDuty
```

## Typical AWS Architecture

```
[Route 53 (DNS)]
       ↓
[CloudFront (CDN)]
       ↓
[ALB (Load Balancer)]
    ↓          ↓
[ECS/Fargate] [ECS/Fargate]   (private subnet)
    ↓
[RDS] [ElastiCache/Redis]      (private subnet)
[SQS] → [Lambda]               (async processing)
[S3]                            (files)
[CloudWatch]                    (monitoring)
```

---

[← Previous: AWS - Main Services](01-aws-servicos.md) | [Next: Azure - Services →](03-azure-servicos.md) | [Back to index](README.md)
