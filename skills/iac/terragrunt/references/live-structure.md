# Infrastructure Live Structure

## Directory Layout

```
infrastructure-live/
├── root.hcl                    # Root configuration
├── modules/                    # Local Terraform modules
├── accounts/                   # Cloud accounts (root level)
│   ├── account.hcl             # Account config (account_id, region, tags)
│   ├── terragrunt.hcl          # exclude: actions = ["all"]
│   └── <org>/                  # Organization unit
│       ├── terragrunt.hcl      # Creates org-level resources
│       ├── components/         # Account-level components (tags, DNS)
│       └── <environment>/
│           ├── env.hcl         # Environment config (prefix, tags)
│           └── <region>/       # Regional resources
│               ├── terragrunt.hcl  # Creates environment-level resources
│               └── components/     # Environment components
│                   └── <service>/
│                       └── terragrunt.hcl
```

## Example Structure

```
infrastructure-live/
├── root.hcl
├── modules/
│   ├── vpc/
│   ├── kubernetes/
│   ├── dns/
│   ├── vault/
│   └── bastion/
└── accounts/
    ├── account.hcl
    └── ORG_NAME/
        ├── terragrunt.hcl
        ├── components/
        │   ├── tag-policies/
        │   │   └── terragrunt.hcl
        │   └── dns/
        │       └── terragrunt.hcl
        ├── ORG_NAME-dev/
        │   ├── env.hcl
        │   └── us-east-1/
        │       ├── terragrunt.hcl
        │       └── components/
        │           ├── vpc/
        │           ├── vault/
        │           ├── kubernetes/
        │           ├── node-group-system/
        │           ├── bastion/
        │           ├── dns-private/
        │           └── dns-public/
        └── ORG_NAME-prod/
            ├── env.hcl
            └── us-east-1/
                ├── terragrunt.hcl
                └── components/
                    ├── vpc/
                    ├── vault/
                    ├── kubernetes/
                    └── bastion/
```

## Configuration Hierarchy

### root.hcl
The root configuration file contains:
- Provider generation (cloud provider using environment variables or profiles)
- Version constraints (OpenTofu/Terraform versions)
- Remote state configuration (S3-compatible backend with state locking)
- Catalog URLs
- Input merging from account/env vars

See: [Root Configuration Guide](root-config.md)

### account.hcl
Account-wide configuration:

```hcl
locals {
  account_name = "acme"
  region       = "us-east-1"
  account_id   = "123456789012"
  environment  = "acme"
  prefix       = "root"

  tags = {
    Project   = "acme"
    ManagedBy = "Terraform"
  }
}
```

### env.hcl
Environment-specific configuration:

```hcl
locals {
  environment = "ORG_NAME-dev"
  prefix      = "dev"

  tags = {
    Project     = "acme"
    Environment = "dev"
  }
}
```

## State Isolation

Each unit gets its own state file through `path_relative_to_include()`:

```
infrastructure-live/
├── accounts/ORG_NAME/ORG_NAME-dev/.../components/kubernetes/ → terragrunt-state-acme-dev/accounts/ORG_NAME/ORG_NAME-dev/.../components/kubernetes/tf.tfstate
├── accounts/ORG_NAME/ORG_NAME-dev/.../components/vpc/       → terragrunt-state-acme-dev/accounts/ORG_NAME/ORG_NAME-dev/.../components/vpc/tf.tfstate
└── accounts/ORG_NAME/ORG_NAME-prod/.../components/kubernetes/→ terragrunt-state-acme-prod/accounts/ORG_NAME/ORG_NAME-prod/.../components/kubernetes/tf.tfstate
```

## References

- [Root Configuration](root-config.md)
- [Multi-Account Strategy](multi-account.md)
- [State Management](state-management.md)
