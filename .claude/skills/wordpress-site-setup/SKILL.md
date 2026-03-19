---
name: wordpress-site-setup
description: Install WordPress on a VPS with WP-CLI, create database, configure wp-config.php with security hardening, set up Nginx server block. Use when user wants to install WordPress on a prepared VPS server.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---

# WordPress Site Setup Skill

## Purpose
Install WordPress on a VPS via WP-CLI (over SSH). Create MariaDB database, configure wp-config.php with security hardening, set up Nginx server block, and return all generated credentials to the user.

## Required Input (from user, NEVER save to files)
- SSH host (IP or domain)
- SSH user (typically root)
- SSH password
- SSH port (default 22)
- Domain name (the site being installed, e.g. example.com)
- MariaDB root password (from vps-server-setup or provided by user)
- WordPress admin email

## SSH Connection
Use `sshpass` for password-based SSH. All commands run via:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

If `sshpass` not installed locally, install it first: `sudo apt-get install -y sshpass`

## Setup Checklist

### 1. Install WP-CLI
- Check if already installed:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'command -v wp &>/dev/null && echo "INSTALLED" || echo "MISSING"'
```
- If MISSING, install:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
  chmod +x wp-cli.phar
  mv wp-cli.phar /usr/local/bin/wp
'
```
- Verify: `wp --info --allow-root`

### 2. Generate Credentials
All credentials are auto-generated and cryptographically secure. Run on the server:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  DB_NAME="wp_$(echo DOMAIN | tr ".-" "_" | cut -c1-10)_$(openssl rand -hex 2)"
  DB_USER="wp_usr_$(openssl rand -hex 2)"
  DB_PASS="$(openssl rand -base64 24)"
  WP_ADMIN_USER="wp_admin_$(openssl rand -hex 2)"
  WP_ADMIN_PASS="$(openssl rand -base64 18)"
  echo "DB_NAME=${DB_NAME}"
  echo "DB_USER=${DB_USER}"
  echo "DB_PASS=${DB_PASS}"
  echo "WP_ADMIN_USER=${WP_ADMIN_USER}"
  echo "WP_ADMIN_PASS=${WP_ADMIN_PASS}"
'
```
Capture all five values from output — they are used in all subsequent steps.

### 3. Create Database
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  mysql -u root -p'"'"'MARIADB_ROOT_PASS'"'"' -e "CREATE DATABASE IF NOT EXISTS ${DB_NAME} DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
  mysql -u root -p'"'"'MARIADB_ROOT_PASS'"'"' -e "CREATE USER IF NOT EXISTS '"'"'${DB_USER}'"'"'@'"'"'localhost'"'"' IDENTIFIED BY '"'"'${DB_PASS}'"'"';"
  mysql -u root -p'"'"'MARIADB_ROOT_PASS'"'"' -e "GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '"'"'${DB_USER}'"'"'@'"'"'localhost'"'"';"
  mysql -u root -p'"'"'MARIADB_ROOT_PASS'"'"' -e "FLUSH PRIVILEGES;"
'
```

### 4. Download WordPress
- Check if WordPress already exists at the target path:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'test -f /var/www/{domain}/wp-config.php && echo "EXISTS" || echo "NEW"'
```
- If EXISTS: warn user, ask before proceeding. Skip download/install steps unless user confirms overwrite.
- If NEW, create directory and download:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  mkdir -p /var/www/{domain}
  wp core download --path=/var/www/{domain} --allow-root
'
```

### 5. Configure wp-config.php
Generate a random table prefix and create config:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  WP_PREFIX="wp_$(openssl rand -hex 2)_"
  wp config create \
    --path=/var/www/{domain} \
    --dbname=${DB_NAME} \
    --dbuser=${DB_USER} \
    --dbpass=${DB_PASS} \
    --dbprefix=${WP_PREFIX} \
    --allow-root
'
```
Set security constants:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  wp config set DISALLOW_FILE_EDIT true  --raw --path=/var/www/{domain} --allow-root
  wp config set WP_DEBUG false           --raw --path=/var/www/{domain} --allow-root
  wp config set WP_POST_REVISIONS 5     --raw --path=/var/www/{domain} --allow-root
  wp config set FORCE_SSL_ADMIN true     --raw --path=/var/www/{domain} --allow-root
  wp config set WP_AUTO_UPDATE_CORE true --raw --path=/var/www/{domain} --allow-root
'
```

### 6. Install WordPress Core
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  wp core install \
    --url="https://{domain}" \
    --title="{domain}" \
    --admin_user="${WP_ADMIN_USER}" \
    --admin_password="${WP_ADMIN_PASS}" \
    --admin_email="{WP_ADMIN_EMAIL}" \
    --path=/var/www/{domain} \
    --allow-root
'
```

### 7. Security Hardening
Set correct file permissions:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  find /var/www/{domain} -type f -exec chmod 644 {} \;
  find /var/www/{domain} -type d -exec chmod 755 {} \;
  chmod 600 /var/www/{domain}/wp-config.php
'
```
Delete default content (attack surface reduction):
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  wp post delete 1 --force --path=/var/www/{domain} --allow-root  2>/dev/null  # Hello World post
  wp post delete 2 --force --path=/var/www/{domain} --allow-root  2>/dev/null  # Sample page
  wp plugin delete hello   --path=/var/www/{domain} --allow-root  2>/dev/null
  wp plugin delete akismet --path=/var/www/{domain} --allow-root  2>/dev/null
  wp theme delete twentytwentythree --path=/var/www/{domain} --allow-root 2>/dev/null
  wp theme delete twentytwentytwo   --path=/var/www/{domain} --allow-root 2>/dev/null
'
```
Configure SEO-friendly permalinks and disable pingbacks:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  wp rewrite structure '"'"'/%postname%/'"'"' --path=/var/www/{domain} --allow-root
  wp option update default_pingback_flag 0 --path=/var/www/{domain} --allow-root
'
```

### 8. Create Nginx Server Block
First detect the PHP-FPM socket:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'find /run/php -name "*.sock" 2>/dev/null | head -1'
```
Capture the socket path (e.g. `/run/php/php8.2-fpm.sock`), then write the server block:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST "cat > /etc/nginx/sites-available/{domain} << 'NGINXEOF'
server {
    listen 80;
    server_name {domain} www.{domain};
    root /var/www/{domain};
    index index.php index.html;

    # Security: block XML-RPC
    location = /xmlrpc.php { deny all; }

    # Security: block REST API user enumeration
    location ~ ^/wp-json/wp/v2/users { deny all; }

    # Security: block direct access to wp-includes PHP files
    location ~* /wp-includes/.*\.php$ { deny all; }

    # PHP handling
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:PHP_FPM_SOCKET;
    }

    # WordPress permalinks
    location / {
        try_files \$uri \$uri/ /index.php?\$args;
    }

    # Static file caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control \"public, no-transform\";
    }

    # Deny hidden files
    location ~ /\. { deny all; }
}
NGINXEOF"
```
Replace `PHP_FPM_SOCKET` with the actual socket path detected above.

Enable the site and reload Nginx:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  ln -sf /etc/nginx/sites-available/{domain} /etc/nginx/sites-enabled/
  rm -f /etc/nginx/sites-enabled/default
  nginx -t && systemctl reload nginx
'
```
If `nginx -t` fails: read the error, fix the config syntax before reloading.

### 9. Set File Ownership
For Debian/Ubuntu (www-data):
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'chown -R www-data:www-data /var/www/{domain}'
```
For CentOS/RHEL (nginx user):
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'chown -R nginx:nginx /var/www/{domain}'
```
Detect OS first with `cat /etc/os-release` if not already known.

### 10. Verify Installation
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  wp core version --path=/var/www/{domain} --allow-root
  wp option get siteurl --path=/var/www/{domain} --allow-root
'
```
Check HTTP response (expect 200 or 301):
```bash
curl -sI http://{domain} | head -5
```

## Output
Return all generated credentials in this table — the orchestrator delivers them to the user:

```
## WordPress Setup Results
| Item | Value |
|------|-------|
| WP Admin URL | https://{domain}/wp-admin |
| WP Admin User | {WP_ADMIN_USER} |
| WP Admin Password | {WP_ADMIN_PASS} |
| DB Name | {DB_NAME} |
| DB User | {DB_USER} |
| DB Password | {DB_PASS} |
| WP Version | X.X |
| Install Path | /var/www/{domain} |
| PHP-FPM Socket | /run/php/phpX.X-fpm.sock |
```

## Error Handling
- **WP already exists at path**: Warn user, skip download and install steps unless they explicitly confirm overwrite
- **DB creation fails**: Check if MariaDB is running (`systemctl status mariadb`) and if root password is correct
- **`nginx -t` fails**: Read error output, fix config syntax, retest before reload — never reload a broken config
- **WP-CLI missing**: Install it (step 1) before proceeding; do not skip
- **Non-critical failures** (e.g. deleting default theme that does not exist): Log and continue, do not abort

## Important Rules
- NEVER save credentials to any file (wp-config.php is the sole exception — it must contain DB creds)
- All generated passwords use `openssl rand` (cryptographically secure)
- Run ALL commands via SSH — never execute server-side operations locally
- Every step is idempotent — safe to re-run if interrupted
- Output ALL generated credentials in the results table — the user cannot retrieve them later
- Use `--allow-root` for all WP-CLI commands when SSH user is root
- Always specify `--path=` for WP-CLI commands
