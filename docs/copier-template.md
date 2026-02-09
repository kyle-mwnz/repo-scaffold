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

### Current Configuration

```yaml
# Copier template configuration for Azure Functions repository scaffolding
_min_copier_version: "9.0.0"
_subdirectory: "project"
_answers_file: .copier-answers.yml

# ============================================
# Questions (User Prompts) - Only 2 inputs!
# ============================================

project_name:
  type: str
  help: |
    Project name (lowercase, alphanumeric, hyphens only).
    This will be used to generate all Azure resource names.
    Examples: customer-orders, payment-processor, inventory-sync
  validator: >-
    {% if not (project_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}
    Must be 3-21 characters, lowercase alphanumeric with hyphens, starting with a letter
    {% endif %}

api_type:
  type: str
  help: What type of API/service is this?
  choices:
    REST API (HTTP triggers, OpenAPI): rest
    SOAP API (HTTP triggers, XML/WSDL): soap
    Event-driven (Service Bus triggers): event

# ============================================
# Computed Variables (Hidden from User)
# ============================================

# Note: Variables without _ prefix so they can be used in tasks
dotnet_version:
  type: str
  default: "10.0"
  when: false

pascal_case_name:
  type: str
  default: "{{ project_name | replace('-', ' ') | title | replace(' ', '') }}"
  when: false

# ============================================
# Post-Generation Tasks
# ============================================

_tasks:
  # Initialize .NET Azure Functions project
  - >-
    cd {{ project_name }} &&
    cd src &&
    func init {{ pascal_case_name }}.Functions
    --worker-runtime dotnet-isolated
    --target-framework net{{ dotnet_version }}

  # Create initial function based on API type
  - >-
    cd {{ project_name }}/src/{{ pascal_case_name }}.Functions &&
    {% if api_type == 'rest' %}
    func new --name HealthCheck --template "HTTP trigger" --authlevel anonymous &&
    func new --name {{ pascal_case_name }}Api --template "HTTP trigger" --authlevel function
    {% elif api_type == 'soap' %}
    func new --name HealthCheck --template "HTTP trigger" --authlevel anonymous &&
    func new --name SoapEndpoint --template "HTTP trigger" --authlevel function
    {% elif api_type == 'event' %}
    func new --name HealthCheck --template "HTTP trigger" --authlevel anonymous &&
    func new --name ProcessMessage --template "Azure Service Bus Queue trigger"
    {% endif %}

  # Create test project
  - >-
    cd {{ project_name }}/tests &&
    dotnet new xunit -n {{ pascal_case_name }}.Functions.Tests &&
    cd {{ pascal_case_name }}.Functions.Tests &&
    dotnet add reference ../../src/{{ pascal_case_name }}.Functions/{{ pascal_case_name }}.Functions.csproj

  # Initialize git repository
  - "cd {{ project_name }} && git init && git add . && git commit -m 'Initial scaffold from repo-scaffolds template'"

# ============================================
# File Handling
# ============================================

_exclude:
  - "copier.yml"
  - "copier.yaml"
  - "docs"
  - "ARCHITECTURE.md"
  - "README.md"
  - ".git"
  - "__pycache__"
  - "*.pyc"
  - "bicep-modules"
  - "pipeline-templates"
```

## User Inputs

The template requires only **2 user inputs**:

| Input | Type | Validation | Example |
|-------|------|------------|---------|
| `project_name` | string | 3-21 chars, lowercase alphanumeric with hyphens, starts with letter | `customer-orders` |
| `api_type` | choice | REST API, SOAP API, or Event-driven | `rest` |

### Project Name Validation

```yaml
project_name:
  validator: >-
    {% if not (project_name | regex_search('^[a-z][a-z0-9-]{2,20}$')) %}
    Must be 3-21 characters, lowercase alphanumeric with hyphens, starting with a letter
    {% endif %}
```

If the validator outputs any text, validation fails with that message.

### API Type Choices

```yaml
api_type:
  choices:
    REST API (HTTP triggers, OpenAPI): rest
    SOAP API (HTTP triggers, XML/WSDL): soap
    Event-driven (Service Bus triggers): event
```

## Computed Variables

Variables calculated from inputs (hidden from user):

| Variable | Value | Purpose |
|----------|-------|---------|
| `dotnet_version` | `"10.0"` | Hardcoded .NET version |
| `pascal_case_name` | `customer-orders` → `CustomerOrders` | .NET project naming |

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
project/
└── {{ project_name }}/
    ├── README.md.jinja
    └── azure-pipelines.yml.jinja
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
# Event-driven specific configuration
{% endif %}
```

## Post-Generation Tasks

Tasks run after file generation:

```yaml
_tasks:
  - "func init ProjectName --worker-runtime dotnet-isolated"
  - "dotnet new xunit -n ProjectName.Tests"
  - "git init && git add . && git commit -m 'Initial commit'"
```

### Task Features

- Run in order, sequentially
- Have access to all template variables
- Support Jinja2 conditionals
- Run from the output directory

## Template Directory Structure

```
repo-scaffolds/
├── copier.yml                           # Configuration
├── README.md
├── ARCHITECTURE.md
├── docs/
│   ├── copier-template.md               # This file
│   ├── developer-guide.md
│   └── naming-conventions.md
└── project/                             # Template subdirectory
    └── {{ project_name }}/              # Templated output folder
        ├── .gitignore
        ├── README.md.jinja
        ├── azure-pipelines.yml.jinja
        ├── infra/
        │   ├── main.bicep.jinja
        │   └── parameters/
        │       ├── nonprod.bicepparam.jinja
        │       └── prod.bicepparam.jinja
        ├── src/
        │   └── .gitkeep
        └── tests/
            └── .gitkeep
```

## Usage

### Create New Project

```bash
# From Azure DevOps git repository (HTTPS)
copier copy https://dev.azure.com/yourorg/yourproject/_git/repo-scaffolds ./

# From local path
copier copy /path/to/repo-scaffolds ./

# With pre-filled answers
copier copy repo-scaffolds ./ --data project_name=my-api --data api_type=rest
```

### Interactive Prompts

```
🎤 Project name (lowercase, alphanumeric, hyphens only):
> customer-orders

🎤 What type of API/service is this?
  (1) REST API (HTTP triggers, OpenAPI)
  (2) SOAP API (HTTP triggers, XML/WSDL)
  (3) Event-driven (Service Bus triggers)
> 1
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
project_name: customer-orders
api_type: rest
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
copier copy . ../test-output --trust --data project_name=test-api --data api_type=rest

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
- `func` (Azure Functions Core Tools v4)
- `dotnet` (.NET SDK 10.0+)

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
