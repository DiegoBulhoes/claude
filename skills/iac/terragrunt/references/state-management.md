# State Management Best Practices

## Remote State Configuration

### S3-Compatible Backend

Recommended configuration in root.hcl:

```hcl
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
```

Key settings:
- **S3 backend**: Standard backend with state locking via DynamoDB (or equivalent)
- **encrypt**: Server-side encryption for state files at rest
- **path_relative_to_include()**: Unique key per unit
- **prefix-based buckets**: Environment isolation via bucket naming

## State Isolation

Each unit gets its own state file through `path_relative_to_include()`:

```
infrastructure-live/
├── accounts/ORG_NAME/ORG_NAME-dev/.../components/kubernetes/ → terragrunt-state-acme-dev/accounts/ORG_NAME/ORG_NAME-dev/.../components/kubernetes/tf.tfstate
├── accounts/ORG_NAME/ORG_NAME-dev/.../components/vpc/       → terragrunt-state-acme-dev/accounts/ORG_NAME/ORG_NAME-dev/.../components/vpc/tf.tfstate
└── accounts/ORG_NAME/ORG_NAME-prod/.../components/kubernetes/→ terragrunt-state-acme-prod/accounts/ORG_NAME/ORG_NAME-prod/.../components/kubernetes/tf.tfstate
```

Benefits:
- Blast radius limited to single unit
- Independent apply/destroy operations
- Clear state file organization

## Environment-Based State Buckets

Use `prefix` in env.hcl for environment isolation:

```hcl
# ORG_NAME-dev/env.hcl
locals {
  environment = "dev"
  prefix      = "dev"
}

# ORG_NAME-qa/env.hcl
locals {
  environment = "qa"
  prefix      = "qa"
}
```

This creates separate buckets per environment:
- `terragrunt-state-acme-dev`
- `terragrunt-state-acme-qa`
- `terragrunt-state-acme-prod`

## State Bucket Setup

### Prerequisites

Before first Terragrunt run, create:
1. S3-compatible bucket with versioning enabled
2. DynamoDB table (or equivalent) for state locking

Alternatively, use `--backend-bootstrap` on first run to auto-create the bucket:

```bash
terragrunt plan --backend-bootstrap
```

### Manual Bucket Creation

Using AWS CLI (adapt for your cloud provider):

```bash
# Create state bucket
aws s3api create-bucket \
  --bucket terragrunt-state-acme-dev \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket terragrunt-state-acme-dev \
  --versioning-configuration Status=Enabled

# Create lock table
aws dynamodb create-table \
  --table-name terragrunt-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

## State Migration

Moving state between backends:

```bash
# 1. Initialize with current backend
terragrunt init

# 2. Update backend configuration in root.hcl

# 3. Re-initialize with migration
terragrunt init -migrate-state
```

## Avoiding Workspaces

Terragrunt recommends against OpenTofu/Terraform workspaces:

**Don't use:**
```hcl
terraform workspace select dev
```

**Do use:** Separate directories per environment with isolated state:
```
accounts/ORG_NAME/ORG_NAME-dev/us-east-1/components/kubernetes/
accounts/ORG_NAME/ORG_NAME-qa/us-east-1/components/kubernetes/
accounts/ORG_NAME/ORG_NAME-prod/us-east-1/components/kubernetes/
```

Each directory = isolated state = clear separation.

## Cross-Account State Access

For reading state from other accounts:

```hcl
data "terraform_remote_state" "shared_vpc" {
  backend = "s3"
  config = {
    bucket = "terragrunt-state-acme-root"
    key    = "accounts/ORG_NAME/components/vpc/tf.tfstate"
    region = "us-east-1"
  }
}
```
