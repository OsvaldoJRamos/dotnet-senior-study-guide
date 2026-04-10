# Azure Pipelines

## O que e

Servico de CI/CD do Azure DevOps. Define pipelines em **YAML** que automatizam build, testes e deploy.

## Estrutura de um Pipeline

```yaml
trigger:           # Quando executar
stages:            # Etapas (Build, Test, Deploy)
  - stage:
    jobs:          # Trabalhos dentro de cada etapa
      - job:
        steps:     # Passos dentro de cada job
```

## Pipeline basico (.NET)

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

steps:
  - task: UseDotNet@2
    displayName: 'Instalar .NET SDK'
    inputs:
      version: $(dotnetVersion)

  - task: DotNetCoreCLI@2
    displayName: 'Restaurar pacotes'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'

  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      projects: '**/*.csproj'
      arguments: '--configuration $(buildConfiguration) --no-restore'

  - task: DotNetCoreCLI@2
    displayName: 'Rodar testes'
    inputs:
      command: 'test'
      projects: '**/*Tests.csproj'
      arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'

  - task: DotNetCoreCLI@2
    displayName: 'Publicar'
    inputs:
      command: 'publish'
      publishWebProjects: true
      arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'

  - task: PublishBuildArtifacts@1
    displayName: 'Publicar artefato'
    inputs:
      PathtoPublish: '$(Build.ArtifactStagingDirectory)'
      ArtifactName: 'drop'
```

## Multi-stage Pipeline (CI + CD)

```yaml
trigger:
  - main

variables:
  buildConfiguration: 'Release'

stages:
  # ========== CI ==========
  - stage: Build
    displayName: 'Build e Testes'
    jobs:
      - job: BuildJob
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            inputs:
              version: '8.0.x'

          - script: dotnet build --configuration $(buildConfiguration)
            displayName: 'Build'

          - script: dotnet test --configuration $(buildConfiguration) --no-build
            displayName: 'Testes'

          - task: DotNetCoreCLI@2
            displayName: 'Publicar'
            inputs:
              command: publish
              publishWebProjects: true
              arguments: '-c $(buildConfiguration) -o $(Build.ArtifactStagingDirectory)'

          - publish: $(Build.ArtifactStagingDirectory)
            artifact: drop

  # ========== CD - Staging ==========
  - stage: DeployStaging
    displayName: 'Deploy Staging'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployToStaging
        environment: 'staging'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'minha-subscription'
                    appType: 'webAppLinux'
                    appName: 'app-api-staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'

  # ========== CD - Producao ==========
  - stage: DeployProd
    displayName: 'Deploy Producao'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployToProd
        environment: 'production'   # pode ter approval gate
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'minha-subscription'
                    appType: 'webAppLinux'
                    appName: 'app-api-prod'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
```

## Pipeline com Docker

```yaml
stages:
  - stage: Build
    jobs:
      - job: DockerBuild
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: Docker@2
            displayName: 'Build e Push'
            inputs:
              containerRegistry: 'meu-acr'
              repository: 'minha-api'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest
```

## Templates (reutilizacao)

```yaml
# templates/dotnet-build.yml
parameters:
  - name: project
    type: string
  - name: configuration
    type: string
    default: 'Release'

steps:
  - task: DotNetCoreCLI@2
    displayName: 'Build ${{ parameters.project }}'
    inputs:
      command: build
      projects: '${{ parameters.project }}'
      arguments: '-c ${{ parameters.configuration }}'

# Pipeline principal
steps:
  - template: templates/dotnet-build.yml
    parameters:
      project: 'src/MinhaApi/MinhaApi.csproj'
```

## Variaveis e Secrets

```yaml
# Variaveis inline
variables:
  buildConfig: 'Release'

# Grupo de variaveis (definido no Azure DevOps)
variables:
  - group: 'production-secrets'

# Variaveis de ambiente
steps:
  - script: echo $(DB_PASSWORD)
    env:
      DB_PASSWORD: $(dbPassword)  # secret nao aparece nos logs
```

## Environments e Approval Gates

No Azure DevOps:
1. Criar **Environment** (ex: "production")
2. Adicionar **Approval check** — alguem precisa aprovar antes do deploy
3. Adicionar **Branch control** — so deploy de `main`

## Conceitos importantes

| Conceito | Descricao |
|----------|-----------|
| **Trigger** | O que dispara o pipeline (push, PR, schedule) |
| **Pool/Agent** | Maquina que executa (hosted ou self-hosted) |
| **Artifact** | Resultado do build (ZIP, Docker image) |
| **Environment** | Destino do deploy (staging, prod) |
| **Condition** | Quando uma stage/job deve executar |
| **Service Connection** | Credenciais para conectar em servicos (Azure, Docker Hub) |

## Boas praticas

1. **Testes antes do deploy** — nunca pule testes no pipeline
2. **Approval gates** em producao — humano aprova antes de deployar
3. **Secrets em variable groups** — nunca hardcode no YAML
4. **Templates** — reutilize steps entre pipelines
5. **Branch policies** — exija PR + build verde antes de merge
6. **Semantic versioning** — versione artefatos (GitVersion, MinVer)
7. **Cache de NuGet** — acelera builds usando cache de pacotes

---

[← Anterior: Terraform](04-terraform.md) | [Voltar ao índice](README.md)
