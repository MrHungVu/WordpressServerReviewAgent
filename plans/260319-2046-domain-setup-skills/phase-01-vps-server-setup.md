# Phase 1: VPS Server Setup Skill

## Context
- Parent plan: [plan.md](./plan.md)
- Reference pattern: `.claude/skills/vps-server-audit/SKILL.md`
- Brainstorm: `plans/reports/brainstorm-260319-2046-domain-setup-agent.md`

## Overview
- **Priority:** P1 (must complete first — other skills depend on server being ready)
- **Status:** pending
- **Description:** Create `vps-server-setup` skill that SSHes into a fresh Linux VPS and installs the LEMP stack (Nginx + PHP-FPM + MariaDB), configures firewall, and applies basic SSH hardening.

## Key Insights
- Must detect OS (Ubuntu/Debian vs CentOS/RHEL) and adapt package manager commands
- MariaDB may already be installed — detect first, install only if missing
- UFW may not be installed on CentOS — use firewalld instead
- PHP version should be latest available in default repos (8.x)
- Every operation must be idempotent (check-before-act)
- SSH hardening must NOT lock user out (warn, don't force key-only auth)

## Requirements

### Functional
- Detect OS type and version
- Update system packages
- Install Nginx (skip if already installed)
- Install PHP-FPM + required extensions (mysql, curl, gd, mbstring, xml, zip, imagick)
- Detect/install MariaDB, secure installation, auto-generate root password
- Configure UFW firewall (allow 22, 80, 443)
- Apply SSH hardening (PermitRootLogin prohibit-password, MaxAuthTries 5)
- Verify all services running (nginx, php-fpm, mariadb)
- Report installed versions and server IP

### Non-Functional
- Idempotent: safe to re-run without breaking anything
- Timeout handling for slow package installs
- Error reporting: if a step fails, report and continue where safe

## Architecture

### SKILL.md Structure
```
---
name: vps-server-setup
description: SSH into VPS, install Nginx + PHP-FPM + MariaDB, configure firewall and SSH hardening. Use when user wants to set up a fresh VPS for WordPress hosting.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---
```

### Command Execution Pattern
All commands via SSH:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

### Idempotency Pattern
```bash
# Example: Nginx install
command -v nginx &>/dev/null && echo "Nginx already installed" || apt install nginx -y
```

### Setup Sequence
1. Detect OS: `cat /etc/os-release`
2. System update: `apt update && apt upgrade -y` (or `yum update -y`)
3. Install Nginx: `apt install nginx -y` (skip if exists)
4. Install PHP-FPM + extensions (skip if exists)
5. Detect MariaDB: `command -v mysql` → install if missing
6. Secure MariaDB: set root password, remove anon users, remove test DB
7. UFW: `apt install ufw -y && ufw allow 22 && ufw allow 80 && ufw allow 443 && ufw --force enable`
8. SSH hardening: modify `/etc/ssh/sshd_config`, reload sshd
9. Verify services: `systemctl status nginx php*-fpm mariadb`
10. Report: PHP version, MariaDB version, Nginx version, server IP

## Related Code Files
- **Create:** `.claude/skills/vps-server-setup/SKILL.md`
- **Reference:** `.claude/skills/vps-server-audit/SKILL.md` (same SSH patterns)

## Implementation Steps

1. Create directory `.claude/skills/vps-server-setup/`
2. Write `SKILL.md` with frontmatter (name, description, allowed-tools)
3. Write Purpose section — one-sentence objective
4. Write Required Input section — SSH host, user, password, port, domain
5. Write SSH Connection section — sshpass pattern (same as audit skill)
6. Write Setup Checklist with 10 sections:
   - Each section: detect → check if needed → install/configure → verify
   - Include both apt (Debian/Ubuntu) and yum (CentOS/RHEL) commands
   - Include idempotency checks before every action
7. Write MariaDB section:
   - Auto-generate root password: `openssl rand -base64 24`
   - Secure installation commands (non-interactive)
   - Output generated password for orchestrator to capture
8. Write Output section — what this skill returns to the orchestrator
9. Write Error Handling section — what to do when commands fail
10. Write Important Rules section — credential handling, never save passwords

## Todo List
- [ ] Create `.claude/skills/vps-server-setup/` directory
- [ ] Write SKILL.md with frontmatter matching existing skill pattern
- [ ] OS detection logic (Ubuntu/Debian vs CentOS/RHEL)
- [ ] Nginx install with idempotency check
- [ ] PHP-FPM + extensions install with idempotency check
- [ ] MariaDB detect/install with auto-generated root password
- [ ] MariaDB secure installation (non-interactive)
- [ ] UFW/firewalld configuration
- [ ] SSH hardening (without locking user out)
- [ ] Service verification checks
- [ ] Output format for orchestrator consumption

## Success Criteria
- Skill can take a fresh Ubuntu 22.04/24.04 VPS → fully configured LEMP server
- All services (Nginx, PHP-FPM, MariaDB) running after setup
- Firewall allows only ports 22, 80, 443
- Re-running the skill doesn't break anything (idempotent)
- MariaDB root password auto-generated and returned to caller

## Risk Assessment
| Risk | Impact | Mitigation |
|------|--------|------------|
| Existing Nginx/Apache conflicts | HIGH | Detect existing web server, warn user |
| Package manager lock (apt lock) | MEDIUM | Wait and retry, or kill stale lock |
| CentOS/RHEL: different package names | MEDIUM | OS detection with separate command paths |
| SSH hardening locks user out | HIGH | Only set `prohibit-password`, don't disable password auth entirely |
| MariaDB already installed, unknown root pass | MEDIUM | Try passwordless, then ask user |

## Security Considerations
- Auto-generated MariaDB root password must be cryptographically strong
- Never save credentials to files on server or locally
- SSH hardening should improve security without breaking current access
- UFW rules must not block current SSH session

## Next Steps
→ Phase 2: `wordpress-site-setup` (depends on LEMP stack being ready)
