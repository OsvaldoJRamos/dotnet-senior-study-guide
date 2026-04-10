# Docker e Kubernetes

## Docker

### O que e

Docker empacota uma aplicacao e suas dependencias em um **container** — ambiente isolado, leve e reproduzivel.

```
┌──────────────────────┐
│     Container         │
│  ┌──────────────────┐│
│  │  Sua aplicacao   ││
│  │  + dependencias  ││
│  │  + runtime       ││
│  └──────────────────┘│
│     Linux kernel      │
└──────────────────────┘
```

### Container vs VM

| Aspecto | Container | VM |
|---------|-----------|-----|
| Peso | MBs (leve) | GBs (pesado) |
| Startup | Segundos | Minutos |
| Isolamento | Processo | SO completo |
| Overhead | Minimo | Alto |
| Uso | Microservices, CI/CD | Isolamento total, legacy |

### Dockerfile (.NET)

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Runtime stage (multi-stage = imagem menor)
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MinhaApp.dll"]
```

### Comandos essenciais

```bash
# Build
docker build -t minha-app:1.0 .

# Rodar
docker run -d -p 8080:8080 --name app minha-app:1.0

# Listar containers
docker ps

# Logs
docker logs app

# Parar e remover
docker stop app && docker rm app
```

### Docker Compose

Orquestra **multiplos containers** localmente:

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
docker compose up -d    # sobe tudo
docker compose down     # derruba tudo
docker compose logs -f  # acompanha logs
```

## Kubernetes (K8s)

### O que e

Plataforma de **orquestracao de containers** em escala. Gerencia deploy, escala, networking e resiliencia.

### Conceitos principais

| Conceito | Descricao |
|----------|-----------|
| **Pod** | Menor unidade — 1 ou mais containers juntos |
| **Deployment** | Gerencia replicas de pods (rolling updates, rollback) |
| **Service** | Expoe pods na rede (load balancer interno) |
| **Ingress** | Roteia trafego externo (HTTP) para services |
| **Namespace** | Isolamento logico (dev, staging, prod) |
| **ConfigMap** | Configuracoes nao-sensiveis |
| **Secret** | Dados sensiveis (senhas, connection strings) |

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

### Comandos essenciais

```bash
kubectl apply -f deployment.yaml      # aplica configuracao
kubectl get pods                       # lista pods
kubectl get services                   # lista services
kubectl scale deployment minha-api --replicas=5  # escala
kubectl rollout undo deployment minha-api        # rollback
kubectl logs -f pod-name               # logs
kubectl describe pod pod-name          # detalhes do pod
```

### Health Checks em .NET

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)
    .AddRedis("localhost:6379");

app.MapHealthChecks("/health");    // liveness
app.MapHealthChecks("/ready");     // readiness
```

## Servicos gerenciados de K8s

| Servico | Provedor |
|---------|----------|
| **AKS** | Azure Kubernetes Service |
| **EKS** | Amazon Elastic Kubernetes Service |
| **GKE** | Google Kubernetes Engine |

Gerenciados = nao precisa gerenciar o control plane (master nodes).

---

[← Anterior: FaaS](02-faas.md) | [Próximo: Terraform →](04-terraform.md) | [Voltar ao índice](README.md)
