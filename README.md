# Ansible Learning Project - RHEL 10.1 Hardening (in progress)

A structured learning environment for **Ansible** using a RHEL 10.1 control node.
The goal is to design, test and evolve **CIS-oriented hardening playbooks** against managed RHEL 10.1 nodes.


## Project Strucutre

```sh
ansible-project/
├── ansible.cfg            # Ansible configuration
├── .gitignore             # Git ignore rules
├── inventory/
│   ├── hosts.ini          # Managed node inventory (ignored by git)
│   └── hosts.ini.example  # Inventory template
├── playbooks/
│   ├── hello.yml          # First simple playbook
│   └── hardening.yml      # CIS-oriented hardening playbook (in progress)
└── roles/                 # Roles go here (future)
```


## Requirements

- RHEL 10.1 control node with ansible-core installed
- One or more RHEL 10.1 managed node reachable via SSH
- SSH key-based authentication configured for the ansible user
- EPEL repository (installed automatically by hello.yml including GPG key import)


## Setup

```sh
# Install Ansible
sudo dnf install -y ansible-core

# Clone this repo
git clone https://github.com/codingoak/ansible-project.git
cd ~/ansible-project

# Create inventory from example and add your managed node IP
cp inventory/hosts.ini.example inventory/hosts.ini
vim inventory/hosts.ini
```


## Usage

```sh
# Test connectivity
ansible managed -m ping

# Run first playbook
ansible-playbook playbooks/hello.yml
```


## Hardening Playbook (hardening.yml)

The hardening.yml playbook implements a CIS‑oriented baseline hardening
for RHEL 10.1 systems. It is intentionally developed step by step, with each
hardening area grouped into logical blocks that can be executed and tested
independently.

### Structure and philosophy

The playbook is divided into numbered blocks, each addressing a specific
hardening topic (system updates, SSH, SELinux, audit logging, etc.).

Each block is:

- clearly scoped
- tagged for selective execution
- committed separately for clean Git history and easier review

This mirrors how Ansible hardening playbooks are typically developed and
maintained in professional environments.

Example:

```sh
ansible-playbook playbooks/hardening.yml --tags update
```

### Block 1 – System updates

Block 1 establishes a secure and up‑to‑date system baseline:

- Updates all installed packages
- Installs dnf-automatic
- Enables **dnf-automatic-install.timer** on RHEL 10

RHEL 10 provides multiple dnf-automatic timers.
This project explicitly enables **dnf-automatic-install.timer**, which
downloads and installs updates automatically, independent of the settings in /etc/dnf/automatic.conf.


### Block 2 – Remove insecure packages

Block 2 removes legacy and insecure packages that are not needed on a modern RHEL 10 server:

| Package | Reason |
|---|---|
| `telnet` | Transmits data including passwords in plaintext |
| `rsh` | Remote shell without encryption, replaced by SSH |
| `rsh-server` | Server component of rsh |
| `ypbind` | NIS client, outdated and insecure directory service |
| `ypserv` | NIS server, outdated and insecure directory service |
| `tftp` | Trivial FTP without authentication or encryption |
| `tftp-server` | Server component of tftp |


### Block 3 – Ensure SELinux is enforcing

Block 3 ensures SELinux is running in enforcing mode on the managed node:
- Checks the current SELinux status at runtime via `getenforce`
- Sets SELinux to `enforcing` persistently in `/etc/selinux/config`
- Sets SELinux to `enforcing` at runtime via `setenforce 1` if not already active

SELinux (Security-Enhanced Linux) is a Mandatory Access Control (MAC) system
built into the Linux kernel. In enforcing mode, it actively blocks any action
that violates the defined security policy – even for the root user.

> **Note:** RHEL 10.1 ships with SELinux in enforcing mode by default.
> This block ensures it has not been accidentally or intentionally disabled.


## Current status

The hardening playbook is **work in progress** and intentionally incomplete.

Planned future blocks include:

- Removal of insecure legacy packages
- SELinux enforcement
- SSH hardening
- Firewall configuration
- Audit logging
- Kernel and service hardening


## Known Issues

- RHEL 10 ships with GPG v6 which is incompatible with Ansible's `rpm_key` module. EPEL GPG key is therefore imported via `rpm --import` directly.


## Available Tags (in progress)

| Tag | Playbook | Description |
|---|---|---|
| - | `hello.yml` | Get uptime, install htop |
| `update` | `hardening.yml` | System updates & automatic patching |
| `packages` | `hardening.yml` | Remove insecure packages (telnet, rsh, ypbind, ypserv, tftp...) |
| `selinux` | `hardening.yml` | Enforce SELinux |
| `ssh` | `hardening.yml` | Harden SSH configuration (planned) |
| `firewall` | `hardening.yml` | Configure firewalld (planned) |
| `audit` | `hardening.yml` | Audit logging (planned) |

## Compliance (in progress)
This project aims to align with:
- CIS RHEL 10 Benchmark (where applicable)
- Real-world operational constraints (e.g. Kubernetes nodes, minimal installs)

Compliance mapping will be added incrementally.
