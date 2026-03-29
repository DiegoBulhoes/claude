# Root Configuration (root.hcl)

The root.hcl file is the central configuration for your Terragrunt live repository.

## Complete Example

```hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  env_vars     = try(read_terragrunt_config(find_in_parent_folders("env.hcl")), { locals = {} })

  account_name = local.account_vars.locals.account_name
  account_id   = local.account_vars.locals.account_id
  region       = local.account_vars.locals.region
  environment  = try(local.env_vars.locals.environment, local.account_vars.locals.environment)
  prefix       = try(local.env_vars.locals.prefix, local.account_vars.locals.prefix)
  tags         = try(local.env_vars.locals.tags, local.account_vars.locals.tags)
}

# Generate provider (uses environment variables or profiles for authentication)
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "PROVIDER_NAME" {
  region = "${local.region}"
}
EOF
}

# Generate OpenTofu version constraints
generate "versions" {
  path      = "versions.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    # Replace with your cloud provider
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
EOF
}

# Remote state with S3-compatible backend
remote_state {
  backend = "s3"
  config = {
    bucket         = format("terragrunt-state-%s-%s",
                      local.account_name,
                      local.prefix)
    key            = "${path_relative_to_include()}/tf.tfstate"
    region         = local.region
    encrypt        = true
    dynamodb_table = "terragrunt-locks"
  }
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
}

catalog {
  urls = [
    "git@github.com:YOUR_ORG/infrastructure-catalog.git"
  ]
}

inputs = merge(
  local.account_vars.locals,
  local.env_vars.locals
)
```

## Key Sections

### Variable Resolution

```hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  env_vars     = try(read_terragrunt_config(find_in_parent_folders("env.hcl")), { locals = {} })
}
```

- `find_in_parent_folders()` searches up the directory tree
- `try()` provides fallback when env.hcl doesn't exist
- There is no `region.hcl` -- `region` lives in `account.hcl`

### Provider Generation

The provider block configures:
- **region**: From account.hcl
- **Authentication**: Via environment variables or CLI profiles (depends on your cloud provider)
- **tags**: Applied to resources at the module level or via provider default tags (provider-dependent)

### Remote State

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = format("terragrunt-state-%s-%s", ...)
    key            = "${path_relative_to_include()}/tf.tfstate"
    region         = local.region
    encrypt        = true
    dynamodb_table = "terragrunt-locks"
  }
}
```

Key features:
- **Environment isolation**: `prefix` creates separate buckets per environment
- **Unique keys**: `path_relative_to_include()` ensures unique state per unit
- **State locking**: DynamoDB table (or equivalent) prevents concurrent modifications
- **Encryption**: Server-side encryption for state files at rest

### Catalog Configuration

```hcl
catalog {
  urls = [
    "git@github.com:YOUR_ORG/infrastructure-catalog.git",
    "git@github.com:YOUR_ORG/infrastructure-extra-catalog.git"  # Multiple catalogs
  ]
}
```

Enables `terragrunt catalog` to browse and scaffold from these repositories.

### Input Merging

```hcl
inputs = merge(
  local.account_vars.locals,
  local.env_vars.locals
)
```

All variables from the hierarchy are merged and passed to units as inputs.

## Account Configuration (account.hcl)

```hcl
locals {
  account_name = "acme"
  region       = "us-east-1"
  account_id   = "123456789012"
  environment  = "production"
  prefix       = "root"

  tags = {
    Project     = "acme"
    Environment = "production"
  }
}
```

## Environment Configuration (env.hcl)

```hcl
locals {
  environment = "dev"
  prefix      = "dev"

  tags = {
    Project     = "acme"
    Environment = "dev"
  }
}
```

## References

- [Live Structure](live-structure.md)
- [State Management](state-management.md)
- [Multi-Account Strategy](multi-account.md)
