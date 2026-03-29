# Role Template

Complete skeleton for a new Ansible role with Molecule testing.

## Directory Structure

```
<role_name>/
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
├── templates/
├── files/
├── meta/
│   └── main.yml
└── molecule/
    └── default/
        ├── molecule.yml
        ├── converge.yml
        ├── prepare.yml
        └── verify.yml
```

## defaults/main.yml

```yaml
---
# <role_name> -- Overridable defaults

# -- Enable or disable the role
<role_name>_enabled: true

# -- Package version to install (use "present" for latest available)
<role_name>_version: "present"

# -- Service state after configuration
<role_name>_service_state: started

# -- Enable service at boot
<role_name>_service_enabled: true

# -- Configuration file path
<role_name>_config_path: "/etc/<role_name>/<role_name>.conf"

# -- User to run the service as
<role_name>_user: "<role_name>"

# -- Group for the service
<role_name>_group: "<role_name>"

# -- Log level
<role_name>_log_level: "info"
```

## vars/main.yml

```yaml
---
# <role_name> -- Internal constants (do not override)

_<role_name>_packages:
  Debian:
    - "<package_name>"
  RedHat:
    - "<package_name>"

_<role_name>_service_name: "<service_name>"

_<role_name>_config_dir: "/etc/<role_name>"

_<role_name>_data_dir: "/var/lib/<role_name>"
```

## tasks/main.yml

```yaml
---
- name: Validate required variables
  ansible.builtin.assert:
    that:
      - <role_name>_enabled is defined
      - <role_name>_user | length > 0
    fail_msg: "Required variables are not properly set"
    quiet: true
  tags:
    - always

- name: Include OS-specific variables
  ansible.builtin.include_vars:
    file: "{{ ansible_os_family }}.yml"
  when: ansible_os_family in ['Debian', 'RedHat']
  tags:
    - always

- name: Install packages
  ansible.builtin.package:
    name: "{{ _<role_name>_packages[ansible_os_family] }}"
    state: "{{ <role_name>_version }}"
  become: true
  tags:
    - install
    - <role_name>

- name: Create system group
  ansible.builtin.group:
    name: "{{ <role_name>_group }}"
    system: true
    state: present
  become: true
  tags:
    - configure
    - <role_name>

- name: Create system user
  ansible.builtin.user:
    name: "{{ <role_name>_user }}"
    group: "{{ <role_name>_group }}"
    system: true
    shell: /usr/sbin/nologin
    home: "{{ _<role_name>_data_dir }}"
    create_home: false
  become: true
  tags:
    - configure
    - <role_name>

- name: Create directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: "{{ <role_name>_user }}"
    group: "{{ <role_name>_group }}"
    mode: "0755"
  become: true
  loop:
    - "{{ _<role_name>_config_dir }}"
    - "{{ _<role_name>_data_dir }}"
  tags:
    - configure
    - <role_name>

- name: Deploy configuration
  ansible.builtin.template:
    src: "<role_name>.conf.j2"
    dest: "{{ <role_name>_config_path }}"
    owner: "{{ <role_name>_user }}"
    group: "{{ <role_name>_group }}"
    mode: "0644"
    validate: "<VALIDATION_COMMAND> %s"
  become: true
  notify: Restart <role_name> service
  tags:
    - configure
    - <role_name>

- name: Manage service
  ansible.builtin.systemd_service:
    name: "{{ _<role_name>_service_name }}"
    state: "{{ <role_name>_service_state }}"
    enabled: "{{ <role_name>_service_enabled }}"
    daemon_reload: true
  become: true
  tags:
    - service
    - <role_name>
```

## handlers/main.yml

```yaml
---
- name: Restart <role_name> service
  ansible.builtin.systemd_service:
    name: "{{ _<role_name>_service_name }}"
    state: restarted
    daemon_reload: true
  become: true

- name: Reload <role_name> service
  ansible.builtin.systemd_service:
    name: "{{ _<role_name>_service_name }}"
    state: reloaded
  become: true
```

## meta/main.yml

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

## molecule/default/molecule.yml

```yaml
---
driver:
  name: docker

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
  env:
    ANSIBLE_FORCE_COLOR: "true"

verifier:
  name: ansible

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

## molecule/default/converge.yml

```yaml
---
- name: Converge
  hosts: all
  become: true
  tasks:
    - name: Include role under test
      ansible.builtin.include_role:
        name: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}"
```

## molecule/default/prepare.yml

```yaml
---
- name: Prepare
  hosts: all
  become: true
  tasks:
    - name: Update package cache (Debian)
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Update package cache (RedHat)
      ansible.builtin.dnf:
        update_cache: true
      when: ansible_os_family == "RedHat"
```

## molecule/default/verify.yml

```yaml
---
- name: Verify
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Gather package facts
      ansible.builtin.package_facts:

    - name: Assert packages are installed
      ansible.builtin.assert:
        that:
          - "'<PACKAGE_NAME>' in ansible_facts.packages"
        fail_msg: "<PACKAGE_NAME> is not installed"
        success_msg: "<PACKAGE_NAME> is installed"

    - name: Gather service facts
      ansible.builtin.service_facts:

    - name: Assert service is running and enabled
      ansible.builtin.assert:
        that:
          - "ansible_facts.services['<SERVICE_NAME>.service'].state == 'running'"
          - "ansible_facts.services['<SERVICE_NAME>.service'].status == 'enabled'"
        fail_msg: "<SERVICE_NAME> is not running or not enabled"
        success_msg: "<SERVICE_NAME> is running and enabled"

    - name: Check configuration file
      ansible.builtin.stat:
        path: "/etc/<role_name>/<role_name>.conf"
      register: _config_file

    - name: Assert configuration file exists with correct permissions
      ansible.builtin.assert:
        that:
          - _config_file.stat.exists
          - _config_file.stat.mode == '0644'
        fail_msg: "Configuration file is missing or has wrong permissions"
        success_msg: "Configuration file exists with correct permissions"

    - name: Check data directory
      ansible.builtin.stat:
        path: "/var/lib/<role_name>"
      register: _data_dir

    - name: Assert data directory exists
      ansible.builtin.assert:
        that:
          - _data_dir.stat.exists
          - _data_dir.stat.isdir
        fail_msg: "Data directory does not exist"
        success_msg: "Data directory exists"
```
