---
name: vps-server-audit
description: SSH into a VPS server running WordPress, audit OS, PHP (system + LSPHP auto-detect), MySQL, web server (Nginx/Apache/OpenLiteSpeed detection), SSL, firewall (UFW + Cloudflare-only rules), security hardening, Redis status, OPcache stats, disk/RAM, file permissions, and WordPress-specific configs. Produces a detailed report with status levels (OK/WARN/CRITICAL) and fix commands. Use when user wants to check/verify/audit a VPS server configuration. Supports both Nginx (v1) and OpenLiteSpeed/LSPHP/Redis (v2) stacks.
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
Audit a VPS server running WordPress via SSH. Check all server-level configurations and produce a comprehensive report. Supports both Nginx (v1) and OpenLiteSpeed/LSPHP/Redis/OPcache (v2) stacks — detects which is present automatically.

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
- **UFW Cloudflare-only rules validation:**
```bash
if command -v ufw &>/dev/null; then
  # Show 80/443 rules
  ufw status numbered 2>/dev/null | grep -E "80|443"
  # Count CF-specific IP range rules
  CF_RULES=$(ufw status 2>/dev/null | grep -cE "(103\.21\.|103\.22\.|104\.16\.|108\.162\.|131\.0\.|141\.101\.|162\.158\.|172\.64\.|173\.245\.|188\.114\.|190\.93\.|197\.234\.|198\.41\.)")
  if [ "$CF_RULES" -gt 0 ]; then
    echo "UFW: Cloudflare-only rules detected ($CF_RULES rules)"
  else
    echo "UFW: WARN - No Cloudflare IP restrictions on 80/443 (direct IP access possible)"
  fi
fi
```

### 5. PHP Configuration
- System PHP version: `php -v 2>/dev/null`
- Key settings: `php -i 2>/dev/null | grep -E "^(memory_limit|max_execution_time|upload_max_filesize|post_max_size|max_input_vars|opcache.enable |opcache.memory_consumption)"`
- PHP-FPM status: `systemctl status php*-fpm 2>/dev/null`
- PHP-FPM pool: `cat /etc/php/*/fpm/pool.d/www.conf 2>/dev/null | grep -E "^(pm |pm\.|request_terminate_timeout)"`
- **LSPHP auto-detect (OLS stacks):**
```bash
# Find all installed LSPHP versions
ls -la /usr/local/lsws/lsphp*/bin/lsphp 2>/dev/null

# Report version for each
for lsphp in /usr/local/lsws/lsphp*/bin/lsphp; do
  [ -x "$lsphp" ] && $lsphp -v 2>/dev/null | head -1
done

# Check LSPHP extensions (use newest installed)
LSPHP_BIN=$(ls /usr/local/lsws/lsphp*/bin/lsphp 2>/dev/null | tail -1)
if [ -n "$LSPHP_BIN" ]; then
  $LSPHP_BIN -m 2>/dev/null | grep -E "redis|opcache|mysqli|imagick|curl|gd|mbstring"
fi
```

### 6. MySQL/MariaDB
- Version: `mysql --version 2>/dev/null`
- Status: `systemctl status mysql 2>/dev/null || systemctl status mariadb 2>/dev/null`
- Key vars: `mysql -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size'; SHOW VARIABLES LIKE 'max_connections'; SHOW VARIABLES LIKE 'query_cache%'; SHOW VARIABLES LIKE 'slow_query_log';" 2>/dev/null`

### 7. Web Server (Nginx, Apache, or OpenLiteSpeed)
- **Detect active web server:**
```bash
NGINX_ACTIVE=$(systemctl is-active nginx 2>/dev/null)
APACHE_ACTIVE=$(systemctl is-active apache2 2>/dev/null || systemctl is-active httpd 2>/dev/null)
OLS_ACTIVE=$(systemctl is-active lsws 2>/dev/null)

echo "Nginx: $NGINX_ACTIVE | Apache: $APACHE_ACTIVE | OLS: $OLS_ACTIVE"
```
- Nginx config test: `nginx -t 2>&1` (if active)
- Apache config test: `apachectl configtest 2>&1` (if active)
- Gzip/deflate: check main config
- Workers/processes config
- SSL config in site vhosts
- **OLS-specific checks (only run if OLS active):**
```bash
if [ "$OLS_ACTIVE" = "active" ]; then
  # OLS version
  /usr/local/lsws/bin/lshttpd -v 2>&1
  # Config syntax test
  /usr/local/lsws/bin/openlitespeed -t 2>&1

  # Parse httpd_config.conf
  if [ -f /usr/local/lsws/conf/httpd_config.conf ]; then
    # Count listeners
    grep -c "^listener " /usr/local/lsws/conf/httpd_config.conf
    # List vhost mappings
    grep -A50 "vhostMap" /usr/local/lsws/conf/httpd_config.conf | head -30
    # List external apps (LSPHP handlers)
    grep "extprocessor\|extAppServer" /usr/local/lsws/conf/httpd_config.conf
    # Check LSAPI_CHILDREN per ext app
    grep -A5 "extprocessor" /usr/local/lsws/conf/httpd_config.conf | grep LSAPI_CHILDREN
  fi

  # WebAdmin console disabled check
  if [ -f /usr/local/lsws/conf/disablewebconsole ]; then
    echo "WebAdmin: DISABLED (OK)"
  else
    echo "WebAdmin: ENABLED (WARN - should be disabled in production)"
  fi

  # CF-Connecting-IP configuration check
  if [ -f /usr/local/lsws/conf/httpd_config.conf ]; then
    grep -E "useClientIpInHeader|clientIpHeader" /usr/local/lsws/conf/httpd_config.conf
    # Expected: useClientIpInHeader 2, clientIpHeader CF-Connecting-IP
    # If missing or 0: logs show CF IPs not real visitor IPs (WARN)
  fi
fi
```

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

### 11. Redis Status
Run only if Redis is detected (`systemctl is-active redis-server` or `redis-server`):
```bash
# Check Redis running state
systemctl is-active redis-server 2>/dev/null || systemctl is-active redis 2>/dev/null

# Memory info
redis-cli INFO memory 2>/dev/null | grep -E "used_memory_human|maxmemory_human|maxmemory_policy"

# Per-database key count (skip empty DBs)
for db in $(seq 0 15); do
  COUNT=$(redis-cli -n $db DBSIZE 2>/dev/null)
  if [ -n "$COUNT" ] && [ "$COUNT" -gt 0 ] 2>/dev/null; then
    echo "Redis DB $db: $COUNT keys"
  fi
done

# Memory fragmentation ratio (>1.5 = WARN, >2.0 = CRITICAL)
redis-cli INFO memory 2>/dev/null | grep "mem_fragmentation_ratio"
```
If Redis not installed, note "Redis: not installed (skip)" and continue.

### 12. OPcache Stats via LSPHP
Run only if LSPHP binary is found:
```bash
LSPHP_BIN=$(ls /usr/local/lsws/lsphp*/bin/lsphp 2>/dev/null | tail -1)
if [ -n "$LSPHP_BIN" ]; then
  $LSPHP_BIN -r '
    $status = opcache_get_status(false);
    if ($status) {
      echo "OPcache enabled: " . ($status["opcache_enabled"] ? "yes" : "no") . "\n";
      echo "Memory used: " . round($status["memory_usage"]["used_memory"]/1024/1024, 1) . "MB\n";
      echo "Memory free: " . round($status["memory_usage"]["free_memory"]/1024/1024, 1) . "MB\n";
      echo "Hit rate: " . round($status["opcache_statistics"]["opcache_hit_rate"], 1) . "%\n";
      echo "Cached files: " . $status["opcache_statistics"]["num_cached_scripts"] . "\n";
      echo "JIT enabled: " . (isset($status["jit"]["enabled"]) && $status["jit"]["enabled"] ? "yes" : "no") . "\n";
    } else {
      echo "OPcache: NOT enabled\n";
    }
  ' 2>/dev/null
else
  echo "OPcache: LSPHP not found (system PHP CLI OPcache may differ from web runtime)"
fi
```
Note: CLI OPcache stats reflect CLI configuration; web runtime stats may differ. Flag hit rate <98% as WARN.

### 13. OLS-Specific Checks
Run only if OLS is detected:
```bash
# Log rotation configured?
if ls /etc/logrotate.d/openlitespeed 2>/dev/null; then
  echo "OLS logrotate: configured (OK)"
else
  echo "OLS logrotate: MISSING (WARN - logs will grow unbounded)"
fi
```

### 14. Backup Status
```bash
# Backup directory exists?
if [ -d /var/backups/wordpress ]; then
  echo "Backup dir: exists (OK)"
  # List most recent backup files
  ls -lt /var/backups/wordpress/*.gz 2>/dev/null | head -3
  # Count backups in last 7 days
  find /var/backups/wordpress -name "*.gz" -mtime -7 2>/dev/null | wc -l
else
  echo "Backup dir: MISSING - /var/backups/wordpress not found (CRITICAL)"
fi

# Backup cron jobs configured?
crontab -l 2>/dev/null | grep -i backup
```

## Report Format
Save report to: `PROJECT_ROOT/vps-reports/vps-audit-DOMAIN-YYYYMMDD.md`

Where PROJECT_ROOT is the current working directory.

```markdown
# VPS Server Audit Report
**Domain:** example.com
**Server:** IP_ADDRESS
**Date:** YYYY-MM-DD HH:MM
**Stack:** Nginx/OpenLiteSpeed (detected)
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
| Redis | OK/WARN/CRITICAL/N/A | brief |
| OPcache | OK/WARN/CRITICAL/N/A | brief |
| Backup | OK/WARN/CRITICAL | brief |

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

## OpenLiteSpeed (if detected)
| Setting | Current | Recommended | Status |
|---------|---------|-------------|--------|
| OLS Version | X.X | latest | OK/WARN |
| LSPHP Version | 8.X | 8.2+ | OK/WARN |
| WebAdmin Console | disabled/enabled | disabled | OK/WARN |
| Config Test | pass/fail | pass | OK/CRITICAL |
| CF-Connecting-IP | configured/missing | configured | OK/WARN |
| LSAPI_CHILDREN | N per site | varies by RAM | OK/WARN |

## Redis (if detected)
| Metric | Value | Status |
|--------|-------|--------|
| Status | active/inactive | OK/CRITICAL |
| Memory Used | XMB / XMB max | OK/WARN |
| Eviction Policy | allkeys-lru | OK/WARN |
| Active DBs | N (list DB IDs) | INFO |
| Fragmentation Ratio | X.X | OK/WARN (>1.5=WARN, >2.0=CRITICAL) |

## OPcache (if LSPHP detected)
| Metric | Value | Status |
|--------|-------|--------|
| Enabled | yes/no | OK/CRITICAL |
| Hit Rate | X% | OK/WARN (<98% is WARN) |
| Memory Used | XMB / XMB total | OK/WARN (>90% is WARN) |
| Cached Files | N | INFO |
| JIT | enabled/disabled | OK/WARN |

## Backup Status
| Check | Status |
|-------|--------|
| Backup dir exists | OK/MISSING |
| Recent backups (last 7 days) | N files / NONE |
| Backup cron | configured / MISSING |

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
- Detect web server type (Nginx / Apache / OLS) — run OLS-specific checks only if OLS is active
- Skip Redis section if Redis is not installed; skip OPcache/LSPHP section if LSPHP not found
- All fix commands must be copy-paste ready
- Explain WHY each fix matters
- Check WordPress path dynamically — don't hardcode /var/www/html
