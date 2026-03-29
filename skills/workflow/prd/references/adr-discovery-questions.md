# ADR Discovery Questions

Structured questions to extract enough detail for Architecture Decision Records. These questions drill into the "why" behind technical choices — not just what we're building, but why this approach over alternatives.

---

## Round A1 — Context & Forces

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | What is the current state? (greenfield, existing system, partial migration) | Determines starting point and migration complexity | — |
| 2 | What forces are driving this decision? (performance, cost, compliance, simplicity, team skill) | Reveals the real priorities behind the choice | — |
| 3 | What constraints are non-negotiable? (HA, latency, cost ceiling, specific tooling) | Hard boundaries that eliminate options | — |
| 4 | What is the expected lifetime of this decision? (temporary fix, 1 year, indefinite) | Short-lived decisions need less rigor, long-lived need more | Indefinite |
| 5 | Is this decision reversible? At what cost? | High reversal cost → more analysis needed upfront | — |

## Round A2 — Options & Trade-offs

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | What options have you already considered? | Avoids re-treading ground, surfaces prior research | — |
| 2 | Have you already ruled anything out? Why? | Captures rejected alternatives and reasoning | — |
| 3 | What is the simplest option that solves the problem? | Anchors discussion — complexity must be justified | — |
| 4 | What is the most robust option regardless of cost? | Defines the upper bound for comparison | — |
| 5 | Are there options the team has experience with vs. new technology? | Operational familiarity reduces risk | Prefer familiar tools |
| 6 | Does any option require additional infrastructure or dependencies? | Hidden cost, operational burden | — |

## Round A3 — Evaluation Criteria

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | What are the top 3 criteria for this decision? (cost, performance, simplicity, maturity, community, security) | Makes evaluation explicit and auditable | Simplicity, maturity, cost |
| 2 | How important is community support / ecosystem? | Affects long-term maintenance and debugging | Important for infra tools |
| 3 | How important is operational complexity? (day-2 burden) | Some options are easy to deploy but hard to maintain | High importance |
| 4 | Are there licensing or vendor lock-in concerns? | Legal, cost, portability | Prefer open source |
| 5 | Does this need to integrate with existing systems? Which? | Compatibility requirements narrow options | — |

## Round A4 — Consequences & Risks

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | What do we gain with the chosen approach? | Positive consequences — the "why we chose this" | — |
| 2 | What do we give up or accept as a trade-off? | Negative consequences — documented, not hidden | — |
| 3 | What could go wrong? (failure modes, edge cases) | Risk identification for mitigation planning | — |
| 4 | Does this decision close any doors for the future? | Opportunity cost, future flexibility | — |
| 5 | What would make us revisit this decision? | Trigger conditions for re-evaluation | — |
| 6 | How do we validate the decision worked? (metrics, timeline) | Success criteria specific to the architecture choice | — |

## Round A5 — Precedent & Patterns

| # | Question | Why it matters | Default if not answered |
|---|----------|---------------|----------------------|
| 1 | Have we made similar decisions before? What did we learn? | Institutional knowledge, avoid repeating mistakes | — |
| 2 | Is there an existing pattern in the project we should follow or break from? | Consistency vs. justified divergence | Follow existing patterns |
| 3 | Are there industry references or benchmarks for this decision? | External validation, best practices | — |
| 4 | Does the team have strong opinions or preferences? | Buy-in matters for adoption | — |

---

## Usage Notes

- **Don't ask all questions** — many answers come from the PRD discovery or existing codebase
- **Round A2 is critical** — the alternatives table is the most valuable part of an ADR. Push for at least 2-3 options even if the user has already decided
- **Capture the "why not"** — rejected alternatives with clear reasoning are more valuable than the chosen option description
- **Link to PRD constraints** — non-negotiables from PRD Round P4 directly feed into ADR Round A1
- **Evaluation criteria from A3** should map to columns in the alternatives comparison table
- **Trigger conditions from A4.5** — save these as future review markers (e.g., "revisit if traffic exceeds 10k rps")

### ADR Quality Checklist

A good ADR answers these without ambiguity:
- [ ] What decision was made?
- [ ] What was the context at the time?
- [ ] What alternatives were considered and why were they rejected?
- [ ] What are the positive and negative consequences?
- [ ] Under what conditions should this decision be revisited?
