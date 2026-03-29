# Terraform Security Checklist

## Secrets Management

- [ ] No secrets hardcoded in `.tf` files
- [ ] No secrets in `.tfvars` files committed to Git
- [ ] Sensitive variables marked with `sensitive = true`
- [ ] Secrets sourced from external vault (HashiCorp Vault, cloud-native secret managers)
- [ ] Dynamic/short-lived credentials used where possible

## State Security

- [ ] Remote backend with encryption at rest
- [ ] State access restricted via IAM policies
- [ ] State locking enabled (DynamoDB, GCS, Consul, etc.)
- [ ] Versioning enabled on state storage
- [ ] State never committed to Git
- [ ] Separate state per environment

## IAM and Access

- [ ] Least-privilege IAM roles for Terraform execution
- [ ] Dedicated service accounts (not personal credentials)
- [ ] MFA required for production access
- [ ] Role-based access with time-limited credentials
- [ ] No wildcard (`*`) IAM permissions in production

## Network Security

- [ ] Default security group denies all traffic
- [ ] Ingress rules use specific CIDR blocks (not `0.0.0.0/0`)
- [ ] Egress restricted to necessary destinations
- [ ] VPC flow logs enabled
- [ ] Private subnets for compute resources
- [ ] Load balancers in public subnets only

## Encryption

- [ ] Encryption at rest for all storage (block volumes, object storage, databases)
- [ ] Encryption in transit (TLS) for all endpoints
- [ ] KMS keys with rotation enabled
- [ ] Customer-managed keys for sensitive workloads

## Compute Security

- [ ] AMI/image from trusted sources only
- [ ] SSH key pairs managed externally (not in Terraform state)
- [ ] Security groups with minimal open ports
- [ ] Instance metadata service v2 (IMDSv2) required
- [ ] No public IPs on compute instances (use bastion/load balancer)

## Database Security

- [ ] No default passwords
- [ ] Databases in private subnets
- [ ] Automated backups enabled
- [ ] Deletion protection enabled for production
- [ ] Audit logging enabled

## CI/CD Pipeline

- [ ] `terraform fmt -check` in CI
- [ ] `terraform validate` in CI
- [ ] `tflint` for provider-specific rules
- [ ] `checkov` or `tfsec` for security scanning
- [ ] Plan output reviewed before apply
- [ ] No auto-apply in production
- [ ] Credentials via OIDC/workload identity (not static secrets)

## Git Hygiene

- [ ] `.gitignore` includes: `*.tfstate`, `*.tfstate.backup`, `.terraform/`, `*.tfvars` (with secrets)
- [ ] `.terraform.lock.hcl` IS committed
- [ ] Pre-commit hooks for `fmt` and `validate`
- [ ] Branch protection on main branch

## Drift Detection

- [ ] Regular `terraform plan` to detect drift
- [ ] Alerts on manual infrastructure changes
- [ ] State refresh before operations

## Sources

- [HashiCorp Terraform Security Practices](https://www.hashicorp.com/en/blog/terraform-security-5-foundational-practices)
- [AWS Terraform Security Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/security.html)
- [CIS Benchmarks](https://www.cisecurity.org/benchmark)
- [OWASP Infrastructure as Code Security](https://owasp.org/www-project-devsecops-guideline/latest/02b-Infrastructure-as-Code)
