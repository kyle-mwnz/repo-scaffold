# Repository Scaffolding System

A template-based system for creating new Azure DevOps repositories with standardised infrastructure, pipelines, and Azure Functions code.

## Overview

This system enables developers to quickly scaffold new repositories with:

- **Azure Functions** (.NET isolated worker) - Generated using Azure Functions Core Tools
- **Bicep Infrastructure** - References centrally-managed modules from a Bicep registry
- **Azure Pipelines** - Extends centrally-managed pipeline templates
- **Consistent Naming** - Azure resource names derived automatically from project name

The key design principle is **centralisation of shared components** - pipeline logic and infrastructure modules are maintained once and consumed by all projects, preventing drift and ensuring consistency.

## Quick Start

### Prerequisites

- [Copier](https://copier.readthedocs.io/) v9.0+ (`pipx install copier`)
- [Azure Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local) v4+
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) with DevOps extension
- .NET SDK 8.0+

### Create a New Project

```bash
# Scaffold from Azure DevOps (replace with your org/project)
copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./

# Or from a local clone
copier copy /path/to/repo-scaffolds ./
```

You'll be prompted for:

| Prompt | Description | Example |
|--------|-------------|---------|
| Project name | Lowercase, alphanumeric with hyphens | `customer-orders` |
| Description | Short project description | `Handles order processing` |
| API type | REST, SOAP, or Event-driven | `REST API` |
| .NET version | Target framework version | `8.0` |
| Azure DevOps org | Your organisation URL | `https://dev.azure.com/yourorg` |
| Azure DevOps project | Project name | `YourProject` |
| Create repo | Auto-create Azure DevOps repo | `Yes` |

### What Gets Created

```
customer-orders/
├── .copier-answers.yml          # Template version tracking (for updates)
├── azure-pipelines.yml          # Extends central pipeline templates
├── README.md
├── infra/
│   ├── main.bicep               # References central Bicep modules
│   └── parameters/
│       ├── nonprod.bicepparam
│       └── prod.bicepparam
├── src/
│   └── CustomerOrders.Functions/
│       ├── CustomerOrders.Functions.csproj
│       ├── Program.cs
│       ├── host.json
│       ├── local.settings.json
│       └── HealthCheck.cs       # Initial function (varies by API type)
└── tests/
    └── CustomerOrders.Functions.Tests/
```

### Update an Existing Project

When the central template is updated:

```bash
cd your-project
copier update
```

This pulls the latest template changes while preserving your customisations.

## Template Types

| Type | Description | Triggers | Additional Resources |
|------|-------------|----------|---------------------|
| **REST API** | HTTP-based RESTful services | HTTP triggers | API Management integration |
| **SOAP API** | XML/WSDL web services | HTTP triggers | API Management integration |
| **Event-driven** | Message-based async processing | Service Bus triggers | Service Bus namespace & queues |

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full system design.

## Documentation

- [Architecture Overview](./ARCHITECTURE.md) - System design and component relationships
- [Developer Guide](./docs/developer-guide.md) - How to use the scaffolding system
- [Copier Template](./docs/copier-template.md) - Template configuration details
- [Pipeline Templates](./docs/pipeline-templates.md) - CI/CD pipeline structure
- [Bicep Modules](./docs/bicep-modules.md) - Infrastructure module registry
- [Naming Conventions](./docs/naming-conventions.md) - Azure resource naming standards

## Key Benefits

| Benefit | How It's Achieved |
|---------|-------------------|
| **No drift on pipelines** | Projects use `extends:` to reference central templates |
| **No drift on infrastructure** | Bicep modules versioned in ACR registry |
| **Consistent naming** | Names derived from project name via convention |
| **Easy updates** | `copier update` pulls template changes |
| **Fast onboarding** | Single command to scaffold complete project |

## Repository Structure

This repository contains:

```
repo-scaffolds/
├── copier.yml                   # Template configuration
├── {{ serviceapplication_name }}/          # Templated project files
├── scripts/                     # Post-generation scripts
├── docs/                        # Documentation
└── README.md
```

Related repositories:

| Repository | Purpose |
|------------|---------|
| `pipeline-templates` | Centrally-managed Azure Pipeline YAML templates |
| `bicep-registry` | Source for Bicep modules published to ACR |

## Contributing

To contribute to the templates:

1. Clone this repository
2. Make changes to template files
3. Test locally: `copier copy . ../test-output --trust`
4. Submit a pull request

See [docs/contributing.md](./docs/contributing.md) for detailed guidelines.

## Support

For issues or questions:

- Check the [troubleshooting guide](./docs/troubleshooting.md)
- Open an issue in this repository
- Contact the platform team
