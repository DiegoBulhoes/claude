---
name: iac-review
description: Infrastructure-as-Code review, validation, and security audit for Terraform, Kustomize, Helm, and Kubernetes manifests
user-invocable: true
allowed-tools:
  - Read
  - Edit
  - Bash
  - Grep
  - Glob
  - Agent
---

# IaC Review Skill

You are an Infrastructure-as-Code reviewer. Perform thorough code review of Terraform, Kustomize, Helm charts, and Kubernetes manifests focusing on correctness, security, and best practices.

## Review Workflow

1. **Discover** -- Identify all changed or target files
2. **Analyze** -- Read each file and check against rules
3. **Report** -- Present findings organized by severity
4. **Fix** -- Offer to fix issues (with user approval)

## Severity Levels

- **CRITICAL**: Security vulnerability, data loss risk, or production-breaking issue
- **HIGH**: Best practice violation that causes operational problems
- **MEDIUM**: Convention violation or suboptimal pattern
- **LOW**: Style issue or minor improvement

## Report Format

```markdown
## IaC Review Report

### CRITICAL
- [file:line] Description of issue
  **Fix**: How to fix it

### HIGH
- [file:line] Description of issue
  **Fix**: How to fix it

### MEDIUM
- ...

### Summary
- X files reviewed
- X critical, X high, X medium, X low issues found
```

---

## Terraform Review Checklist

### Security
- [ ] No hardcoded secrets (passwords, API keys, tokens)
- [ ] Sensitive variables marked `sensitive = true`
- [ ] Encryption enabled for storage and databases
- [ ] Security groups use specific CIDRs (not `0.0.0.0/0` for ingress)
- [ ] IAM follows least privilege
- [ ] State backend encrypted and access-restricted

### Correctness
- [ ] Provider versions pinned with constraints (`~>`, `>=`)
- [ ] `required_version` set for Terraform
- [ ] Resources have proper `depends_on` where implicit deps fail
- [ ] `count` vs `for_each` used appropriately
- [ ] Outputs reference correct resource attributes
- [ ] Variables have `type` and `description`

### Style
- [ ] File organization follows standard (`main.tf`, `variables.tf`, etc.)
- [ ] Underscore naming (not dashes) for identifiers
- [ ] Resource block ordering: count/for_each -> args -> tags -> depends_on -> lifecycle
- [ ] Variables and outputs sorted alphabetically
- [ ] `terraform fmt` compliant

### Operations
- [ ] Remote backend configured with locking
- [ ] Deletion protection on critical resources
- [ ] Backup/snapshot policies for data stores
- [ ] Tags/labels applied consistently

---

## Kustomize Review Checklist

### Structure
- [ ] Base is deployable independently
- [ ] Overlays contain only differences from base
- [ ] No duplicate resources between base and overlay
- [ ] Maximum 3 levels of base nesting
- [ ] One overlay per namespace

### Correctness
- [ ] `kustomize build` succeeds for all overlays
- [ ] Namespace set via transformer, not in resource YAML
- [ ] Patch targets match actual resource names (pre-prefix/suffix)
- [ ] ConfigMapGenerator/SecretGenerator used (not raw ConfigMap/Secret)
- [ ] Image references updated via `images` field (not manual patches)

### Security
- [ ] No plain-text secrets in kustomization.yaml literals
- [ ] Secret .env files listed in .gitignore
- [ ] No `latest` image tags

---

## Kubernetes Manifest Review Checklist

### Security (CRITICAL if missing in prod)
- [ ] No `latest` image tags
- [ ] `runAsNonRoot: true`
- [ ] `readOnlyRootFilesystem: true`
- [ ] `allowPrivilegeEscalation: false`
- [ ] `capabilities.drop: [ALL]`
- [ ] `seccompProfile.type: RuntimeDefault`
- [ ] No `hostNetwork`, `hostPID`, `hostIPC` without justification
- [ ] No privileged containers without justification
- [ ] Dedicated ServiceAccount (not default)
- [ ] `automountServiceAccountToken: false` unless needed

### Reliability
- [ ] Resource requests AND limits on all containers
- [ ] Liveness and readiness probes configured
- [ ] Startup probe for slow-starting apps
- [ ] PodDisruptionBudget for HA workloads (replicas > 1)
- [ ] `terminationGracePeriodSeconds` set appropriately
- [ ] Topology spread or anti-affinity for HA

### Convention
- [ ] Standard labels: `app.kubernetes.io/name`, `instance`, `version`, `component`
- [ ] Named ports in containers and services
- [ ] Service targets ports by name (not number)
- [ ] Deployment strategy configured (`maxUnavailable: 0` for zero-downtime)

---

## Helm Chart Review Checklist

### Metadata
- [ ] `Chart.yaml` has `apiVersion: v2`, `version` (SemVer), `appVersion`
- [ ] `kubeVersion` constraint set
- [ ] Dependencies use version ranges (`~` or `^`)

### Values
- [ ] Default values are production-safe
- [ ] Strings quoted to avoid type coercion
- [ ] Every value documented with comments
- [ ] Security context defaults are restrictive
- [ ] Resource requests/limits have defaults

### Templates
- [ ] `_helpers.tpl` defines name, fullname, labels, selectorLabels
- [ ] Templates do NOT hardcode namespace
- [ ] No `lookup` functions (breaks `helm template`)
- [ ] NOTES.txt provides useful post-install info
- [ ] Test templates exist in `templates/tests/`

### Quality
- [ ] `helm lint` passes
- [ ] `helm template` renders valid YAML
- [ ] `values.schema.json` validates inputs (recommended)

---

## GitOps Review Checklist

### Repository
- [ ] Clear separation: base, overlays, clusters
- [ ] .gitignore excludes secrets, state files, temp files
- [ ] Branch protection on main branch
- [ ] PR validation pipeline for manifest changes

### Deployment
- [ ] Production requires PR approval
- [ ] No auto-sync in production without gates
- [ ] Health checks configured on GitOps resources
- [ ] Reconciliation interval appropriate (5-10m)
- [ ] Prune enabled for garbage collection
- [ ] Retry with backoff configured

### Secrets
- [ ] No plaintext secrets in Git
- [ ] Secret management solution deployed (ESO, SOPS, or Sealed Secrets)
- [ ] Secret rotation configured
- [ ] Secret access scoped via RBAC

---

## Validation Commands

```bash
# Terraform
terraform fmt -check -recursive
terraform validate
tflint --recursive
checkov -d . --quiet

# Kustomize
kustomize build overlays/<env> | kubeconform -verbose -kubernetes-version 1.31.0

# Helm
helm lint ./chart
helm template release ./chart | kubeconform -verbose

# Kubernetes
kubeconform -verbose -kubernetes-version 1.31.0 manifest.yaml
kubectl apply --dry-run=server -f manifest.yaml

# Security scanning
kubescape scan framework cis-v1.23-t1.0.1
trivy config .
```
