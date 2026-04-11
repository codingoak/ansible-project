# Ansible Learning Project - RHEL 10.1 Hardening

A structured learning environment for **Ansible** using a RHEL 10.1 control node.
The goal is to write test and evolve **CIS-oriented hardening playbooks** against managed RHEL 10.1 nodes.

## Project Structure

```sh
ansible-project/
├── ansible.cfg                  # Ansible configuration
├── .gitignore                   # Git ignore rules
├── inventory/
│   ├── hosts.ini                # Managed node inventory (ignored by git)
│   └── hosts.ini.example        # Inventory template
├── playbooks/
│   ├── hello.yml                # First simple playbook
│   └── hardening.yml            # CIS-oriented hardening playbook
└── roles/                       # Roles go here (future)
```

## Requirements

- RHEL 10.1 control node with Ansible installed
- RHEL 10.1 managed node reachable via SSH
- SSH key-based authentication configured for the `ansible` user
- EPEL repository (installed automatically by hello.yml including GPG key import)
- Ansible collections: `ansible.posix`

```sh
# Install required collections
ansible-galaxy collection install ansible.posix
```

## Setup

```sh
# Install Ansible
sudo dnf install -y ansible-core

# Clone this repo
git clone https://github.com/codingoak/ansible-project.git
cd ansible-project

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

# Dry-run hardening playbook (no changes)
ansible-playbook playbooks/hardening.yml --check --diff

# Run full hardening
ansible-playbook playbooks/hardening.yml

# Run specific section only
ansible-playbook playbooks/hardening.yml --tags ssh
```

## Available Tags

| Tag | Playbook | Description |
|---|---|---|
| - | `hello.yml` | Get uptime, install EPEL and htop |
| `update` | `hardening.yml` | System updates + dnf-automatic |
| `packages` | `hardening.yml` | Remove insecure packages (telnet, rsh, ypbind, ypserv, tftp) |
| `selinux` | `hardening.yml` | Enforce SELinux |
| `ssh` | `hardening.yml` | Harden SSH configuration |
| `firewall` | `hardening.yml` | Configure firewalld |
| `audit` | `hardening.yml` | Enable audit logging |
| `password` | `hardening.yml` | Password quality policy |
| `sysctl` | `hardening.yml` | Kernel hardening |
| `services` | `hardening.yml` | Disable unnecessary services |

---

## Playbook Details

### hello.yml
A simple introductory playbook to verify the Ansible setup:
- Retrieves and displays system uptime
- Imports the EPEL GPG key via `rpm --import`
- Installs the EPEL repository
- Installs `htop`

> **Note:** RHEL 10 ships with GPG v6 which is incompatible with Ansible's
> `rpm_key` module. The EPEL GPG key is therefore imported via `rpm --import` directly.

---

### hardening.yml

#### Block 1 – System updates
Block 1 establishes a secure and up-to-date system baseline:
- Updates all installed packages
- Installs `dnf-automatic`
- Enables **dnf-automatic-install.timer** on RHEL 10

RHEL 10 provides multiple dnf-automatic timers.
This project explicitly enables **dnf-automatic-install.timer**, which
downloads and installs updates automatically, independent of the settings
in `/etc/dnf/automatic.conf`.

> **Note:** Automatic installation of updates may occasionally require a reboot
> (e.g. kernel updates). In production environments consider using
> **dnf-automatic-download.timer** instead and scheduling reboots manually.

---

#### Block 2 – Remove insecure packages
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

> **Note:** If none of these packages are installed, all tasks will show `ok`
> instead of `changed` – this is expected and correct behavior (idempotent).

---

#### Block 3 – Ensure SELinux is enforcing
Block 3 ensures SELinux is running in enforcing mode on the managed node:
- Checks the current SELinux status at runtime via `getenforce`
- Sets SELinux to `enforcing` persistently in `/etc/selinux/config`
- Sets SELinux to `enforcing` at runtime via `setenforce 1` if not already active

SELinux (Security-Enhanced Linux) is a Mandatory Access Control (MAC) system
built into the Linux kernel. In enforcing mode, it actively blocks any action
that violates the defined security policy – even for the root user.

> **Note:** RHEL 10.1 ships with SELinux in enforcing mode by default.
> This block ensures it has not been accidentally or intentionally disabled.

---

#### Block 4 – Harden SSH
Block 4 hardens the SSH configuration on the managed node:

| Setting | Value | Reason |
|---|---|---|
| `PermitRootLogin` | `no` | Prevents direct root access via SSH |
| `PasswordAuthentication` | `no` | Forces SSH key-based authentication |
| `ClientAliveInterval` | `300` | Disconnects idle sessions after 5 minutes |
| `MaxAuthTries` | `3` | Limits brute-force attempts |
| `X11Forwarding` | `no` | Disables unnecessary graphical forwarding |
| `PermitEmptyPasswords` | `no` | Prevents login with empty passwords |

Additionally enforces secure cryptographic algorithms:

| Type | Allowed algorithms |
|---|---|
| Key exchange | `curve25519-sha256`, `diffie-hellman-group14-sha256` |
| Ciphers | `aes256-gcm@openssh.com`, `chacha20-poly1305@openssh.com` |
| MACs | `hmac-sha2-256`, `hmac-sha2-512` |

> **Note:** A handler restarts `sshd` automatically after any change to the SSH
> configuration, but only once at the end of the play – not after every single task.

---

#### Block 5 – Configure firewall
Block 5 configures firewalld on the managed node:
- Installs firewalld if not present
- Enables and starts the firewalld service
- Sets the default zone to `drop` – all incoming traffic is blocked by default
- Explicitly allows SSH so the managed node remains reachable

| Zone | Default policy | Allowed services |
|---|---|---|
| `drop` | Block all incoming traffic | `ssh` |

> **Note:** The `drop` zone silently discards all packets that do not match
> an explicit rule – no ICMP reject message is sent back to the sender.
> This makes the server invisible to port scanners.

---

#### Block 6 – Configure audit logging
Block 6 sets up auditd to monitor critical system activity:
- Installs and enables the `auditd` service
- Writes custom audit rules to `/etc/audit/rules.d/hardening.rules`

| Key `-k` | Monitored paths | Events |
|---|---|---|
| `identity` | `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` | Write, attribute change |
| `sudo` | `/etc/sudoers`, `/etc/sudoers.d/` | Write, attribute change |
| `logins` | `/var/log/lastlog`, `/var/run/faillock/` | Write, attribute change |
| `sshd` | `/etc/ssh/sshd_config` | Write, attribute change |
| `exec_root` | All commands executed as root via sudo | Always |
| `network` | `/etc/hosts`, `/etc/sysconfig/network` | Write, attribute change |

> **Note:** A handler reloads the audit rules via `augenrules --load`
> automatically after any change to the rules file.

---

#### Block 7 – Password policy, kernel hardening & services

**Password policy** – enforced via `pwquality` and `login.defs`:

| Setting | Value | Reason |
|---|---|---|
| `minlen` | `12` | Minimum password length |
| `dcredit` | `-1` | At least 1 digit required |
| `ucredit` | `-1` | At least 1 uppercase letter required |
| `lcredit` | `-1` | At least 1 lowercase letter required |
| `ocredit` | `-1` | At least 1 special character required |
| `maxrepeat` | `3` | Max 3 consecutive identical characters |
| `PASS_MAX_DAYS` | `90` | Password expires after 90 days |

**Kernel hardening** – enforced via `sysctl`:

| Parameter | Value | Reason |
|---|---|---|
| `net.ipv4.ip_forward` | `0` | Server is not a router |
| `net.ipv4.tcp_syncookies` | `1` | SYN flood protection |
| `net.ipv4.conf.all.accept_redirects` | `0` | Ignore ICMP redirects |
| `net.ipv6.conf.all.accept_redirects` | `0` | Ignore IPv6 ICMP redirects |
| `net.ipv4.conf.all.accept_source_route` | `0` | Disable source routing |
| `net.ipv4.conf.all.rp_filter` | `1` | Enable reverse path filter |
| `kernel.randomize_va_space` | `2` | Enable full ASLR |
| `fs.suid_dumpable` | `0` | Restrict core dumps |

**Disabled services:**

| Service | Reason |
|---|---|
| `bluetooth` | Not needed on a server |
| `avahi-daemon` | mDNS/DNS-SD not needed on a server |
| `cups` | Printing service not needed on a server |

> **Note:** Service disable tasks use `failed_when: false` – if a service
> is not installed the task is skipped silently without failing the playbook.

---

## Compliance Verification

After running the full hardening playbook, verify the results with OpenSCAP:

```sh
# Install OpenSCAP on the managed node
ansible managed -m ansible.builtin.dnf -a "name=openscap-scanner,scap-security-guide state=present" --become

# Run CIS benchmark scan
ansible managed -m ansible.builtin.command -a \
  "oscap xccdf eval \
   --profile xccdf_org.ssgproject.content_profile_cis \
   --report /tmp/scap-report.html \
   /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml" -- become

# Fetch report to control node
ansible managed -m ansible.builtin.fetch -a \
  "src=/tmp/scap-report.html dest=./scap-reports/ flat=no"
```

The HTML report shows which CIS controls pass or fail and provides
remediation guidance for any findings.


## Known Issues

- RHEL 10 ships with GPG v6 which is incompatible with Ansible's `rpm_key` module.
  EPEL GPG key is therefore imported via `rpm --import` directly.
