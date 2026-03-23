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

# WordPress Site Setup Skill (OLS v2)

## Purpose
Install one WordPress site on an OpenLiteSpeed VPS via SSH. Creates per-site: directory structure, MariaDB DB+user (minimal privileges), OLS vhost with isolated LSPHP worker pool, per-site php.ini, Redis object cache (per-DB), LiteSpeed Cache plugin, WooCommerce, custom wp-admin URL (direct /wp-login.php blocked), and a staggered daily backup script. Safe to call multiple times on the same server — each call appends to config, never overwrites.

## Required Input (from user/orchestrator, NEVER save to files)
- SSH host (IP or domain)
- SSH user (typically root)
- SSH password
- SSH port (default 22)
- Domain name (e.g. example.com)
- MariaDB root password (from vps-server-setup / Phase 1)
- WordPress admin email
- `site_number` — zero-based integer (0, 1, 2...). Determines Redis DB# and cron stagger. First site = 0.
- RAM profile — one of: `2GB`, `4GB`, `8GB+`
- `LSAPI_CHILDREN` — number of LSPHP worker processes per site (provided by orchestrator based on RAM/num_sites)
- Custom login URL slug (e.g. `my-secret-login` — no slashes)
- LSPHP binary path (default: `/usr/local/lsws/lsphp82/bin/lsphp`)

## SSH Connection
Use `sshpass` for password-based SSH. All remote commands run via:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```
If `sshpass` not installed locally: `sudo apt-get install -y sshpass` (or `brew install hudochenkov/sshpass/sshpass` on macOS).

## Setup Checklist

### 1. Create Site Directory Structure
Check before create (idempotent):
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  mkdir -p /var/www/{domain}/html
  mkdir -p /var/www/{domain}/logs
  mkdir -p /var/www/{domain}/tmp
  chown -R www-data:www-data /var/www/{domain}
'
```

### 2. Generate Credentials
All credentials are auto-generated and cryptographically secure. Run on the server and capture all output values:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  DOMAIN_SLUG=$(echo {domain} | tr ".-" "__" | cut -c1-10)
  DB_NAME="wp_${DOMAIN_SLUG}_$(openssl rand -hex 2)"
  DB_USER="wp_usr_$(openssl rand -hex 3)"
  DB_PASS="$(openssl rand -base64 24)"
  WP_ADMIN_USER="wp_admin_$(openssl rand -hex 2)"
  WP_ADMIN_PASS="$(openssl rand -base64 18)"
  WP_PREFIX="wp_$(openssl rand -hex 2)_"
  echo "DOMAIN_SLUG=${DOMAIN_SLUG}"
  echo "DB_NAME=${DB_NAME}"
  echo "DB_USER=${DB_USER}"
  echo "DB_PASS=${DB_PASS}"
  echo "WP_ADMIN_USER=${WP_ADMIN_USER}"
  echo "WP_ADMIN_PASS=${WP_ADMIN_PASS}"
  echo "WP_PREFIX=${WP_PREFIX}"
'
```
Capture all seven values — they are used in all subsequent steps.

### 3. Create MariaDB Database and User (minimal privileges)
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  mysql -u root -p'"'"'MARIADB_ROOT_PASS'"'"' <<SQLEOF
CREATE DATABASE IF NOT EXISTS \`${DB_NAME}\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS '"'"'${DB_USER}'"'"'@'"'"'localhost'"'"' IDENTIFIED BY '"'"'${DB_PASS}'"'"';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER,
      CREATE TEMPORARY TABLES, LOCK TABLES, REFERENCES
      ON \`${DB_NAME}\`.* TO '"'"'${DB_USER}'"'"'@'"'"'localhost'"'"';
FLUSH PRIVILEGES;
SQLEOF
'
```
**IMPORTANT:** Do NOT use `GRANT ALL PRIVILEGES`. Use only the specific grants above.

### 4. Install WP-CLI (if missing) and LSPHP Wrapper
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  # Install WP-CLI binary if missing
  if ! command -v wp &>/dev/null; then
    curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
    chmod +x wp-cli.phar
    mv wp-cli.phar /usr/local/bin/wp
  fi

  # Create LSPHP wrapper if system PHP lacks mysqli (common on OLS servers)
  if ! php -m 2>/dev/null | grep -q mysqli; then
    cat > /usr/local/bin/wp-lsphp << '"'"'WPCLIEOF'"'"'
#!/bin/bash
/usr/local/lsws/lsphp82/bin/lsphp /usr/local/bin/wp "$@"
WPCLIEOF
    chmod +x /usr/local/bin/wp-lsphp
    echo "WP_CLI=wp-lsphp"
  else
    echo "WP_CLI=wp"
  fi
'
```
Capture `WP_CLI` value from output. Use it for all subsequent WP-CLI calls. If `wp-lsphp` was created, all WP-CLI commands use `/usr/local/bin/wp-lsphp --allow-root`; otherwise use `wp --allow-root`.

### 5. Download WordPress Core
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  if [ ! -f /var/www/{domain}/html/wp-login.php ]; then
    ${WP_CLI} core download --path=/var/www/{domain}/html --allow-root
  else
    echo "WordPress files already present — skipping download"
  fi
'
```

### 6. Create wp-config.php with Redis + Security + Custom Login Constants
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  WP_PATH="/var/www/{domain}/html"

  # Create base config if not exists
  if [ ! -f ${WP_PATH}/wp-config.php ]; then
    ${WP_CLI} config create \
      --path=${WP_PATH} \
      --dbname="${DB_NAME}" \
      --dbuser="${DB_USER}" \
      --dbpass="${DB_PASS}" \
      --dbprefix="${WP_PREFIX}" \
      --allow-root
  fi

  # Security constants
  ${WP_CLI} config set DISALLOW_FILE_EDIT true  --raw --path=${WP_PATH} --allow-root
  ${WP_CLI} config set WP_DEBUG false            --raw --path=${WP_PATH} --allow-root
  ${WP_CLI} config set WP_POST_REVISIONS 5      --raw --path=${WP_PATH} --allow-root
  ${WP_CLI} config set FORCE_SSL_ADMIN true      --raw --path=${WP_PATH} --allow-root
  ${WP_CLI} config set WP_AUTO_UPDATE_CORE true  --raw --path=${WP_PATH} --allow-root
  ${WP_CLI} config set AUTOMATIC_UPDATER_DISABLED false --raw --path=${WP_PATH} --allow-root

  # Redis object cache config (one DB per site)
  ${WP_CLI} config set WP_REDIS_HOST '"'"'127.0.0.1'"'"'  --path=${WP_PATH} --allow-root
  ${WP_CLI} config set WP_REDIS_PORT 6379               --raw --path=${WP_PATH} --allow-root
  ${WP_CLI} config set WP_REDIS_DATABASE {SITE_NUMBER}  --raw --path=${WP_PATH} --allow-root
  ${WP_CLI} config set WP_REDIS_PREFIX '"'"'{DOMAIN_SLUG}:'"'"' --path=${WP_PATH} --allow-root

  # Custom login URL constant (used by OLS rewrite and optional PHP plugin)
  ${WP_CLI} config set CUSTOM_LOGIN_SLUG '"'"'{LOGIN_SLUG}'"'"' --path=${WP_PATH} --allow-root
'
```
Replace `{SITE_NUMBER}` with the zero-based site number input, `{DOMAIN_SLUG}` with the captured slug, `{LOGIN_SLUG}` with the custom login slug input.

### 7. Install WordPress Core
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  WP_PATH="/var/www/{domain}/html"
  # Only install if not already done
  if ! ${WP_CLI} core is-installed --path=${WP_PATH} --allow-root 2>/dev/null; then
    ${WP_CLI} core install \
      --url="https://{domain}" \
      --title="{domain}" \
      --admin_user="${WP_ADMIN_USER}" \
      --admin_password="${WP_ADMIN_PASS}" \
      --admin_email="{WP_ADMIN_EMAIL}" \
      --path=${WP_PATH} \
      --allow-root
  else
    echo "WordPress already installed — skipping core install"
  fi
'
```

### 8. Create Per-Site PHP.ini
Select `memory_limit` based on RAM profile: `96M` (2GB), `128M` (4GB), `256M` (8GB+).
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST "
  mkdir -p /usr/local/lsws/conf/php-ini/{DOMAIN_SLUG}
  cat > /usr/local/lsws/conf/php-ini/{DOMAIN_SLUG}/php.ini << 'PHPINIEOF'
memory_limit = {PHP_MEMORY_LIMIT}
upload_max_filesize = 100M
post_max_size = 100M
max_execution_time = 300
max_input_time = 300
PHPINIEOF
"
```
Replace `{PHP_MEMORY_LIMIT}` with the value from RAM profile above, `{DOMAIN_SLUG}` with the captured domain slug.

### 9. Create OLS Vhost Config
Write `/usr/local/lsws/conf/vhosts/{domain}.conf`:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST "
  cat > /usr/local/lsws/conf/vhosts/{domain}.conf << 'OLSVHOSTEOF'
virtualHost {DOMAIN_SLUG} {
  vhRoot                  /var/www/{domain}/html
  allowSymbolLink         1
  enableScript            1
  restrained              0

  errorlog /var/www/{domain}/logs/error.log {
    useServer             0
    logLevel              WARN
    rollingSize           10M
  }

  accesslog /var/www/{domain}/logs/access.log {
    useServer             0
    logFormat             \"%h %l %u %t \\\"%r\\\" %>s %b\"
    rollingSize           10M
  }

  vhssl {
    keyFile               /usr/local/lsws/conf/cert/{domain}.key
    certFile              /usr/local/lsws/conf/cert/{domain}.crt
  }

  rewrite {
    enable                1
    autoLoadHtaccess      1
    rules                 <<<END_rules
# Custom login URL -- maps /{LOGIN_SLUG} to wp-login.php
RewriteRule ^/{LOGIN_SLUG}$ /wp-login.php [QSA,L]
# Block direct wp-login.php access (allow POST for form submission + logout/reset flows)
RewriteCond %{THE_REQUEST} /wp-login\.php
RewriteCond %{REQUEST_URI} ^/wp-login\.php
RewriteCond %{QUERY_STRING} !^action=(logout|rp|resetpass|postpass)
RewriteCond %{HTTP_REFERER} !/{LOGIN_SLUG}
RewriteRule ^wp-login\.php$ - [F,L]
# Block xmlrpc.php
RewriteRule ^xmlrpc\.php$ - [F,L]
# Block PHP execution in wp-includes
RewriteRule ^wp-includes/.*\.php$ - [F,L]
# WordPress permalinks
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
    END_rules
  }

  context /wp-content/ {
    location              /var/www/{domain}/html/wp-content/
    allowBrowse           1
    enableExpires         1
    expiresByType         image/*=A31536000, text/css=A2592000, application/javascript=A2592000
    extraHeaders          set Cache-Control \"public, no-transform\"
  }

  context / {
    location              /var/www/{domain}/html/
    allowBrowse           1
    rewrite {
      enable              1
      inherit             1
    }
  }
}
OLSVHOSTEOF
"
```

### 10. Register Vhost in httpd_config.conf — STAGING APPROACH (CRITICAL)
**NEVER write directly to the production config.** Use copy → append → validate → atomic swap to protect running sites.

Determine memory soft/hard limits based on RAM profile (values from orchestrator):
- 2GB: `memSoftLimit 350M`, `memHardLimit 400M`
- 4GB: `memSoftLimit 500M`, `memHardLimit 600M`
- 8GB+: `memSoftLimit 800M`, `memHardLimit 1000M`

```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST "
  # 1. Staging copy
  cp /usr/local/lsws/conf/httpd_config.conf /usr/local/lsws/conf/httpd_config.conf.staging

  # 2. Append extprocessor entry to STAGING
  cat >> /usr/local/lsws/conf/httpd_config.conf.staging << 'EXTEOF'

extprocessor lsphp82-{DOMAIN_SLUG} {
  type                    lsapi
  address                 uds://tmp/lshttpd/lsphp82.{DOMAIN_SLUG}.sock
  maxConns                {LSAPI_CHILDREN}
  env                     LSAPI_CHILDREN={LSAPI_CHILDREN}
  env                     PHP_INI_SCAN_DIR=/usr/local/lsws/conf/php-ini/{DOMAIN_SLUG}/
  initTimeout             60
  retryTimeout            0
  respBuffer              0
  autoStart               2
  path                    /usr/local/lsws/lsphp82/bin/lsphp
  backlog                 100
  instances               1
  priority                0
  memSoftLimit            {PHP_MEM_SOFT}
  memHardLimit            {PHP_MEM_HARD}
  procSoftLimit           1000
  procHardLimit           1200
}

EXTEOF

  # 3. Append vhostMap entries for both HTTP and HTTPS listeners
  sed -i '/^vhostMap Default {/a\\  {domain}    {DOMAIN_SLUG}\\n  www.{domain}    {DOMAIN_SLUG}' /usr/local/lsws/conf/httpd_config.conf.staging
  sed -i '/^vhostMap HTTPS {/a\\  {domain}    {DOMAIN_SLUG}\\n  www.{domain}    {DOMAIN_SLUG}' /usr/local/lsws/conf/httpd_config.conf.staging

  # 4. Append vhost include to STAGING
  echo 'include /usr/local/lsws/conf/vhosts/{domain}.conf' >> /usr/local/lsws/conf/httpd_config.conf.staging

  # 5. VALIDATE — MUST pass before swap
  /usr/local/lsws/bin/openlitespeed -t -f /usr/local/lsws/conf/httpd_config.conf.staging
  if [ \$? -ne 0 ]; then
    echo 'ERROR: OLS config validation failed. Staging config rejected.'
    rm -f /usr/local/lsws/conf/httpd_config.conf.staging
    echo 'Production config UNTOUCHED. Existing sites still running.'
    exit 1
  fi

  # 6. ATOMIC SWAP — backup production, promote staging
  cp /usr/local/lsws/conf/httpd_config.conf /usr/local/lsws/conf/httpd_config.conf.bak
  mv /usr/local/lsws/conf/httpd_config.conf.staging /usr/local/lsws/conf/httpd_config.conf

  # 7. Graceful restart (zero downtime)
  /usr/local/lsws/bin/lswsctrl restart
"
```
Replace `{PHP_MEM_SOFT}` and `{PHP_MEM_HARD}` with values from RAM profile above. If OLS config validation fails — STOP and report. Do not proceed.

### 11. Install and Configure LiteSpeed Cache Plugin with Redis Object Cache
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  WP_PATH="/var/www/{domain}/html"
  ${WP_CLI} plugin install litespeed-cache --activate --path=${WP_PATH} --allow-root

  # Configure Redis as object cache backend
  ${WP_CLI} option update litespeed.conf.object_kind 1       --path=${WP_PATH} --allow-root  # 1=Redis
  ${WP_CLI} option update litespeed.conf.object_host '"'"'127.0.0.1'"'"' --path=${WP_PATH} --allow-root
  ${WP_CLI} option update litespeed.conf.object_port 6379    --path=${WP_PATH} --allow-root
  ${WP_CLI} option update litespeed.conf.object_db_id {SITE_NUMBER} --path=${WP_PATH} --allow-root
  ${WP_CLI} option update litespeed.conf.object_persistent 1 --path=${WP_PATH} --allow-root
'
```

### 12. Install and Configure WooCommerce
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  WP_PATH="/var/www/{domain}/html"
  ${WP_CLI} plugin install woocommerce --activate --path=${WP_PATH} --allow-root

  # LSCache must not cache cart/checkout/account pages
  ${WP_CLI} option update litespeed.conf.cdn_exc '"'"'/cart/*
/checkout/*
/my-account/*'"'"' --path=${WP_PATH} --allow-root
'
```

### 13. Security Hardening, Default Content Cleanup, and Permalinks
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  WP_PATH="/var/www/{domain}/html"

  # File permissions
  find /var/www/{domain}/html -type f -exec chmod 644 {} \;
  find /var/www/{domain}/html -type d -exec chmod 755 {} \;
  chmod 600 /var/www/{domain}/html/wp-config.php

  # Ownership
  chown -R www-data:www-data /var/www/{domain}

  # Delete default attack-surface content (errors suppressed — may not exist)
  ${WP_CLI} post delete 1 --force --path=${WP_PATH} --allow-root 2>/dev/null
  ${WP_CLI} post delete 2 --force --path=${WP_PATH} --allow-root 2>/dev/null
  ${WP_CLI} plugin delete hello    --path=${WP_PATH} --allow-root 2>/dev/null
  ${WP_CLI} plugin delete akismet  --path=${WP_PATH} --allow-root 2>/dev/null
  ${WP_CLI} theme delete twentytwentythree --path=${WP_PATH} --allow-root 2>/dev/null
  ${WP_CLI} theme delete twentytwentytwo   --path=${WP_PATH} --allow-root 2>/dev/null

  # SEO-friendly permalinks + disable pingbacks
  ${WP_CLI} rewrite structure '"'"'/%postname%/'"'"' --path=${WP_PATH} --allow-root
  ${WP_CLI} option update default_pingback_flag 0 --path=${WP_PATH} --allow-root
'
```

### 14. Create Per-Site Backup Script and Staggered Cron
First create a `.my.cnf` credential file so the backup script never embeds plaintext passwords:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST "
  cat > /var/www/{domain}/.my.cnf << 'MYCNFEOF'
[mysqldump]
user={DB_USER}
password={DB_PASS}
MYCNFEOF
  chmod 600 /var/www/{domain}/.my.cnf
  chown www-data:www-data /var/www/{domain}/.my.cnf
"
```
Write the backup script `/usr/local/bin/backup-{DOMAIN_SLUG}.sh`:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST "
  cat > /usr/local/bin/backup-{DOMAIN_SLUG}.sh << 'BACKUPEOF'
#!/bin/bash
SITE=\"{domain}\"
BACKUP_DIR=\"/var/backups/wordpress\"
DATE=\$(date +%Y%m%d-%H%M%S)
DB_NAME=\"{DB_NAME}\"
MY_CNF=\"/var/www/\${SITE}/.my.cnf\"

mkdir -p \"\${BACKUP_DIR}\"

# Database backup — credentials from .my.cnf, no plaintext password in script
mysqldump --defaults-file=\"\${MY_CNF}\" \${DB_NAME} | gzip > \"\${BACKUP_DIR}/\${SITE}_db_\${DATE}.sql.gz\"

# Files backup — exclude cache directories
tar --exclude='wp-content/cache' --exclude='wp-content/litespeed' \\
  -czf \"\${BACKUP_DIR}/\${SITE}_files_\${DATE}.tar.gz\" \\
  /var/www/\${SITE}/html 2>/dev/null

# Retention: remove backups older than 30 days
find \"\${BACKUP_DIR}\" -name \"\${SITE}_*\" -mtime +30 -delete

echo \"Backup complete: \${SITE} @ \${DATE}\"
BACKUPEOF
  chmod +x /usr/local/bin/backup-{DOMAIN_SLUG}.sh
"
```
Add cron entry — staggered by `site_number` (site 0 = 02:00, site 1 = 02:30, site 2 = 03:00, etc.):
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  CRON_MINUTE=$(( {SITE_NUMBER} * 30 % 60 ))
  CRON_HOUR=$(( 2 + {SITE_NUMBER} * 30 / 60 ))
  (crontab -l 2>/dev/null; echo "${CRON_MINUTE} ${CRON_HOUR} * * * /usr/local/bin/backup-{DOMAIN_SLUG}.sh >> /var/log/backup-{DOMAIN_SLUG}.log 2>&1") | crontab -
'
```
Capture `CRON_HOUR` and `CRON_MINUTE` for the output report.

### 15. Verify Installation
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  WP_PATH="/var/www/{domain}/html"

  # OLS config syntax check
  /usr/local/lsws/bin/openlitespeed -t

  # WP-CLI health checks
  ${WP_CLI} core version --path=${WP_PATH} --allow-root
  ${WP_CLI} option get siteurl --path=${WP_PATH} --allow-root
  ${WP_CLI} plugin list --path=${WP_PATH} --allow-root --format=table

  # HTTP response check (direct to server with Host header — Cloudflare may not be live yet)
  curl -sk -H "Host: {domain}" https://localhost/ | head -20
'
```
If `openlitespeed -t` fails: investigate `/usr/local/lsws/logs/error.log` and fix before continuing.

## Output
Return ALL generated credentials in this copy-friendly block. The user must save this immediately — credentials cannot be retrieved later.

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
  DB Creds File:    /var/www/{DOMAIN}/.my.cnf (chmod 600)

── SECURITY ───────────────────────────────────────────────────
  Custom Login:     /{LOGIN_SLUG}  (direct /wp-login.php blocked)
  XML-RPC:          blocked (OLS rewrite)
  File Editor:      disabled (DISALLOW_FILE_EDIT)
  SSL:              forced (FORCE_SSL_ADMIN)
  File Perms:       644 files / 755 dirs / 600 wp-config

═══════════════════════════════════════════════════════════════
```

## Error Handling
- **OLS config validation fails (Step 10):** STOP immediately. Staging file is deleted automatically. Production config and all running sites are untouched. Report error to user with `/usr/local/lsws/logs/error.log` contents.
- **WP already installed:** `wp core is-installed` detects this; skip download + install steps. Warn user if unexpected.
- **DB creation fails:** Check MariaDB running (`systemctl status mariadb`) and that root password is correct.
- **WP-CLI fails with system PHP:** Create LSPHP wrapper (Step 4). Always test with `wp-lsphp --info --allow-root` after creation.
- **Redis not running:** Verify with `redis-cli ping`. If DOWN, check `systemctl status redis` — Redis should have been installed in Phase 1.
- **Custom login rewrite too restrictive:** Allow POST to wp-login.php from correct referer. Test password reset flow separately after setup.
- **Non-critical failures** (deleting default content that does not exist): Log and continue. Do not abort.

## Important Rules
- NEVER save credentials to any file — `.my.cnf` and `wp-config.php` are the only authorized exceptions (required for operation)
- All generated passwords use `openssl rand` (cryptographically secure)
- Run ALL commands via SSH — never execute server-side operations locally
- Every step is idempotent — check-before-act; safe to re-run if interrupted
- NEVER use `GRANT ALL PRIVILEGES` — use only the specific grants listed in Step 3
- NEVER write directly to `httpd_config.conf` — always use staging copy + validate + atomic swap (Step 10)
- Use `--allow-root` for all WP-CLI commands when SSH user is root
- Always specify `--path=` for all WP-CLI commands
- Output ALL generated credentials in the results block — the user cannot retrieve them later
- `site_number` must be unique per site on the server — never reuse a number (Redis DB collision)
- The orchestrator (`domain-setup-agent`) handles calling this skill N times for N sites and passes correct `site_number`, `LSAPI_CHILDREN`, and RAM profile for each call
