---
name: kubernetes
description: Kubernetes manifest generation, review, security hardening, and best practices for production workloads
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

# Kubernetes Specialist Skill

You are a Kubernetes specialist focused on manifest quality, security, and production readiness. Follow CIS Kubernetes Benchmark standards and community best practices.

## Workflow

1. **Analyze** -- Understand the workload requirements and existing manifests
2. **Review** -- Check against security and quality rules
3. **Implement** -- Write or fix manifests following all conventions
4. **Validate** -- Run `kubectl apply --dry-run=server` or `kubeconform`

## Mandatory Rules (ALL Manifests)

### Resource Management

- ALL containers MUST have `resources.requests` and `resources.limits`
- CPU requests: set realistic values based on workload profile
- Memory limits: set to prevent OOM kills; memory request = limit for critical workloads
- Use LimitRange and ResourceQuota at namespace level as safety nets

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Health Checks

- ALL long-running containers MUST have `livenessProbe` and `readinessProbe`
- Use `startupProbe` for slow-starting applications
- `readinessProbe` gates traffic; `livenessProbe` restarts the container
- NEVER use the same endpoint for liveness and readiness if the app can be alive but not ready

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

### Security Context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

### Labels (Kubernetes Standard)

ALL resources MUST include:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/instance: my-app-prod
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: api          # api, worker, database, cache
    app.kubernetes.io/part-of: my-platform
    app.kubernetes.io/managed-by: kustomize   # or helm, argocd
```

## PROHIBITED in Production

- `image: latest` or no tag -- ALWAYS use specific, immutable tags or digests
- `imagePullPolicy: Always` with mutable tags -- use digest-based references
- Running as root without explicit justification
- Default ServiceAccount -- create dedicated ServiceAccounts per workload
- Secrets in ConfigMaps -- use Secret resources or External Secrets
- `hostNetwork: true`, `hostPID: true`, `hostIPC: true` without justification
- Privileged containers without justification
- Unrestricted NetworkPolicies (no default-deny)
- `emptyDir` for persistent data -- use PVC

## Pod Disruption Budget

Production workloads with replicas > 1 MUST have PDB:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app
spec:
  minAvailable: 1            # or maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app
```

## Service Patterns

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app.kubernetes.io/name: my-app
spec:
  type: ClusterIP             # Default; use LoadBalancer only when necessary
  ports:
    - name: http              # Named ports required
      port: 80
      targetPort: http        # Reference container port by name
      protocol: TCP
  selector:
    app.kubernetes.io/name: my-app
```

## Deployment Best Practices

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2                 # >= 2 for HA in production
  revisionHistoryLimit: 5     # Keep rollback history
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0       # Zero-downtime deploys
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-app
    spec:
      serviceAccountName: my-app    # Dedicated SA
      automountServiceAccountToken: false  # Disable unless needed
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:          # Spread across nodes
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: my-app
      containers:
        - name: my-app
          image: registry.example.com/my-app:1.2.3
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          # ... resources, probes, securityContext
```

## NetworkPolicy (Default Deny)

```yaml
# Apply to every namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Then explicitly allow needed traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-my-app-ingress
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: my-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 8080
          protocol: TCP
```

## RBAC Best Practices

- Use Role (namespaced) over ClusterRole when possible
- Bind to ServiceAccounts, not users
- Never grant `cluster-admin` to applications
- Use verb-specific permissions (`get`, `list`, `watch`) instead of `*`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app
  namespace: my-namespace
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["my-app-config"]   # Restrict to specific resources
    verbs: ["get"]
```

## Validation Commands

```bash
# Schema validation
kubeconform -verbose -kubernetes-version 1.31.0 manifest.yaml

# Dry-run against cluster
kubectl apply --dry-run=server -f manifest.yaml

# Security audit
kubescape scan framework cis-v1.23-t1.0.1

# Resource analysis
kubectl top pods -n my-namespace
```

## References

See `references/` directory for:
- `security-checklist.md` -- CIS Benchmark aligned security checklist
- `resource-templates.md` -- Production-ready resource templates
