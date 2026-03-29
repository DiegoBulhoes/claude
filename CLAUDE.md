# Claude Skills Repository

Shareable, cloud-agnostic Claude Code skills and agents for Infrastructure as Code, Kubernetes, GitOps, and development workflows.

## Repository Structure

```
skills/                          # Organized by category
  iac/                           # Infrastructure as Code
    terraform/                   # Terraform/OpenTofu code generation and module creation
    terragrunt/                  # Terragrunt patterns and practices
    ansible/                     # Ansible roles, playbooks, and Molecule testing
    iac-review/                  # IaC review and security audit
  kubernetes/                    # Kubernetes ecosystem
    kubernetes/                  # K8s manifest generation and hardening
    helm/                        # Helm chart creation
    kustomize/                   # Kustomize overlay patterns
    gitops/                      # GitOps workflows (ArgoCD/FluxCD)
  workflow/                      # Development workflows
    explore/                     # Repository structure mapping
    audit/                       # Convention compliance auditing
    technical-docs/              # Technical documentation
    prd/                         # Product-focused PRD with interview flow
    tech-spec/                   # Technical deep-dive (uses PRD as input)
agents/                          # Reusable agent definitions
  spec-writer.md                 # PRD + Tech Spec interviewer and generator
  terraform-expert.md            # IaC specialist
  ansible-expert.md              # Ansible specialist
  cloud-troubleshooter.md        # Cloud infrastructure diagnosis
```

## Conventions

- Skills use `SKILL.md` with YAML frontmatter (`name`, `description`, `user-invocable`, `allowed-tools`)
- Each skill may have a `references/` subdirectory with supporting documentation
- Agents use YAML frontmatter (`name`, `description`, `model`, `color`)
- **All content must be cloud-provider-agnostic** — no AWS, GCP, Azure, or OCI-specific code
- Use `<PLACEHOLDER>` syntax for project-specific values in examples
- English language for all content
- Mermaid diagrams use the dark-mode palette defined in `technical-docs` references

## Adding a New Skill

1. Create `skills/<category>/<skill-name>/SKILL.md`
2. Add `references/` directory if supporting docs are needed
3. Update `README.md` catalog table

## Installing Skills in a Project

Skills are copied individually into the consuming project's `.claude/skills/` directory. The category subdirectories (`iac/`, `kubernetes/`, `workflow/`) exist only for organization in this repository — skills should be flattened when installed:

```bash
# Copy a skill
cp -r skills/iac/terraform/ <project>/.claude/skills/terraform/

# Copy an agent
cp agents/spec-writer.md <project>/.claude/agents/

# Copy the spec-writer's skills (prd + tech-spec)
cp -r skills/workflow/prd/ <project>/.claude/skills/prd/
cp -r skills/workflow/tech-spec/ <project>/.claude/skills/tech-spec/
```
