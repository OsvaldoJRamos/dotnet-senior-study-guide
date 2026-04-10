# AWS Load Balancers

## Why Load Balancers

Distribute incoming traffic across multiple targets (EC2 instances, containers, IPs) to ensure:
- **High availability** — if one instance dies, traffic goes to healthy ones
- **Scalability** — add more targets as traffic grows
- **SSL termination** — offload HTTPS to the load balancer
- **Health checks** — automatically remove unhealthy targets

## Types of Load Balancers

### ALB (Application Load Balancer) — Layer 7

Operates at HTTP/HTTPS level. The most common choice for web APIs and microservices.

```
Client → [ALB] → /api/users    → [Target Group: Users Service]
                → /api/orders   → [Target Group: Orders Service]
                → /api/payments → [Target Group: Payments Service]
```

**Features:**
- Path-based routing (`/api/users` → service A, `/api/orders` → service B)
- Host-based routing (`api.example.com` → service A, `admin.example.com` → service B)
- Weighted target groups (canary deployments: 90% v1, 10% v2)
- WebSocket support
- HTTP/2 support
- Integration with WAF (Web Application Firewall)
- gRPC support
- Authentication integration (Cognito, OIDC)

### NLB (Network Load Balancer) — Layer 4

Operates at TCP/UDP level. Ultra-low latency, millions of requests per second.

```
Client → [NLB] → TCP:443 → [Target Group: Instances]
```

**Features:**
- Static IP per Availability Zone
- Preserves source IP
- Ultra-low latency (~100µs vs ~400µs for ALB)
- Handles millions of requests per second
- TLS termination

**When to use NLB over ALB:**
- Need static IPs
- Non-HTTP protocols (TCP, UDP, TLS)
- Extreme performance requirements
- Gaming, IoT, real-time streaming

### GLB (Gateway Load Balancer) — Layer 3

For third-party virtual appliances (firewalls, intrusion detection). Rarely used directly.

## Key Concepts

### Target Groups

A group of targets that receive traffic from the load balancer:

```
[ALB]
  ├── Target Group: "api-tg" (port 8080)
  │     ├── Instance i-111 (healthy)
  │     ├── Instance i-222 (healthy)
  │     └── Instance i-333 (unhealthy → removed)
  │
  └── Target Group: "web-tg" (port 80)
        ├── Instance i-444
        └── Instance i-555
```

Target types: **instance**, **IP**, **Lambda function**, **ALB** (for chaining).

### Health Checks

The ALB periodically checks if targets are healthy:

```
ALB → GET /health → Target
    ← 200 OK     → Healthy (receives traffic)
    ← 500 Error  → Unhealthy (removed from rotation)

Settings:
  Path: /health
  Interval: 30s
  Healthy threshold: 3 consecutive successes
  Unhealthy threshold: 2 consecutive failures
  Timeout: 5s
```

In .NET:

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)
    .AddRedis("localhost:6379")
    .AddCheck("custom", () => HealthCheckResult.Healthy());

app.MapHealthChecks("/health");
```

### Listener Rules

Define how the ALB routes requests:

```
Listener (HTTPS:443)
  ├── Rule 1: IF path = /api/*     THEN forward to api-tg
  ├── Rule 2: IF host = admin.*    THEN forward to admin-tg
  ├── Rule 3: IF header X-Test     THEN forward to canary-tg
  └── Default: forward to web-tg
```

### Sticky Sessions (Session Affinity)

Routes requests from the same client to the same target:

```
Client A → [ALB] → Instance 1 (cookie set)
Client A → [ALB] → Instance 1 (same target because of cookie)
```

- Uses cookies (application-based or duration-based)
- **Avoid when possible** — breaks horizontal scaling
- Use distributed session (Redis) instead

## SSL/TLS Termination

ALB handles HTTPS, forwards HTTP to targets:

```
Client ──HTTPS──→ [ALB] ──HTTP──→ [Targets]
                    ↑
            AWS Certificate Manager (ACM)
            (free SSL certificates)
```

Benefits:
- Targets don't need to handle SSL
- Centralized certificate management
- Free certificates from ACM

## Cross-Zone Load Balancing

Distributes traffic evenly across all targets in all AZs:

```
Without cross-zone:
  AZ-A (2 instances): 50% traffic → 25% each
  AZ-B (8 instances): 50% traffic → 6.25% each  ← uneven!

With cross-zone (default for ALB):
  All 10 instances: 10% each ← even
```

## Connection Draining (Deregistration Delay)

When an instance is being removed (scaling down, deployment), the ALB:
1. Stops sending new requests to the instance
2. Waits for in-flight requests to complete (default: 300s)
3. Then removes the instance

Set a lower value (30-60s) for faster deployments.

## ALB vs NLB Summary

| Aspect | ALB | NLB |
|--------|-----|-----|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP) |
| Routing | Path, host, headers, query string | Port-based only |
| Latency | ~400µs | ~100µs |
| Static IP | No (use Global Accelerator) | Yes |
| WebSocket | Yes | Yes |
| Source IP | X-Forwarded-For header | Preserved natively |
| Price | Per hour + LCU | Per hour + NLCU |
| Best for | Web APIs, microservices | TCP, UDP, extreme performance |

## Common Architecture

```
[Route 53] → [CloudFront CDN]
                    ↓
              [ALB (public subnet)]
              ├── /api/* → [ECS Fargate: API (private subnet)]
              └── /*     → [S3 + CloudFront: Static frontend]
                              ↓
                    [RDS (private subnet)]
```

---

[← Previous: AWS EC2 and Compute](04-aws-ec2-and-compute.md) | [Next: AWS Logging and Monitoring →](06-aws-logging-and-monitoring.md) | [Back to index](README.md)
