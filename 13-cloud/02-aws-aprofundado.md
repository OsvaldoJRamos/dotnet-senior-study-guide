# AWS - Conceitos Aprofundados

## IAM (Identity and Access Management)

Controle de **quem pode fazer o que** na AWS.

### Conceitos

| Conceito | Descricao |
|----------|-----------|
| **User** | Pessoa ou aplicacao com credenciais |
| **Group** | Conjunto de users com mesmas permissoes |
| **Role** | Identidade temporaria assumida por servicos (EC2, Lambda) |
| **Policy** | Documento JSON que define permissoes |

### Principio do menor privilegio

Sempre conceder **apenas** as permissoes necessarias:

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

### Roles para servicos

Em vez de colocar access keys no codigo, atribua uma **role** ao servico:

```
EC2 Instance ──assume──→ Role "api-role" ──permite──→ S3, SQS, DynamoDB
```

> **Nunca** coloque access keys no codigo ou em variaveis de ambiente em producao. Use IAM Roles.

## VPC (Virtual Private Cloud)

Rede virtual isolada na AWS:

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

### Conceitos

| Conceito | Descricao |
|----------|-----------|
| **Subnet publica** | Tem acesso direto a internet (via Internet Gateway) |
| **Subnet privada** | Sem acesso direto — usa NAT Gateway para sair |
| **Security Group** | Firewall stateful no nivel da instancia |
| **NACL** | Firewall stateless no nivel da subnet |
| **Internet Gateway** | Porta de entrada/saida para internet |
| **NAT Gateway** | Permite subnet privada acessar internet (saida only) |

### Regra pratica

- **ALB** na subnet publica (recebe trafego externo)
- **Aplicacao** na subnet privada (nao exposta diretamente)
- **Banco de dados** na subnet privada (isolado)

## S3 (Simple Storage Service)

### Classes de armazenamento

| Classe | Uso | Custo |
|--------|-----|-------|
| **S3 Standard** | Acesso frequente | $$ |
| **S3 Infrequent Access** | Acesso raro, mas rapido | $ |
| **S3 Glacier** | Arquivo, recuperacao em horas | ¢ |
| **S3 Glacier Deep Archive** | Arquivo longo prazo | ¢¢ |

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

URLs temporarias para upload/download sem expor o bucket:

```csharp
var request = new GetPreSignedUrlRequest
{
    BucketName = "meu-bucket",
    Key = "relatorio.pdf",
    Expires = DateTime.UtcNow.AddMinutes(15)
};
string url = s3Client.GetPreSignedURL(request);
// URL valida por 15 minutos
```

## SQS (Simple Queue Service)

Fila de mensagens gerenciada:

```csharp
// Enviar
var sendRequest = new SendMessageRequest
{
    QueueUrl = "https://sqs.us-east-1.amazonaws.com/123/minha-fila",
    MessageBody = JsonSerializer.Serialize(pedido)
};
await sqsClient.SendMessageAsync(sendRequest);

// Receber
var receiveRequest = new ReceiveMessageRequest
{
    QueueUrl = queueUrl,
    MaxNumberOfMessages = 10,
    WaitTimeSeconds = 20  // long polling
};
var response = await sqsClient.ReceiveMessageAsync(receiveRequest);
```

### SQS Standard vs FIFO

| Aspecto | Standard | FIFO |
|---------|----------|------|
| Ordem | Melhor esforco | Garantida |
| Duplicatas | Pode haver | Exactly-once |
| Throughput | Ilimitado | 300 msgs/s (3000 com batching) |
| Uso | Maioria dos casos | Quando ordem importa |

## SNS (Simple Notification Service)

Pub/Sub gerenciado — uma mensagem para **multiplos destinos**:

```
[Publisher] → [SNS Topic] → [SQS Queue 1] (processamento)
                           → [SQS Queue 2] (analytics)
                           → [Lambda] (notificacao)
                           → [Email]
```

## CloudWatch

Monitoramento e observabilidade:

- **Metrics**: CPU, memoria, latencia, erros
- **Logs**: logs centralizados de todos os servicos
- **Alarms**: notificacao quando metrica ultrapassa threshold
- **Dashboards**: visualizacao em tempo real

```
CloudWatch Alarm (CPU > 80%) → SNS Topic → Email/Slack/PagerDuty
```

## Arquitetura tipica na AWS

```
[Route 53 (DNS)]
       ↓
[CloudFront (CDN)]
       ↓
[ALB (Load Balancer)]
    ↓          ↓
[ECS/Fargate] [ECS/Fargate]   (subnet privada)
    ↓
[RDS] [ElastiCache/Redis]      (subnet privada)
[SQS] → [Lambda]               (processamento async)
[S3]                            (arquivos)
[CloudWatch]                    (monitoramento)
```

---

[← Anterior: AWS - Serviços Principais](01-aws-servicos.md) | [Próximo: Azure - Serviços →](03-azure-servicos.md) | [Voltar ao índice](README.md)
