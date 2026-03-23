# Phase 2: WordPress Site Setup Skill

## Context
- Parent plan: [plan.md](./plan.md)
- Depends on: [Phase 1 — VPS Server Setup](./phase-01-vps-server-setup.md)
- Reference: `.claude/skills/wordpress-site-audit/SKILL.md` (inverse of this)

## Overview
- **Priority:** P1
- **Status:** pending
- **Description:** Create `wordpress-site-setup` skill that installs WP-CLI, creates MariaDB database, downloads WordPress, configures wp-config.php with security hardening, creates Nginx server block, and delivers admin credentials.

## Key Insights
- WP-CLI is the primary tool — install it first if not present
- Auto-generate ALL credentials (DB name/user/pass, WP admin user/pass)
- Non-default DB prefix critical for security (e.g., `wp_x7k_` not `wp_`)
- Nginx server block must handle PHP-FPM upstream correctly
- Security hardening should match what the audit skill checks for (so audit scores 8+)
- File ownership must be correct (www-data:www-data on Debian/Ubuntu)

## Requirements

### Functional
- Install WP-CLI (skip if exists)
- Auto-generate MariaDB database name, user, password
- Create database and grant privileges
- Download WordPress to `/var/www/{domain}`
- Generate wp-config.php with fresh salts, non-default prefix, security constants
- Run `wp core install` with auto-generated admin credentials
- Apply security hardening (XML-RPC block, REST API user enum block, file perms)
- Create Nginx server block for the domain
- Delete default WordPress content (Hello World post, sample page, unused themes)
- Set correct file ownership (www-data)

### Non-Functional
- Idempotent: detect existing WordPress install, skip if present
- Generated passwords must be cryptographically strong (openssl rand)
- All credentials returned to orchestrator (never saved to files)

## Architecture

### SKILL.md Structure
```
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
```

### Setup Sequence
1. Install WP-CLI: download phar, chmod, move to /usr/local/bin/wp
2. Generate credentials: `openssl rand -base64 24` for DB pass, `openssl rand -base64 18` for WP admin pass
3. Create DB: `mysql -e "CREATE DATABASE IF NOT EXISTS wp_{slug}_{rand}"`
4. Grant DB user: `mysql -e "GRANT ALL ON db.* TO 'user'@'localhost' IDENTIFIED BY 'pass'"`
5. Create web root: `mkdir -p /var/www/{domain}`
6. Download WP: `wp core download --path=/var/www/{domain} --allow-root`
7. Create wp-config: `wp config create --dbname=X --dbuser=X --dbpass=X --dbprefix=wp_{rand}_ --allow-root`
8. Set constants: `wp config set DISALLOW_FILE_EDIT true --raw --allow-root`
9. Install WP: `wp core install --url=https://{domain} --title="{domain}" --admin_user=... --admin_password=... --admin_email=... --allow-root`
10. Security hardening: file perms, delete defaults, block XML-RPC
11. Create Nginx server block → symlink → test → reload
12. Set ownership: `chown -R www-data:www-data /var/www/{domain}`

### Nginx Server Block Template
```nginx
server {
    listen 80;
    server_name {domain} www.{domain};
    root /var/www/{domain};
    index index.php index.html;

    # Security: block xmlrpc
    location = /xmlrpc.php { deny all; }

    # Security: block REST API user enumeration
    location ~ ^/wp-json/wp/v2/users { deny all; }

    # Security: block wp-includes browsing
    location ~* /wp-includes/.*\.php$ { deny all; }

    # PHP handling
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock;
    }

    # WordPress permalinks
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # Static file caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # Deny hidden files
    location ~ /\. { deny all; }
}
```

## Related Code Files
- **Create:** `.claude/skills/wordpress-site-setup/SKILL.md`
- **Reference:** `.claude/skills/wordpress-site-audit/SKILL.md` (audit checks we must pass)

## Implementation Steps

1. Create directory `.claude/skills/wordpress-site-setup/`
2. Write `SKILL.md` with frontmatter
3. Write Purpose, Required Input, SSH Connection sections
4. Write WP-CLI Installation section (with idempotency check)
5. Write Credential Generation section:
   - DB name: `wp_{domain_slug}_{4char_random}`
   - DB user: `wp_usr_{4char_random}`
   - DB password: `openssl rand -base64 24`
   - WP admin user: `wp_admin_{4char_random}`
   - WP admin password: `openssl rand -base64 18`
6. Write Database Setup section (CREATE DATABASE, GRANT)
7. Write WordPress Download & Configure section (wp core download, config create, config set)
8. Write WordPress Install section (wp core install)
9. Write Security Hardening section:
   - DISALLOW_FILE_EDIT, WP_DEBUG=false
   - File permissions (644/755/600)
   - Delete default content
   - Delete unused themes
10. Write Nginx Server Block section (template above, detect PHP-FPM socket path)
11. Write File Ownership section (www-data)
12. Write Output section — credentials table for orchestrator
13. Write Error Handling and Important Rules sections

## Todo List
- [ ] Create `.claude/skills/wordpress-site-setup/` directory
- [ ] Write SKILL.md with frontmatter
- [ ] WP-CLI installation with idempotency
- [ ] Credential auto-generation logic
- [ ] MariaDB database + user creation
- [ ] WordPress download and wp-config.php generation
- [ ] Security constants (DISALLOW_FILE_EDIT, WP_DEBUG)
- [ ] `wp core install` with auto-generated credentials
- [ ] File permissions hardening (644/755/600)
- [ ] Default content cleanup (posts, themes)
- [ ] Nginx server block template with PHP-FPM
- [ ] XML-RPC and REST API user enum blocks in Nginx
- [ ] File ownership (www-data:www-data)
- [ ] Output format with all generated credentials

## Success Criteria
- WordPress accessible at `http://{domain}` after setup (before SSL)
- Admin login works at `/wp-admin` with generated credentials
- Audit skill (`wordpress-site-audit`) scores 7+/10 on freshly installed site
- wp-config.php has non-default DB prefix, DISALLOW_FILE_EDIT=true, WP_DEBUG=false
- File permissions: 644 files, 755 dirs, 600 wp-config.php
- XML-RPC blocked, REST API user enum blocked
- No default "Hello World" post or sample page

## Risk Assessment
| Risk | Impact | Mitigation |
|------|--------|------------|
| WordPress already installed at path | HIGH | Check if wp-config.php exists, warn user |
| PHP-FPM socket path varies by version | MEDIUM | Auto-detect: `find /run/php -name "*.sock"` |
| MariaDB root password unknown | MEDIUM | Try passwordless, try password from Phase 1 |
| WP-CLI download URL changes | LOW | Check wp-cli.org for current URL |
| Nginx sites-enabled path varies | LOW | Check both `/etc/nginx/sites-enabled/` and `/etc/nginx/conf.d/` |

## Security Considerations
- All generated passwords use `openssl rand` (cryptographically secure)
- wp-config.php set to 600 permissions (owner read/write only)
- Database user has privileges only on its own database
- XML-RPC blocked at Nginx level (before PHP processes request)
- REST API user enumeration blocked

## Next Steps
→ Phase 3: `cloudflare-domain-setup` (needs server IP + WordPress installed)
