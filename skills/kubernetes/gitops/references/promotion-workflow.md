# Environment Promotion Workflow (ArgoCD)

## Standard Promotion Flow

```
Build -> Dev (auto-sync) -> QA (PR) -> Prod (PR + approval)
```

### Step 1: CI Builds and Pushes Image

```bash
# CI pipeline tags image with Git SHA or SemVer
docker build -t registry.example.com/my-app:1.2.3 .
docker push registry.example.com/my-app:1.2.3
```

### Step 2: Update Dev Overlay (Auto-merge or Direct Push)

```yaml
# overlays/cluster/dev/my-app/kustomization.yaml
images:
  - name: my-app
    newName: registry.example.com/my-app
    newTag: "1.2.3"
```

ArgoCD auto-syncs dev. Verify deployment health:

```bash
argocd app get my-app-dev
argocd app wait my-app-dev --health
```

### Step 3: Promote to QA (PR)

```bash
# Create promotion PR
git checkout -b promote/my-app-1.2.3-to-qa

# Update QA overlay with same image tag
# overlays/cluster/qa/my-app/kustomization.yaml
```

PR triggers:
- `kustomize build` validation
- `kubeconform` schema check
- ArgoCD diff preview (via CI or `argocd app diff`)
- Reviewer approval (optional for QA)

### Step 4: Promote to Production (PR + Approval)

```bash
git checkout -b promote/my-app-1.2.3-to-prod

# Update prod overlay
# overlays/cluster/prod/my-app/kustomization.yaml
```

PR requires:
- `kustomize build` validation
- `kubeconform` schema check
- **Mandatory reviewer approval**
- **Successful QA deployment verification**
- Optional: change window check

### Step 5: Post-Deploy Verification

```bash
# Check ArgoCD sync status
argocd app get my-app-prod

# Check rollout status
kubectl rollout status deployment/my-app -n my-app-prod --timeout=300s

# Check pod health
kubectl get pods -n my-app-prod -l app.kubernetes.io/name=my-app

# Check logs for errors
kubectl logs -n my-app-prod -l app.kubernetes.io/name=my-app --tail=100

# Verify endpoints
kubectl get endpoints -n my-app-prod my-app
```

## Rollback Procedure

### Option 1: Git Revert (Preferred)

```bash
# Revert the promotion commit
git revert <promotion-commit-sha>
git push
# ArgoCD auto-syncs to previous state
```

### Option 2: ArgoCD Rollback

```bash
# List sync history
argocd app history my-app-prod

# Rollback to previous revision
argocd app rollback my-app-prod <history-id>

# Then update Git to match (important!)
```

### Option 3: Image Tag Update

```yaml
# Update overlay to previous known-good tag
# overlays/cluster/prod/my-app/kustomization.yaml
images:
  - name: my-app
    newName: registry.example.com/my-app
    newTag: "1.2.2"   # Previous version
```

## Automated Promotion with ArgoCD Image Updater

```yaml
# Application with image updater annotations
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: my-app=registry.example.com/my-app
    argocd-image-updater.argoproj.io/my-app.update-strategy: semver
    argocd-image-updater.argoproj.io/my-app.allow-tags: "regexp:^[0-9]+\\.[0-9]+\\.[0-9]+$"
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: kustomization
spec:
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: main
    path: overlays/cluster/dev/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Image Updater Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| `semver` | Latest SemVer tag | Production (stable releases) |
| `latest` | Most recently pushed | Dev (latest build) |
| `name` | Alphabetically last | Tags like `YYYY-MM-DD` |
| `digest` | Track digest changes | Mutable tag tracking |

## CI Validation for Promotion PRs

```yaml
# .github/workflows/validate-promotion.yaml
name: Validate Promotion
on:
  pull_request:
    paths:
      - 'overlays/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate all overlays
        run: |
          for dir in overlays/*/; do
            echo "Validating $dir..."
            kustomize build "$dir" | kubeconform -verbose -kubernetes-version 1.31.0
          done

  argocd-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: ArgoCD Diff
        uses: argoproj/argo-cd-action@v2
        with:
          command: app diff
          options: --app-name my-app-prod --local ./overlays/cluster/prod/my-app
```

## Sources

- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
