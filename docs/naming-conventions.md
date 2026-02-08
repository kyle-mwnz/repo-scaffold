# Naming Conventions

This document defines the naming conventions for Azure resources and repositories created by the scaffolding system.

## Overview

All resource names are **derived from the project name** using consistent patterns. This ensures:

- **Predictability** - Know resource names without looking them up
- **Consistency** - All projects follow the same patterns
- **Compliance** - Names meet Azure naming rules automatically
- **Discoverability** - Easy to find related resources

## Project Name Rules

The project name is the foundation for all other names. It must:

| Rule | Requirement | Example |
|------|-------------|---------|
| Length | 3-21 characters | `customer-orders` |
| Characters | Lowercase alphanumeric and hyphens | `payment-api` |
| Start | Must start with a letter | `a-valid-name` |
| Pattern | `^[a-z][a-z0-9-]{2,20}$` | `inventory-sync` |

**Valid examples:** `customer-orders`, `payment-api`, `inventory-sync`, `order-processor`

**Invalid examples:** `CustomerOrders` (uppercase), `123-api` (starts with number), `my_api` (underscore)

## Naming Patterns

### Repositories

| Resource | Pattern | Example |
|----------|---------|---------|
| Azure DevOps Repo | `{project}-service` | `customer-orders-service` |

### Resource Groups

| Resource | Pattern | Example |
|----------|---------|---------|
| NonProd | `rg-{project}-nonprod-aue` | `rg-customer-orders-nonprod-aue` |
| Prod | `rg-{project}-prod-aue` | `rg-customer-orders-prod-aue` |

### Azure Functions

| Resource | Pattern | Example |
|----------|---------|---------|
| Function App | `func-{project}-{env}-aue` | `func-customer-orders-nonprod-aue` |
| App Service Plan | `asp-func-{project}-{env}-aue` | `asp-func-customer-orders-nonprod-aue` |

### Storage

| Resource | Pattern | Example | Notes |
|----------|---------|---------|-------|
| Storage Account | `st{project}{env}aue` | `stcustomerordersnonprodaue` | No hyphens, max 24 chars |

**Storage Account Constraints:**
- 3-24 characters
- Lowercase letters and numbers only (no hyphens)
- Must be globally unique

For long project names, the storage account name is truncated:
- `customer-orders` → `stcustomerordersnonprodaue` (26 chars - will be truncated)
- Solution: Use shorter project names or implement hashing

### Monitoring

| Resource | Pattern | Example |
|----------|---------|---------|
| Application Insights | `appi-{project}-{env}-aue` | `appi-customer-orders-nonprod-aue` |
| Log Analytics Workspace | `log-{project}-{env}-aue` | `log-customer-orders-nonprod-aue` |

### Messaging (Event-driven)

| Resource | Pattern | Example |
|----------|---------|---------|
| Service Bus Namespace | `sb-{project}-{env}-aue` | `sb-customer-orders-nonprod-aue` |
| Service Bus Queue | `{queue-name}` | `incoming-orders` |
| Service Bus Topic | `{topic-name}` | `order-events` |

### Security

| Resource | Pattern | Example |
|----------|---------|---------|
| Key Vault | `kv-{project}-{env}-aue` | `kv-customer-orders-nonprod-aue` |

**Key Vault Constraints:**
- 3-24 characters
- Alphanumerics and hyphens
- Must start with a letter
- Must be globally unique

### API Management (REST/SOAP)

| Resource | Pattern | Example |
|----------|---------|---------|
| APIM API | `{project}-api` | `customer-orders-api` |

## Components Breakdown

### Prefix/Suffix Reference

| Abbreviation | Meaning |
|--------------|---------|
| `rg-` | Resource Group |
| `func-` | Function App |
| `asp-` | App Service Plan |
| `st` | Storage Account |
| `appi-` | Application Insights |
| `log-` | Log Analytics |
| `sb-` | Service Bus |
| `kv-` | Key Vault |
| `-aue` | Australia East region |

### Environment Codes

| Code | Environment |
|------|-------------|
| `nonprod` | Non-production (dev, test, UAT) |
| `prod` | Production |

### Region Codes

| Code | Region |
|------|--------|
| `aue` | Australia East |
| `aus` | Australia Southeast |
| `sea` | Southeast Asia |

## Complete Example

For project name `customer-orders`:

| Resource | NonProd Name | Prod Name |
|----------|--------------|-----------|
| Repository | `customer-orders-service` | - |
| Resource Group | `rg-customer-orders-nonprod-aue` | `rg-customer-orders-prod-aue` |
| Function App | `func-customer-orders-nonprod-aue` | `func-customer-orders-prod-aue` |
| Storage Account | `stcustomerordersnonprodaue` | `stcustomerordersprodaue` |
| App Insights | `appi-customer-orders-nonprod-aue` | `appi-customer-orders-prod-aue` |
| Key Vault | `kv-customer-orders-nonprod-aue` | `kv-customer-orders-prod-aue` |
| Service Bus | `sb-customer-orders-nonprod-aue` | `sb-customer-orders-prod-aue` |

## Implementation in Templates

### Copier Variables

In `copier.yml`, computed variables derive names:

```yaml
serviceapplication_name:
  type: str
  validator: "{% if not (serviceapplication_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}Invalid{% endif %}"

_storage_name_base:
  type: str
  default: "{{ serviceapplication_name | replace('-', '') }}"
  when: false

_pascal_case:
  type: str
  default: "{{ serviceapplication_name | replace('-', ' ') | title | replace(' ', '') }}"
  when: false
```

### Bicep Variables

In `main.bicep.jinja`:

```bicep
var projectName = '{{ serviceapplication_name }}'
var nameSuffix = '${projectName}-${environment}-aue'
var storageNameSuffix = '${replace(projectName, '-', '')}${environment}aue'

// Usage
var functionAppName = 'func-${nameSuffix}'
var storageAccountName = 'st${storageNameSuffix}'
```

### Pipeline Variables

In `azure-pipelines.yml.jinja`:

```yaml
variables:
  projectName: '{{ serviceapplication_name }}'
  
# Used in parameters
environments:
  - name: nonprod
    resourceGroup: 'rg-{{ serviceapplication_name }}-nonprod-aue'
  - name: prod
    resourceGroup: 'rg-{{ serviceapplication_name }}-prod-aue'
```

## Handling Edge Cases

### Long Project Names

Storage accounts have a 24-character limit. For long project names:

**Option 1: Truncation (current approach)**
```
project: customer-order-processor (24 chars)
storage: stcustomerorderprocessorprodaue (32 chars - TOO LONG)
truncated: stcustomerorderproc...aue (24 chars)
```

**Option 2: Hashing**
```bicep
var storageBase = replace(projectName, '-', '')
var storageHash = substring(uniqueString(projectName), 0, 6)
var storageName = 'st${take(storageBase, 10)}${storageHash}${environment}aue'
```

**Recommendation:** Keep project names under 15 characters to avoid truncation.

### Global Uniqueness

Some resources require globally unique names:

| Resource | Scope | Solution |
|----------|-------|----------|
| Storage Account | Global | Include org/team prefix or hash |
| Key Vault | Global | Include org/team prefix or hash |
| Function App | Global | Include org/team prefix |

If collisions occur, add an org prefix:

```
func-{org}-{project}-{env}-{region}
func-contoso-customer-orders-nonprod-aue
```

### Special Characters

Different resources have different character restrictions:

| Resource | Allowed Characters |
|----------|-------------------|
| Resource Group | Alphanumeric, hyphens, underscores, periods, parentheses |
| Function App | Alphanumeric, hyphens |
| Storage Account | **Lowercase** alphanumeric only |
| Key Vault | Alphanumeric, hyphens |

The convention uses only characters valid for all resource types.

## Validation

### In Copier Template

```yaml
serviceapplication_name:
  type: str
  validator: >-
    {% if not (serviceapplication_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}
    Project name must be 3-21 characters, lowercase alphanumeric with hyphens, starting with a letter.
    Examples: customer-orders, payment-api, inventory-sync
    {% endif %}
```

### In Bicep

```bicep
@description('Project name')
@minLength(3)
@maxLength(21)
param projectName string

// Validate storage account name length
var storageAccountName = 'st${replace(projectName, '-', '')}${environment}aue'
// This will fail at deployment if > 24 chars
```

### In Pipeline

```yaml
- script: |
    if [[ ! "$(projectName)" =~ ^[a-z][a-z0-9-]{2,20}$ ]]; then
      echo "Invalid project name format"
      exit 1
    fi
  displayName: 'Validate Project Name'
```

## Tagging Convention

All resources include standard tags:

| Tag | Value | Example |
|-----|-------|---------|
| `project` | Project name | `customer-orders` |
| `environment` | Environment code | `nonprod` |
| `managed-by` | How resource is managed | `bicep` |
| `repository` | Source repository | `customer-orders-service` |

```bicep
var commonTags = {
  project: projectName
  environment: environment
  'managed-by': 'bicep'
  repository: '${projectName}-service'
}
```

## Migration Guide

If you have existing resources that don't follow these conventions:

1. **Inventory** - List current resource names
2. **Map** - Create mapping from old to new names
3. **Plan** - Determine migration approach:
   - Rename in place (if supported)
   - Create new, migrate data, delete old
4. **Execute** - Migrate during maintenance window
5. **Update** - Update references in code and config

**Note:** Storage accounts and Key Vaults cannot be renamed. You must create new resources and migrate data.

## Related Documentation

- [Azure Naming Rules](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules)
- [Azure Naming Conventions (CAF)](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Architecture Overview](../ARCHITECTURE.md)
