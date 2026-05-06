# DevOps

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

Containers are **lighter and faster** but share the kernel, so they offer weaker isolation than VMs. VMs are better when you need **full OS-level security boundaries**. In practice, modern clouds run containers **inside** VMs (e.g., AKS/EKS nodes are VMs hosting containers) — you get the container density with a VM security boundary.

Deep dive: [Docker and Kubernetes](../15-devops/03-docker-and-kubernetes.md)

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

Deep dive: [Docker and Kubernetes](../15-devops/03-docker-and-kubernetes.md)

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

Deep dive: [Docker and Kubernetes](../15-devops/03-docker-and-kubernetes.md)

</details>

---

### 4. In Kubernetes, what is a Pod, a Deployment, a Service, and an Ingress?

<details>
<summary>Reveal answer</summary>

- **Pod** -- the smallest deployable unit; one or more containers sharing network and storage. Usually one container per pod.
- **Deployment** -- manages a set of identical pods (replicas), handles rolling updates and rollbacks.
- **Service** -- stable network endpoint (ClusterIP, NodePort, LoadBalancer) that routes traffic to pods using label selectors.
- **Ingress** -- HTTP/HTTPS routing rules (host/path-based) that sit in front of services, typically backed by an ingress controller like NGINX.

Deep dive: [Docker and Kubernetes](../15-devops/03-docker-and-kubernetes.md)

</details>

---

### 5. How do Kubernetes liveness, readiness, and startup probes differ?

<details>
<summary>Reveal answer</summary>

- **Liveness probe** — "Is the container still alive?" If it fails, Kubernetes **restarts** the container. Use for detecting deadlocks or hung processes.
- **Readiness probe** — "Can this container handle traffic?" If it fails, the pod is **removed from the Service's endpoints** (no traffic routed to it). Use for warm-up time or dependency availability.
- **Startup probe** — "Has the app finished starting?" Runs **before** liveness/readiness take over. Use for slow-starting apps so a long boot doesn't trigger restart loops.

A pod can be alive but not ready (e.g., waiting for a database connection). Always configure liveness + readiness for production; add startup for apps that take more than a few seconds to initialize.

Deep dive: [Docker and Kubernetes](../15-devops/03-docker-and-kubernetes.md)

</details>

---

### 6. What is the difference between ConfigMap and Secret in Kubernetes?

<details>
<summary>Reveal answer</summary>

Both inject configuration into pods, but they target different data:

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| Data type | Non-sensitive config (feature flags, log levels, URLs) | Sensitive data (passwords, tokens, certs) |
| Storage | Plain text in etcd | Base64-encoded in etcd (encryption at rest requires explicit config) |
| Access control | Regular RBAC | Regular RBAC (usually tighter) |
| Consumption | Env vars, volumes | Env vars, volumes |

Important: by default, a Kubernetes `Secret` is **not encrypted** — it's only base64-encoded. Enable **encryption at rest** via `EncryptionConfiguration` on the API server, or use an external secret manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) via CSI drivers or operators.

Deep dive: [Docker and Kubernetes](../15-devops/03-docker-and-kubernetes.md)

</details>

---

### 7. What is Terraform state, and why should you never store it locally in a team environment?

<details>
<summary>Reveal answer</summary>

**Terraform state** is a JSON file (`terraform.tfstate`) that maps your configuration to real-world resources. It tracks what exists so Terraform knows what to create, update, or destroy.

Storing it locally is dangerous because:
- **No shared visibility** -- teammates don't see each other's changes.
- **No locking** -- concurrent runs can corrupt state or create duplicate resources.
- **Risk of loss** -- deleting the file means Terraform "forgets" all resources.

Use a **remote backend** (S3 + DynamoDB, Azure Blob + lease, Terraform Cloud) for shared access and state locking.

Deep dive: [Terraform](../15-devops/04-terraform.md)

</details>

---

### 8. What is the difference between `terraform plan` and `terraform apply`?

<details>
<summary>Reveal answer</summary>

- **`terraform plan`** -- a **dry run**. It compares the desired state (code) with the current state (state file + real infrastructure) and shows what would change: resources to add, modify, or destroy. Changes nothing.
- **`terraform apply`** -- **executes** the plan, making the actual changes to infrastructure.

**Best practice**: always run `plan` first, review the output, then `apply`. In CI/CD, save the plan to a file (`-out=plan.tfplan`) and apply that exact plan to avoid drift between plan and apply.

Deep dive: [Terraform](../15-devops/04-terraform.md)

</details>

---

### 9. What are the benefits of Infrastructure as Code (IaC)?

<details>
<summary>Reveal answer</summary>

- **Repeatability** -- same code produces identical environments every time.
- **Version control** -- infrastructure changes are tracked in git with full history.
- **Code review** -- team reviews infra changes before they're applied.
- **Automation** -- integrate with CI/CD for automated provisioning and teardown.
- **Documentation** -- the code *is* the documentation of what exists.
- **Drift detection** -- compare desired state to actual state and fix discrepancies.

Tools: **Terraform** (multi-cloud), **Bicep/ARM** (Azure), **CloudFormation** (AWS), **Pulumi** (general-purpose, real programming languages).

Deep dive: [Terraform](../15-devops/04-terraform.md)

</details>

---

### 10. What are the typical stages in a CI/CD pipeline, and what are approval gates?

<details>
<summary>Reveal answer</summary>

Typical stages:
1. **Build** -- compile, restore dependencies.
2. **Test** -- unit tests, integration tests, code analysis.
3. **Publish** -- create artifacts (Docker image, NuGet package).
4. **Deploy to Staging** -- deploy to a non-production environment.
5. **Deploy to Production** -- deploy to live.

**Approval gates** are manual or automated checkpoints between stages. A human or policy must approve before the pipeline proceeds (e.g., require a QA lead's approval before production deploy). In Azure Pipelines, these are configured as **environment approvals**.

Deep dive: [Azure Pipelines](../15-devops/05-azure-pipelines.md)

</details>

---

### 11. What's the difference between blue/green and canary deployments?

<details>
<summary>Reveal answer</summary>

Both minimize downtime and risk, but differently:

- **Blue/Green** — run two identical environments. "Blue" serves all traffic; "Green" has the new version, fully tested. Switch traffic atomically (DNS, load balancer). Rollback = switch back. Fast, simple, but doubles infra cost during deploy.
- **Canary** — route a small percentage of live traffic (1–5%) to the new version, monitor error rate/latency, then gradually shift 10% → 50% → 100%. Catches problems with real traffic before full rollout. Requires good metrics/alerting and a traffic-splitting layer (service mesh, ALB weighted target groups, etc.).

Other variants:
- **Rolling update** — replace instances N at a time (Kubernetes `Deployment` default). Simple, no extra infra, but no atomic rollback.
- **Shadow / dark launch** — send a copy of live traffic to the new version but discard responses. Tests performance without user impact.

Deep dive: [CI/CD](../15-devops/01-ci-cd.md)

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

Deep dive: [FaaS](../15-devops/02-faas.md)

</details>

---

### 13. What is the difference between `git merge` and `git rebase`? When do you use each?

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

Deep dive: [Git Essentials](../15-devops/06-git-essentials.md)

</details>

---

### 14. How do you undo a commit? What's the difference between reset, revert, and amend?

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

Deep dive: [Git Essentials](../15-devops/06-git-essentials.md)

</details>

---

### 15. What does interactive rebase do, and when would you use it?

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

Deep dive: [Git Essentials](../15-devops/06-git-essentials.md)

</details>

---

### 16. What is `git reflog` and why is it your safety net?

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

Deep dive: [Git Essentials](../15-devops/06-git-essentials.md)

</details>

---

### 17. What is `git cherry-pick` and when would you use it?

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

Deep dive: [Git Essentials](../15-devops/06-git-essentials.md)

</details>

---

### 18. What is GitOps?

<details>
<summary>Reveal answer</summary>

**GitOps** is an operational model where **a Git repository is the single source of truth for system state**, and a reconciliation agent continuously applies that state to the cluster.

Four principles (OpenGitOps):
1. **Declarative** — the desired state is expressed declaratively (YAML, HCL).
2. **Versioned and immutable** — state is stored in Git with full history.
3. **Pulled automatically** — agents (ArgoCD, Flux) pull from Git; humans don't `kubectl apply`.
4. **Continuously reconciled** — the agent detects drift and auto-corrects.

Benefits: rollback is `git revert`, audit trail is the git log, PR reviews gate production changes. Biggest shift: CI builds artifacts, but **CD lives in the cluster**, not in the pipeline.

Deep dive: [CI/CD](../15-devops/01-ci-cd.md)

</details>

---
