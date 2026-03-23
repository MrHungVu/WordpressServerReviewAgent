# Phase 1: vps-server-setup v2 (MAJOR REWRITE)

## Context Links

- Current skill: `.claude/skills/vps-server-setup/SKILL.md`
- Research: [OLS Production](../reports/researcher-260323-1708-openlitespeed-wordpress-production.md)
- Research: [Multi-Site Best Practices](../reports/researcher-260323-1708-multi-site-openlitespeed-best-practices.md)
- Brainstorm: [Gap Analysis](../reports/brainstorm-260323-1703-ols-stack-upgrade.md)

## Overview

- **Priority:** P1 (foundation -- all other phases depend on this)
- **Status:** complete
- **Effort:** 10h
- **Description:** Complete rewrite of vps-server-setup from Nginx/LEMP to OLS + LSPHP 8.2 + MariaDB + Redis + OPcache. Adds OS hardening (sudo user, SSH key auth, custom port), Cloudflare-only firewall, fail2ban OLS jails, adaptive memory tuning, and backup infrastructure.

## Key Insights

- OLS installed via official LiteSpeed repo (`repo.litespeed.sh`), NOT CyberPanel
- Plain text config mode disables WebAdmin console automatically
- `useClientIpInHeader` + trusted CF IPs restores real visitor IP
- Adaptive memory: detect RAM at setup, compute profiles for MariaDB/Redis/OPcache/LSAPI_CHILDREN
- Default `_Primary_` vhost must return 403 to block direct IP access
- All SSH hardening steps (root login, password auth, port change) are OPTIONAL -- only applied if user provides SSH key/custom port
- **IDEMPOTENT:** Must detect existing OLS/MariaDB/Redis. If already installed, skip install steps and return existing config. Critical for multi-site additions (adding site #2 to existing VPS)
<!-- Updated: Validation Session 1 - Added idempotent detection requirement -->
- **CentOS support:** Ubuntu primary + basic CentOS with yum equivalents for all apt commands
<!-- Updated: Validation Session 1 - Added CentOS support requirement -->

## Requirements

### Functional
- F1: Detect OS (Debian/Ubuntu vs CentOS/RHEL) and set package manager
- F2: Create non-root sudo user (optional, if username provided)
- F3: SSH hardening: custom port, key auth, disable root+password (optional, with safety checks)
- F4: Set timezone to UTC
- F5: Install OLS + LSPHP 8.2 with all required extensions from LiteSpeed repo
- F6: Switch OLS to plain text config, disable WebAdmin
- F7: Configure base httpd_config.conf: listeners 80+443, CF-Connecting-IP, trusted CF IPs
- F8: Default vhost `_Primary_` returns 403 (blocks direct IP)
- F9: Install + secure MariaDB, auto-gen root password, tune my.cnf adaptively
- F10: Install Redis, configure maxmemory adaptive, allkeys-lru, bind localhost
- F11: Tune LSPHP OPcache: adaptive memory, JIT, validate_timestamps=0
- F12: UFW: fetch CF IPs from API, allow 80/443 from CF only, allow SSH from admin IP
- F13: fail2ban: SSH jail + OLS access log jail
- F14: OLS log rotation via logrotate
- F15: Create backup infrastructure dirs
- F16: Output all generated creds + RAM profile to orchestrator

### Non-Functional
- Idempotent: safe to re-run on partial setups
- Every step check-before-act
- All commands run remotely via sshpass SSH
- No credentials saved to files

## Architecture

```
VPS Server After Setup:
├── /usr/local/lsws/                    # OLS root
│   ├── conf/httpd_config.conf          # Main config (plain text)
│   ├── conf/vhosts/                    # Empty (sites added by Phase 2)
│   ├── conf/cert/                      # SSL certs dir (populated by Phase 2/3)
│   ├── conf/php-ini/                   # Per-site php.ini dirs (Phase 2)
│   ├── conf/disablewebconsole          # WebAdmin disabled marker
│   ├── lsphp82/                        # LSPHP binary + extensions
│   ├── logs/                           # Access + error logs
│   └── tmp/                            # LSPHP sockets dir
├── /etc/mysql/mariadb.conf.d/          # MariaDB tuning
├── /etc/redis/redis.conf               # Redis config
├── /var/backups/wordpress/             # Backup storage
└── UFW: SSH from admin_ip, 80+443 from CF IPs only
```

## Related Code Files

- **Modify:** `.claude/skills/vps-server-setup/SKILL.md` (complete rewrite)

## Implementation Steps

### Step 0: SKILL.md Frontmatter
Keep same format as v1:
```yaml
---
name: vps-server-setup
description: SSH into a fresh Linux VPS and install OpenLiteSpeed + LSPHP 8.2 + MariaDB + Redis with security hardening, Cloudflare-only firewall, adaptive memory tuning. Foundation for multi-site WordPress hosting. Idempotent.
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
- SSH host (IP address)
- SSH password
- SSH user (default: root)
- SSH port (default: 22)
- Domain name (for context/report -- first site)
- Admin IP (for UFW SSH whitelist)
- New sudo username (optional -- skip if empty)
- SSH public key (optional -- skip if empty)
- Custom SSH port (optional -- skip if empty)
```

### Step 2: Detect OS + Existing Stack
```bash
cat /etc/os-release
```
- Set `PKG_MGR=apt` for Debian/Ubuntu, `PKG_MGR=yum` for CentOS/RHEL
- Detect version for repo compatibility

**Idempotent detection (MUST run before any installs):**
```bash
OLS_INSTALLED=0; MARIADB_INSTALLED=0; REDIS_INSTALLED=0
systemctl is-active lsws &>/dev/null && OLS_INSTALLED=1
systemctl is-active mariadb &>/dev/null && MARIADB_INSTALLED=1
systemctl is-active redis-server &>/dev/null && REDIS_INSTALLED=1
```
If ALL three active: skip install steps 10-14, detect LSPHP version, detect RAM profile, ask user for existing MariaDB root password, return existing config to orchestrator.
If SOME active: continue with missing components only (check-before-install pattern).
<!-- Updated: Validation Session 1 - Added idempotent stack detection -->

### Step 3: System Update
```bash
# Wait for apt lock
while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 2; done
apt update && apt upgrade -y
# CentOS: yum update -y
```

### Step 4: Detect RAM + Set Memory Profile
```bash
TOTAL_RAM_MB=$(free -m | awk '/^Mem:/{print $2}')
```
Compute adaptive profile:
```
if TOTAL_RAM_MB <= 2500:  PROFILE=2GB,  LSAPI_BASE=8,  OPCACHE_MEM=128, MARIADB_POOL=350, REDIS_MEM=200
if TOTAL_RAM_MB <= 5000:  PROFILE=4GB,  LSAPI_BASE=12, OPCACHE_MEM=256, MARIADB_POOL=768, REDIS_MEM=256
if TOTAL_RAM_MB > 5000:   PROFILE=8GB+, LSAPI_BASE=20, OPCACHE_MEM=512, MARIADB_POOL=2048, REDIS_MEM=512
```
Output profile to conversation for orchestrator.

### Step 5: Create Sudo User (optional)
Only if `NEW_SUDO_USER` provided:
```bash
id ${NEW_SUDO_USER} &>/dev/null || useradd -m -s /bin/bash ${NEW_SUDO_USER}
usermod -aG sudo ${NEW_SUDO_USER}
echo "${NEW_SUDO_USER}:$(openssl rand -base64 18)" | chpasswd
```

### Step 6: SSH Key Auth (optional)
Only if `SSH_PUBLIC_KEY` provided:
```bash
TARGET_USER=${NEW_SUDO_USER:-root}
mkdir -p /home/${TARGET_USER}/.ssh
echo "${SSH_PUBLIC_KEY}" >> /home/${TARGET_USER}/.ssh/authorized_keys
chmod 700 /home/${TARGET_USER}/.ssh
chmod 600 /home/${TARGET_USER}/.ssh/authorized_keys
chown -R ${TARGET_USER}:${TARGET_USER} /home/${TARGET_USER}/.ssh
```
**Test key auth works before disabling password:**
```bash
# Verify PubkeyAuthentication is enabled
grep -q "^PubkeyAuthentication yes" /etc/ssh/sshd_config || echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config
```

### Step 7: SSH Port Change (optional)
Only if `CUSTOM_SSH_PORT` provided:
```bash
sed -i "s/^#*Port .*/Port ${CUSTOM_SSH_PORT}/" /etc/ssh/sshd_config
```
**CRITICAL:** Update UFW BEFORE restarting sshd to avoid lockout.

### Step 8: SSH Hardening
```bash
sed -i 's/^#*MaxAuthTries.*/MaxAuthTries 5/' /etc/ssh/sshd_config
grep -q "^MaxAuthTries" /etc/ssh/sshd_config || echo "MaxAuthTries 5" >> /etc/ssh/sshd_config
```
If SSH key was installed AND tested:
```bash
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
```
**DO NOT disable password auth unless key auth is confirmed working.**

### Step 9: Set Timezone UTC
```bash
timedatectl set-timezone UTC
```

### Step 10: Install OpenLiteSpeed + LSPHP 8.2
```bash
# Add LiteSpeed repo
wget -O - https://repo.litespeed.sh | bash

# Install OLS
apt install openlitespeed -y

# Install LSPHP 8.2 + extensions
apt install lsphp82 lsphp82-common lsphp82-mysql lsphp82-curl lsphp82-gd \
  lsphp82-mbstring lsphp82-xml lsphp82-zip lsphp82-imagick lsphp82-intl \
  lsphp82-bcmath lsphp82-redis lsphp82-opcache -y
```
Stop Apache if running:
```bash
systemctl stop apache2 2>/dev/null; systemctl disable apache2 2>/dev/null
```

### Step 11: Switch to Plain Text Config + Disable WebAdmin
```bash
# Disable WebAdmin console
touch /usr/local/lsws/conf/disablewebconsole

# Create cert directory for origin certs
mkdir -p /usr/local/lsws/conf/cert
mkdir -p /usr/local/lsws/conf/vhosts
mkdir -p /usr/local/lsws/conf/php-ini
```

### Step 12: Configure Base httpd_config.conf
Write the base config with listeners, CF-Connecting-IP, and default 403 vhost. Fetch Cloudflare IP ranges first:
```bash
CF_IPV4=$(curl -s https://www.cloudflare.com/ips-v4)
CF_IPV6=$(curl -s https://www.cloudflare.com/ips-v6)
```

Write `/usr/local/lsws/conf/httpd_config.conf`:
```
serverName                ProductionServer
user                      nobody
group                     nogroup
autoRestart               1
priority                  0

useClientIpInHeader       2
clientIpHeader            CF-Connecting-IP

# Trusted Cloudflare IPs (auto-populated at setup time)
accessControl {
  allow                   {CF_IPV4_LINES}
  allow                   {CF_IPV6_LINES}
  allow                   {ADMIN_IP}
}

listener Default {
  address                 *:80
  secure                  0
}

listener HTTPS {
  address                 *:443
  secure                  1
  sslProtocol             24
  sslCiphers              ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
}

# Default vhost: blocks direct IP access
virtualHost _Primary_ {
  vhRoot                  /var/www/html
  configFile              /usr/local/lsws/conf/vhosts/_default.conf
}

vhostMap Default {
  *                       _Primary_
}

vhostMap HTTPS {
  *                       _Primary_
}

# Per-site includes added by wordpress-site-setup
# include /usr/local/lsws/conf/vhosts/{domain}.conf
```

Default vhost config `/usr/local/lsws/conf/vhosts/_default.conf`:
```
docRoot                   /var/www/html

rewrite {
  enable                  1
  rules                   <<<END_rules
RewriteRule .* - [F,L]
  END_rules
}
```

### Step 13: Install + Secure MariaDB
```bash
apt install mariadb-server -y
systemctl enable mariadb && systemctl start mariadb
```
Generate root password and secure:
```bash
MARIADB_ROOT_PASS=$(openssl rand -base64 24)
mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '${MARIADB_ROOT_PASS}';"
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "DELETE FROM mysql.user WHERE User='';"
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "DROP DATABASE IF EXISTS test;"
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "FLUSH PRIVILEGES;"
```

Tune `/etc/mysql/mariadb.conf.d/50-server.cnf` (append or replace):
```ini
[mysqld]
innodb_buffer_pool_size = ${MARIADB_POOL}M
max_connections = 150
thread_cache_size = 32
table_open_cache = 2048
table_definition_cache = 1000
max_allowed_packet = 64M
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```
```bash
systemctl restart mariadb
```

### Step 14: Install + Configure Redis
```bash
apt install redis-server -y
```
Edit `/etc/redis/redis.conf`:
```bash
sed -i "s/^# maxmemory .*/maxmemory ${REDIS_MEM}mb/" /etc/redis/redis.conf
sed -i "s/^# maxmemory-policy .*/maxmemory-policy allkeys-lru/" /etc/redis/redis.conf
sed -i "s/^bind .*/bind 127.0.0.1 ::1/" /etc/redis/redis.conf
```
```bash
systemctl enable redis-server && systemctl restart redis-server
```

### Step 15: Tune LSPHP OPcache
Create/edit `/usr/local/lsws/lsphp82/etc/php/8.2/mods-available/opcache.ini`:
```ini
opcache.enable=1
opcache.memory_consumption=${OPCACHE_MEM}
opcache.interned_strings_buffer=32
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.revalidate_freq=0
opcache.enable_cli=0
opcache.jit=tracing
opcache.jit_buffer_size=128M
```

### Step 16: Configure UFW (Cloudflare-only)
```bash
command -v ufw &>/dev/null || apt install ufw -y

# Reset to defaults
ufw --force reset

# Default deny incoming
ufw default deny incoming
ufw default allow outgoing

# Allow SSH from admin IP only (use current or custom port)
ACTIVE_SSH_PORT=${CUSTOM_SSH_PORT:-$(grep -E "^Port " /etc/ssh/sshd_config | awk '{print $2}')}
ACTIVE_SSH_PORT=${ACTIVE_SSH_PORT:-22}
ufw allow from ${ADMIN_IP} to any port ${ACTIVE_SSH_PORT} proto tcp

# Allow 80/443 from Cloudflare IPs only
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  ufw allow from $ip to any port 80 proto tcp
  ufw allow from $ip to any port 443 proto tcp
done
for ip in $(curl -s https://www.cloudflare.com/ips-v6); do
  ufw allow from $ip to any port 80 proto tcp
  ufw allow from $ip to any port 443 proto tcp
done

ufw --force enable
```

**IMPORTANT:** Run this AFTER SSH port change (Step 7) but BEFORE sshd restart.

### Step 17: Restart SSH (if port/config changed)
```bash
systemctl reload sshd
```

### Step 18: Install + Configure fail2ban
```bash
apt install fail2ban -y
```
Create `/etc/fail2ban/jail.local`:
```ini
[sshd]
enabled = true
port = ${ACTIVE_SSH_PORT}
maxretry = 5
bantime = 3600
findtime = 600

[openlitespeed-auth]
enabled = true
port = http,https
filter = openlitespeed-auth
logpath = /usr/local/lsws/logs/access.log
maxretry = 10
bantime = 3600
findtime = 600
```
Create `/etc/fail2ban/filter.d/openlitespeed-auth.conf`:
```ini
[Definition]
failregex = ^<HOST> .* "(GET|POST) .*/wp-login\.php.*" (200|302)
ignoreregex =
```
```bash
systemctl enable fail2ban && systemctl restart fail2ban
```

### Step 19: OLS Log Rotation
Create `/etc/logrotate.d/openlitespeed`:
```
/usr/local/lsws/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 640 nobody nogroup
    sharedscripts
    postrotate
        /usr/local/lsws/bin/lswsctrl restart > /dev/null 2>&1 || true
    endscript
}
```

### Step 20: Backup Infrastructure
```bash
mkdir -p /var/backups/wordpress
```

### Step 21: Start OLS + Verify
```bash
# Validate config
/usr/local/lsws/bin/openlitespeed -t

# Enable + start
systemctl enable lsws
systemctl start lsws
```
Verify all services:
```bash
systemctl is-active lsws
/usr/local/lsws/lsphp82/bin/lsphp -v
systemctl is-active mariadb
systemctl is-active redis-server
ufw status numbered
fail2ban-client status
```

### Step 22: Output Results
**IMPORTANT:** Output ALL information in copy-friendly format so user can save it.

```
═══════════════════════════════════════════════════════════════
  VPS SERVER SETUP COMPLETE (OLS v2.0)
  COPY AND SAVE ALL INFORMATION BELOW
═══════════════════════════════════════════════════════════════

── SERVER ─────────────────────────────────────────────────────
  Server IP:            {SERVER_IP}
  OS:                   {OS_VERSION}
  RAM:                  {TOTAL_RAM_MB}MB (Profile: {PROFILE})

── SSH ACCESS ─────────────────────────────────────────────────
  SSH Host:             {SERVER_IP}
  SSH Port:             {ACTIVE_SSH_PORT}
  SSH User:             {SSH_USER}

── WEB SERVER ─────────────────────────────────────────────────
  OpenLiteSpeed:        {OLS_VERSION}
  LSPHP:                8.2
  LSPHP Path:           /usr/local/lsws/lsphp82/bin/lsphp
  Config:               /usr/local/lsws/conf/httpd_config.conf
  WebAdmin:             disabled

── DATABASE ───────────────────────────────────────────────────
  MariaDB:              {MARIADB_VERSION}
  MariaDB Root Password:{MARIADB_ROOT_PASS}

── CACHING ────────────────────────────────────────────────────
  Redis:                active (maxmemory={REDIS_MEM}mb)
  OPcache:              {OPCACHE_MEM}MB, JIT=tracing

── SECURITY ───────────────────────────────────────────────────
  Firewall:             UFW active
    SSH:                port {ACTIVE_SSH_PORT} from {ADMIN_IP} only
    HTTP/HTTPS:         80+443 from Cloudflare IPs only
    Default:            deny all other
  fail2ban:             active (sshd + openlitespeed-auth)
  Direct IP Access:     blocked (returns 403)

── RESOURCE PROFILE ───────────────────────────────────────────
  LSAPI_CHILDREN base:  {LSAPI_BASE} per site
  MariaDB Buffer Pool:  {MARIADB_POOL}MB
  Redis maxmemory:      {REDIS_MEM}MB
  OPcache memory:       {OPCACHE_MEM}MB

── PATHS ──────────────────────────────────────────────────────
  OLS Root:             /usr/local/lsws/
  Vhosts Dir:           /usr/local/lsws/conf/vhosts/
  Cert Dir:             /usr/local/lsws/conf/cert/
  PHP.ini Dir:          /usr/local/lsws/conf/php-ini/
  Logs:                 /usr/local/lsws/logs/
  Backups:              /var/backups/wordpress/

═══════════════════════════════════════════════════════════════
  SAVE THE MARIADB ROOT PASSWORD — IT WILL NOT BE SHOWN AGAIN
═══════════════════════════════════════════════════════════════
```

## Todo List

- [x] Rewrite SKILL.md frontmatter + purpose section
- [x] Implement OS detection
- [x] Implement RAM detection + adaptive profile calculation
- [x] Implement optional sudo user creation
- [x] Implement optional SSH key auth + port change (with lockout safety)
- [x] Implement OLS + LSPHP 8.2 installation from LiteSpeed repo
- [x] Implement plain text config switch + WebAdmin disable
- [x] Write base httpd_config.conf with CF-Connecting-IP + listeners
- [x] Write default 403 vhost config
- [x] Implement MariaDB install + secure + adaptive tuning
- [x] Implement Redis install + configure (adaptive maxmemory)
- [x] Implement OPcache tuning (adaptive memory + JIT)
- [x] Implement UFW Cloudflare-only rules (auto-fetch CF IPs)
- [x] Implement fail2ban with OLS access log jail
- [x] Implement OLS log rotation
- [x] Create backup directory infrastructure
- [x] Validate with `openlitespeed -t` + verify all services
- [x] Output results with RAM profile to orchestrator

## Success Criteria

- `systemctl is-active lsws` returns `active`
- `/usr/local/lsws/bin/openlitespeed -t` passes with no errors
- `redis-cli ping` returns `PONG`
- `mysql -u root -p'{pass}' -e "SELECT 1"` succeeds
- UFW shows only SSH from admin IP + 80/443 from CF ranges
- `/usr/local/lsws/conf/disablewebconsole` exists
- Direct IP access to port 80/443 returns 403
- fail2ban shows 2 active jails

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| SSH lockout after port/key change | HIGH | Change UFW BEFORE sshd restart; only disable password if key confirmed |
| OLS config syntax error | HIGH | Always `openlitespeed -t` before `systemctl start lsws` |
| LiteSpeed repo unavailable | MEDIUM | Fallback: download OLS binary directly from litespeedtech.com |
| MariaDB already has root password | LOW | Detect and skip ALTER USER, report "existing password" |
| Cloudflare IP list changes | LOW | Auto-fetch at setup time; document manual refresh procedure |

## Security Considerations

- SSH key-only auth after key confirmed (optional, not forced)
- UFW default deny + Cloudflare-only whitelist
- MariaDB root password auto-generated, never saved to file
- WebAdmin console permanently disabled
- fail2ban protects SSH + OLS login brute force
- OLS runs as `nobody:nogroup` (least privilege)
- Redis bound to localhost only

## Next Steps

After this phase: Phase 2 (wordpress-site-setup) uses OLS + LSPHP + MariaDB root password + RAM profile from this phase.
