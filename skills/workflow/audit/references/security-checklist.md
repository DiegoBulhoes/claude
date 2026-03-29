# IaC Security Scanning Checklist

## Credential Exposure

| Check | Pattern | Severity | Action |
|-------|---------|----------|--------|
| Git-tracked secrets | `env_local.hcl` in `git ls-files` | CRITICAL | Remove from git, add to `.gitignore` |
| Hardcoded passwords | `password\s*=\s*"[^"]+"` in `.tf`/`.hcl` | CRITICAL | Use secret manager or `env_local.hcl` |
| Private keys in code | `private_key`, `BEGIN RSA`, `BEGIN EC` | CRITICAL | Use secret manager |
| Hardcoded cloud resource IDs | `arn:`, `ocid1.`, `/subscriptions/` outside `mock_outputs`/account config | HIGH | Use dependency outputs or variables |
| API keys | `api_key`, `access_key`, `secret_key` | CRITICAL | Use environment variables |

## State File Security

| Check | Expected | Risk if Missing |
|-------|----------|----------------|
| Backend encryption | Encryption configured in backend block | State contains plaintext secrets |
| Backend storage access | Private, IAM-restricted | Unauthorized state access |
| State locking | Backend supports locking (DynamoDB, versioning, etc.) | Concurrent modifications |
| No local state | No `terraform.tfstate` in repo | Secrets committed to git |

## Network Security

| Check | Rule |
|-------|------|
| No `0.0.0.0/0` ingress | Unless explicitly for public load balancers |
| Private subnets for compute | Workers, databases, API endpoints |
| Security groups over broad rules | Scoped, per-resource security groups preferred |
| VPN-only management access | Bastion or VPN for SSH/kubectl |

## IAM Best Practices

| Check | Rule |
|-------|------|
| Least privilege | Policies scoped to specific resource groups, not root/organization |
| No wildcard resources | Broad `*` permissions only at scoped levels |
| Workload Identity | Service accounts with scoped policies |
| Separate network/app groups | Network resources isolated from app resources |

## Provider-Managed Tags and Metadata

| Check | Rule |
|-------|------|
| Provider-managed tags lifecycle | Every resource with auto-set tags has `lifecycle { ignore_changes }` |
| `prevent_destroy` | On critical resources (key vaults, storage, databases) |
| Management tags | `ManagedBy = "Terraform"` on all resources |

## Tools

| Tool | Purpose | Command |
|------|---------|---------|
| `terraform validate` | Syntax and type checking | `terraform validate` |
| `terraform fmt -check` | Formatting compliance | `terraform fmt -check -recursive` |
| `tflint` | Linting and best practices | `tflint --recursive` |
| `trivy` | Security scanning | `trivy config infrastructure-live/modules/` |
| `checkov` | Policy-as-code | `checkov -d infrastructure-live/modules/` |

## Sources

- HashiCorp Security Best Practices: https://developer.hashicorp.com/terraform/language/settings/backends/configuration#credentials-and-sensitive-data
- OWASP Infrastructure as Code Security: https://cheatsheetseries.owasp.org/cheatsheets/Infrastructure_as_Code_Security_Cheat_Sheet.html
- Trivy IaC Scanning: https://aquasecurity.github.io/trivy/latest/docs/scanner/misconfiguration/
