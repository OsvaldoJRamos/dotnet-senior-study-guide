# DevOps and Cloud

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between a container and a virtual machine?

<details>
<summary>Reveal answer</summary>

| Aspect | Container | Virtual Machine |
|--------|-----------|-----------------|
| Isolation | Process-level (shares host OS kernel) | Full OS with its own kernel |
| Startup | Seconds | Minutes |
| Size | Megabytes | Gigabytes |
| Overhead | Minimal | Hypervisor + full OS |
| Use case | Microservices, CI/CD | Full OS isolation, legacy apps |

Containers are **lighter and faster** but share the kernel, so they offer weaker isolation than VMs. VMs are better when you need **full OS-level security boundaries**.

Deep dive: [Docker and Kubernetes](../12-devops/03-docker-and-kubernetes.md)

</details>

---

### 2. What is a multi-stage Docker build and why should you use it?

<details>
<summary>Reveal answer</summary>

A multi-stage build uses **multiple `FROM` statements** in a single Dockerfile. Each stage can use a different base image. Only the final stage's filesystem ends up in the output image.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Benefits**: the SDK and source code stay in the build stage, so the final image is **much smaller** (hundreds of MB saved) and has a **smaller attack surface**.

Deep dive: [Docker and Kubernetes](../12-devops/03-docker-and-kubernetes.md)

</details>

---

### 3. What is Docker Compose and when would you use it?

<details>
<summary>Reveal answer</summary>

**Docker Compose** is a tool for defining and running **multi-container applications** with a single YAML file (`docker-compose.yml`). You declare services, networks, and volumes, then run `docker compose up`.

Use it for:
- **Local development** -- spin up your app + database + Redis + message broker with one command.
- **Integration testing** -- reproducible test environments.
- **Simple deployments** -- single-host setups where Kubernetes is overkill.

It is **not** a production orchestrator -- for that, use Kubernetes.

Deep dive: [Docker and Kubernetes](../12-devops/03-docker-and-kubernetes.md)

</details>

---

### 4. In Kubernetes, what is a Pod, a Deployment, a Service, and an Ingress?

<details>
<summary>Reveal answer</summary>

- **Pod** -- the smallest deployable unit; one or more containers sharing network and storage. Usually one container per pod.
- **Deployment** -- manages a set of identical pods (replicas), handles rolling updates and rollbacks.
- **Service** -- stable network endpoint (ClusterIP, NodePort, LoadBalancer) that routes traffic to pods using label selectors.
- **Ingress** -- HTTP/HTTPS routing rules (host/path-based) that sit in front of services, typically backed by an ingress controller like NGINX.

Deep dive: [Docker and Kubernetes](../12-devops/03-docker-and-kubernetes.md)

</details>

---

### 5. How do Kubernetes liveness and readiness probes differ?

<details>
<summary>Reveal answer</summary>

- **Liveness probe** -- "Is the container still alive?" If it fails, Kubernetes **restarts** the container. Use for detecting deadlocks or hung processes.
- **Readiness probe** -- "Can this container handle traffic?" If it fails, the pod is **removed from the Service's endpoints** (no traffic routed to it). Use for warm-up time or dependency availability.

A pod can be alive but not ready (e.g., waiting for a database connection). Always configure both probes for production workloads.

Deep dive: [Docker and Kubernetes](../12-devops/03-docker-and-kubernetes.md)

</details>

---

### 6. What is Terraform state, and why should you never store it locally in a team environment?

<details>
<summary>Reveal answer</summary>

**Terraform state** is a JSON file (`terraform.tfstate`) that maps your configuration to real-world resources. It tracks what exists so Terraform knows what to create, update, or destroy.

Storing it locally is dangerous because:
- **No shared visibility** -- teammates don't see each other's changes.
- **No locking** -- concurrent runs can corrupt state or create duplicate resources.
- **Risk of loss** -- deleting the file means Terraform "forgets" all resources.

Use a **remote backend** (S3 + DynamoDB, Azure Blob + lease, Terraform Cloud) for shared access and state locking.

Deep dive: [Terraform](../12-devops/04-terraform.md)

</details>

---

### 7. What is the difference between `terraform plan` and `terraform apply`?

<details>
<summary>Reveal answer</summary>

- **`terraform plan`** -- a **dry run**. It compares the desired state (code) with the current state (state file + real infrastructure) and shows what would change: resources to add, modify, or destroy. Changes nothing.
- **`terraform apply`** -- **executes** the plan, making the actual changes to infrastructure.

**Best practice**: always run `plan` first, review the output, then `apply`. In CI/CD, save the plan to a file (`-out=plan.tfplan`) and apply that exact plan to avoid drift between plan and apply.

Deep dive: [Terraform](../12-devops/04-terraform.md)

</details>

---

### 8. What are the typical stages in a CI/CD pipeline, and what are approval gates?

<details>
<summary>Reveal answer</summary>

Typical stages:
1. **Build** -- compile, restore dependencies.
2. **Test** -- unit tests, integration tests, code analysis.
3. **Publish** -- create artifacts (Docker image, NuGet package).
4. **Deploy to Staging** -- deploy to a non-production environment.
5. **Deploy to Production** -- deploy to live.

**Approval gates** are manual or automated checkpoints between stages. A human or policy must approve before the pipeline proceeds (e.g., require a QA lead's approval before production deploy). In Azure Pipelines, these are configured as **environment approvals**.

Deep dive: [Azure Pipelines](../12-devops/05-azure-pipelines.md)

</details>

---

### 9. What is the difference between ALB and NLB in AWS?

<details>
<summary>Reveal answer</summary>

| Feature | ALB (Application) | NLB (Network) |
|---------|-------------------|----------------|
| OSI Layer | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) |
| Routing | Path-based, host-based, header-based | Port-based |
| Performance | Good, with some latency | Ultra-low latency, millions of req/s |
| TLS termination | Yes | Yes (TLS passthrough also supported) |
| Use case | REST APIs, microservices | gRPC, gaming, IoT, static IP needs |

Choose **ALB** for HTTP routing features. Choose **NLB** for raw throughput and low latency.

Deep dive: [AWS Load Balancers](../13-cloud/05-aws-load-balancers.md)

</details>

---

### 10. What is the difference between SQS and SNS in AWS?

<details>
<summary>Reveal answer</summary>

- **SQS (Simple Queue Service)** -- **queue** (point-to-point). Messages are stored until a consumer polls and processes them. Supports standard and FIFO queues.
- **SNS (Simple Notification Service)** -- **topic** (pub/sub). Messages are pushed to all subscribers (SQS queues, Lambda, HTTP, email).

They are often used **together**: SNS fans out a message to multiple SQS queues, where each queue serves a different consumer. This is the **fanout pattern**.

Deep dive: [AWS - In Depth](../13-cloud/02-aws-in-depth.md)

</details>

---

### 11. What is the difference between Azure App Service, Container Apps, and AKS?

<details>
<summary>Reveal answer</summary>

| Service | Abstraction | Best for |
|---------|------------|----------|
| **App Service** | PaaS, fully managed | Simple web apps, APIs, quick deployment |
| **Container Apps** | Serverless containers, built on K8s | Microservices, event-driven, auto-scaling without K8s complexity |
| **AKS** | Managed Kubernetes | Full K8s control, complex orchestration, existing K8s expertise |

**Rule of thumb**: start with App Service. Move to Container Apps for container-based microservices. Use AKS only when you need full Kubernetes capabilities (custom operators, service mesh, etc.).

Deep dive: [Azure - Services](../13-cloud/03-azure-services.md)

</details>

---

### 12. What is a "cold start" in FaaS (serverless functions) and how can you mitigate it?

<details>
<summary>Reveal answer</summary>

A **cold start** occurs when a function hasn't been invoked recently and the platform must spin up a new instance: allocate resources, load the runtime, initialize dependencies. This adds **latency** (hundreds of ms to several seconds for .NET).

Mitigations:
- **Provisioned concurrency** (AWS Lambda) / **Always Ready instances** (Azure Functions Premium) -- keep instances warm.
- **Smaller deployment packages** -- faster loading.
- **Native AOT** (.NET 8+) -- drastically reduces startup time.
- **Keep-alive pings** -- periodic invocations to prevent de-allocation.

Deep dive: [FaaS](../12-devops/02-faas.md)

</details>

---

### 13. What are the benefits of Infrastructure as Code (IaC)?

<details>
<summary>Reveal answer</summary>

- **Repeatability** -- same code produces identical environments every time.
- **Version control** -- infrastructure changes are tracked in git with full history.
- **Code review** -- team reviews infra changes before they're applied.
- **Automation** -- integrate with CI/CD for automated provisioning and teardown.
- **Documentation** -- the code *is* the documentation of what exists.
- **Drift detection** -- compare desired state to actual state and fix discrepancies.

Tools: **Terraform** (multi-cloud), **Bicep/ARM** (Azure), **CloudFormation** (AWS), **Pulumi** (general-purpose, real programming languages).

Deep dive: [Terraform](../12-devops/04-terraform.md)

</details>

---

### 14. What is IAM in AWS and what is the principle of least privilege?

<details>
<summary>Reveal answer</summary>

**IAM (Identity and Access Management)** controls **who** can do **what** on **which** AWS resources. It uses users, groups, roles, and policies.

The **principle of least privilege** means granting only the minimum permissions needed to perform a task. For example, a Lambda function that reads from S3 should only have `s3:GetObject` on the specific bucket, not `s3:*` on `*`.

Best practices:
- Use **IAM roles** (not long-lived access keys) for services.
- Use **Managed Identity** in Azure (equivalent concept).
- Regularly audit unused permissions.

Deep dive: [AWS - In Depth](../13-cloud/02-aws-in-depth.md)

</details>

---

### 15. What is the difference between CloudWatch (AWS) and Application Insights (Azure)?

<details>
<summary>Reveal answer</summary>

Both are monitoring and observability services, but scoped differently:

- **CloudWatch** -- AWS-native. Collects metrics, logs, and alarms for AWS resources. You configure dashboards, set alarms on thresholds, and stream logs. More infrastructure-focused.
- **Application Insights** -- Azure-native (part of Azure Monitor). APM-focused: automatic request tracing, dependency tracking, exception logging, live metrics, and application map. More application-focused.

Both support custom metrics, distributed tracing, and alerting. CloudWatch is broader (infra + app), while Application Insights is deeper for application performance out of the box.

Deep dive: [AWS Logging and Monitoring](../13-cloud/06-aws-logging-and-monitoring.md) | [Azure - Services](../13-cloud/03-azure-services.md)

</details>

---

### 16. What is a VPC in AWS, and why would you use subnets?

<details>
<summary>Reveal answer</summary>

A **VPC (Virtual Private Cloud)** is your own isolated virtual network within AWS. You control IP ranges, subnets, route tables, and gateways.

**Subnets** partition the VPC:
- **Public subnets** -- have a route to an Internet Gateway; used for load balancers, bastion hosts.
- **Private subnets** -- no direct internet access; used for application servers, databases. Access the internet via a NAT Gateway if needed.

This separation enforces the principle of **defense in depth** -- databases are never directly exposed to the internet.

Deep dive: [AWS - In Depth](../13-cloud/02-aws-in-depth.md)

</details>

---

### 17. What is the difference between `git merge` and `git rebase`? When do you use each?

<details>
<summary>Reveal answer</summary>

- **`git merge`** creates a **merge commit** that joins two histories. Original commits are preserved unchanged. Non-destructive but adds "railroad" structure to the graph.
- **`git rebase`** moves your branch commits to sit **on top of** another branch. Commits are **replaced** with new SHAs, producing a **linear** history. Requires `--force-with-lease` when pushing an already-published branch.

| Use case | Choose |
|----------|--------|
| Integrating a finished feature into main | **Merge** (or squash merge) |
| Keeping a feature branch up to date with main | **Rebase** |
| Cleaning up local commits before opening a PR | **Interactive rebase** |
| Shared public branch others have pulled | **Never rebase** — use merge |

**Golden rule**: never rebase commits that other people have already pulled.

Deep dive: [Git Essentials](../12-devops/06-git-essentials.md)

</details>

---

### 18. How do you undo a commit? What's the difference between reset, revert, and amend?

<details>
<summary>Reveal answer</summary>

| Command | Effect | Safe on pushed commits? |
|---------|--------|-------------------------|
| `git commit --amend` | Rewrites the last commit (new SHA) | ❌ requires force-push |
| `git reset --soft HEAD~1` | Undo commit, keep changes **staged** | ❌ rewrites history |
| `git reset --mixed HEAD~1` | Undo commit, keep changes **unstaged** (default) | ❌ rewrites history |
| `git reset --hard HEAD~1` | Undo commit and **discard** changes | ❌ rewrites history and deletes work |
| `git revert <sha>` | Creates a **new** commit that inverts `<sha>` | ✅ safe — adds history instead of rewriting |

Rule of thumb: **`revert` on shared history, `reset`/`amend` only on local commits.** If you must update a pushed branch after an amend or rebase, use `git push --force-with-lease` to avoid overwriting others' work.

Deep dive: [Git Essentials](../12-devops/06-git-essentials.md)

</details>

---

### 19. What does interactive rebase do, and when would you use it?

<details>
<summary>Reveal answer</summary>

`git rebase -i HEAD~N` opens an editor listing the last N commits, letting you rewrite them:

- `pick` — keep as-is
- `reword` — change commit message
- `squash` / `fixup` — merge into previous commit (fixup drops the message)
- `edit` — pause the rebase to amend that commit
- `drop` — delete the commit
- Reorder lines to reorder commits

Typical use: **clean up local history before opening a PR** — squash "fix typo"/"WIP" noise into meaningful commits, reword bad messages, drop dead-end experiments. Never do this on commits others have already pulled.

Deep dive: [Git Essentials](../12-devops/06-git-essentials.md)

</details>

---

### 20. What is `git reflog` and why is it your safety net?

<details>
<summary>Reveal answer</summary>

`git reflog` records **every change `HEAD` has made locally** — including changes made by reset, rebase, amend, and branch switches. Even commits that look "deleted" after a hard reset are still in the reflog for ~90 days (default GC window).

```bash
git reflog
# e3f1a2b HEAD@{0}: reset: moving to HEAD~5
# 7a9c4d0 HEAD@{1}: commit: important work

git reset --hard 7a9c4d0   # recover the "lost" state
```

Any time you panic after a destructive Git operation, check the reflog first before assuming the work is gone.

Deep dive: [Git Essentials](../12-devops/06-git-essentials.md)

</details>

---

### 21. What is `git cherry-pick` and when would you use it?

<details>
<summary>Reveal answer</summary>

`git cherry-pick <sha>` applies a specific commit from another branch onto the current one, creating a new commit with the same content.

```bash
git cherry-pick abc1234
git cherry-pick abc1234..def5678   # a range
```

Common scenarios:
- Port a bug fix from `main` onto a release branch.
- Grab a single commit from a feature branch without merging the whole thing.

If it conflicts, resolve, `git add`, then `git cherry-pick --continue` (or `--abort` to bail out).

Deep dive: [Git Essentials](../12-devops/06-git-essentials.md)

</details>

---
