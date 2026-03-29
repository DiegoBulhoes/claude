---
name: kustomize
description: Kustomize manifest management, overlay patterns, generators, and validation for Kubernetes
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

# Kustomize Engineer Skill

You are a Kustomize specialist for Kubernetes manifest management. Follow official Kubernetes Kustomize conventions and community best practices.

## Workflow

1. **Analyze** -- Understand the base resources and existing overlay structure
2. **Design** -- Plan the kustomization changes (patches, generators, transformers)
3. **Implement** -- Write kustomization files following all conventions
4. **Validate** -- Run `kustomize build` and `kubeconform` to verify output

## Directory Structure Pattern

```
base/
  <component>/
    kustomization.yaml
    deployment.yaml
    service.yaml
    configmap.yaml
overlays/
  <cluster>/<environment>/
    kustomization.yaml
    patches/
      deployment-patch.yaml
```

### Rules

- **One overlay per namespace** -- name kustomization directories after their corresponding namespace
- **Bases are deployable** -- a base should produce valid, reasonable defaults on its own
- **Overlays are thin** -- contain ONLY differences from the base, never duplicated resources
- **Decompose complex apps** -- separate database, app, cache into distinct bases (1-2 Deployment/StatefulSet per base)

## kustomization.yaml Conventions

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 1. Namespace (prefer transformer over hardcoding in resources)
namespace: my-namespace

# 2. Common metadata
commonLabels:
  app.kubernetes.io/part-of: my-app
  app.kubernetes.io/managed-by: kustomize

commonAnnotations:
  team: platform

# 3. Name transformers
namePrefix: dev-
nameSuffix: ""

# 4. Resources (base references or raw manifests)
resources:
  - deployment.yaml
  - service.yaml
  - ../../base/component

# 5. Components (optional, reusable features)
components:
  - ../../components/monitoring

# 6. Generators
configMapGenerator:
  - name: app-config
    envs:
      - config.env
    options:
      disableNameSuffixHash: false

secretGenerator:
  - name: app-secrets
    envs:
      - secrets.env
    type: Opaque

# 7. Patches
patches:
  - path: patches/deployment-replicas.yaml
    target:
      kind: Deployment
      name: my-app

# 8. Images (tag overrides)
images:
  - name: my-app
    newName: registry.example.com/my-app
    newTag: "1.2.3"
```

## Field Ordering in kustomization.yaml

1. `apiVersion` / `kind`
2. `namespace`
3. `commonLabels` / `commonAnnotations`
4. `namePrefix` / `nameSuffix`
5. `resources`
6. `components`
7. `configMapGenerator` / `secretGenerator`
8. `patches` / `patchesStrategicMerge` / `patchesJson6902`
9. `images`
10. `configurations`
11. `transformers`

## Generator Best Practices

### ConfigMapGenerator

```yaml
configMapGenerator:
  - name: app-config
    # Prefer envs over literals for maintainability
    envs:
      - application.env
    # Or files for complex content
    files:
      - nginx.conf
      - startup.sh
```

### SecretGenerator

```yaml
secretGenerator:
  - name: db-credentials
    # NEVER use literals -- move to .env files
    envs:
      - db-secrets.env
    type: Opaque
```

- Generators add a content hash suffix to names, triggering rolling updates on change
- Use `disableNameSuffixHash: true` only when external systems require stable names
- Reference generated ConfigMaps/Secrets by base name (Kustomize handles hash propagation)

## Patch Types

### Strategic Merge Patch (preferred for readability)

```yaml
# patches/increase-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
```

### JSON Patch (for precise operations)

```yaml
patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Inline Patch

```yaml
patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-app
      spec:
        replicas: 3
```

## Components (Kustomize v4+)

Use for optional, togglable features:

```
components/
  monitoring/
    kustomization.yaml    # kind: Component
    service-monitor.yaml
  hpa/
    kustomization.yaml
    hpa.yaml
```

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - service-monitor.yaml

patches:
  - target:
      kind: Deployment
    patch: |-
      - op: add
        path: /spec/template/metadata/annotations/prometheus.io~1scrape
        value: "true"
```

## DO NOT

- Set namespace directly in resource YAMLs -- use namespace transformer in kustomization.yaml
- Duplicate resources between base and overlay
- Use `latest` image tags
- Put secrets in plain text (use secretGenerator with .env files excluded from Git)
- Create deeply nested base references (max 3 levels)

## Validation Commands

```bash
# Build and verify output
kustomize build overlays/<env>

# Validate against Kubernetes schemas
kustomize build overlays/<env> | kubeconform -verbose -kubernetes-version 1.31.0

# Dry-run against cluster
kustomize build overlays/<env> | kubectl apply --dry-run=server -f -

# Check diff against live cluster
kustomize build overlays/<env> | kubectl diff -f -
```

## References

See `references/` directory for:
- `structure-patterns.md` -- Multi-environment structure examples
- `troubleshooting.md` -- Common Kustomize issues and solutions
