# Cloud and Infrastructure

> Read the questions, think about your answer, then click to reveal.

---

### 1. What are the main EC2 instance types, and when would you use each purchasing option?

<details>
<summary>Reveal answer</summary>

**Instance families** (mnemonic: the first letter):
- **T** (burstable) -- dev/test, small workloads with variable CPU.
- **M** (general purpose) -- balanced CPU/memory, most web apps.
- **C** (compute optimized) -- CPU-heavy tasks (batch processing, ML inference).
- **R** (memory optimized) -- in-memory caches, large datasets.
- **G/P** (GPU) -- machine learning training, video encoding.

**Purchasing options**:

| Option | Discount | Commitment | Use case |
|--------|----------|-----------|----------|
| **On-Demand** | None | None | Unpredictable workloads, short-term |
| **Reserved (1-3yr)** | Up to 72% | Steady-state | Production baselines |
| **Savings Plans** | Up to 72% | $/hr commitment | Flexible across instance types |
| **Spot** | Up to 90% | None (can be interrupted) | Fault-tolerant batch, CI/CD, big data |

**Rule of thumb**: use Reserved/Savings Plans for your baseline, On-Demand for spikes, and Spot for interruptible work.

Deep dive: [EC2 and Compute](../12-cloud/04-aws-ec2-and-compute.md)

</details>

---

### 2. When would you use an ALB vs an NLB?

<details>
<summary>Reveal answer</summary>

| Aspect | ALB (Application) | NLB (Network) |
|--------|-------------------|---------------|
| Layer | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) |
| Routing | Path, host, header, query string | Port-based |
| Performance | Good, but adds latency | Ultra-low latency, millions of req/s |
| TLS termination | Yes (with ACM certificates) | Yes, or pass-through |
| Static IP | No (use DNS name) | Yes (Elastic IPs) |
| WebSocket | Yes | Yes |
| Health checks | HTTP/HTTPS (status codes) | TCP, HTTP, HTTPS |

**Use ALB** for web applications, APIs, microservices that need content-based routing (e.g., `/api/*` to one target group, `/web/*` to another).

**Use NLB** for extreme performance, non-HTTP protocols (gRPC, game servers, IoT), when you need a static IP, or when you need to preserve the client's source IP.

Deep dive: [AWS Load Balancers](../12-cloud/05-aws-load-balancers.md)

</details>

---

### 3. What are the S3 storage classes, and how do you choose between them?

<details>
<summary>Reveal answer</summary>

| Class | Access pattern | Cost | Retrieval |
|-------|---------------|------|-----------|
| **S3 Standard** | Frequent | Highest storage | Free |
| **S3 Intelligent-Tiering** | Unknown/changing | Auto-moves between tiers | Free |
| **S3 Standard-IA** | Infrequent (monthly) | Lower storage, per-GB retrieval | Milliseconds (immediate) |
| **S3 One Zone-IA** | Infrequent, non-critical | Cheapest IA | Milliseconds (immediate) |
| **S3 Glacier Instant** | Quarterly, immediate access | Very low | Milliseconds |
| **S3 Glacier Flexible** | 1-2 times/year | Very low | Minutes to hours |
| **S3 Glacier Deep Archive** | Rarely (compliance) | Lowest | 12-48 hours |

**Choosing**: use **S3 Lifecycle rules** to automatically transition objects between classes based on age. Example: Standard for 30 days, then Standard-IA for 90 days, then Glacier for long-term archival.

**Intelligent-Tiering** is ideal when access patterns are unpredictable -- it monitors and moves objects automatically with no retrieval fees.

Deep dive: [AWS Services](../12-cloud/01-aws-services.md)

</details>

---

### 4. What is the principle of least privilege in IAM, and how does it apply to roles vs users?

<details>
<summary>Reveal answer</summary>

**Principle of least privilege**: grant only the **minimum permissions** needed to perform a task, and no more. This limits the blast radius if credentials are compromised.

**Roles vs Users**:

| Aspect | IAM User | IAM Role |
|--------|----------|----------|
| Identity | Permanent, for people or service accounts | Temporary, assumed by entities |
| Credentials | Long-lived access keys (risky) | Temporary via STS (auto-rotated) |
| Best for | Human console access (with MFA) | EC2 instances, Lambda, cross-account |

**Best practices**:
1. **Use roles, not access keys** -- attach roles to EC2/ECS/Lambda; never embed keys in code.
2. **Use IAM policies with specific resources** -- `"Resource": "arn:aws:s3:::my-bucket/*"`, not `"*"`.
3. **Use conditions** -- restrict by IP, time, MFA status.
4. **Use AWS Organizations SCPs** -- guardrails across all accounts.
5. **Review with IAM Access Analyzer** -- find unused permissions and public resources.

Deep dive: [AWS Services](../12-cloud/01-aws-services.md)

</details>

---

### 5. What is a VPC, and what is the difference between public and private subnets? How do Security Groups differ from NACLs?

<details>
<summary>Reveal answer</summary>

A **VPC (Virtual Private Cloud)** is your isolated network in AWS. You define the IP range (CIDR block) and divide it into subnets across availability zones.

**Public vs Private subnets**:
- **Public subnet** -- has a route to an **Internet Gateway (IGW)**. Resources get public IPs and can be reached from the internet (e.g., ALB, bastion hosts).
- **Private subnet** -- no direct internet route. Resources access the internet via a **NAT Gateway** in a public subnet (outbound only). Use for app servers, databases.

**Security Groups vs NACLs**:

| Aspect | Security Group | NACL |
|--------|---------------|------|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow return traffic explicitly) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (lowest number first) |
| Default | Deny all inbound, allow all outbound | Default NACL allows all; custom NACLs default to deny all (you must add allow rules) |

**Best practice**: use Security Groups as your primary firewall (per-instance, stateful). Use NACLs as an additional layer for subnet-wide rules (e.g., blocking known malicious IPs).

Deep dive: [AWS In Depth](../12-cloud/02-aws-in-depth.md)

</details>

---

### 6. What is the difference between SQS and SNS? When would you use each?

<details>
<summary>Reveal answer</summary>

| Aspect | SQS (Simple Queue Service) | SNS (Simple Notification Service) |
|--------|---------------------------|----------------------------------|
| Pattern | **Queue** (point-to-point) | **Pub/Sub** (fan-out) |
| Consumers | One consumer processes each message | Multiple subscribers get every message |
| Persistence | Messages stored until processed (up to 14 days) | No long-term persistence, but SNS has retry policies and supports DLQs to capture undeliverable messages |
| Delivery | Pull-based (consumer polls) | Push-based (SNS pushes to subscribers) |

**Use SQS** for: decoupling services, work queues, buffering writes, ensuring each task is processed exactly once (FIFO queue).

**Use SNS** for: broadcasting events to multiple consumers, sending notifications (email, SMS, HTTP), fan-out patterns.

**Common pattern -- SNS + SQS fan-out**: SNS publishes an event, multiple SQS queues subscribe. Each queue processes the event independently. This combines fan-out (SNS) with reliable, independent processing (SQS).

Deep dive: [AWS Services](../12-cloud/01-aws-services.md)

</details>

---

### 7. What is Kinesis Data Firehose, and what are its main use cases?

<details>
<summary>Reveal answer</summary>

**Kinesis Data Firehose** is a fully managed service for **loading streaming data** into destinations like S3, Redshift, OpenSearch, or HTTP endpoints. You don't manage any infrastructure.

**Key characteristics**:
- **Near real-time** -- delivers data in batches (buffered by size or time, minimum ~60 seconds).
- **Automatic scaling** -- handles throughput without provisioning shards.
- **Transformation** -- can invoke a Lambda function to transform records in-flight (e.g., JSON to Parquet, enrich, filter).
- **Compression and encryption** -- supports GZIP, Snappy, and server-side encryption.

**Use cases**:
- Streaming application logs to S3 for analytics.
- Loading clickstream data into Redshift.
- Sending metrics to OpenSearch for real-time dashboards.
- ETL pipeline: transform and deliver IoT sensor data.

**Firehose vs Kinesis Data Streams**: Firehose is simpler (no shard management) but higher latency. Use Data Streams when you need sub-second processing or custom consumers.

Deep dive: [AWS Services](../12-cloud/01-aws-services.md)

</details>

---

### 8. What are CloudWatch Logs and X-Ray, and how do they complement each other?

<details>
<summary>Reveal answer</summary>

**CloudWatch Logs**:
- Centralized log storage and querying for AWS services and applications.
- **Log Groups** contain **Log Streams** (one per source, e.g., container instance).
- **CloudWatch Logs Insights** -- SQL-like query language for searching and aggregating logs.
- **Metric Filters** -- extract metrics from log patterns (e.g., count ERROR lines, create alarms).

**AWS X-Ray**:
- **Distributed tracing** -- tracks a request as it flows through multiple services (API Gateway -> Lambda -> DynamoDB -> SQS).
- Generates a **Service Map** showing dependencies, latencies, and error rates.
- Each request gets a **Trace ID** propagated via headers.
- Helps identify **bottlenecks** and **failing downstream services**.

**How they complement each other**: CloudWatch Logs tells you **what happened** (detailed log output). X-Ray tells you **where time was spent** and **which service failed** in a distributed call chain. Use CloudWatch to drill into the specific error once X-Ray shows you which service is the culprit.

Deep dive: [AWS Logging and Monitoring](../12-cloud/06-aws-logging-and-monitoring.md)

</details>

---

### 9. When would you choose Azure App Service vs Container Apps vs AKS?

<details>
<summary>Reveal answer</summary>

| Aspect | App Service | Container Apps | AKS |
|--------|-------------|---------------|-----|
| Abstraction | PaaS (highest) | Serverless containers | Full Kubernetes |
| Best for | Web apps, APIs (simple) | Microservices, event-driven | Complex orchestration, full control |
| Scaling | Auto-scale rules | KEDA-based (HTTP, queue, custom) | HPA, cluster autoscaler |
| Containers | Optional (code deploy) | Required | Required |
| Networking | Managed, VNet integration | Managed, VNet integration | Full Kubernetes networking |
| Cost | Per plan (always running) | Per consumption (scale to zero) | Per node (always running) |
| Learning curve | Low | Medium | High |

**Decision flow**:
1. Simple web app/API, small team, no container experience -> **App Service**.
2. Microservices, need scale-to-zero, event-driven, moderate complexity -> **Container Apps**.
3. Need full Kubernetes ecosystem (service mesh, custom operators, advanced networking) -> **AKS**.

**Key insight**: Container Apps runs on Kubernetes under the hood but hides the complexity. It's the sweet spot for most microservice architectures.

Deep dive: [Azure Services](../12-cloud/03-azure-services.md)

</details>

---

### 10. What is Managed Identity in Azure, and why is it preferred over connection strings with secrets?

<details>
<summary>Reveal answer</summary>

**Managed Identity** lets an Azure resource (App Service, VM, Container App) authenticate to other Azure services (Key Vault, SQL, Storage) **without storing credentials in code or configuration**.

**How it works**:
1. Azure assigns an identity (backed by an Azure AD service principal) to your resource.
2. Your code requests a token from the local metadata endpoint (no secrets involved).
3. Azure AD issues a token scoped to the target resource.
4. The token is used to authenticate (e.g., to Azure SQL, Blob Storage).

**Two types**:
- **System-assigned** -- tied to the resource lifecycle. Deleted when the resource is deleted.
- **User-assigned** -- independent lifecycle. Can be shared across multiple resources.

**Why it's preferred**:
- **No secrets to rotate** -- tokens are automatically managed and short-lived.
- **No secrets in config** -- eliminates connection strings with passwords in `appsettings.json` or environment variables.
- **Reduced blast radius** -- even if config is leaked, there are no credentials to exploit.

```csharp
// No connection string needed
var credential = new DefaultAzureCredential();
var client = new BlobServiceClient(new Uri("https://myaccount.blob.core.windows.net"), credential);
```

Deep dive: [Azure Services](../12-cloud/03-azure-services.md)

</details>

---

### 11. What is Terraform, and how do state, plan, and apply work together?

<details>
<summary>Reveal answer</summary>

**Terraform** is an Infrastructure as Code (IaC) tool that lets you define cloud resources in declarative `.tf` files and provision them across any cloud provider.

**Core workflow**:

1. **`terraform plan`** -- compares your `.tf` files against the **state file** and shows what will change (create, update, destroy). This is a **dry run** -- nothing is modified.
2. **`terraform apply`** -- executes the plan, making real changes to your infrastructure. Updates the state file after each resource operation.
3. **State file (`terraform.tfstate`)** -- a JSON file that maps your declared resources to real-world infrastructure IDs. It's Terraform's **source of truth** about what exists.

**State best practices**:
- **Never store state locally in production** -- use a **remote backend** (S3 + DynamoDB for locking, Terraform Cloud, Azure Blob).
- **Enable state locking** -- prevents two people from running `apply` simultaneously.
- **State contains secrets** -- treat it as sensitive (encrypt at rest, restrict access).

**Modules** are reusable packages of Terraform configuration. They promote DRY infrastructure code:

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}
```

Deep dive: [AWS In Depth](../12-cloud/02-aws-in-depth.md)

</details>

---

### 12. What is the difference between a Docker container and a virtual machine?

<details>
<summary>Reveal answer</summary>

| Aspect | Docker Container | Virtual Machine |
|--------|-----------------|-----------------|
| Isolation | Process-level (shared OS kernel) | Hardware-level (own OS kernel) |
| Startup | Seconds | Minutes |
| Size | MBs (only app + dependencies) | GBs (full OS) |
| Overhead | Minimal (no hypervisor) | Significant (hypervisor + guest OS) |
| Density | Hundreds per host | Tens per host |
| Security | Weaker isolation (shared kernel) | Stronger isolation (separate kernel) |
| Portability | "Build once, run anywhere" with Docker runtime | Requires compatible hypervisor |

**When to use containers**: microservices, CI/CD pipelines, consistent dev/prod environments, horizontal scaling.

**When to use VMs**: strong security isolation requirements (multi-tenant), running different OS kernels, legacy applications that need a full OS, compliance requirements.

**In practice**: most modern architectures run containers **inside** VMs (e.g., ECS on EC2, AKS nodes are VMs running containers). You get the security boundary of VMs with the density and speed of containers.

Deep dive: [AWS In Depth](../12-cloud/02-aws-in-depth.md)

</details>

---
