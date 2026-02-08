# Developer Guide

This guide walks you through using the repository scaffolding system to create new projects and manage them throughout their lifecycle.

## Prerequisites

Before you begin, ensure you have the following installed:

| Tool | Version | Installation |
|------|---------|--------------|
| Copier | 9.0+ | `pipx install copier` |
| Azure Functions Core Tools | 4.x | [Install Guide](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local#install-the-azure-functions-core-tools) |
| .NET SDK | 8.0+ | [Download](https://dotnet.microsoft.com/download) |
| Azure CLI | Latest | [Install Guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) |
| Git | Latest | [Download](https://git-scm.com/downloads) |

### Verify Installation

```bash
# Check Copier
copier --version

# Check Azure Functions Core Tools
func --version

# Check .NET SDK
dotnet --version

# Check Azure CLI
az --version

# Login to Azure (required for repo creation)
az login
az extension add --name azure-devops
az devops configure --defaults organization=https://dev.azure.com/yourorg project=YourProject
```

## Creating a New Project

### Step 1: Run the Scaffold Command

```bash
# From any directory where you want to create the project
copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./
```

Or if you have the template cloned locally:

```bash
copier copy /path/to/repo-scaffolds ./
```

### Step 2: Answer the Prompts

You'll be asked a series of questions:

```
🎤 Project name (lowercase, alphanumeric, hyphens only):
> customer-orders

🎤 Short description of the project:
> Handles customer order processing and notifications

🎤 What type of API/service is this?
  (1) REST API (HTTP triggers, OpenAPI)
  (2) SOAP API (HTTP triggers, XML/WSDL)
  (3) Event-driven (Service Bus triggers)
> 1

🎤 .NET version for Azure Functions:
  (1) 8.0
  (2) 9.0
> 1

🎤 Azure DevOps organization URL:
> https://dev.azure.com/yourorg

🎤 Azure DevOps project name:
> YourProject

🎤 Create Azure DevOps repository automatically? [Y/n]:
> Y
```

### Step 3: Wait for Generation

Copier will:

1. Generate project files from templates
2. Run `func init` to create the Azure Functions project
3. Run `func new` to create the initial function
4. Create a test project
5. Initialize git repository
6. Create Azure DevOps repository (if selected)
7. Push initial commit

```
Copying from template
    create  customer-orders/
    create  customer-orders/.gitignore
    create  customer-orders/README.md
    create  customer-orders/azure-pipelines.yml
    create  customer-orders/infra/
    ...

Running task 1 of 6: func init CustomerOrders.Functions...
Running task 2 of 6: func new --name HealthCheck...
Running task 3 of 6: dotnet new xunit...
Running task 4 of 6: git init...
Running task 5 of 6: az repos create...
Running task 6 of 6: git push...

Done! Created project at ./customer-orders
```

### Step 4: Open Your Project

```bash
cd customer-orders
code .  # Or your preferred editor
```

## Project Structure

After scaffolding, your project will have this structure:

```
customer-orders/
├── .copier-answers.yml          # Template tracking (commit this!)
├── .gitignore
├── README.md
├── azure-pipelines.yml          # CI/CD pipeline configuration
├── infra/
│   ├── main.bicep               # Infrastructure definition
│   └── parameters/
│       ├── nonprod.bicepparam   # NonProd environment parameters
│       └── prod.bicepparam      # Prod environment parameters
├── src/
│   └── CustomerOrders.Functions/
│       ├── CustomerOrders.Functions.csproj
│       ├── Program.cs
│       ├── host.json
│       ├── local.settings.json
│       └── HealthCheck.cs       # Initial function
└── tests/
    └── CustomerOrders.Functions.Tests/
        └── CustomerOrders.Functions.Tests.csproj
```

## Local Development

### Running Locally

```bash
cd src/CustomerOrders.Functions

# Restore dependencies
dotnet restore

# Run the function app
func start
```

The function will be available at `http://localhost:7071/api/HealthCheck`.

### Adding New Functions

```bash
cd src/CustomerOrders.Functions

# HTTP trigger
func new --name GetOrders --template "HTTP trigger"

# Service Bus trigger (for event-driven projects)
func new --name ProcessOrder --template "Azure Service Bus Queue trigger"

# Timer trigger
func new --name DailyReport --template "Timer trigger"
```

### Running Tests

```bash
cd tests/CustomerOrders.Functions.Tests
dotnet test
```

### Local Settings

Edit `src/CustomerOrders.Functions/local.settings.json` for local configuration:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated"
  }
}
```

**Important:** Never commit `local.settings.json` to source control (it's in `.gitignore`).

## Working with Infrastructure

### Understanding the Bicep Files

`infra/main.bicep` defines your Azure resources. It references centrally-managed modules:

```bicep
// References a shared module from the Bicep registry
module functionApp 'br:yourorgbicepregistry.azurecr.io/bicep/function-app:latest' = {
  name: 'deploy-functionapp'
  params: {
    name: 'func-customer-orders-${environment}-aue'
    // ...
  }
}
```

### Modifying Infrastructure

To add or modify resources:

1. Edit `infra/main.bicep`
2. Use existing modules from the registry where possible
3. For custom resources, add them directly to the Bicep file

Example - adding a custom resource:

```bicep
// Custom resource not in the central registry
resource customResource 'Microsoft.SomeProvider/resources@2023-01-01' = {
  name: 'custom-${nameSuffix}'
  location: location
  properties: {
    // ...
  }
}
```

### Environment Parameters

Each environment has its own parameter file:

`infra/parameters/nonprod.bicepparam`:
```bicep
using '../main.bicep'

param environment = 'nonprod'
```

`infra/parameters/prod.bicepparam`:
```bicep
using '../main.bicep'

param environment = 'prod'
```

### Testing Infrastructure Changes

```bash
# Validate Bicep syntax
az bicep build --file infra/main.bicep

# What-if deployment (see what would change)
az deployment group what-if \
  --resource-group rg-customer-orders-nonprod-aue \
  --template-file infra/main.bicep \
  --parameters infra/parameters/nonprod.bicepparam
```

## Working with Pipelines

### Pipeline Structure

Your `azure-pipelines.yml` extends central templates:

```yaml
extends:
  template: pipelines/function-app-rest.yml@templates
  parameters:
    projectName: $(projectName)
    # ...
```

This means:
- Build and deploy logic is managed centrally
- You only configure project-specific parameters
- Updates to central templates apply automatically

### Triggering Pipelines

Pipelines trigger automatically on:
- Push to `main` branch
- Pull request to `main` branch

To run manually:
1. Go to Azure DevOps → Pipelines
2. Select your pipeline
3. Click "Run pipeline"

### Pipeline Stages

| Stage | Description | Trigger |
|-------|-------------|---------|
| Build | Compile, test, package | Every commit |
| Infrastructure_nonprod | Deploy Bicep to nonprod | After Build |
| Deploy_nonprod | Deploy function to nonprod | After Infrastructure |
| Infrastructure_prod | Deploy Bicep to prod | After nonprod (with approval) |
| Deploy_prod | Deploy function to prod | After Infrastructure_prod |

### Viewing Pipeline Results

1. Go to Azure DevOps → Pipelines → your-pipeline
2. Click on a run to see stages
3. Click on a stage to see jobs
4. Click on a job to see step logs

## Updating Your Project

### When Templates Change

When the central scaffolding template is updated, you can pull those changes:

```bash
cd customer-orders
copier update
```

Copier will:
1. Show what files would change
2. Ask for confirmation
3. Apply changes, preserving your customisations

**What gets updated:**
- `azure-pipelines.yml` structure
- `infra/main.bicep` module references
- Configuration files

**What's preserved:**
- Your application code
- Custom resources you've added
- Project-specific configuration

### Handling Conflicts

If Copier detects conflicts:

```
Conflict in azure-pipelines.yml
  (o) Overwrite with new version
  (s) Skip (keep your version)
  (d) Show diff
  (e) Edit merged version
> d
```

Review the diff and choose how to resolve.

### Skipping Updates

To skip updating specific files:

```bash
copier update --skip azure-pipelines.yml
```

## Troubleshooting

### Scaffold Fails: "func not found"

Install Azure Functions Core Tools:

```bash
# macOS
brew install azure-functions-core-tools@4

# Windows
npm install -g azure-functions-core-tools@4 --unsafe-perm true
```

### Scaffold Fails: "az repos create failed"

1. Check you're logged in: `az login`
2. Check DevOps extension: `az extension add --name azure-devops`
3. Check permissions in Azure DevOps project

### Pipeline Fails: "Template not found"

1. Verify `pipeline-templates` repository exists
2. Check repository name in `resources.repositories`
3. Ensure service account has read access

### Bicep Fails: "Module not found"

1. Check ACR name in module reference
2. Verify module version exists: `az acr repository show-tags --name yourorgbicepregistry --repository bicep/function-app`
3. Check service connection has ACR pull permissions

### Local Run Fails: "Storage emulator not found"

Install and start Azurite:

```bash
# Install
npm install -g azurite

# Start
azurite --silent --location ./azurite --debug ./azurite/debug.log
```

Or use a real storage account in `local.settings.json`.

## Best Practices

### Code Organisation

```
src/CustomerOrders.Functions/
├── Functions/           # Function classes (HTTP triggers, etc.)
│   ├── HealthCheck.cs
│   └── OrderFunctions.cs
├── Services/            # Business logic
│   └── OrderService.cs
├── Models/              # DTOs, entities
│   └── Order.cs
├── Program.cs           # Dependency injection setup
└── host.json            # Function host configuration
```

### Dependency Injection

Use the built-in DI in `Program.cs`:

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(services =>
    {
        services.AddScoped<IOrderService, OrderService>();
        services.AddHttpClient();
    })
    .Build();

host.Run();
```

### Configuration

Use configuration providers:

```csharp
// In Program.cs
.ConfigureAppConfiguration(config =>
{
    config.AddEnvironmentVariables();
    config.AddAzureKeyVault(/* ... */);
})
```

### Testing

Write unit tests for business logic:

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task ProcessOrder_ValidOrder_ReturnsSuccess()
    {
        // Arrange
        var service = new OrderService();
        var order = new Order { Id = 1, Amount = 100 };

        // Act
        var result = await service.ProcessAsync(order);

        // Assert
        Assert.True(result.Success);
    }
}
```

### Security

- Never commit secrets to source control
- Use Azure Key Vault for sensitive configuration
- Use managed identities for Azure service authentication
- Review OWASP guidelines for function security

## Getting Help

- **Template issues:** Open an issue in the `repo-scaffolds` repository
- **Pipeline issues:** Check the `pipeline-templates` repository
- **Infrastructure issues:** Check the `bicep-registry` repository
- **General questions:** Contact the platform team

## Quick Reference

| Task | Command |
|------|---------|
| Create new project | `copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./` |
| Update project | `copier update` |
| Run locally | `func start` |
| Add function | `func new --name Name --template "Template"` |
| Run tests | `dotnet test` |
| Validate Bicep | `az bicep build --file infra/main.bicep` |
| What-if deployment | `az deployment group what-if ...` |

## Related Documentation

- [Architecture Overview](../ARCHITECTURE.md)
- [Copier Template](./copier-template.md)
- [Pipeline Templates](./pipeline-templates.md)
- [Bicep Modules](./bicep-modules.md)
- [Naming Conventions](./naming-conventions.md)
