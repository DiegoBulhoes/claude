# Claude Code Skills & Agents

Reusable, cloud-agnostic skills and agents for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), covering Infrastructure as Code, Kubernetes, GitOps, and development workflows.

## Skills Catalog

### Infrastructure as Code

| Skill | Invoke | Description |
|-------|--------|-------------|
| `terraform` | `/terraform` | Terraform/OpenTofu code generation with registry integration |
| `terragrunt` | `/terragrunt` | Terragrunt patterns: catalogs, stacks, live repos, state management |
| `terraform-module` | `/terraform-module` | Terraform module creation and best practices |
| `iac-review` | `/iac-review` | IaC review and security audit for Terraform, Kustomize, Helm, K8s |

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
| `plan` | `/plan` | Implementation planning with structured todo and impact analysis |
| `audit` | `/audit` | Convention compliance auditing for modules and components |
| `technical-docs` | `/technical-docs` | Technical documentation with dark-mode Mermaid diagrams |

## Agents

| Agent | Description |
|-------|-------------|
| `terraform-expert` | Senior IaC specialist for Terraform/Terragrunt debugging, module design, state management |
| `cloud-troubleshooter` | Cloud infrastructure diagnosis: networking, IAM, quotas, state locks |
| `repo-explorer` | Repository navigation: find files, trace dependencies, understand structure |

## Installation

### Option 1: Copy individual skills

```bash
# Copy a skill to your project
cp -r skills/iac/terraform/ <your-project>/.claude/skills/terraform/

# Copy an agent
cp agents/terraform-expert.md <your-project>/.claude/agents/
```

### Option 2: Git submodule

```bash
git submodule add <repo-url> .claude-skills
# Then copy or symlink the skills you need into .claude/skills/
```

### Option 3: Symlink (local development)

```bash
ln -s /path/to/claude-skills/skills/iac/terraform .claude/skills/terraform
ln -s /path/to/claude-skills/agents/terraform-expert.md .claude/agents/terraform-expert.md
```

> **Note:** The category directories (`iac/`, `kubernetes/`, `workflow/`) are organizational only. When installing, skills should land in `.claude/skills/<skill-name>/` (flattened).

## Customization

- Override cloud-specific patterns by editing skill `references/` files
- Add project-specific context via `CLAUDE.md` in your project root
- Combine skills with project-specific agents for maximum effectiveness

## License

MIT
