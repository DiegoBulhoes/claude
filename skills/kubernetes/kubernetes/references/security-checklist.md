# Kubernetes Security Checklist (CIS Benchmark Aligned)

## Pod Security

- [ ] All containers run as non-root (`runAsNonRoot: true`)
- [ ] Read-only root filesystem (`readOnlyRootFilesystem: true`)
- [ ] No privilege escalation (`allowPrivilegeEscalation: false`)
- [ ] All capabilities dropped (`capabilities.drop: [ALL]`)
- [ ] Seccomp profile set (`seccompProfile.type: RuntimeDefault`)
- [ ] No privileged containers unless documented exception
- [ ] No `hostNetwork`, `hostPID`, `hostIPC` unless documented exception
- [ ] No `hostPath` volumes in production
- [ ] `automountServiceAccountToken: false` when not needed

## Image Security

- [ ] Images use specific tags or SHA digests (never `latest`)
- [ ] Images pulled from trusted registries only
- [ ] Image vulnerability scanning enabled (Trivy, OCIR scanning)
- [ ] Base images updated regularly for security patches
- [ ] Multi-stage builds to minimize image size and attack surface

## Network Security

- [ ] Default-deny NetworkPolicy in every namespace
- [ ] Ingress rules allow only required traffic
- [ ] Egress rules restrict outbound connections
- [ ] mTLS between services (via service mesh or cert-manager)
- [ ] TLS termination at ingress with valid certificates
- [ ] No services exposed with `type: NodePort` in production

## RBAC

- [ ] No `cluster-admin` bindings for applications
- [ ] ServiceAccounts created per workload (not default)
- [ ] Role over ClusterRole when possible
- [ ] Verb-specific permissions (no wildcards `*`)
- [ ] ResourceNames specified when applicable
- [ ] Regular RBAC audit (`kubectl auth can-i --list`)

## Secret Management

- [ ] Secrets never stored in plain text in Git
- [ ] External Secrets Operator or Sealed Secrets for secret injection
- [ ] Secret rotation automated
- [ ] Secrets mounted as volumes (not environment variables) for sensitive data
- [ ] RBAC restricts secret access to necessary workloads only

## Resource Management

- [ ] Resource requests AND limits set on all containers
- [ ] LimitRange set per namespace
- [ ] ResourceQuota set per namespace
- [ ] PodDisruptionBudget for HA workloads
- [ ] HorizontalPodAutoscaler configured for variable workloads

## Availability

- [ ] replicas >= 2 for production workloads
- [ ] Pod anti-affinity or topology spread constraints
- [ ] PodDisruptionBudget configured
- [ ] Liveness, readiness, and startup probes configured
- [ ] `terminationGracePeriodSeconds` set appropriately
- [ ] Rolling update strategy with `maxUnavailable: 0`

## Namespace Isolation

- [ ] Workloads separated by namespace (not all in `default`)
- [ ] NetworkPolicies enforce namespace boundaries
- [ ] ResourceQuota per namespace prevents resource starvation
- [ ] RBAC scoped to namespace level

## Audit and Monitoring

- [ ] Kubernetes audit logging enabled
- [ ] Pod security events monitored
- [ ] Failed authentication attempts alerted
- [ ] Resource usage monitored (Prometheus + Grafana)
- [ ] Container runtime security (Falco or equivalent)

## Supply Chain Security

- [ ] Image signing and verification (Cosign/Notary)
- [ ] Admission controllers for policy enforcement (OPA/Gatekeeper, Kyverno)
- [ ] SBOM generation for container images
- [ ] Registry access restricted via IAM

## Sources

- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [OWASP Kubernetes Security](https://owasp.org/www-project-kubernetes-top-ten/)
