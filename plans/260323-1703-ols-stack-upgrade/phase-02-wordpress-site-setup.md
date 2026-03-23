# Phase 2: wordpress-site-setup v2 (MAJOR REWRITE)

## Context Links

- Current skill: `.claude/skills/wordpress-site-setup/SKILL.md`
- Research: [OLS Production](../reports/researcher-260323-1708-openlitespeed-wordpress-production.md)
- Research: [Multi-Site Best Practices](../reports/researcher-260323-1708-multi-site-openlitespeed-best-practices.md)
- Depends on: [Phase 1](phase-01-vps-server-setup.md)

## Overview

- **Priority:** P1 (called N times for N sites on same VPS)
- **Status:** complete
- **Effort:** 10h
- **Description:** Complete rewrite for OLS multi-site. Creates per-site: directory, MariaDB DB+user, OLS vhost with separate LSPHP worker pool, Redis DB assignment, LSCache plugin, WooCommerce, custom wp-admin URL, per-site backup script. Designed to be called multiple times on the same server.

## Key Insights

- Each call creates 1 WordPress site; orchestrator calls N times for N sites
- `site_number` parameter determines Redis DB# (0, 1, 2...) and staggered cron times
- LSAPI_CHILDREN calculated from RAM profile + num_sites (passed by orchestrator)
- OLS vhost registered in httpd_config.conf via append (not rewrite)
- Per-site php.ini via `PHP_INI_SCAN_DIR` env var on ext app
- WP-CLI uses LSPHP binary: `/usr/local/lsws/lsphp82/bin/lsphp /usr/local/bin/wp`
- Custom login URL: OLS rewrite `/{slug}` -> `/wp-login.php`, block direct `/wp-login.php`

## Requirements

### Functional
- F1: Create `/var/www/{domain}/html/`, `logs/`, `tmp/`
- F2: Create MariaDB DB + user per site with minimal privileges
- F3: Install WP-CLI using LSPHP binary (if missing)
- F4: Download + install WordPress via WP-CLI
- F5: Generate wp-config.php with: random prefix, security constants, Redis config, custom login URL
- F6: Create OLS vhost config with separate LSPHP ext app (socket per site)
- F7: Register vhost in httpd_config.conf (listener mapping + vhostMap + ext app)
- F8: Create per-site php.ini with adaptive memory_limit
- F9: Install + activate LiteSpeed Cache plugin, configure Redis object cache
- F10: Install + activate WooCommerce, configure LSCache WooCommerce exclusions
- F11: Set file permissions (644/755/600), set ownership www-data
- F12: Delete default WP content, disable pingbacks, set permalinks
- F13: Create per-site backup script + cron (staggered)
- F14: Verify: WP-CLI health, curl test, `openlitespeed -t`, graceful restart
- F15: Output all generated credentials to orchestrator

### Non-Functional
- Idempotent: check-before-act on every step
- Called N times safely (append, not overwrite main config)
- All commands via sshpass SSH
- No credentials saved except wp-config.php (required)

## Architecture

```
Per-Site Layout:
/var/www/{domain}/
├── html/              # WordPress root (DocumentRoot)
│   ├── wp-config.php  # DB creds + Redis config + custom login
│   └── ...
├── logs/              # Per-site access + error logs
└── tmp/               # PHP upload temp

OLS Config:
/usr/local/lsws/conf/
├── httpd_config.conf  # Appended: extAppServer + vhostMap + include
├── vhosts/{domain}.conf   # Per-site vhost
├── php-ini/{domain-slug}/php.ini  # Per-site PHP config
└── cert/{domain}.crt + .key       # SSL cert (from Phase 3)
```

## Related Code Files

- **Modify:** `.claude/skills/wordpress-site-setup/SKILL.md` (complete rewrite)

## Implementation Steps

### Step 0: SKILL.md Frontmatter
```yaml
---
name: wordpress-site-setup
description: Install WordPress on OLS VPS with WP-CLI (via LSPHP), create DB, OLS vhost with isolated LSPHP worker pool, Redis object cache, LiteSpeed Cache plugin, WooCommerce, custom wp-admin URL, per-site backup. Multi-site aware -- safe to call N times. Idempotent.
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

### Step 1: Required Input
```
- SSH host, user, password, port
- Domain name
- MariaDB root password (from Phase 1)
- WordPress admin email
- site_number (0, 1, 2... for Redis DB assignment)
- RAM profile (2GB/4GB/8GB+) and LSAPI_CHILDREN (from orchestrator)
- Custom login URL slug (e.g., "my-secret-login")
- LSPHP path (default: /usr/local/lsws/lsphp82/bin/lsphp)
```

### Step 2: Create Site Directory
```bash
mkdir -p /var/www/${DOMAIN}/html
mkdir -p /var/www/${DOMAIN}/logs
mkdir -p /var/www/${DOMAIN}/tmp
chown -R www-data:www-data /var/www/${DOMAIN}
```

### Step 3: Generate Credentials
```bash
DOMAIN_SLUG=$(echo ${DOMAIN} | tr '.-' '__' | cut -c1-10)
DB_NAME="wp_${DOMAIN_SLUG}_$(openssl rand -hex 2)"
DB_USER="wp_usr_$(openssl rand -hex 3)"
DB_PASS=$(openssl rand -base64 24)
WP_ADMIN_USER="wp_admin_$(openssl rand -hex 2)"
WP_ADMIN_PASS=$(openssl rand -base64 18)
WP_PREFIX="wp_$(openssl rand -hex 2)_"
```

### Step 4: Create Database
```bash
mysql -u root -p"${MARIADB_ROOT_PASS}" <<SQLEOF
CREATE DATABASE IF NOT EXISTS \`${DB_NAME}\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER,
      CREATE TEMPORARY TABLES, LOCK TABLES, REFERENCES
      ON \`${DB_NAME}\`.* TO '${DB_USER}'@'localhost';
FLUSH PRIVILEGES;
SQLEOF
```

### Step 5: Install WP-CLI (if missing)
```bash
if ! command -v wp &>/dev/null; then
  curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
  chmod +x wp-cli.phar
  mv wp-cli.phar /usr/local/bin/wp
fi
```
**WP-CLI via LSPHP wrapper:** Create `/usr/local/bin/wp-lsphp` if system PHP lacks mysqli:
```bash
if ! php -m 2>/dev/null | grep -q mysqli; then
  cat > /usr/local/bin/wp-lsphp << 'WPCLIEOF'
#!/bin/bash
/usr/local/lsws/lsphp82/bin/lsphp /usr/local/bin/wp "$@"
WPCLIEOF
  chmod +x /usr/local/bin/wp-lsphp
  WP_CLI="/usr/local/bin/wp-lsphp"
else
  WP_CLI="wp"
fi
```

### Step 6: Download + Install WordPress
```bash
# Check if already exists
if [ ! -f /var/www/${DOMAIN}/html/wp-config.php ]; then
  ${WP_CLI} core download --path=/var/www/${DOMAIN}/html --allow-root
fi
```

### Step 7: Create wp-config.php
```bash
${WP_CLI} config create \
  --path=/var/www/${DOMAIN}/html \
  --dbname="${DB_NAME}" \
  --dbuser="${DB_USER}" \
  --dbpass="${DB_PASS}" \
  --dbprefix="${WP_PREFIX}" \
  --allow-root
```

Set security + Redis + custom login constants:
```bash
WP_PATH="/var/www/${DOMAIN}/html"
${WP_CLI} config set DISALLOW_FILE_EDIT true --raw --path=${WP_PATH} --allow-root
${WP_CLI} config set WP_DEBUG false --raw --path=${WP_PATH} --allow-root
${WP_CLI} config set WP_POST_REVISIONS 5 --raw --path=${WP_PATH} --allow-root
${WP_CLI} config set FORCE_SSL_ADMIN true --raw --path=${WP_PATH} --allow-root
${WP_CLI} config set WP_AUTO_UPDATE_CORE true --raw --path=${WP_PATH} --allow-root
${WP_CLI} config set AUTOMATIC_UPDATER_DISABLED false --raw --path=${WP_PATH} --allow-root

# Redis config
${WP_CLI} config set WP_REDIS_HOST '127.0.0.1' --path=${WP_PATH} --allow-root
${WP_CLI} config set WP_REDIS_PORT 6379 --raw --path=${WP_PATH} --allow-root
${WP_CLI} config set WP_REDIS_DATABASE ${SITE_NUMBER} --raw --path=${WP_PATH} --allow-root
${WP_CLI} config set WP_REDIS_PREFIX "${DOMAIN_SLUG}:" --path=${WP_PATH} --allow-root

# Custom login URL
${WP_CLI} config set JEsuspended_LOGIN_SLUG "${LOGIN_SLUG}" --path=${WP_PATH} --allow-root
```

### Step 8: Install WordPress Core
```bash
${WP_CLI} core install \
  --url="https://${DOMAIN}" \
  --title="${DOMAIN}" \
  --admin_user="${WP_ADMIN_USER}" \
  --admin_password="${WP_ADMIN_PASS}" \
  --admin_email="${WP_ADMIN_EMAIL}" \
  --path=${WP_PATH} \
  --allow-root
```

### Step 9: Create OLS Vhost Config
Write `/usr/local/lsws/conf/vhosts/${DOMAIN}.conf`:
```
virtualHost ${DOMAIN_SLUG} {
  vhRoot                  /var/www/${DOMAIN}/html
  allowSymbolLink         1
  enableScript            1
  restrained              0

  errorlog /var/www/${DOMAIN}/logs/error.log {
    useServer             0
    logLevel              WARN
    rollingSize           10M
  }

  accesslog /var/www/${DOMAIN}/logs/access.log {
    useServer             0
    logFormat             "%h %l %u %t \"%r\" %>s %b"
    rollingSize           10M
  }

  # SSL
  vhssl {
    keyFile               /usr/local/lsws/conf/cert/${DOMAIN}.key
    certFile              /usr/local/lsws/conf/cert/${DOMAIN}.crt
  }

  # Rewrite rules
  rewrite {
    enable                1
    autoLoadHtaccess      1
    rules                 <<<END_rules
# Custom login URL
RewriteRule ^/${LOGIN_SLUG}$ /wp-login.php [QSA,L]
# Block direct wp-login.php access (except POST for login form submission)
RewriteCond %{THE_REQUEST} /wp-login\.php
RewriteCond %{REQUEST_URI} ^/wp-login\.php
RewriteCond %{QUERY_STRING} !^action=(logout|rp|resetpass|postpass)
RewriteCond %{HTTP_REFERER} !/${LOGIN_SLUG}
RewriteRule ^wp-login\.php$ - [F,L]
# Block xmlrpc.php
RewriteRule ^xmlrpc\.php$ - [F,L]
# Block wp-includes PHP execution
RewriteRule ^wp-includes/.*\.php$ - [F,L]
# WordPress permalinks
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
    END_rules
  }

  # Static content context
  context /wp-content/ {
    location              /var/www/${DOMAIN}/html/wp-content/
    allowBrowse           1
    enableExpires         1
    expiresByType         image/*=A31536000, text/css=A2592000, application/javascript=A2592000
    extraHeaders          set Cache-Control "public, no-transform"
  }

  # PHP context
  context / {
    location              /var/www/${DOMAIN}/html/
    allowBrowse           1

    rewrite {
      enable              1
      inherit             1
    }
  }
}
```

### Step 10: Create Per-Site PHP.ini
```bash
mkdir -p /usr/local/lsws/conf/php-ini/${DOMAIN_SLUG}
```
Write `/usr/local/lsws/conf/php-ini/${DOMAIN_SLUG}/php.ini`:
```ini
memory_limit = 256M
upload_max_filesize = 100M
post_max_size = 100M
max_execution_time = 300
max_input_time = 300
```
Adjust `memory_limit` based on RAM profile: 96M (2GB), 128M (4GB), 256M (8GB+).

### Step 11: Register Vhost in httpd_config.conf (STAGING APPROACH)
<!-- Updated: Validation Session 1 - Staging config pattern to protect running sites -->
**CRITICAL:** Never write directly to production config. Use staging file + validate + atomic swap.

```bash
# 1. Create staging copy
cp /usr/local/lsws/conf/httpd_config.conf /usr/local/lsws/conf/httpd_config.conf.staging

# 2. Append to STAGING copy (not production)
cat >> /usr/local/lsws/conf/httpd_config.conf.staging << OLSEOF

extprocessor lsphp82-${DOMAIN_SLUG} {
  type                    lsapi
  address                 uds://tmp/lshttpd/lsphp82.${DOMAIN_SLUG}.sock
  maxConns                ${LSAPI_CHILDREN}
  env                     LSAPI_CHILDREN=${LSAPI_CHILDREN}
  env                     PHP_INI_SCAN_DIR=/usr/local/lsws/conf/php-ini/${DOMAIN_SLUG}/
  initTimeout             60
  retryTimeout            0
  respBuffer              0
  autoStart               2
  path                    /usr/local/lsws/lsphp82/bin/lsphp
  backlog                 100
  instances               1
  priority                0
  memSoftLimit            ${PHP_MEM_SOFT}
  memHardLimit            ${PHP_MEM_HARD}
  procSoftLimit           1000
  procHardLimit           1200
}

OLSEOF

# 3. Append vhostMap entries to STAGING
sed -i "/^vhostMap Default {/a\\  ${DOMAIN}    ${DOMAIN_SLUG}\\n  www.${DOMAIN}    ${DOMAIN_SLUG}" /usr/local/lsws/conf/httpd_config.conf.staging
sed -i "/^vhostMap HTTPS {/a\\  ${DOMAIN}    ${DOMAIN_SLUG}\\n  www.${DOMAIN}    ${DOMAIN_SLUG}" /usr/local/lsws/conf/httpd_config.conf.staging

# 4. Append vhost include to STAGING
echo "include /usr/local/lsws/conf/vhosts/${DOMAIN}.conf" >> /usr/local/lsws/conf/httpd_config.conf.staging

# 5. VALIDATE staging config (MUST pass before swap)
/usr/local/lsws/bin/openlitespeed -t -f /usr/local/lsws/conf/httpd_config.conf.staging
if [ $? -ne 0 ]; then
  echo "ERROR: OLS config validation failed for ${DOMAIN}. Staging config rejected."
  rm -f /usr/local/lsws/conf/httpd_config.conf.staging
  echo "Production config UNTOUCHED. Existing sites still running."
  exit 1
fi

# 6. ATOMIC SWAP: backup production, promote staging
cp /usr/local/lsws/conf/httpd_config.conf /usr/local/lsws/conf/httpd_config.conf.bak
mv /usr/local/lsws/conf/httpd_config.conf.staging /usr/local/lsws/conf/httpd_config.conf

# 7. Graceful restart (zero downtime)
/usr/local/lsws/bin/lswsctrl restart
```

### Step 12: Install + Configure LSCache Plugin
```bash
${WP_CLI} plugin install litespeed-cache --activate --path=${WP_PATH} --allow-root

# Configure Redis object cache
${WP_CLI} option update litespeed.conf.object_kind 1 --path=${WP_PATH} --allow-root  # 1=Redis
${WP_CLI} option update litespeed.conf.object_host '127.0.0.1' --path=${WP_PATH} --allow-root
${WP_CLI} option update litespeed.conf.object_port 6379 --path=${WP_PATH} --allow-root
${WP_CLI} option update litespeed.conf.object_db_id ${SITE_NUMBER} --path=${WP_PATH} --allow-root
${WP_CLI} option update litespeed.conf.object_persistent 1 --path=${WP_PATH} --allow-root
```

### Step 13: Install + Configure WooCommerce
```bash
${WP_CLI} plugin install woocommerce --activate --path=${WP_PATH} --allow-root

# LSCache WooCommerce exclusions (auto-detected by plugin, but ensure)
${WP_CLI} option update litespeed.conf.cdn_exc '/cart/*
/checkout/*
/my-account/*' --path=${WP_PATH} --allow-root
```

### Step 14: Security Hardening
```bash
# File permissions
find /var/www/${DOMAIN}/html -type f -exec chmod 644 {} \;
find /var/www/${DOMAIN}/html -type d -exec chmod 755 {} \;
chmod 600 /var/www/${DOMAIN}/html/wp-config.php

# Ownership
chown -R www-data:www-data /var/www/${DOMAIN}

# Delete default content
${WP_CLI} post delete 1 --force --path=${WP_PATH} --allow-root 2>/dev/null
${WP_CLI} post delete 2 --force --path=${WP_PATH} --allow-root 2>/dev/null
${WP_CLI} plugin delete hello --path=${WP_PATH} --allow-root 2>/dev/null
${WP_CLI} plugin delete akismet --path=${WP_PATH} --allow-root 2>/dev/null
${WP_CLI} theme delete twentytwentythree --path=${WP_PATH} --allow-root 2>/dev/null
${WP_CLI} theme delete twentytwentytwo --path=${WP_PATH} --allow-root 2>/dev/null

# Permalinks + disable pingbacks
${WP_CLI} rewrite structure '/%postname%/' --path=${WP_PATH} --allow-root
${WP_CLI} option update default_pingback_flag 0 --path=${WP_PATH} --allow-root
```

### Step 15: Per-Site Backup Script
<!-- Updated: Validation Session 1 - Use .my.cnf instead of plaintext password in script -->

**First, create per-site .my.cnf for mysqldump credentials:**
```bash
cat > /var/www/${DOMAIN}/.my.cnf << MYCNFEOF
[mysqldump]
user=${DB_USER}
password=${DB_PASS}
MYCNFEOF
chmod 600 /var/www/${DOMAIN}/.my.cnf
chown www-data:www-data /var/www/${DOMAIN}/.my.cnf
```

Write `/usr/local/bin/backup-${DOMAIN_SLUG}.sh`:
```bash
#!/bin/bash
SITE="${DOMAIN}"
BACKUP_DIR="/var/backups/wordpress"
DATE=$(date +%Y%m%d-%H%M%S)
DB_NAME="${DB_NAME}"
MY_CNF="/var/www/${SITE}/.my.cnf"

# Database backup (credentials from .my.cnf, no plaintext password)
mysqldump --defaults-file="${MY_CNF}" ${DB_NAME} | gzip > "${BACKUP_DIR}/${SITE}_db_${DATE}.sql.gz"

# Files backup (exclude cache)
tar --exclude='wp-content/cache' --exclude='wp-content/litespeed' \
  -czf "${BACKUP_DIR}/${SITE}_files_${DATE}.tar.gz" \
  /var/www/${SITE}/html 2>/dev/null

# Retention: 30 days
find "${BACKUP_DIR}" -name "${SITE}_*" -mtime +30 -delete

echo "Backup complete: ${SITE} @ ${DATE}"
```
```bash
chmod +x /usr/local/bin/backup-${DOMAIN_SLUG}.sh
```

Add cron (staggered by site_number: site0=2:00, site1=2:30, site2=3:00...):
```bash
CRON_MINUTE=$((SITE_NUMBER * 30 % 60))
CRON_HOUR=$((2 + SITE_NUMBER * 30 / 60))
(crontab -l 2>/dev/null; echo "${CRON_MINUTE} ${CRON_HOUR} * * * /usr/local/bin/backup-${DOMAIN_SLUG}.sh >> /var/log/backup-${DOMAIN_SLUG}.log 2>&1") | crontab -
```

### Step 16: Verify
```bash
# Validate OLS config
/usr/local/lsws/bin/openlitespeed -t

# Graceful restart to pick up new vhost
/usr/local/lsws/bin/lswsctrl restart

# WP-CLI health
${WP_CLI} core version --path=${WP_PATH} --allow-root
${WP_CLI} option get siteurl --path=${WP_PATH} --allow-root
${WP_CLI} plugin list --path=${WP_PATH} --allow-root --format=table

# curl test (via Cloudflare or direct with Host header)
curl -sk -H "Host: ${DOMAIN}" https://localhost/ | head -20
```

### Step 17: Output Results
**IMPORTANT:** Output ALL information in copy-friendly format. User must be able to copy-paste this entire block and save it.

```
═══════════════════════════════════════════════════════════════
  WORDPRESS SITE SETUP COMPLETE — {DOMAIN}
  COPY AND SAVE ALL INFORMATION BELOW IMMEDIATELY
═══════════════════════════════════════════════════════════════

── LOGIN ──────────────────────────────────────────────────────
  Site URL:         https://{DOMAIN}
  Admin Login URL:  https://{DOMAIN}/{LOGIN_SLUG}
  Admin Username:   {WP_ADMIN_USER}
  Admin Password:   {WP_ADMIN_PASS}
  Admin Email:      {WP_ADMIN_EMAIL}

── DATABASE ───────────────────────────────────────────────────
  DB Host:          localhost
  DB Name:          {DB_NAME}
  DB User:          {DB_USER}
  DB Password:      {DB_PASS}
  DB Prefix:        {WP_PREFIX}

── SERVER ─────────────────────────────────────────────────────
  Install Path:     /var/www/{DOMAIN}/html
  wp-config.php:    /var/www/{DOMAIN}/html/wp-config.php
  OLS Vhost:        /usr/local/lsws/conf/vhosts/{DOMAIN}.conf
  PHP.ini:          /usr/local/lsws/conf/php-ini/{DOMAIN_SLUG}/php.ini
  LSPHP Workers:    {LSAPI_CHILDREN}
  PHP Memory:       {PHP_MEMORY_LIMIT}

── CACHING ────────────────────────────────────────────────────
  Redis DB:         {SITE_NUMBER}
  Redis Prefix:     {DOMAIN_SLUG}:
  LSCache Plugin:   active
  Object Cache:     Redis (127.0.0.1:6379 DB {SITE_NUMBER})

── WOOCOMMERCE ────────────────────────────────────────────────
  WooCommerce:      active
  Cache Excluded:   /cart/*, /checkout/*, /my-account/*

── BACKUP ─────────────────────────────────────────────────────
  Backup Script:    /usr/local/bin/backup-{DOMAIN_SLUG}.sh
  Backup Schedule:  Daily at {CRON_HOUR}:{CRON_MINUTE} UTC
  Backup Dir:       /var/backups/wordpress/
  Retention:        30 days
  DB Creds File:    /var/www/{DOMAIN}/.my.cnf

── SECURITY ───────────────────────────────────────────────────
  Custom Login:     /{LOGIN_SLUG} (default /wp-login.php blocked)
  XML-RPC:          blocked (OLS rewrite)
  File Editor:      disabled (DISALLOW_FILE_EDIT)
  SSL:              forced (FORCE_SSL_ADMIN)
  File Perms:       644 files / 755 dirs / 600 wp-config

═══════════════════════════════════════════════════════════════
```

## Todo List

- [x] Rewrite SKILL.md frontmatter + purpose
- [x] Implement site directory creation
- [x] Implement credential generation
- [x] Implement MariaDB DB+user with minimal privileges
- [x] Implement WP-CLI install via LSPHP wrapper
- [x] Implement WordPress download + install
- [x] Implement wp-config.php with Redis + security + custom login constants
- [x] Write OLS vhost config with per-site LSPHP ext app + SSL + rewrite rules
- [x] Write per-site php.ini
- [x] Implement httpd_config.conf append (extAppServer + vhostMap + include)
- [x] Implement LSCache plugin install + Redis object cache config
- [x] Implement WooCommerce install + LSCache exclusions
- [x] Implement file permissions + ownership
- [x] Implement default content cleanup + permalinks
- [x] Implement per-site backup script + staggered cron
- [x] Implement verification (openlitespeed -t + restart + curl)
- [x] Output results table with all credentials

## Success Criteria

- WordPress accessible at `https://{domain}/` (via curl with Host header)
- Custom login URL `/{slug}` loads wp-login.php; direct `/wp-login.php` returns 403
- `openlitespeed -t` passes after vhost registration
- Redis DB {site_number} has keys after first page load: `redis-cli -n {N} DBSIZE`
- LSCache plugin active: `wp plugin is-active litespeed-cache`
- WooCommerce active: `wp plugin is-active woocommerce`
- Backup script runs without error: `/usr/local/bin/backup-{slug}.sh`
- File permissions: wp-config.php = 600, files = 644, dirs = 755

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| httpd_config.conf append corrupts config | HIGH | Always `openlitespeed -t` before restart; keep backup of config |
| WP-CLI fails with LSPHP | MEDIUM | Fallback wrapper script; test with `wp-lsphp --info` |
| Custom login rewrite too restrictive | MEDIUM | Allow POST to wp-login.php from referer; test password reset flow |
| Redis DB collision between sites | LOW | Orchestrator assigns sequential numbers; never reuse |
| Backup cron overlap with high-traffic | LOW | Stagger by site_number; backups run at 2-4 AM |

## Security Considerations

- DB user gets minimal privileges (no GRANT, no SUPER, no FILE)
- wp-config.php permissions 600 (owner-only)
- Custom login URL blocks brute-force scanners at server level
- xmlrpc.php blocked via OLS rewrite
- wp-includes PHP execution blocked
- DISALLOW_FILE_EDIT prevents theme/plugin editor
- Backup script contains DB password -- set 700 permissions

## Next Steps

After this phase: Phase 3 (cloudflare-domain-setup) deploys origin cert to `/usr/local/lsws/conf/cert/{domain}.*` and configures CF zone.
