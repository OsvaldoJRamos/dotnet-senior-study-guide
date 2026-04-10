# Terraform (Infrastructure as Code)

## O que e

Terraform e uma ferramenta de **Infrastructure as Code (IaC)** da HashiCorp. Define infraestrutura em arquivos declarativos (HCL) e provisiona em qualquer cloud (AWS, Azure, GCP).

## Por que IaC

| Sem IaC (manual) | Com IaC (Terraform) |
|-------------------|---------------------|
| "Clica aqui, configura ali" | Codigo versionado no Git |
| Impossivel reproduzir | Reproduzivel em qualquer ambiente |
| Sem historico de mudancas | Git blame mostra quem mudou o que |
| "Funciona na minha cloud" | Mesmo codigo = mesma infra |
| Propenso a erro humano | Automatizado e revisavel em PR |

## Conceitos fundamentais

### Provider

Plugin que conecta o Terraform a um cloud provider:

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

Componente de infraestrutura:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-minha-app-prod"
  location = "eastus2"
}

resource "azurerm_app_service_plan" "main" {
  name                = "asp-minha-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_name            = "B1"
}

resource "azurerm_linux_web_app" "api" {
  name                = "app-minha-api-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_app_service_plan.main.id

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
  description = "Ambiente (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "db_password" {
  description = "Senha do banco"
  type        = string
  sensitive   = true  # nao aparece nos logs
}

# Uso
resource "azurerm_resource_group" "main" {
  name = "rg-minha-app-${var.environment}"
}
```

### Output

Valores de saida apos o apply:

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

Terraform mantem um **state file** (`terraform.tfstate`) que mapeia recursos declarados vs recursos reais na cloud.

### Remote state (obrigatorio em time)

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "minha-app.tfstate"
  }
}
```

> **Nunca** commite o `terraform.tfstate` no Git — contem secrets.

## Workflow basico

```bash
# 1. Inicializa providers e backend
terraform init

# 2. Mostra o que vai mudar (dry-run)
terraform plan

# 3. Aplica as mudancas
terraform apply

# 4. Destroi tudo (cuidado!)
terraform destroy
```

## Exemplo completo: API + SQL Server no Azure

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
  service_plan_id     = azurerm_app_service_plan.main.id

  app_settings = {
    "ConnectionStrings__Default" = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name};Database=${azurerm_mssql_database.main.name};User=${azurerm_mssql_server.main.administrator_login};Password=${var.db_password}"
  }
}
```

## Exemplo: Infra na AWS

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

## Modules (reutilizacao)

```hcl
# Modulo reutilizavel
module "api" {
  source      = "./modules/web-app"
  environment = "prod"
  app_name    = "minha-api"
  sku         = "P1v2"
}

module "api_staging" {
  source      = "./modules/web-app"
  environment = "staging"
  app_name    = "minha-api"
  sku         = "B1"
}
```

## Terraform vs outras ferramentas

| Aspecto | Terraform | ARM/Bicep (Azure) | CloudFormation (AWS) | Pulumi |
|---------|-----------|--------------------|-----------------------|--------|
| Multi-cloud | Sim | Nao (so Azure) | Nao (so AWS) | Sim |
| Linguagem | HCL | JSON/Bicep | JSON/YAML | C#, TS, Python |
| State | Arquivo (remote) | Gerenciado pelo Azure | Gerenciado pela AWS | Arquivo (remote) |
| Ecossistema | Maior | Azure only | AWS only | Crescendo |

## Boas praticas

1. **Remote state** — nunca local em time
2. **State locking** — evita duas pessoas aplicando ao mesmo tempo
3. **Modules** — reutilize codigo entre ambientes
4. **`terraform plan` em CI** — review antes de apply
5. **Variáveis sensíveis** — use `sensitive = true` e secret managers
6. **Workspaces ou pastas por ambiente** — isole dev/staging/prod
7. **`.gitignore`** — ignore `*.tfstate`, `*.tfstate.backup`, `.terraform/`

---

[← Anterior: Docker e Kubernetes](03-docker-e-kubernetes.md) | [Próximo: Azure Pipelines →](05-azure-pipelines.md) | [Voltar ao índice](README.md)
