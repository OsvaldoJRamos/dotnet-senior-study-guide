# CI/CD

## Continuous Integration (CI)

É a prática de **integrar (merge) regularmente** código com o restante da organização. Antigamente era comum indivíduos ou times manterem seu código isolado em branches por muitos meses e fazer merge raramente.

### Práticas de CI:
- Commits frequentes na branch principal
- Build automatizado a cada commit
- Testes automatizados executados a cada build
- Feedback rápido em caso de falha

## Continuous Delivery (CD)

É uma filosofia e conjunto de práticas em torno de **manter sua aplicação sempre em um estado deployável**. Para alcançar isso, construímos um **deployment pipeline** que serve para validar a corretude das mudanças e entregá-las através de uma série de ambientes de teste, culminando em um deploy em produção.

## CI/CD juntos

É a prática de fazer merge de mudanças frequentemente enquanto os devs trabalham e ter essas mudanças passando por uma série de **testes automatizados**. Ao completar, essas mudanças são empacotadas em um release candidate que pode então ser deployed automaticamente em produção. Times praticando CI/CD tipicamente produzem muitos release candidates em um único dia.

## Ferramentas

- **GitHub Actions** — CI/CD integrado ao GitHub
- **Azure DevOps** — pipelines YAML, integração com Azure
- **Jenkins** — open source, altamente customizável
- **GitLab CI** — integrado ao GitLab

## Exemplo: GitHub Actions para .NET

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

## Exemplo: Azure DevOps Pipeline

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

## Boas práticas

1. **Testes automatizados** — sem testes, CI/CD não tem valor
2. **Build rápido** — idealmente menos de 10 minutos
3. **Ambientes iguais** — dev, staging e produção devem ser o mais parecidos possível
4. **Rollback fácil** — sempre tenha como voltar à versão anterior
5. **Feature flags** — permitem deploy de código sem ativar funcionalidades
6. **Semantic versioning** — versione seus releases (MAJOR.MINOR.PATCH)

---

[Próximo: FaaS →](02-faas.md) | [Voltar ao índice](README.md)
