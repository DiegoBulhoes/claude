---
name: gitops
description: ArgoCD-based GitOps workflows, repository structure, environment promotion, and secret management for Kubernetes
user-invocable: true
allowed-tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
  - Agent
---

# GitOps Workflow Skill (ArgoCD)

You are a GitOps specialist for Kubernetes infrastructure using ArgoCD. Follow OpenGitOps principles and ArgoCD best practices.

## Core Principles (OpenGitOps)

1. **Declarative** -- All desired system state is expressed declaratively
2. **Versioned and Immutable** -- Desired state is stored in Git (immutable, versioned, retainable)
3. **Pulled Automatically** -- ArgoCD pulls desired state from Git and applies it
4. **Continuously Reconciled** -- ArgoCD continuously observes and reconciles actual state with desired state

## Repository Structure Patterns

### Pattern 1: Monorepo (Recommended for small-medium teams)

```
├── base/                    # Shared resource definitions
│   ├── cert-manager/
│   ├── ingress-nginx/
│   ├── external-secrets/
│   ├── monitoring/
│   └── app-a/
├── overlays/                # Environment-specific configurations
│   ├── <cluster>/
│   │   ├── dev/
│   │   ├── qa/
│   │   └── prod/
└── clusters/                # ArgoCD Application/ApplicationSet definitions
    ├── dev/
    ├── qa/
    └── prod/
```

### Pattern 2: Infrastructure + Apps Separation

```
├── infrastructure/
│   ├── base/               # Cluster addons (sync wave -1)
│   │   ├── cert-manager/
│   │   ├── ingress/
│   │   └── monitoring/
│   └── overlays/
│       ├── dev/
│       └── prod/
├── apps/
│   ├── base/               # Application manifests (sync wave 0+)
│   │   ├── api/
│   │   └── worker/
│   └── overlays/
│       ├── dev/
│       └── prod/
└── clusters/
    ├── dev/
    └── prod/
```

### Pattern 3: Repo per Team (Enterprise)

- **Platform team**: Infrastructure repo (cluster addons, policies, RBAC)
- **App teams**: Each team owns their app deployment repo
- **ArgoCD**: References all repos via Application/ApplicationSet resources

## App of Apps Pattern

Use a root Application that manages all other Applications:

```yaml
# clusters/prod/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: main
    path: clusters/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

## Environment Promotion Strategies

### Strategy 1: Directory-Based (Kustomize Overlays) -- Recommended

```
overlays/
├── dev/     # Auto-sync from main branch
├── qa/      # Promote via PR updating image tags
└── prod/    # Promote via PR with approval gates
```

**Promotion flow:**
1. Developer pushes image tag update to `overlays/dev/`
2. Dev auto-syncs via ArgoCD
3. QA promotion: PR updating `overlays/qa/` image tags
4. Prod promotion: PR with required approvals updating `overlays/prod/`

### Strategy 2: Branch-Based

```
main     -> syncs to dev
release  -> syncs to qa
stable   -> syncs to prod
```

**Not recommended**: Makes change history harder to follow across environments.

### Strategy 3: Image Tag Promotion (ArgoCD Image Updater)

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: my-app=registry.example.com/my-app
    argocd-image-updater.argoproj.io/my-app.update-strategy: semver
    argocd-image-updater.argoproj.io/my-app.allow-tags: "regexp:^[0-9]+\\.[0-9]+\\.[0-9]+$"
    argocd-image-updater.argoproj.io/write-back-method: git
```

1. CI builds and pushes tagged image
2. ArgoCD Image Updater detects new tag matching policy
3. Image Updater commits tag update to the overlay
4. ArgoCD syncs the change

## ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: main
    path: overlays/cluster/prod/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## ApplicationSet (Multi-Environment)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/org/gitops-repo.git
        revision: main
        directories:
          - path: overlays/cluster-*/*/my-app
  template:
    metadata:
      name: '{{path.basename}}-{{path[1]}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops-repo.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}-{{path[1]}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### ApplicationSet Generators

| Generator | Use Case |
|-----------|----------|
| `git` (directory) | One app per overlay directory |
| `git` (file) | Config-driven from JSON/YAML files |
| `list` | Explicit list of clusters/envs |
| `cluster` | One app per registered cluster |
| `matrix` | Combine two generators (e.g., cluster x app) |
| `merge` | Override specific params per combination |

## Sync Waves and Hooks

### Sync Waves (Ordering)

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"    # Lower = earlier
```

Order: CRDs (wave -2) -> Namespaces (wave -1) -> Infrastructure (wave 0) -> Apps (wave 1) -> Post-install (wave 2)

### Sync Hooks

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync       # Before sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

| Hook | When | Use Case |
|------|------|----------|
| `PreSync` | Before sync | DB migrations, schema changes |
| `Sync` | During sync | Normal resource sync |
| `PostSync` | After sync | Notifications, smoke tests |
| `SyncFail` | On failure | Cleanup, alert |

## ArgoCD Projects

Isolate teams/environments with AppProjects:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-backend
  namespace: argocd
spec:
  description: Backend team applications
  sourceRepos:
    - 'https://github.com/org/backend-gitops.git'
  destinations:
    - namespace: 'backend-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
  roles:
    - name: admin
      policies:
        - p, proj:team-backend:admin, applications, *, team-backend/*, allow
      groups:
        - backend-team
```

## Secret Management (External Secrets Operator)

Use ESO with one of: HashiCorp Vault, OpenBao, AWS Secrets Manager, or OCI Vault.

```yaml
# ClusterSecretStore (one per cluster)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: https://vault.example.com    # or OpenBao endpoint
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: external-secrets

---
# ExternalSecret (per workload)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: my-app-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-url
      remoteRef:
        key: apps/my-app
        property: database-url
```

### Rules

- **NEVER** commit plaintext secrets to Git
- **NEVER** store secret values in kustomization.yaml literals
- **NEVER** use static credentials for ESO auth in production — use Kubernetes auth or IRSA
- Use `ClusterSecretStore` for shared backends, `SecretStore` for namespace-scoped
- Always set `refreshInterval` — do not rely on manual syncs
- Use `creationPolicy: Owner` to avoid orphaned secrets
- Separate secret paths by environment and application (`apps/<env>/<app>`)

## Reconciliation Best Practices

- Set `prune: true` for garbage collection of removed resources
- Use `selfHeal: true` to auto-correct manual drift
- Use sync waves to enforce reconciliation order
- Configure retry with exponential backoff for transient failures
- Set `ServerSideApply=true` for large or CRD-heavy manifests
- Use `PrunePropagationPolicy=foreground` for clean deletions
- Use health checks and `argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true` for CRDs

## Notifications

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]
  template.app-sync-succeeded: |
    message: |
      Application {{.app.metadata.name}} synced successfully.
      Revision: {{.app.status.sync.revision}}
```

## DO NOT

- Manually `kubectl apply` in ArgoCD-managed namespaces (causes drift)
- Use mutable image tags (`latest`, `dev`) in production overlays
- Store ArgoCD credentials in the same repo it manages
- Auto-sync production without approval gates
- Skip health checks on critical infrastructure components
- Use `replace=true` sync option unless absolutely necessary (deletes and recreates)
- Put secrets in Application `spec.source.helm.parameters`

## Validation Commands

```bash
# Check ArgoCD app status
argocd app get <app-name>

# List all apps and sync status
argocd app list

# Diff what would change
argocd app diff <app-name>

# Manual sync (when auto-sync is off)
argocd app sync <app-name>

# Hard refresh (clear cache)
argocd app get <app-name> --hard-refresh

# Validate kustomize overlays
kustomize build overlays/prod/ | kubeconform -verbose
```

## References

See `references/` directory for:
- `promotion-workflow.md` -- Step-by-step promotion guide with ArgoCD
- `secret-management-comparison.md` -- ESO setup for Vault, OpenBao, AWS Secrets Manager, and OCI Vault
