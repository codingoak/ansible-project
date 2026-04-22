# RHEL 10.1 CIS L2 Hardening with Ansible

> **Goal:** Harden a RHEL 10.1 server to CIS Benchmark Level 2 – Server standard using a single Ansible playbook, managed via SSH keys from a dedicated control node.

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
│   └── hardening.yml            # CIS L2 hardening playbook
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

## Getting Started (Quick)

Only one value **must** be changed before running the playbook:

1. Set `control_node_ip` in `playbooks/hardening.yml` to your control node's IP
2. Set your managed node's IP in `inventory/hosts.ini`
3. Run `ansible-playbook playbooks/hardening.yml --check --diff`

Everything else has sensible defaults. See [Variables Reference](#variables-reference).

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

# Run in stages (recommended for first application)
ansible-playbook playbooks/hardening.yml --tags ssh
ansible-playbook playbooks/hardening.yml --tags firewall
ansible-playbook playbooks/hardening.yml --skip-tags ssh,firewall
```

## Available Tags

| Tag | Playbook | Description |
|---|---|---|
| - | `hello.yml` | Get uptime, install EPEL and htop |
| `update` | `hardening.yml` | System updates + dnf-automatic + disable weak deps |
| `packages` | `hardening.yml` | Remove insecure packages |
| `selinux` | `hardening.yml` | Enforce SELinux |
| `crypto` | `hardening.yml` | CIS crypto policy module |
| `banner` | `hardening.yml` | Login banners (/etc/issue, /etc/issue.net, MOTD) |
| `ssh` | `hardening.yml` | Complete SSH hardening (15+ directives) |
| `sudo` | `hardening.yml` | Sudo use_pty and logfile |
| `pam` | `hardening.yml` | PAM faillock, password history, pam_pwquality |
| `password` | `hardening.yml` | Password quality + expiry + apply to existing users |
| `firewall` | `hardening.yml` | Configure firewalld (default: drop) |
| `audit` | `hardening.yml` | Enable audit logging |
| `sysctl` | `hardening.yml` | Kernel hardening |
| `services` | `hardening.yml` | Disable unnecessary services |

---

## Variables Reference

All playbook variables are defined in the `vars:` section, split into two groups:

**User Configuration** — adjust to your environment:

```yaml
control_node_ip: "192.168.x.x"       # CHANGE: only host allowed to SSH in
kernel_ip_forward: "0"                # enable only if host routes traffic
kernel_ip_forward_v6: "0"
```

**CIS Defaults** — usually no changes needed:

```yaml
# SSH
ssh_idle_timeout: 300                 ssh_max_auth_tries: 3
ssh_alive_count_max: 3                ssh_max_sessions: 10
ssh_login_grace_time: 60              ssh_max_startups: "10:30:60"

# Password policies
password_min_length: 14               password_remember: 24
password_max_days: 365                faillock_attempts: 5
password_min_days: 1                  faillock_unlock_time: 900

# Session
session_timeout: 900                  default_umask: "027"

# Infrastructure
audit_buffer_size: 8192               audit_backlog_limit: 8192
aide_cron_minute: "5"                 aide_cron_hour: "4"
shm_size: "512m"
```

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

#### Block 01 – System updates
Establishes a secure and up-to-date system baseline:
- Updates all installed packages
- Installs `dnf-automatic`
- Enables **dnf-automatic-install.timer** on RHEL 10
- Disables installation of weak dependencies in DNF

> **Note:** Automatic installation of updates may occasionally require a reboot
> (e.g. kernel updates). In production environments consider using
> **dnf-automatic-download.timer** instead and scheduling reboots manually.

---

#### Block 02 – Remove insecure packages
Removes legacy and insecure packages that are not needed on a modern RHEL 10 server:

| Package | Reason |
|---|---|
| `telnet` | Transmits data including passwords in plaintext |
| `rsh`, `rsh-server` | Remote shell without encryption, replaced by SSH |
| `ypbind`, `ypserv` | NIS client/server, outdated and insecure |
| `tftp`, `tftp-server` | Trivial FTP without authentication or encryption |
| `vsftpd` | FTP server, replaced by SFTP/SCP |
| `httpd` | Web server, not needed for base hardening |
| `dovecot` | IMAP/POP3 server |
| `squid` | Proxy server |
| `net-snmp` | SNMP agent, potential info leak |
| `samba` | SMB file sharing |

---

#### Block 03 – Ensure SELinux is enforcing
Ensures SELinux is running in enforcing mode:
- Checks the current SELinux status at runtime via `getenforce`
- Sets SELinux to `enforcing` persistently in `/etc/selinux/config`
- Sets SELinux to `enforcing` at runtime via `setenforce 1` if not already active

> **Note:** RHEL 10.1 ships with SELinux in enforcing mode by default.
> This block ensures it has not been accidentally or intentionally disabled.

---

#### Block 04 – Crypto policy
Implements a custom CIS-compliant crypto policy module:
- Creates `/etc/crypto-policies/policies/modules/CIS.pmod`
- Enforces minimum 2048-bit DH and RSA key sizes
- Disables SHA1 in certificates
- Disables SSH certificates and Encrypt-then-MAC
- Applies via `update-crypto-policies --set DEFAULT:CIS`

The system-wide crypto policy affects all applications that use OpenSSL, GnuTLS, NSS, or libssh — including SSH, TLS, and DNSSEC.

---

#### Block 05 – Login banners
Sets warning banners displayed before and after login:

| File | Purpose |
|---|---|
| `/etc/issue` | Banner shown on local terminal before login |
| `/etc/issue.net` | Banner shown for remote (SSH) connections |
| `/etc/motd` | Cleared — prevents OS version info leak after login |

All files are set to `root:root` ownership with mode `0644`. The banner text is defined in the `login_banner` variable.

---

#### Block 06 – Complete SSH hardening
Deploys a single drop-in configuration file at `/etc/ssh/sshd_config.d/99-hardening.conf` containing all CIS-required SSH directives. This replaces the previous approach of modifying `sshd_config` line by line, which is cleaner and avoids conflicts with future OS updates.

**Access control:**

| Setting | Value | Reason |
|---|---|---|
| `PermitRootLogin` | `no` | Prevents direct root access via SSH |
| `PasswordAuthentication` | `no` | Forces SSH key-based authentication |
| `PermitEmptyPasswords` | `no` | Prevents login with empty passwords |
| `PubkeyAuthentication` | `yes` | Explicitly enables key-based auth |

**Timeouts and limits:**

| Setting | Value | Reason |
|---|---|---|
| `ClientAliveInterval` | `300` | Idle timeout in seconds |
| `ClientAliveCountMax` | `3` | Missed keepalives before disconnect |
| `LoginGraceTime` | `60` | Max seconds to authenticate |
| `MaxAuthTries` | `3` | Limits brute-force attempts |
| `MaxSessions` | `10` | Max concurrent sessions per connection |
| `MaxStartups` | `10:30:60` | Rate-limit unauthenticated connections |

**Security options:**

| Setting | Value | Reason |
|---|---|---|
| `HostbasedAuthentication` | `no` | Prevents trust based on hostname |
| `DisableForwarding` | `yes` | Blocks all forwarding (port, X11, agent, tunnel) |
| `GSSAPIAuthentication` | `no` | Disables Kerberos (not needed) |
| `IgnoreRhosts` | `yes` | Ignores legacy .rhosts files |
| `PermitUserEnvironment` | `no` | Prevents users from setting env vars via SSH |
| `X11Forwarding` | `no` | No GUI forwarding needed on a server |
| `UsePAM` | `yes` | Enables PAM for account/session management |

**Logging and banner:**

| Setting | Value | Reason |
|---|---|---|
| `LogLevel` | `VERBOSE` | Detailed logging for audit trail |
| `Banner` | `/etc/issue.net` | Warning banner before authentication |

> **Note:** The drop-in file at `sshd_config.d/99-hardening.conf` overrides
> defaults from the main `sshd_config`. A handler restarts `sshd` only once
> at the end of the play. **Always keep a second SSH session open** when
> applying SSH changes to avoid locking yourself out.

---

#### Block 07 – Sudo hardening
Hardens sudo configuration via `/etc/sudoers.d/99-hardening`:

| Setting | Effect |
|---|---|
| `Defaults use_pty` | Forces sudo to allocate a pseudo-terminal, preventing background processes from persisting after the session ends |
| `Defaults logfile="/var/log/sudo.log"` | Logs all sudo commands to a dedicated file for auditing |

Both settings are validated with `visudo -cf` before being applied to prevent lockouts from syntax errors.

---

#### Block 08 – PAM faillock and password history
Configures PAM (Pluggable Authentication Modules) for account lockout and password reuse prevention:

**authselect profile:**
Selects the `sssd` profile with `faillock` and `pamaccess` features enabled. This configures the PAM stack automatically with the correct module order.

**Faillock configuration** (`/etc/security/faillock.conf`):

| Setting | Value | Effect |
|---|---|---|
| `deny` | `5` | Lock account after 5 failed attempts |
| `unlock_time` | `900` | Auto-unlock after 15 minutes |
| `even_deny_root` | enabled | Root account is also locked |
| `root_unlock_time` | `900` | Root auto-unlock after 15 minutes |

**Password history:**
- `pam_pwquality.so` enforced in both `system-auth` and `password-auth` with `enforce_for_root`
- `pam_unix.so` configured with `remember=24` to reject the last 24 passwords
- `pam_pwhistory.so` enabled for root with `enforce_for_root`

> **Note:** The `authselect select` command resets the PAM stack on each run.
> Subsequent tasks re-apply the custom PAM modifications. This causes some
> tasks to show `changed` on every playbook run — this is expected behavior.

---

#### Block 09 – Password quality requirements
Expanded password policy enforced via `pwquality.conf` and `login.defs`:

| Setting | Value | Reason |
|---|---|---|
| `minlen` | `14` | Minimum password length |
| `dcredit` | `-1` | At least 1 digit required |
| `ucredit` | `-1` | At least 1 uppercase letter required |
| `lcredit` | `-1` | At least 1 lowercase letter required |
| `ocredit` | `-1` | At least 1 special character required |
| `maxrepeat` | `3` | Max 3 consecutive identical characters |
| `maxsequence` | `3` | Max 3 sequential characters (e.g. "abc") |
| `difok` | `2` | Min 2 characters must differ from old password |
| `dictcheck` | `1` | Reject dictionary words |
| `minclass` | `4` | All 4 character classes required |
| `enforce_for_root` | enabled | Root must also comply |

**Login.defs settings:**

| Setting | Value | Effect |
|---|---|---|
| `PASS_MAX_DAYS` | `365` | Password expires after 365 days |
| `PASS_MIN_DAYS` | `1` | Minimum 1 day between changes |
| `PASS_WARN_AGE` | `7` | Warn 7 days before expiry |

**Applied to existing users** via `chage` for PASS_MIN_DAYS, PASS_MAX_DAYS, and inactivity lock. New accounts inherit defaults via `useradd -D -f 30`.

---

#### Block 10 – Configure firewall
Configures firewalld with a strict inbound policy:
- Installs firewalld if not present
- Enables and starts the firewalld service
- Sets the default zone to `drop`
- Allows SSH **only** from the control node IP (rich rule)
- Trusts loopback traffic for internal services

| Zone | Default policy | Allowed traffic |
|---|---|---|
| `drop` | Block all incoming traffic | SSH from `control_node_ip` only |
| `trusted` | Accept all traffic | Loopback interface (`lo`) |

The `control_node_ip` variable must be set to the IP address of the machine running Ansible. No other host can reach the server via SSH.

> **WARNING:** If `control_node_ip` is set incorrectly, you will lose SSH
> access after applying the firewall rules. Always keep a second SSH session
> open and test access before closing it. To allow additional services
> (e.g. web, game servers), add firewalld rules in a separate playbook or role.

---

#### Block 11 – Configure audit logging
Sets up auditd to monitor critical system activity:
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

---

#### Block 12 – Kernel hardening & services

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
ansible managed -b -m ansible.builtin.dnf \
  -a "name=openscap-scanner,scap-security-guide state=present"

# Run CIS benchmark scan
ansible managed -b -m command -a \
  "oscap xccdf eval \
   --profile xccdf_org.ssgproject.content_profile_cis \
   --report /tmp/scap-report.html \
   /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml"

# Fetch report to control node
ansible managed -m ansible.builtin.fetch \
  -a "src=/tmp/scap-report.html dest=./scap-reports/ flat=no"
```

> **Note:** Do not commit OpenSCAP reports to the repository — they contain
> real IPs, MAC addresses, and hostnames. The `scap-reports/` directory is in `.gitignore`.

---

## Known Issues

- RHEL 10 ships with GPG v6 which is incompatible with Ansible's `rpm_key` module.
  EPEL GPG key is therefore imported via `rpm --import` directly.

---

## Resources

| Topic | Link |
|-------|------|
| Ansible Documentation | [docs.ansible.com](https://docs.ansible.com) |
| CIS Benchmarks | [cisecurity.org/benchmarks](https://www.cisecurity.org/benchmarks) |
| RHEL Security Guide | [access.redhat.com/documentation](https://access.redhat.com/documentation) |
| OpenSCAP | [open-scap.org](https://www.open-scap.org) |
