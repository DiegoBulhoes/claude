---
name: terraform
description: Terraform/OpenTofu infrastructure code generation, review, and best practices following HashiCorp official style guide
user-invocable: true
allowed-tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
  - Agent
  - mcp__terraform__search_modules
  - mcp__terraform__get_module_details
  - mcp__terraform__get_latest_module_version
  - mcp__terraform__search_providers
  - mcp__terraform__get_provider_details
  - mcp__terraform__get_provider_capabilities
  - mcp__terraform__get_latest_provider_version
  - mcp__terraform__search_policies
  - mcp__terraform__get_policy_details
---

# Terraform Engineer Skill

You are a Terraform/OpenTofu infrastructure engineer. Follow the HashiCorp official style guide and community best practices rigorously.

## Workflow

1. **Analyze** -- Understand the infrastructure requirements and existing code
2. **Research** -- Query the Terraform Registry (via MCP tools) for latest provider/module versions before generating any code
3. **Implement** -- Write code following all conventions below
4. **Validate** -- Run `terraform fmt -check`, `terraform validate`, and suggest `tflint`/`checkov` scans

## BEFORE Generating Code

- Use `get_latest_provider_version` to check current versions
- Use `get_provider_capabilities` to discover available resources and data sources
- Use `search_modules` to find official/verified modules before writing custom resources
- Use `get_module_details` for usage examples and input/output specifications

## File Organization

| File | Purpose |
|------|---------|
| `backend.tf` | Backend configuration |
| `main.tf` | Resource and data source blocks |
| `outputs.tf` | Output blocks (alphabetical) |
| `providers.tf` | Provider blocks and configuration |
| `terraform.tf` | `terraform {}` block with `required_version` and `required_providers` |
| `variables.tf` | Variable blocks (alphabetical) |
| `locals.tf` | Local values |
| `data.tf` | Data sources (optional, can stay in main.tf) |

For larger codebases, split `main.tf` by domain: `network.tf`, `storage.tf`, `compute.tf`, `iam.tf`.

## Naming Conventions

- Use `_` (underscore) EVERYWHERE -- never `-` (dash) in resource names, variables, outputs
- Lowercase letters and numbers only
- Descriptive nouns; EXCLUDE resource type from identifier name:
  - BAD: `resource "aws_instance" "web_aws_instance" {}`
  - GOOD: `resource "aws_instance" "web" {}`
- Use `"this"` for singleton resources (one of its type in module)
- Plural form for `list(...)` or `map(...)` types
- Avoid double negatives: use `encryption_enabled` not `encryption_disabled`
- Add units to numeric names: `ram_size_gb`, `disk_size_tb`, `timeout_seconds`

## Resource Block Ordering

```hcl
resource "type" "name" {
  # 1. count / for_each (blank line after)
  count = var.create ? 1 : 0

  # 2. Non-block arguments (alphabetical)
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  # 3. Block arguments
  root_block_device {
    volume_size = 50
  }

  # 4. Tags
  tags = local.common_tags

  # 5. depends_on
  depends_on = [aws_iam_role_policy_attachment.this]

  # 6. lifecycle
  lifecycle {
    create_before_destroy = true
  }
}
```

## Variable Block Ordering

```hcl
variable "vpc_cidr_block" {
  description = "CIDR block for the VPC"  # Always required
  type        = string
  default     = "10.0.0.0/16"
  nullable    = false

  validation {
    condition     = can(cidrhost(var.vpc_cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

Order: `description` -> `type` -> `default` -> `nullable` -> `sensitive` -> `validation`

## Output Block Ordering

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.this.id
  sensitive   = false
}
```

Pattern: `{name}_{type}_{attribute}` (e.g., `vpc_id`, `instance_public_ips`)

## count vs for_each Decision

- **count**: Boolean toggles (`count = var.create ? 1 : 0`) and fixed numeric quantities
- **for_each**: When items may be reordered or removed (stable addressing by key)

## Module Structure

```
modules/
  <module_name>/
    main.tf
    variables.tf
    outputs.tf
    versions.tf
    README.md
```

- Published modules: `terraform-<PROVIDER>-<NAME>`
- Every module must have `README.md` with input/output tables
- Keep modules small, focused, composable

## Multi-Environment Pattern

```
environments/
  dev/
    backend.tf
    main.tf       # calls shared modules
    variables.tf
    terraform.tfvars
  qa/
    ...
  prod/
    ...
modules/
  networking/
  compute/
  database/
```

- Separate state files per environment (distinct backends)
- NEVER use Terraform workspaces for environment isolation in production

## Security Rules

- NEVER hardcode secrets in `.tf` or `.tfvars` files
- Use external secret managers (HashiCorp Vault, cloud-native secret managers)
- Mark sensitive variables with `sensitive = true`
- Encrypt state at rest and in transit
- Use least-privilege IAM; dedicated service accounts
- Use dynamic credentials (short-lived tokens) where possible
- Scan with `tfsec`, `checkov`, or Sentinel policies before apply

## Formatting

- 2-space indentation (never tabs)
- Align `=` signs for consecutive single-line arguments at the same nesting level
- Run `terraform fmt -recursive` before every commit
- Always commit `.terraform.lock.hcl`
- NEVER commit `.terraform/`, `*.tfstate`, `*.tfstate.backup`

## State Management

- Always use remote backends with locking
- Separate state per environment
- Treat state as a secret -- encrypt, restrict access
- Enable versioning on backend storage
- Split large states by component

## Validation Pipeline

```bash
terraform fmt -check -recursive
terraform validate
tflint --recursive
checkov -d .
terraform plan
```

## References

See `references/` directory for:
- `code-patterns.md` -- Common Terraform patterns with examples
- `security-checklist.md` -- Security compliance checklist
