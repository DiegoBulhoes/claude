# PRD Discovery Questions

Structured questions to extract enough detail for a complete PRD. Ask in rounds — adapt based on answers, skip what's already clear from context.

---

## Round P1 — Problem & Motivation

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | What problem are we solving? | Anchors the entire PRD — everything flows from this | — |
| 2 | Why now? What triggered this work? | Urgency, priority justification | — |
| 3 | Who is affected? (end users, developers, ops, platform) | Identifies stakeholders and impact radius | ops/platform team |
| 4 | What happens if we do nothing? | Calibrates priority — is this critical or nice-to-have? | — |
| 5 | Is there a deadline or external constraint? | Timeline, compliance, release gates | No hard deadline |

## Round P2 — Success Criteria & Scope

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | What does "done" look like? (measurable outcomes) | Prevents scope creep, enables validation | — |
| 2 | How will we measure success? (metrics, SLIs, tests) | Defines acceptance criteria | Manual verification |
| 3 | What is explicitly IN scope? | Boundaries for the team | — |
| 4 | What is explicitly OUT of scope? | Prevents gold-plating and scope drift | — |
| 5 | Are there related tasks we should NOT touch? | Avoids accidental regressions | — |
| 6 | Is this a one-time task or recurring? | Determines if automation is worth it | One-time |

## Round P3 — Users & Stakeholders

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Who will use or consume the output? | Shapes design decisions | The team itself |
| 2 | Who needs to approve before execution? | Identifies blockers and review gates | User (you) |
| 3 | Are other teams or systems affected? | Cross-cutting impact, communication needs | No |
| 4 | Who should be notified when done? | Post-completion communication | User only |

## Round P4 — Constraints & Dependencies

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Are there budget or resource constraints? | Limits compute, storage, tooling choices | Stay within current infra |
| 2 | Are there compliance or security requirements? | Non-negotiable constraints that shape design | Follow existing security posture |
| 3 | What external dependencies exist? (APIs, services, other teams) | Identifies blockers before execution starts | — |
| 4 | What internal dependencies exist? (other PRDs, features, infra) | Determines execution order, `depends_on` field | — |
| 5 | Are there technology constraints? (must use X, can't use Y) | Narrows solution space | Follow project conventions |

## Round P5 — Risks & Rollback

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | What is the worst thing that can happen? | Identifies high-impact risks early | — |
| 2 | What has failed before in similar tasks? | Learns from history, avoids repeat mistakes | — |
| 3 | Is this reversible? How do we roll back? | Determines blast radius and safety net | — |
| 4 | Does this need a maintenance window or downtime? | Scheduling, communication needs | No downtime |
| 5 | What monitoring/alerting covers this area? | Post-deploy observability | Existing stack |

---

## Usage Notes

- **Don't ask all questions** — extract answers from context, `CLAUDE.md`, and the user's initial brief
- **Group by relevance** — if the user gives a detailed brief, skip P1 and go straight to clarifications in P2-P4
- **Present assumptions** — "I'll assume no hard deadline, single stakeholder (you), and in-scope is limited to X — correct?"
- **Fill `depends_on` and `references`** from P4 answers — map to existing PRD numbers when possible
- **Priority mapping** from P1 answers:
  - "production is broken" / "compliance deadline" → `alta`
  - "performance degradation" / "tech debt causing friction" → `media`
  - "nice to have" / "future-proofing" → `baixa`
