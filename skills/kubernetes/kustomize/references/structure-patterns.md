# Kustomize Structure Patterns

## Pattern 1: Simple App (Single Component)

```
app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── resources.yaml
    ├── qa/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── resources.yaml
    └── prod/
        ├── kustomization.yaml
        └── patches/
            ├── resources.yaml
            ├── replicas.yaml
            └── hpa.yaml
```

## Pattern 2: Multi-Component App

```
platform/
├── base/
│   ├── api/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── worker/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── database/
│   │   ├── kustomization.yaml
│   │   └── statefulset.yaml
│   └── cache/
│       ├── kustomization.yaml
│       └── deployment.yaml
├── components/
│   ├── monitoring/
│   │   └── kustomization.yaml
│   ├── hpa/
│   │   └── kustomization.yaml
│   └── network-policies/
│       └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml   # references: base/api, base/worker
    ├── qa/
    │   └── kustomization.yaml   # + components/monitoring
    └── prod/
        └── kustomization.yaml   # + components/monitoring, hpa, network-policies
```

## Pattern 3: Multi-Cluster GitOps

```
clusters/
├── cluster-a/
│   ├── infrastructure/
│   │   ├── kustomization.yaml
│   │   └── sources/
│   └── apps/
│       ├── kustomization.yaml
│       └── patches/
└── cluster-b/
    ├── infrastructure/
    └── apps/

infrastructure/
├── base/
│   ├── cert-manager/
│   ├── ingress-nginx/
│   ├── external-secrets/
│   └── monitoring/
└── overlays/
    ├── cluster-a/
    └── cluster-b/

apps/
├── base/
│   ├── app-a/
│   └── app-b/
└── overlays/
    ├── cluster-a/
    │   ├── dev/
    │   └── prod/
    └── cluster-b/
        └── prod/
```

## Overlay Example: Dev

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: app-dev

resources:
  - ../../base/api
  - ../../base/worker

commonLabels:
  environment: dev

images:
  - name: api
    newName: registry.example.com/api
    newTag: dev-latest
  - name: worker
    newName: registry.example.com/worker
    newTag: dev-latest

patches:
  - target:
      kind: Deployment
    patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: not-important
      spec:
        replicas: 1
        template:
          spec:
            containers:
              - name: app
                resources:
                  requests:
                    memory: "128Mi"
                    cpu: "100m"
                  limits:
                    memory: "256Mi"
                    cpu: "200m"

configMapGenerator:
  - name: env-config
    behavior: merge
    envs:
      - config.env
```

## Overlay Example: Production

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: app-prod

resources:
  - ../../base/api
  - ../../base/worker
  - ../../base/database

components:
  - ../../components/monitoring
  - ../../components/hpa
  - ../../components/network-policies

commonLabels:
  environment: prod

images:
  - name: api
    newName: registry.example.com/api
    newTag: "1.5.2"
  - name: worker
    newName: registry.example.com/worker
    newTag: "1.5.2"

patches:
  - path: patches/api-resources.yaml
  - path: patches/worker-resources.yaml
  - path: patches/pdb.yaml

configMapGenerator:
  - name: env-config
    behavior: merge
    envs:
      - config.env
```

## Sources

- [Kubernetes Kustomize Documentation](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [FluxCD Repository Structure Guide](https://fluxcd.io/flux/guides/repository-structure/)
- [Kustomize Best Practices (OpenAnalytics)](https://www.openanalytics.eu/blog/2021/02/23/kustomize-best-practices/)
