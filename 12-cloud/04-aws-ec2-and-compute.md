# AWS EC2 and Compute

## EC2 (Elastic Compute Cloud)

Virtual servers (instances) in the cloud. Full control over the operating system, networking, and storage.

### Instance Lifecycle

```
Launch → Running → Stop → Stopped → Start → Running → Terminate (deleted)
                     ↓
                  Hibernate (RAM saved to EBS)
```

- **Running**: you pay for compute + storage
- **Stopped**: you pay only for storage (EBS volumes)
- **Terminated**: instance is deleted. The **root** EBS volume defaults to `DeleteOnTermination = true`, so it's deleted with the instance. **Non-root attached volumes default to `DeleteOnTermination = false`** — they survive termination unless you explicitly configure them to delete. Check the block device mapping if you care.

### Instance Types

| Family | Use case | Example |
|--------|----------|---------|
| **t3/t4g** | General purpose, burstable (web servers, dev) | t3.medium, t4g.small |
| **m6i/m7g** | General purpose, steady (app servers, backends) | m6i.xlarge |
| **c6i/c7g** | Compute optimized (batch processing, ML inference) | c6i.2xlarge |
| **r6i/r7g** | Memory optimized (in-memory caches, databases) | r6i.large |
| **i3/i4i** | Storage optimized (data warehousing, Elasticsearch) | i3.xlarge |
| **g5/p5** | GPU (ML training, video encoding) | g5.xlarge |

> **Graviton (g suffix)**: ARM-based processors by AWS. Up to 40% better price-performance than x86. Use `*g*` instance types (t4g, m7g, c7g).

### Purchasing Options

| Option | Description | Savings |
|--------|-------------|---------|
| **On-Demand** | Pay per hour/second, no commitment | 0% (baseline) |
| **Reserved** | 1 or 3-year commitment | 30-72% |
| **Savings Plans** | Flexible commitment ($/hour) | 30-72% |
| **Spot Instances** | Unused capacity, can be interrupted | Up to 90% |
| **Dedicated Hosts** | Physical server for you only (compliance) | Varies |

> **Spot Instances** are great for batch processing, CI/CD runners, and stateless workloads that can handle interruption.

### AMI (Amazon Machine Image)

A template that contains the OS, software, and configuration for launching instances.

```
Custom AMI workflow:
1. Launch base instance (Amazon Linux, Ubuntu, Windows)
2. Install your app, dependencies, configuration
3. Create AMI from the instance
4. Launch new instances from the AMI (pre-configured)
```

### User Data (bootstrap script)

Script that runs on first boot:

```bash
#!/bin/bash
yum update -y
yum install -y docker
systemctl start docker
docker pull myapp:latest
docker run -d -p 80:8080 myapp:latest
```

## Auto Scaling

Automatically adjusts the number of instances based on demand.

```
[Auto Scaling Group]
  ├── Min: 2 instances (always running)
  ├── Desired: 4 instances (normal traffic)
  └── Max: 10 instances (peak traffic)

[Scaling Policies]
  ├── Scale out: CPU > 70% for 5 min → add 2 instances
  └── Scale in:  CPU < 30% for 10 min → remove 1 instance
```

### Launch Template

Defines how new instances are created (AMI, instance type, security group, user data):

```
Launch Template:
  AMI: ami-0123456789
  Instance Type: t3.medium
  Security Group: sg-webapp
  Key Pair: my-key
  User Data: bootstrap.sh
  IAM Role: ec2-app-role
```

### Target Tracking Scaling

```
"Keep average CPU at 60%"
→ ASG automatically adds/removes instances to maintain target
```

## ECS and Fargate (deeper)

### ECS (Elastic Container Service)

Orchestrates Docker containers on AWS:

```
[ECS Cluster]
  └── [Service] (desired count: 3)
       ├── [Task] → Container A + Container B (sidecar)
       ├── [Task] → Container A + Container B
       └── [Task] → Container A + Container B
```

| Concept | Description |
|---------|-------------|
| **Cluster** | Logical grouping of tasks/services |
| **Task Definition** | Blueprint (image, CPU, memory, ports, env vars) |
| **Task** | Running instance of a task definition (1+ containers) |
| **Service** | Maintains desired count of tasks (auto-restart, rolling updates) |

### ECS Launch Types

| Launch Type | You manage | Best for |
|-------------|-----------|----------|
| **Fargate** | Nothing (serverless) | Most workloads |
| **EC2** | The instances | GPU, custom networking, cost optimization |

### Task Definition example

```json
{
  "family": "my-api",
  "cpu": "256",
  "memory": "512",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:latest",
      "portMappings": [{ "containerPort": 8080 }],
      "environment": [
        { "name": "ASPNETCORE_ENVIRONMENT", "value": "Production" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### ECR (Elastic Container Registry)

Private Docker registry on AWS:

```bash
# Login
aws ecr get-login-password | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t my-api .
docker tag my-api:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:latest
```

## When to use what

| Scenario | Service |
|----------|---------|
| Full control, custom OS, legacy apps | EC2 |
| Containerized apps, simple management | ECS + Fargate |
| K8s-native workloads | EKS |
| Event-driven functions | Lambda |
| Batch processing, interruptible | EC2 Spot or Fargate Spot |

---

[← Previous: Azure Services](03-azure-services.md) | [Next: AWS Load Balancers →](05-aws-load-balancers.md) | [Back to index](README.md)
