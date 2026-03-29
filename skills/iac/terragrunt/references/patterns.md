# Terragrunt Patterns Guide

## Repository Separation Pattern

### Modules in Separate Repos (Recommended)

Consider maintaining modules in **separate Git repositories** rather than in the catalog:

```
# Module repos (separate per module)
github.com/YOUR_ORG/modules/database.git
github.com/YOUR_ORG/modules/kubernetes.git
github.com/YOUR_ORG/modules/vpc.git

# Catalog repo (units and stacks only)
github.com/YOUR_ORG/infrastructure-catalog.git
  └── units/
  └── stacks/

# Live repos (deployments)
github.com/YOUR_ORG/infrastructure-prod.git
github.com/YOUR_ORG/infrastructure-platform.git
```

**Trade-off:** Requires more initial development effort to set up, but each module gets a proper development workflow.

**Benefits:**
- Independent semantic versioning per module
- Dedicated CI/CD pipeline with automated testing (Terratest)
- Pre-commit hooks for formatting, validation, and security scanning
- Auto-generated documentation (terraform-docs)
- Clear team ownership boundaries
- Isolated blast radius for changes
- Explicit dependency management via version refs

**Pre-commit Hooks for Modules:**

Each module repo can have dedicated quality gates:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.81.0
    hooks:
      - id: terraform_fmt
        args:
          - --args=-no-color
          - --args=-diff
          - --args=-write=true
      - id: terraform_validate
        args:
          - --args=-json
          - --args=-no-color
  - repo: https://github.com/bridgecrewio/checkov.git
    rev: 2.3.334
    hooks:
      - id: checkov
        files: \.tf$
  - repo: https://github.com/terraform-docs/terraform-docs
    rev: "v0.16.0"
    hooks:
      - id: terraform-docs-go
        args: ["markdown", "--output-file", "../README.md", "./app"]
```

**Semantic Versioning with semantic-release:**

Automate module releases with conventional commits:

```yaml
# .releaserc.yml
ci: true
branches:
  - main
  - master
verifyConditions:
  - "@semantic-release/changelog"
  - "@semantic-release/git"
  - "@semantic-release/gitlab"  # or @semantic-release/github
analyzeCommits:
  - path: "@semantic-release/commit-analyzer"
prepare:
  - path: "@semantic-release/changelog"
    changelogFile: app/CHANGELOG.md
  - path: "@semantic-release/git"
    message: "RELEASE: ${nextRelease.version}"
    assets: ["app/CHANGELOG.md"]
publish:
  - "@semantic-release/gitlab"  # or @semantic-release/github
success: false
fail: false
npmPublish: false
```

This enables:
- Automatic version bumps based on commit messages (`feat:`, `fix:`, `BREAKING CHANGE:`)
- Auto-generated changelogs
- Git tags for each release (e.g., `v1.2.0`)
- Units reference specific versions: `?ref=v1.2.0`

### Modules in Catalog (Alternative - Monorepo)

Alternatively, keep modules in the same repository as the catalog. This is the pattern used by [Gruntwork's terragrunt-infrastructure-catalog-example](https://github.com/gruntwork-io/terragrunt-infrastructure-catalog-example).

```
infrastructure-catalog/
├── modules/                    # OpenTofu/Terraform modules
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── kubernetes/
│   └── object-storage/
├── units/                      # Terragrunt units (wrap modules)
│   ├── database/
│   │   └── terragrunt.hcl
│   └── kubernetes/
│       └── terragrunt.hcl
└── stacks/                     # Terragrunt stacks
    └── backend-api/
        └── terragrunt.stack.hcl
```

**Units reference local modules:**

```hcl
# units/database/terragrunt.hcl
terraform {
  source = "${get_repo_root()}/modules/database"
}
```

**Trade-offs:**

| Monorepo (modules in catalog) | Multi-repo (separate module repos) |
|-------------------------------|-------------------------------------|
| Easier global changes across codebase | Independent versioning per module |
| Single CI/CD pipeline | Dedicated CI/CD per module with Terratest |
| Unified testing with Terratest | Clear team ownership boundaries |
| Simpler initial setup | Pre-commit hooks per module |
| Catalog version = module version | Isolated blast radius for changes |

**When to use monorepo:**
- Smaller teams with shared ownership
- Rapid iteration during early development
- Simpler governance requirements

**When to use multi-repo:**
- Large teams with distinct ownership
- Strict versioning and release processes
- Enterprise environments with compliance requirements

### Boilerplate Modules Pattern

A hybrid approach uses `/modules` for **catalog discovery** (scaffolding) while units reference **external modules**:

```
infrastructure-catalog/
├── modules/                    # Boilerplate templates (for `terragrunt catalog`)
│   └── database/
│       ├── main.tf             # Variable definitions for discovery
│       └── .boilerplate/
│           ├── boilerplate.yml # Interactive prompts
│           └── terragrunt.hcl  # Template → external module
├── units/                      # Actual Terragrunt configs
│   └── database/
│       └── terragrunt.hcl      # References external module
└── stacks/
```

**Boilerplate template:**

```yaml
# modules/database/.boilerplate/boilerplate.yml
variables:
  - name: ModuleVersion
    description: "Module version to use"
    type: string
    default: "v1.0.0"
  - name: InstanceType
    description: "Database instance type"
    type: string
    default: "db.t3.medium"
```

```hcl
# modules/database/.boilerplate/terragrunt.hcl
terraform {
  source = "git::git@github.com:YOUR_ORG/modules/database.git//app?ref={{ .ModuleVersion }}"
}

inputs = {
  instance_type = "{{ .InstanceType }}"
}
```

This enables `terragrunt catalog` to show available modules with interactive scaffolding while keeping actual module code in separate repos.

### Unit Source Pattern

Units reference external module repos via Git URL:

```hcl
terraform {
  source = "git::git@github.com:YOUR_ORG/modules/database.git//app?ref=${values.version}"
}
```

**Or reference public modules:**

```hcl
terraform {
  # Using public community modules
  source = "git::https://github.com/terraform-modules/terraform-aws-vpc.git?ref=${try(values.version, "v3.6.0")}"
}
```

Key points:
- `//app` - Path within the module repo (submodule)
- `?ref=${values.version}` - Version comes from stack's values
- Can mix public and private modules in the same catalog

## Values Pattern

Units receive ALL configuration through the `values` object:

```hcl
# In unit terragrunt.hcl
inputs = {
  # Required - must be provided
  name = values.name

  # Optional - provide defaults with try()
  instance_type = try(values.instance_type, "db.t3.medium")

  # Auto-detect from config presence
  create_feature = try(values.create_feature, length(try(values.feature_config, {})) > 0)
}
```

Stacks pass values to units:

```hcl
# In terragrunt.stack.hcl
unit "service" {
  source = "git::...//units/service?ref=main"
  path   = "service"
  values = {
    version       = "v1.0.0"
    name          = "my-service"
    instance_type = "db.t3.large"
  }
}
```

## Reference Resolution Pattern

Units resolve symbolic references like `"../vault"` to actual dependency outputs:

```hcl
# Stack provides simple reference
unit "database" {
  values = {
    vault_secret_id = "../vault"
  }
}

# Unit resolves to actual resource ID
dependency "vault" {
  config_path = try(values.vault_path, "../vault")
  mock_outputs = {
    secret_ids = {
      db_password = "mock-secret-id"
    }
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  vault_secret_id = try(values.vault_secret_id, "") == "../vault" ?
    dependency.vault.outputs.secret_ids["db_password"] :
    values.vault_secret_id
}
```

Common references: `"../vault"`, `"../vpc"`, `"../kubernetes"`, `"../dns"`

## Dependency Path Override Pattern

Always allow path overrides for flexible composition:

```hcl
dependency "vpc" {
  config_path  = try(values.vpc_path, "../vpc")  # Override or default
  skip_outputs = !try(values.use_vpc, false)      # Conditional

  mock_outputs = {
    vpc_id = "mock-vpc-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

## Optional Dependencies Pattern

Use `skip_outputs` or `enabled` for conditional dependencies:

```hcl
dependency "vault" {
  enabled      = try(values.use_vault, false)
  config_path  = try(values.vault_path, "../vault")
  mock_outputs = { vault_id = "mock-vault-id" }
}

# Or with skip_outputs
dependency "dns" {
  config_path  = try(values.dns_path, "../dns")
  skip_outputs = !try(values.use_dns, false)
  mock_outputs = { zone_id = "mock-zone-id" }
}
```

## Cross-Account Provider Pattern

For resources in different accounts (e.g., DNS in a shared account):

```hcl
# In unit
locals {
  use_shared_account = try(values.shared_account_id, null) != null
  shared_account_id  = try(values.shared_account_id, "")
}

# Some providers support assuming roles or using aliases for cross-account access.
# Configure an additional provider alias or pass the target account ID as an input:

inputs = {
  account_id = local.use_shared_account ? local.shared_account_id : values.account_id
}
```

## Environment State Isolation

Each environment gets its own state bucket via `prefix`:

```hcl
# env.hcl
locals {
  environment = "dev"
  prefix      = "dev"
}
```

Results in:
- `terragrunt-state-acme-dev`
- `terragrunt-state-acme-qa`
- `terragrunt-state-acme-prod`

## Git URL Syntax

**CRITICAL: The refspec goes AFTER the double slash, not before**

```hcl
# CORRECT
source = "git::git@github.com:org/repo.git//units/dns?ref=main"

# WRONG - will cause "git refspec" error
source = "git::git@github.com:org/repo.git?ref=main//units/dns"
```

## Two Version Pattern

Stacks manage two different versions:

1. **Catalog ref** - Git branch/tag for the catalog (in source URL)
2. **Module version** - Git branch/tag for the underlying module (passed via values)

```hcl
unit "database" {
  # Catalog version - where to get the unit from
  source = "git::...//units/database?ref=main"

  values = {
    # Module version - what version of the database module to use
    version = "v3.0.1"
  }
}
```

## Auto-Detection Pattern

Auto-detect features from configuration presence:

```hcl
inputs = {
  # If feature_config is provided and non-empty, create the feature
  create_feature = try(values.create_feature, length(keys(try(values.feature_config, {}))) > 0)

  # Check if any record uses a reference
  use_vpc = try(
    anytrue([
      for record in try(values.records, []) :
        try(record.vpc_id == "../vpc", false)
    ]),
    false
  )
}
```

## Common Pitfalls

1. **Git refspec error:** Use `//path?ref=branch` NOT `?ref=branch//path`
2. **Heredoc in ternary:** Wrap in parentheses: `condition ? (\n<<-EOF\n...\nEOF\n) : ""`
3. **Duplicate dependencies:** Each dependency block should appear only once
4. **Missing mock outputs:** Always provide for plan/validate commands
5. **Hardcoded local paths:** Use local paths only for testing, never in committed code
