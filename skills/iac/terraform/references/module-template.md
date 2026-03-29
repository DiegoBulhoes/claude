# Module Template

Copy-paste skeleton for creating new Terraform modules.

## File: main.tf

```hcl
terraform {
  required_providers {
    # Replace with the provider your module requires
    # AWS example:
    #   aws = { source = "hashicorp/aws" }
    # Azure example:
    #   azurerm = { source = "hashicorp/azurerm" }
    # GCP example:
    #   google = { source = "hashicorp/google" }
    <provider_alias> = {
      source = "<provider_namespace>/<provider_name>"
    }
  }
}

resource "<provider>_RESOURCE_TYPE" "this" {
  name = var.name

  tags = var.tags

  lifecycle {
    ignore_changes = [
      # Ignore provider-managed tags that are auto-populated
      # Adjust per provider — see cloud-resource-patterns.md
    ]
  }
}
```

## File: variables.tf

```hcl
# Required variables first, then optional with defaults.
# Alphabetical within each group.

variable "name" {
  description = "Name of the resource"
  type        = string
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

## File: outputs.tf

```hcl
output "id" {
  description = "The ID of the created resource"
  value       = <provider>_RESOURCE_TYPE.this.id
}

output "name" {
  description = "The name of the created resource"
  value       = <provider>_RESOURCE_TYPE.this.name
}
```

## File: terragrunt.hcl (component)

```hcl
include "root" {
  path   = "${get_repo_root()}/infrastructure-live/terragrunt.hcl"
  expose = true
}

terraform {
  source = "${get_repo_root()}/modules/<provider>/MODULE_NAME"
}

dependency "network" {
  config_path = "../network"
  mock_outputs = {
    vpc_id    = "vpc-mock-12345"
    subnet_id = "subnet-mock-12345"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  name = "RESOURCE_NAME-${include.root.locals.env}"
  tags = merge(include.root.locals.tags, { ManagedBy = "Terraform" })
}
```

## for_each Pattern

When creating multiple resources of the same type:

### variables.tf

```hcl
variable "resources" {
  description = "Map of resources to create"
  type = map(object({
    name   = string
    config = optional(map(string), {})
  }))
  default = {}
}
```

### main.tf

```hcl
resource "<provider>_RESOURCE_TYPE" "this" {
  for_each = var.resources

  name = each.value.name

  tags = var.tags

  lifecycle {
    ignore_changes = [
      # Ignore provider-managed tags (adjust per provider)
    ]
  }
}
```

### outputs.tf

```hcl
output "resource_ids" {
  description = "Map of resource keys to IDs"
  value       = { for k, v in <provider>_RESOURCE_TYPE.this : k => v.id }
}

output "resource_details" {
  description = "Map of resource keys to full details"
  value = { for k, v in <provider>_RESOURCE_TYPE.this : k => {
    id   = v.id
    name = v.name
  }}
}
```

## prevent_destroy Pattern

For critical resources that should never be accidentally deleted:

```hcl
resource "<provider>_RESOURCE_TYPE" "this" {
  name = var.name

  tags = var.tags

  lifecycle {
    prevent_destroy = true
    ignore_changes = [
      # Ignore provider-managed tags (adjust per provider)
    ]
  }
}
```

## Sources

- Terraform Module Development: https://developer.hashicorp.com/terraform/language/modules/develop
- terraform-docs: https://terraform-docs.io/
