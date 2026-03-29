# Exploration Checklist

Step-by-step checklist for comprehensive repository exploration.

## Phase 1: Repository Skeleton

- [ ] Read `CLAUDE.md` for architecture overview
- [ ] List all environments: `ls <LIVE_DIR>/<TENANT>/`
- [ ] List all modules: `ls <MODULES_DIR>/`
- [ ] Count total components per environment
- [ ] Verify `account.hcl` exists and has expected fields

## Phase 2: Component Trace (Per Component)

For each component found:

- [ ] Read `terragrunt.hcl`
- [ ] Verify `include "root"` with `expose = true`
- [ ] Extract `terraform { source }` — is it local or registry?
- [ ] List all `dependency {}` blocks and their `config_path`
- [ ] Verify `mock_outputs` exist on each dependency
- [ ] Check for `env_local.hcl.example` (if needed)
- [ ] Extract `inputs` block — check tags and labels
- [ ] Note any `generate` blocks (provider pins, etc.)

### Quick Trace Command

```bash
# Extract key info from a component
FILE="<LIVE_DIR>/<TENANT>/<ENV>/<REGION>/components/<COMPONENT>/terragrunt.hcl"

echo "=== Source ==="
grep -A2 "source" "$FILE" | head -3

echo "=== Dependencies ==="
grep "config_path" "$FILE"

echo "=== Tags ==="
grep -A1 "tags\|labels" "$FILE"
```

## Phase 3: Module Inventory (Per Module)

For each module found:

- [ ] Verify file structure: `main.tf`, `variables.tf`, `outputs.tf`, `README.md`
- [ ] Check `required_providers` in `main.tf`
- [ ] Count resources created
- [ ] Verify provider-managed tag lifecycle on resources with managed tags
- [ ] Check standard variables expected by convention
- [ ] Verify all outputs have `description`
- [ ] Find all consuming components

### Quick Inventory Command

```bash
# Check module completeness
MODULE="<MODULES_DIR>/<MODULE_NAME>"

echo "=== Files ==="
ls "$MODULE/"

echo "=== Resources ==="
grep "^resource " "$MODULE/main.tf"

echo "=== Variables ==="
grep "^variable " "$MODULE/variables.tf"

echo "=== Outputs ==="
grep "^output " "$MODULE/outputs.tf"

echo "=== Consumers ==="
grep -rl "modules/$(basename $MODULE)" <LIVE_DIR>/ \
  --include="*.hcl" --exclude-dir=".terragrunt-cache"
```

## Phase 4: Dependency Graph

- [ ] Extract all dependency relationships
- [ ] Build Mermaid graph
- [ ] Identify circular dependencies (should be none)
- [ ] Identify components with no dependencies (root components)
- [ ] Identify components with most dependents (critical path)

### Graph Extraction Command

```bash
# Build dependency pairs: component -> dependency
find <LIVE_DIR> -name "terragrunt.hcl" \
  -not -path "*/.terragrunt-cache/*" | while read f; do
  component=$(dirname "$f" | sed 's|.*/components/||')
  grep "config_path" "$f" | sed 's/.*"\(.*\)"/\1/' | sed 's|\.\./||' | while read dep; do
    echo "$component -> $dep"
  done
done | sort -u
```

## Phase 5: Environment Parity

- [ ] Compare component lists across all environments
- [ ] Identify components unique to one environment
- [ ] Verify expected differences (some environments may be minimal)
- [ ] Check configuration consistency (same module versions)

### Parity Check Command

```bash
# Compare environments (adjust paths to match your repo structure)
diff <(ls <LIVE_DIR>/<TENANT>/dev/<REGION>/components/ 2>/dev/null | sort) \
     <(ls <LIVE_DIR>/<TENANT>/staging/<REGION>/components/ 2>/dev/null | sort)

diff <(ls <LIVE_DIR>/<TENANT>/dev/<REGION>/components/ 2>/dev/null | sort) \
     <(ls <LIVE_DIR>/<TENANT>/prod/<REGION>/components/ 2>/dev/null | sort)
```

## Phase 6: Gaps Report

After all phases, compile:

- [ ] Orphaned modules (no consumers)
- [ ] Missing provider-managed tag lifecycle
- [ ] Missing mock_outputs
- [ ] Missing README.md on modules
- [ ] Environment parity gaps
- [ ] Missing env_local.hcl.example
- [ ] Stale references to old paths/modules

## Sources

- Terragrunt Documentation: https://terragrunt.gruntwork.io/docs/
- Terraform Module Development: https://developer.hashicorp.com/terraform/language/modules/develop
