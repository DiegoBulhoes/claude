# Compliance Matrix

Complete compliance matrix for all checks performed by the audit skill.

## Component Compliance

| # | Check | Where | Expected | Severity |
|---|-------|-------|----------|----------|
| C1 | Root include | `terragrunt.hcl` | `include "root" { expose = true }` | HIGH |
| C2 | Source hardcoded | `terraform { source }` | Not via `envcommon` | HIGH |
| C3 | Tags | `inputs` | `merge(include.root.locals.tags, { ManagedBy = "Terraform" })` | MEDIUM |
| C4 | Mock outputs | `dependency {}` | Every dependency has `mock_outputs` | HIGH |
| C5 | env_local.hcl.example | Component dir | Exists when component needs external resource IDs | MEDIUM |
| C6 | No secrets | All `.hcl` files | No passwords, keys, or tokens | CRITICAL |

## Module Compliance

| # | Check | Where | Expected | Severity |
|---|-------|-------|----------|----------|
| M1 | File structure | Module dir | `main.tf`, `variables.tf`, `outputs.tf`, `README.md` | HIGH |
| M2 | Required providers | `main.tf` | `required_providers` block with pinned version | HIGH |
| M3 | Provider-managed tags lifecycle | Every resource with provider-managed tags | `lifecycle { ignore_changes = [...] }` for auto-set tag keys | HIGH |
| M4 | Output descriptions | `outputs.tf` | Every output has `description` | LOW |
| M5 | Variable descriptions | `variables.tf` | Every variable has `description` and `type` | LOW |
| M6 | README freshness | `README.md` | Matches `terraform-docs` output | LOW |
| M7 | No decorative comments | `.tf` files | No `#====`, `#----`, `#****` blocks | LOW |

## Resource Hierarchy Compliance

| # | Check | Where | Expected | Severity |
|---|-------|-------|----------|----------|
| R1 | Resource grouping exists | Each environment | Logical grouping component present (e.g., resource groups, projects, compartments) | HIGH |
| R2 | Network isolation | Network components | Reference the network resource group | HIGH |
| R3 | Application isolation | Non-network components | Reference the appropriate application resource group | HIGH |
| R4 | Core modules local | Core components | Local module source, not external registry | HIGH |
| R5 | Network module local | Network component | Local module variant, not external registry | MEDIUM |

## Security Compliance

| # | Check | Where | Expected | Severity |
|---|-------|-------|----------|----------|
| X1 | No git-tracked secrets | `git ls-files` | No `env_local.hcl` in output | CRITICAL |
| X2 | No hardcoded cloud resource IDs | `.hcl` files | `arn:`, `ocid1.`, `/subscriptions/` only in `mock_outputs`/account config | HIGH |
| X3 | No credentials | All files | No `private_key`, `password =`, `secret =` literals | CRITICAL |
| X4 | No local state | Repo root | No `terraform.tfstate` files | CRITICAL |

## Scoring

```
CRITICAL: Must fix before merge — security risk or data exposure
HIGH:     Must fix before deploy — breaks conventions or causes drift
MEDIUM:   Should fix — improves maintainability
LOW:      Nice to have — cosmetic or documentation
```

## Report Template

```markdown
# Audit Report — YYYY-MM-DD

## Summary
| Metric | Count |
|--------|-------|
| Components audited | N |
| Modules audited | N |
| Critical issues | N |
| High issues | N |
| Medium issues | N |
| Low issues | N |
| Fully compliant | N |

## Critical Issues
| File:Line | Check | Issue | Fix |
|-----------|-------|-------|-----|

## High Issues
| File:Line | Check | Issue | Fix |
|-----------|-------|-------|-----|

## Medium/Low Issues
| File:Line | Check | Issue | Fix |
|-----------|-------|-------|-----|
```

## Sources

- Terraform Style Guide: https://developer.hashicorp.com/terraform/language/style
