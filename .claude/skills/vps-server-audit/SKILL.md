---
name: vps-server-audit
description: SSH into a VPS server running WordPress, audit OS, PHP, MySQL, web server, SSL, firewall, security hardening, disk/RAM, file permissions, and WordPress-specific configs. Produces a detailed report with status levels (OK/WARN/CRITICAL) and fix commands. Use when user wants to check/verify/audit a VPS server configuration.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---

# VPS Server Audit Skill

## Purpose
Audit a VPS server running WordPress via SSH. Check all server-level configurations and produce a comprehensive report.

## Required Input (from user, NEVER save to files)
- SSH host (IP or domain)
- SSH user (typically root)
- SSH password
- SSH port (default 22)
- Domain name (for SSL checks)

## SSH Connection
Use `sshpass` for password-based SSH. All commands run via:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

If `sshpass` not installed, install it first: `sudo apt-get install -y sshpass`

## Audit Checklist

### 1. System Info
- OS version: `cat /etc/os-release`
- Kernel: `uname -r`
- Uptime: `uptime`
- CPU: `nproc && cat /proc/cpuinfo | grep "model name" | head -1`
- RAM: `free -h`
- Disk: `df -h`
- Swap: `swapon --show`

### 2. Security Updates
- Pending updates: `apt list --upgradable 2>/dev/null | head -20` or `yum check-update | head -20`
- Auto updates config: `cat /etc/apt/apt.conf.d/20auto-upgrades 2>/dev/null`
- Last update timestamp: `stat /var/cache/apt/pkgcache.bin 2>/dev/null | grep Modify`

### 3. SSH Hardening
- Config: `grep -E "^(PermitRootLogin|PasswordAuthentication|Port |PubkeyAuthentication|MaxAuthTries|AllowUsers)" /etc/ssh/sshd_config`
- Check if root login should be disabled
- Check if password auth should be switched to key-only

### 4. Firewall
- UFW: `ufw status verbose 2>/dev/null`
- iptables: `iptables -L -n 2>/dev/null | head -30`
- fail2ban: `systemctl status fail2ban 2>/dev/null && fail2ban-client status 2>/dev/null`

### 5. PHP Configuration
- Version: `php -v 2>/dev/null`
- Key settings: `php -i 2>/dev/null | grep -E "^(memory_limit|max_execution_time|upload_max_filesize|post_max_size|max_input_vars|opcache.enable |opcache.memory_consumption)"`
- PHP-FPM status: `systemctl status php*-fpm 2>/dev/null`
- PHP-FPM pool: `cat /etc/php/*/fpm/pool.d/www.conf 2>/dev/null | grep -E "^(pm |pm\.|request_terminate_timeout)"`

### 6. MySQL/MariaDB
- Version: `mysql --version 2>/dev/null`
- Status: `systemctl status mysql 2>/dev/null || systemctl status mariadb 2>/dev/null`
- Key vars: `mysql -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size'; SHOW VARIABLES LIKE 'max_connections'; SHOW VARIABLES LIKE 'query_cache%'; SHOW VARIABLES LIKE 'slow_query_log';" 2>/dev/null`

### 7. Web Server (Nginx or Apache)
- Detect which: `systemctl status nginx 2>/dev/null; systemctl status apache2 2>/dev/null; systemctl status httpd 2>/dev/null`
- Config test: `nginx -t 2>&1` or `apachectl configtest 2>&1`
- Gzip/deflate: check main config
- Workers/processes config
- SSL config in site vhosts

### 8. SSL/TLS
- Certificate: `echo | openssl s_client -connect DOMAIN:443 2>/dev/null | openssl x509 -noout -dates -subject 2>/dev/null`
- Auto-renewal: `systemctl status certbot.timer 2>/dev/null; crontab -l 2>/dev/null | grep certbot`

### 9. WordPress-Specific Server Checks
- Find WP installations: `find /var/www -name wp-config.php -maxdepth 4 2>/dev/null`
- wp-config.php permissions: `find /var/www -name wp-config.php -exec stat -c "%a %U:%G %n" {} \; 2>/dev/null`
- Uploads dir permissions: `find /var/www -path "*/wp-content/uploads" -exec stat -c "%a %U:%G %n" {} \; 2>/dev/null`
- Debug mode: `find /var/www -name wp-config.php -exec grep WP_DEBUG {} \; 2>/dev/null`
- WP-Cron vs system cron: `crontab -l 2>/dev/null | grep wp-cron`
- WP-CLI: `which wp 2>/dev/null; wp --version 2>/dev/null`

### 10. Resource Usage
- Top processes: `ps aux --sort=-%mem | head -10`
- Load average: `cat /proc/loadavg`
- IO wait: `iostat -x 1 1 2>/dev/null | tail -5`

## Report Format
Save report to: `PROJECT_ROOT/vps-reports/vps-audit-DOMAIN-YYYYMMDD.md`

Where PROJECT_ROOT is the current working directory.

```markdown
# VPS Server Audit Report
**Domain:** example.com
**Server:** IP_ADDRESS
**Date:** YYYY-MM-DD HH:MM
**Overall Status:** HEALTHY / NEEDS ATTENTION / CRITICAL

## Summary
| Category | Status | Issues |
|----------|--------|--------|
| OS & Updates | OK/WARN/CRITICAL | brief |
| SSH Security | OK/WARN/CRITICAL | brief |
| Firewall | OK/WARN/CRITICAL | brief |
| PHP | OK/WARN/CRITICAL | brief |
| Database | OK/WARN/CRITICAL | brief |
| Web Server | OK/WARN/CRITICAL | brief |
| SSL/TLS | OK/WARN/CRITICAL | brief |
| WordPress Config | OK/WARN/CRITICAL | brief |
| Resources | OK/WARN/CRITICAL | brief |

## Detailed Findings
### [Category Name]
**Status:** OK/WARN/CRITICAL
**Current:** what is configured now
**Recommended:** what should be configured
**Fix:**
\```bash
# copy-paste ready command to fix
\```
**Why:** explanation of why this matters

## Action Items
### Critical (Fix Now)
- [ ] item with exact commands

### Warnings (Should Fix)
- [ ] item with exact commands

### Recommendations (Nice to Have)
- [ ] item with exact commands
```

## Important Rules
- NEVER save SSH credentials to any file
- Run ALL commands via SSH remotely, not locally
- If a command fails, note it in report and move on
- Detect OS type (Debian/Ubuntu vs CentOS/RHEL) and adjust commands accordingly
- All fix commands must be copy-paste ready
- Explain WHY each fix matters
- Check WordPress path dynamically — don't hardcode /var/www/html
