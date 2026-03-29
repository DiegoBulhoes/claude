# Secret Management with External Secrets Operator (ESO)

## Architecture

```
Developer -> Secret Store (source of truth)
                    |
                    v
           ExternalSecret CRD
                    |
                    v
        ESO Controller (in-cluster)
                    |
                    v
           Kubernetes Secret
```

## Supported Backends

| Backend | Auth Methods | Notes |
|---------|-------------|-------|
| **HashiCorp Vault** | Kubernetes, Token, AppRole, LDAP | Most feature-rich, self-hosted |
| **OpenBao** | Kubernetes, Token, AppRole | Open-source Vault fork, same API |
| **AWS Secrets Manager** | IRSA, JWT, static credentials | Native AWS integration |
| **OCI Vault** | Instance Principal, User Principal | Oracle Cloud native |

## Pros

- Single source of truth (vault/secret store)
- Automatic rotation via `refreshInterval`
- Multi-tenancy via namespaced SecretStores
- No secrets in Git (not even encrypted)
- Audit trail in the secret store
- Works across clusters
- Unified API regardless of backend

## Cons

- Requires vault/secret store infrastructure
- Additional operator to maintain
- Network dependency on vault availability
- More complex initial setup

## ClusterSecretStore Examples

### HashiCorp Vault / OpenBao

```yaml
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
```

> OpenBao uses the same `vault` provider config — just point `server` to your OpenBao instance.

### AWS Secrets Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

### OCI Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: oci-vault
spec:
  provider:
    oracle:
      vault: <vault-ocid>
      region: <region>
      auth:
        secretRef:
          privatekey:
            name: oci-credentials
            key: privatekey
          fingerprint:
            name: oci-credentials
            key: fingerprint
        tenancy: <tenancy-ocid>
        user: <user-ocid>
```

## ExternalSecret Usage

Works the same regardless of backend:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: my-app
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault-backend          # reference any ClusterSecretStore
    kind: ClusterSecretStore
  target:
    name: my-app-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-url
      remoteRef:
        key: apps/my-app
        property: database-url
    - secretKey: api-key
      remoteRef:
        key: apps/my-app
        property: api-key
```

### Templated Secrets

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-connection
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-connection
    template:
      engineVersion: v2
      data:
        connection-string: "postgresql://{{ .user }}:{{ .password }}@{{ .host }}:5432/{{ .dbname }}"
  data:
    - secretKey: user
      remoteRef:
        key: apps/my-app/db
        property: username
    - secretKey: password
      remoteRef:
        key: apps/my-app/db
        property: password
    - secretKey: host
      remoteRef:
        key: apps/my-app/db
        property: host
    - secretKey: dbname
      remoteRef:
        key: apps/my-app/db
        property: dbname
```

### Fetch All Keys from a Path

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-all-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: my-app-all-secrets
  dataFrom:
    - extract:
        key: apps/my-app
```

## Multi-Environment Pattern

Use one ClusterSecretStore per cluster, with ExternalSecrets pointing to environment-specific paths:

```yaml
# base/my-app/external-secret.yaml
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
  dataFrom:
    - extract:
        key: apps/ENV_PLACEHOLDER/my-app    # patched per overlay
```

```yaml
# overlays/prod/kustomization.yaml
patches:
  - target:
      kind: ExternalSecret
      name: my-app-secrets
    patch: |
      - op: replace
        path: /spec/dataFrom/0/extract/key
        value: apps/prod/my-app
```

## Best Practices

- Use `ClusterSecretStore` for shared backends, `SecretStore` for namespace-scoped access
- Set `refreshInterval` based on sensitivity (15m for DB creds, 1h for API keys)
- Use `creationPolicy: Owner` so deleting ExternalSecret cleans up the K8s Secret
- Monitor ESO with metrics: `externalsecret_status_condition`, `externalsecret_sync_calls_total`
- Use `PushSecret` CRD to sync K8s secrets back to the vault (useful for generated certs)
- Separate secret paths by environment and application (`apps/<env>/<app>`)

## Rules

- **NEVER** commit plaintext secrets to Git
- **NEVER** store secret values in kustomization.yaml literals
- **NEVER** use static credentials for ESO auth in production — use Kubernetes auth or IRSA
- Always set `refreshInterval` — do not rely on manual syncs
- Always use `creationPolicy: Owner` to avoid orphaned secrets

## Sources

- [External Secrets Operator Docs](https://external-secrets.io/)
- [ESO Provider: HashiCorp Vault](https://external-secrets.io/latest/provider/hashicorp-vault/)
- [ESO Provider: AWS Secrets Manager](https://external-secrets.io/latest/provider/aws-secrets-manager/)
- [ESO Provider: Oracle](https://external-secrets.io/latest/provider/oracle-vault/)
- [OpenBao](https://openbao.org/)
- [ArgoCD Secret Management](https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/)
