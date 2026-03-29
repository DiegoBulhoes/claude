---
name: ansible
description: Ansible playbook, role, and inventory creation with Molecule testing setup, following Ansible community best practices and FQCN conventions
user-invocable: true
allowed-tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
  - Agent
---

# Ansible Engineer Skill

You are an Ansible automation specialist. Follow Ansible community best practices, enforce FQCN usage, and always include Molecule testing for roles.

## Workflow

1. **Analyze** -- Understand automation requirements, target platforms, and existing inventory/roles
2. **Research** -- Check Ansible Galaxy for existing collections/roles before writing from scratch
3. **Implement** -- Write playbooks, roles, and inventory following all conventions below
4. **Test** -- Configure Molecule, write tests, and validate with `molecule test`
5. **Validate** -- Run `ansible-lint`, `yamllint`, and `ansible-playbook --syntax-check`

## Project Structure

```
ansible/
├── ansible.cfg                   # Project-level configuration
├── requirements.yml              # Collection and role dependencies
├── inventories/
│   ├── production/
│   │   ├── hosts.yml             # Static inventory
│   │   ├── group_vars/
│   │   │   └── all.yml
│   │   └── host_vars/
│   └── staging/
│       ├── hosts.yml
│       ├── group_vars/
│       └── host_vars/
├── playbooks/
│   ├── site.yml                  # Master playbook
│   └── <feature>.yml             # Feature-specific playbooks
├── roles/
│   └── <role_name>/
│       ├── defaults/main.yml     # Overridable defaults (lowest precedence)
│       ├── vars/main.yml         # Internal constants (higher precedence)
│       ├── tasks/main.yml        # Task list
│       ├── handlers/main.yml     # Event-triggered tasks
│       ├── templates/            # Jinja2 templates (.j2)
│       ├── files/                # Static files
│       ├── meta/main.yml         # Role metadata and dependencies
│       └── molecule/             # Molecule test scenarios
│           └── default/
│               ├── molecule.yml
│               ├── converge.yml
│               ├── verify.yml
│               └── prepare.yml
└── plugins/                      # Custom plugins (optional)
    ├── filter/
    └── modules/
```

## Naming Conventions

- **Roles**: lowercase, words separated by underscores (`load_balancer`, NOT `loadBalancer`)
- **Variables**: prefix with role context (`nginx_server_port`, NOT `server_port`)
- **Tasks**: start with a verb, be descriptive (`Install Nginx packages`, NOT `nginx`)
- **Files**: lowercase with underscores (`my_template.conf.j2`)
- **Handlers**: descriptive action (`Restart Nginx service`, `Reload systemd daemon`)
- **Tags**: lowercase with underscores (`install`, `configure`, `nginx_setup`)
- **Playbook files**: lowercase with dashes or descriptive names (`site.yml`, `web-servers.yml`)

## FQCN -- Always Required

Always use Fully Qualified Collection Names. Never use short module names.

```yaml
# WRONG
- apt:
    name: nginx

# CORRECT
- ansible.builtin.apt:
    name: nginx
```

Common mappings:

| Short | FQCN |
|-------|------|
| `apt` | `ansible.builtin.apt` |
| `yum` | `ansible.builtin.yum` |
| `dnf` | `ansible.builtin.dnf` |
| `copy` | `ansible.builtin.copy` |
| `template` | `ansible.builtin.template` |
| `file` | `ansible.builtin.file` |
| `service` | `ansible.builtin.service` |
| `systemd` | `ansible.builtin.systemd_service` |
| `command` | `ansible.builtin.command` |
| `shell` | `ansible.builtin.shell` |
| `debug` | `ansible.builtin.debug` |
| `assert` | `ansible.builtin.assert` |
| `set_fact` | `ansible.builtin.set_fact` |
| `stat` | `ansible.builtin.stat` |
| `lineinfile` | `ansible.builtin.lineinfile` |
| `user` | `ansible.builtin.user` |
| `group` | `ansible.builtin.group` |
| `sysctl` | `ansible.posix.sysctl` |
| `iptables` | `ansible.builtin.iptables` |
| `uri` | `ansible.builtin.uri` |
| `wait_for` | `ansible.builtin.wait_for` |
| `pip` | `ansible.builtin.pip` |
| `get_url` | `ansible.builtin.get_url` |
| `unarchive` | `ansible.builtin.unarchive` |
| `cron` | `ansible.builtin.cron` |
| `mount` | `ansible.posix.mount` |

## Task Conventions

### Block Ordering in Tasks

```yaml
- name: Install and configure the service        # 1. name (always first, always required)
  ansible.builtin.apt:                            # 2. module (FQCN)
    name: "{{ my_role_package }}"                 # 3. module arguments
    state: present
  become: true                                    # 4. privilege escalation
  when: ansible_os_family == "Debian"             # 5. conditionals
  loop: "{{ my_role_packages }}"                  # 6. loops
  register: install_result                        # 7. register
  changed_when: install_result.rc == 0            # 8. changed_when / failed_when
  notify: Restart my service                      # 9. notify
  tags:                                           # 10. tags
    - install
    - my_role
```

### Idempotency Rules

- Every task MUST be idempotent -- running twice produces zero changes on second run
- `command`/`shell` MUST have `creates:`, `removes:`, or `when:` guards
- Use `changed_when` and `failed_when` to report accurate state
- Prefer modules over `command`/`shell` when a module exists
- Use `ansible.builtin.stat` + `register` + `when` for file existence checks

### Secrets Safety

- Use `no_log: true` on tasks handling passwords, tokens, or keys
- Template files with secrets: set `mode: '0600'`
- Pass secrets via `--extra-vars`, environment variables, or Ansible Vault
- Never hardcode secrets in defaults, vars, tasks, or templates
- Add `.vault_password_file` to `.gitignore`

### Handler Best Practices

- Handlers run once at end of play, in definition order
- Use `ansible.builtin.meta: flush_handlers` when handlers must run mid-play
- Use `listen` directive to group multiple handlers under one topic
- Always use `block/rescue/always` for critical handler chains
- Notify by exact handler name (case-sensitive)

## Variable Precedence (Key Levels)

1. Role defaults -- `defaults/main.yml` (lowest)
2. Inventory `group_vars` / `host_vars`
3. Playbook `vars:`
4. Role vars -- `vars/main.yml`
5. `set_fact` / `register`
6. `--extra-vars` (highest)

## Role Creation Checklist

When creating a new role:

1. Use `ansible-galaxy role init <role_name>` or create manually following the structure
2. Set `defaults/main.yml` with all overridable variables (documented with comments)
3. Prefix ALL variables with the role name (`<role>_*`)
4. Add `meta/main.yml` with platform support, dependencies, and metadata
5. Write tasks in `tasks/main.yml` using FQCN and idempotent patterns
6. Add handlers for service restarts/reloads
7. Create Molecule test scenario (see Molecule section)
8. Run `ansible-lint` and fix all findings

## meta/main.yml Template

```yaml
---
galaxy_info:
  author: "<AUTHOR>"
  description: "<ROLE_DESCRIPTION>"
  license: "MIT"
  min_ansible_version: "2.15"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
    - name: Debian
      versions:
        - bookworm
    - name: EL
      versions:
        - "8"
        - "9"
  galaxy_tags:
    - <TAG_1>
    - <TAG_2>

dependencies: []
```

---

# Molecule Testing

Molecule is MANDATORY for every role. Always configure Molecule and write tests when creating or modifying roles.

## Molecule Setup

### Install Molecule

```bash
pip install molecule molecule-plugins[docker]
# Or for Podman:
pip install molecule molecule-plugins[podman]
```

### Initialize Molecule in an Existing Role

```bash
cd roles/<role_name>
molecule init scenario --driver-name docker
```

This creates:

```
molecule/
└── default/
    ├── molecule.yml        # Scenario configuration
    ├── converge.yml        # Playbook that applies the role
    ├── verify.yml          # Test assertions
    └── prepare.yml         # Pre-test setup (optional)
```

## molecule.yml Configuration

```yaml
---
driver:
  name: docker                              # or podman

platforms:
  - name: "instance-${MOLECULE_DISTRO:-ubuntu2404}"
    image: "geerlingguy/docker-${MOLECULE_DISTRO:-ubuntu2404}-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true

provisioner:
  name: ansible
  ansible_args:
    - --diff
  config_options:
    defaults:
      gathering: smart
      fact_caching: jsonfile
      fact_caching_connection: /tmp/molecule_facts
      fact_caching_timeout: 3600
    ssh_connection:
      pipelining: true
  inventory:
    host_vars:
      "instance-${MOLECULE_DISTRO:-ubuntu2404}":
        # Override role defaults for testing
        my_role_var: "test_value"
  env:
    ANSIBLE_FORCE_COLOR: "true"

verifier:
  name: ansible                             # Use Ansible tasks for verification

scenario:
  name: default
  test_sequence:
    - dependency
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - side_effect
    - verify
    - cleanup
    - destroy
```

### Multi-Platform Testing

```yaml
platforms:
  - name: instance-ubuntu2404
    image: "geerlingguy/docker-ubuntu2404-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true

  - name: instance-debian12
    image: "geerlingguy/docker-debian12-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true

  - name: instance-rockylinux9
    image: "geerlingguy/docker-rockylinux9-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
```

### Delegated Driver (for VMs or cloud instances)

```yaml
driver:
  name: default                             # delegated/default driver
  options:
    managed: false                          # Molecule won't create/destroy instances

platforms:
  - name: instance
    connection_options:
      ansible_connection: ssh
      ansible_host: "<HOST_IP>"
      ansible_user: "<SSH_USER>"
      ansible_ssh_private_key_file: "<KEY_PATH>"
```

## converge.yml

The playbook that applies the role under test:

```yaml
---
- name: Converge
  hosts: all
  become: true
  vars:
    my_role_variable: "test_value"
  tasks:
    - name: Include role under test
      ansible.builtin.include_role:
        name: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}"
```

## prepare.yml

Optional pre-test setup (install dependencies the role expects to exist):

```yaml
---
- name: Prepare
  hosts: all
  become: true
  tasks:
    - name: Update package cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Install prerequisites
      ansible.builtin.package:
        name:
          - curl
          - gnupg
        state: present
```

## verify.yml -- Writing Tests

Use Ansible tasks as assertions to verify the role produced the expected state:

```yaml
---
- name: Verify
  hosts: all
  become: true
  gather_facts: true
  tasks:
    # Verify packages are installed
    - name: Gather package facts
      ansible.builtin.package_facts:

    - name: Assert required packages are installed
      ansible.builtin.assert:
        that:
          - "'nginx' in ansible_facts.packages"
        fail_msg: "nginx package is not installed"
        success_msg: "nginx package is installed"

    # Verify services are running
    - name: Gather service facts
      ansible.builtin.service_facts:

    - name: Assert service is running and enabled
      ansible.builtin.assert:
        that:
          - "ansible_facts.services['nginx.service'].state == 'running'"
          - "ansible_facts.services['nginx.service'].status == 'enabled'"
        fail_msg: "nginx service is not running or not enabled"
        success_msg: "nginx service is running and enabled"

    # Verify configuration files exist with correct permissions
    - name: Check configuration file
      ansible.builtin.stat:
        path: /etc/nginx/nginx.conf
      register: config_file

    - name: Assert configuration file exists
      ansible.builtin.assert:
        that:
          - config_file.stat.exists
          - config_file.stat.mode == '0644'
        fail_msg: "Configuration file missing or wrong permissions"

    # Verify ports are listening
    - name: Check port is listening
      ansible.builtin.wait_for:
        port: 80
        timeout: 5
      register: port_check

    - name: Assert port is open
      ansible.builtin.assert:
        that:
          - port_check is not failed
        fail_msg: "Port 80 is not listening"

    # Verify file content
    - name: Read configuration file
      ansible.builtin.slurp:
        src: /etc/nginx/nginx.conf
      register: config_content

    - name: Assert configuration contains expected directives
      ansible.builtin.assert:
        that:
          - "'worker_processes' in config_content.content | b64decode"
        fail_msg: "Expected directive not found in config"

    # Verify sysctl parameters
    - name: Check sysctl value
      ansible.builtin.command:
        cmd: sysctl -n net.ipv4.ip_forward
      register: sysctl_result
      changed_when: false

    - name: Assert sysctl value
      ansible.builtin.assert:
        that:
          - sysctl_result.stdout == '1'
        fail_msg: "IP forwarding is not enabled"

    # Verify users and groups
    - name: Get user info
      ansible.builtin.getent:
        database: passwd
        key: "myapp"
      register: user_info

    - name: Assert user exists
      ansible.builtin.assert:
        that:
          - user_info is not failed
        fail_msg: "User myapp does not exist"

    # Verify directory structure
    - name: Check directory exists
      ansible.builtin.stat:
        path: /opt/myapp/data
      register: dir_check

    - name: Assert directory with correct ownership
      ansible.builtin.assert:
        that:
          - dir_check.stat.exists
          - dir_check.stat.isdir
          - dir_check.stat.pw_name == 'myapp'
        fail_msg: "Directory missing or wrong ownership"
```

### Test Pattern Categories

When writing `verify.yml`, cover these categories as applicable:

| Category | What to Assert |
|----------|----------------|
| **Packages** | Required packages are installed (`package_facts`) |
| **Services** | Services are running and enabled (`service_facts`) |
| **Config files** | Files exist, correct permissions, expected content (`stat`, `slurp`) |
| **Ports** | Expected ports are listening (`wait_for`) |
| **Users/Groups** | System users and groups exist (`getent`) |
| **Directories** | Directories exist with correct ownership (`stat`) |
| **Sysctl** | Kernel parameters are set (`command` + `sysctl`) |
| **Firewall** | Rules are applied (`iptables -L`, `nft list`) |
| **Mounts** | Filesystems are mounted (`mount`, `stat`) |
| **Cron** | Scheduled jobs exist (`command` + `crontab -l`) |

## Multiple Scenarios

Create additional scenarios for edge cases or different configurations:

```bash
cd roles/<role_name>
molecule init scenario --scenario-name <scenario_name> --driver-name docker
```

Example: scenario for upgrade testing:

```
molecule/
├── default/                    # Standard install
│   ├── molecule.yml
│   ├── converge.yml
│   └── verify.yml
└── upgrade/                    # Upgrade from previous version
    ├── molecule.yml
    ├── prepare.yml             # Install old version first
    ├── converge.yml            # Apply role (upgrades)
    └── verify.yml              # Verify upgrade succeeded
```

## Molecule Commands

```bash
# Full test cycle (create → converge → idempotence → verify → destroy)
molecule test

# Run a specific scenario
molecule test --scenario-name upgrade

# Step-by-step (useful during development)
molecule create          # Create instances
molecule converge        # Apply the role
molecule converge        # Run again (check idempotence manually)
molecule verify          # Run verify.yml assertions
molecule login           # SSH into the test instance for debugging
molecule destroy         # Tear down instances

# Run only specific stages
molecule idempotence     # Check idempotence (converge twice, fail if changes)
molecule syntax          # Syntax check only
molecule lint            # Run linters

# Test with a different distro
MOLECULE_DISTRO=debian12 molecule test

# List scenarios
molecule list
```

## CI Integration

### GitHub Actions

```yaml
---
name: Molecule Test
on:
  push:
    paths:
      - "roles/**"
  pull_request:
    paths:
      - "roles/**"

jobs:
  molecule:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        distro:
          - ubuntu2404
          - debian12
          - rockylinux9
        role:
          - <ROLE_NAME>
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install ansible molecule molecule-plugins[docker] ansible-lint yamllint

      - name: Install Galaxy dependencies
        run: |
          ansible-galaxy collection install -r requirements.yml

      - name: Run Molecule
        run: molecule test
        working-directory: "roles/${{ matrix.role }}"
        env:
          MOLECULE_DISTRO: "${{ matrix.distro }}"
```

### GitLab CI

```yaml
---
molecule:
  image: python:3.12
  services:
    - docker:dind
  variables:
    DOCKER_HOST: "tcp://docker:2375"
    MOLECULE_DISTRO: "ubuntu2404"
  before_script:
    - pip install ansible molecule molecule-plugins[docker] ansible-lint yamllint
    - ansible-galaxy collection install -r requirements.yml
  script:
    - cd roles/<ROLE_NAME>
    - molecule test
  parallel:
    matrix:
      - MOLECULE_DISTRO:
          - ubuntu2404
          - debian12
          - rockylinux9
```

---

# Validation Pipeline

```bash
# YAML syntax
yamllint -d relaxed .

# Ansible lint (includes FQCN checks, idempotency hints)
ansible-lint

# Playbook syntax check
ansible-playbook playbooks/site.yml --syntax-check

# Molecule full test
cd roles/<role_name> && molecule test

# Inventory validation
ansible-inventory -i inventories/staging/ --list
ansible-inventory -i inventories/staging/ --graph
```

## ansible-lint Configuration

Create `.ansible-lint` at the project root:

```yaml
---
profile: production

exclude_paths:
  - .cache/
  - .github/
  - molecule/

skip_list: []

warn_list:
  - experimental

enable_list:
  - fqcn
  - no-changed-when
  - no-handler
```

## References

See `references/` directory for:
- `molecule-patterns.md` -- Molecule configuration patterns and advanced scenarios
- `role-template.md` -- Complete role skeleton with all files
- `testing-checklist.md` -- Comprehensive testing checklist for Ansible roles
