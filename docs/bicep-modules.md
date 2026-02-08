# Bicep Module Registry

This document describes the centrally-managed Bicep modules stored in Azure Container Registry (ACR) that scaffolded projects reference.

## Overview

Bicep modules are reusable infrastructure components published to an Azure Container Registry. Projects reference these modules using the `br:` (Bicep Registry) syntax, ensuring:

- **No drift** - Infrastructure definitions are maintained centrally
- **Versioning** - Modules are versioned for controlled rollouts
- **Consistency** - All projects use the same infrastructure patterns
- **Compliance** - Security and compliance configurations are standardised

## Repository Structure

```
bicep-registry/
├── azure-pipelines.yml              # Pipeline to publish modules to ACR
├── README.md
├── modules/
│   ├── function-app/
│   │   ├── main.bicep
│   │   ├── README.md
│   │   └── test/
│   │       └── main.test.bicep
│   ├── app-insights/
│   │   ├── main.bicep
│   │   └── README.md
│   ├── storage-account/
│   │   ├── main.bicep
│   │   └── README.md
│   ├── service-bus/
│   │   ├── main.bicep
│   │   └── README.md
│   ├── key-vault/
│   │   ├── main.bicep
│   │   └── README.md
│   └── apim-api/
│       ├── main.bicep
│       └── README.md
└── compositions/
    ├── rest-api-stack.bicep
    ├── soap-api-stack.bicep
    └── event-driven-stack.bicep
```

## Setting Up the Registry

### Create Azure Container Registry

```bash
# Create resource group for shared infrastructure
az group create \
  --name rg-shared-infra-prod-aue \
  --location australiaeast

# Create ACR with Premium SKU (required for Bicep modules)
az acr create \
  --name yourorgbicepregistry \
  --resource-group rg-shared-infra-prod-aue \
  --location australiaeast \
  --sku Premium \
  --admin-enabled false
```

### Configure Bicep to Use Registry

Create `bicepconfig.json` in your repository root:

```json
{
  "moduleAliases": {
    "br": {
      "myregistry": {
        "registry": "yourorgbicepregistry.azurecr.io",
        "modulePath": "bicep"
      }
    }
  }
}
```

## Module Definitions

### Function App Module

`modules/function-app/main.bicep`:

```bicep
@description('Name of the Function App')
param name string

@description('Location for resources')
param location string = resourceGroup().location

@description('Name of the associated Storage Account')
param storageAccountName string

@description('Name of the Application Insights instance')
param appInsightsName string

@description('.NET version')
@allowed(['8.0', '9.0'])
param dotnetVersion string = '8.0'

@description('Function worker runtime')
@allowed(['dotnet-isolated', 'dotnet'])
param functionWorkerRuntime string = 'dotnet-isolated'

@description('App Service Plan SKU')
param skuName string = 'Y1'

@description('Tags to apply to resources')
param tags object = {}

// App Service Plan (Consumption)
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-${name}'
  location: location
  tags: tags
  sku: {
    name: skuName
    tier: skuName == 'Y1' ? 'Dynamic' : 'Standard'
  }
  properties: {
    reserved: true  // Linux
  }
}

// Reference existing storage account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

// Reference existing App Insights
resource appInsights 'Microsoft.Insights/components@2020-02-02' existing = {
  name: appInsightsName
}

// Function App
resource functionApp 'Microsoft.Web/sites@2023-01-01' = {
  name: name
  location: location
  tags: tags
  kind: 'functionapp,linux'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'DOTNET-ISOLATED|${dotnetVersion}'
      ftpsState: 'Disabled'
      minTlsVersion: '1.2'
      http20Enabled: true
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTSHARE'
          value: toLower(name)
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: functionWorkerRuntime
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString
        }
      ]
    }
  }
}

// Outputs
output id string = functionApp.id
output name string = functionApp.name
output defaultHostName string = functionApp.properties.defaultHostName
output principalId string = functionApp.identity.principalId
```

### Storage Account Module

`modules/storage-account/main.bicep`:

```bicep
@description('Name of the Storage Account (3-24 chars, lowercase alphanumeric)')
@minLength(3)
@maxLength(24)
param name string

@description('Location for resources')
param location string = resourceGroup().location

@description('Storage account SKU')
@allowed(['Standard_LRS', 'Standard_GRS', 'Standard_ZRS'])
param skuName string = 'Standard_LRS'

@description('Tags to apply to resources')
param tags object = {}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: name
  location: location
  tags: tags
  sku: {
    name: skuName
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    networkAcls: {
      defaultAction: 'Allow'
      bypass: 'AzureServices'
    }
  }
}

output id string = storageAccount.id
output name string = storageAccount.name
output primaryEndpoints object = storageAccount.properties.primaryEndpoints
```

### Application Insights Module

`modules/app-insights/main.bicep`:

```bicep
@description('Name of the Application Insights instance')
param name string

@description('Location for resources')
param location string = resourceGroup().location

@description('Log Analytics Workspace ID (optional)')
param workspaceId string = ''

@description('Tags to apply to resources')
param tags object = {}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: name
  location: location
  tags: tags
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: !empty(workspaceId) ? workspaceId : null
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

output id string = appInsights.id
output name string = appInsights.name
output connectionString string = appInsights.properties.ConnectionString
output instrumentationKey string = appInsights.properties.InstrumentationKey
```

### Service Bus Module

`modules/service-bus/main.bicep`:

```bicep
@description('Name of the Service Bus namespace')
param name string

@description('Location for resources')
param location string = resourceGroup().location

@description('Service Bus SKU')
@allowed(['Basic', 'Standard', 'Premium'])
param skuName string = 'Standard'

@description('Queues to create')
param queues array = []

@description('Topics to create')
param topics array = []

@description('Tags to apply to resources')
param tags object = {}

resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: name
  location: location
  tags: tags
  sku: {
    name: skuName
    tier: skuName
  }
  properties: {
    minimumTlsVersion: '1.2'
  }
}

// Create queues
resource serviceBusQueues 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = [for queue in queues: {
  parent: serviceBusNamespace
  name: queue.name
  properties: {
    maxDeliveryCount: contains(queue, 'maxDeliveryCount') ? queue.maxDeliveryCount : 10
    deadLetteringOnMessageExpiration: contains(queue, 'deadLettering') ? queue.deadLettering : true
    lockDuration: 'PT1M'
  }
}]

// Create topics (Standard/Premium only)
resource serviceBusTopics 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = [for topic in topics: if (skuName != 'Basic') {
  parent: serviceBusNamespace
  name: topic.name
  properties: {
    maxSizeInMegabytes: 1024
  }
}]

output id string = serviceBusNamespace.id
output name string = serviceBusNamespace.name
output namespaceName string = serviceBusNamespace.name
output endpoint string = serviceBusNamespace.properties.serviceBusEndpoint
```

### Key Vault Module

`modules/key-vault/main.bicep`:

```bicep
@description('Name of the Key Vault')
param name string

@description('Location for resources')
param location string = resourceGroup().location

@description('Tenant ID for RBAC')
param tenantId string = subscription().tenantId

@description('Enable soft delete')
param enableSoftDelete bool = true

@description('Soft delete retention days')
@minValue(7)
@maxValue(90)
param softDeleteRetentionDays int = 90

@description('Tags to apply to resources')
param tags object = {}

resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: name
  location: location
  tags: tags
  properties: {
    tenantId: tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    enableRbacAuthorization: true
    enableSoftDelete: enableSoftDelete
    softDeleteRetentionInDays: softDeleteRetentionDays
    enabledForDeployment: false
    enabledForDiskEncryption: false
    enabledForTemplateDeployment: true
    networkAcls: {
      defaultAction: 'Allow'
      bypass: 'AzureServices'
    }
  }
}

output id string = keyVault.id
output name string = keyVault.name
output vaultUri string = keyVault.properties.vaultUri
```

## Publishing Modules

### Pipeline: `azure-pipelines.yml`

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - modules/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  acrName: 'yourorgbicepregistry'
  azureSubscription: 'your-service-connection'

stages:
  - stage: Validate
    displayName: 'Validate Modules'
    jobs:
      - job: ValidateBicep
        displayName: 'Validate Bicep Syntax'
        steps:
          - task: AzureCLI@2
            displayName: 'Bicep Build (All Modules)'
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                for module in modules/*/main.bicep; do
                  echo "Validating $module..."
                  az bicep build --file "$module"
                done

  - stage: Publish
    displayName: 'Publish to Registry'
    dependsOn: Validate
    jobs:
      - job: PublishModules
        displayName: 'Publish Bicep Modules'
        steps:
          - task: AzureCLI@2
            displayName: 'Publish Modules to ACR'
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Get version from build number or git tag
                VERSION="${BUILD_BUILDNUMBER:-1.0.$(date +%Y%m%d)}"
                
                for module_dir in modules/*/; do
                  module_name=$(basename "$module_dir")
                  module_file="${module_dir}main.bicep"
                  
                  if [ -f "$module_file" ]; then
                    echo "Publishing $module_name version $VERSION..."
                    
                    # Publish with version tag
                    az bicep publish \
                      --file "$module_file" \
                      --target "br:$(acrName).azurecr.io/bicep/${module_name}:${VERSION}"
                    
                    # Also publish as latest
                    az bicep publish \
                      --file "$module_file" \
                      --target "br:$(acrName).azurecr.io/bicep/${module_name}:latest"
                    
                    echo "Published $module_name"
                  fi
                done
```

### Manual Publishing

```bash
# Login to Azure
az login

# Publish a specific module
az bicep publish \
  --file modules/function-app/main.bicep \
  --target br:yourorgbicepregistry.azurecr.io/bicep/function-app:1.0.0

# Publish as latest
az bicep publish \
  --file modules/function-app/main.bicep \
  --target br:yourorgbicepregistry.azurecr.io/bicep/function-app:latest
```

## Using Modules

### In Scaffolded Projects

```bicep
// infra/main.bicep
targetScope = 'resourceGroup'

param environment string
param location string = 'australiaeast'

var projectName = 'customer-orders'
var nameSuffix = '${projectName}-${environment}-aue'

// Storage Account
module storageAccount 'br:yourorgbicepregistry.azurecr.io/bicep/storage-account:latest' = {
  name: 'deploy-storage'
  params: {
    name: 'st${replace(projectName, '-', '')}${environment}aue'
    location: location
    tags: {
      project: projectName
      environment: environment
    }
  }
}

// Application Insights
module appInsights 'br:yourorgbicepregistry.azurecr.io/bicep/app-insights:latest' = {
  name: 'deploy-appinsights'
  params: {
    name: 'appi-${nameSuffix}'
    location: location
    tags: {
      project: projectName
      environment: environment
    }
  }
}

// Function App
module functionApp 'br:yourorgbicepregistry.azurecr.io/bicep/function-app:latest' = {
  name: 'deploy-functionapp'
  params: {
    name: 'func-${nameSuffix}'
    location: location
    storageAccountName: storageAccount.outputs.name
    appInsightsName: appInsights.outputs.name
    dotnetVersion: '8.0'
    tags: {
      project: projectName
      environment: environment
    }
  }
  dependsOn: [
    storageAccount
    appInsights
  ]
}

output functionAppName string = functionApp.outputs.name
output functionAppUrl string = functionApp.outputs.defaultHostName
```

### Using Aliases

With `bicepconfig.json`:

```json
{
  "moduleAliases": {
    "br": {
      "modules": {
        "registry": "yourorgbicepregistry.azurecr.io",
        "modulePath": "bicep"
      }
    }
  }
}
```

Reference modules with shorter syntax:

```bicep
module functionApp 'br/modules:function-app:latest' = {
  // ...
}
```

## Versioning Strategy

### Semantic Versioning

- **Major** (1.0.0 → 2.0.0): Breaking changes (parameter removal, behaviour changes)
- **Minor** (1.0.0 → 1.1.0): New features, new parameters with defaults
- **Patch** (1.0.0 → 1.0.1): Bug fixes, documentation updates

### Version Tags

| Tag | Usage | When Updated |
|-----|-------|--------------|
| `latest` | Development, non-critical environments | Every merge to main |
| `1.2.3` | Production, pinned versions | On release |
| `1.2` | Auto-update patches within minor | On release |

### Referencing Versions

```bicep
// Always latest (auto-updates)
module app 'br:acr/bicep/function-app:latest' = { }

// Specific version (pinned)
module app 'br:acr/bicep/function-app:1.2.3' = { }

// Minor version (gets patch updates)
module app 'br:acr/bicep/function-app:1.2' = { }
```

## Testing Modules

### Unit Tests with Bicep Test Framework

`modules/function-app/test/main.test.bicep`:

```bicep
// Test file for function-app module
targetScope = 'resourceGroup'

module testFunctionApp '../main.bicep' = {
  name: 'test-function-app'
  params: {
    name: 'func-test-001'
    storageAccountName: 'sttest001'
    appInsightsName: 'appi-test-001'
    location: 'australiaeast'
    dotnetVersion: '8.0'
  }
}
```

### Running Tests

```bash
# Validate module compiles
az bicep build --file modules/function-app/main.bicep

# Validate against Azure (requires existing resource group)
az deployment group validate \
  --resource-group rg-test \
  --template-file modules/function-app/test/main.test.bicep

# What-if deployment
az deployment group what-if \
  --resource-group rg-test \
  --template-file modules/function-app/test/main.test.bicep
```

## Listing Published Modules

```bash
# List all repositories (modules) in ACR
az acr repository list --name yourorgbicepregistry

# List tags for a specific module
az acr repository show-tags \
  --name yourorgbicepregistry \
  --repository bicep/function-app

# Get module manifest
az acr manifest show \
  --registry yourorgbicepregistry \
  --name bicep/function-app:latest
```

## Access Control

### ACR Permissions

| Role | Permissions | Who |
|------|-------------|-----|
| AcrPush | Push modules | CI/CD pipelines |
| AcrPull | Pull modules | All developers, deployment pipelines |
| AcrDelete | Delete modules | Administrators only |

### Granting Access

```bash
# Get ACR resource ID
ACR_ID=$(az acr show --name yourorgbicepregistry --query id -o tsv)

# Grant pull access to a service principal
az role assignment create \
  --assignee <service-principal-id> \
  --role AcrPull \
  --scope $ACR_ID

# Grant pull access to Azure DevOps service connection
az role assignment create \
  --assignee <service-connection-object-id> \
  --role AcrPull \
  --scope $ACR_ID
```

## Troubleshooting

### Common Issues

**Module not found**

```
Error: Unable to restore module br:acr/bicep/function-app:1.0
```

- Verify ACR name and module path
- Check module version exists: `az acr repository show-tags`
- Ensure you're logged in: `az acr login --name yourorgbicepregistry`

**Access denied**

```
Error: UNAUTHORIZED: authentication required
```

- Verify service connection has AcrPull role
- Check if using correct subscription
- Regenerate credentials if needed

**Version mismatch**

- Clear local Bicep cache: `rm -rf ~/.bicep`
- Restore modules: `az bicep restore --file main.bicep`

## Related Documentation

- [Bicep Registry Documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/private-module-registry)
- [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/)
- [Architecture Overview](../ARCHITECTURE.md)
