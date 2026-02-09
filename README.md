# Repository Scaffolding System

A template-based system for creating new Azure DevOps repositories with Azure Functions and Bicep infrastructure.

## Quick Start

### Prerequisites

- [Copier](https://copier.readthedocs.io/) v9.0+ (`pipx install copier`)
- [Azure Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local) v4+
- [.NET SDK 10.0](https://dotnet.microsoft.com/download)

### Create a New Project

```bash
copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./
```

You'll be prompted for:

| Prompt | Description | Example |
|--------|-------------|---------|
| serviceapplication name | Lowercase, alphanumeric with hyphens | `customer-orders` |
| API type | REST, SOAP, or Event-driven | `REST API` |

### What Gets Created

```
customer-orders/
├── .copier-answers.yml          # Template tracking (for updates)
├── .gitignore
├── azure-pipelines.yml          # CI/CD pipeline
├── README.md
├── infra/
│   ├── main.bicep               # Azure infrastructure
│   └── parameters/
│       ├── nonprod.bicepparam
│       └── prod.bicepparam
├── src/
│   └── CustomerOrders.Functions/
│       ├── CustomerOrders.Functions.csproj
│       ├── Program.cs
│       ├── host.json
│       └── HealthCheck.cs
└── tests/
    └── CustomerOrders.Functions.Tests/
```

## Template Types

| Type | Description | Initial Functions |
|------|-------------|-------------------|
| **REST API** | HTTP-based RESTful services | HealthCheck, {ProjectName}Api |
| **SOAP API** | XML/WSDL web services | HealthCheck, SoapEndpoint |
| **Event-driven** | Message-based async processing | HealthCheck, ProcessMessage |

## Azure Resources

The Bicep template creates:

| Resource | Naming Pattern |
|----------|----------------|
| Resource Group | `rg-{project}-{env}-aue` |
| Function App | `func-{project}-{env}-aue` |
| App Service Plan | `asp-func-{project}-{env}-aue` |
| Storage Account | `st{project}{env}aue` |

## Documentation

- [Developer Guide](./docs/developer-guide.md) - How to use scaffolded projects
- [Copier Template](./docs/copier-template.md) - Template configuration details
- [Naming Conventions](./docs/naming-conventions.md) - Azure resource naming standards

## Updating a Project

When the template is updated, pull changes to existing projects:

```bash
cd your-project
copier update
```

## Repository Structure

```
repo-scaffolds/
├── copier.yml                   # Template configuration
├── project/
│   └── {{ serviceapplication_name }}/      # Templated project files
├── docs/                        # Documentation
└── README.md
```

## Testing Locally

```bash
copier copy . ../test-output --trust
```
