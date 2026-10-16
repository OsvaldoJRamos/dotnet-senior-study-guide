# Secrets Management

Secrets — connection strings, API keys, private keys, SaaS tokens — leak through three well-known routes: **source control**, **application logs**, and **process memory dumps**. The senior job is picking a mechanism per environment, rotating secrets on a schedule, and never falling back to a `.env` committed by a new hire on Friday.

## Decision table

| Environment | Use |
|---|---|
| Local developer machine (.NET) | `dotnet user-secrets` — keeps secrets out of the repo, specific to your user profile |
| CI pipeline | CI provider's secret store (GitHub Actions secrets, Azure DevOps library, AWS CodeBuild env) + OIDC federation where possible |
| Azure workload | **Azure Key Vault** via Managed Identity |
| AWS workload | **AWS Secrets Manager** (or Parameter Store for non-rotating config) via IAM role |
| Kubernetes | External Secrets Operator pulling from Key Vault / Secrets Manager; avoid base64 "Kubernetes Secrets" as the source of truth |
| On-prem | HashiCorp Vault (most common) or Conjur |

> Rule of thumb: whichever backend you pick, the **app should authenticate to it with a workload identity** (Managed Identity, IAM role, IRSA, Workload Identity Federation) — **not** a long-lived token that is itself a secret.

## .NET User Secrets (dev only)

Microsoft's own guidance: *"The Secret Manager tool doesn't encrypt the stored secrets and shouldn't be treated as a trusted store. It's for development purposes only."*

Initialize and use:

```bash
# In the project folder
dotnet user-secrets init
dotnet user-secrets set "Movies:ServiceApiKey" "12345"
dotnet user-secrets list
dotnet user-secrets remove "Movies:ServiceApiKey"
dotnet user-secrets clear
```

Storage path:

- Windows: `%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json`
- Linux / macOS: `~/.microsoft/usersecrets/<user_secrets_id>/secrets.json`

The `<user_secrets_id>` is a GUID added to the project file as `<UserSecretsId>`. `WebApplication.CreateBuilder` calls `AddUserSecrets` automatically when `EnvironmentName` is `Development`, so reading `builder.Configuration["Movies:ServiceApiKey"]` just works.

> User Secrets is only for your machine. Never ship `secrets.json` anywhere, and never use it as a production strategy.

## Environment variables

Environment variables are the lowest common denominator — supported by every CI system, container orchestrator, and serverless host.

Known pitfalls the docs flag:

- *"Environment variables are commonly stored as plain, unencrypted text. If the machine or process is compromised, environment variables are accessible to untrusted parties."*
- The `:` separator for hierarchical keys isn't supported by Bash. Use `__` (double underscore): *"All platforms support the double underscore (`__`) syntax and automatically replace it with a colon (`:`)."* Example: `ConnectionStrings__Default` → `ConnectionStrings:Default`.
- In the default ASP.NET Core setup, *"key values read from the environment override values read from `appsettings.json`, `appsettings.{ENVIRONMENT}.json`, and user secrets"* (only command-line arguments rank higher). That is useful for prod overrides, but it can mask misconfiguration silently.

Fine for non-secret config and for small deployments. For real secrets, treat env vars as a **transport**: let the orchestrator inject values from a real secret manager. Don't hard-code them in `docker-compose.yml`. The next section expands on this.

## Environment variable vs secret

They are on **different axes**, which is why people conflate them:

- A **secret** is a *classification of data*: anything that grants access (passwords, API keys, private keys, tokens, connection strings with credentials).
- An **environment variable** is a *delivery mechanism*: one of several ways a value reaches the process.

The confusion comes from the 12-factor app, which defines config as *"everything that is likely to vary between deploys"*, explicitly including *"Credentials to external services"*, and says *"The twelve-factor app stores config in environment variables."* That was good advice against committing config files. It is weaker advice for secrets.

| | Config (non-secret) | Secret |
|---|---|---|
| Examples | Feature flags, URLs, timeouts, log levels | Passwords, API keys, signing keys, connection strings with credentials |
| Impact if leaked | Mostly harmless | Grants access, so it's a breach |
| Source of truth | Repo (`appsettings.json`), env, config service | A secret manager (Key Vault, Secrets Manager, Vault), **never** the repo |
| Rotation | On deploy | On a schedule and on incident, ideally without a redeploy |
| Access audit | Not needed | Reads can be audited by the vault (AWS CloudTrail records Secrets Manager calls; Key Vault logs once you enable logging) |

### Three ways to deliver a secret to the app

| Mechanism | How | Rotation | Leak surface |
|---|---|---|---|
| **Environment variable** | Orchestrator injects it at start: K8s `secretKeyRef`, ECS `secrets`/`valueFrom`, App Service settings | **Needs a restart.** K8s: *"a Secret update will not be seen by the container unless it is restarted."* ECS: *"the container will not receive the updated value automatically."* | Every process in the container, child processes, `/proc/<pid>/environ`, logs, crash dumps |
| **Mounted file** | K8s Secret volume; Docker/Compose secrets at `/run/secrets/<name>` | K8s volumes update *"using an eventually-consistent approach"* (not with `subPath` mounts) | File permissions; only code that opens the file |
| **Fetch from the vault at runtime** | `AddAzureKeyVault`, AWS SDK, Vault client, authenticated by **workload identity** | Poll or refresh in-process, no restart | Process memory (still visible in memory dumps); reads can be audited |

Docker made the file-based design choice explicitly for Swarm: *"Docker secrets do not set environment variables directly. This was a conscious decision, because environment variables can unintentionally be leaked between containers."*

### Why environment variables leak

- **Inheritance.** *"When a child process is created via fork(2), it inherits a copy of its parent's environment."* Every tool you shell out to gets your database password.
- **`/proc/<pid>/environ`** exposes *"the initial environment that was set when the currently executing program was started"* to anyone who passes the kernel's ptrace access check.
- **Logs and dumps.** OWASP: *"environment variables are generally accessible to all processes and may be included in logs or system dumps. Using environment variables is therefore not recommended unless the other methods are not possible."* Docker Compose's docs add that they *"can also be printed in logs when debugging errors without your knowledge."*
- **Baked into images.** *"Build arguments and environment variables are inappropriate for passing secrets to your build, because they persist in the final image."* `ENV` values are visible with `docker inspect`.
- **Kubernetes Secrets aren't encrypted by default.** They are *"stored unencrypted in the API server's underlying data store (etcd)"*, and *"Base64 encoding is not an encryption method."*

### .NET: reading secrets from files or the vault

```csharp
// Mounted files (K8s Secret volume, /run/secrets): file name = key, file content = value.
// A file named ConnectionStrings__Default becomes ConnectionStrings:Default.
// Package: Microsoft.Extensions.Configuration.KeyPerFile
builder.Configuration.AddKeyPerFile("/run/secrets", optional: true, reloadOnChange: true);

// Straight from Key Vault, no secret to bootstrap. DefaultAzureCredential uses the managed identity in Azure;
// for production, Microsoft recommends a specific credential such as ManagedIdentityCredential.
// Packages: Azure.Extensions.AspNetCore.Configuration.Secrets, Azure.Identity
builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential(),
    new AzureKeyVaultConfigurationOptions { ReloadInterval = TimeSpan.FromMinutes(5) });
```

Both end up in `IConfiguration`, so application code doesn't change. Note the Key Vault provider's defaults: *"By default, the configuration provider caches secrets for the application lifetime"*, and `ReloadInterval` defaults to `null`, meaning no reload.

### Platform shortcuts

- **Azure App Service:** *"app settings are variables passed as environment variables to the application code"* and are *"encrypted at rest"*. Changing them *"triggers an app restart."* Key Vault references (`@Microsoft.KeyVault(SecretUri=...)`) keep the secret in Key Vault with *"No code changes"*. App Service *"refetches them every 24 hours."*
- **AWS Lambda:** env vars are encrypted at rest with KMS, but AWS still *"recommend[s] that you use AWS Secrets Manager instead of environment variables to store database credentials."*
- **AWS ECS:** `secrets` + `valueFrom` pulls from Secrets Manager / SSM into an env var at container start. It is convenient, but *"Applications that run on the container and container logs and debugging tools have access to the environment variables."*

### Preference order

1. **No secret at all:** workload identity (Managed Identity, IAM role) to reach the database, storage or queue.
2. **Fetch from the vault at runtime**, authenticated by workload identity.
3. **Orchestrator-mounted file** sourced from the vault.
4. **Orchestrator-injected env var** sourced from the vault (ECS `valueFrom`, App Service Key Vault reference). Acceptable, but the process only sees a rotated value after a restart (ECS: launch a new task; App Service: automatic refetch within 24 hours or on any config change).
5. **Never:** a secret value typed into `docker-compose.yml`, a K8s manifest, a Dockerfile `ENV`/`ARG`, or a committed pipeline file.

> **Interview answer:** "An environment variable is a way to deliver a value; a secret is a kind of value. You can deliver a secret through an env var, and many platforms do, but env vars are plain text, inherited by child processes, readable through `/proc`, and easy to leak into logs and images, and rotation needs a restart. So I keep secrets in a vault, prefer workload identity so there's no secret at all, and if I need delivery, I mount a file or fetch at runtime. I use env vars for non-secret config."

## Azure Key Vault

From the Microsoft overview, Key Vault solves three adjacent problems:

- **Secrets Management** — API keys, passwords, connection strings.
- **Key Management** — cryptographic keys (software or HSM-protected).
- **Certificate Management** — TLS/SSL certs with auto-renewal.

### Tiers

- **Standard tier** — data encrypted using software modules validated to FIPS 140 Level 1.
- **Premium tier** — adds HSM-protected keys on Marvell LiquidSecurity HSMs validated to FIPS 140-3 Level 3.

### Access control

Authentication is always against Microsoft Entra ID. For authorization, Azure offers two models:

- **Azure RBAC** (the newer, recommended model).
- **Key Vault access policies** (the older, per-vault model).

> Pick one model per vault; mixing them is a known source of "it works in dev, 403 in prod" incidents. Per the Azure Key Vault RBAC guide, *"Starting with API version 2026-02-01, Azure RBAC is the default access control model for newly created key vaults"* — prefer Azure RBAC for all new vaults.

### .NET client — Managed Identity preferred

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

// DefaultAzureCredential walks through: env vars -> Managed Identity -> Visual Studio -> Azure CLI...
// In Azure, Managed Identity is picked up automatically — no connection string needed.
var client = new SecretClient(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential());

KeyVaultSecret secret = await client.GetSecretAsync("Db-ConnectionString");
string value = secret.Value;
```

Or wire Key Vault into configuration directly, using the Azure Key Vault configuration provider (package `Azure.Extensions.AspNetCore.Configuration.Secrets`) so `builder.Configuration["Db-ConnectionString"]` resolves from the vault.

## AWS Secrets Manager

From the AWS docs: *"AWS Secrets Manager helps you manage, retrieve, and rotate database credentials, application credentials, OAuth tokens, API keys, and other secrets throughout their lifecycles."*

### Rotation

Secrets Manager can **rotate** credentials on a schedule. For non-managed rotation, rotation runs via an AWS Lambda function that you own; for supported services (RDS, DocumentDB, Redshift), AWS provides managed rotation that handles the Lambda for you.

Two patterns the AWS docs explicitly document:

- **Single-user rotation** — one credential, changed in place. Brief outage as old credential invalidates.
- **Alternating-user rotation** — two users, alternate each rotation. Zero-downtime at the cost of a second user in the database.

### .NET client

```csharp
using Amazon.SecretsManager;
using Amazon.SecretsManager.Model;

var client = new AmazonSecretsManagerClient(); // uses IAM role in prod (EC2 / ECS / EKS / Lambda)

var response = await client.GetSecretValueAsync(new GetSecretValueRequest
{
    SecretId = "prod/myapp/db"
});

string json = response.SecretString; // parse into your shape (usually JSON with user/password/host)
```

### Key Vault vs Secrets Manager

| | Azure Key Vault | AWS Secrets Manager |
|---|---|---|
| Secrets | ✅ | ✅ |
| Keys / HSM | ✅ (Premium uses FIPS 140-3 Level 3 HSMs) | Use **AWS KMS** instead |
| Certificates | ✅ | Use **AWS Certificate Manager** instead |
| Built-in rotation | Limited (no generic rotation Lambda concept) | First-class, including managed rotation for RDS and others |
| Auth | Entra ID + RBAC / access policies | IAM roles / policies |
| Price model | Per operation | Per secret per month + per API call |

For **cryptographic keys**, AWS splits them across multiple services: AWS KMS (keys), ACM (certificates), Secrets Manager (opaque secrets). Azure Key Vault puts all three in one product with different APIs.

## Rotation strategies

Rotation is the second half of secret hygiene. A secret that never changes is one that has already leaked somewhere you haven't noticed.

### Rotation cadence by secret type

| Secret type | Typical cadence |
|---|---|
| Database credentials | 30–90 days, or on-demand after personnel changes |
| API keys to external SaaS | 90 days or per vendor policy |
| Signing keys (JWT, OIDC) | 90 days with overlapping key lifetime |
| TLS certificates | Per CA (ACME/Let's Encrypt = 90 days; public CAs = 1 year) |
| Break-glass admin credentials | Immediately after use |

### Patterns

- **Rotate in place** — works only if callers fetch fresh on each request (acceptable for low-throughput apps). Risky otherwise because cached connections use the old credential.
- **Dual-credential / alternating users** — the Secrets Manager "Alternating users" pattern. Zero-downtime.
- **Key-rollover window** — publish N+1 valid keys simultaneously, move readers to the new one, then retire the old. The default model for JWT signing keys.
- **Automated via the secret manager** — always prefer this; rotation code you write will bit-rot.

### Revocation

When a secret leaks — committed to git, pasted in a Slack channel, shown in a demo — **rotate immediately**, don't debate. If it was in source control, **also rotate anything that was built with access to it** (CI tokens, signing keys, webhooks).

## Anti-patterns

- **Committing `.env` files or `appsettings.Development.json` with real credentials.** Use `.gitignore`; use a pre-commit hook with `gitleaks` / `trufflehog`.
- **"Environment variable with the actual value" in the Kubernetes manifest.** Use Secret objects sealed with SOPS/sealed-secrets or External Secrets Operator.
- **One Key Vault shared across every environment.** Use one vault per environment (dev / staging / prod), and separate vaults per app when possible. Separate blast radius.
- **Logging the secret "just to debug".** Install a log scrubber; review the logging code with the same seriousness as auth code.
- **Letting the developer laptop be the source of truth** — if prod uses a secret that only lives on Alice's machine, the bus factor is 1.
- **Long-lived static tokens in CI.** Prefer OIDC federation: GitHub Actions, Azure DevOps, GitLab all support short-lived token issuance against Entra ID / AWS IAM.

---

[← Previous: Authorization](03-authorization.md) | [Next: Threat Modeling →](05-threat-modeling.md) | [Back to index](README.md)
