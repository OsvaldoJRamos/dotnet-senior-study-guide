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
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Essential commands

```bash
# Build
docker build -t my-app:1.0 .

# Run
docker run -d -p 8080:8080 --name app my-app:1.0

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
# (the top-level `version:` key is obsolete in the modern Compose spec — omit it)
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
  name: my-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
        - name: api
          image: my-app:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          startupProbe:
            httpGet:
              path: /healthz/live
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz/live
              port: 8080
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 8080
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: my-api-service
spec:
  selector:
    app: my-api
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
kubectl scale deployment my-api --replicas=5  # scales
kubectl rollout undo deployment my-api        # rollback
kubectl logs -f pod-name               # logs
kubectl describe pod pod-name          # pod details
```

### Health Checks in .NET

Split liveness and readiness — **never use the same endpoint for both**. If readiness (which checks the DB, Redis, etc.) is also the liveness probe, a transient DB blip will cause K8s to restart the pod in a loop and it will never recover.

```csharp
builder.Services.AddHealthChecks()
    // Liveness: only the app itself. Always healthy unless the process is wedged.
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    // Readiness: external dependencies. If these fail, stop routing traffic but DO NOT restart.
    .AddSqlServer(connectionString, tags: new[] { "ready" })
    .AddRedis("localhost:6379", tags: new[] { "ready" });

// Liveness — "am I alive?" — only checks self-liveness.
app.MapHealthChecks("/healthz/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

// Readiness — "can I serve traffic?" — checks dependencies.
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

### Startup probes (for slow-starting apps)

Some .NET apps have a long warmup (EF Core model caching, plugin scanning, JIT of large assemblies, gRPC reflection, AOT-less cold start). During that time the liveness probe may fail and K8s will restart the pod **before it ever finishes starting**.

Use a **startup probe** to guard the warmup window:

- While the startup probe is failing, liveness and readiness probes are **paused**.
- Once the startup probe succeeds, K8s switches to liveness/readiness as usual.
- Configure `failureThreshold × periodSeconds` to cover your worst-case startup (e.g., 30 × 10s = 5 min).

Without a startup probe, you'd be forced to set a very long `initialDelaySeconds` on liveness, which would also delay detection of a real wedge after the app is up.

## Managed K8s services

| Service | Provider |
|---------|----------|
| **AKS** | Azure Kubernetes Service |
| **EKS** | Amazon Elastic Kubernetes Service |
| **GKE** | Google Kubernetes Engine |

Managed = no need to manage the control plane (master nodes).

---

[← Previous: FaaS](02-faas.md) | [Next: Terraform →](04-terraform.md) | [Back to index](README.md)
