# Infrastructure Catalog Structure

## Directory Layout

```
infrastructure-catalog/
├── units/                      # Terragrunt units (building blocks)
│   ├── bastion/
│   │   └── terragrunt.hcl
│   ├── dns/
│   │   └── terragrunt.hcl
│   ├── database/
│   │   └── terragrunt.hcl
│   ├── kubernetes/
│   │   └── terragrunt.hcl
│   ├── object-storage/
│   │   └── terragrunt.hcl
│   ├── vault/
│   │   └── terragrunt.hcl
│   └── vpc/
│       └── terragrunt.hcl
└── stacks/                     # Template stacks (compositions)
    ├── kubernetes-cluster/
    │   └── terragrunt.stack.hcl
    ├── backend-api/
    │   └── terragrunt.stack.hcl
    └── data-platform/
        └── terragrunt.stack.hcl
```

## Units

Units are the building blocks of your infrastructure catalog. Each unit:
- Wraps a single OpenTofu/Terraform module
- Receives configuration through `values.xxx`
- Declares dependencies on other units
- Provides mock outputs for plan/validate

### Unit Pattern

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "git::git@github.com:YOUR_ORG/modules/database.git//app?ref=${values.version}"
}

dependency "vpc" {
  config_path  = try(values.vpc_path, "../vpc")
  skip_outputs = !try(values.use_vpc, true)

  mock_outputs = {
    vpc_id             = "mock-vpc-id"
    private_subnet_ids = ["mock-subnet-id"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  name           = values.name
  environment    = values.environment
  vpc_id         = dependency.vpc.outputs.vpc_id
  account_id     = values.account_id
}
```

## Stacks

Stacks compose multiple units into deployable infrastructure:

### Template Stacks (in catalog)

```hcl
locals {
  service     = values.service
  environment = values.environment
  domain      = values.domain

  fqdn = "${values.service}-${values.environment}.${values.domain}"

  common_tags = merge(try(values.tags, {}), {
    Stack       = "kubernetes-cluster"
    Service     = values.service
    Environment = values.environment
  })
}

unit "object-storage" {
  source = "git::git@github.com:YOUR_ORG/infrastructure-catalog.git//units/object-storage?ref=${values.catalog_version}"
  path   = "object-storage"

  values = {
    version = values.module_version
    bucket  = "my-bucket-${values.environment}"
    tags    = local.common_tags
  }
}

unit "dns" {
  source = "git::git@github.com:YOUR_ORG/infrastructure-catalog.git//units/dns?ref=${values.catalog_version}"
  path   = "dns"

  values = {
    version   = values.module_version
    zone_name = local.fqdn
  }
}
```

### Deployment Stacks (in live repo)

```hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  env_vars     = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  environment  = local.env_vars.locals.environment
  service      = "my-service"
}

unit "database" {
  source = "git::git@github.com:YOUR_ORG/infrastructure-catalog.git//units/database?ref=main"
  path   = "database"

  values = {
    version        = "v1.0.0"
    name           = "${local.service}-${local.environment}"
    account_id     = local.env_vars.locals.account_id
    subnet_id      = local.env_vars.locals.database_subnet_id
    instance_type  = "db.t3.medium"
    tags = merge(local.account_vars.locals.tags, {
      Service = local.service
    })
  }
}
```

## References

- [Unit Template](../assets/catalog-structure/units/template/)
- [Stack Template](../assets/catalog-structure/stacks/template/)
