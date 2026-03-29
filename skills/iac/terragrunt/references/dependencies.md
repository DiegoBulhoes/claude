# Unit Interdependencies

Units within a stack can depend on each other, creating a DAG (Directed Acyclic Graph) of resources.

## Dependency Patterns

### Fan-Out Pattern (Kubernetes Example)

```
kubernetes (core cluster)
├── kubernetes-config (depends on kubernetes)
├── node-group-system (depends on kubernetes)
└── workload-policies (depends on kubernetes)
```

### Chain Pattern (DNS Example)

```
vpc → dns-private → dns-public
       ↑
      vault
```

### Multiple Dependencies (Node Group Example)

```
node-group-system
├── depends on kubernetes (for cluster)
└── depends on vpc (for subnet placement)
```

## How Dependencies Work

### 1. Stack passes dependency paths via values

```hcl
# terragrunt.stack.hcl
unit "kubernetes" {
  source = "${local.catalog_path}//units/kubernetes?ref=main"
  path   = "kubernetes"
  values = { ... }
}

unit "node-group-system" {
  source = "${local.catalog_path}//units/node-group?ref=main"
  path   = "node-group-system"
  values = {
    cluster_path = "../kubernetes"  # Relative path to kubernetes unit
    version      = "v1.0.0"
  }
}

unit "workload-policies" {
  source = "${local.catalog_path}//units/workload-identity?ref=main"
  path   = "workload-policies"
  values = {
    cluster_path = "../kubernetes"  # Same dependency, different unit
    version      = "v1.0.0"
  }
}
```

### 2. Catalog units resolve paths to dependencies

```hcl
# units/node-group/terragrunt.hcl
dependency "kubernetes" {
  config_path = values.cluster_path  # "../kubernetes" from stack

  mock_outputs = {
    cluster_id   = "mock-cluster-id"
    cluster_name = "mock-cluster"
  }
  mock_outputs_allowed_terraform_commands = ["init", "validate", "plan", "destroy"]
}

inputs = {
  cluster_id = dependency.kubernetes.outputs.cluster_id
}
```

## Conditional Dependencies

Enable/disable dependencies based on configuration:

```hcl
# units/dns/terragrunt.hcl

# Only enable VPC dependency if using private DNS
dependency "vpc" {
  enabled      = try(values.scope, "GLOBAL") == "PRIVATE"
  config_path  = try(values.vpc_path, "../vpc")
  mock_outputs = {
    vpc_id = "mock-vpc-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

# Only enable vault dependency if encrypting records
dependency "vault" {
  enabled      = try(values.use_vault, false)
  config_path  = try(values.vault_path, "../vault")
  mock_outputs = {
    vault_id = "mock-vault-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

## Smart Skip Outputs

Skip dependency outputs based on whether they're actually needed:

```hcl
# units/dns/terragrunt.hcl
dependency "vpc" {
  config_path = try(values.vpc_path, "../vpc")

  # Only fetch outputs if this is a private DNS zone
  skip_outputs = try(values.scope, "GLOBAL") != "PRIVATE"

  mock_outputs = {
    vpc_id      = "mock-vpc-id"
    resolver_id = "mock-resolver-id"
  }
}
```

## Reference Resolution in Inputs

Resolve symbolic references to actual dependency outputs:

```hcl
inputs = {
  # Replace "../vpc" with actual VPC ID
  vpc_id = try(values.vpc_id, "") == "../vpc" ?
    dependency.vpc.outputs.vpc_id :
    values.vpc_id

  # Replace "../vault" with actual Vault ID
  vault_id = try(values.vault_id, "") == "../vault" ?
    dependency.vault.outputs.vault_id :
    values.vault_id
}
```

## Provider Generation from Dependencies

Generate providers that authenticate using dependency outputs:

```hcl
# units/kubernetes-config/terragrunt.hcl
generate "provider_kubectl" {
  path      = "cluster_auth.tf"
  if_exists = "overwrite"
  contents  = <<EOF
data "cloud_kubernetes_cluster" "cluster" {
  cluster_id = "${dependency.kubernetes.outputs.cluster_id}"
}

provider "kubectl" {
  host                   = "${dependency.kubernetes.outputs.cluster_endpoint}"
  cluster_ca_certificate = base64decode("${dependency.kubernetes.outputs.cluster_ca_certificate}")
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "cloud-cli"
    args        = ["kubernetes", "get-token", "--cluster-id", "${dependency.kubernetes.outputs.cluster_id}"]
  }
}
EOF
}
```

## Applying Single Units with Dependencies

When applying a single unit that has dependencies, the dependencies must already exist:

```bash
# First apply the base unit
terragrunt stack run apply --filter '.terragrunt-stack/kubernetes'

# Then apply dependent units (kubernetes must be applied first)
terragrunt stack run apply --filter '.terragrunt-stack/node-group-system'

# Or apply a unit and all its dependencies
terragrunt stack run apply --filter '.terragrunt-stack/node-group-system...'
```

## Best Practices

1. **Always provide mock outputs** - Required for plan/validate without real dependencies
2. **Use `enabled` for optional dependencies** - Don't fetch outputs for unused features
3. **Use `skip_outputs` for conditional fetching** - Based on actual usage in inputs
4. **Allow path overrides** - `try(values.X_path, "../default")` for flexibility
5. **Document required outputs** - In mock_outputs, show what the dependency must provide

## References

- [Patterns Guide](patterns.md)
- [Stack Commands](stack-commands.md)
