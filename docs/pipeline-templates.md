# Pipeline Templates

This document describes the centrally-managed Azure Pipeline templates that scaffolded projects reference.

## Overview

Pipeline templates are stored in a dedicated `pipeline-templates` repository. Scaffolded projects use the `extends:` keyword to reference these templates, ensuring:

- **No drift** - All projects use the same build/deploy logic
- **Centralised updates** - Changes to templates automatically apply to all projects
- **Consistency** - Standardised stages, jobs, and steps across all services

## Repository Structure

```
pipeline-templates/
├── azure-pipelines.yml              # Pipeline to validate templates
├── README.md
├── templates/
│   ├── stages/
│   │   ├── build-dotnet.yml         # Build .NET projects
│   │   ├── deploy-infra.yml         # Deploy Bicep infrastructure
│   │   └── deploy-function.yml      # Deploy Azure Function App
│   ├── jobs/
│   │   ├── dotnet-build.yml         # .NET build job
│   │   ├── dotnet-test.yml          # .NET test job
│   │   ├── bicep-validate.yml       # Validate Bicep files
│   │   └── bicep-what-if.yml        # Bicep what-if analysis
│   └── steps/
│       ├── azure-login.yml          # Azure CLI login
│       ├── dotnet-restore.yml       # NuGet restore
│       ├── dotnet-build.yml         # dotnet build
│       ├── dotnet-test.yml          # dotnet test
│       ├── dotnet-publish.yml       # dotnet publish
│       └── deploy-bicep.yml         # Bicep deployment
└── pipelines/
    ├── function-app-rest.yml        # Complete pipeline for REST API
    ├── function-app-soap.yml        # Complete pipeline for SOAP API
    └── function-app-event.yml       # Complete pipeline for Event-driven
```

## How Projects Reference Templates

### In Scaffolded Project: `azure-pipelines.yml`

```yaml
trigger:
  branches:
    include:
      - main

resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/heads/main

variables:
  - name: projectName
    value: 'customer-orders'
  - name: azureSubscription
    value: 'your-service-connection'

extends:
  template: pipelines/function-app-rest.yml@templates
  parameters:
    projectName: $(projectName)
    azureSubscription: $(azureSubscription)
    dotnetVersion: '8.0'
    environments:
      - name: nonprod
        resourceGroup: 'rg-customer-orders-nonprod-aue'
      - name: prod
        resourceGroup: 'rg-customer-orders-prod-aue'
        dependsOn: nonprod
```

## Pipeline Templates

### Main Pipelines

#### `pipelines/function-app-rest.yml`

Complete pipeline for REST API projects:

```yaml
parameters:
  - name: projectName
    type: string
  - name: azureSubscription
    type: string
  - name: dotnetVersion
    type: string
    default: '8.0'
  - name: environments
    type: object
    default:
      - name: nonprod
        resourceGroup: ''
      - name: prod
        resourceGroup: ''
        dependsOn: nonprod

stages:
  # Build Stage
  - template: ../templates/stages/build-dotnet.yml
    parameters:
      projectName: ${{ parameters.projectName }}
      dotnetVersion: ${{ parameters.dotnetVersion }}
      projectPath: 'src/${{ parameters.projectName }}.Functions'

  # Deploy to each environment
  - ${{ each env in parameters.environments }}:
    # Infrastructure
    - template: ../templates/stages/deploy-infra.yml
      parameters:
        stageName: 'Infrastructure_${{ env.name }}'
        environment: ${{ env.name }}
        resourceGroup: ${{ env.resourceGroup }}
        azureSubscription: ${{ parameters.azureSubscription }}
        bicepFile: 'infra/main.bicep'
        bicepParams: 'infra/parameters/${{ env.name }}.bicepparam'
        ${{ if env.dependsOn }}:
          dependsOn:
            - 'Infrastructure_${{ env.dependsOn }}'
            - 'Deploy_${{ env.dependsOn }}'

    # Function App
    - template: ../templates/stages/deploy-function.yml
      parameters:
        stageName: 'Deploy_${{ env.name }}'
        environment: ${{ env.name }}
        azureSubscription: ${{ parameters.azureSubscription }}
        functionAppName: 'func-${{ parameters.projectName }}-${{ env.name }}-aue'
        dependsOn:
          - 'Infrastructure_${{ env.name }}'
```

#### `pipelines/function-app-event.yml`

Event-driven pipeline (includes Service Bus setup):

```yaml
parameters:
  - name: projectName
    type: string
  - name: azureSubscription
    type: string
  - name: dotnetVersion
    type: string
    default: '8.0'
  - name: environments
    type: object

stages:
  # Build Stage
  - template: ../templates/stages/build-dotnet.yml
    parameters:
      projectName: ${{ parameters.projectName }}
      dotnetVersion: ${{ parameters.dotnetVersion }}
      projectPath: 'src/${{ parameters.projectName }}.Functions'

  # Deploy to each environment
  - ${{ each env in parameters.environments }}:
    # Infrastructure (includes Service Bus)
    - template: ../templates/stages/deploy-infra.yml
      parameters:
        stageName: 'Infrastructure_${{ env.name }}'
        environment: ${{ env.name }}
        resourceGroup: ${{ env.resourceGroup }}
        azureSubscription: ${{ parameters.azureSubscription }}
        bicepFile: 'infra/main.bicep'
        bicepParams: 'infra/parameters/${{ env.name }}.bicepparam'

    # Function App
    - template: ../templates/stages/deploy-function.yml
      parameters:
        stageName: 'Deploy_${{ env.name }}'
        environment: ${{ env.name }}
        azureSubscription: ${{ parameters.azureSubscription }}
        functionAppName: 'func-${{ parameters.projectName }}-${{ env.name }}-aue'
        dependsOn:
          - 'Infrastructure_${{ env.name }}'
```

### Stage Templates

#### `templates/stages/build-dotnet.yml`

```yaml
parameters:
  - name: projectName
    type: string
  - name: dotnetVersion
    type: string
  - name: projectPath
    type: string
  - name: testProjectPath
    type: string
    default: ''

stages:
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: Build
        displayName: 'Build .NET Project'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            displayName: 'Install .NET SDK'
            inputs:
              packageType: 'sdk'
              version: '${{ parameters.dotnetVersion }}.x'

          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: 'restore'
              projects: '${{ parameters.projectPath }}/**/*.csproj'

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              projects: '${{ parameters.projectPath }}/**/*.csproj'
              arguments: '--configuration Release --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Test'
            inputs:
              command: 'test'
              projects: 'tests/**/*.csproj'
              arguments: '--configuration Release --no-build --collect:"XPlat Code Coverage"'
            condition: and(succeeded(), ne('${{ parameters.testProjectPath }}', ''))

          - task: DotNetCoreCLI@2
            displayName: 'Publish'
            inputs:
              command: 'publish'
              projects: '${{ parameters.projectPath }}/**/*.csproj'
              arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)/app'
              publishWebProjects: false
              zipAfterPublish: true

          - task: PublishBuildArtifacts@1
            displayName: 'Publish Artifact'
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)/app'
              artifactName: 'function-app'

          - task: CopyFiles@2
            displayName: 'Copy Bicep Files'
            inputs:
              sourceFolder: 'infra'
              contents: '**'
              targetFolder: '$(Build.ArtifactStagingDirectory)/infra'

          - task: PublishBuildArtifacts@1
            displayName: 'Publish Infra Artifact'
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)/infra'
              artifactName: 'infrastructure'
```

#### `templates/stages/deploy-infra.yml`

```yaml
parameters:
  - name: stageName
    type: string
  - name: environment
    type: string
  - name: resourceGroup
    type: string
  - name: azureSubscription
    type: string
  - name: bicepFile
    type: string
  - name: bicepParams
    type: string
  - name: dependsOn
    type: object
    default: ['Build']

stages:
  - stage: ${{ parameters.stageName }}
    displayName: 'Deploy Infrastructure (${{ parameters.environment }})'
    dependsOn: ${{ parameters.dependsOn }}
    jobs:
      - deployment: DeployInfrastructure
        displayName: 'Deploy Bicep'
        environment: ${{ parameters.environment }}
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: infrastructure

                - task: AzureCLI@2
                  displayName: 'What-If Analysis'
                  inputs:
                    azureSubscription: ${{ parameters.azureSubscription }}
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group what-if \
                        --resource-group ${{ parameters.resourceGroup }} \
                        --template-file $(Pipeline.Workspace)/infrastructure/${{ parameters.bicepFile }} \
                        --parameters $(Pipeline.Workspace)/infrastructure/${{ parameters.bicepParams }} \
                        --parameters environment=${{ parameters.environment }}

                - task: AzureCLI@2
                  displayName: 'Deploy Bicep'
                  inputs:
                    azureSubscription: ${{ parameters.azureSubscription }}
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group ${{ parameters.resourceGroup }} \
                        --template-file $(Pipeline.Workspace)/infrastructure/${{ parameters.bicepFile }} \
                        --parameters $(Pipeline.Workspace)/infrastructure/${{ parameters.bicepParams }} \
                        --parameters environment=${{ parameters.environment }}
```

#### `templates/stages/deploy-function.yml`

```yaml
parameters:
  - name: stageName
    type: string
  - name: environment
    type: string
  - name: azureSubscription
    type: string
  - name: functionAppName
    type: string
  - name: dependsOn
    type: object
    default: []

stages:
  - stage: ${{ parameters.stageName }}
    displayName: 'Deploy Function App (${{ parameters.environment }})'
    dependsOn: ${{ parameters.dependsOn }}
    jobs:
      - deployment: DeployFunctionApp
        displayName: 'Deploy to Azure Functions'
        environment: ${{ parameters.environment }}
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: function-app

                - task: AzureFunctionApp@2
                  displayName: 'Deploy Function App'
                  inputs:
                    connectedServiceNameARM: ${{ parameters.azureSubscription }}
                    appType: 'functionAppLinux'
                    appName: ${{ parameters.functionAppName }}
                    package: '$(Pipeline.Workspace)/function-app/**/*.zip'
                    deploymentMethod: 'auto'
```

## Job Templates

### `templates/jobs/bicep-validate.yml`

```yaml
parameters:
  - name: azureSubscription
    type: string
  - name: resourceGroup
    type: string
  - name: bicepFile
    type: string
    default: 'infra/main.bicep'

jobs:
  - job: ValidateBicep
    displayName: 'Validate Bicep'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - task: AzureCLI@2
        displayName: 'Bicep Build (Lint)'
        inputs:
          azureSubscription: ${{ parameters.azureSubscription }}
          scriptType: 'bash'
          scriptLocation: 'inlineScript'
          inlineScript: |
            az bicep build --file ${{ parameters.bicepFile }}

      - task: AzureCLI@2
        displayName: 'Validate Deployment'
        inputs:
          azureSubscription: ${{ parameters.azureSubscription }}
          scriptType: 'bash'
          scriptLocation: 'inlineScript'
          inlineScript: |
            az deployment group validate \
              --resource-group ${{ parameters.resourceGroup }} \
              --template-file ${{ parameters.bicepFile }}
```

## Step Templates

### `templates/steps/azure-login.yml`

```yaml
parameters:
  - name: azureSubscription
    type: string

steps:
  - task: AzureCLI@2
    displayName: 'Azure Login'
    inputs:
      azureSubscription: ${{ parameters.azureSubscription }}
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        echo "Logged in to Azure"
        az account show
```

## Environment Approvals

For production deployments, configure environment approvals in Azure DevOps:

1. Go to **Pipelines** → **Environments**
2. Create environments: `nonprod`, `prod`
3. For `prod`, add **Approvals and checks**:
   - Required approvers
   - Business hours check (optional)
   - Exclusive lock (optional)

## Service Connections

Pipelines require Azure service connections:

1. Go to **Project Settings** → **Service connections**
2. Create **Azure Resource Manager** connection
3. Use the connection name in `azureSubscription` parameter

## Pipeline Variables

### Variable Groups

Create variable groups for shared configuration:

```yaml
variables:
  - group: 'azure-common'  # Contains azureSubscription, etc.
  - name: projectName
    value: 'my-project'
```

### Secret Variables

Store secrets in variable groups or Azure Key Vault:

```yaml
variables:
  - group: 'my-project-secrets'
```

## Extending Templates

### Adding Custom Steps

Projects can add steps before/after template stages by using jobs instead of extends:

```yaml
stages:
  - stage: CustomStage
    jobs:
      - template: templates/jobs/dotnet-build.yml@templates
        parameters:
          projectPath: 'src/MyProject'
      
      - job: CustomJob
        steps:
          - script: echo "Custom step"
```

### Overriding Parameters

Templates accept parameters that can be customised:

```yaml
extends:
  template: pipelines/function-app-rest.yml@templates
  parameters:
    dotnetVersion: '9.0'  # Override default
    # ... other parameters
```

## Versioning

### Branch Strategy

- `main` - Production-ready templates
- `develop` - Work in progress
- Feature branches for changes

### Referencing Specific Versions

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/tags/v1.2.0  # Pin to specific version
```

## Testing Templates

### Local Validation

```bash
# Validate YAML syntax
az pipelines build definition show --definition-id 1 --detect true
```

### Template Repository Pipeline

The `pipeline-templates` repo has its own pipeline to validate templates:

```yaml
# azure-pipelines.yml in pipeline-templates repo
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

steps:
  - script: |
      echo "Validating YAML syntax..."
      for file in $(find . -name "*.yml" -o -name "*.yaml"); do
        echo "Checking $file"
        python -c "import yaml; yaml.safe_load(open('$file'))"
      done
    displayName: 'Validate YAML Syntax'
```

## Troubleshooting

### Common Issues

**Template not found**

- Verify repository name in `resources.repositories`
- Check `ref` points to existing branch/tag
- Ensure service account has read access to template repo

**Parameter errors**

- Verify parameter names match template definition
- Check required parameters are provided
- Ensure correct types (string, object, boolean)

**Stage dependencies**

- Ensure `dependsOn` references existing stage names
- Check for circular dependencies

## Related Documentation

- [Azure Pipelines YAML Schema](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema)
- [Template Types & Usage](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates)
- [Architecture Overview](../ARCHITECTURE.md)
