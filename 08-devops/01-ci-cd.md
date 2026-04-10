# CI/CD

## Continuous Integration (CI)

It is the practice of **regularly integrating (merging)** code with the rest of the organization. In the past, it was common for individuals or teams to keep their code isolated in branches for many months and merge rarely.

### CI Practices:
- Frequent commits to the main branch
- Automated build on every commit
- Automated tests run on every build
- Fast feedback in case of failure

## Continuous Delivery (CD)

It is a philosophy and set of practices around **keeping your application always in a deployable state**. To achieve this, we build a **deployment pipeline** that serves to validate the correctness of changes and deliver them through a series of test environments, culminating in a production deploy.

## CI/CD together

It is the practice of merging changes frequently while developers work and having those changes go through a series of **automated tests**. Upon completion, these changes are packaged into a release candidate that can then be automatically deployed to production. Teams practicing CI/CD typically produce many release candidates in a single day.

## Tools

- **GitHub Actions** — CI/CD integrated with GitHub
- **Azure DevOps** — YAML pipelines, Azure integration
- **Jenkins** — open source, highly customizable
- **GitLab CI** — integrated with GitLab

## Example: GitHub Actions for .NET

```yaml
name: .NET CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: 8.0.x

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore

    - name: Test
      run: dotnet test --no-build --verbosity normal
```

## Example: Azure DevOps Pipeline

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: UseDotNet@2
    inputs:
      packageType: 'sdk'
      version: '8.0.x'

  - script: dotnet restore
    displayName: 'Restore'

  - script: dotnet build --configuration Release --no-restore
    displayName: 'Build'

  - script: dotnet test --configuration Release --no-build
    displayName: 'Test'

  - script: dotnet publish --configuration Release --output $(Build.ArtifactStagingDirectory)
    displayName: 'Publish'

  - task: PublishBuildArtifacts@1
    inputs:
      PathtoPublish: '$(Build.ArtifactStagingDirectory)'
      ArtifactName: 'app'
```

## Best practices

1. **Automated tests** — without tests, CI/CD has no value
2. **Fast build** — ideally less than 10 minutes
3. **Identical environments** — dev, staging, and production should be as similar as possible
4. **Easy rollback** — always have a way to revert to the previous version
5. **Feature flags** — allow deploying code without activating features
6. **Semantic versioning** — version your releases (MAJOR.MINOR.PATCH)

---

[Next: FaaS →](02-faas.md) | [Back to index](README.md)
