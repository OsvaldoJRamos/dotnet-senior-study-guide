# AWS - Servicos Principais

## Servicos mencionados no estudo

| Servico | Categoria | Descricao |
|---------|-----------|-----------|
| **S3** | Storage | Armazenamento de objetos (arquivos, imagens, backups) |
| **EC2** | Compute | Servidores virtuais (VMs) |
| **ECS** | Containers | Orquestracao de containers Docker |
| **Fargate** | Containers | ECS **serverless** (sem gerenciar servidores) |
| **Lambda** | Serverless | FaaS - funcoes executadas por eventos |
| **RDS** | Database | Bancos relacionais gerenciados (SQL Server, PostgreSQL, MySQL) |
| **DynamoDB** | Database | Banco NoSQL (key-value + document) |
| **ALB** | Networking | Application Load Balancer - distribui trafego HTTP |

## Equivalentes no Azure

| AWS | Azure | Descricao |
|-----|-------|-----------|
| S3 | Blob Storage | Storage de objetos |
| EC2 | Virtual Machines | VMs |
| ECS/Fargate | Container Apps | Containers gerenciados |
| Lambda | Azure Functions | Serverless functions |
| RDS | Azure SQL / Cosmos DB | Bancos gerenciados |
| DynamoDB | Cosmos DB | NoSQL gerenciado |
| ALB | Application Gateway | Load balancer L7 |
| SQS | Azure Service Bus | Fila de mensagens |

## Quando usar cada servico

- **S3**: arquivos estaticos, backups, data lake
- **EC2**: quando precisa de controle total da VM
- **ECS/Fargate**: aplicacoes containerizadas (preferir Fargate para menos gerenciamento)
- **Lambda**: processamento de eventos, cron jobs, APIs leves
- **RDS**: banco relacional sem gerenciar infra
- **DynamoDB**: alta escala, acesso por chave, baixa latencia
- **ALB**: distribuir trafego entre multiplas instancias

---

[Próximo: AWS Aprofundado →](02-aws-aprofundado.md) | [Voltar ao índice](README.md)
