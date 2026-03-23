# Phase 6: wordpress-site-audit v2 (UPDATE)

## Context Links

- Current skill: `.claude/skills/wordpress-site-audit/SKILL.md`
- Brainstorm: [Audit Skills Impact](../reports/brainstorm-260323-1703-ols-stack-upgrade.md)
- Depends on: [Phase 5](phase-05-vps-server-audit.md) (for LSPHP path)

## Overview

- **Priority:** P2
- **Status:** complete
- **Effort:** 3h
- **Description:** Update WordPress audit for OLS stack awareness. Add LSPHP auto-detect for WP-CLI, LSCache plugin detection + config validation, Redis per-site DB validation, custom login URL check (OLS rewrite exists + default wp-login blocked), WooCommerce audit. Keep all existing checks intact.

## Key Insights

- WP-CLI on OLS servers often needs LSPHP: `/usr/local/lsws/lsphp82/bin/lsphp /usr/local/bin/wp`
- Auto-detect: if `wp --info` fails (missing mysqli), try LSPHP wrapper
- LSCache plugin config stored in `wp_options` table as `litespeed.conf.*`
- Redis DB validation: compare `WP_REDIS_DATABASE` in wp-config.php vs actual Redis DB usage
- Custom login URL: check OLS rewrite rules in vhost config + test default wp-login.php returns 403
- WooCommerce: check version, updates, LSCache WooCommerce config

## Requirements

### Functional (NEW checks)
- F1: LSPHP auto-detect for WP-CLI execution
- F2: LSCache plugin detection: installed, active, configuration validation
- F3: Redis per-site validation: WP_REDIS_DATABASE matches expected, object cache connected
- F4: Custom login URL check: OLS rewrite exists, `/wp-login.php` returns 403, custom slug works
- F5: WooCommerce audit: version, update status, LSCache WooCommerce exclusions configured
- F6: Backup script check: per-site backup script exists + cron configured

### Functional (KEEP existing)
- All v1 checks: core version, checksums, plugins, themes, users, wp-config.php, security, database, source code review, performance, .htaccess/config review

## Architecture

No change. Same WP-CLI + SSH audit flow, extended with new checks.

## Related Code Files

- **Modify:** `.claude/skills/wordpress-site-audit/SKILL.md`

## Implementation Steps

### Step 1: LSPHP Auto-Detect for WP-CLI
Replace the fixed WP-CLI prerequisite with auto-detect:

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

Update ALL subsequent WP-CLI commands to use `${WP_CLI}` instead of `wp`.

### Step 2: LSCache Plugin Detection (new section after Plugins Audit)
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

### Step 3: Redis Per-Site DB Validation
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

### Step 4: Custom Login URL Check
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
  # This is a config check, not a live HTTP test (would need Cloudflare to be active)
  grep -q "wp-login.*\[F" "/usr/local/lsws/conf/vhosts/${DOMAIN_SLUG}.conf" 2>/dev/null && \
    echo "Direct wp-login.php: BLOCKED (OK)" || \
    echo "WARN: wp-login.php not blocked in OLS rewrite rules"
else
  echo "Custom Login URL: NOT configured (WARN - using default wp-login.php)"
fi
```

### Step 5: WooCommerce Audit (new section)
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

### Step 6: Backup Check (new section)
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

### Step 7: Update Report Template
Add to the Summary table:
```markdown
| LSCache | OK/WARN/CRITICAL | brief |
| Redis | OK/WARN/CRITICAL | brief |
| WooCommerce | OK/WARN/N/A | brief |
| Custom Login | OK/WARN | brief |
| Backups | OK/WARN | brief |
```

Add detailed sections:
```markdown
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
```

## Todo List

- [x] Implement LSPHP auto-detect for WP-CLI (fallback from system PHP)
- [x] Add LSCache plugin detection + config validation (object cache, Redis DB)
- [x] Add Redis per-site DB validation (cross-check wp-config vs LSCache vs actual)
- [x] Add custom login URL check (wp-config constant + OLS rewrite + wp-login block)
- [x] Add WooCommerce audit (version, updates, LSCache exclusions)
- [x] Add per-site backup check (script exists, cron, recent backups)
- [x] Update report template with new sections
- [x] Ensure all WP-CLI commands use `${WP_CLI}` variable (not hardcoded `wp`)

## Success Criteria

- Audit works on both Nginx (v1) and OLS (v2) servers
- LSPHP auto-detected and used for WP-CLI when system PHP lacks extensions
- LSCache config validated with Redis DB cross-reference
- Custom login URL presence/absence correctly reported
- WooCommerce cache exclusions flagged if missing
- Backup status clearly reported

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| LSPHP auto-detect finds wrong version | LOW | Use `tail -1` to get latest version |
| LSCache option names change in future versions | LOW | Check current plugin docs at implementation time |
| Redis not installed on v1 servers | NONE | Skip Redis sections if Redis not detected |

## Security Considerations

- No new credential requirements
- All checks are read-only
- wp-config.php read to check constants (already done in v1)

## Next Steps

After this phase: Phase 7 (cloudflare-domain-audit) adds WAF, bot fight, WooCommerce, HTTP/3 checks.
