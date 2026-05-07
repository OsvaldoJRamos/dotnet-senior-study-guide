# Terraform (Infrastructure as Code)

## What it is

Terraform is an **Infrastructure as Code (IaC)** tool by HashiCorp. It defines infrastructure in declarative files (HCL) and provisions on any cloud (AWS, Azure, GCP).

## Why IaC

| Without IaC (manual) | With IaC (Terraform) |
|-------------------|---------------------|
| "Click here, configure there" | Code versioned in Git |
| Impossible to reproduce | Reproducible in any environment |
| No change history | Git blame shows who changed what |
| "Works on my cloud" | Same code = same infra |
| Prone to human error | Automated and reviewable in PRs |

## Fundamental concepts

### Provider

Plugin that connects Terraform to a cloud provider:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

### Resource

Infrastructure component:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-my-app-prod"
  location = "eastus2"
}

resource "azurerm_service_plan" "main" {
  name                = "asp-my-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_name            = "B1"
}

# Note: `azurerm_app_service_plan` is deprecated in azurerm v3+.
# Use `azurerm_service_plan` (schema is simpler — `os_type` and `sku_name`
# directly, no more `kind`/`reserved` juggling).

resource "azurerm_linux_web_app" "api" {
  name                = "app-my-api-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      dotnet_version = "8.0"
    }
  }
}
```

### Variables

```hcl
# variables.tf
variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # does not appear in logs
}

# Usage
resource "azurerm_resource_group" "main" {
  name = "rg-my-app-${var.environment}"
}
```

### Output

Output values after apply:

```hcl
output "api_url" {
  value = azurerm_linux_web_app.api.default_hostname
}

output "db_connection_string" {
  value     = azurerm_mssql_database.main.connection_string
  sensitive = true
}
```

## State

Terraform maintains a **state file** (`terraform.tfstate`) that maps declared resources vs actual resources in the cloud.

### Remote state (mandatory for teams)

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "my-app.tfstate"
  }
}
```

> **Never** commit `terraform.tfstate` to Git — it contains secrets.

## Basic workflow

```bash
# 1. Initializes providers and backend
terraform init

# 2. Shows what will change (dry-run)
terraform plan

# 3. Applies changes
terraform apply

# 4. Destroys everything (be careful!)
terraform destroy
```

## Full example: API + SQL Server on Azure

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-api-${var.environment}"
  location = "eastus2"
}

resource "azurerm_mssql_server" "main" {
  name                         = "sql-api-${var.environment}"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.db_password
}

resource "azurerm_mssql_database" "main" {
  name      = "db-api"
  server_id = azurerm_mssql_server.main.id
  sku_name  = "S0"
}

resource "azurerm_linux_web_app" "api" {
  name                = "app-api-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  app_settings = {
    "ConnectionStrings__Default" = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name};Database=${azurerm_mssql_database.main.name};User=${azurerm_mssql_server.main.administrator_login};Password=${var.db_password}"
  }
}
```

> **Don't put plaintext passwords in `app_settings`.** For production, store secrets in **Azure Key Vault** and reference them from App Service using **Managed Identity** — either via Key Vault reference syntax (`@Microsoft.KeyVault(SecretUri=...)`) or by resolving them in the app with `DefaultAzureCredential`. This keeps secrets out of Terraform state, CI logs, and portal views.

## Example: Infra on AWS

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "vpc-api-${var.environment}" }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_ecs_cluster" "main" {
  name = "cluster-api-${var.environment}"
}

resource "aws_ecs_task_definition" "api" {
  family                   = "api"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  container_definitions = jsonencode([{
    name      = "api"
    image     = "${var.ecr_repo}:latest"
    portMappings = [{ containerPort = 8080 }]
  }])
}

resource "aws_ecs_service" "api" {
  name            = "api-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = [aws_subnet.public.id]
    assign_public_ip = true
  }
}
```

## Modules (reuse)

```hcl
# Reusable module
module "api" {
  source      = "./modules/web-app"
  environment = "prod"
  app_name    = "my-api"
  sku         = "P1v2"
}

module "api_staging" {
  source      = "./modules/web-app"
  environment = "staging"
  app_name    = "my-api"
  sku         = "B1"
}
```

## Terraform vs other tools

| Aspect | Terraform | ARM/Bicep (Azure) | CloudFormation (AWS) | Pulumi |
|---------|-----------|--------------------|-----------------------|--------|
| Multi-cloud | Yes | No (Azure only) | No (AWS only) | Yes |
| Language | HCL | JSON/Bicep | JSON/YAML | C#, TS, Python |
| State | File (remote) | Managed by Azure | Managed by AWS | File (remote) |
| Ecosystem | Largest | Azure only | AWS only | Growing |

## Best practices

1. **Remote state** — never local for teams
2. **State locking** — prevents two people from applying at the same time
3. **Modules** — reuse code across environments
4. **`terraform plan` in CI** — review before apply
5. **Sensitive variables** — use `sensitive = true` and secret managers
6. **Separate root modules per environment** — one folder per env (`envs/dev`, `envs/staging`, `envs/prod`), each with its own backend. HashiCorp **does not recommend CLI workspaces for production environment isolation** — they share the same backend and provider config, making drift and blast-radius hard to control. Workspaces are best for ephemeral/testing scenarios (feature branches, per-developer sandboxes).
7. **`.gitignore`** — ignore `*.tfstate`, `*.tfstate.backup`, `.terraform/`

---

[← Previous: Docker and Kubernetes](03-docker-and-kubernetes.md) | [Next: Azure Pipelines →](05-azure-pipelines.md) | [Back to index](README.md)
