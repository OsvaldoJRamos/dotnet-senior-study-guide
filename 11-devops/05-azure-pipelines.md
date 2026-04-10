# Azure Pipelines

## What it is

CI/CD service from Azure DevOps. Defines pipelines in **YAML** that automate build, tests, and deployment.

## Pipeline Structure

```yaml
trigger:           # When to execute
stages:            # Stages (Build, Test, Deploy)
  - stage:
    jobs:          # Jobs within each stage
      - job:
        steps:     # Steps within each job
```

## Basic pipeline (.NET)

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
    displayName: 'Install .NET SDK'
    inputs:
      version: $(dotnetVersion)

  - task: DotNetCoreCLI@2
    displayName: 'Restore packages'
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
    displayName: 'Run tests'
    inputs:
      command: 'test'
      projects: '**/*Tests.csproj'
      arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'

  - task: DotNetCoreCLI@2
    displayName: 'Publish'
    inputs:
      command: 'publish'
      publishWebProjects: true
      arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'

  - task: PublishBuildArtifacts@1
    displayName: 'Publish artifact'
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
    displayName: 'Build and Tests'
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
            displayName: 'Tests'

          - task: DotNetCoreCLI@2
            displayName: 'Publish'
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
                    azureSubscription: 'my-subscription'
                    appType: 'webAppLinux'
                    appName: 'app-api-staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'

  # ========== CD - Production ==========
  - stage: DeployProd
    displayName: 'Deploy Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployToProd
        environment: 'production'   # can have approval gate
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'my-subscription'
                    appType: 'webAppLinux'
                    appName: 'app-api-prod'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
```

## Pipeline with Docker

```yaml
stages:
  - stage: Build
    jobs:
      - job: DockerBuild
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: Docker@2
            displayName: 'Build and Push'
            inputs:
              containerRegistry: 'my-acr'
              repository: 'my-api'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest
```

## Templates (reuse)

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

# Main pipeline
steps:
  - template: templates/dotnet-build.yml
    parameters:
      project: 'src/MyApi/MyApi.csproj'
```

## Variables and Secrets

```yaml
# Inline variables
variables:
  buildConfig: 'Release'

# Variable group (defined in Azure DevOps)
variables:
  - group: 'production-secrets'

# Environment variables
steps:
  - script: echo $(DB_PASSWORD)
    env:
      DB_PASSWORD: $(dbPassword)  # secret does not appear in logs
```

## Environments and Approval Gates

In Azure DevOps:
1. Create an **Environment** (e.g.: "production")
2. Add an **Approval check** — someone must approve before the deploy
3. Add **Branch control** — only deploy from `main`

## Important concepts

| Concept | Description |
|----------|-----------|
| **Trigger** | What starts the pipeline (push, PR, schedule) |
| **Pool/Agent** | Machine that executes (hosted or self-hosted) |
| **Artifact** | Build result (ZIP, Docker image) |
| **Environment** | Deploy destination (staging, prod) |
| **Condition** | When a stage/job should execute |
| **Service Connection** | Credentials to connect to services (Azure, Docker Hub) |

## Best practices

1. **Tests before deploy** — never skip tests in the pipeline
2. **Approval gates** in production — a human approves before deploying
3. **Secrets in variable groups** — never hardcode in YAML
4. **Templates** — reuse steps across pipelines
5. **Branch policies** — require PR + green build before merge
6. **Semantic versioning** — version artifacts (GitVersion, MinVer)
7. **NuGet cache** — speeds up builds using package cache

---

[← Previous: Terraform](04-terraform.md) | [Back to index](README.md)
