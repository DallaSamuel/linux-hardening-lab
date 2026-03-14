# Linux Hardening Lab — CyberJKD

**Target System:** Kali Linux (WSL2 on Windows)  
**Date:** March 14, 2026  
**Author:** Samuel Dalla (CyberJKD)  
**Phase:** Roadmap Phase 1 — Systems Foundation

---

## Overview

This document details the systematic hardening of a Kali Linux installation running on WSL2, completed as Phase 1 of the CyberJKD cybersecurity learning roadmap. The objective was to reduce the attack surface, disable unnecessary services, configure firewall rules, harden SSH access, and secure sensitive file permissions. All changes are documented as a reproducible checklist that can be applied to any Debian-based Linux system.

The approach follows a simple principle — audit first, change second, verify third. No change was made without understanding why it matters.

---

## Baseline Audit

Before any changes were made a full audit was conducted to understand the current state of the system.

**Open ports before hardening:**

| Port | Service | Risk |
|------|---------|------|
| 53 | DNS (WSL2 internal) | Low — internal only |
| 323 | chronyd time sync | Low — localhost only |
| 3350 | xrdp session manager | Medium — unnecessary service |
| 3389 | RDP daemon | High — exposed on all interfaces |

**Running services before hardening:**

| Service | Status |
|---------|--------|
| xrdp | Running — unnecessary |
| xrdp-sesman | Running — unnecessary |
| lightdm | Running — unnecessary on WSL2 |
| rtkit-daemon | Running — unnecessary |
| cron | Running — required |
| dbus | Running — required |
| polkit | Running — required |
| unattended-upgrades | Running — beneficial, kept |
| core systemd services | Running — required |

**Key finding:** Port 3389 RDP was exposed on all interfaces with no restriction. RDP is one of the most commonly targeted services in real-world attacks — used for brute force, credential stuffing, and exploitation of known vulnerabilities such as BlueKeep. Its presence on a default installation with no firewall represents a significant unnecessary risk.

![Baseline Ports](01_baseline_ports.png)

---

## Hardening Actions

### 1. Disabled Unnecessary Services

Four services were identified as unnecessary and disabled:
```bash
sudo systemctl stop xrdp
sudo systemctl disable xrdp
sudo systemctl stop xrdp-sesman
sudo systemctl disable xrdp-sesman
sudo systemctl stop lightdm
sudo systemctl disable lightdm
sudo systemctl stop rtkit-daemon
sudo systemctl disable rtkit-daemon
```

| Service | Reason for Disabling |
|---------|---------------------|
| xrdp | RDP not required — was exposing port 3389 on all interfaces |
| xrdp-sesman | Dependency of xrdp — no longer needed |
| lightdm | Display manager unnecessary on a headless WSL2 environment |
| rtkit-daemon | Realtime scheduling not required for this system |

**Note:** unattended-upgrades was left running. Automatic security updates are a defensive control — disabling them would reduce security not improve it.

![Services After Hardening](02_services_after_hardening.png)


---

### 2. SSH Hardening

Configuration file modified: `/etc/ssh/sshd_config`

Four settings were hardened:

```
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
```

| Setting | Value Set | Reason |
|---------|-----------|--------|
| PermitRootLogin | no | Prevents direct root login over SSH |
| PasswordAuthentication | no | Forces key-based authentication only — eliminates brute force via password |
| X11Forwarding | no | Disables GUI forwarding over SSH — unnecessary and increases attack surface |
| MaxAuthTries | 3 | Limits failed authentication attempts — slows brute force attempts |

SSH service restarted to apply changes:
```bash
sudo service ssh restart
```

---

### 3. Firewall Configuration

Tool used: ufw (Uncomplicated Firewall)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

| Rule | Value | Reason |
|------|-------|--------|
| Default incoming | Deny | Block all unsolicited inbound connections |
| Default outgoing | Allow | System can initiate outbound connections normally |
| Port 22 SSH | Allow | Explicitly permit SSH — only intentional open port |

![UFW Status](03_ufw_status.png)

---

### 4. File Permission Hardening
```bash
sudo chmod 600 /etc/shadow
sudo chmod 644 /etc/passwd
sudo chmod 640 /etc/group
sudo chmod 600 /etc/gshadow
```

| File | Permissions | Reason |
|------|-------------|--------|
| /etc/shadow | 600 | Contains password hashes — root read/write only |
| /etc/passwd | 644 | User account info — readable by all, writable by root only |
| /etc/group | 640 | Group membership info — restricted read |
| /etc/gshadow | 600 | Group password hashes — root only |

![File Permissions](04_file_permissions.png)


---

### 5. User Security Checks

**Check 1 — Empty passwords:**
```bash
sudo awk -F: '($2 == "") {print $1}' /etc/shadow
```
Result: No output — no accounts with empty passwords found.

**Check 2 — Unauthorized UID 0 accounts:**
```bash
awk -F: '($3 == "0") {print $1}' /etc/passwd
```
Result: Only `root` holds UID 0 — no unauthorized privilege escalation risk found.

**Check 3 — World-writable files in /etc:**
```bash
sudo find /etc -type f -perm -o+w 2>/dev/null
```
Result: No output — no world-writable files found in /etc.

---

## Post-Hardening Audit

| Port | Service | Status |
|------|---------|--------|
| 53 | DNS (WSL2 internal) | Unchanged — expected |
| 323 | chronyd | Unchanged — localhost only, low risk |
| 22 | SSH | Intentional — locked down with key-only auth |
| 3350 | xrdp-sesman | Removed from attack surface |
| 3389 | RDP | Removed from attack surface |

**Ports 3389 and 3350 are completely gone.**


## Hardening Checklist

- [ ✓ ] Audited open ports before making any changes
- [ ✓ ] Audited running services before making any changes
- [ ✓ ] Disabled xrdp service
- [ ✓ ] Disabled xrdp-sesman service
- [ ✓ ] Disabled lightdm display manager
- [ ✓ ] Disabled rtkit-daemon
- [ ✓ ] Set PermitRootLogin no in sshd_config
- [ ✓ ] Set PasswordAuthentication no in sshd_config
- [ ✓ ] Set X11Forwarding no in sshd_config
- [ ✓ ] Set MaxAuthTries 3 in sshd_config
- [ ✓ ] Restarted SSH service to apply changes
- [ ✓ ] Installed and configured ufw firewall
- [ ✓ ] Set default deny incoming policy
- [ ✓ ] Set default allow outgoing policy
- [ ✓ ] Allowed SSH through firewall explicitly
- [ ✓ ] Enabled ufw
- [ ✓ ] Set /etc/shadow permissions to 600
- [ ✓ ] Set /etc/passwd permissions to 644
- [ ✓ ] Set /etc/group permissions to 640
- [ ✓ ] Set /etc/gshadow permissions to 600
- [ ✓ ] Verified no empty password accounts exist
- [ ✓ ] Verified no unauthorized UID 0 accounts exist
- [ ✓ ] Verified no world-writable files in /etc
- [ ✓ ] Confirmed post-hardening port audit matches expected state


## Key Takeaways

A default Kali Linux installation is not production-hardened. RDP was exposed on all interfaces, unnecessary services were running, and SSH was configured with permissive defaults. Through systematic auditing and hardening the attack surface was significantly reduced.

Three principles drove every decision in this lab:

**Least privilege** — every service, user, and file permission was reduced to the minimum necessary to function.

**Attack surface reduction** — if a service is not needed it should not be running. Every open port is a potential entry point.

**Audit before action** — understanding the baseline state before making changes is what separates disciplined security work from guesswork.

This process mirrors real-world Linux server hardening procedures used by security engineers and system administrators in enterprise environments. The same checklist can be adapted and applied to Ubuntu, Debian, or any other Debian-based production server.

---

*[github.com/DallaSamuel](https://github.com/DallaSamuel)*

*CyberJKD — Becoming dangerous through fundamentals.*
