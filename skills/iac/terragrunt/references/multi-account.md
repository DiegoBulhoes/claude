# Multi-Account Strategy

## Account Structure

Recommended cloud account structure:

```
Organization (root)
└── ORG_NAME (organizational unit)
    ├── tag-policies (billing tags)
    ├── dns (shared DNS zone)
    ├── ORG_NAME-dev
    │   ├── networking    (VPC, transit gateway, private DNS)
    │   └── workloads     (Kubernetes, bastion, node groups, etc.)
    ├── ORG_NAME-qa
    │   ├── networking
    │   └── workloads
    ├── ORG_NAME-staging
    │   ├── networking
    │   └── workloads
    └── ORG_NAME-prod
        ├── networking
        └── workloads
```

## Live Repository Structure

```
infrastructure-live/
├── root.hcl
├── accounts/
│   ├── account.hcl
│   ├── terragrunt.hcl
│   └── ORG_NAME/
│       ├── terragrunt.hcl
│       ├── components/
│       │   ├── tag-policies/
│       │   │   └── terragrunt.hcl
│       │   └── dns/
│       │       └── terragrunt.hcl
│       ├── ORG_NAME-dev/
│       │   ├── env.hcl
│       │   └── REGION/
│       │       ├── terragrunt.hcl
│       │       └── components/
│       │           ├── vpc/
│       │           ├── kubernetes/
│       │           ├── node-group-system/
│       │           ├── dns-private/
│       │           ├── dns-public/
│       │           └── bastion/
│       ├── ORG_NAME-qa/
│       │   ├── env.hcl
│       │   └── REGION/
│       │       └── components/
│       └── ORG_NAME-prod/
│           ├── env.hcl
│           └── REGION/
│               └── components/
```

## Cloud Authentication

### Environment Variable Authentication

Cloud providers typically use environment variables for authentication:

```bash
# Example: AWS
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"

# Or use a named profile
export AWS_PROFILE="my-profile"
```

The root.hcl generates the provider automatically:

```hcl
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "PROVIDER_NAME" {
  region = "${local.region}"
}
EOF
}
```

## Account Isolation

### Per-Environment State Buckets

Each environment has its own state bucket:

```hcl
bucket = format("terragrunt-state-%s-%s",
  local.account_name,
  local.prefix)
```

Results in:
- `terragrunt-state-acme-dev`
- `terragrunt-state-acme-qa`
- `terragrunt-state-acme-prod`

### Account-Level Variables

account.hcl provides organization-wide values:

```hcl
# accounts/account.hcl
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

### Environment-Level Variables

env.hcl provides environment-specific values:

```hcl
# ORG_NAME-dev/env.hcl
locals {
  environment = "ORG_NAME-dev"
  prefix      = "dev"

  tags = {
    Project     = "acme"
    Environment = "dev"
  }
}

# ORG_NAME-prod/env.hcl
locals {
  environment = "ORG_NAME-prod"
  prefix      = "prod"

  tags = {
    Project     = "acme"
    Environment = "prod"
  }
}
```

## Cross-Account DNS

For DNS zones shared across accounts (e.g., parent zone delegates to child zone):

```hcl
# Account-level DNS zone (DOMAIN_NAME)
# delegates subdomains to environment-specific zones

unit "dns" {
  values = {
    zone_name = "DOMAIN_NAME"
    records = {
      ns_dev = {
        domain        = "dev.DOMAIN_NAME"
        rtype         = "NS"
        nameservers   = ["ns1.example-dns.net", "ns2.example-dns.net"]
      }
    }
  }
}
```

## Deployment Order

Deploy shared infrastructure first:

1. Organization unit (ORG_NAME)
2. Tag policies (billing tags)
3. DNS zone (shared)
4. Environment account (ORG_NAME-dev)
5. Sub-accounts / resource groups
6. VPC (networking)
7. Vault + secrets
8. Kubernetes cluster
9. Node groups
10. Workload policies (IAM)
11. Remaining services (bastion, DNS, databases, buckets)

Use `run-all` with dependency ordering:

```bash
cd infrastructure-live
terragrunt run-all apply
```
