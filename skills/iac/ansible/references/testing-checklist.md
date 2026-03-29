# Ansible Role Testing Checklist

## Before Writing Tests

- [ ] Role has `defaults/main.yml` with all configurable variables documented
- [ ] Role has `meta/main.yml` with platform support declared
- [ ] Role uses FQCN for all modules
- [ ] All `command`/`shell` tasks have idempotency guards

## Molecule Setup

- [ ] `molecule/default/molecule.yml` exists with Docker/Podman driver
- [ ] Platform images match `meta/main.yml` declared platforms
- [ ] `converge.yml` includes the role under test
- [ ] `prepare.yml` installs any prerequisites the role expects
- [ ] `verify.yml` contains assertion tasks

## Test Coverage -- What to Verify

### Package Installation
- [ ] Required packages are installed (`package_facts` + `assert`)
- [ ] Package versions match expected constraints (when pinned)
- [ ] Unnecessary packages are NOT installed (negative assertion)

### Service Management
- [ ] Services are running (`service_facts` + `assert`)
- [ ] Services are enabled at boot
- [ ] Service restarts work (handler verification)

### Configuration Files
- [ ] Config files exist at expected paths (`stat`)
- [ ] File permissions are correct (`stat.mode`)
- [ ] File ownership is correct (`stat.pw_name`, `stat.gr_name`)
- [ ] File content contains expected directives (`slurp` + `b64decode`)
- [ ] Sensitive files have restricted permissions (0600)
- [ ] Template variables are rendered correctly

### Users and Groups
- [ ] System users are created (`getent`)
- [ ] System groups are created
- [ ] Users have correct shell, home directory, and group membership

### Directories
- [ ] Required directories exist (`stat`)
- [ ] Directory permissions are correct
- [ ] Directory ownership is correct

### Network
- [ ] Expected ports are listening (`wait_for`)
- [ ] Firewall rules are applied (when role manages firewall)
- [ ] Service responds to health checks (`uri`)

### System Configuration
- [ ] Sysctl parameters are set correctly (`command` + `sysctl -n`)
- [ ] Cron jobs are registered (when applicable)
- [ ] Log rotation is configured (when applicable)

## Idempotence

- [ ] `molecule idempotence` passes (zero changes on second run)
- [ ] No tasks report `changed` on second convergence
- [ ] `command`/`shell` tasks use `changed_when: false` or proper guards

## Multi-Platform

- [ ] Tests pass on all declared platforms in `meta/main.yml`
- [ ] OS-specific logic (package names, paths) is handled via `include_vars` or conditionals
- [ ] CI matrix covers all target distributions

## Security

- [ ] No secrets in test files (use test-specific dummy values)
- [ ] Sensitive tasks have `no_log: true` in the role
- [ ] File permissions on secrets are restrictive (0600)
- [ ] Service runs as non-root user when possible

## Edge Cases (Additional Scenarios)

- [ ] Role works on fresh install (default scenario)
- [ ] Role handles re-run gracefully (idempotence)
- [ ] Role handles upgrade path (upgrade scenario, if applicable)
- [ ] Role handles custom variable overrides
- [ ] Role fails gracefully with invalid input (`assert` tasks in role)

## Validation Pipeline

```bash
# 1. YAML syntax
yamllint -d relaxed roles/<role_name>/

# 2. Ansible lint
ansible-lint roles/<role_name>/

# 3. Molecule full test
cd roles/<role_name> && molecule test

# 4. Multi-platform test
cd roles/<role_name>
MOLECULE_DISTRO=ubuntu2404 molecule test
MOLECULE_DISTRO=debian12 molecule test
MOLECULE_DISTRO=rockylinux9 molecule test
```
