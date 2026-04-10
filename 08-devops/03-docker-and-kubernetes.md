# Docker and Kubernetes

## Docker

### What it is

Docker packages an application and its dependencies into a **container** — an isolated, lightweight, and reproducible environment.

```
┌──────────────────────┐
│     Container         │
│  ┌──────────────────┐│
│  │  Your application ││
│  │  + dependencies  ││
│  │  + runtime       ││
│  └──────────────────┘│
│     Linux kernel      │
└──────────────────────┘
```

### Container vs VM

| Aspect | Container | VM |
|---------|-----------|-----|
| Size | MBs (lightweight) | GBs (heavy) |
| Startup | Seconds | Minutes |
| Isolation | Process | Full OS |
| Overhead | Minimal | High |
| Use case | Microservices, CI/CD | Full isolation, legacy |

### Dockerfile (.NET)

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Runtime stage (multi-stage = smaller image)
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MinhaApp.dll"]
```

### Essential commands

```bash
# Build
docker build -t minha-app:1.0 .

# Run
docker run -d -p 8080:8080 --name app minha-app:1.0

# List containers
docker ps

# Logs
docker logs app

# Stop and remove
docker stop app && docker rm app
```

### Docker Compose

Orchestrates **multiple containers** locally:

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ConnectionStrings__Default=Server=db;Database=app;User=sa;Password=Str0ng!
    depends_on:
      - db
      - redis

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=Str0ng!
    ports:
      - "1433:1433"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

```bash
docker compose up -d    # starts everything
docker compose down     # tears everything down
docker compose logs -f  # follows logs
```

## Kubernetes (K8s)

### What it is

A **container orchestration** platform at scale. It manages deployment, scaling, networking, and resilience.

### Main concepts

| Concept | Description |
|----------|-----------|
| **Pod** | Smallest unit — 1 or more containers together |
| **Deployment** | Manages pod replicas (rolling updates, rollback) |
| **Service** | Exposes pods on the network (internal load balancer) |
| **Ingress** | Routes external traffic (HTTP) to services |
| **Namespace** | Logical isolation (dev, staging, prod) |
| **ConfigMap** | Non-sensitive configurations |
| **Secret** | Sensitive data (passwords, connection strings) |

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: minha-api
  template:
    metadata:
      labels:
        app: minha-api
    spec:
      containers:
        - name: api
          image: minha-app:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: minha-api-service
spec:
  selector:
    app: minha-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Essential commands

```bash
kubectl apply -f deployment.yaml      # applies configuration
kubectl get pods                       # lists pods
kubectl get services                   # lists services
kubectl scale deployment minha-api --replicas=5  # scales
kubectl rollout undo deployment minha-api        # rollback
kubectl logs -f pod-name               # logs
kubectl describe pod pod-name          # pod details
```

### Health Checks in .NET

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)
    .AddRedis("localhost:6379");

app.MapHealthChecks("/health");    // liveness
app.MapHealthChecks("/ready");     // readiness
```

## Managed K8s services

| Service | Provider |
|---------|----------|
| **AKS** | Azure Kubernetes Service |
| **EKS** | Amazon Elastic Kubernetes Service |
| **GKE** | Google Kubernetes Engine |

Managed = no need to manage the control plane (master nodes).

---

[← Previous: FaaS](02-faas.md) | [Next: Terraform →](04-terraform.md) | [Back to index](README.md)
