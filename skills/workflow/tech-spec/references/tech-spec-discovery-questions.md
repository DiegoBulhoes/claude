# Tech Spec Discovery Questions

Structured question bank for the technical interview. Ask **one question at a time**, always with a "why this matters" line. Adapt based on answers — skip what's obvious, drill into what's complex.

---

## Round T1 — Problem & System Type

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | What problem are we solving? Can you describe it in one sentence? | Forces clarity — if you can't state the problem simply, the design will be unfocused | — |
| 2 | Why now? What's the trigger? | Urgency shapes trade-offs — "compliance deadline in 30 days" is different from "nice to have" | Production incident, new business requirement, tech debt, scale limit, compliance |
| 3 | What type of system is this? | Determines architectural patterns, data flow, and technology choices | API (request/response), Event-driven (async), Batch (scheduled), Streaming (continuous), CRON (periodic), Hybrid |
| 4 | Who consumes this system? | Shapes API design, auth model, SLO targets | Internal services, external clients, mobile apps, third-party integrations, other teams |
| 5 | What exists today? | Greenfield vs evolution changes everything — constraints, migration, backward compat | Greenfield (nothing exists), Rewrite (replacing existing), Evolution (extending existing) |

## Round T2 — Scale & Performance

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | What throughput do you expect? (day 1 and 12 months out) | Drives architecture: 100 rps is different from 100k rps | Requests/sec, events/sec, records/batch, messages/sec |
| 2 | What's the latency target? | Determines sync vs async, caching, DB choice | p50 < 50ms, p99 < 200ms, p99 < 1s, "doesn't matter" (batch) |
| 3 | What's the data volume? (current and growth) | Storage costs, DB choice, partitioning strategy | MB, GB, TB — growth rate per month |
| 4 | What's the concurrency model? | Thread pool sizing, connection limits, queuing | Concurrent users, parallel workers, open connections |
| 5 | What's the traffic pattern? | Autoscaling config, capacity planning | Steady, daily peaks, weekly spikes, seasonal bursts, unpredictable |
| 6 | Is there a hard ceiling on resources? | Constrains instance types, replica counts | CPU, memory, budget, node count |

## Round T3 — Data & State

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | What data does this system own? What data does it read from others? | Defines boundaries, prevents distributed monolith | Own: entities you CRUD. Read: data you query from other systems |
| 2 | What consistency model do you need? | Strong consistency costs latency and availability | Strong (banking), Eventual (social feed), Causal (chat) |
| 3 | What's the read/write ratio? | Drives DB choice, caching strategy, read replicas | Read-heavy (100:1), Balanced (10:1), Write-heavy (1:10) |
| 4 | How long must data be retained? | Storage tiers, lifecycle policies, compliance | Hours (cache), Days (hot), Months (warm), Years (cold/compliance) |
| 5 | Will the schema evolve? How? | Migration strategy, backward compat, versioning | Additive only, breaking changes with versioning, schema registry |
| 6 | Any data that's especially sensitive? | Encryption, masking, access controls, audit | PII, financial, health, credentials, tokens |

## Round T4 — Dependencies & Integration

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | What are the upstream dependencies? (who calls you) | API design, rate limiting, auth, documentation | Internal services, API gateway, message broker, scheduled triggers |
| 2 | What are the downstream dependencies? (what you call) | Failure handling, circuit breakers, timeouts, fallbacks | Databases, external APIs, message brokers, other microservices |
| 3 | Sync or async communication? | Latency, coupling, failure isolation | Sync (HTTP/gRPC), Async (events/messages), Mix (command=sync, notification=async) |
| 4 | Need a message broker? | Event-driven patterns, decoupling, guaranteed delivery | Kafka, RabbitMQ, SQS, NATS, Redis Streams, none |
| 5 | Any external / third-party APIs? | Rate limits, SLAs, fallback strategies, vendor lock-in | Payment gateways, email services, cloud APIs, partner integrations |
| 6 | How do you handle dependency failures? | Resilience patterns | Retry with backoff, circuit breaker, fallback/cache, fail-fast, dead letter queue |

## Round T5 — Security

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | How do callers authenticate? | API security, token management | OAuth2/OIDC, API keys, mTLS, JWT, session cookies |
| 2 | How is authorization handled? | Permission model, multi-tenancy | RBAC (roles), ABAC (attributes), policy engine (OPA/Cedar), ACLs |
| 3 | Is this multi-tenant? | Data isolation, query scoping, noisy neighbor protection | Shared DB (row-level), Schema-per-tenant, DB-per-tenant |
| 4 | What data classification applies? | Encryption requirements, access controls, audit | Public, Internal, Confidential, Restricted/PII |
| 5 | Any compliance requirements? | Constraints on architecture, logging, data residency | SOC2, LGPD, HIPAA, PCI-DSS, ISO 27001, none |
| 6 | What needs audit logging? | Compliance, forensics, debugging | All mutations, auth events, data access, admin actions |

## Round T6 — Infrastructure & Deploy

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | Where does this run? | Cloud provider, region, constraints | AWS, OCI, GCP, Azure, on-prem, hybrid |
| 2 | Containers or serverless? | Operational model, cold starts, scaling | Kubernetes (EKS/OKE/GKE), Fargate/Cloud Run, Lambda/Functions, VMs |
| 3 | How many environments? | Parity, cost, testing strategy | dev + prod, dev + staging + prod, more |
| 4 | CI/CD pipeline? | Deployment automation, testing gates | GitHub Actions, GitLab CI, ArgoCD, Jenkins, cloud-native |
| 5 | IaC tool? | Infrastructure provisioning, drift detection | Terraform, Terragrunt, Pulumi, CloudFormation, CDK |
| 6 | GitOps? | Deployment model, drift reconciliation | ArgoCD, FluxCD, none (push-based CI/CD) |

> **Note**: For cloud-specific details, load [cloud-discovery-questions.md](cloud-discovery-questions.md). For Kubernetes details, load [k8s-discovery-questions.md](k8s-discovery-questions.md).

## Round T7 — Observability

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | What are the key SLIs? | Defines what "healthy" means | Availability, latency, error rate, throughput, saturation |
| 2 | What SLO targets? | Alert thresholds, error budgets | 99.9% availability, p99 < 200ms, error rate < 0.1% |
| 3 | Structured logging? | Queryability, correlation | JSON logs, correlation IDs, trace context, log levels |
| 4 | Distributed tracing? | Cross-service debugging | OpenTelemetry, Jaeger, Tempo, X-Ray, none |
| 5 | What dashboards do you need? | Operational visibility | Golden signals (latency, traffic, errors, saturation), business metrics, SLO dashboard |
| 6 | Who gets paged? | Incident response model | On-call rotation, team channel, escalation path |

## Round T8 — Cost & Trade-offs

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | What's the budget? | Constrains all architectural decisions | Monthly limit, annual budget, "minimize but no hard limit" |
| 2 | Build or buy? | Engineering time vs licensing cost | Build (control, no vendor lock-in) vs Buy (faster, less ops) |
| 3 | What are we optimizing for? | Drives every trade-off | Cost, Performance, Reliability, Developer experience, Time-to-market |
| 4 | What complexity is acceptable? | Team size, expertise, maintenance burden | Simple (team of 2), Moderate (dedicated team), Complex (platform team supports) |
| 5 | Any vendor lock-in concerns? | Portability, negotiation leverage | Cloud-agnostic required, some lock-in OK, full managed services OK |

## Round T9 — Rollout & Migration

| # | Question | Why it matters | Suggestions |
|---|----------|---------------|-------------|
| 1 | Rollout strategy? | Risk management, blast radius | Big bang, Canary (1% → 10% → 100%), Blue-green, Feature flags |
| 2 | Is this replacing something? | Migration plan, dual-write, backward compat | No (greenfield), Yes (needs migration plan) |
| 3 | Backward compatibility? | API versioning, consumer coordination | Required (N-1 compat), Not required (single consumer), Breaking changes OK with notice |
| 4 | How do we roll back? | Blast radius, recovery time | Revert deploy, feature flag off, DB rollback, traffic shift |
| 5 | What defines "launch success"? | Go/no-go criteria | Metrics thresholds, smoke tests, user acceptance, load test passed |

---

## Usage Notes

- **One question at a time** — present one question, wait for the answer, then ask the next
- **Adapt** — if the user answers multiple questions in one response, acknowledge and move on
- **Push back** — "it should scale" is not an answer. Ask: "to what? 100 rps? 10k rps? 100k rps?"
- **Flag contradictions** — "real-time" + "eventual consistency" + "cheap" → pick two
- **Skip what's known** — if `CLAUDE.md` says "we use PostgreSQL", don't ask about DB choice
- **Group when obvious** — if it's clearly a simple API, skip the streaming/batch/CRON questions
- **Stop when ready** — you don't need to ask every question. When you have enough context for a quality Tech Spec, stop and generate
