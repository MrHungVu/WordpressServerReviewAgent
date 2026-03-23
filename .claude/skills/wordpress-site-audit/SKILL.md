---
name: wordpress-site-audit
description: Audit WordPress installation via WP-CLI over SSH and source code review. Auto-detects LSPHP for OLS servers. Checks core version, plugins, themes, user security, database prefix, REST API exposure, XML-RPC, login protection, file permissions, code quality, LSCache config, Redis per-site DB validation, custom login URL, and WooCommerce. Researches plugins on internet for known vulnerabilities. Use when user wants to check/verify/audit a WordPress site.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---

# WordPress Site Audit Skill

## Purpose
Audit a WordPress installation via WP-CLI (over SSH) and direct source code review. Check application-level security, plugins, themes, and code quality. Works on both Nginx (v1) and OpenLiteSpeed (OLS v2) servers.

## Required Input (from user, NEVER save to files)
- SSH host, user, password, port (reuse from vps-server-audit if available)
- Domain name
- WordPress admin URL (optional, for reference)
- WordPress install path (auto-detect if not provided)

## Prerequisites — LSPHP Auto-Detect for WP-CLI
On OLS servers, system PHP may lack the mysqli extension. Auto-detect the correct PHP interpreter:

```bash
# Try system PHP first
if wp --info --allow-root 2>/dev/null | grep -q "WP-CLI"; then
  WP_CLI="wp"
else
  # Try LSPHP wrapper
  LSPHP_BIN=$(ls /usr/local/lsws/lsphp*/bin/lsphp 2>/dev/null | tail -1)
  if [ -n "$LSPHP_BIN" ]; then
    # Check if WP-CLI phar exists
    if [ -f /usr/local/bin/wp ]; then
      WP_CLI="$LSPHP_BIN /usr/local/bin/wp"
    else
      # Install WP-CLI
      curl -sO https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
      chmod +x wp-cli.phar
      mv wp-cli.phar /usr/local/bin/wp
      WP_CLI="$LSPHP_BIN /usr/local/bin/wp"
    fi
    # Verify
    $WP_CLI --info --allow-root 2>/dev/null
  else
    echo "CRITICAL: Neither system PHP nor LSPHP found. WP-CLI audit cannot proceed."
  fi
fi
```

ALL subsequent WP-CLI commands use `${WP_CLI}` instead of hardcoded `wp`.

## SSH Command Pattern
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

## Auto-Detect WordPress Path
```bash
find /var/www -name wp-config.php -maxdepth 4 2>/dev/null
```

## WP-CLI Commands (run with --allow-root if root user, --path=WP_PATH)
All WP-CLI commands: `${WP_CLI} --allow-root --path=/path/to/wordpress COMMAND`

## Audit Checklist

### 1. WordPress Core
- Version: `${WP_CLI} core version`
- Check updates: `${WP_CLI} core check-update`
- Verify checksums: `${WP_CLI} core verify-checksums` (detects modified core files)

### 2. Plugins Audit
- List all: `${WP_CLI} plugin list --format=csv`
- Check for updates: `${WP_CLI} plugin list --update=available --format=csv`
- List inactive: `${WP_CLI} plugin list --status=inactive --format=csv`
- For EACH plugin:
  - Note version, status, update availability
  - WebSearch: "[plugin-name] wordpress vulnerability" to check known CVEs
  - WebSearch: "[plugin-name] wordpress security issue" for recent reports
  - Check plugin slug on WordPress.org for last updated date, compatibility

### 3. LSCache Plugin Detection
Run after Plugins Audit. Check LSCache config and Redis object cache:

```bash
# Check if LSCache installed and active
${WP_CLI} plugin is-active litespeed-cache --path=${WP_PATH} --allow-root 2>/dev/null
LSCACHE_STATUS=$?

if [ $LSCACHE_STATUS -eq 0 ]; then
  echo "LSCache: ACTIVE"

  # Check LSCache version
  ${WP_CLI} plugin get litespeed-cache --field=version --path=${WP_PATH} --allow-root

  # Check object cache config
  OBJECT_KIND=$(${WP_CLI} option get litespeed.conf.object_kind --path=${WP_PATH} --allow-root 2>/dev/null)
  OBJECT_HOST=$(${WP_CLI} option get litespeed.conf.object_host --path=${WP_PATH} --allow-root 2>/dev/null)
  OBJECT_PORT=$(${WP_CLI} option get litespeed.conf.object_port --path=${WP_PATH} --allow-root 2>/dev/null)
  OBJECT_DB=$(${WP_CLI} option get litespeed.conf.object_db_id --path=${WP_PATH} --allow-root 2>/dev/null)

  echo "Object Cache: kind=${OBJECT_KIND} host=${OBJECT_HOST} port=${OBJECT_PORT} db=${OBJECT_DB}"

  # Validate object cache is connected
  ${WP_CLI} eval 'echo wp_cache_get("test_key") !== false ? "connected" : "not_connected";' \
    --path=${WP_PATH} --allow-root 2>/dev/null

else
  echo "LSCache: NOT ACTIVE (WARN - recommended for OLS)"
fi
```

### 4. Redis Per-Site DB Validation
Cross-validate wp-config.php constants vs LSCache vs actual Redis DB:

```bash
# Read WP_REDIS_DATABASE from wp-config.php
WP_REDIS_DB=$(grep "WP_REDIS_DATABASE" ${WP_PATH}/wp-config.php 2>/dev/null | grep -oP '\d+')
WP_REDIS_PREFIX=$(grep "WP_REDIS_PREFIX" ${WP_PATH}/wp-config.php 2>/dev/null | grep -oP "'[^']+'" | tr -d "'")

echo "wp-config WP_REDIS_DATABASE: ${WP_REDIS_DB:-not set}"
echo "wp-config WP_REDIS_PREFIX: ${WP_REDIS_PREFIX:-not set}"

# Check actual Redis DB has keys
if [ -n "$WP_REDIS_DB" ]; then
  ACTUAL_KEYS=$(redis-cli -n ${WP_REDIS_DB} DBSIZE 2>/dev/null | awk '{print $2}')
  echo "Redis DB ${WP_REDIS_DB} actual keys: ${ACTUAL_KEYS}"

  # Cross-validate: LSCache object_db_id should match WP_REDIS_DATABASE
  if [ -n "$OBJECT_DB" ] && [ "$OBJECT_DB" != "$WP_REDIS_DB" ]; then
    echo "WARN: LSCache object_db_id (${OBJECT_DB}) != wp-config WP_REDIS_DATABASE (${WP_REDIS_DB})"
  fi
fi
```

Skip this section if Redis is not installed on the server.

### 5. Themes Audit
- List all: `${WP_CLI} theme list --format=csv`
- Active theme: `${WP_CLI} theme list --status=active --format=csv`
- Check updates: `${WP_CLI} theme list --update=available --format=csv`
- Inactive themes should be deleted (attack surface)
- Check if child theme is used (important for updates)

### 6. User Security
- List users: `${WP_CLI} user list --format=csv`
- Check if any admin username is "admin" (CRITICAL)
- Check user roles — minimize admin accounts
- Check for unused/dormant accounts

### 7. WordPress Configuration (wp-config.php)
- Read via SSH: `cat WP_PATH/wp-config.php`
- Check:
  - `WP_DEBUG` should be false in production
  - `DISALLOW_FILE_EDIT` should be true
  - `WP_POST_REVISIONS` — should be limited (3-5)
  - `DB table prefix` — should NOT be default `wp_`
  - Security keys/salts — should be set (not empty)
  - `WP_SITEURL` and `WP_HOME` — match domain
  - `FORCE_SSL_ADMIN` — should be true

### 8. Custom Login URL Check
Check if a custom login URL is configured to protect wp-login.php:

```bash
# Check wp-config.php for custom login constant
LOGIN_SLUG=$(grep -oP "LOGIN_SLUG.*?'([^']+)'" ${WP_PATH}/wp-config.php 2>/dev/null | grep -oP "'[^']+'" | tr -d "'")

if [ -n "$LOGIN_SLUG" ]; then
  echo "Custom Login URL: /${LOGIN_SLUG}"

  # Check OLS vhost rewrite rules contain the slug
  DOMAIN_SLUG=$(basename $(dirname ${WP_PATH}))
  if [ -f "/usr/local/lsws/conf/vhosts/${DOMAIN_SLUG}.conf" ]; then
    grep -q "${LOGIN_SLUG}" "/usr/local/lsws/conf/vhosts/${DOMAIN_SLUG}.conf" 2>/dev/null && \
      echo "OLS rewrite for custom login: CONFIGURED" || \
      echo "WARN: Custom login slug in wp-config but NOT in OLS rewrite rules"
  fi

  # Test: direct wp-login.php should be blocked (check OLS rewrite)
  grep -q "wp-login.*\[F" "/usr/local/lsws/conf/vhosts/${DOMAIN_SLUG}.conf" 2>/dev/null && \
    echo "Direct wp-login.php: BLOCKED (OK)" || \
    echo "WARN: wp-login.php not blocked in OLS rewrite rules"
else
  echo "Custom Login URL: NOT configured (WARN - using default wp-login.php)"
fi
```

### 9. Security Checks
- XML-RPC: `${WP_CLI} option get xmlrpc_enabled 2>/dev/null` or check if xmlrpc.php accessible
- REST API: check if user enumeration possible via `/wp-json/wp/v2/users`
- File editor: verify DISALLOW_FILE_EDIT
- .htaccess security rules (if Apache/Nginx) or OLS rewrite rules
- Directory listing disabled
- wp-includes/wp-admin protection

### 10. WooCommerce Audit
Check WooCommerce status and LSCache compatibility:

```bash
# Check if WooCommerce active
${WP_CLI} plugin is-active woocommerce --path=${WP_PATH} --allow-root 2>/dev/null
WOO_STATUS=$?

if [ $WOO_STATUS -eq 0 ]; then
  echo "WooCommerce: ACTIVE"
  ${WP_CLI} plugin get woocommerce --field=version --path=${WP_PATH} --allow-root
  ${WP_CLI} plugin get woocommerce --field=update_available --path=${WP_PATH} --allow-root

  # Check LSCache WooCommerce exclusions
  if [ $LSCACHE_STATUS -eq 0 ]; then
    CDN_EXC=$(${WP_CLI} option get litespeed.conf.cdn_exc --path=${WP_PATH} --allow-root 2>/dev/null)
    echo "LSCache cache exclusions: ${CDN_EXC}"
    echo "$CDN_EXC" | grep -q "cart" && echo "Cart exclusion: OK" || echo "WARN: /cart/ not excluded from cache"
    echo "$CDN_EXC" | grep -q "checkout" && echo "Checkout exclusion: OK" || echo "WARN: /checkout/ not excluded"
    echo "$CDN_EXC" | grep -q "my-account" && echo "My-account exclusion: OK" || echo "WARN: /my-account/ not excluded"
  fi
else
  echo "WooCommerce: NOT installed"
fi
```

### 11. Database
- DB prefix: `${WP_CLI} db prefix`
- DB size: `${WP_CLI} db size --format=csv`
- Check for orphaned tables
- Transients cleanup needed: `${WP_CLI} transient list --format=count 2>/dev/null`

### 12. Source Code Review (via SSH file access)
- Check for hardcoded credentials in theme/plugin files
- Check for eval(), base64_decode() in non-core files (malware indicators)
- Check custom plugin/theme code quality
- Check .htaccess for suspicious redirects
```bash
# Malware scan - suspicious patterns in wp-content
grep -rl "eval(base64_decode" WP_PATH/wp-content/ 2>/dev/null
grep -rl "eval(gzinflate" WP_PATH/wp-content/ 2>/dev/null
grep -rl "@include.*\.ico" WP_PATH/wp-content/ 2>/dev/null
grep -rl "shell_exec\|system\|passthru\|exec(" WP_PATH/wp-content/plugins/ 2>/dev/null
```

### 13. Performance Checks
- Object cache: `${WP_CLI} cache type 2>/dev/null`
- Autoloaded options size: `${WP_CLI} db query "SELECT SUM(LENGTH(option_value)) as autoload_size FROM $(${WP_CLI} db prefix)options WHERE autoload='yes'" --allow-root`
- Cron events: `${WP_CLI} cron event list --format=csv`
- Check for excessive cron jobs

### 14. .htaccess / Nginx / OLS Config Review
- Read .htaccess (Nginx/Apache): `cat WP_PATH/.htaccess`
- Read OLS vhost config: `cat /usr/local/lsws/conf/vhosts/DOMAIN.conf`
- Check security headers (X-Frame-Options, X-Content-Type-Options, etc.)
- Check redirect rules
- Check if wp-admin/wp-login has IP restriction or rate limiting

### 15. Backup Check
```bash
# Per-site backup script
DOMAIN_SLUG=$(echo ${DOMAIN} | tr '.-' '__')
ls -la /usr/local/bin/backup-${DOMAIN_SLUG}.sh 2>/dev/null && \
  echo "Backup script: EXISTS" || echo "WARN: No backup script found"

# Check cron
crontab -l 2>/dev/null | grep "backup-${DOMAIN_SLUG}" && \
  echo "Backup cron: CONFIGURED" || echo "WARN: No backup cron job"

# Check recent backups
RECENT=$(find /var/backups/wordpress/ -name "${DOMAIN}*" -mtime -7 2>/dev/null | wc -l)
echo "Recent backups (7 days): ${RECENT}"
if [ "$RECENT" -eq 0 ]; then
  echo "WARN: No backups in last 7 days"
fi
```

## Report Format
Save report to: `PROJECT_ROOT/vps-reports/wordpress-audit-DOMAIN-YYYYMMDD.md`

```markdown
# WordPress Site Audit Report
**Domain:** example.com
**WordPress Version:** X.X.X
**PHP Version:** X.X
**Date:** YYYY-MM-DD HH:MM
**Overall Status:** HEALTHY / NEEDS ATTENTION / CRITICAL

## Summary
| Category | Status | Issues |
|----------|--------|--------|
| Core Version | OK/WARN/CRITICAL | brief |
| Plugins | OK/WARN/CRITICAL | X outdated, Y inactive |
| Themes | OK/WARN/CRITICAL | brief |
| User Security | OK/WARN/CRITICAL | brief |
| Configuration | OK/WARN/CRITICAL | brief |
| Security | OK/WARN/CRITICAL | brief |
| Database | OK/WARN/CRITICAL | brief |
| Code Quality | OK/WARN/CRITICAL | brief |
| Performance | OK/WARN/CRITICAL | brief |
| LSCache | OK/WARN/CRITICAL | brief |
| Redis | OK/WARN/CRITICAL | brief |
| WooCommerce | OK/WARN/N/A | brief |
| Custom Login | OK/WARN | brief |
| Backups | OK/WARN | brief |

## Plugin Inventory
| Plugin | Version | Status | Update Available | Known Vulnerabilities |
|--------|---------|--------|------------------|----------------------|
| name | ver | active/inactive | yes/no | CVE or none |

## Theme Inventory
| Theme | Version | Status | Update Available |
|-------|---------|--------|------------------|
| name | ver | active/inactive | yes/no |

## Detailed Findings
### [Category]
**Status:** OK/WARN/CRITICAL
**Current:** what is found
**Recommended:** what should be
**Fix:**
\```bash
# WP-CLI command or manual fix
\```
**Why:** explanation

## LSCache Configuration
| Setting | Value | Status |
|---------|-------|--------|
| Plugin Active | yes/no | OK/WARN |
| Object Cache | Redis DB {N} | OK/WARN |
| WooCommerce Exclusions | configured/missing | OK/WARN |

## Redis Validation
| Check | Value | Status |
|-------|-------|--------|
| WP_REDIS_DATABASE | {N} | OK/WARN |
| Actual Keys in DB | {count} | INFO |
| LSCache DB Match | yes/no | OK/WARN |

## Custom Login URL
| Check | Status |
|-------|--------|
| Login Slug Configured | yes/no |
| OLS Rewrite Rule | present/missing |
| wp-login.php Blocked | yes/no |

## WooCommerce
| Setting | Value | Status |
|---------|-------|--------|
| Version | X.X | OK/WARN |
| Update Available | yes/no | OK/WARN |
| Cache Exclusions | cart/checkout/my-account | OK/WARN |

## Malware Scan Results
- Suspicious files found: list or "None detected"

## Action Items
### Critical (Fix Now)
- [ ] item with exact WP-CLI commands

### Warnings (Should Fix)
- [ ] item

### Recommendations (Nice to Have)
- [ ] item
```

## Important Rules
- NEVER save any credentials to files
- Use `${WP_CLI}` variable (never hardcode `wp`) — auto-detected in Prerequisites
- Use WP-CLI with --allow-root when SSH user is root
- Always specify --path= for WP-CLI commands
- Research EVERY active plugin for known vulnerabilities via WebSearch
- Check plugin last updated date — plugins not updated in 2+ years are risky
- Look for malware indicators in source code
- Be specific — provide exact WP-CLI commands to fix issues
- Skip Redis sections if Redis is not installed on the server
- Custom Login URL and OLS rewrite checks apply to OLS servers only; skip gracefully on Nginx
