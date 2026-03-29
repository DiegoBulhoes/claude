# Kubernetes Discovery Questions

When the user confirms the target is a Kubernetes environment, use this question tree to gather enough detail to produce a near-complete cluster specification. Ask in rounds — adapt based on answers, skip what's already answered.

---

## Round K1 — Cluster Foundation

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | New cluster or existing? | Determines if we provision infra or just workloads | — |
| 2 | Which distribution? (K3s, EKS, AKS, GKE, kubeadm, RKE2, OpenShift, Talos) | Affects networking defaults, storage, API flags | K3s |
| 3 | How many control-plane nodes? (1, 3, 5) | HA vs simplicity trade-off | 1 (dev), 3 (prod) |
| 4 | How many worker nodes? Autoscaling? | Capacity planning | 2 |
| 5 | Node instance type / arch? (x86, ARM, mixed) | Image compatibility, resource limits | x86 |
| 6 | Which cloud provider? (AWS, GCP, Azure, OCI, Hetzner, bare-metal) | Provider-specific modules, CSI, CCM | — |
| 7 | Managed or self-managed? | Operational burden, who patches kubelet | Managed if EKS/AKS/GKE, self-managed otherwise |

## Round K2 — Networking

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Which CNI? (Calico, Cilium, Flannel, Weave, cloud-native VPC CNI) | Network policies, performance, eBPF support | Calico |
| 2 | Pod CIDR? | Must not overlap with VPC/VCN CIDR | `10.42.0.0/16` |
| 3 | Service CIDR? | Must not overlap with pod or node CIDR | `10.43.0.0/16` |
| 4 | Which ingress controller? (nginx, Traefik, HAProxy, Istio gateway, Envoy Gateway, cloud ALB, none) | HTTP routing, TLS termination, annotations | nginx-ingress |
| 5 | Need a service mesh? (Istio, Linkerd, Cilium mesh, none) | mTLS, observability, traffic management | none |
| 6 | DNS provider for external records? (Cloudflare, Route53, Cloud DNS, Azure DNS, none) | external-dns setup | — |
| 7 | Need a tunnel / zero-trust access? (Cloudflare Tunnel, Tailscale, WireGuard, none) | Expose services without public LB | none |
| 8 | Load balancer type? (cloud LB, MetalLB, kube-vip, NodePort only) | How Services type=LoadBalancer work | Cloud LB if managed, MetalLB if bare-metal |

## Round K3 — Storage

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Which CSI / storage backend? (Longhorn, Rook-Ceph, EBS CSI, cloud block storage, local-path, none) | PVC provisioning, snapshots, backup | Cloud default if managed, Longhorn if self-managed |
| 2 | Need object storage? (MinIO, S3, GCS, Azure Blob, none) | For Loki, Tempo, Mimir, backups | Cloud-native if available, MinIO if self-hosted |
| 3 | Default StorageClass? | What PVCs get when no class specified | CSI default |
| 4 | Backup strategy? (Velero, CSI snapshots, etcd snapshots, cloud snapshots, none) | Disaster recovery | etcd snapshots |
| 5 | Retention / size limits? | Cost, disk pressure | 7 days, 50Gi total |

## Round K4 — Security & Secrets

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Secrets management? (Vault/OpenBao, AWS SM, Azure KV, GCP SM, SOPS, sealed-secrets, none) | How apps get credentials | — |
| 2 | External Secrets Operator (ESO) or native CSI secrets driver? | Secret sync mechanism | ESO |
| 3 | TLS certificate management? (cert-manager, cloud-managed, manual) | HTTPS automation | cert-manager |
| 4 | Certificate issuer? (Let's Encrypt, private CA, cloud CA) | ClusterIssuer config | Let's Encrypt (DNS-01) |
| 5 | Network policies? (deny-all-by-default, namespace-isolated, open) | Security posture | namespace-isolated |
| 6 | Pod security standards? (restricted, baseline, privileged) | PSA enforcement | baseline |
| 7 | RBAC model? (namespace-scoped, cluster-admin only, OIDC-integrated) | Access control | namespace-scoped |
| 8 | Image scanning / admission control? (Trivy, Kyverno, OPA/Gatekeeper, none) | Supply chain security | none (add later) |

## Round K5 — GitOps & Deployment

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | GitOps tool? (ArgoCD, Flux, none — just CI/CD apply) | Deployment model | ArgoCD |
| 2 | App of Apps, ApplicationSets, or flat? | Multi-app management pattern | App of Apps |
| 3 | Manifest format? (Helm, Kustomize, raw YAML, mix) | Templating strategy | Helm + Kustomize overlay |
| 4 | Mono-repo or multi-repo? | Where app manifests live | Mono-repo |
| 5 | Auto-sync or manual sync? | How changes propagate | Auto-sync with prune + self-heal |
| 6 | Environment promotion strategy? (overlay per env, branch per env, ApplicationSet matrix) | Multi-env support | Kustomize overlays |
| 7 | Need sync waves / ordering? | Dependency management between apps | Yes — waves 0-5 |

## Round K6 — Observability

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Monitoring stack? (Prometheus + Grafana, Datadog, New Relic, cloud-native, none) | Metrics collection and visualization | kube-prometheus-stack |
| 2 | Logs? (Loki, EFK/ELK, CloudWatch, cloud logging, none) | Log aggregation | Loki |
| 3 | Traces? (Tempo, Jaeger, X-Ray, cloud tracing, none) | Distributed tracing | Tempo |
| 4 | OpenTelemetry? (otel-operator + collector, SDK-only, none) | Instrumentation strategy | otel-operator |
| 5 | Long-term metrics storage? (Mimir, Thanos, Cortex, cloud, Prometheus only) | Retention beyond 15 days | Prometheus only |
| 6 | Alerting? (Alertmanager, PagerDuty, Opsgenie, Slack, email) | Incident notification | Alertmanager |
| 7 | Dashboard requirements? (pre-built, custom, both) | Grafana setup | Pre-built + custom |
| 8 | SLO tracking? (Sloth, Pyrra, manual, none) | Reliability targets | none (add later) |

## Round K7 — Data & Workloads

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Need a database? (PostgreSQL, MySQL, Redis, MongoDB — operator or Helm) | Stateful workload planning | — |
| 2 | Message queue? (RabbitMQ, Kafka, NATS, none) | Async communication | none |
| 3 | Workflow / scheduler? (Airflow, Argo Workflows, Temporal, none) | Data pipeline needs | none |
| 4 | CI runner inside cluster? (GitHub Actions runner, GitLab runner, Tekton, none) | Build inside K8s | none |
| 5 | Any specific apps to deploy? | Drives the app list | — |

## Round K8 — Operations & Cost

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Budget constraints? (free tier, monthly limit, no limit) | Instance types, replica counts | — |
| 2 | HA requirements? (single-node OK, multi-AZ, multi-region) | Architecture complexity | single-AZ multi-node |
| 3 | Disaster recovery? (RTO/RPO targets, or best-effort) | Backup + restore planning | best-effort |
| 4 | Who operates? (single SRE, team, fully automated) | Runbook depth, alerting sensitivity | single SRE |
| 5 | Node maintenance strategy? (drain + cordon, rolling update, immutable) | Upgrade workflow | drain + cordon |

---

## Usage Notes

- **Don't ask all questions** — use the defaults column and skip what's obvious from context
- **Group by relevance** — if user says "EKS + ArgoCD + Helm" upfront, skip K1/K5 and focus on K2-K4/K6
- **Offer defaults** — present smart defaults and let the user override: "I'll assume Calico for CNI, nginx for ingress, and EBS CSI for storage — any changes?"
- **Derive from existing infra** — if `CLAUDE.md` or existing files answer a question, don't re-ask it. State your assumption and move on
- **Respect the round limit** — even with all these questions available, the interview should be 3-5 rounds max for standalone mode, 2-3 for subagent mode. Batch related questions together
