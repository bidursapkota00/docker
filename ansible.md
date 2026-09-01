# Ansible Complete Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Ansible](#introducing-ansible)
2. [Installation & Setup](#installation--setup)
3. [Inventory](#inventory)
4. [Ad-Hoc Commands](#ad-hoc-commands)
5. [Playbooks](#playbooks)
6. [Modules](#modules)
7. [Variables](#variables)
8. [Facts & Magic Variables](#facts--magic-variables)
9. [Conditionals](#conditionals)
10. [Loops](#loops)
11. [Handlers & Notifications](#handlers--notifications)
12. [Templates (Jinja2)](#templates-jinja2)
13. [Roles](#roles)
14. [Ansible Galaxy](#ansible-galaxy)
15. [Ansible Vault](#ansible-vault)
16. [Error Handling](#error-handling)
17. [Tags](#tags)
18. [Delegation & Local Actions](#delegation--local-actions)
19. [Ansible Configuration](#ansible-configuration)
20. [Best Practices](#best-practices)

---

## Introducing Ansible

Ansible is an open-source automation tool used for configuration management, application deployment, task automation, and orchestration. Unlike tools such as Puppet or Chef, Ansible is agentless — it connects to remote machines over SSH (or WinRM for Windows) and executes tasks without requiring any software to be installed on the managed nodes.

Ansible is commonly used to configure servers and install software, deploy applications consistently across environments, orchestrate multi-tier application rollouts, enforce desired state across infrastructure, automate repetitive sysadmin tasks, and provision cloud resources.

Core concepts of Ansible are:

- **Control Node**: The machine where Ansible is installed and from which you run commands. This is your workstation or a CI/CD server.
- **Managed Node**: A remote machine that Ansible manages. Also called a host.
- **Inventory**: A file that lists managed nodes, organized into groups.
- **Module**: A unit of code that Ansible runs on managed nodes (e.g., `apt`, `copy`, `service`, `user`).
- **Task**: A single action that calls a module with specific arguments.
- **Play**: A set of tasks applied to a group of hosts.
- **Playbook**: A YAML file containing one or more plays — the main way you define automation.
- **Role**: A structured, reusable collection of tasks, variables, files, templates, and handlers.
- **Facts**: System information automatically gathered from managed nodes (OS, IP, memory, etc.).
- **Handler**: A task that runs only when notified by another task (e.g., restart a service after a config change).
- **Idempotency**: Running the same playbook multiple times produces the same result without side effects.

---

## Installation & Setup

### Install Ansible

**macOS (Homebrew)**:

```bash
brew install ansible
```

**Linux (Ubuntu/Debian)**:

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

**Using pip (any OS)**:

```bash
pip install ansible
```

`pip install` is the most portable method and always gives you the latest version. Using a virtual environment is recommended to avoid conflicts.

**Verify Installation**:

```bash
ansible --version
ansible-playbook --version
```

### SSH Setup

Ansible communicates with managed nodes over SSH. Set up key-based authentication to avoid entering passwords:

```bash
# Generate SSH key pair (if you don't have one)
ssh-keygen -t ed25519 -C "ansible"

# Copy public key to managed nodes
ssh-copy-id user@192.168.1.10
ssh-copy-id user@192.168.1.11
ssh-copy-id user@192.168.1.12

# Test SSH access
ssh user@192.168.1.10
```

`ssh-copy-id` installs your public key on the remote host's `~/.ssh/authorized_keys`, enabling passwordless login. Ansible uses these keys by default.

### Project Structure

A typical Ansible project:

```
ansible-project/
├── ansible.cfg                # Configuration file
├── inventory/
│   ├── production             # Production inventory
│   └── staging                # Staging inventory
├── group_vars/
│   ├── all.yml                # Variables for all hosts
│   ├── webservers.yml         # Variables for webservers group
│   └── dbservers.yml          # Variables for dbservers group
├── host_vars/
│   └── web1.example.com.yml   # Variables for a specific host
├── playbooks/
│   ├── site.yml               # Main playbook
│   ├── webservers.yml         # Web server playbook
│   └── dbservers.yml          # Database server playbook
├── roles/
│   ├── common/
│   ├── nginx/
│   └── postgres/
├── templates/
│   └── nginx.conf.j2
├── files/
│   └── app.conf
└── vault/
    └── secrets.yml            # Encrypted secrets
```

---

## Inventory

The inventory defines which hosts Ansible manages and how they are organized.

### INI Format

```ini
# inventory/production

# Ungrouped hosts
10.0.0.50

# Group: webservers
[webservers]
web1.example.com
web2.example.com
web3.example.com ansible_host=192.168.1.13 ansible_port=2222

# Group: dbservers
[dbservers]
db1.example.com
db2.example.com

# Group: loadbalancers
[loadbalancers]
lb1.example.com

# Group of groups
[production:children]
webservers
dbservers
loadbalancers

# Group variables
[webservers:vars]
http_port=80
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_key

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Each line under a group header is a host. `ansible_host` overrides the hostname used to connect. `ansible_port` overrides the SSH port. `[group:children]` creates a group that contains other groups. `[group:vars]` sets variables for all hosts in that group. `[all:vars]` applies to every host in the inventory.

### YAML Format

```yaml
# inventory/production.yml
all:
  vars:
    ansible_python_interpreter: /usr/bin/python3

  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
        web3.example.com:
          ansible_host: 192.168.1.13
          ansible_port: 2222
      vars:
        http_port: 80
        ansible_user: deploy

    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
      vars:
        db_port: 5432

    loadbalancers:
      hosts:
        lb1.example.com:

    production:
      children:
        webservers:
        dbservers:
        loadbalancers:
```

YAML inventory is more readable and supports complex structures. Both formats are functionally identical.

### Host Connection Variables

| Variable                         | Description                                |
| -------------------------------- | ------------------------------------------ |
| `ansible_host`                   | IP or hostname to connect to               |
| `ansible_port`                   | SSH port (default: 22)                     |
| `ansible_user`                   | SSH username                               |
| `ansible_ssh_private_key_file`   | Path to SSH private key                    |
| `ansible_password`               | SSH password (use vault, avoid plaintext)   |
| `ansible_become`                 | Enable privilege escalation (sudo)         |
| `ansible_become_user`            | User to become (default: root)             |
| `ansible_become_password`        | Password for privilege escalation          |
| `ansible_python_interpreter`     | Path to Python on the managed node         |
| `ansible_connection`             | Connection type (ssh, local, docker, winrm)|

### Inventory Commands

```bash
ansible-inventory --list -i inventory/production       # List all hosts as JSON
ansible-inventory --graph -i inventory/production      # Show group hierarchy
ansible-inventory --host web1.example.com              # Show vars for a host

ansible all --list-hosts -i inventory/production       # List all hosts
ansible webservers --list-hosts -i inventory/production # List hosts in group
```

`--list` outputs the full inventory in JSON. `--graph` shows a tree view of groups and hosts. `--host` shows all variables that apply to a specific host.

### Dynamic Inventory

For cloud environments, Ansible supports dynamic inventory scripts and plugins that query AWS, GCP, Azure, etc. to discover hosts at runtime.

```bash
# AWS EC2 plugin
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment:
    - production
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
compose:
  ansible_host: public_ip_address
```

```bash
ansible-inventory -i inventory/aws_ec2.yml --graph
```

Dynamic inventory plugins auto-discover hosts and group them by tags, regions, or other attributes. Install the collection first: `ansible-galaxy collection install amazon.aws`.

---

## Ad-Hoc Commands

Ad-hoc commands run a single task on one or more hosts without writing a playbook. They are useful for quick one-off operations.

### Syntax

```bash
ansible <pattern> -m <module> -a "<arguments>" -i <inventory>
```

### Common Examples

```bash
# Ping all hosts (test connectivity)
ansible all -m ping -i inventory/production

# Ping a specific group
ansible webservers -m ping

# Run a shell command
ansible all -m shell -a "uptime"

# Run a command (safer, no shell features)
ansible all -m command -a "df -h"

# Copy a file to remote hosts
ansible webservers -m copy -a "src=./app.conf dest=/etc/app/app.conf owner=root mode=0644"

# Install a package
ansible webservers -m apt -a "name=nginx state=present" --become

# Start a service
ansible webservers -m service -a "name=nginx state=started enabled=yes" --become

# Create a user
ansible all -m user -a "name=deploy shell=/bin/bash state=present" --become

# Gather facts
ansible web1.example.com -m setup

# Gather specific facts
ansible web1.example.com -m setup -a "filter=ansible_distribution*"

# Reboot all servers
ansible all -m reboot --become

# Check disk space
ansible all -m shell -a "df -h /"

# Run with 10 parallel forks
ansible all -m ping -f 10
```

`-m` specifies the module. `-a` passes arguments to the module. `--become` runs with sudo (privilege escalation). `-f` sets the number of parallel connections (default: 5). The `ping` module is not ICMP ping — it verifies Ansible connectivity (SSH + Python).

### Patterns

```bash
ansible all -m ping                    # All hosts
ansible webservers -m ping             # A group
ansible web1.example.com -m ping       # A single host
ansible 'webservers:dbservers' -m ping # Multiple groups (union)
ansible 'webservers:&production' -m ping # Intersection
ansible 'webservers:!web3.example.com' -m ping # Exclusion
ansible '*.example.com' -m ping        # Wildcard
ansible 'webservers[0]' -m ping        # First host in group
ansible 'webservers[0:2]' -m ping      # First 3 hosts (slice)
```

`:` is union (OR), `:&` is intersection (AND), `:!` is exclusion (NOT). Patterns can be combined.

---

## Playbooks

Playbooks are YAML files that define automation workflows. A playbook contains one or more plays, and each play maps a group of hosts to a set of tasks.

### Basic Playbook

```yaml
# playbooks/site.yml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  gather_facts: yes

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Copy Nginx configuration
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
      notify: Restart Nginx

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

`---` marks the start of a YAML document. `name` is a human-readable description. `hosts` specifies which inventory group to target. `become: yes` enables sudo for all tasks. `gather_facts: yes` (default) collects system information before running tasks. `notify` triggers a handler only when the task makes a change. Handlers run once at the end of the play, even if notified multiple times.

### Running Playbooks

```bash
ansible-playbook playbooks/site.yml -i inventory/production

# Useful flags
ansible-playbook site.yml --check            # Dry run (no changes)
ansible-playbook site.yml --diff             # Show file changes
ansible-playbook site.yml --check --diff     # Dry run with diffs
ansible-playbook site.yml --limit webservers # Run on specific group
ansible-playbook site.yml --limit web1.example.com # Run on one host
ansible-playbook site.yml --tags deploy      # Run only tagged tasks
ansible-playbook site.yml --skip-tags debug  # Skip tagged tasks
ansible-playbook site.yml --start-at-task "Install Nginx"  # Start from task
ansible-playbook site.yml --step             # Confirm each task
ansible-playbook site.yml --list-tasks       # List all tasks
ansible-playbook site.yml --list-hosts       # List target hosts
ansible-playbook site.yml -v                 # Verbose
ansible-playbook site.yml -vvv              # Very verbose (debug)
ansible-playbook site.yml --forks 20        # Parallel execution
ansible-playbook site.yml -e "env=prod"     # Pass extra variables
```

`--check` runs in dry-run mode — modules report what would change without making actual changes. `--diff` shows the before/after of file changes. `--limit` restricts execution to specific hosts or groups. `--step` prompts for confirmation before each task. `-v` through `-vvvv` increases verbosity.

### Multiple Plays

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

- name: Configure database servers
  hosts: dbservers
  become: yes
  tasks:
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present

- name: Configure all servers
  hosts: all
  become: yes
  tasks:
    - name: Set timezone
      timezone:
        name: UTC
```

A playbook can contain multiple plays. Each play targets a different group of hosts and runs its own set of tasks. Plays execute sequentially from top to bottom.

### Including & Importing

```yaml
# playbooks/site.yml — main entrypoint
---
- import_playbook: webservers.yml
- import_playbook: dbservers.yml
```

```yaml
# Within a play — include task files
- name: Configure app
  hosts: webservers
  become: yes
  tasks:
    - import_tasks: tasks/install.yml          # Static include (parsed at load)
    - include_tasks: tasks/configure.yml       # Dynamic include (parsed at runtime)
    - include_tasks: tasks/deploy.yml
      when: deploy_app | default(false)
```

`import_tasks` is static — processed when the playbook is parsed. It supports `--list-tasks` and tags work as expected. `include_tasks` is dynamic — processed at runtime. It supports loops and conditionals on the include itself. Use `import_*` by default; use `include_*` when you need conditional or looped includes.

---

## Modules

Modules are the building blocks of Ansible tasks. Each module performs a specific action on managed nodes. Ansible ships with thousands of built-in modules.

### Package Management

```yaml
# APT (Debian/Ubuntu)
- name: Install packages
  apt:
    name:
      - nginx
      - curl
      - git
    state: present
    update_cache: yes

- name: Remove a package
  apt:
    name: apache2
    state: absent
    purge: yes

- name: Upgrade all packages
  apt:
    upgrade: dist
    update_cache: yes

# YUM/DNF (RHEL/CentOS/Fedora)
- name: Install packages
  yum:
    name:
      - httpd
      - php
    state: present

# pip (Python)
- name: Install Python packages
  pip:
    name:
      - flask
      - gunicorn
    virtualenv: /opt/myapp/venv
    virtualenv_python: python3
```

`state: present` ensures the package is installed. `state: absent` removes it. `state: latest` installs or upgrades to the latest version. `purge: yes` removes the package along with its configuration files.

### File Management

```yaml
# Copy file from control node to managed node
- name: Copy configuration file
  copy:
    src: files/app.conf
    dest: /etc/app/app.conf
    owner: root
    group: root
    mode: "0644"
    backup: yes

# Copy inline content
- name: Create file with content
  copy:
    content: |
      server_name = {{ server_name }}
      port = {{ app_port }}
    dest: /etc/app/settings.conf

# Create directory
- name: Create app directory
  file:
    path: /opt/myapp
    state: directory
    owner: deploy
    group: deploy
    mode: "0755"

# Create symbolic link
- name: Create symlink
  file:
    src: /opt/myapp/current
    dest: /var/www/app
    state: link

# Delete file or directory
- name: Remove old logs
  file:
    path: /var/log/old-app
    state: absent

# Download file from URL
- name: Download binary
  get_url:
    url: https://example.com/app-v1.0.tar.gz
    dest: /tmp/app-v1.0.tar.gz
    checksum: sha256:abc123def456...

# Synchronize (rsync)
- name: Sync application files
  synchronize:
    src: ./app/
    dest: /opt/myapp/
    delete: yes
    rsync_opts:
      - "--exclude=.git"

# Unarchive
- name: Extract archive
  unarchive:
    src: /tmp/app-v1.0.tar.gz
    dest: /opt/myapp/
    remote_src: yes
```

`copy` transfers files from the control node. `file` manages file properties and can create directories, links, or delete files. `get_url` downloads from a URL with optional checksum verification. `synchronize` wraps rsync for efficient file syncing. `unarchive` extracts tar/zip archives. `remote_src: yes` means the archive is already on the managed node.

### Line-in-File & Blockinfile

```yaml
# Ensure a line exists in a file
- name: Enable IP forwarding
  lineinfile:
    path: /etc/sysctl.conf
    regexp: "^net.ipv4.ip_forward"
    line: "net.ipv4.ip_forward = 1"
    state: present

# Remove a line
- name: Remove old entry
  lineinfile:
    path: /etc/hosts
    regexp: "^192.168.1.50"
    state: absent

# Insert a block of text
- name: Add SSH config block
  blockinfile:
    path: /etc/ssh/sshd_config
    marker: "# {mark} ANSIBLE MANAGED BLOCK"
    block: |
      Match Group developers
        AllowTcpForwarding yes
        X11Forwarding no
```

`lineinfile` ensures a single line is present, absent, or modified in a file. `regexp` finds the line to replace. `blockinfile` manages a block of text delimited by markers. `{mark}` is replaced with `BEGIN` and `END`.

### User & Group Management

```yaml
- name: Create application group
  group:
    name: appgroup
    gid: 1500
    state: present

- name: Create application user
  user:
    name: deploy
    uid: 1500
    group: appgroup
    groups: sudo,docker
    shell: /bin/bash
    home: /home/deploy
    create_home: yes
    generate_ssh_key: yes
    ssh_key_bits: 4096
    state: present

- name: Add authorized key
  authorized_key:
    user: deploy
    key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    state: present
```

`user` creates, modifies, or removes user accounts. `groups` is a comma-separated list of supplementary groups. `generate_ssh_key` creates an SSH key pair for the user. `authorized_key` manages SSH authorized keys.

### Service Management

```yaml
- name: Start and enable Nginx
  service:
    name: nginx
    state: started
    enabled: yes

- name: Restart a service
  service:
    name: nginx
    state: restarted

# systemd-specific options
- name: Reload systemd and restart service
  systemd:
    name: myapp
    state: restarted
    daemon_reload: yes
    enabled: yes
```

`state: started` starts the service if not running. `state: stopped` stops it. `state: restarted` always restarts. `state: reloaded` reloads configuration without restarting. `enabled: yes` configures the service to start on boot. `daemon_reload: yes` runs `systemctl daemon-reload` before managing the service.

### Command Execution

```yaml
# command: run a command (no shell features)
- name: Check disk space
  command: df -h /
  register: disk_output
  changed_when: false

# shell: run through /bin/sh (supports pipes, redirects)
- name: Find large files
  shell: find /var/log -type f -size +100M | head -20
  register: large_files
  changed_when: false

# script: run a local script on remote hosts
- name: Run setup script
  script: scripts/setup.sh
  args:
    creates: /opt/myapp/.installed

# raw: run a command without Python (for bootstrap)
- name: Install Python on minimal systems
  raw: apt-get install -y python3
  become: yes
```

`command` is safer — it does not invoke a shell, so pipes and redirects do not work. `shell` passes the command through `/bin/sh`, enabling shell features. `register` saves the output to a variable. `changed_when: false` tells Ansible this task never makes changes (useful for read-only commands). `creates` skips the task if the specified file already exists. `raw` executes directly without the Ansible module subsystem, useful when Python is not yet installed on the managed node.

### Cron Jobs

```yaml
- name: Schedule daily backup
  cron:
    name: "Daily backup"
    minute: "0"
    hour: "2"
    job: "/opt/scripts/backup.sh >> /var/log/backup.log 2>&1"
    user: root
    state: present

- name: Remove a cron job
  cron:
    name: "Old cleanup job"
    state: absent
```

`cron` manages crontab entries. `name` is a description used as a unique identifier — Ansible uses it to find and update existing entries.

### Docker

```yaml
- name: Start Redis container
  docker_container:
    name: redis
    image: redis:7
    state: started
    restart_policy: always
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

- name: Pull Docker image
  docker_image:
    name: nginx
    tag: latest
    source: pull
```

The `docker_container` and `docker_image` modules require the `docker` Python package on the managed node and the `community.docker` collection.

---

## Variables

Variables make playbooks reusable and configurable. They can be defined in many places with a clear precedence order.

### Defining Variables

```yaml
# In a playbook
- name: Configure app
  hosts: webservers
  vars:
    app_name: myapp
    app_port: 8080
    app_env: production
    packages:
      - nginx
      - python3
      - git
    db_config:
      host: db.example.com
      port: 5432
      name: mydb

  tasks:
    - name: Install packages
      apt:
        name: "{{ packages }}"
        state: present
```

### Variable Files

```yaml
# In a playbook, reference external variable files
- name: Configure app
  hosts: webservers
  vars_files:
    - vars/common.yml
    - "vars/{{ environment }}.yml"

  tasks:
    - name: Debug
      debug:
        msg: "App: {{ app_name }}, Port: {{ app_port }}"
```

```yaml
# vars/common.yml
app_name: myapp
log_level: info

# vars/production.yml
app_port: 80
debug_mode: false
replicas: 3
```

`vars_files` loads variables from external YAML files. File paths can include variable interpolation, allowing environment-specific configs.

### Group & Host Variables

```yaml
# group_vars/all.yml — applies to every host
---
ntp_server: time.google.com
timezone: UTC
ansible_user: deploy

# group_vars/webservers.yml — applies to webservers group
---
http_port: 80
nginx_worker_processes: auto
ssl_enabled: true

# host_vars/web1.example.com.yml — applies to one host
---
nginx_worker_processes: 4
ssl_cert_path: /etc/ssl/web1.pem
```

Ansible automatically loads variables from `group_vars/<group>.yml` and `host_vars/<hostname>.yml`. No configuration needed — the directory names are conventions that Ansible recognizes.

### Variable Precedence (Lowest to Highest)

| Priority | Source                                    |
| -------- | ----------------------------------------- |
| 1        | Role defaults (`roles/x/defaults/main.yml`) |
| 2        | Inventory file variables                   |
| 3        | Inventory `group_vars/`                    |
| 4        | Inventory `host_vars/`                     |
| 5        | Playbook `group_vars/`                     |
| 6        | Playbook `host_vars/`                      |
| 7        | Play `vars`                                |
| 8        | Play `vars_files`                          |
| 9        | Play `vars_prompt`                         |
| 10       | Registered variables                       |
| 11       | `set_fact` / `include_vars`                |
| 12       | Role vars (`roles/x/vars/main.yml`)        |
| 13       | Task `vars`                                |
| 14       | Block `vars`                               |
| 15       | Extra vars (`-e` on command line) — **always wins** |

Extra variables passed with `-e` have the highest precedence and override everything else. Role defaults have the lowest precedence and are designed to be easily overridden.

### Registered Variables

```yaml
- name: Check if config exists
  stat:
    path: /etc/app/config.yml
  register: config_file

- name: Create config if missing
  copy:
    src: config.yml
    dest: /etc/app/config.yml
  when: not config_file.stat.exists

- name: Run health check
  uri:
    url: "http://localhost:{{ app_port }}/health"
    return_content: yes
  register: health_check

- name: Show health status
  debug:
    msg: "Status: {{ health_check.status }}, Body: {{ health_check.content }}"
```

`register` stores the result of a task in a variable. The variable contains return values like `.rc` (return code), `.stdout`, `.stderr`, `.changed`, and module-specific fields.

### Prompting for Variables

```yaml
- name: Deploy application
  hosts: webservers
  vars_prompt:
    - name: deploy_version
      prompt: "Which version to deploy?"
      default: "latest"
      private: no

    - name: db_password
      prompt: "Enter database password"
      private: yes
      encrypt: sha512_crypt
      confirm: yes

  tasks:
    - name: Show version
      debug:
        msg: "Deploying version {{ deploy_version }}"
```

`vars_prompt` asks for input before the play runs. `private: yes` hides the input (like a password prompt). `encrypt` hashes the input with the specified algorithm. `confirm: yes` asks the user to enter the value twice.

---

## Facts & Magic Variables

Facts are system information automatically gathered from managed nodes at the beginning of each play.

### Gathering Facts

```yaml
- name: Show system info
  hosts: all
  gather_facts: yes                    # Default: yes

  tasks:
    - name: Show OS
      debug:
        msg: "{{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Show IP
      debug:
        msg: "{{ ansible_default_ipv4.address }}"

    - name: Show memory
      debug:
        msg: "{{ ansible_memtotal_mb }} MB total memory"
```

### Common Facts

```yaml
ansible_hostname                       # Short hostname
ansible_fqdn                           # Fully qualified domain name
ansible_distribution                   # OS name (Ubuntu, CentOS, etc.)
ansible_distribution_version           # OS version (22.04, 9.3, etc.)
ansible_distribution_release           # OS release codename (jammy, etc.)
ansible_os_family                      # OS family (Debian, RedHat, etc.)
ansible_architecture                   # CPU architecture (x86_64, aarch64)
ansible_kernel                         # Kernel version
ansible_default_ipv4.address           # Primary IPv4 address
ansible_default_ipv4.interface         # Primary network interface
ansible_all_ipv4_addresses             # List of all IPv4 addresses
ansible_memtotal_mb                    # Total memory in MB
ansible_processor_vcpus                # Number of CPU cores
ansible_devices                        # Disk devices
ansible_mounts                         # Mounted filesystems
ansible_env                            # Environment variables
ansible_pkg_mgr                        # Package manager (apt, yum, dnf)
ansible_service_mgr                    # Service manager (systemd, etc.)
```

### Viewing All Facts

```bash
# Ad-hoc command
ansible web1.example.com -m setup

# Filter specific facts
ansible web1.example.com -m setup -a "filter=ansible_distribution*"
ansible web1.example.com -m setup -a "filter=ansible_memory_mb"
```

### Custom Facts

Place custom fact files on managed nodes at `/etc/ansible/facts.d/`. Files must be `.fact` extension and return JSON or INI.

```ini
# /etc/ansible/facts.d/app.fact (on managed node)
[app]
version=2.1.0
environment=production
```

Access with `ansible_local.app.version`:

```yaml
- name: Show custom fact
  debug:
    msg: "App version: {{ ansible_local.app.version }}"
```

### Set Facts at Runtime

```yaml
- name: Calculate memory threshold
  set_fact:
    memory_threshold_mb: "{{ (ansible_memtotal_mb * 0.8) | int }}"
    cacheable: yes

- name: Use calculated fact
  debug:
    msg: "Memory threshold: {{ memory_threshold_mb }} MB"
```

`set_fact` creates variables at runtime based on other facts or task results. `cacheable: yes` persists the fact across plays if fact caching is enabled.

### Magic Variables

```yaml
inventory_hostname                     # Current host's name as in inventory
inventory_hostname_short               # Short version (before first dot)
ansible_play_hosts                     # All hosts in the current play
ansible_play_batch                     # Hosts in the current batch (serial)
groups                                 # Dictionary of all groups and hosts
groups['webservers']                   # List of hosts in webservers group
group_names                            # List of groups current host belongs to
hostvars                               # Variables for all hosts
hostvars['web1.example.com']           # Variables for a specific host
ansible_check_mode                     # True if running in check mode
ansible_version                        # Ansible version info
role_path                              # Path to the current role
playbook_dir                           # Directory of the playbook
```

```yaml
# Access another host's variables
- name: Show DB host IP
  debug:
    msg: "DB IP: {{ hostvars['db1.example.com']['ansible_default_ipv4']['address'] }}"
```

`hostvars` lets you access any host's facts and variables, even from a different group.

---

## Conditionals

Conditionals control whether a task runs based on variables, facts, or previous task results.

### when

```yaml
# Based on OS
- name: Install on Debian/Ubuntu
  apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"

- name: Install on RHEL/CentOS
  yum:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

# Boolean
- name: Enable SSL
  template:
    src: ssl.conf.j2
    dest: /etc/nginx/conf.d/ssl.conf
  when: ssl_enabled | default(false)

# Variable defined
- name: Set custom DNS
  template:
    src: resolv.conf.j2
    dest: /etc/resolv.conf
  when: custom_dns is defined

# Multiple conditions (AND)
- name: Configure production web server
  template:
    src: prod.conf.j2
    dest: /etc/app/config.conf
  when:
    - environment == "production"
    - ansible_os_family == "Debian"
    - ansible_memtotal_mb >= 2048

# OR condition
- name: Install on Debian or RedHat
  package:
    name: curl
    state: present
  when: ansible_os_family == "Debian" or ansible_os_family == "RedHat"

# Based on registered variable
- name: Check if app is running
  command: pgrep myapp
  register: app_status
  ignore_errors: yes
  changed_when: false

- name: Start app if not running
  command: /opt/myapp/start.sh
  when: app_status.rc != 0

# String tests
- name: Run on hosts starting with web
  debug:
    msg: "This is a web server"
  when: inventory_hostname is match("web.*")

# Version comparison
- name: Only on Ubuntu 22+
  debug:
    msg: "Modern Ubuntu"
  when: ansible_distribution == "Ubuntu" and ansible_distribution_major_version | int >= 22
```

`when` takes a raw Jinja2 expression (no `{{ }}`). Multiple list items are joined with `and`. `is defined` / `is not defined` checks if a variable exists. `| default(false)` provides a fallback. `.rc` is the return code from a command (0 = success).

### Block Conditionals

```yaml
- name: Install and configure on Debian
  block:
    - name: Install packages
      apt:
        name:
          - nginx
          - certbot
        state: present

    - name: Copy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf

    - name: Start service
      service:
        name: nginx
        state: started
  when: ansible_os_family == "Debian"
  become: yes
```

`block` groups multiple tasks and applies `when`, `become`, and error handling to all of them. This avoids repeating `when` on every task.

---

## Loops

Loops repeat a task over a list of items.

### Simple Loop

```yaml
- name: Install multiple packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl
    - vim

# Shorthand (preferred for apt)
- name: Install multiple packages
  apt:
    name:
      - nginx
      - git
      - curl
      - vim
    state: present
```

`loop` iterates over a list. `{{ item }}` refers to the current element. For the `apt` module, passing a list directly to `name` is more efficient than looping.

### Loop with Dictionaries

```yaml
- name: Create multiple users
  user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: "alice", uid: 1001, groups: "sudo,docker" }
    - { name: "bob", uid: 1002, groups: "docker" }
    - { name: "carol", uid: 1003, groups: "developers" }
```

Each `item` is a dictionary, and you access fields with `item.name`, `item.uid`, etc.

### Loop with Index

```yaml
- name: Create files with index
  copy:
    content: "Server {{ ansible_loop.index }}"
    dest: "/tmp/server-{{ ansible_loop.index }}.txt"
  loop:
    - alpha
    - beta
    - gamma
  loop_control:
    extended: yes
```

`loop_control.extended: yes` enables `ansible_loop.index` (1-based), `ansible_loop.index0` (0-based), `ansible_loop.first`, `ansible_loop.last`, and `ansible_loop.length`.

### Loop Control

```yaml
- name: Process items with custom label
  debug:
    msg: "Processing {{ item.name }}"
  loop: "{{ users }}"
  loop_control:
    label: "{{ item.name }}"           # Show only name in output (less verbose)
    pause: 2                            # Wait 2 seconds between items
```

`label` customizes what is displayed in the output for each iteration. Without it, the entire item dictionary is printed, which can be noisy. `pause` adds a delay between iterations.

### Looping Over Dictionaries

```yaml
- name: Set sysctl parameters
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    sysctl_set: yes
  loop: "{{ sysctl_params | dict2items }}"
  vars:
    sysctl_params:
      net.ipv4.ip_forward: 1
      net.core.somaxconn: 65535
      vm.swappiness: 10
```

`dict2items` converts a dictionary to a list of `{key, value}` pairs, making it iterable with `loop`.

### Nested Loops

```yaml
- name: Grant database privileges
  mysql_user:
    name: "{{ item[0] }}"
    priv: "{{ item[1] }}.*:ALL"
    state: present
  with_nested:
    - ["alice", "bob"]
    - ["app_db", "analytics_db"]
```

`with_nested` creates a cross-product of the lists. This example creates privileges for each user on each database.

### Loop with Conditionals

```yaml
- name: Start only enabled services
  service:
    name: "{{ item.name }}"
    state: started
  loop:
    - { name: "nginx", enabled: true }
    - { name: "redis", enabled: true }
    - { name: "memcached", enabled: false }
  when: item.enabled
```

`when` is evaluated for each iteration. Items that do not match are skipped.

---

## Handlers & Notifications

Handlers are special tasks that run only when notified and execute once at the end of the play, regardless of how many tasks notify them.

### Basic Handlers

```yaml
- name: Configure web server
  hosts: webservers
  become: yes

  tasks:
    - name: Update Nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - Validate Nginx Config
        - Restart Nginx

    - name: Update SSL certificate
      copy:
        src: ssl/cert.pem
        dest: /etc/ssl/app.pem
      notify: Restart Nginx

    - name: Update application config
      template:
        src: app.conf.j2
        dest: /etc/app/config.yml
      notify: Restart App

  handlers:
    - name: Validate Nginx Config
      command: nginx -t
      changed_when: false
      listen: "validate nginx"

    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
      listen: "restart web"

    - name: Restart App
      service:
        name: myapp
        state: restarted
```

Handlers run only when the notifying task reports `changed`. If a task does not change anything (already in desired state), the handler is not triggered. `notify` takes a handler name or a list of names. `listen` allows multiple handlers to respond to the same notification topic.

### Flush Handlers

```yaml
- name: Update config
  template:
    src: app.conf.j2
    dest: /etc/app/config.yml
  notify: Restart App

# Force handlers to run NOW instead of at end of play
- meta: flush_handlers

- name: Run health check (needs app restarted first)
  uri:
    url: "http://localhost:{{ app_port }}/health"
  register: health
```

`meta: flush_handlers` forces all pending handlers to execute immediately. This is useful when subsequent tasks depend on the handler having run (e.g., a service restart before a health check).

---

## Templates (Jinja2)

Templates use the Jinja2 templating engine to generate dynamic configuration files. Template files have the `.j2` extension by convention.

### Basic Template

```jinja2
{# templates/nginx.conf.j2 #}

# Managed by Ansible - DO NOT EDIT MANUALLY
# Generated for {{ inventory_hostname }} on {{ ansible_date_time.iso8601 }}

worker_processes {{ nginx_worker_processes | default('auto') }};

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
}

http {
    server {
        listen {{ http_port | default(80) }};
        server_name {{ server_name }};
        root {{ document_root | default('/var/www/html') }};

{% if ssl_enabled | default(false) %}
        listen 443 ssl;
        ssl_certificate {{ ssl_cert_path }};
        ssl_certificate_key {{ ssl_key_path }};
{% endif %}

{% for location in app_locations | default([]) %}
        location {{ location.path }} {
            proxy_pass http://{{ location.backend }};
        }
{% endfor %}
    }
}
```

`{{ }}` outputs a variable. `{% %}` contains logic (if/for). `{# #}` is a comment. `| default('value')` provides a fallback.

### Using Templates in Tasks

```yaml
- name: Deploy Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    validate: "nginx -t -c %s"
    backup: yes
  notify: Restart Nginx
```

`template` renders the Jinja2 file with current variables and copies the result to the managed node. `validate` runs a command to check the generated file before placing it (%s is replaced with the temp file path). `backup` creates a backup of the original file.

### Jinja2 Filters

```jinja2
{# String operations #}
{{ name | upper }}                     {# BIDUR #}
{{ name | lower }}                     {# bidur #}
{{ name | capitalize }}                {# Bidur #}
{{ name | replace("old", "new") }}     {# string replacement #}
{{ items | join(", ") }}               {# join list into string #}
{{ path | basename }}                  {# file.txt from /path/to/file.txt #}
{{ path | dirname }}                   {# /path/to from /path/to/file.txt #}

{# Default values #}
{{ variable | default("fallback") }}   {# use fallback if undefined #}
{{ variable | default(omit) }}         {# omit parameter if undefined #}

{# Type conversions #}
{{ "42" | int }}                       {# convert to integer #}
{{ 42 | string }}                      {# convert to string #}
{{ "true" | bool }}                    {# convert to boolean #}
{{ items | to_json }}                  {# convert to JSON string #}
{{ items | to_yaml }}                  {# convert to YAML string #}
{{ items | to_nice_json(indent=2) }}   {# pretty JSON #}

{# List operations #}
{{ items | length }}                   {# list length #}
{{ items | first }}                    {# first element #}
{{ items | last }}                     {# last element #}
{{ items | unique }}                   {# remove duplicates #}
{{ items | sort }}                     {# sort list #}
{{ items | flatten }}                  {# flatten nested lists #}
{{ items | select("match", "web.*") }} {# filter by regex #}
{{ items | map(attribute="name") | list }}  {# extract attribute #}

{# Math #}
{{ values | max }}                     {# maximum value #}
{{ values | min }}                     {# minimum value #}
{{ values | sum }}                     {# sum of values #}
{{ 1024 | human_readable }}            {# 1.0 KB #}

{# Hashing & encoding #}
{{ password | password_hash("sha512") }}  {# hash password #}
{{ data | b64encode }}                 {# base64 encode #}
{{ data | b64decode }}                 {# base64 decode #}
{{ file_content | hash("md5") }}       {# MD5 hash #}

{# IP address #}
{{ "192.168.1.0/24" | ipaddr("network") }}  {# 192.168.1.0 #}
{{ "192.168.1.5" | ipaddr }}           {# validate IP #}

{# Regex #}
{{ input | regex_search("v([0-9]+)", "\\1") }}  {# extract match #}
{{ input | regex_replace("^old", "new") }}      {# regex replace #}
```

### Template Control Structures

```jinja2
{# Conditional #}
{% if environment == "production" %}
log_level = warn
{% elif environment == "staging" %}
log_level = info
{% else %}
log_level = debug
{% endif %}

{# For loop #}
{% for server in backend_servers %}
upstream_server {{ server.host }}:{{ server.port }} weight={{ server.weight | default(1) }};
{% endfor %}

{# For loop with index and conditions #}
{% for user in users %}
{% if not loop.first %},{% endif %}
  "{{ user.name }}": {
    "role": "{{ user.role }}"
  }
{% endfor %}

{# Whitespace control (- strips whitespace) #}
{% for item in items -%}
{{ item }}
{%- endfor %}

{# Loop variables #}
{# loop.index     → 1-based index #}
{# loop.index0    → 0-based index #}
{# loop.first     → true on first iteration #}
{# loop.last      → true on last iteration #}
{# loop.length    → total iterations #}
```

### Lookups

```yaml
# Read a file on the control node
- name: Set SSH key
  authorized_key:
    user: deploy
    key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

# Read an environment variable
- name: Use env var
  debug:
    msg: "Home: {{ lookup('env', 'HOME') }}"

# Read from a password file (create if missing)
- name: Generate password
  debug:
    msg: "{{ lookup('password', '/tmp/app_password length=20 chars=ascii_letters,digits') }}"

# Read from a pipe
- name: Get git commit
  debug:
    msg: "{{ lookup('pipe', 'git rev-parse HEAD') }}"

# Read lines from a file
- name: Process each line
  debug:
    msg: "{{ item }}"
  loop: "{{ lookup('file', 'servers.txt').splitlines() }}"
```

Lookups run on the control node and return data from various sources. `file` reads a file. `env` reads an environment variable. `password` generates or retrieves a password. `pipe` runs a command and returns its output.

---

## Roles

Roles provide a structured way to organize playbooks into reusable components. A role encapsulates tasks, variables, files, templates, and handlers into a standardized directory structure.

### Role Structure

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml           # Main task list (entry point)
    ├── handlers/
    │   └── main.yml           # Handler definitions
    ├── templates/
    │   └── nginx.conf.j2      # Jinja2 templates
    ├── files/
    │   └── index.html         # Static files
    ├── vars/
    │   └── main.yml           # High-priority variables
    ├── defaults/
    │   └── main.yml           # Default variables (easily overridden)
    ├── meta/
    │   └── main.yml           # Role metadata and dependencies
    ├── tests/
    │   ├── inventory
    │   └── test.yml
    └── README.md
```

`tasks/main.yml` is automatically loaded when the role is used. `defaults/main.yml` contains default variables with the lowest precedence. `vars/main.yml` contains variables with high precedence. `handlers/main.yml` defines handlers available to the role's tasks. `meta/main.yml` declares dependencies on other roles. `templates/` and `files/` are automatically searched by the `template` and `copy` modules.

### Creating a Role

```bash
ansible-galaxy role init nginx         # Scaffold a new role
```

```yaml
# roles/nginx/defaults/main.yml
---
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_server_name: localhost
nginx_document_root: /var/www/html
nginx_ssl_enabled: false
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install Nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Deploy Nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: "nginx -t -c %s"
  notify: Restart Nginx

- name: Deploy default site
  copy:
    src: index.html
    dest: "{{ nginx_document_root }}/index.html"

- name: Ensure Nginx is running
  service:
    name: nginx
    state: started
    enabled: yes
```

```yaml
# roles/nginx/handlers/main.yml
---
- name: Restart Nginx
  service:
    name: nginx
    state: restarted

- name: Reload Nginx
  service:
    name: nginx
    state: reloaded
```

```yaml
# roles/nginx/meta/main.yml
---
dependencies:
  - role: common
  - role: firewall
    vars:
      firewall_allowed_ports:
        - "{{ nginx_port }}"
```

`dependencies` lists roles that must run before this role. Variables can be passed to dependent roles.

### Using Roles

```yaml
# playbooks/site.yml
---
- name: Configure web servers
  hosts: webservers
  become: yes

  roles:
    - common
    - nginx
    - role: app
      vars:
        app_port: 8080

# With role execution control
- name: Configure with tags
  hosts: webservers
  become: yes

  roles:
    - role: nginx
      tags: web
    - role: postgres
      tags: db
      when: "'dbservers' in group_names"
```

```yaml
# Include roles dynamically
- name: Dynamic role usage
  hosts: webservers
  become: yes

  tasks:
    - name: Common setup
      include_role:
        name: common

    - name: Web setup
      include_role:
        name: nginx
      vars:
        nginx_port: 8080
      when: install_nginx | default(true)
```

`roles:` applies roles at the beginning of the play (before tasks). `include_role` includes a role dynamically within the tasks section, supporting conditionals and loops.

---

## Ansible Galaxy

Ansible Galaxy is a repository for sharing and downloading roles and collections. It is also a command-line tool for managing them.

### Installing Roles

```bash
# Install from Galaxy
ansible-galaxy role install geerlingguy.docker
ansible-galaxy role install geerlingguy.nginx

# Install specific version
ansible-galaxy role install geerlingguy.docker,6.1.0

# Install from GitHub
ansible-galaxy role install git+https://github.com/user/role.git,main

# Install to a custom path
ansible-galaxy role install geerlingguy.docker -p ./roles/

# Install from requirements file
ansible-galaxy role install -r requirements.yml
```

### Requirements File

```yaml
# requirements.yml
roles:
  - name: geerlingguy.docker
    version: "6.1.0"
  - name: geerlingguy.nginx
    version: "3.2.0"
  - name: geerlingguy.postgresql
  - src: https://github.com/user/custom-role.git
    scm: git
    version: main
    name: custom-role

collections:
  - name: community.docker
    version: ">=3.0.0"
  - name: amazon.aws
    version: "7.0.0"
  - name: community.general
```

```bash
ansible-galaxy install -r requirements.yml  # Install roles + collections
```

### Collections

Collections are a packaging format that bundles modules, plugins, roles, and playbooks.

```bash
# Install a collection
ansible-galaxy collection install community.docker
ansible-galaxy collection install amazon.aws

# Install from requirements file
ansible-galaxy collection install -r requirements.yml

# List installed collections
ansible-galaxy collection list
```

### Managing Roles

```bash
ansible-galaxy role list                # List installed roles
ansible-galaxy role remove geerlingguy.docker  # Remove a role
ansible-galaxy role search nginx       # Search Galaxy for roles
ansible-galaxy role info geerlingguy.nginx  # Show role details
```

---

## Ansible Vault

Ansible Vault encrypts sensitive data (passwords, API keys, certificates) so they can be safely committed to version control.

### Encrypting Files

```bash
# Create a new encrypted file
ansible-vault create vault/secrets.yml

# Encrypt an existing file
ansible-vault encrypt group_vars/production.yml

# Decrypt a file
ansible-vault decrypt group_vars/production.yml

# View encrypted file without decrypting
ansible-vault view vault/secrets.yml

# Edit encrypted file (decrypts, opens editor, re-encrypts)
ansible-vault edit vault/secrets.yml

# Change vault password
ansible-vault rekey vault/secrets.yml
```

### Encrypting Strings

```bash
# Encrypt a single variable
ansible-vault encrypt_string 'SuperSecretPassword' --name 'db_password'
```

Output:

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  6630343036353135623833...
```

Paste this directly into your variable files. The encrypted value can be stored in version control safely.

### Using Vault in Playbooks

```yaml
# group_vars/production.yml
---
app_name: myapp
db_host: db.example.com

# Encrypted values inline
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  6630343036353135623833...

api_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  3861376131353631323562...
```

```yaml
# Or reference a separate encrypted file
- name: Deploy app
  hosts: webservers
  vars_files:
    - vars/common.yml
    - vault/secrets.yml             # Encrypted file

  tasks:
    - name: Configure database
      template:
        src: db.conf.j2
        dest: /etc/app/db.conf
```

### Running with Vault

```bash
# Prompt for vault password
ansible-playbook site.yml --ask-vault-pass

# Use a password file
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Use environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook site.yml
```

The vault password file should contain the password as a single line and be excluded from version control (`chmod 600 ~/.vault_pass`).

### Multiple Vault IDs

```bash
# Encrypt with a specific vault ID
ansible-vault encrypt --vault-id prod@prompt group_vars/production.yml
ansible-vault encrypt --vault-id dev@~/.vault_pass_dev group_vars/dev.yml

# Run with multiple vault IDs
ansible-playbook site.yml \
  --vault-id prod@prompt \
  --vault-id dev@~/.vault_pass_dev
```

Vault IDs allow different passwords for different environments. Each encrypted file is tagged with its vault ID.

---

## Error Handling

### Ignoring Errors

```yaml
- name: Check if service exists
  command: systemctl status myapp
  register: service_status
  ignore_errors: yes

- name: Install if not present
  apt:
    name: myapp
    state: present
  when: service_status.rc != 0
```

`ignore_errors: yes` allows the playbook to continue even if the task fails. The task is marked as failed but does not stop execution.

### Failed When

```yaml
- name: Run health check
  uri:
    url: "http://localhost:8080/health"
    return_content: yes
  register: health_check
  failed_when: "'healthy' not in health_check.content"

- name: Check application logs
  shell: tail -100 /var/log/app.log | grep -c ERROR
  register: error_count
  failed_when: error_count.stdout | int > 10
  changed_when: false
```

`failed_when` overrides when Ansible considers a task failed. This lets you define custom failure conditions based on output content, return codes, or registered variables.

### Changed When

```yaml
- name: Run database migration
  command: /opt/app/manage.py migrate
  register: migration_output
  changed_when: "'No migrations to apply' not in migration_output.stdout"

- name: Check config syntax
  command: nginx -t
  changed_when: false                  # This never changes anything
```

`changed_when` controls when a task is reported as "changed". Setting it to `false` for read-only commands keeps output clean and prevents handlers from being triggered unnecessarily.

### Block / Rescue / Always

```yaml
- name: Deploy application
  block:
    - name: Pull latest code
      git:
        repo: "https://github.com/myorg/myapp.git"
        dest: /opt/myapp
        version: "{{ app_version }}"

    - name: Install dependencies
      pip:
        requirements: /opt/myapp/requirements.txt
        virtualenv: /opt/myapp/venv

    - name: Run migrations
      command: /opt/myapp/venv/bin/python manage.py migrate
      args:
        chdir: /opt/myapp

    - name: Restart application
      service:
        name: myapp
        state: restarted

  rescue:
    - name: Rollback to previous version
      git:
        repo: "https://github.com/myorg/myapp.git"
        dest: /opt/myapp
        version: "{{ previous_version }}"

    - name: Restart with old version
      service:
        name: myapp
        state: restarted

    - name: Send failure notification
      slack:
        token: "{{ slack_token }}"
        channel: "#deployments"
        msg: "Deployment of {{ app_version }} FAILED. Rolled back to {{ previous_version }}."

  always:
    - name: Clean up temp files
      file:
        path: /tmp/deploy-artifacts
        state: absent

    - name: Log deployment result
      lineinfile:
        path: /var/log/deployments.log
        line: "{{ ansible_date_time.iso8601 }} - {{ app_version }} - {{ 'FAILED' if ansible_failed_task is defined else 'SUCCESS' }}"
        create: yes
```

`block` groups tasks together. `rescue` runs only if a task in the block fails — like try/catch. `always` runs regardless of success or failure — like finally. This pattern is essential for safe deployments with automatic rollback.

### Retries

```yaml
- name: Wait for application to be ready
  uri:
    url: "http://localhost:{{ app_port }}/health"
    status_code: 200
  register: health
  until: health.status == 200
  retries: 30
  delay: 10
```

`until` retries the task until the condition is true. `retries` is the maximum number of attempts (default: 3). `delay` is seconds between retries (default: 5). This example waits up to 5 minutes (30 × 10s) for the application to become healthy.

### Any Errors Fatal

```yaml
- name: Critical setup
  hosts: webservers
  any_errors_fatal: true               # Stop on ALL hosts if ANY host fails

  tasks:
    - name: Verify connectivity
      wait_for:
        port: 443
        timeout: 30
```

`any_errors_fatal: true` stops execution on all hosts if any single host fails. Without this, Ansible only removes the failed host and continues on the rest. Use it for tasks where partial completion is dangerous (e.g., database migrations, cluster updates).

---

## Tags

Tags let you selectively run or skip specific tasks, roles, or plays.

### Tagging Tasks

```yaml
- name: Install base packages
  apt:
    name:
      - vim
      - curl
      - git
    state: present
  tags:
    - packages
    - setup

- name: Configure NTP
  template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
  tags:
    - ntp
    - config

- name: Deploy application
  git:
    repo: "{{ app_repo }}"
    dest: /opt/myapp
    version: "{{ app_version }}"
  tags:
    - deploy
    - app

- name: Always run this
  debug:
    msg: "This always runs"
  tags: always
```

`tags: always` ensures the task runs regardless of which tags are specified on the command line (unless explicitly skipped with `--skip-tags always`).

### Running with Tags

```bash
ansible-playbook site.yml --tags deploy          # Run only "deploy" tasks
ansible-playbook site.yml --tags "deploy,config" # Run deploy and config tasks
ansible-playbook site.yml --skip-tags setup      # Skip setup tasks
ansible-playbook site.yml --list-tags            # List all available tags
```

### Tagging Roles and Includes

```yaml
roles:
  - role: common
    tags: common
  - role: nginx
    tags: web
  - role: postgres
    tags: database

tasks:
  - import_tasks: tasks/deploy.yml
    tags: deploy
```

When you tag a role, all tasks within that role inherit the tag.

---

## Delegation & Local Actions

### Delegate To

```yaml
# Run a task on a different host
- name: Remove server from load balancer
  command: /opt/lb/remove_server.sh {{ inventory_hostname }}
  delegate_to: lb1.example.com

# Run a task on the control node
- name: Add DNS record
  route53:
    zone: example.com
    record: "{{ inventory_hostname }}"
    type: A
    value: "{{ ansible_default_ipv4.address }}"
  delegate_to: localhost

# Wait for port on the managed node from control node
- name: Wait for server to come up
  wait_for:
    host: "{{ ansible_host }}"
    port: 22
    delay: 10
    timeout: 300
  delegate_to: localhost
```

`delegate_to` runs the task on a different host than the play target. `delegate_to: localhost` runs on the control node. This is useful for orchestration tasks like updating load balancers, DNS records, or monitoring systems.

### Local Action

```yaml
# Shorthand for delegate_to: localhost
- name: Create local temp directory
  local_action:
    module: file
    path: /tmp/deploy-{{ inventory_hostname }}
    state: directory

# Equivalent to:
- name: Create local temp directory
  file:
    path: /tmp/deploy-{{ inventory_hostname }}
    state: directory
  delegate_to: localhost
```

### Run Once

```yaml
- name: Run database migration (only once)
  command: /opt/app/manage.py migrate
  run_once: true

- name: Send deployment notification
  slack:
    token: "{{ slack_token }}"
    channel: "#deployments"
    msg: "Deployment complete on {{ ansible_play_hosts | length }} hosts"
  run_once: true
  delegate_to: localhost
```

`run_once: true` executes the task on the first host only, even if the play targets multiple hosts. Combined with `delegate_to: localhost`, it runs a task exactly once on the control node.

### Serial Execution

```yaml
- name: Rolling update
  hosts: webservers
  serial: 2                            # Update 2 hosts at a time
  become: yes

  tasks:
    - name: Remove from load balancer
      command: /opt/lb/remove.sh {{ inventory_hostname }}
      delegate_to: lb1.example.com

    - name: Update application
      git:
        repo: "{{ app_repo }}"
        dest: /opt/myapp
        version: "{{ app_version }}"

    - name: Restart application
      service:
        name: myapp
        state: restarted

    - name: Wait for health check
      uri:
        url: "http://{{ ansible_host }}:{{ app_port }}/health"
      retries: 10
      delay: 5
      until: health.status == 200
      register: health

    - name: Add back to load balancer
      command: /opt/lb/add.sh {{ inventory_hostname }}
      delegate_to: lb1.example.com
```

`serial: 2` runs the play on 2 hosts at a time instead of all hosts in parallel. You can also use a percentage: `serial: "25%"`. This is essential for zero-downtime rolling deployments.

```yaml
# Graduated serial execution
serial:
  - 1                                  # First batch: 1 host (canary)
  - 5                                  # Second batch: 5 hosts
  - "100%"                             # Remaining: all at once
```

---

## Ansible Configuration

### ansible.cfg

Ansible reads configuration from `ansible.cfg` in the following order (first found wins):

1. `ANSIBLE_CONFIG` environment variable
2. `./ansible.cfg` (current directory)
3. `~/.ansible.cfg` (home directory)
4. `/etc/ansible/ansible.cfg` (global)

```ini
# ansible.cfg
[defaults]
inventory = ./inventory/production
remote_user = deploy
private_key_file = ~/.ssh/deploy_key
host_key_checking = False
retry_files_enabled = False
forks = 20
timeout = 30
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
stdout_callback = yaml
callbacks_enabled = timer, profile_tasks
roles_path = ./roles:~/.ansible/roles:/etc/ansible/roles
collections_path = ./collections:~/.ansible/collections

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
control_path_dir = /tmp/.ansible/cp
```

`forks` controls how many hosts Ansible manages in parallel (default: 5). `gathering = smart` caches facts and only gathers them once per host. `pipelining = True` reduces SSH operations by executing modules without copying them, significantly improving performance. `stdout_callback = yaml` makes output more readable. `host_key_checking = False` disables SSH host key verification (convenient for development, risky for production).

### Environment Variables

```bash
export ANSIBLE_CONFIG=./ansible.cfg
export ANSIBLE_INVENTORY=./inventory/production
export ANSIBLE_REMOTE_USER=deploy
export ANSIBLE_FORKS=20
export ANSIBLE_HOST_KEY_CHECKING=False
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
export ANSIBLE_STDOUT_CALLBACK=yaml
```

Environment variables override `ansible.cfg` settings and use the prefix `ANSIBLE_`.

---

## Best Practices

### Playbook Organization

- **Use roles** for reusable, testable components. Keep playbooks thin — they should mostly assemble roles.
- **One play per purpose**: Separate web server, database, and monitoring into different plays or playbooks.
- **Use `site.yml` as the main entrypoint** that imports other playbooks.
- **Keep tasks small and focused**: Each task should do one thing.
- **Use meaningful names** for every play, task, handler, and variable.
- **Use `ansible-lint`** to catch common mistakes and enforce style.

### Variables

- **Use `defaults/main.yml`** in roles for variables that users should override.
- **Use `vars/main.yml`** in roles for internal variables that should not change.
- **Use `group_vars/` and `host_vars/`** for environment-specific values.
- **Never hardcode values** in tasks — use variables instead.
- **Prefix role variables** with the role name to avoid collisions (e.g., `nginx_port`, `postgres_max_connections`).

### Security

- **Use Ansible Vault** for all secrets (passwords, API keys, certificates).
- **Never commit unencrypted secrets** to version control.
- **Use SSH key authentication** instead of passwords.
- **Restrict `become`** to tasks that need it, not entire plays.
- **Use `no_log: true`** on tasks that handle sensitive data.

```yaml
- name: Set database password
  mysql_user:
    name: app
    password: "{{ db_password }}"
  no_log: true                         # Hide output from logs
```

### Idempotency

- **Use modules instead of `command`/`shell`** whenever possible. Modules are idempotent; raw commands are not.
- **Use `creates` and `removes`** with `command`/`shell` to make them idempotent.
- **Use `changed_when`** to accurately report changes for command tasks.
- **Test idempotency** by running the playbook twice — the second run should show zero changes.

### Performance

- **Enable `pipelining`** in `ansible.cfg` for faster SSH execution.
- **Increase `forks`** for large inventories (default 5 is too low for hundreds of hosts).
- **Use `gathering = smart`** with fact caching to avoid redundant fact collection.
- **Use `free` strategy** when task order does not matter across hosts.

```yaml
- name: Fast parallel tasks
  hosts: all
  strategy: free                       # Don't wait for slowest host
  tasks:
    - name: Update packages
      apt:
        upgrade: dist
```

- **Limit fact gathering** when you do not need facts: `gather_facts: no`.
- **Use `async` and `poll`** for long-running tasks.

```yaml
- name: Run long database backup
  command: /opt/scripts/backup.sh
  async: 3600                          # Allow up to 1 hour
  poll: 0                              # Don't wait (fire and forget)
  register: backup_job

- name: Wait for backup to finish
  async_status:
    jid: "{{ backup_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 60
```

`async` runs the task in the background. `poll: 0` means do not wait for it to finish. `async_status` checks the job status later.

### Testing

```bash
# Lint your playbooks
ansible-lint playbooks/site.yml

# Syntax check
ansible-playbook site.yml --syntax-check

# Dry run with diff
ansible-playbook site.yml --check --diff

# Run on a single host first
ansible-playbook site.yml --limit web1.example.com

# Use Molecule for role testing
pip install molecule molecule-docker
cd roles/nginx
molecule init scenario
molecule test
```

`ansible-lint` catches common mistakes, deprecated features, and style issues. `molecule` is a testing framework that provisions temporary infrastructure (Docker containers), runs your role, and validates the results. Always test changes on a single host or staging environment before applying to production.

### Common `.gitignore`

```gitignore
# Ansible
*.retry
.vault_pass
*.vault_pass*

# Sensitive variable files
**/vault/*.yml
!**/vault/*.yml.example

# Fact cache
/tmp/ansible_facts/

# Downloaded roles and collections
roles/external/
collections/
.ansible/
```
