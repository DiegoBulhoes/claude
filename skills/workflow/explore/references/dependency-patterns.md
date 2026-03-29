# Dependency Patterns

Common dependency patterns found in Terragrunt-managed infrastructure.

## Core Dependency Chain

```
foundation
  └── networking
        ├── virtual-network (uses network resources)
        │     └── cluster (uses network + compute resources)
        │           ├── node-pool
        │           ├── workload-policies
        │           └── kubernetes
        ├── secrets-vault
        │     └── secrets
        ├── bastion
        ├── container-registry
        ├── dns-private (uses network resources)
        ├── dns-public
        ├── database (uses secrets for credentials)
        └── storage
```

## Common Dependency Blocks

### Foundation / Base Component (Most Used)

Almost every component depends on a foundation or base component:

```hcl
dependency "foundation" {
  config_path = "../foundation"
  mock_outputs = {
    resource_group = "mock-resource-group"
    network_id     = "mock-network-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

### Network (Network-Dependent Components)

```hcl
dependency "network" {
  config_path = "../virtual-network"
  mock_outputs = {
    network_id = "mock-network-id"
    subnet_id  = { "workers" = "mock-subnet-id" }
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

### Cluster (Post-Cluster Components)

```hcl
dependency "cluster" {
  config_path = "../cluster"
  mock_outputs = {
    cluster_id = "mock-cluster-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

### Secrets (Database Components)

```hcl
dependency "secrets" {
  config_path = "../secrets"
  mock_outputs = {
    ssh_public_keys = { "bastion" = "ssh-rsa MOCK" }
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

## Cross-Boundary Dependencies

Some components need resources from multiple logical groupings:

| Component | Network Resources | Compute Resources |
|-----------|------------------|-------------------|
| virtual-network | VPC/VNet, subnets, gateways | — |
| cluster | VPC/VNet, routing | Node groups, cluster |
| dns-private | DNS view, resolver | Zone |
| bastion | — | Bastion host/service |

## Dependency Resolution Order

Terragrunt resolves dependencies automatically with `run-all`:

```
1. foundation (no deps)
2. virtual-network (depends on: foundation)
   secrets-vault (depends on: foundation)
3. secrets (depends on: secrets-vault)
   cluster (depends on: foundation, virtual-network)
4. node-pool (depends on: cluster)
   workload-policies (depends on: foundation, cluster)
   bastion (depends on: foundation)
5. dns-private (depends on: foundation, virtual-network)
   dns-public (depends on: foundation)
   container-registry (depends on: foundation)
6. database (depends on: foundation, secrets, virtual-network)
7. storage (depends on: foundation)
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Circular dependencies | `run-all` fails with cycle error | Redesign component boundaries |
| Missing mock_outputs | CI `plan` fails | Always add mock_outputs |
| Hardcoded IDs in deps | Breaks in other environments | Use dependency outputs |
| Over-depending | Slow `plan` due to cascade | Only depend on what you consume |

## Sources

- Terragrunt Dependencies: https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#dependency
- Terragrunt run-all: https://terragrunt.gruntwork.io/docs/features/execute-terraform-commands-on-multiple-modules-at-once/
