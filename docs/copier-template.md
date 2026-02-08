# Copier Template Configuration

This document describes the Copier template configuration used for scaffolding new repositories.

## Overview

[Copier](https://copier.readthedocs.io/) is a Python-based project templating tool that uses Jinja2 for variable substitution. It was chosen for this system because it supports:

- Rich variable validation
- Post-generation tasks (shell commands)
- Template updates for existing projects
- Conditional file inclusion

## Installation

```bash
# Using pipx (recommended)
pipx install copier

# Using pip
pip install copier

# Verify installation
copier --version
```

## Configuration File: `copier.yml`

The `copier.yml` file defines all template variables, validation rules, and post-generation tasks.

### Full Configuration

```yaml
# Copier template configuration for Azure Functions repository scaffolding
_min_copier_version: "9.0.0"
_subdirectory: "{{ serviceapplication_name }}"

# ============================================
# Questions (User Prompts)
# ============================================

serviceapplication_name:
  type: str
  help: |
    Project name (lowercase, alphanumeric, hyphens only).
    This will be used to generate all Azure resource names.
    Examples: customer-orders, payment-processor, inventory-sync
  validator: >-
    {% if not (serviceapplication_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}
    Must be 3-21 characters, lowercase alphanumeric with hyphens, starting with a letter
    {% endif %}

project_description:
  type: str
  help: Short description of the project (used in README and Azure tags)
  default: "Azure Functions service"

api_type:
  type: str
  help: What type of API/service is this?
  choices:
    REST API (HTTP triggers, OpenAPI): rest
    SOAP API (HTTP triggers, XML/WSDL): soap
    Event-driven (Service Bus triggers): event

dotnet_version:
  type: str
  help: .NET version for Azure Functions
  default: "8.0"
  choices:
    - "8.0"
    - "9.0"

azure_devops_org:
  type: str
  help: |
    Azure DevOps organization URL.
    Example: https://dev.azure.com/yourorg

azure_devops_project:
  type: str
  help: Azure DevOps project name where the repository will be created

create_repo:
  type: bool
  help: Create Azure DevOps repository automatically?
  default: true

# ============================================
# Computed Variables (Hidden from User)
# ============================================

_function_app_name:
  type: str
  default: "{{ serviceapplication_name | replace('-', '') }}"
  when: false

_pascal_case_name:
  type: str
  default: "{{ serviceapplication_name | replace('-', ' ') | title | replace(' ', '') }}"
  when: false

_storage_account_name:
  type: str
  default: "st{{ serviceapplication_name | replace('-', '') }}"
  when: false

# ============================================
# Post-Generation Tasks
# ============================================

_tasks:
  # Initialize .NET Azure Functions project
  - >-
    cd {{ serviceapplication_name }}/src &&
    func init {{ _pascal_case_name }}.Functions
    --worker-runtime dotnet-isolated
    --target-framework net{{ dotnet_version }}

  # Create initial function based on API type
  - >-
    cd {{ serviceapplication_name }}/src/{{ _pascal_case_name }}.Functions &&
    {% if api_type == 'rest' %}
    func new --name HealthCheck --template "HTTP trigger" --authlevel anonymous
    {% elif api_type == 'soap' %}
    func new --name SoapEndpoint --template "HTTP trigger" --authlevel function
    {% elif api_type == 'event' %}
    func new --name ProcessMessage --template "Azure Service Bus Queue trigger"
    {% endif %}

  # Create test project
  - >-
    cd {{ serviceapplication_name }}/tests &&
    dotnet new xunit -n {{ _pascal_case_name }}.Functions.Tests &&
    cd {{ _pascal_case_name }}.Functions.Tests &&
    dotnet add reference ../../src/{{ _pascal_case_name }}.Functions/{{ _pascal_case_name }}.Functions.csproj

  # Initialize git repository
  - "cd {{ serviceapplication_name }} && git init && git add . && git commit -m 'Initial scaffold from template'"

  # Create Azure DevOps repo (if requested)
  - >-
    {% if create_repo %}
    az repos create
    --name {{ serviceapplication_name }}-service
    --org {{ azure_devops_org }}
    --project "{{ azure_devops_project }}"
    --output none
    {% else %}
    echo "Skipping Azure DevOps repo creation"
    {% endif %}

  # Add remote and push (if repo created)
  - >-
    {% if create_repo %}
    cd {{ serviceapplication_name }} &&
    git remote add origin {{ azure_devops_org }}/{{ azure_devops_project }}/_git/{{ serviceapplication_name }}-service &&
    git push -u origin main
    {% else %}
    echo "Skipping git remote setup"
    {% endif %}

# ============================================
# File Handling
# ============================================

_exclude:
  - "copier.yml"
  - "copier.yaml"
  - "*.md"
  - ".git"
  - "__pycache__"
  - "*.pyc"
  - "scripts/"
  - "docs/"

# Conditional exclusions based on API type
_skip_if_exists:
  - ".copier-answers.yml"
```

## Variable Types

### String Variables

```yaml
serviceapplication_name:
  type: str
  help: Project name
  default: "my-project"  # Optional default value
```

### Choice Variables

```yaml
api_type:
  type: str
  choices:
    Display Name 1: value1
    Display Name 2: value2
```

Or as a simple list:

```yaml
dotnet_version:
  type: str
  choices:
    - "8.0"
    - "9.0"
```

### Boolean Variables

```yaml
create_repo:
  type: bool
  default: true
```

### Computed Variables

Variables calculated from other inputs (hidden from user):

```yaml
_pascal_case_name:
  type: str
  default: "{{ serviceapplication_name | replace('-', ' ') | title | replace(' ', '') }}"
  when: false  # Hides from user prompts
```

## Validation

Use the `validator` field with Jinja2 conditionals:

```yaml
serviceapplication_name:
  type: str
  validator: >-
    {% if not (serviceapplication_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}
    Error message shown to user
    {% endif %}
```

If the validator outputs any text, validation fails with that message.

## Jinja2 Filters

Useful filters for transforming variables:

| Filter | Input | Output |
|--------|-------|--------|
| `lower` | `MyProject` | `myproject` |
| `upper` | `MyProject` | `MYPROJECT` |
| `title` | `my project` | `My Project` |
| `replace('-', '')` | `my-project` | `myproject` |
| `replace('-', ' ') \| title \| replace(' ', '')` | `my-project` | `MyProject` |

## Template Files

### File Naming

Template files can include variables in their names:

```
{{ serviceapplication_name }}/
├── {{ serviceapplication_name }}.sln
└── src/
    └── {{ _pascal_case_name }}.Functions/
```

### File Extensions

- `.jinja` - Processed through Jinja2, extension removed
- No extension - Copied as-is

Example:
- `azure-pipelines.yml.jinja` → `azure-pipelines.yml`
- `.gitignore` → `.gitignore`

### Conditional Content

Use Jinja2 conditionals within template files:

```yaml
# In azure-pipelines.yml.jinja
{% if api_type == 'event' %}
- stage: DeployServiceBus
  # ...
{% endif %}
```

### Conditional Files

Exclude files based on conditions in `copier.yml`:

```yaml
_exclude:
  - "{% if api_type != 'event' %}**/service-bus-*{% endif %}"
```

## Post-Generation Tasks

Tasks run after file generation:

```yaml
_tasks:
  - "echo 'Running task 1'"
  - "cd {{ serviceapplication_name }} && npm install"
  - >-
    {% if some_condition %}
    conditional command
    {% endif %}
```

### Task Features

- Run in order, sequentially
- Have access to all template variables
- Support Jinja2 conditionals
- Run from the output directory

### Common Tasks

```yaml
_tasks:
  # Initialize Azure Functions
  - "func init ProjectName --worker-runtime dotnet-isolated"
  
  # Create .NET project
  - "dotnet new webapi -n ProjectName"
  
  # Install dependencies
  - "dotnet restore"
  
  # Initialize git
  - "git init && git add . && git commit -m 'Initial commit'"
  
  # Create Azure DevOps repo
  - "az repos create --name repo-name --org https://dev.azure.com/org --project project"
```

## Template Directory Structure

```
repo-scaffolds/
├── copier.yml                           # Configuration
├── {{ serviceapplication_name }}/                  # Templated output folder
│   ├── .gitignore
│   ├── README.md.jinja
│   ├── azure-pipelines.yml.jinja
│   ├── infra/
│   │   ├── main.bicep.jinja
│   │   └── parameters/
│   │       ├── nonprod.bicepparam.jinja
│   │       └── prod.bicepparam.jinja
│   ├── src/
│   │   └── .gitkeep
│   └── tests/
│       └── .gitkeep
├── scripts/                             # Excluded from output
│   └── helpers.sh
└── docs/                                # Excluded from output
```

## Usage

### Create New Project

```bash
# From Azure DevOps git repository (HTTPS)
copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./

# From local path
copier copy /path/to/repo-scaffolds ./

# With pre-filled answers
copier copy repo-scaffolds ./ --data serviceapplication_name=my-api --data api_type=rest

# Non-interactive (requires all answers)
copier copy repo-scaffolds ./ --data-file answers.yml
```

### Update Existing Project

```bash
cd existing-project
copier update

# Or update from specific version
copier update --vcs-ref v2.0.0
```

### Answers File

When a project is scaffolded, Copier creates `.copier-answers.yml`:

```yaml
# This file was auto-generated by Copier
_commit: abc123
_src_path: https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds
serviceapplication_name: customer-orders
project_description: Handles customer order processing
api_type: rest
dotnet_version: "8.0"
azure_devops_org: https://dev.azure.com/yourorg
azure_devops_project: YourProject
create_repo: true
```

This file:
- Tracks which template version was used
- Stores answers for future updates
- Should be committed to the repository

## Testing Templates

### Local Testing

```bash
# Test template generation
copier copy . ../test-output --trust

# Test with specific answers
copier copy . ../test-output --trust --data serviceapplication_name=test-api

# Overwrite existing test output
copier copy . ../test-output --trust --overwrite
```

### Validation

```bash
# Validate copier.yml syntax
copier copy . /dev/null --pretend
```

## Troubleshooting

### Common Issues

**Task fails with "command not found"**

Ensure required tools are installed:
- `func` (Azure Functions Core Tools)
- `dotnet` (.NET SDK)
- `az` (Azure CLI)

**Variables not substituted**

- Check file has `.jinja` extension (or is in a templated directory)
- Verify variable names match exactly

**Validation always fails**

- Ensure validator outputs nothing on success
- Check Jinja2 syntax in validator

### Debug Mode

```bash
copier copy . ../test-output --trust --vcs-ref HEAD -v
```

## Related Documentation

- [Copier Official Docs](https://copier.readthedocs.io/)
- [Jinja2 Template Designer](https://jinja.palletsprojects.com/en/3.1.x/templates/)
- [Architecture Overview](../ARCHITECTURE.md)
