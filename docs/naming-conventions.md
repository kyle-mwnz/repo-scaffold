# Naming Conventions

This document defines the naming conventions for Azure resources and repositories created by the scaffolding system.

## Overview

All resource names are **derived from the serviceapplication name** using consistent patterns. This ensures:

- **Predictability** - Know resource names without looking them up
- **Consistency** - All projects follow the same patterns
- **Compliance** - Names meet Azure naming rules automatically
- **Discoverability** - Easy to find related resources

## serviceapplication name Rules

The serviceapplication name is the foundation for all other names. It must:

| Rule | Requirement | Example |
|------|-------------|---------|
| Length | 3-21 characters | `customer-orders` |
| Characters | Lowercase alphanumeric and hyphens | `payment-api` |
| Start | Must start with a letter | `a-valid-name` |
| Pattern | `^[a-z][a-z0-9-]{2,20}$` | `inventory-sync` |

**Valid examples:** `customer-orders`, `payment-api`, `inventory-sync`, `order-processor`

**Invalid examples:** `CustomerOrders` (uppercase), `123-api` (starts with number), `my_api` (underscore)

## Naming Patterns

### Resource Groups

| Resource | Pattern | Example |
|----------|---------|---------|
| NonProd | `rg-{project}-nonprod-aue` | `rg-customer-orders-nonprod-aue` |
| Prod | `rg-{project}-prod-aue` | `rg-customer-orders-prod-aue` |

### Azure Functions

| Resource | Pattern | Example |
|----------|---------|---------|
| Function App | `func-{project}-{env}-aue` | `func-customer-orders-nonprod-aue` |
| App Service Plan | `asp-{project}-{env}-aue` | `asp-customer-orders-nonprod-aue` |

### Storage

| Resource | Pattern | Example | Notes |
|----------|---------|---------|-------|
| Storage Account | `st{project}{env}aue` | `stcustomerordersnonprodaue` | No hyphens, max 24 chars |

**Storage Account Constraints:**
- 3-24 characters
- Lowercase letters and numbers only (no hyphens)
- Must be globally unique

For long serviceapplication names, the storage account name may need truncation.

### .NET Projects

| Resource | Pattern | Example |
|----------|---------|---------|
| Functions Project | `{PascalCaseName}.Functions` | `CustomerOrders.Functions` |
| Test Project | `{PascalCaseName}.Functions.Tests` | `CustomerOrders.Functions.Tests` |

## Components Breakdown

### Prefix/Suffix Reference

| Abbreviation | Meaning |
|--------------|---------|
| `rg-` | Resource Group |
| `func-` | Function App |
| `asp-` | App Service Plan |
| `st` | Storage Account |
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

For serviceapplication name `customer-orders`:

| Resource | NonProd Name | Prod Name |
|----------|--------------|-----------|
| Resource Group | `rg-customer-orders-nonprod-aue` | `rg-customer-orders-prod-aue` |
| Function App | `func-customer-orders-nonprod-aue` | `func-customer-orders-prod-aue` |
| App Service Plan | `asp-customer-orders-nonprod-aue` | `asp-customer-orders-prod-aue` |
| Storage Account | `stcustomerordersnonprodaue` | `stcustomerordersprodaue` |

## Implementation in Templates

### Copier Variables

In `copier.yml`, computed variables derive names:

```yaml
serviceapplication_name:
  type: str
  validator: "{% if not (serviceapplication_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}Invalid{% endif %}"

pascal_case_name:
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

## Handling Edge Cases

### Long serviceapplication names

Storage accounts have a 24-character limit. For long serviceapplication names:

```
project: customer-order-processor (24 chars)
storage: stcustomerorderprocessorprodaue (32 chars - TOO LONG)
```

**Recommendation:** Keep serviceapplication names under 15 characters to avoid truncation.

### Global Uniqueness

Some resources require globally unique names:

| Resource | Scope | Solution |
|----------|-------|----------|
| Storage Account | Global | Include org/team prefix or hash |
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

The convention uses only characters valid for all resource types.

## Validation

### In Copier Template

```yaml
serviceapplication_name:
  type: str
  validator: >-
    {% if not (serviceapplication_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}
    serviceapplication name must be 3-21 characters, lowercase alphanumeric with hyphens, starting with a letter.
    Examples: customer-orders, payment-api, inventory-sync
    {% endif %}
```

### In Bicep

```bicep
@description('serviceapplication name')
@minLength(3)
@maxLength(21)
param projectName string

// Validate storage account name length
var storageAccountName = 'st${replace(projectName, '-', '')}${environment}aue'
// This will fail at deployment if > 24 chars
```

## Related Documentation

- [Azure Naming Rules](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules)
- [Azure Naming Conventions (CAF)](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Architecture Overview](../ARCHITECTURE.md)
