# AWS - Main Services

## Services mentioned in the study

| Service | Category | Description |
|---------|----------|-------------|
| **S3** | Storage | Object storage (files, images, backups) |
| **EC2** | Compute | Virtual servers (VMs) |
| **ECS** | Containers | Docker container orchestration |
| **Fargate** | Containers | **Serverless** ECS (no server management) |
| **Lambda** | Serverless | FaaS - event-driven functions |
| **RDS** | Database | Managed relational databases (SQL Server, PostgreSQL, MySQL) |
| **DynamoDB** | Database | NoSQL database (key-value + document) |
| **ALB** | Networking | Application Load Balancer - distributes HTTP traffic |

## Azure Equivalents

| AWS | Azure | Description |
|-----|-------|-------------|
| S3 | Blob Storage | Object storage |
| EC2 | Virtual Machines | VMs |
| ECS/Fargate | Container Apps | Managed containers |
| Lambda | Azure Functions | Serverless functions |
| RDS | Azure SQL / Cosmos DB | Managed databases |
| DynamoDB | Cosmos DB | Managed NoSQL |
| ALB | Application Gateway | L7 load balancer |
| SQS | Azure Service Bus | Message queue |

## When to use each service

- **S3**: static files, backups, data lake
- **EC2**: when you need full control over the VM
- **ECS/Fargate**: containerized applications (prefer Fargate for less management overhead)
- **Lambda**: event processing, cron jobs, lightweight APIs
- **RDS**: relational database without managing infrastructure
- **DynamoDB**: high scale, key-based access, low latency
- **ALB**: distribute traffic across multiple instances

---

[Next: AWS In Depth →](02-aws-in-depth.md) | [Back to index](README.md)
