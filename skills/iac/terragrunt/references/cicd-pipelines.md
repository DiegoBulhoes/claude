# CI/CD Pipeline Examples

## Overview

This guide provides CI/CD pipeline templates for **Terragrunt Stacks** (explicit stacks using `terragrunt.stack.hcl`). These templates are suggestions that can be adapted to your organization's needs.

**Key features:**
- `terragrunt stack run` commands for explicit stacks
- Cloud provider authentication via environment variables
- SSH-based Git access (recommended over HTTPS)
- Provider caching for performance
- Selective unit targeting with `--filter` ([docs](https://terragrunt.gruntwork.io/docs/features/filter/))

> **Why SSH over HTTPS?**
> - **Enhanced security**: SSH keys provide stronger authentication than passwords or tokens
> - **Credential-free operations**: Once configured, no credentials needed for each Git operation
> - **No token expiration**: Unlike HTTPS tokens, SSH keys don't expire unexpectedly mid-pipeline

---

## Terragrunt Stack Commands

When working with explicit stacks (`terragrunt.stack.hcl`), use `terragrunt stack run` instead of `terragrunt run-all`:

```bash
# Navigate to stack directory
cd accounts/ORG_NAME/ORG_NAME-prod/us-east-1/my-service/

# Plan entire stack
terragrunt stack run plan

# Apply entire stack
terragrunt stack run apply

# Target specific units within the stack (using --filter)
terragrunt stack run plan --filter '.terragrunt-stack/database'
terragrunt stack run apply --filter '.terragrunt-stack/kubernetes'

# Target unit and its dependencies
terragrunt stack run apply --filter '.terragrunt-stack/kubernetes...'

# Destroy stack
terragrunt stack run destroy
```

### Useful Flags

| Flag | Description | Use Case |
|------|-------------|----------|
| `--filter` | Flexible unit targeting (recommended) | Target units, dependencies, patterns |
| `--queue-include-dir` | Target specific units by path (legacy) | Simple path-based targeting |
| `--queue-ignore-dag-order` | Run units concurrently | Faster plans (dangerous for apply) |
| `--queue-ignore-errors` | Continue on failures | Identify all errors at once |
| `--out-dir` | Save plan files to directory | Artifact storage for apply stage |
| `--parallelism N` | Limit concurrent units | Prevent rate limiting |

> **Note:** The `.terragrunt-stack` directory is auto-generated. Use `terragrunt stack generate` to pre-generate it, or `terragrunt stack clean` to remove it.

---

## GitLab CI

> **Best Practice: Reusable Templates**
>
> Structure GitLab CI with reusable templates (`.template-name`) that can be extended and overridden:
> - **Consistency**: All jobs follow the same patterns
> - **Maintainability**: Update logic in one place
> - **Flexibility**: Override specific steps when needed
>
> These templates are suggestions--adapt them to your organization's requirements.

### Base Templates (`.gitlab-ci.yml`)

```yaml
stages:
  - checks
  - plan
  - apply

default:
  image: "ghcr.io/opentofu/opentofu:latest"

variables:
  TG_STACK_PATH: "."              # Path to terragrunt.stack.hcl directory
  TG_PARALLELISM: "10"
  # Performance: Provider caching
  TG_PROVIDER_CACHE: "1"
  TG_PROVIDER_CACHE_DIR: "/tmp/provider-cache"
  TG_DOWNLOAD_DIR: "/tmp/module-cache"

# -----------------------------------------------------------------------------
# REUSABLE TEMPLATES
# -----------------------------------------------------------------------------

.terragrunt-cache:
  cache:
    key: terragrunt-${CI_COMMIT_REF_SLUG}
    paths:
      - /tmp/provider-cache
      - /tmp/module-cache
    policy: pull-push

.ssh-setup:
  before_script:
    - |
      mkdir -p ~/.ssh && chmod 700 ~/.ssh
      ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts 2>/dev/null
      ssh-keyscan -t rsa gitlab.com >> ~/.ssh/known_hosts 2>/dev/null

      # Setup SSH key for private repos (implementation depends on your secret management)
      # Options: SOPS, HashiCorp Vault, cloud-native secret managers, etc.
      # <RETRIEVE_SSH_KEY_FROM_SECRET_MANAGER> > ~/.ssh/id_rsa
      chmod 0400 ~/.ssh/id_rsa

# -----------------------------------------------------------------------------
# CHECK TEMPLATES
# -----------------------------------------------------------------------------

.terragrunt_version_check_template:
  stage: checks
  cache: {}
  script:
    - |
      echo "===== Version Check ====="
      # Adapt version checks to your tooling
      terragrunt --version
      tofu --version || terraform --version
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH'

.terragrunt_fmt_template:
  stage: checks
  cache: {}
  script:
    - cd $TG_STACK_PATH
    - terragrunt hclfmt --terragrunt-check
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH'

# -----------------------------------------------------------------------------
# STACK PLAN TEMPLATE
# -----------------------------------------------------------------------------

.terragrunt_stack_plan_template:
  stage: plan
  extends:
    - .terragrunt-cache
  script:
    - cd $TG_STACK_PATH
    - |
      echo "===== Stack Plan ====="

      # Plan entire stack
      terragrunt stack run plan \
        --parallelism ${TG_PARALLELISM}

      # Optional: Use --out-dir to save plans for apply stage
      # terragrunt stack run plan --out-dir ${CI_PROJECT_DIR}/plans
  artifacts:
    paths:
      - $TG_STACK_PATH/.terragrunt-stack/**/tfplan
    expire_in: 1 day
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - $TG_STACK_PATH/**/*
    - if: '$CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH'
      changes:
        - $TG_STACK_PATH/**/*

# -----------------------------------------------------------------------------
# STACK APPLY TEMPLATE
# -----------------------------------------------------------------------------

.terragrunt_stack_apply_template:
  stage: apply
  extends:
    - .terragrunt-cache
  script:
    - cd $TG_STACK_PATH
    - |
      echo "===== Stack Apply ====="

      # Re-plan and apply (recommended for safety)
      terragrunt stack run plan \
        --parallelism ${TG_PARALLELISM}

      terragrunt stack run apply \
        --parallelism ${TG_PARALLELISM}
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
      changes:
        - $TG_STACK_PATH/**/*
    - if: '$CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH'
      changes:
        - $TG_STACK_PATH/**/*

# -----------------------------------------------------------------------------
# TARGETED UNIT TEMPLATES (Optional)
# -----------------------------------------------------------------------------
# Use these when you need to deploy specific units within a stack

.terragrunt_stack_plan_unit_template:
  stage: plan
  extends:
    - .terragrunt-cache
  script:
    - cd $TG_STACK_PATH
    - |
      echo "===== Plan Unit: ${TG_TARGET_UNIT} ====="
      terragrunt stack run plan \
        --queue-include-dir ".terragrunt-stack/${TG_TARGET_UNIT}" \
        --parallelism ${TG_PARALLELISM}
  variables:
    TG_TARGET_UNIT: ""  # Override per job (e.g., "database", "kubernetes")

.terragrunt_stack_apply_unit_template:
  stage: apply
  extends:
    - .terragrunt-cache
  script:
    - cd $TG_STACK_PATH
    - |
      echo "===== Apply Unit: ${TG_TARGET_UNIT} ====="
      terragrunt stack run plan \
        --queue-include-dir ".terragrunt-stack/${TG_TARGET_UNIT}" \
        --parallelism ${TG_PARALLELISM}

      terragrunt stack run apply \
        --queue-include-dir ".terragrunt-stack/${TG_TARGET_UNIT}" \
        --parallelism ${TG_PARALLELISM}
  variables:
    TG_TARGET_UNIT: ""
```

---

### Cloud Authentication Pattern

```yaml
# cloud/.gitlab-ci-cloud.yml

.cloud-variables:
  variables:
    CLOUD_REGION: "us-east-1"
    TG_TARGET_ENVIRONMENT: ""      # Set per environment (e.g., "dev", "qa", "prod")

.cloud-auth:
  before_script:
    - |
      echo "===== Cloud Authentication ====="

      # Cloud authentication via environment variables
      # These should be set as masked CI/CD variables.
      # Adapt to your cloud provider (AWS, GCP, Azure, etc.)

      # SSH setup for private repos (see .ssh-setup template)
      mkdir -p ~/.ssh && chmod 700 ~/.ssh
      ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts 2>/dev/null
      ssh-keyscan -t rsa gitlab.com >> ~/.ssh/known_hosts 2>/dev/null
      # <RETRIEVE_SSH_KEY_FROM_SECRET_MANAGER> > ~/.ssh/id_rsa
      chmod 0400 ~/.ssh/id_rsa

# Example: Dev Stack
.dev-stack:
  extends: .cloud-variables
  variables:
    TG_STACK_PATH: "accounts/ORG_NAME/ORG_NAME-dev/us-east-1/my-service"
    TG_TARGET_ENVIRONMENT: "dev"
    CLOUD_REGION: "us-east-1"

# Example jobs using stack templates
cloud:dev:fmt:
  extends:
    - .terragrunt_fmt_template
    - .dev-stack
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - accounts/ORG_NAME/ORG_NAME-dev/**/*

cloud:dev:plan:
  extends:
    - .terragrunt_stack_plan_template
    - .cloud-auth
    - .dev-stack
  needs: ["cloud:dev:fmt"]
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - accounts/ORG_NAME/ORG_NAME-dev/**/*

cloud:dev:apply:
  extends:
    - .terragrunt_stack_apply_template
    - .cloud-auth
    - .dev-stack
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main"'
      changes:
        - accounts/ORG_NAME/ORG_NAME-dev/**/*
```

---

### Targeting Specific Units

When you need to deploy specific units within a stack (e.g., only the database component):

```yaml
# Example: Target a specific unit within the stack

cloud:dev:database:plan:
  extends:
    - .terragrunt_stack_plan_unit_template
    - .cloud-auth
    - .dev-stack
  variables:
    TG_TARGET_UNIT: "database"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - accounts/ORG_NAME/ORG_NAME-dev/us-east-1/my-service/terragrunt.stack.hcl

cloud:dev:database:apply:
  extends:
    - .terragrunt_stack_apply_unit_template
    - .cloud-auth
    - .dev-stack
  variables:
    TG_TARGET_UNIT: "database"
  needs: ["cloud:dev:database:plan"]
  rules:
    - if: '$CI_COMMIT_REF_NAME == "main"'
      when: manual
```

> **Tip:** Use `--queue-include-dir` to target multiple units:
> ```bash
> terragrunt stack run plan \
>   --queue-include-dir ".terragrunt-stack/object-storage" \
>   --queue-include-dir ".terragrunt-stack/database"
> ```

---

## GitHub Actions

### Cloud Authentication with GitHub Actions

```yaml
name: Terragrunt Deploy

on:
  pull_request:
    paths:
      - 'accounts/**'
  push:
    branches: [main]
    paths:
      - 'accounts/**'

permissions:
  contents: read

jobs:
  plan:
    runs-on: ubuntu-latest
    env:
      # Set cloud provider credentials as GitHub secrets
      # Adapt to your cloud provider (AWS, GCP, Azure, etc.)
      TG_PROVIDER_CACHE: "1"
      TG_PROVIDER_CACHE_DIR: /tmp/provider-cache
      TG_DOWNLOAD_DIR: /tmp/module-cache
    steps:
      - uses: actions/checkout@v4

      - name: Configure Cloud Credentials
        run: |
          # Adapt this step to your cloud provider authentication method
          echo "Configure your cloud provider credentials here"

      - uses: actions/cache@v4
        with:
          path: |
            /tmp/provider-cache
            /tmp/module-cache
          key: terragrunt-cache-${{ hashFiles('**/*.hcl') }}
          restore-keys: |
            terragrunt-cache-

      - name: Plan
        run: |
          cd accounts/ORG_NAME/ORG_NAME-dev/us-east-1/my-service
          terragrunt stack run plan --parallelism 10

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure Cloud Credentials
        run: |
          # Adapt this step to your cloud provider authentication method
          echo "Configure your cloud provider credentials here"

      - name: Apply
        run: |
          cd accounts/ORG_NAME/ORG_NAME-dev/us-east-1/my-service
          terragrunt stack run apply --parallelism 10
```

### Key Differences from GitLab CI

| GitLab CI | GitHub Actions |
|-----------|----------------|
| `extends: .template` | `uses: ./.github/workflows/reusable.yml` |
| `rules: changes:` | `paths:` filter or `dorny/paths-filter` |
| `needs: [job]` | `needs: [job]` (same) |
| `when: manual` | `environment:` with required reviewers |

---

## Cloud Authentication Methods

### Method 1: Environment Variable Authentication (Recommended for CI/CD)

Set cloud provider credentials as masked CI/CD variables. Adapt to your provider:

| Provider | Key Variables |
|----------|--------------|
| **AWS** | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION` |
| **GCP** | `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_PROJECT` |
| **Azure** | `ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID` |

### Method 2: Instance/Workload Identity (For cloud-hosted runners)

When running CI/CD on cloud-hosted instances, use identity-based authentication:

```hcl
# AWS example: IAM role via instance profile
provider "aws" {
  region = "us-east-1"
  # No explicit credentials needed - uses instance profile
}
```

Requires:
1. IAM role/identity attached to the CI runner instance
2. Appropriate permissions policy

### Method 3: OIDC Federation (For GitHub Actions / GitLab CI)

Use OIDC to get short-lived credentials without storing secrets:

```yaml
# GitHub Actions example with AWS OIDC
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
    aws-region: us-east-1
```

---

## References

### Terragrunt Stack Commands
- [Terragrunt Stacks](https://terragrunt.gruntwork.io/docs/features/stacks/) - Official documentation for explicit stacks
- [Terragrunt Run Command](https://terragrunt.gruntwork.io/docs/reference/cli/commands/run/) - CLI reference for run flags
- [Run Queue](https://terragrunt.gruntwork.io/docs/features/run-queue/) - Queue flags and filtering

### CI/CD Basics
- [Terragrunt Performance Guide](performance.md)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
