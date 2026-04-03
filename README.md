# Claude Code Skills & Agents

Reusable, cloud-agnostic skills and agents for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), covering Infrastructure as Code, Kubernetes, GitOps, development workflows, and programming languages.

## Skills Catalog

### Infrastructure as Code

| Skill | Invoke | Description |
|-------|--------|-------------|
| `terraform` | `/terraform` | Terraform/OpenTofu code generation, module creation, and registry integration |
| `terragrunt` | `/terragrunt` | Terragrunt patterns: catalogs, stacks, live repos, state management |
| `ansible` | `/ansible` | Ansible playbook, role creation with Molecule testing setup and CI integration |
| `iac-review` | `/iac-review` | IaC review and security audit for Terraform, Kustomize, Helm, K8s |

### Development

| Skill | Invoke | Description |
|-------|--------|-------------|
| `golang` | `/golang` | Go code generation, project layout, naming, style, error handling, testing, concurrency, performance, and security |

### Kubernetes & GitOps

| Skill | Invoke | Description |
|-------|--------|-------------|
| `kubernetes` | `/kubernetes` | Kubernetes manifest generation, security hardening, CIS benchmark |
| `helm` | `/helm` | Helm chart creation, values organization, best practices |
| `kustomize` | `/kustomize` | Kustomize manifest management, overlay patterns, generators |
| `gitops` | `/gitops` | ArgoCD-based GitOps workflows, environment promotion, ESO secrets |

### Workflow

| Skill | Invoke | Description |
|-------|--------|-------------|
| `explore` | `/explore` | Repository structure mapping and dependency analysis |
| `audit` | `/audit` | Convention compliance auditing for modules and components |
| `technical-docs` | `/technical-docs` | Technical documentation with dark-mode Mermaid diagrams |
| `prd` | `/prd` | Product-focused PRD — interactive interview that produces PRD with features, milestones, team roles, and acceptance criteria |
| `tech-spec` | `/tech-spec` | Technical deep-dive — architecture, data model, NFRs, security, cost estimates, rollout plan. Uses PRD as input |

## Agents

| Agent | Description |
|-------|-------------|
| `spec-writer` | Conducts structured interviews (one question at a time) to produce PRDs and Tech Specs. Orchestrates the PRD → Tech Spec pipeline. Uses `/prd` and `/tech-spec` skills |
| `terraform-expert` | Senior IaC specialist for Terraform/Terragrunt debugging, module design, state management |
| `ansible-expert` | Ansible specialist for playbooks, roles, inventory, and configuration management |
| `cloud-troubleshooter` | Cloud infrastructure diagnosis: networking, IAM, quotas, state locks |

## Spec Writer Workflow

The `spec-writer` agent produces documentation in two phases:

```
Phase 1: PRD Interview → PRD (what & why)
              ↓
Phase 2: Tech Spec Interview → Tech Spec (how, at what cost, what can go wrong)
```

**PRD** (product-focused): Context, Team & Roles, Proposed Solution, Features (table), Acceptance Criteria, Milestones (table), Dependencies, Risks, Architecture Overview, Decision Log.

**Tech Spec** (technical): Architecture, Technical Decisions, Requirements (WHEN/THEN), Data Model, Security, Infrastructure, Observability, Cost Estimate, Rollout Plan, Decision Log.

The PRD's Team & Roles section maps human roles to Claude agents, enabling automated agent team composition in the Tech Spec.

## Installation

### Option 1: Copy individual skills

```bash
# Copy a skill to your project
cp -r skills/iac/terraform/ <your-project>/.claude/skills/terraform/

# Copy the spec-writer agent + both skills it needs
cp agents/spec-writer.md <your-project>/.claude/agents/
cp -r skills/workflow/prd/ <your-project>/.claude/skills/prd/
cp -r skills/workflow/tech-spec/ <your-project>/.claude/skills/tech-spec/
```

### Option 2: Git submodule

```bash
git submodule add <repo-url> .claude-skills
# Then copy or symlink the skills you need into .claude/skills/
```

### Option 3: Symlink (local development)

```bash
ln -s /path/to/claude-skills/skills/iac/terraform .claude/skills/terraform
ln -s /path/to/claude-skills/agents/spec-writer.md .claude/agents/spec-writer.md
```

> **Note:** The category directories (`iac/`, `kubernetes/`, `workflow/`) are organizational only. When installing, skills should land in `.claude/skills/<skill-name>/` (flattened).

## Customization

- Override cloud-specific patterns by editing skill `references/` files
- Add project-specific context via `CLAUDE.md` in your project root
- Combine skills with project-specific agents for maximum effectiveness

## Inspirations & References

Skills in this repository are informed by community best practices and established style guides:

| Reference | Used in |
|-----------|---------|
| [Go Proverbs](https://go-proverbs.github.io/) | `/golang` -- Rob Pike's guiding principles, cultural foundation of Go |
| [Uber Go Style Guide](https://github.com/uber-go/guide) | `/golang` -- Production-tested conventions from one of the largest Go codebases |
| [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang) | `/golang` -- Studied for structure, depth, and practical rule formulation |
| [Effective Go](https://go.dev/doc/effective_go) | `/golang` -- Official Go team guidance |
| [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) | `/golang` -- Community wiki of code review patterns |
| [terraform-plugin-framework](https://developer.hashicorp.com/terraform/plugin/framework) | `/golang` -- Terraform provider development patterns |
| [cloudflare/terraform-provider-cloudflare](https://github.com/cloudflare/terraform-provider-cloudflare) | `/golang` -- Reference provider: service-per-resource layout, schema patterns, test templates |
| [Kubebuilder Book](https://book.kubebuilder.io/) | `/golang` -- Kubernetes operator development with controller-runtime |

## License

MIT
