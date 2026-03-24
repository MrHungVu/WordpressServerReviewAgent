---
name: domain-review-agent
description: Orchestrating agent that runs vps-server-audit, cloudflare-domain-audit, and wordpress-site-audit skills to produce a full domain review. Collects credentials once, runs VPS + Cloudflare audits in parallel, then WordPress audit, and produces a combined final report with OLS-specific cross-validations (Redis DB isolation, LSAPI_CHILDREN RAM budget, LSPHP+WP compatibility). Use when user says "review domain", "check domain", "audit domain", or wants full site verification.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
  - Skill
---

# Domain Review Agent

## Purpose
Orchestrate all 3 audit skills to produce a comprehensive domain review. This is the main entry point users interact with.

## Trigger Phrases

### Audit-Only Mode (read-only, produce report)
- "review domain [domain]"
- "check domain [domain]"
- "audit [domain]"
- "verify [domain] setup"
- "is [domain] configured correctly?"

### Check-and-Fix Mode (audit + auto-fix all issues)
- "check setup domain [domain]"
- "check setup domain [domain] custom_login_slug [slug]"
- "fix domain [domain]"

**Check-and-Fix Mode** runs all 3 audits, then **automatically fixes every issue found** via SSH. See [Step 6: Auto-Fix](#step-6-auto-fix-check-and-fix-mode-only) for details.

If `custom_login_slug` is provided, the agent ensures the mu-plugin is installed with that slug. If not provided, it reads the existing `CUSTOM_LOGIN_SLUG` from wp-config.php or generates a random `login-{hex4}`.

## Step 1: Collect Credentials
Ask user for ALL of these (use AskUserQuestion tool). NEVER save any of these to files:

### Required
1. **Domain name** (e.g., example.com)
2. **SSH host** (IP address)
3. **SSH user** (typically root)
4. **SSH password**
5. **SSH port** (default: 22)
6. **Cloudflare email**
7. **Cloudflare Global API Key** (or API Token)

### Optional
8. **WordPress install path** (auto-detected if not provided)
9. **WordPress admin URL** (for reference only)
10. **Custom login slug** (for check-and-fix mode — e.g., `my-admin`)

## Step 2: Verify Connectivity
Before running audits, verify:
```bash
# Test SSH connection
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p PORT USER@HOST 'echo "SSH OK"'

# Test Cloudflare API
curl -s "https://api.cloudflare.com/client/v4/user" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.success'
```

If either fails, inform user and ask to verify credentials.

## Step 3: Run Audits (Parallel where possible)
Execute the 3 skills. VPS audit and Cloudflare audit can run in PARALLEL since they're independent. WordPress audit runs AFTER VPS audit (needs SSH + WP path from VPS findings).

### Parallel Phase (Skills 1 & 2)
1. **Activate `vps-server-audit` skill** — pass SSH credentials and domain
2. **Activate `cloudflare-domain-audit` skill** — pass Cloudflare credentials and domain

### Sequential Phase (Skill 3, after VPS audit completes)
3. **Activate `wordpress-site-audit` skill** — pass SSH credentials, domain, and WordPress path discovered during VPS audit

## Step 4: Cross-Reference Findings
After all 3 audits complete, cross-reference:

### Existing Cross-Reference Checks
- **DNS ↔ Server**: Cloudflare A record IP matches actual VPS IP
- **SSL ↔ Cloudflare**: Cloudflare SSL mode matches server SSL config (Full Strict needs valid origin cert)
- **Cloudflare cache ↔ WordPress**: Page rules bypass cache for wp-admin
- **Server PHP ↔ WordPress**: PHP version compatible with WP + plugins
- **Firewall ↔ Cloudflare**: Server firewall allows Cloudflare IPs

### New OLS-Specific Cross-Reference Checks

#### OLS SSL Cert Path ↔ Cloudflare Origin Cert
- VPS audit: check `/usr/local/lsws/conf/cert/{domain}.crt` exists
- CF audit: origin cert deployed with Full Strict mode
- Cross-check: cert file exists AND SSL mode is strict
- If OLS has cert but CF SSL not strict → WARN: downgrade risk
- If CF is strict but no cert on OLS → CRITICAL: SSL will fail

#### CF-Connecting-IP ↔ Cloudflare Proxy Status
- VPS audit: `useClientIpInHeader` setting in `httpd_config.conf`
- CF audit: domain proxied (orange cloud) on DNS records
- Cross-check: if proxied AND `useClientIpInHeader` missing → WARN: logs show CF IPs instead of real visitor IPs
- If not proxied → skip this check

#### LSPHP Version ↔ WordPress Compatibility
- VPS audit: LSPHP version detected
- WP audit: WordPress version + active plugin requirements
- Cross-check:
  - PHP 8.2 supports WP 6.2+
  - Check if any plugin warns about PHP version incompatibility
  - Flag if LSPHP < 8.1 (no JIT, reduced WP version support)

#### Redis DB Assignment Validation (No Duplicates Across Sites)
- WP audit (per site): `WP_REDIS_DATABASE` value from `wp-config.php`
- VPS audit: `redis-cli` per-DB key counts
- Cross-check:
  - No two sites share same Redis DB number
  - All DB numbers within valid range 0-15
  - DB numbers with keys match sites that have Redis configured
  - Any DB with keys but no matching site → WARN: orphan cache (stale data)

#### LSAPI_CHILDREN RAM Budget Validation
- VPS audit: total RAM, `LSAPI_CHILDREN` per ext app config
- Formula:
  - `total_workers = sum(LSAPI_CHILDREN per site)`
  - `estimated_php_mem = total_workers × 50MB` (average WP worker)
  - `available_ram = total_ram - OS(500MB) - MariaDB(innodb_pool) - Redis(maxmemory) - OPcache - OLS(200MB)`
- Cross-check:
  - `estimated_php_mem < available_ram` → OK
  - `estimated_php_mem >= available_ram` → WARN: memory overcommit risk, suggest reducing `LSAPI_CHILDREN` or upgrading RAM

#### Backup Coverage Validation
- WP audit (per site): backup script exists + cron configured + recent backup files
- Cross-check:
  - Every detected WordPress installation has a backup script
  - Every backup cron ran within last 48 hours
  - Any site without backup script or overdue cron → WARN

## Step 5: Generate Final Report
Save to: `PROJECT_ROOT/vps-reports/domain-review-DOMAIN-YYYYMMDD.md`

```markdown
# Full Domain Review Report
**Domain:** example.com
**Date:** YYYY-MM-DD HH:MM
**Overall Health Score:** X/10

## Executive Summary
Brief 3-5 sentence overview of domain health across all 3 layers.

## Health Dashboard
| Layer | Status | Critical | Warnings | Score |
|-------|--------|----------|----------|-------|
| VPS Server | HEALTHY/ATTENTION/CRITICAL | count | count | X/10 |
| Cloudflare | HEALTHY/ATTENTION/CRITICAL | count | count | X/10 |
| WordPress | HEALTHY/ATTENTION/CRITICAL | count | count | X/10 |

## Cross-Reference Issues
Issues found by comparing findings across layers:

| Check | VPS | Cloudflare | WordPress | Status |
|-------|-----|------------|-----------|--------|
| DNS A Record | {ip} | {ip} | - | OK/MISMATCH |
| SSL Mode | cert at OLS path | Full Strict | FORCE_SSL_ADMIN | OK/WARN |
| HTTPS | OLS listener 443 | Always HTTPS | siteurl=https | OK/WARN |
| CF-Connecting-IP | configured | proxied | - | OK/WARN |
| LSPHP ↔ WP | LSPHP 8.2 | - | WP 6.X | OK/WARN |
| Redis Isolation | DB 0,1,2... | - | WP_REDIS_DATABASE | OK/WARN |
| Memory Budget | {ram}MB avail | - | {workers}x50MB est | OK/WARN |
| Backup Coverage | scripts exist | - | per-site cron | OK/WARN |

## Critical Actions (Fix Immediately)
Priority-ordered list from ALL 3 audits:
1. [ ] [LAYER] issue — fix command
2. [ ] [LAYER] issue — fix command

## Warnings (Fix Soon)
1. [ ] [LAYER] issue — fix
2. [ ] [LAYER] issue — fix

## Recommendations (Improve)
1. [ ] [LAYER] suggestion
2. [ ] [LAYER] suggestion

## Individual Reports
- [VPS Server Audit](./vps-audit-DOMAIN-YYYYMMDD.md)
- [Cloudflare Audit](./cloudflare-audit-DOMAIN-YYYYMMDD.md)
- [WordPress Audit](./wordpress-audit-DOMAIN-YYYYMMDD.md)
```

## Step 6: Auto-Fix (Check-and-Fix Mode Only)

**Only runs when triggered by "check setup domain" or "fix domain" phrases.** Skipped in audit-only mode.

After Steps 3-5 (audit + report), iterate through ALL issues found (Critical + Warnings) and fix them automatically via SSH and Cloudflare API. Process in this order:

### 6a. VPS Server Fixes (via SSH)
For each VPS issue, run the fix command from the audit report. Common auto-fixes:
- **Missing custom login mu-plugin:** Install `custom-login-url.php` mu-plugin (from `wordpress-site-setup` Step 8). Use the `custom_login_slug` provided by user, or read existing `CUSTOM_LOGIN_SLUG` from wp-config.php, or generate `login-$(openssl rand -hex 4)`.
- **CUSTOM_LOGIN_SLUG not in wp-config.php:** Set it via WP-CLI: `wp config set CUSTOM_LOGIN_SLUG '{slug}' --path={WP_PATH} --allow-root`
- **File permissions wrong:** `find {path} -type f -exec chmod 644 {} \;` + `find {path} -type d -exec chmod 755 {} \;` + `chmod 600 wp-config.php`
- **wp-config.php missing security constants:** Set DISALLOW_FILE_EDIT, FORCE_SSL_ADMIN, WP_DEBUG=false via WP-CLI
- **Missing backup script/cron:** Create per-site backup script + .my.cnf + staggered cron (from `wordpress-site-setup` Step 15)
- **Redis not configured in LSCache:** Set litespeed.conf.object_kind=1, object_host, object_port, object_db_id via WP-CLI
- **OLS config validation failure:** Run `openlitespeed -t`, show error, attempt fix
- **fail2ban not running:** `systemctl enable fail2ban && systemctl restart fail2ban`
- **UFW not CF-only:** Re-apply Cloudflare-only rules (from `vps-server-setup` Step 15)
- **Outdated WordPress/plugins:** `wp core update` + `wp plugin update --all`

### 6b. Cloudflare Fixes (via API)
- **SSL not Full Strict:** `PATCH /zones/{id}/settings/ssl` → `{"value":"strict"}`
- **Always HTTPS off:** `PATCH /zones/{id}/settings/always_use_https` → `{"value":"on"}`
- **HSTS not enabled:** `PATCH /zones/{id}/settings/security_header` → enable
- **HTTP/3 off:** `PATCH /zones/{id}/settings/http3` → `{"value":"on"}`
- **Bot Fight Mode off:** Enable via API
- **Missing WooCommerce page rules:** Create cart/checkout/my-account bypass rules
- **Missing rate limiting:** Create wp-login.php + xmlrpc.php rules
- **Brotli off:** Enable via API
- **TLS < 1.2:** Set min_tls_version to 1.2

### 6c. Verify Fixes
After all fixes applied, run a quick verification:
```bash
# Verify mu-plugin active
${WP_CLI} eval 'echo defined("CUSTOM_LOGIN_SLUG") ? CUSTOM_LOGIN_SLUG : "NOT_DEFINED";' --path=${WP_PATH} --allow-root

# Test login URL protection
curl -sk -o /dev/null -w "%{http_code}" https://${DOMAIN}/wp-login.php  # Should be 404
curl -sk -o /dev/null -w "%{http_code}" https://${DOMAIN}/wp-admin/    # Should be 302 to homepage
curl -sk -o /dev/null -w "%{http_code}" https://${DOMAIN}/${LOGIN_SLUG} # Should be 200

# OLS config valid
/usr/local/lsws/bin/openlitespeed -t

# Cloudflare SSL check
curl -sI https://${DOMAIN} | grep -i "cf-ray\|strict-transport"
```

### 6d. Report Fixes Applied
Append to the report:
```
## Auto-Fix Results
| # | Issue | Layer | Fix Applied | Result |
|---|-------|-------|-------------|--------|
| 1 | Missing mu-plugin | WP | Installed custom-login-url.php | OK |
| 2 | SSL not strict | CF | Set Full Strict | OK |
| ... | ... | ... | ... | ... |

Login URL: https://{domain}/{slug}
wp-login.php: returns 404
wp-admin/: redirects to homepage (unauthenticated)
```

## Important Rules
- NEVER save credentials anywhere — keep only in conversation context
- Credentials are single-session only
- If a skill/audit fails, still produce report with available data
- Health score: 10 = perfect, deduct 2 per critical, 1 per warning
- Minimum score is 1 (never 0)
- Always cross-reference between layers — that's the value-add
- Present fixes in priority order across all layers
- Make all fix commands copy-paste ready
- **Check-and-fix mode:** Auto-fix ALL issues found — do not ask for confirmation per fix. The user triggered this mode knowing fixes will be applied.
- **Always verify** login URL protection after mu-plugin install (curl test for 404/302/200)
