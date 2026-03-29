# Naming Conventions

## Repository Names

### Catalog Repositories

| Pattern | Example | Use Case |
|---------|---------|----------|
| `infrastructure-<org>-catalog` | `infrastructure-acme-catalog` | Single cloud or multi-cloud catalog |
| `infrastructure-<cloud>-<org>-catalog` | `infrastructure-aws-acme-catalog` | Cloud-specific catalogs |

### Live Repositories

| Pattern | Example | Use Case |
|---------|---------|----------|
| `infrastructure-<org>-live` | `infrastructure-acme-live` | Single live repo |
| `infrastructure-<cloud>-<org>-live` | `infrastructure-aws-acme-live` | Cloud-specific live repos |

### Module Repositories

| Pattern | Example |
|---------|---------|
| `terraform-<provider>-<name>` | `terraform-aws-vpc`, `terraform-aws-eks` |
| `modules-<org>-<name>` | `modules-acme-networking` |

## Directory and Resource Names

| Type | Convention | Examples |
|------|------------|----------|
| **Units** | lowercase, hyphen-separated | `eks-config`, `dns-public` |
| **Stacks** | lowercase, hyphen-separated, descriptive | `kubernetes-cluster`, `database-system` |
| **Environments** | lowercase | `dev`, `qa`, `prod` |
| **Accounts** | lowercase with org prefix | `acme-prod`, `acme-dev` |

## State Bucket Naming

State bucket names follow the pattern:

```
terragrunt-state-{account_name}-{prefix}
```

| With prefix | Without prefix |
|-------------|----------------|
| `terragrunt-state-acme-dev` | `terragrunt-state-acme-root` |

The prefix comes from `env.hcl`:

```hcl
locals {
  environment = "dev"
  prefix      = "dev"
}
```
