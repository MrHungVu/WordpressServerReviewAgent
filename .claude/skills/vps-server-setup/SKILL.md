---
name: vps-server-setup
description: SSH into a fresh Linux VPS and install Nginx + PHP-FPM + MariaDB (LEMP stack), configure firewall (UFW/firewalld), SSH hardening, and fail2ban. Every step is idempotent — safe to run on already-partial setups. Use when user wants to set up a fresh VPS for WordPress hosting.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---

# VPS Server Setup Skill

## Purpose
Set up a fresh Linux VPS with LEMP stack (Nginx + PHP-FPM + MariaDB) for WordPress hosting.

## Required Input (from user/orchestrator, NEVER save to files)
- SSH host (IP address)
- SSH user (typically root)
- SSH password
- SSH port (default 22)
- Domain name (for context/report)

## SSH Connection
Use `sshpass` for password-based SSH. All commands run via:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

If `sshpass` not installed locally:
- macOS: `brew install hudochenkov/sshpass/sshpass`
- Linux: `apt install sshpass -y`

## Setup Checklist

### 1. Detect OS
```bash
cat /etc/os-release
```
- Determine distro family: Debian/Ubuntu → use `apt`; CentOS/RHEL/AlmaLinux → use `yum`/`dnf`
- Store pkg manager for all subsequent steps

### 2. System Update
Wait for apt lock if needed, then update:
```bash
while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 2; done
apt update && apt upgrade -y
# or for CentOS:
yum update -y
```

### 3. Install Nginx
Check before act:
```bash
command -v nginx &>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```
Install if missing:
```bash
# Debian/Ubuntu
apt install nginx -y
# CentOS
yum install nginx -y
```
Enable, start, and remove Apache if running:
```bash
systemctl enable nginx && systemctl start nginx
systemctl stop apache2 2>/dev/null; systemctl disable apache2 2>/dev/null
systemctl stop httpd 2>/dev/null; systemctl disable httpd 2>/dev/null
```

### 4. Install PHP-FPM + Extensions
Check before act:
```bash
command -v php &>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```
Install:
```bash
# Debian/Ubuntu
apt install php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-zip php-imagick php-intl php-bcmath -y
# CentOS
yum install php-fpm php-mysqlnd php-curl php-gd php-mbstring php-xml php-zip php-imagick php-intl php-bcmath -y
```
Enable and start:
```bash
systemctl enable php*-fpm && systemctl start php*-fpm
```
Detect PHP-FPM socket path (pass to orchestrator for Nginx config):
```bash
find /run/php -name "*.sock" 2>/dev/null || find /var/run/php -name "*.sock" 2>/dev/null
```

### 5. Install MariaDB
Check before act:
```bash
command -v mysql &>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```
Install:
```bash
# Debian/Ubuntu
apt install mariadb-server -y
# CentOS
yum install mariadb-server -y
```
Enable and start:
```bash
systemctl enable mariadb && systemctl start mariadb
```

### 6. Secure MariaDB
Auto-generate a root password:
```bash
openssl rand -base64 24
```
Apply non-interactive secure installation using the generated password:
```bash
mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'GENERATED_PASSWORD';"
mysql -u root -p'GENERATED_PASSWORD' -e "DELETE FROM mysql.user WHERE User='';"
mysql -u root -p'GENERATED_PASSWORD' -e "DROP DATABASE IF EXISTS test;"
mysql -u root -p'GENERATED_PASSWORD' -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%';"
mysql -u root -p'GENERATED_PASSWORD' -e "FLUSH PRIVILEGES;"
```
- **IMPORTANT:** Output the generated password to conversation — orchestrator needs it for WordPress DB setup
- If MariaDB already had a password set (ALTER USER fails), report "existing installation detected, root password unknown" and continue

### 7. Configure Firewall (UFW / firewalld)
**Debian/Ubuntu — UFW:**
```bash
command -v ufw &>/dev/null || apt install ufw -y
# Allow the current SSH port (may not be 22)
SSH_PORT=$(grep -E "^Port " /etc/ssh/sshd_config | awk '{print $2}')
SSH_PORT=${SSH_PORT:-22}
ufw allow ${SSH_PORT}/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```
**CentOS — firewalld:**
```bash
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

### 8. SSH Hardening
Read current settings first:
```bash
grep -E "^(PermitRootLogin|MaxAuthTries)" /etc/ssh/sshd_config
```
Apply safe hardening (MaxAuthTries only — do NOT touch PermitRootLogin):
```bash
sed -i 's/^MaxAuthTries.*/MaxAuthTries 5/' /etc/ssh/sshd_config
grep -q "^MaxAuthTries" /etc/ssh/sshd_config || echo "MaxAuthTries 5" >> /etc/ssh/sshd_config
systemctl reload sshd
```
- **DO NOT** change `PermitRootLogin` — fresh VPS uses password auth only; changing it locks out the user
- **DO NOT** disable `PasswordAuthentication` — user may not have key-based auth configured
- SSH key setup + PermitRootLogin hardening should be done manually after setup, once key auth is confirmed working

### 9. Install fail2ban
Check before act:
```bash
command -v fail2ban-client &>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```
Install and enable:
```bash
apt install fail2ban -y  # or yum install fail2ban -y
systemctl enable fail2ban && systemctl start fail2ban
```

### 10. Verify All Services
```bash
systemctl is-active nginx
systemctl is-active php*-fpm
systemctl is-active mariadb
systemctl is-active ufw 2>/dev/null || systemctl is-active firewalld 2>/dev/null
nginx -v 2>&1
php -v | head -1
mysql --version
hostname -I | awk '{print $1}'
```

## Output
Return this summary to the orchestrator:

```
## Setup Results
- **Server IP:** X.X.X.X
- **OS:** Ubuntu 24.04 / etc.
- **Nginx:** version
- **PHP-FPM:** version + socket path
- **MariaDB:** version
- **MariaDB Root Password:** [generated password — keep secure]
- **Firewall:** UFW/firewalld active (22, 80, 443 open)
- **SSH Hardening:** MaxAuthTries=5 (PermitRootLogin unchanged — harden manually after key auth setup)
- **fail2ban:** Active
- **PHP-FPM Socket:** /run/php/phpX.X-fpm.sock
```

## Error Handling
- If any package install fails, log the error and continue with remaining steps — never abort the entire setup
- If MariaDB already has a root password set, report it and skip the secure step
- If Apache is detected running, stop+disable it and warn user in output
- If a service fails to start, report status output and continue

## Important Rules
- NEVER save SSH credentials to any file
- Run ALL commands via SSH remotely, not locally
- Every step must check-before-act (idempotent — safe on partial setups)
- Output the MariaDB root password to the conversation — orchestrator needs it
- Wait for apt lock before running apt commands
