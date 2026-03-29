---
name: helm
description: Helm chart creation, management, values organization, and best practices following official Helm conventions
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

# Helm Chart Engineer Skill

You are a Helm chart specialist. Follow the official Helm best practices guide and community conventions.

## Workflow

1. **Analyze** -- Understand chart requirements and existing values
2. **Research** -- Check Artifact Hub / Helm repos for existing charts before writing from scratch
3. **Implement** -- Write charts following all conventions
4. **Validate** -- Run `helm lint`, `helm template`, and `helm install --dry-run`

## Chart Structure

```
my-chart/
├── Chart.yaml              # Chart metadata (required)
├── Chart.lock              # Dependency lock file
├── values.yaml             # Default configuration (required)
├── values.schema.json      # Input validation schema (recommended)
├── templates/              # Kubernetes manifests
│   ├── _helpers.tpl        # Template helpers and partials
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── NOTES.txt           # Post-install notes
│   └── tests/
│       └── test-connection.yaml
├── charts/                 # Dependency charts
├── crds/                   # Custom Resource Definitions
└── README.md               # Documentation
```

## Naming Conventions

- **Chart names**: lowercase, words separated by dashes (`my-app`, NOT `MyApp` or `my_app`)
- **No underscores or dots** in chart names
- **Variable names**: lowerCamelCase (`serverPort`, NOT `server-port` or `ServerPort`)
- **Template names**: `<chart-name>.<template-name>` (e.g., `my-app.fullname`)

## Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: A Helm chart for My Application
type: application                    # or library
version: 1.2.3                       # Chart version (SemVer 2)
appVersion: "2.1.0"                  # Application version
kubeVersion: ">= 1.28.0"           # Kubernetes version constraint
home: https://github.com/org/my-app
sources:
  - https://github.com/org/my-app
maintainers:
  - name: Team Platform
    email: platform@example.com
dependencies:
  - name: postgresql
    version: "~13.0"                 # Patch-level range
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

## Versioning

- Follow **SemVer 2** (MAJOR.MINOR.PATCH)
  - **MAJOR**: Breaking changes (removed values, renamed keys, changed defaults that break)
  - **MINOR**: New features, new values (backwards-compatible)
  - **PATCH**: Bug fixes (backwards-compatible)
- `version` (chart) and `appVersion` (application) are independent
- Use version ranges for dependencies: `~1.2.3` (patch), `^1.2.3` (minor), `>= 1.0.0`

## values.yaml Organization

### Structure Rules

- **Flat over nested** for simple values:
  ```yaml
  # GOOD: flat
  serverName: nginx
  serverPort: 80

  # GOOD: nested only for large related groups
  server:
    name: nginx
    port: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
  ```

- **Quote ALL strings** to avoid YAML type coercion:
  ```yaml
  # DANGEROUS: YAML interprets these as non-string
  enabled: true       # boolean
  count: 1234         # integer
  version: 1.0        # float

  # SAFE: explicit strings
  nodeSelector: "true"
  appVersion: "1.0"
  ```

- **Document every property** with comments:
  ```yaml
  # -- Number of replicas for the deployment
  replicaCount: 1

  # -- Container image configuration
  image:
    # -- Image repository
    repository: nginx
    # -- Image pull policy
    pullPolicy: IfNotPresent
    # -- Image tag (defaults to chart appVersion)
    tag: ""
  ```

- **Override-friendly**: Prefer maps over arrays for `--set` compatibility:
  ```yaml
  # BAD: arrays are hard to override with --set
  env:
    - name: FOO
      value: bar

  # GOOD: map is easy to override
  env:
    FOO: bar
  ```

### Standard values.yaml Template

```yaml
# -- Number of replicas
replicaCount: 1

image:
  # -- Image repository
  repository: my-app
  # -- Image pull policy
  pullPolicy: IfNotPresent
  # -- Image tag override (defaults to appVersion)
  tag: ""

# -- Image pull secrets
imagePullSecrets: []

# -- Override chart name
nameOverride: ""
# -- Override full name
fullnameOverride: ""

serviceAccount:
  # -- Create service account
  create: true
  # -- Service account annotations
  annotations: {}
  # -- Service account name override
  name: ""
  # -- Automount service account token
  automount: false

# -- Pod annotations
podAnnotations: {}

# -- Pod security context
podSecurityContext:
  runAsNonRoot: true
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

# -- Container security context
securityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  runAsUser: 1000
  capabilities:
    drop:
      - ALL

service:
  # -- Service type
  type: ClusterIP
  # -- Service port
  port: 80

ingress:
  # -- Enable ingress
  enabled: false
  # -- Ingress class name
  className: ""
  # -- Ingress annotations
  annotations: {}
  # -- Ingress hosts
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  # -- Ingress TLS configuration
  tls: []

# -- Resource requests and limits
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

autoscaling:
  # -- Enable HPA
  enabled: false
  # -- Minimum replicas
  minReplicas: 2
  # -- Maximum replicas
  maxReplicas: 10
  # -- Target CPU utilization
  targetCPUUtilizationPercentage: 70
  # -- Target memory utilization
  targetMemoryUtilizationPercentage: 80

# -- Node selector
nodeSelector: {}

# -- Tolerations
tolerations: []

# -- Affinity rules
affinity: {}

# -- Topology spread constraints
topologySpreadConstraints: []
```

## Template Best Practices

### _helpers.tpl

```yaml
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### DO NOT in Templates

- Do NOT set `namespace` in templates -- leave it to the deployer (`helm install --namespace`)
- Do NOT use `helm.sh/hook` for critical data migrations without backup plan
- Do NOT hardcode image registries -- parameterize in values.yaml
- Do NOT use `lookup` function in templates (breaks `helm template`)

## Dependencies

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "~13.0"               # Patch-level matching (recommended default)
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled   # Toggle dependency
  - name: redis
    version: "^18.0"               # Minor-level matching
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

```bash
helm dependency update    # Fetch and lock dependencies
helm dependency build     # Build from lock file
```

## Validation Commands

```bash
# Lint chart
helm lint ./my-chart

# Template rendering (no cluster needed)
helm template my-release ./my-chart --values values-prod.yaml

# Dry-run against cluster
helm install my-release ./my-chart --dry-run --debug

# Diff before upgrade (requires helm-diff plugin)
helm diff upgrade my-release ./my-chart --values values-prod.yaml

# Schema validation
helm template ./my-chart | kubeconform -verbose
```

## References

See `references/` directory for:
- `conventions.md` -- Helm naming and formatting conventions quick reference
