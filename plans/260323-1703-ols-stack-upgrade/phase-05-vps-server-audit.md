# Phase 5: vps-server-audit v2 (UPDATE)

## Context Links

- Current skill: `.claude/skills/vps-server-audit/SKILL.md`
- Brainstorm: [Audit Skills Impact](../reports/brainstorm-260323-1703-ols-stack-upgrade.md)
- Research: [OLS Production](../reports/researcher-260323-1708-openlitespeed-wordpress-production.md)

## Overview

- **Priority:** P2
- **Status:** complete
- **Effort:** 3h
- **Description:** Update VPS audit to detect and validate OLS stack alongside existing Nginx/Apache detection. Add LSPHP auto-detect, OLS config parsing, Redis status checks, OPcache stats, UFW Cloudflare rules validation, WebAdmin disabled check, CF-Connecting-IP config check. Keep all existing audit checks -- they still apply.

## Key Insights

- Must support BOTH Nginx (v1 setups) and OLS (v2 setups) -- detect which is present
- OLS detection: check `systemctl status lsws` + existence of `/usr/local/lsws/`
- LSPHP auto-detect: `ls /usr/local/lsws/lsphp*/bin/lsphp` (find all installed versions)
- OLS config is at `/usr/local/lsws/conf/httpd_config.conf` (plain text)
- Redis per-site validation: iterate DB 0-15, check DBSIZE for each
- Existing PHP/MySQL/firewall/SSH/fail2ban checks remain unchanged

## Requirements

### Functional (NEW checks to add)
- F1: Detect OLS: `systemctl status lsws`, check `/usr/local/lsws/`
- F2: LSPHP auto-detect: find all `lsphp*` binaries, report version
- F3: OLS config validation: parse httpd_config.conf for listeners, vhosts, ext apps
- F4: WebAdmin console disabled: check `/usr/local/lsws/conf/disablewebconsole` exists
- F5: CF-Connecting-IP configured: grep `useClientIpInHeader` + `clientIpHeader` in httpd_config.conf
- F6: Redis status: `systemctl status redis`, `redis-cli INFO memory`, `DBSIZE` per active DB
- F7: OPcache stats: run LSPHP to get `opcache_get_status()` output
- F8: UFW Cloudflare rules: verify 80/443 rules only allow CF IP ranges
- F9: OLS log rotation check: verify `/etc/logrotate.d/openlitespeed` exists
- F10: Backup infrastructure: check `/var/backups/wordpress/` exists + recent backup files

### Functional (KEEP existing)
- All v1 checks: OS, kernel, uptime, CPU, RAM, disk, security updates, SSH hardening, firewall, PHP (system), MySQL/MariaDB, web server (Nginx/Apache), SSL/TLS, WordPress-specific server checks, resource usage, fail2ban

## Architecture

No architectural change. Same SSH-based audit, more checks added.

```
Audit Flow (updated):
1. System info (existing)
2. Security updates (existing)
3. SSH hardening (existing)
4. Firewall (existing + NEW: CF IP validation)
5. PHP (existing + NEW: LSPHP detection)
6. MySQL/MariaDB (existing)
7. Web Server (existing + NEW: OLS detection + config parse)
8. SSL/TLS (existing + NEW: OLS cert path check)
9. WordPress-specific (existing)
10. Resource usage (existing)
11. NEW: Redis status + per-DB stats
12. NEW: OPcache stats
13. NEW: OLS-specific (WebAdmin, CF-Connecting-IP, log rotation)
14. NEW: Backup status
```

## Related Code Files

- **Modify:** `.claude/skills/vps-server-audit/SKILL.md`

## Implementation Steps

### Step 1: Update Web Server Detection (Section 7)
Add OLS detection alongside Nginx/Apache:

```bash
# Detect web server type
NGINX_ACTIVE=$(systemctl is-active nginx 2>/dev/null)
APACHE_ACTIVE=$(systemctl is-active apache2 2>/dev/null || systemctl is-active httpd 2>/dev/null)
OLS_ACTIVE=$(systemctl is-active lsws 2>/dev/null)

if [ "$OLS_ACTIVE" = "active" ]; then
  echo "WEB_SERVER=OpenLiteSpeed"
  /usr/local/lsws/bin/lshttpd -v 2>&1
  # Config validation
  /usr/local/lsws/bin/openlitespeed -t 2>&1
fi
```

### Step 2: Add LSPHP Detection (extend Section 5)
```bash
# Auto-detect LSPHP versions
ls -la /usr/local/lsws/lsphp*/bin/lsphp 2>/dev/null
# Get version of each
for lsphp in /usr/local/lsws/lsphp*/bin/lsphp; do
  $lsphp -v 2>/dev/null | head -1
done
# Check LSPHP extensions
LSPHP_BIN=$(ls /usr/local/lsws/lsphp*/bin/lsphp 2>/dev/null | tail -1)
$LSPHP_BIN -m 2>/dev/null | grep -E "redis|opcache|mysqli|imagick|curl|gd|mbstring"
```

### Step 3: Add OLS Config Parsing (new subsection under Section 7)
```bash
if [ -f /usr/local/lsws/conf/httpd_config.conf ]; then
  # Count listeners
  grep -c "^listener " /usr/local/lsws/conf/httpd_config.conf
  # List vhost mappings
  grep -A50 "vhostMap" /usr/local/lsws/conf/httpd_config.conf | head -30
  # List external apps
  grep "extprocessor\|extAppServer" /usr/local/lsws/conf/httpd_config.conf
  # Check LSAPI_CHILDREN per ext app
  grep -A5 "extprocessor" /usr/local/lsws/conf/httpd_config.conf | grep LSAPI_CHILDREN
fi
```

### Step 4: Add WebAdmin Console Check
```bash
if [ -f /usr/local/lsws/conf/disablewebconsole ]; then
  echo "WebAdmin: DISABLED (OK)"
else
  echo "WebAdmin: ENABLED (WARN - should be disabled in production)"
fi
```

### Step 5: Add CF-Connecting-IP Check
```bash
if [ -f /usr/local/lsws/conf/httpd_config.conf ]; then
  grep -E "useClientIpInHeader|clientIpHeader" /usr/local/lsws/conf/httpd_config.conf
  # Expect: useClientIpInHeader 2, clientIpHeader CF-Connecting-IP
fi
```
If `useClientIpInHeader` is missing or 0: WARN -- logs will show CF IPs, not real visitor IPs.

### Step 6: Add Redis Status Check (new Section 11)
```bash
# Redis running?
systemctl is-active redis-server 2>/dev/null || systemctl is-active redis 2>/dev/null

# Memory info
redis-cli INFO memory 2>/dev/null | grep -E "used_memory_human|maxmemory_human|maxmemory_policy"

# Per-database key count
for db in $(seq 0 15); do
  COUNT=$(redis-cli -n $db DBSIZE 2>/dev/null | awk '{print $2}')
  if [ "$COUNT" -gt 0 ] 2>/dev/null; then
    echo "Redis DB $db: $COUNT keys"
  fi
done

# Memory fragmentation ratio
redis-cli INFO memory 2>/dev/null | grep "mem_fragmentation_ratio"
```

### Step 7: Add OPcache Stats (new Section 12)
```bash
# Use LSPHP to check OPcache status
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
fi
```

### Step 8: Add UFW Cloudflare Rules Validation (extend Section 4)
```bash
if command -v ufw &>/dev/null; then
  # Check if 80/443 rules specify source IPs (should be CF ranges)
  ufw status numbered 2>/dev/null | grep -E "80|443"
  # Count CF-specific rules
  CF_RULES=$(ufw status 2>/dev/null | grep -cE "(103\.21\.|103\.22\.|104\.16\.|108\.162\.|131\.0\.|141\.101\.|162\.158\.|172\.64\.|173\.245\.|188\.114\.|190\.93\.|197\.234\.|198\.41\.)")
  if [ "$CF_RULES" -gt 0 ]; then
    echo "UFW: Cloudflare-only rules detected ($CF_RULES rules)"
  else
    echo "UFW: WARN - No Cloudflare IP restrictions on 80/443 (direct IP access possible)"
  fi
fi
```

### Step 9: Add Log Rotation + Backup Check (new Section 13/14)
```bash
# Log rotation
ls -la /etc/logrotate.d/openlitespeed 2>/dev/null && echo "OLS logrotate: configured" || echo "OLS logrotate: MISSING"

# Backup infrastructure
ls -la /var/backups/wordpress/ 2>/dev/null
# Check most recent backup
ls -lt /var/backups/wordpress/*.gz 2>/dev/null | head -3
# Check backup cron jobs
crontab -l 2>/dev/null | grep backup
```

### Step 10: Update Report Format
Add new sections to audit report:

```markdown
## OpenLiteSpeed (if detected)
| Setting | Current | Recommended | Status |
|---------|---------|-------------|--------|
| OLS Version | X.X | latest | OK/WARN |
| LSPHP Version | 8.X | 8.2+ | OK/WARN |
| WebAdmin Console | disabled/enabled | disabled | OK/WARN |
| Config Format | plain text/XML | plain text | OK/WARN |
| CF-Connecting-IP | configured/missing | configured | OK/WARN |
| LSAPI_CHILDREN | N per site | varies by RAM | OK/WARN |

## Redis
| Metric | Value | Status |
|--------|-------|--------|
| Status | active/inactive | OK/CRITICAL |
| Memory Used | XMB / XMB max | OK/WARN |
| Eviction Policy | allkeys-lru | OK/WARN |
| Active DBs | N | INFO |
| Fragmentation Ratio | X.X | OK/WARN (>1.5 is WARN) |

## OPcache
| Metric | Value | Status |
|--------|-------|--------|
| Enabled | yes/no | OK/CRITICAL |
| Hit Rate | X% | OK/WARN (<98% is WARN) |
| Memory | X/XMB used | OK/WARN (>90% is WARN) |
| Cached Files | N | INFO |
| JIT | enabled/disabled | OK/WARN |

## Backup Status
| Check | Status |
|-------|--------|
| Backup dir exists | OK/MISSING |
| Recent backups | X files in last 7 days / NONE |
| Backup cron | configured / MISSING |
```

## Todo List

- [x] Add OLS detection alongside Nginx/Apache in web server section
- [x] Add LSPHP auto-detect (find all versions, check extensions)
- [x] Add OLS config parsing (listeners, vhosts, ext apps, LSAPI_CHILDREN)
- [x] Add WebAdmin disabled check
- [x] Add CF-Connecting-IP config check
- [x] Add Redis status + per-DB key count + memory/fragmentation
- [x] Add OPcache stats via LSPHP (hit rate, memory, JIT, cached files)
- [x] Add UFW Cloudflare-only rules validation
- [x] Add OLS log rotation check
- [x] Add backup infrastructure + recent backups check
- [x] Update report template with new sections

## Success Criteria

- Audit correctly identifies OLS stack (not just Nginx/Apache)
- LSPHP version detected and extensions listed
- Redis per-DB stats show active sites
- OPcache hit rate reported (flag if <98%)
- WebAdmin disabled status verified
- UFW Cloudflare-only rules flagged if missing
- Report includes all new sections with OK/WARN/CRITICAL status

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| OLS not detected on Nginx servers | LOW | Check both; OLS sections only render if OLS detected |
| LSPHP OPcache stats fail (CLI OPcache disabled) | LOW | Fallback: report "OPcache CLI disabled, check via web" |
| Redis not installed on v1 servers | LOW | Skip Redis section if not detected |

## Security Considerations

- No new credential requirements (reuses existing SSH creds)
- Redis audit reads INFO only (no data access)
- OPcache stats are read-only PHP function
- No write operations on server

## Next Steps

After this phase: Phase 6 (wordpress-site-audit) adds LSPHP WP-CLI, LSCache, Redis per-site, WooCommerce checks.
