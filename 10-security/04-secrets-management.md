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

- *"Environment variables are generally stored in plain, unencrypted text. If the machine or process is compromised, environment variables can be accessed by untrusted parties."*
- The `:` separator for hierarchical keys isn't supported by Bash. Use `__` (double underscore); .NET's configuration rewrites `__` to `:`. Example: `ConnectionStrings__Default` → `ConnectionStrings:Default`.
- Environment variables **override** all previously loaded configuration sources, which is useful for prod overrides but can mask misconfiguration silently.

Fine for non-secret config and for small deployments. For real secrets, treat env vars as a **transport** — let the orchestrator inject values from a real secret manager, don't hard-code them in `docker-compose.yml`.

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

> Pick one model per vault; mixing them is a known source of "it works in dev, 403 in prod" incidents. Microsoft recommends Azure RBAC for new vaults.

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
