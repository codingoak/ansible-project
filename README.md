# Ansible Learning Project - RHEL 10.1 Hardening (in progress)

A structured learning environment for Ansibel using a RHEL 10.1 controle node.
The goal is to write and test playbooks agains a managed node.

## Prject Strucutre

```sh
ansile-lernprojekt/
  ansible.dfg      # Ansible configuration
  .gitignore       # Git ignore rules
  inventory/
    hosts.ini      # Managed node inventory
  playbooks/
    hello.yml      # First simple playbook
    hardening.yml  # CIS-oriented hardening playbook (future) 
  roles/           # Roles go here (future)
```

## Requirements

- RHEL 10.1 control node with Ansible installed
- RHEL 10.1 managed node reachable via SSH
- SSH key-based authentication configured for the `ansible` user

## Setup

```sh

# Install Ansible
sudo dnf install -y ansible-core

# Clone this repo
git clone https://github.com/codingoak/ansible-project.git
cd ~/ansible-project

# Update the managed node IP in inventory
vim inventory/hosts.ini

## Usage (in progress)

## Available Tags (in progress)

## Compliance (in progress)
```

