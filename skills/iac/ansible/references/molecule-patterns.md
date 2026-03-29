# Molecule Configuration Patterns

## Driver Selection Guide

| Driver | Use Case | Pros | Cons |
|--------|----------|------|------|
| Docker | Default for most roles | Fast, no infra needed | No real systemd (needs workaround) |
| Podman | Rootless containers | Rootless, daemonless | Slightly different config |
| Delegated | VMs, cloud, bare metal | Real OS, full systemd | Slower, needs infra management |
| Vagrant | Full VM testing | Real OS, multiple providers | Heavy, slow startup |

## Advanced molecule.yml Patterns

### Shared Group Variables

```yaml
provisioner:
  name: ansible
  inventory:
    group_vars:
      all:
        my_role_debug_mode: true
        my_role_log_level: debug
    host_vars:
      instance-ubuntu:
        my_role_package_manager: apt
      instance-rocky:
        my_role_package_manager: dnf
```

### Environment-Specific Overrides

```yaml
provisioner:
  name: ansible
  env:
    ANSIBLE_FORCE_COLOR: "true"
    ANSIBLE_VERBOSITY: "${MOLECULE_VERBOSITY:-0}"
  playbooks:
    prepare: prepare.yml
    converge: converge.yml
    verify: verify.yml
    side_effect: side_effect.yml
    cleanup: cleanup.yml
```

### Docker Network Configuration

```yaml
platforms:
  - name: instance-server
    image: "geerlingguy/docker-ubuntu2404-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
    networks:
      - name: molecule-net
    groups:
      - servers

  - name: instance-client
    image: "geerlingguy/docker-ubuntu2404-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
    networks:
      - name: molecule-net
    groups:
      - clients
```

### Podman Driver

```yaml
driver:
  name: podman

platforms:
  - name: instance
    image: "docker.io/geerlingguy/docker-ubuntu2404-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
```

### Custom Dockerfile

When pre-built images don't meet your needs:

```yaml
platforms:
  - name: instance
    image: "my-custom-ansible-test:latest"
    dockerfile: ../resources/Dockerfile.j2
    pre_build_image: false
```

`resources/Dockerfile.j2`:

```dockerfile
FROM {{ item.image }}

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    sudo \
    systemd \
    && apt-get clean

CMD ["/lib/systemd/systemd"]
```

## Scenario Patterns

### Scenario: High Availability / Multi-Node

Test roles that configure clusters or require inter-node communication:

```yaml
# molecule/ha/molecule.yml
platforms:
  - name: node-1
    image: "geerlingguy/docker-ubuntu2404-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
    networks:
      - name: cluster-net
    groups:
      - cluster

  - name: node-2
    image: "geerlingguy/docker-ubuntu2404-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
    networks:
      - name: cluster-net
    groups:
      - cluster

  - name: node-3
    image: "geerlingguy/docker-ubuntu2404-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
    networks:
      - name: cluster-net
    groups:
      - cluster
```

### Scenario: Side Effects

Test external interactions (e.g., API calls, DNS registration):

```yaml
# molecule/default/side_effect.yml
---
- name: Side Effect
  hosts: all
  become: true
  tasks:
    - name: Simulate external event
      ansible.builtin.uri:
        url: "http://instance-server:8080/health"
        method: GET
        status_code: 200
      retries: 5
      delay: 3
```

### Scenario: Idempotence-Only

Quick scenario that only checks idempotence:

```yaml
# molecule/idempotence/molecule.yml
scenario:
  name: idempotence
  test_sequence:
    - dependency
    - destroy
    - create
    - converge
    - idempotence
    - destroy
```

### Scenario: Upgrade Path

Test upgrading from a previous version:

```yaml
# molecule/upgrade/prepare.yml
---
- name: Prepare -- Install previous version
  hosts: all
  become: true
  vars:
    my_role_version: "1.0.0"
  tasks:
    - name: Simulate previous version installation
      ansible.builtin.include_role:
        name: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}"
```

```yaml
# molecule/upgrade/converge.yml
---
- name: Converge -- Upgrade to current version
  hosts: all
  become: true
  vars:
    my_role_version: "2.0.0"
  tasks:
    - name: Include role under test (upgrade)
      ansible.builtin.include_role:
        name: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}"
```

## Dependency Management in Molecule

### Galaxy Dependencies

```yaml
# molecule/default/requirements.yml
---
collections:
  - name: ansible.posix
    version: ">=1.5.0"
  - name: community.general
    version: ">=8.0.0"

roles:
  - name: geerlingguy.docker
    version: "7.1.0"
```

Reference in `molecule.yml`:

```yaml
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml
```

### Shell Dependencies

```yaml
dependency:
  name: shell
  command: |
    ansible-galaxy collection install -r requirements.yml --force
    pip install jmespath
```

## Debugging Tips

```bash
# SSH into test instance
molecule login --host instance-ubuntu2404

# Run converge with verbose output
molecule converge -- -vvv

# Run only specific tags
molecule converge -- --tags configure

# Keep instances after failed test (for debugging)
molecule test --destroy=never

# Reset and start fresh
molecule destroy && molecule converge
```
