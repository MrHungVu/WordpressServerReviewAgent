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

# VPS Server Setup Skill (OLS v2)

## Purpose
Set up a fresh Linux VPS with OpenLiteSpeed + LSPHP 8.2 + MariaDB + Redis for WordPress hosting. Replaces Nginx/LEMP stack. Idempotent — safe to re-run on partial setups and safe to run on a VPS that already has OLS (adding a second site, for example).

## Required Input (from user/orchestrator, NEVER save to files)
- SSH host (IP address)
- SSH password
- SSH user (default: root)
- SSH port (default: 22)
- Domain name (for context/report — first site)
- New sudo username (optional — skip user creation if empty)
- SSH public key (optional — skip key auth setup if empty)
- Custom SSH port (optional — skip port change if empty)

## SSH Connection
Use `sshpass` for password-based SSH. All commands run remotely via:
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

If `sshpass` not installed locally:
- macOS: `brew install hudochenkov/sshpass/sshpass`
- Linux: `apt install sshpass -y`

## Setup Checklist

### 1. Detect OS + Existing Stack

Detect OS family:
```bash
cat /etc/os-release
```
- Debian/Ubuntu → `PKG_MGR=apt`
- CentOS/RHEL/AlmaLinux → `PKG_MGR=yum`
- Store detected distro name and version for repo compatibility

**Idempotent stack detection — run this BEFORE any installs:**
```bash
OLS_INSTALLED=0; MARIADB_INSTALLED=0; REDIS_INSTALLED=0
systemctl is-active lsws &>/dev/null && OLS_INSTALLED=1
systemctl is-active mariadb &>/dev/null && MARIADB_INSTALLED=1
systemctl is-active redis-server &>/dev/null && REDIS_INSTALLED=1
echo "OLS=$OLS_INSTALLED MARIADB=$MARIADB_INSTALLED REDIS=$REDIS_INSTALLED"
```

Decision logic:
- **All three active:** Skip install steps 5–9. Detect existing LSPHP version. Detect RAM profile. Ask user for existing MariaDB root password. Proceed to step 10 (UFW) and output existing config summary.
- **Some active:** Continue — each component below has its own check-before-install guard.
- **None active:** Full fresh install.

### 2. System Update

Wait for apt lock (Ubuntu/Debian only), then update:
```bash
while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 2; done
apt update && apt upgrade -y
# CentOS/RHEL:
# yum update -y
```

### 3. Detect RAM + Set Adaptive Memory Profile

```bash
TOTAL_RAM_MB=$(free -m | awk '/^Mem:/{print $2}')
echo "Total RAM: ${TOTAL_RAM_MB}MB"
```

Compute adaptive profile based on detected RAM:

| Condition | PROFILE | LSAPI_BASE | OPCACHE_MEM | MARIADB_POOL | REDIS_MEM |
|-----------|---------|------------|-------------|--------------|-----------|
| RAM <= 2500 MB | 2GB | 8 | 128 | 350 | 200 |
| RAM <= 5000 MB | 4GB | 12 | 256 | 768 | 256 |
| RAM > 5000 MB | 8GB+ | 20 | 512 | 2048 | 512 |

Store these as shell variables (PROFILE, LSAPI_BASE, OPCACHE_MEM, MARIADB_POOL, REDIS_MEM) and output the profile to conversation for the orchestrator.

### 4. Create Sudo User (optional)

Only if `NEW_SUDO_USER` was provided — skip entirely if empty:
```bash
id ${NEW_SUDO_USER} &>/dev/null || useradd -m -s /bin/bash ${NEW_SUDO_USER}
usermod -aG sudo ${NEW_SUDO_USER}
SUDO_PASS=$(openssl rand -base64 18)
echo "${NEW_SUDO_USER}:${SUDO_PASS}" | chpasswd
echo "Sudo user created: ${NEW_SUDO_USER}"
```
Record generated sudo user password for output section.

### 5. Install OpenLiteSpeed + LSPHP 8.2

Skip if `OLS_INSTALLED=1` (detected in step 1).

Stop Apache if running (conflicts on ports 80/443):
```bash
systemctl stop apache2 2>/dev/null; systemctl disable apache2 2>/dev/null
systemctl stop httpd 2>/dev/null; systemctl disable httpd 2>/dev/null
```

Add LiteSpeed repo and install:
```bash
# Add official LiteSpeed repo
wget -O - https://repo.litespeed.sh | bash

# Install OpenLiteSpeed
apt install openlitespeed -y
# CentOS: yum install openlitespeed -y

# Install LSPHP 8.2 + all required WordPress extensions
apt install lsphp82 lsphp82-common lsphp82-mysql lsphp82-curl lsphp82-gd \
  lsphp82-mbstring lsphp82-xml lsphp82-zip lsphp82-imagick lsphp82-intl \
  lsphp82-bcmath lsphp82-redis lsphp82-opcache -y
# CentOS: yum install lsphp82 lsphp82-common lsphp82-mysqlnd ... -y
```

### 6. Plain Text Config + Disable WebAdmin

```bash
# Disable WebAdmin console permanently
touch /usr/local/lsws/conf/disablewebconsole

# Create required directories
mkdir -p /usr/local/lsws/conf/cert
mkdir -p /usr/local/lsws/conf/vhosts
mkdir -p /usr/local/lsws/conf/php-ini
mkdir -p /usr/local/lsws/tmp
```

### 7. Configure Base httpd_config.conf

Fetch Cloudflare IP ranges to populate the trusted IPs list:
```bash
CF_IPV4=$(curl -s https://www.cloudflare.com/ips-v4)
CF_IPV6=$(curl -s https://www.cloudflare.com/ips-v6)
```

Build the access control allow lines by formatting each IP range on its own `allow` line, then write `/usr/local/lsws/conf/httpd_config.conf`:

```
serverName                ProductionServer
user                      www-data
group                     www-data
autoRestart               1
priority                  0

useClientIpInHeader       2
clientIpHeader            CF-Connecting-IP

accessControl {
  allow                   {CF_IPV4_RANGE_1}
  allow                   {CF_IPV4_RANGE_2}
  # ... all fetched CF IPv4 ranges ...
  allow                   {CF_IPV6_RANGE_1}
  # ... all fetched CF IPv6 ranges ...
}

listener Default {
  address                 *:80
  secure                  0
}

listener HTTPS {
  address                 *:443
  secure                  1
  sslProtocol             24  # bitmask: TLSv1.2 (8) + TLSv1.3 (16) = 24
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

# Per-site vhost includes are added by wordpress-site-setup
# include /usr/local/lsws/conf/vhosts/{domain}.conf
```

Write the default 403 vhost to `/usr/local/lsws/conf/vhosts/_default.conf`:
```
docRoot                   /var/www/html

rewrite {
  enable                  1
  rules                   <<<END_rules
RewriteRule .* - [F,L]
  END_rules
}
```

### 8. SSH Key Auth (optional)

Only if `SSH_PUBLIC_KEY` was provided — skip entirely if empty:
```bash
TARGET_USER=${NEW_SUDO_USER:-root}
TARGET_HOME=$(getent passwd ${TARGET_USER} | cut -d: -f6)
mkdir -p ${TARGET_HOME}/.ssh
echo "${SSH_PUBLIC_KEY}" >> ${TARGET_HOME}/.ssh/authorized_keys
chmod 700 ${TARGET_HOME}/.ssh
chmod 600 ${TARGET_HOME}/.ssh/authorized_keys
chown -R ${TARGET_USER}:${TARGET_USER} ${TARGET_HOME}/.ssh
```

Enable public key auth in sshd_config (verify it's enabled before disabling password):
```bash
grep -q "^PubkeyAuthentication yes" /etc/ssh/sshd_config \
  || echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config
```

### 9. SSH Port Change (optional)

Only if `CUSTOM_SSH_PORT` was provided — skip entirely if empty.

**CRITICAL ORDER:** Update UFW rule BEFORE restarting sshd to prevent lockout. Do step 10 (UFW) before step 9b (sshd restart).

```bash
# 9a. Change sshd port in config
sed -i "s/^#*Port .*/Port ${CUSTOM_SSH_PORT}/" /etc/ssh/sshd_config
grep -q "^Port " /etc/ssh/sshd_config || echo "Port ${CUSTOM_SSH_PORT}" >> /etc/ssh/sshd_config
# sshd restart happens in step 11, AFTER UFW is updated
```

### 10. SSH Hardening

```bash
# MaxAuthTries (always applied)
sed -i 's/^#*MaxAuthTries.*/MaxAuthTries 5/' /etc/ssh/sshd_config
grep -q "^MaxAuthTries" /etc/ssh/sshd_config || echo "MaxAuthTries 5" >> /etc/ssh/sshd_config
```

Only if SSH key was installed (step 8 ran):
```bash
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
```

**DO NOT disable PasswordAuthentication unless SSH key was installed and step 8 completed successfully.**

### 11. Set Timezone UTC

```bash
timedatectl set-timezone UTC
```

### 12. Install + Secure MariaDB

Skip if `MARIADB_INSTALLED=1` (detected in step 1).

```bash
apt install mariadb-server -y
# CentOS: yum install mariadb-server -y
systemctl enable mariadb && systemctl start mariadb
```

Secure MariaDB with auto-generated root password:
```bash
MARIADB_ROOT_PASS=$(openssl rand -base64 24)
mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '${MARIADB_ROOT_PASS}';"
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "DELETE FROM mysql.user WHERE User='';"
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "DROP DATABASE IF EXISTS test;"
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%';"
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "FLUSH PRIVILEGES;"
```

If ALTER USER fails (password already set from previous partial run), report "existing MariaDB root password detected — skipping ALTER USER" and ask user to supply the existing password.

Tune `/etc/mysql/mariadb.conf.d/50-server.cnf` with adaptive values (append under `[mysqld]`):
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

### 13. Install + Configure Redis

Skip if `REDIS_INSTALLED=1` (detected in step 1).

```bash
apt install redis-server -y
# CentOS: yum install redis -y
```

Configure Redis for caching use (localhost-only, adaptive maxmemory, LRU eviction):
```bash
# Bind to localhost only
sed -i "s/^bind .*/bind 127.0.0.1 ::1/" /etc/redis/redis.conf

# Set adaptive maxmemory (use the REDIS_MEM value from step 3)
sed -i "s/^# maxmemory .*/maxmemory ${REDIS_MEM}mb/" /etc/redis/redis.conf
# If the line exists without comment:
sed -i "s/^maxmemory .*/maxmemory ${REDIS_MEM}mb/" /etc/redis/redis.conf

# Set eviction policy
sed -i "s/^# maxmemory-policy .*/maxmemory-policy allkeys-lru/" /etc/redis/redis.conf
sed -i "s/^maxmemory-policy .*/maxmemory-policy allkeys-lru/" /etc/redis/redis.conf
```
```bash
systemctl enable redis-server && systemctl restart redis-server
```

### 14. Tune LSPHP OPcache

Write `/usr/local/lsws/lsphp82/etc/php/8.2/mods-available/opcache.ini` with adaptive memory values:
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

### 15. Configure UFW (Cloudflare-only firewall)

```bash
command -v ufw &>/dev/null || apt install ufw -y

# Reset to clean defaults
ufw --force reset
ufw default deny incoming
ufw default allow outgoing

# Determine active SSH port (custom if changed, otherwise current config or default 22)
ACTIVE_SSH_PORT=${CUSTOM_SSH_PORT:-$(grep -E "^Port " /etc/ssh/sshd_config | awk '{print $2}')}
ACTIVE_SSH_PORT=${ACTIVE_SSH_PORT:-22}

# Allow SSH from any IP (dynamic IP users — protected by fail2ban + key auth + MaxAuthTries)
ufw allow ${ACTIVE_SSH_PORT}/tcp

# Allow 80 and 443 from Cloudflare IPs only
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

CentOS (firewalld) equivalent:
```bash
firewall-cmd --permanent --add-port=${ACTIVE_SSH_PORT}/tcp
# Add each CF IP range for 80/443, then:
firewall-cmd --reload
```

### 16. Restart SSH (apply config changes)

Run this AFTER UFW is updated (step 15) to avoid lockout:
```bash
systemctl reload sshd
```

### 17. Install + Configure fail2ban

```bash
apt install fail2ban -y
# CentOS: yum install fail2ban -y
```

Write `/etc/fail2ban/jail.local`:
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

Write `/etc/fail2ban/filter.d/openlitespeed-auth.conf`:
```ini
[Definition]
failregex = ^<HOST> .* "(GET|POST) .*/wp-login\.php.*" (200|302)
ignoreregex =
```

```bash
systemctl enable fail2ban && systemctl restart fail2ban
```

### 18. OLS Log Rotation

Write `/etc/logrotate.d/openlitespeed`:
```
/usr/local/lsws/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 640 www-data www-data
    sharedscripts
    postrotate
        /usr/local/lsws/bin/lswsctrl restart > /dev/null 2>&1 || true
    endscript
}
```

### 19. Backup Infrastructure

```bash
mkdir -p /var/backups/wordpress
```

### 20. Validate OLS Config + Start Services

Validate config syntax BEFORE starting OLS:
```bash
/usr/local/lsws/bin/openlitespeed -t
```

If validation passes, enable and start:
```bash
systemctl enable lsws
systemctl start lsws
```

Verify all services are active:
```bash
systemctl is-active lsws
/usr/local/lsws/lsphp82/bin/lsphp -v
systemctl is-active mariadb
mysql -u root -p"${MARIADB_ROOT_PASS}" -e "SELECT 1;" 2>&1
redis-cli ping
systemctl is-active redis-server
ufw status numbered
fail2ban-client status
```

## Output

Output ALL information in copy-friendly format so user can save credentials before they are lost. The MariaDB root password is never stored to a file — this output is the only place it appears.

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
  Redis:                active (maxmemory={REDIS_MEM}mb, allkeys-lru)
  OPcache:              {OPCACHE_MEM}MB, JIT=tracing

── SECURITY ───────────────────────────────────────────────────
  Firewall:             UFW active
    SSH:                port {ACTIVE_SSH_PORT} open (protected by fail2ban + key auth)
    HTTP/HTTPS:         80+443 from Cloudflare IPs only
    Default:            deny all other
  fail2ban:             active (sshd + openlitespeed-auth jails)
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

Also pass to orchestrator (for Phase 2 handoff):
- MariaDB root password
- RAM profile (PROFILE, LSAPI_BASE, OPCACHE_MEM, MARIADB_POOL, REDIS_MEM)
- Active SSH port
- OLS version
- LSPHP path: `/usr/local/lsws/lsphp82/bin/lsphp`

## Error Handling

- If any package install fails, log the error and continue with remaining steps — never abort the entire setup
- If LiteSpeed repo is unreachable, fallback: download OLS binary directly from `litespeedtech.com/open-source/openlitespeed/download`
- If `openlitespeed -t` config validation fails, show the error output and stop — do NOT start OLS with a broken config
- If MariaDB already has a root password (ALTER USER fails), report "existing MariaDB root password detected" and ask user for the existing password
- If Cloudflare IP fetch fails, warn and allow user to confirm before opening 80/443 globally (temporary fallback only)
- If a service fails to start after setup, show `systemctl status {service}` and `journalctl -u {service} -n 20` output
- If Apache is found running, stop+disable it before installing OLS and warn in output

## Important Rules

- NEVER save SSH credentials or MariaDB passwords to any file
- Run ALL commands via SSH remotely using `sshpass`, not locally
- Every step must check-before-act (idempotent — safe on partial setups)
- NEVER start OLS without first running `openlitespeed -t` config validation
- NEVER disable PasswordAuthentication unless SSH key auth was installed in step 8
- ALWAYS update UFW BEFORE restarting sshd when changing SSH port (lockout prevention)
- Output the MariaDB root password and RAM profile to the conversation — orchestrator needs both for Phase 2
- Wait for apt lock before running apt commands on Debian/Ubuntu
