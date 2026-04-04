# Ansible Learning Project - RHEL 10.1 Hardening (in progress)

A structured learning environment for Ansibel using a RHEL 10.1 controle node.
The goal is to write and test playbooks agains a managed node.


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

- RHEL 10.1 control node with Ansible installed
- RHEL 10.1 managed node reachable via SSH
- SSH key-based authentication configured for the `ansible` user
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


## Usage (in progress)

```sh
# Test connectivity
ansible managed -m ping

# Run first playbook
ansible-playbook playbooks/hello.yml
```

## Known Issues

- RHEL 10 ships with GPG v6 which is incompatible with Ansible's `rpm_key` module. EPEL GPG key is therefore imported via `rpm --import` directly.


## Available Tags (in progress)

| Tag | Playbook | Description |
|---|---|---|
| - | `hello.yml` | Get uptime, install htop |


## Compliance (in progress)

