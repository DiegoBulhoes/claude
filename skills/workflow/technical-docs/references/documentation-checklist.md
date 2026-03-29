# Documentation Quality Checklist

Use this checklist for every document produced by the technical-docs skill.

## Pre-Writing Checklist

- [ ] Read `CLAUDE.md` for architecture facts
- [ ] Read relevant `docs/` files for existing context
- [ ] Trace Terragrunt hierarchy for referenced components
- [ ] Read module source files (`main.tf`, `variables.tf`, `outputs.tf`)
- [ ] Run `ls -lah` on component directories before updating diagrams

## Content Checklist

### Every Document Must Have

- [ ] Clear title (`# Page Title`)
- [ ] Purpose statement (first paragraph)
- [ ] At least 5 distinct source file citations
- [ ] No unverified claims ("likely", "probably", "should")
- [ ] Real HCL snippets from actual files (never fabricated)
- [ ] Dark-mode Mermaid diagrams (where applicable)

### Source Citations

- [ ] Format: `(path/to/file:line_number)`
- [ ] All cited files verified to exist
- [ ] Line numbers are accurate (not off by more than 2 lines)
- [ ] No citations to `.terragrunt-cache` files
- [ ] No citations to local config files (gitignored)

### Mermaid Diagrams

- [ ] Uses `graph TD` (not `graph TB` or `graph LR`)
- [ ] Dark-mode colors: fills `#1a1a2e`, `#0f3460`, `#16213e`
- [ ] Light text: `color:#eee` or `color:#e6edf3`
- [ ] VPN/connectivity block present with dashed connections
- [ ] All environments shown (no "same structure" shortcuts)
- [ ] Invisible edges (`~~~`) for vertical subgraph ordering
- [ ] Matches actual filesystem state

### Tables

- [ ] Headers are clear and concise
- [ ] Data is aligned and formatted consistently
- [ ] No empty cells without explanation
- [ ] CIDRs/IPs match current config files

## Document-Type Specific

### Architectural Deep-Dive

- [ ] Context/motivation section
- [ ] Component map with Mermaid diagram
- [ ] Dependency flow diagram
- [ ] Failure modes and mitigations
- [ ] Operational runbook with commands

### Onboarding Guide

- [ ] Prerequisites listed (tools, versions, access)
- [ ] Step-by-step first deploy walkthrough
- [ ] Common operations section
- [ ] Troubleshooting FAQ
- [ ] Glossary of terms

### Module Reference

- [ ] Resources created table
- [ ] Inputs table (from `variables.tf`)
- [ ] Outputs table (from `outputs.tf`)
- [ ] Usage example (from component `terragrunt.hcl`)
- [ ] Naming convention documented

## Post-Writing Checklist

- [ ] All cited files still exist: `grep -oP '\([^:)]+:\d+\)' doc.md | cut -d: -f1 | tr -d '(' | sort -u`
- [ ] No stale CIDR/IP references in documentation
- [ ] No references to old module paths
- [ ] Heading hierarchy is consistent (H1 > H2 > H3 > H4)
- [ ] No orphaned links or broken references
- [ ] Mermaid diagrams render correctly

## Sources

- Google Technical Writing Course: https://developers.google.com/tech-writing
