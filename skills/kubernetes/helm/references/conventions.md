# Helm Conventions Quick Reference

## Chart Naming

| Rule | Example |
|------|---------|
| Lowercase only | `my-app` not `MyApp` |
| Dashes for separators | `my-app` not `my_app` |
| No dots | `my-app` not `my.app` |
| SemVer 2 versioning | `1.2.3` |

## Variable Naming

| Rule | Example |
|------|---------|
| lowerCamelCase | `serverPort` not `server_port` |
| Start with lowercase | `replicaCount` not `ReplicaCount` |
| Boolean prefix | `enabled`, `create`, `use` |

## Template Naming

| Rule | Example |
|------|---------|
| Prefixed with chart name | `my-app.fullname` |
| Lowercase | `my-app.labels` not `my-app.Labels` |

## Label Standards

```yaml
# Required (Helm standard)
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ include "my-app.chart" . }}

# Recommended (Kubernetes standard)
app.kubernetes.io/component: api
app.kubernetes.io/part-of: platform
```

## YAML Formatting

- 2-space indentation (never tabs)
- Quote strings that could be misinterpreted: `"true"`, `"1.0"`, `"null"`, `"yes"`
- Use `|` for multi-line strings, `>-` for folded single-line
- Empty lines between logical sections

## Annotation vs Label

| Use | For |
|-----|-----|
| **Labels** | Selection, grouping, filtering (`kubectl get -l`) |
| **Annotations** | Non-identifying metadata, tool configuration, documentation |

## Common Hooks

| Hook | When |
|------|------|
| `pre-install` | Before any resources are installed |
| `post-install` | After all resources are installed |
| `pre-upgrade` | Before any resources are upgraded |
| `post-upgrade` | After all resources are upgraded |
| `pre-delete` | Before any resources are deleted |
| `post-delete` | After all resources are deleted |

## Dependency Version Ranges

| Syntax | Meaning | Use When |
|--------|---------|----------|
| `~1.2.3` | `>= 1.2.3, < 1.3.0` | Default; get patches automatically |
| `^1.2.3` | `>= 1.2.3, < 2.0.0` | Trust upstream for backwards-compat |
| `>= 1.0.0` | Any version >= 1.0.0 | Very flexible; risky in prod |
| `1.2.3` | Exact version | Maximum control; no auto-updates |

## Sources

- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Helm General Conventions](https://helm.sh/docs/chart_best_practices/conventions/)
- [Helm Values](https://helm.sh/docs/chart_best_practices/values/)
