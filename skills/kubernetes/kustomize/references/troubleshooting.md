# Kustomize Troubleshooting Guide

## Common Issues and Solutions

### 1. "resource not found" when referencing a base

**Symptom:** `Error: accumulating resources: ...`

**Causes and fixes:**
- Missing `kustomization.yaml` in the referenced directory
- Typo in the resource path (paths are relative to the kustomization.yaml location)
- Resource YAML has syntax errors

```bash
# Debug: verify the base builds independently
kustomize build base/<component>
```

### 2. Patch target not found

**Symptom:** `Error: no matches for target`

**Causes:**
- Resource name in patch doesn't match the actual resource name
- `namePrefix`/`nameSuffix` applied BEFORE patches -- patch must target the ORIGINAL name
- Resource kind mismatch (Deployment vs StatefulSet)

```yaml
# WRONG: if namePrefix is "dev-", don't target "dev-my-app"
patches:
  - target:
      kind: Deployment
      name: dev-my-app  # Wrong!

# CORRECT: target the original name
patches:
  - target:
      kind: Deployment
      name: my-app  # Original name from base
```

### 3. ConfigMap/Secret hash not propagated

**Symptom:** Pods not restarting after ConfigMap change

**Causes:**
- Using `disableNameSuffixHash: true` (intentional or accidental)
- Referencing ConfigMap by hardcoded name instead of Kustomize-managed name
- Using `envFrom` or `volumes` with a name that doesn't match the generator name

**Fix:** Ensure ConfigMap/Secret references in Deployments match the generator name exactly (without hash).

### 4. Namespace not applied to all resources

**Symptom:** Some resources deployed to wrong namespace

**Causes:**
- Namespace set directly in resource YAML overrides the transformer
- CRDs may not be recognized by the namespace transformer

**Fix:**
```yaml
# Remove namespace from resource YAMLs
# Set it only in kustomization.yaml:
namespace: my-namespace

# For CRDs, add configuration:
configurations:
  - namespace-config.yaml
```

### 5. Strategic merge patch deletes instead of merging

**Symptom:** Containers or volumes disappear after patch

**Cause:** Strategic merge patches merge by key. For containers, the key is `name`. If the patch container name doesn't match, it's treated as a replacement.

```yaml
# WRONG: wrong container name
spec:
  template:
    spec:
      containers:
        - name: wrong-name  # Must match exactly
          resources: ...

# CORRECT: match the container name from base
spec:
  template:
    spec:
      containers:
        - name: my-app  # Exact match
          resources:
            requests:
              memory: "512Mi"
```

### 6. "already registered id" conflict

**Symptom:** `Error: may not add resource with an already registered id`

**Cause:** Same resource defined in multiple bases or overlays referenced together.

**Fix:** Restructure to avoid duplicate resource definitions. Use a shared base instead.

### 7. Image tag not updating

**Symptom:** Kustomize build shows old image tag

**Causes:**
- `images` transformer matches by `name` field -- must match exactly what's in the Deployment
- Init containers need separate image entries
- Sidecar containers may use different image names

```yaml
# Check what image names are in your resources:
images:
  - name: nginx          # Must match spec.containers[].image prefix
    newName: my-registry/nginx
    newTag: "1.25"
```

## Debugging Commands

```bash
# Build and inspect full output
kustomize build overlays/dev | less

# Validate against schemas
kustomize build overlays/dev | kubeconform -verbose

# Compare environments
diff <(kustomize build overlays/dev) <(kustomize build overlays/prod)

# Check what resources exist
kustomize build overlays/dev | grep -E "^kind:|^  name:"

# Dry-run against cluster
kustomize build overlays/dev | kubectl apply --dry-run=server -f - 2>&1

# See diff against live state
kustomize build overlays/dev | kubectl diff -f -
```

## Sources

- [Kustomize GitHub Issues](https://github.com/kubernetes-sigs/kustomize/issues)
- [Kustomize FAQ](https://kubectl.docs.kubernetes.io/faq/kustomize/)
