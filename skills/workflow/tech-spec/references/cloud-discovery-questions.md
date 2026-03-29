# Cloud Infrastructure Discovery Questions

When the user confirms the target involves cloud infrastructure, use this question tree to gather enough detail to produce a near-complete specification. Ask in rounds — adapt based on answers, skip what's already answered.

Provider-specific examples are noted as **(AWS)** / **(OCI)** where relevant, but the questions apply to any cloud.

---

## Round C1 — Account & Organization

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Which cloud provider? (AWS, OCI, GCP, Azure, multi-cloud) | Provider-specific services, Terraform providers | — |
| 2 | Single account/tenancy or multi-account? | Isolation, billing, IAM boundaries | Single account |
| 3 | Resource hierarchy? (AWS: Organizations + OUs / OCI: Compartments / GCP: Folders + Projects) | Governance, policy inheritance | Root + one child per environment |
| 4 | Which region(s)? | Latency, compliance, data residency, service availability | — |
| 5 | Multi-region or single-region? | DR, latency requirements | Single-region |
| 6 | Landing zone / account factory? (AWS: Control Tower / OCI: Landing Zone / GCP: Fabric FAST) | Account provisioning, guardrails | none |
| 7 | IaC state backend? (S3+DynamoDB, OCI Object Storage, GCS, Terraform Cloud) | Terraform remote state | Cloud-native object storage |
| 8 | Existing network or greenfield? | Constraints on CIDR, peering, existing resources | — |

## Round C2 — Networking

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | VPC/VCN CIDR? | IP space planning, peering constraints | `10.0.0.0/16` |
| 2 | How many availability zones/domains? (2, 3) | HA level, cost | 2 (or all ADs in OCI region) |
| 3 | Subnet strategy? (public + private, public + private + isolated) | Security posture, NAT needs | public + private |
| 4 | NAT for private subnets? (NAT Gateway, single or per-AZ) | Outbound internet for private subnets | Single NAT Gateway |
| 5 | Service endpoint / gateway? (AWS: VPC Endpoints / OCI: Service Gateway / GCP: Private Google Access) | Access cloud services without traversing internet | Yes for storage + DB |
| 6 | Cross-network connectivity? (VPC Peering, Transit Gateway, DRG, PrivateLink) | Cross-VPC / cross-account / hybrid | none |
| 7 | DNS? (Route53, OCI DNS, Cloud DNS, external) | Domain management, service discovery | Cloud-native DNS |
| 8 | VPN or dedicated link? (Site-to-Site VPN, Direct Connect, FastConnect) | Hybrid connectivity | none |
| 9 | Flow logs? (VPC Flow Logs, VCN Flow Logs) | Network troubleshooting, compliance | Enabled (reject-only) |
| 10 | Firewall model? (Security Groups, NSGs, firewall rules — per-service or per-tier) | Traffic filtering granularity | Per-service |

## Round C3 — Compute

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Compute model? (VMs, managed K8s, serverless, containers, mix) | Architecture pattern | — |
| 2 | Instance type / shape? (AWS: instance family / OCI: Flex shapes / general, compute, memory, ARM) | Cost vs performance | General purpose or ARM if compatible |
| 3 | ARM architecture? (AWS: Graviton / OCI: Ampere A1) | Cost savings (20-40% cheaper), energy efficiency | Yes if workload supports it |
| 4 | Spot / preemptible instances? (for workers, batch, CI runners) | Cost optimization | none |
| 5 | Autoscaling? (ASG, instance pool autoscaling, node pool autoscaler, HPA) | Elasticity | Target-tracking autoscaling |
| 6 | OS image? (Amazon Linux, Oracle Linux, Ubuntu, Bottlerocket, custom) | Patching, compatibility | Provider default Linux |
| 7 | Remote access strategy? (AWS: SSM Session Manager / OCI: Bastion Service / SSH bastion / none) | Security, audit trail | Managed bastion service |

## Round C4 — Storage & Data

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Database engine? (managed relational, autonomous, NoSQL, in-memory cache, none) | Data tier | — |
| 2 | Managed DB specifics? (AWS: RDS/Aurora / OCI: Autonomous DB/DB System / engine: PostgreSQL, MySQL, Oracle) | Service selection | — |
| 3 | DB high availability? (Multi-AZ, read replicas, standby, RAC) | HA, read scaling | Multi-AZ/standby for prod |
| 4 | Object storage needs? (what for — assets, logs, backups, data lake?) | Bucket design | — |
| 5 | Object storage lifecycle? (archival tier, expiration, versioning) | Cost management | Versioning on, archive after 90d |
| 6 | Shared filesystem? (AWS: EFS / OCI: FSS / NFS) | Multi-instance/pod mounts | none |
| 7 | Block storage performance tier? (balanced, high IOPS, throughput) | Compute-attached storage | Balanced |
| 8 | Backup strategy? (cloud-native backup service, DB automatic backups, Velero, custom) | DR, compliance | Cloud-native automatic backups |
| 9 | Encryption at rest? (provider-managed keys, CMK, HSM) | Compliance, key management | Provider-managed keys |

## Round C5 — Security & IAM

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | IAM model? (AWS: IAM roles + Identity Center / OCI: Identity Domains / OIDC federation) | Access control | Cloud-native IAM |
| 2 | Workload identity? (AWS: IRSA / OCI: Instance Principals + Workload Identity / GCP: Workload Identity) | Service-to-cloud API auth without static creds | Yes |
| 3 | Secrets management? (AWS: Secrets Manager / OCI: Vault / HashiCorp Vault / external) | Credential storage | Cloud-native secrets service |
| 4 | Encryption key management? (provider-managed, CMK, HSM-backed) | Key control level | Provider-managed |
| 5 | WAF? (cloud WAF, Cloudflare, none) | Web application protection | none |
| 6 | Threat detection? (AWS: GuardDuty / OCI: Cloud Guard / GCP: Security Command Center) | Security posture | Enabled |
| 7 | Vulnerability scanning? (compute images, container images, both, none) | Security posture | Compute scanning |
| 8 | Preventive guardrails? (AWS: SCPs / OCI: tag-based policies / GCP: Org Policies) | Policy enforcement | none (single account) |

## Round C6 — Load Balancing & Ingress

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Load balancer type? (AWS: ALB/NLB / OCI: Flexible LB/Network LB / L7 vs L4) | Traffic routing, TLS termination | L7 (ALB / Flexible LB) |
| 2 | LB sizing? (OCI: min/max bandwidth / AWS: fixed or auto) | Cost, throughput | Auto / Flexible |
| 3 | TLS certificates? (AWS: ACM / OCI: Certificates Service / Let's Encrypt / imported) | HTTPS automation | Cloud-native certificate service |
| 4 | CDN? (CloudFront, OCI CDN, Cloudflare, none) | Static assets, global latency | none |
| 5 | Custom domain? | DNS record, certificate | — |

## Round C7 — Observability

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Monitoring? (cloud-native, Prometheus + Grafana, Datadog, self-hosted) | Metrics and dashboards | Cloud-native (CloudWatch / OCI Monitoring) |
| 2 | Logging? (cloud-native logs, OpenSearch, Loki, external) | Log aggregation | Cloud-native (CloudWatch Logs / OCI Logging) |
| 3 | Tracing? (AWS: X-Ray / OCI: APM / Tempo, Jaeger, none) | Distributed tracing | none |
| 4 | Alerting? (cloud alarms, Alertmanager, PagerDuty, Slack) | Alert routing | Cloud-native alarms |
| 5 | Log retention? | Cost, compliance | 30 days |
| 6 | Cost monitoring? (AWS: Cost Explorer + Budgets / OCI: Cost Analysis + Budgets) | Budget tracking | Budgets with alerts |

## Round C8 — Operations & Cost

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Budget constraints? (free tier, monthly limit, reserved, savings plans, BYOL) | Instance sizing, service selection | — |
| 2 | Licensing? (OCI: BYOL for Oracle DB/Java / AWS: BYOL via Marketplace or License Manager) | Significant cost savings on OCI | — |
| 3 | Environment strategy? (separate accounts/compartments, separate VPCs/VCNs, shared infra) | Isolation level | Separate network per environment |
| 4 | CI/CD? (GitHub Actions, GitLab CI, cloud-native CI/CD, Jenkins) | Deployment automation | GitHub Actions |
| 5 | DR requirements? (RTO/RPO targets, pilot light, warm standby, active-active) | Architecture complexity | Best-effort, DB backups |
| 6 | Tagging strategy? (required tags, naming convention, cost-tracking) | Cost allocation, automation | `Environment`, `Project`, `Owner` |
| 7 | Tag enforcement? (AWS: SCP + Config rules / OCI: defined tags + tag defaults) | Governance | Freeform tags (enforce later) |
| 8 | Who operates? (single SRE, team, fully managed) | Runbook depth, alerting | — |

---

## Usage Notes

- **Don't ask all questions** — use the defaults column and skip what's obvious from context
- **Group by relevance** — if user says "EKS + RDS + ALB" or "OKE + Autonomous DB" upfront, skip basics and drill into specifics
- **Offer defaults** — present smart defaults and let the user override: "I'll assume 2 AZs, private subnets with NAT, cloud-native LB + TLS — any changes?"
- **Derive from existing infra** — if `CLAUDE.md` or existing Terraform files answer a question, don't re-ask
- **Combine with K8s questions** — if the target is managed K8s (EKS, OKE, GKE), use these for cloud infra and K8s discovery questions for cluster-level details
- **Provider strengths** — call out relevant cost advantages: OCI Always Free (Ampere A1, Autonomous DB, 200GB Object Storage), OCI BYOL licensing, AWS Graviton pricing, AWS free tier
- **Respect the round limit** — 3-5 rounds max standalone, 2-3 for subagent mode
