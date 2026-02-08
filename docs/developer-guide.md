# Developer Guide

This guide walks you through using the repository scaffolding system to create new projects and manage them throughout their lifecycle.

## Prerequisites

Before you begin, ensure you have the following installed:

| Tool | Version | Installation |
|------|---------|--------------|
| Copier | 9.0+ | `pipx install copier` |
| Azure Functions Core Tools | 4.x | [Install Guide](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local#install-the-azure-functions-core-tools) |
| .NET SDK | 10.0+ | [Download](https://dotnet.microsoft.com/download) |
| Git | Latest | [Download](https://git-scm.com/downloads) |
| Git Credential Manager | Latest | [Download](https://github.com/git-ecosystem/git-credential-manager/releases) |

### Verify Installation

```bash
# Check Copier
copier --version

# Check Azure Functions Core Tools
func --version

# Check .NET SDK
dotnet --version

# Check Git Credential Manager
git credential-manager --version
```

## Azure DevOps Authentication

Copier uses `git clone` to fetch the template repository. For Azure DevOps HTTPS URLs, you need to configure authentication.

### Option 1: Git Credential Manager (Recommended)

Git Credential Manager (GCM) handles authentication automatically and securely. On first use, it will open a browser window for you to authenticate with Azure DevOps.

**Installation:**

```bash
# macOS
brew install git-credential-manager

# Windows (included with Git for Windows)
# Already installed if you have Git for Windows

# Linux
# Download from https://github.com/git-ecosystem/git-credential-manager/releases
```

**Configuration:**

```bash
# Configure git to use GCM (usually automatic)
git config --global credential.helper manager

# For Azure DevOps specifically
git config --global credential.https://dev.azure.com.useHttpPath true
```

**First Use:**

When you run `copier copy https://dev.azure.com/...`, GCM will:
1. Open a browser window
2. Prompt you to sign in to Azure DevOps
3. Cache your credentials securely for future use

### Option 2: Personal Access Token (PAT)

If you can't use Git Credential Manager, create a Personal Access Token:

1. Go to Azure DevOps > User Settings > Personal Access Tokens
2. Click "New Token"
3. Give it a name (e.g., "Copier CLI")
4. Set expiration as needed
5. Select scope: **Code (Read)**
6. Click "Create" and copy the token

**Using the PAT:**

```bash
# Option A: Configure git credential store (will prompt once, then remember)
git config --global credential.helper store

# Then run copier - when prompted:
# - Username: anything (e.g., your email)
# - Password: paste your PAT
copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./

# Option B: Embed in URL (less secure, avoid in shared scripts)
copier copy https://PAT@dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./
```

### Troubleshooting Authentication

**"Repository not found" or "Authentication failed"**

1. Verify the URL is correct:
   ```
   https://dev.azure.com/{org}/{project}/_git/{repo-name}
   ```

2. Check you have access to the repository in Azure DevOps

3. Clear cached credentials and try again:
   ```bash
   # Clear GCM cache for Azure DevOps
   git credential-manager erase
   # Enter: protocol=https
   # Enter: host=dev.azure.com
   # Press Enter twice
   ```

4. If using PAT, verify it hasn't expired and has **Code (Read)** scope

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

You'll be asked only **2 questions**:

```
🎤 Project name (lowercase, alphanumeric, hyphens only):
> customer-orders

🎤 What type of API/service is this?
  (1) REST API (HTTP triggers, OpenAPI)
  (2) SOAP API (HTTP triggers, XML/WSDL)
  (3) Event-driven (Service Bus triggers)
> 1
```

### Step 3: Wait for Generation

Copier will:

1. Generate project files from templates
2. Run `func init` to create the Azure Functions project (.NET 10 isolated)
3. Run `func new` to create the initial function(s)
4. Create a test project with xUnit
5. Initialize git repository with initial commit

```
Copying from template
    create  customer-orders/
    create  customer-orders/.gitignore
    create  customer-orders/README.md
    create  customer-orders/azure-pipelines.yml
    create  customer-orders/infra/
    ...

Running task 1 of 4: func init CustomerOrders.Functions...
Running task 2 of 4: func new --name HealthCheck...
Running task 3 of 4: dotnet new xunit...
Running task 4 of 4: git init...

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
├── azure-pipelines.yml          # Standalone build pipeline
├── infra/
│   ├── main.bicep               # Hello world infrastructure
│   └── parameters/
│       ├── nonprod.bicepparam   # NonProd environment parameters
│       └── prod.bicepparam      # Prod environment parameters
├── src/
│   └── CustomerOrders.Functions/
│       ├── CustomerOrders.Functions.csproj
│       ├── Program.cs
│       ├── host.json
│       ├── local.settings.json
│       ├── HealthCheck.cs       # Health check function
│       └── CustomerOrdersApi.cs # Main API function (REST) or other based on api_type
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

`infra/main.bicep` defines your Azure resources as a simple "hello world" stack:

- **Storage Account** - Required for Azure Functions
- **App Service Plan** - Consumption plan (Y1)
- **Function App** - Linux, .NET 10 isolated worker

```bicep
// Example resource naming
var functionAppName = 'func-${projectName}-${environment}-aue'
var storageAccountName = 'st${replace(projectName, '-', '')}${environment}aue'
```

### Modifying Infrastructure

To add or modify resources, edit `infra/main.bicep` directly:

```bicep
// Add a custom resource
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

## Working with the Pipeline

### Pipeline Structure

Your `azure-pipelines.yml` is a standalone build pipeline that:

1. Builds the .NET project
2. Runs tests
3. Publishes build artifacts

**Note:** This is a simplified "hello world" pipeline. Deployment stages can be added as needed.

### Triggering Pipelines

To run the pipeline:
1. Push your code to Azure DevOps
2. Go to Azure DevOps > Pipelines
3. Create a new pipeline pointing to your repository
4. Select the existing `azure-pipelines.yml`

### Customizing the Pipeline

Edit `azure-pipelines.yml` to add deployment stages, additional tests, or other steps as needed.

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
- `infra/main.bicep` template changes
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

## Troubleshooting

### Scaffold Fails: "Repository not found" or Authentication Error

See [Azure DevOps Authentication](#azure-devops-authentication) above. Common fixes:

```bash
# Clear cached credentials
git credential-manager erase
# Enter: protocol=https
# Enter: host=dev.azure.com
# Press Enter twice

# Then try again
copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./
```

### Scaffold Fails: "func not found"

Install Azure Functions Core Tools:

```bash
# macOS
brew install azure-functions-core-tools@4

# Windows
npm install -g azure-functions-core-tools@4 --unsafe-perm true
```

### Scaffold Fails: "dotnet not found"

Install .NET SDK 10.0 or later from [dotnet.microsoft.com](https://dotnet.microsoft.com/download).

### Local Run Fails: "Storage emulator not found"

Install and start Azurite:

```bash
# Install
npm install -g azurite

# Start
azurite --silent --location ./azurite --debug ./azurite/debug.log
```

Or use a real storage account in `local.settings.json`.

### Bicep Validation Fails

```bash
# Check Bicep CLI is installed
az bicep version

# Upgrade Bicep
az bicep upgrade
```

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

## Quick Reference

| Task | Command |
|------|---------|
| Create new project | `copier copy https://dev.azure.com/.../repo-scaffolds ./` |
| Update project | `copier update` |
| Run locally | `func start` |
| Add function | `func new --name Name --template "Template"` |
| Run tests | `dotnet test` |
| Validate Bicep | `az bicep build --file infra/main.bicep` |
| What-if deployment | `az deployment group what-if ...` |

## Related Documentation

- [Architecture Overview](../ARCHITECTURE.md)
- [Copier Template](./copier-template.md)
- [Naming Conventions](./naming-conventions.md)
