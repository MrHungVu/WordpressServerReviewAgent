---
name: wordpress-site-audit
description: Audit WordPress installation via WP-CLI over SSH and source code review. Checks core version, plugins, themes, user security, database prefix, REST API exposure, XML-RPC, login protection, file permissions, and code quality. Researches plugins on internet for known vulnerabilities. Use when user wants to check/verify/audit a WordPress site.
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
Audit a WordPress installation via WP-CLI (over SSH) and direct source code review. Check application-level security, plugins, themes, and code quality.

## Required Input (from user, NEVER save to files)
- SSH host, user, password, port (reuse from vps-server-audit if available)
- Domain name
- WordPress admin URL (optional, for reference)
- WordPress install path (auto-detect if not provided)

## Prerequisites
- SSH access must be established (same as vps-server-audit)
- WP-CLI should be installed on server (check: `wp --version`)
- If WP-CLI not installed, install it:
```bash
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
mv wp-cli.phar /usr/local/bin/wp
```

## SSH Command Pattern
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

## Auto-Detect WordPress Path
```bash
find /var/www -name wp-config.php -maxdepth 4 2>/dev/null
```

## WP-CLI Commands (run with --allow-root if root user, --path=WP_PATH)
All WP-CLI commands: `wp --allow-root --path=/path/to/wordpress COMMAND`

## Audit Checklist

### 1. WordPress Core
- Version: `wp core version`
- Check updates: `wp core check-update`
- Verify checksums: `wp core verify-checksums` (detects modified core files)

### 2. Plugins Audit
- List all: `wp plugin list --format=csv`
- Check for updates: `wp plugin list --update=available --format=csv`
- List inactive: `wp plugin list --status=inactive --format=csv`
- For EACH plugin:
  - Note version, status, update availability
  - WebSearch: "[plugin-name] wordpress vulnerability" to check known CVEs
  - WebSearch: "[plugin-name] wordpress security issue" for recent reports
  - Check plugin slug on WordPress.org for last updated date, compatibility

### 3. Themes Audit
- List all: `wp theme list --format=csv`
- Active theme: `wp theme list --status=active --format=csv`
- Check updates: `wp theme list --update=available --format=csv`
- Inactive themes should be deleted (attack surface)
- Check if child theme is used (important for updates)

### 4. User Security
- List users: `wp user list --format=csv`
- Check if any admin username is "admin" (CRITICAL)
- Check user roles — minimize admin accounts
- Check for unused/dormant accounts

### 5. WordPress Configuration (wp-config.php)
- Read via SSH: `cat WP_PATH/wp-config.php`
- Check:
  - `WP_DEBUG` should be false in production
  - `DISALLOW_FILE_EDIT` should be true
  - `WP_POST_REVISIONS` — should be limited (3-5)
  - `DB table prefix` — should NOT be default `wp_`
  - Security keys/salts — should be set (not empty)
  - `WP_SITEURL` and `WP_HOME` — match domain
  - `FORCE_SSL_ADMIN` — should be true

### 6. Security Checks
- XML-RPC: `wp option get xmlrpc_enabled 2>/dev/null` or check if xmlrpc.php accessible
- REST API: check if user enumeration possible via `/wp-json/wp/v2/users`
- File editor: verify DISALLOW_FILE_EDIT
- .htaccess security rules (if Apache)
- Directory listing disabled
- wp-includes/wp-admin protection

### 7. Database
- DB prefix: `wp db prefix`
- DB size: `wp db size --format=csv`
- Check for orphaned tables
- Transients cleanup needed: `wp transient list --format=count 2>/dev/null`

### 8. Source Code Review (via SSH file access)
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

### 9. Performance Checks
- Object cache: `wp cache type 2>/dev/null`
- Autoloaded options size: `wp db query "SELECT SUM(LENGTH(option_value)) as autoload_size FROM $(wp db prefix)options WHERE autoload='yes'" --allow-root`
- Cron events: `wp cron event list --format=csv`
- Check for excessive cron jobs

### 10. .htaccess / Nginx Config Review
- Read .htaccess: `cat WP_PATH/.htaccess`
- Check security headers (X-Frame-Options, X-Content-Type-Options, etc.)
- Check redirect rules
- Check if wp-admin/wp-login has IP restriction or rate limiting

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
- Use WP-CLI with --allow-root when SSH user is root
- Always specify --path= for WP-CLI commands
- Research EVERY active plugin for known vulnerabilities via WebSearch
- Check plugin last updated date — plugins not updated in 2+ years are risky
- Look for malware indicators in source code
- Be specific — provide exact WP-CLI commands to fix issues
