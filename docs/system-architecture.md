# System Architecture

## Overview

ReviewServerConfigAgent is a modular, skill-based system for both auditing and setting up WordPress domains. The architecture separates concerns by skill type (audit vs setup) and by infrastructure layer (VPS, Cloudflare, WordPress), with top-level orchestration for each workflow.

### Audit Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                   Domain Review Agent                        │
│              (Orchestrator, cross-layer validation)          │
└───────────────┬──────────────┬──────────────┬────────────────┘
                │              │              │
        ┌───────▼──────┐  ┌────▼─────────┐  ┌─▼──────────────┐
        │ VPS Audit    │  │ Cloudflare   │  │ WordPress      │
        │ Skill        │  │ Audit Skill  │  │ Audit Skill    │
        └───────┬──────┘  └────┬─────────┘  └─┬──────────────┘
                │              │              │
        ┌───────▼──────┐  ┌────▼─────────┐  ┌─▼──────────────┐
        │ SSH Tunnel   │  │ Cloudflare   │  │ WP-CLI Binary  │
        │ & Commands   │  │ API v4       │  │ Execution      │
        └──────────────┘  └──────────────┘  └────────────────┘
```

### Setup Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                   Domain Setup Agent                         │
│              (Orchestrator, sequential execution)            │
└───────────────┬──────────────┬──────────────┬────────────────┘
                │              │              │
        ┌───────▼──────┐  ┌────▼─────────┐  ┌─▼──────────────┐
        │ VPS Server   │  │ WordPress    │  │ Cloudflare     │
        │ Setup Skill  │  │ Site Setup   │  │ Domain Setup   │
        │              │  │ Skill        │  │ Skill          │
        └───────┬──────┘  └────┬─────────┘  └─┬──────────────┘
                │              │              │
        ┌───────▼──────┐  ┌────▼─────────┐  ┌─▼──────────────┐
        │ Install LEMP │  │ WP-CLI + WP  │  │ Zone + DNS +   │
        │ + firewall   │  │ + DB + config│  │ SSL + security │
        └──────────────┘  └──────────────┘  └────────────────┘
```

## Component Architecture

### 1. VPS Server Audit Skill

**Purpose:** Audit Linux server infrastructure and configuration.

**Transport:** SSH (paramiko or sshpass)

**Key Subsystems:**

| Subsystem | Commands | Output |
|-----------|----------|--------|
| **OS Detection** | `uname -a`, `lsb_release -a` | OS, kernel, distribution |
| **Security Updates** | `apt list --upgradable` | Pending updates count |
| **PHP Audit** | `php -v`, `php -i`, `php -r "phpinfo()"` | Version, extensions, limits |
| **MySQL Audit** | `mysql -e "SELECT VERSION()"`, user grants | Version, users, privileges |
| **Web Server** | Check `/etc/nginx`, `/etc/apache2`, `/etc/lsws` | Detected type + config analysis |
| **SSL Certificates** | `find /etc -name "*.crt"`, openssl validation | Certificate expiry, chain |
| **Firewall** | `iptables -L`, `ufw status`, check port 80/443 | Rules, open ports, IP restrictions |
| **Fail2ban** | `systemctl status fail2ban`, jail list | Installation status, active jails |
| **Open Ports** | `netstat -tlnp` or `ss -tlnp` | All listening ports and services |

**Error Handling:**
- SSH timeout: Retry 3x with 5s delay
- Command not found: Fallback to alternative detection method
- Permission denied: Log as warning (non-critical data)
- MySQL connection failure: Use `apt show mysql-server` instead

**Output:** VPS audit report with 0-10 health score

---

### 2. Cloudflare Domain Audit Skill

**Purpose:** Audit Cloudflare DNS, SSL/TLS, and security configuration.

**Transport:** Cloudflare API v4 (https://api.cloudflare.com/client/v4/)

**Authentication:** Bearer token (API token, not API key)

**Key Subsystems:**

| API Endpoint | Validation |
|--------------|-----------|
| **GET /zones** | Find domain zone ID |
| **GET /zones/:id/dns_records** | List all DNS records (A, CNAME, MX, TXT) |
| **GET /zones/:id/ssl/certificate_packs** | Validate SSL mode (Full, Full Strict, Flexible) |
| **GET /zones/:id/ssl/universal/settings** | Check HSTS, min TLS version |
| **GET /zones/:id/page_rules** | Review cache bypass rules (`/cart/*`, `/checkout/*`) |
| **GET /zones/:id/firewall/rules** | Count and analyze firewall rules |
| **GET /zones/:id/security_settings** | Check DDoS protection, rate limiting |
| **GET /zones/:id/performance** | Minification, caching, compression settings |

**Cross-Validation:**
- Compare DNS A record to VPS server IP
- Verify Cloudflare SSL mode matches origin server certificate
- Confirm cache rules align with WordPress URL structure

**Error Handling:**
- 401 Unauthorized: Invalid or expired API token
- 404 Not Found: Domain not on Cloudflare (re-check spelling)
- 429 Too Many Requests: Apply exponential backoff
- 500 Server Error: Retry after 30 seconds

**Output:** Cloudflare audit report with 0-10 health score

---

### 3. WordPress Site Audit Skill

**Purpose:** Audit WordPress installation, plugins, themes, users, and database integrity.

**Transport:** WP-CLI execution via SSH or local shell

**Execution Method:**
```bash
# Standard
wp --path=/var/www/html --allow-root

# OpenLiteSpeed/CyberPanel fallback (if system PHP lacks mysqli)
/usr/local/lsws/lsphp83/bin/php /usr/bin/wp --path=/var/www/html --allow-root
```

**Key Subsystems:**

| Subsystem | WP-CLI Commands | Validation |
|-----------|-----------------|-----------|
| **Core** | `wp core version`, `wp core verify-checksums` | Version, security patches, file integrity |
| **Plugins** | `wp plugin list --format=json` | All installed, versions, update status, vulnerability detection |
| **Themes** | `wp theme list --format=json` | All installed, versions, active theme |
| **Users** | `wp user list --format=json` | Accounts, roles, privileges, creation dates |
| **Database** | `wp db check`, `wp db tables --scope=all` | Integrity, table count, transients |
| **Options** | `wp option get siteurl`, `wp option get home` | Site URL consistency (HTTP/HTTPS) |
| **Sessions** | Query `wp_usermeta` for session_tokens | Active sessions, stale tokens |
| **Malware** | Grep for malicious patterns in `/wp-content/plugins`, `/wp-content/themes` | Known signatures, suspicious code |
| **Backups** | Check plugin metadata for backup capability | Backup plugin presence |
| **Transients** | `wp transient list` | Count of stale transients (cache bloat) |

**Plugin Vulnerability Detection:**
- Parse plugin header comments for version
- Cross-reference against known vulnerability databases (manual for now)
- Flag plugins 3+ minor versions behind latest

**Error Handling:**
- WP-CLI not found: Suggest installation or path adjustment
- Database connection failure: Check MySQL credentials and connectivity
- Permission denied: Escalate with --allow-root
- Timeout: Increase WP-CLI timeout or query specific subsystem

**Output:** WordPress audit report with 0-10 health score

---

### 4. Domain Review Agent (Orchestrator)

**Purpose:** Sequence all audits, cross-validate findings, and produce consolidated report.

**Workflow:**

```
1. User Input
   └─ Parse domain name, collect credentials (SSH, Cloudflare token)

2. Sequential Audit Execution
   ├─ Run VPS Server Audit
   ├─ Run Cloudflare Domain Audit
   └─ Run WordPress Site Audit

3. Cross-Layer Validation
   ├─ DNS A record ↔ VPS server IP
   ├─ SSL mode (Cloudflare) ↔ Origin certificate (VPS)
   ├─ Cache rules ↔ WordPress URLs
   ├─ HTTPS enforcement (Cloudflare) ↔ Site URL (WordPress)
   ├─ PHP limits ↔ Cloudflare upload limits
   ├─ Firewall rules ↔ Legitimate traffic patterns
   └─ Email MX records ↔ Email provider configuration

4. Finding Consolidation
   ├─ Merge duplicate findings across layers
   ├─ Assign severity (Critical / Warning / Recommendation)
   ├─ Calculate composite health score
   └─ Prioritize by severity + impact

5. Report Generation
   ├─ Write individual reports (VPS, Cloudflare, WordPress)
   ├─ Write combined domain review report
   └─ Save all to vps-reports/ directory
```

**Health Score Calculation:**

```
VPS Score (0-10):
  - Deduction: -2 per critical security issue
  - Deduction: -1 per warning
  - Baseline: 10 / (1 + total_deductions)

Cloudflare Score (0-10):
  - Deduction: -2 per missing security header
  - Deduction: -1 per configuration warning
  - Baseline: 10 / (1 + total_deductions)

WordPress Score (0-10):
  - Deduction: -2 per critical security issue (malware, etc.)
  - Deduction: -0.5 per outdated plugin
  - Baseline: 10 / (1 + total_deductions)

Overall Score:
  - Average of VPS, Cloudflare, WordPress scores
  - Weighted toward critical issues (2x weight)
```

**Finding Prioritization:**

1. **Critical Issues** (fix immediately)
   - Malware detected
   - SQL injection vulnerabilities
   - Missing SSL/HTTPS enforcement
   - Brute-force attack vectors unprotected
   - Expired certificates

2. **Warnings** (fix soon)
   - Outdated plugins/themes
   - Missing security headers
   - Weak PHP configuration limits
   - Unpatched system updates
   - Missing backups

3. **Recommendations** (improve when possible)
   - Performance optimizations
   - SEO enhancements
   - Best practice upgrades
   - Hardening beyond baseline security

**Output:** Domain review report (combined summary + links to individual reports)

---

## Setup Skills Architecture

### 1. VPS Server Setup Skill

**Purpose:** Install and configure LEMP stack (Linux, Nginx, PHP-FPM, MariaDB) with security hardening.

**Transport:** SSH (paramiko or sshpass)

**Key Operations:**

| Operation | Commands | Output |
|-----------|----------|--------|
| **OS Detection** | `uname -a`, `lsb_release -a` | Detect distribution, set package manager |
| **System Updates** | `apt update && apt upgrade -y` | Fresh, patched system base |
| **Nginx Install** | `apt install -y nginx`, enable/start | Web server running on 80/443 |
| **PHP-FPM Install** | `apt install -y php-fpm php-*`, configure | PHP 8.1+ with required extensions |
| **MariaDB Install** | `apt install -y mariadb-server`, secure | Database server, root password set |
| **Firewall Setup** | `ufw enable`, allow SSH/80/443 | UFW active, common ports open |
| **SSH Hardening** | Disable root login, change port (optional) | SSH secured against brute force |
| **fail2ban Setup** | Install and configure jails | Automatic IP banning for attacks |

**Credential Generation:**
- Auto-generate MySQL root password (openssl rand -base64 32)
- Return password to orchestrator only (never persisted)

**Idempotency:** Safe to re-run; skips already-installed packages

**Output:** VPS setup report with connection details and generated credentials

---

### 2. WordPress Site Setup Skill

**Purpose:** Install WordPress, configure database, and apply security hardening.

**Transport:** SSH + WP-CLI

**Key Operations:**

| Operation | Commands | Output |
|-----------|----------|--------|
| **WP-CLI Install** | Download/configure WP-CLI binary | WP-CLI available at `/usr/local/bin/wp` |
| **WordPress Download** | `wp core download`, verify checksums | Latest WordPress in `/var/www/html` |
| **Database Creation** | Create DB + user via MySQL | Auto-gen DB name + credentials |
| **wp-config.php** | Write config, set auth salts, disable edit | Security hardening applied |
| **File Permissions** | 755 for dirs, 644 for files, 600 for config | Correct ownership/permissions |
| **Nginx Server Block** | Create vhost config, SSL placeholder | Nginx ready for domain |
| **Security Rules** | Block XML-RPC, user enum, REST API abuses | Standard WP security patterns |

**Credential Generation:**
- Auto-gen database name: `wp_{domain}_{random}`
- Auto-gen database user + password
- Auto-gen WordPress admin user + password
- All credentials passed to orchestrator, stored in setup report only

**Output:** WordPress setup report with installation details and admin credentials

---

### 3. Cloudflare Domain Setup Skill

**Purpose:** Configure Cloudflare for the domain (zone, DNS, SSL, security).

**Transport:** Cloudflare API v4 (https://api.cloudflare.com/client/v4/)

**Key Operations:**

| API Endpoint | Operation | Result |
|--------------|-----------|--------|
| **POST /zones** | Create or detect zone | Zone ID for domain |
| **POST /zones/:id/dns_records** | Add A record | Point domain to server IP |
| **POST /zones/:id/dns_records** | Add CNAME for www | www subdomain proxied |
| **POST /client/v4/certificates** | Create Origin Certificate | 15-year self-signed cert (Cloudflare) |
| **PUT /zones/:id/ssl/certificate_packs** | Enable SSL Full Strict | Strict SSL mode active |
| **PUT /zones/:id/ssl/universal/settings** | Configure SSL/TLS | HSTS, min TLS 1.2, Brotli |
| **POST /zones/:id/page_rules** | Create cache bypass rules | `/wp-admin` + `/wp-login.php` bypass |
| **PUT /zones/:id/security_settings** | Set security defaults | DDoS protection, rate limiting |

**Origin Certificate Install:**
- Generate origin cert + key pair via Cloudflare API
- Deploy to server via SSH (via vps-server-setup)
- Update Nginx SSL config to use origin cert

**Credential Flow:**
- Cloudflare API token passed in
- Origin cert + key returned to orchestrator
- Cert deployed to server then cleared from memory

**Output:** Cloudflare setup report with DNS, SSL, and security settings

---

### 4. Domain Setup Agent (Orchestrator)

**Purpose:** Sequence all 3 setup skills, capture credentials, produce setup report.

**Workflow:**

```
1. User Input
   └─ Parse domain name, collect SSH + Cloudflare credentials

2. Sequential Setup Execution
   ├─ VPS Server Setup
   │   └─ Generate MySQL root password
   ├─ WordPress Site Setup
   │   ├─ Wait for DB (from step 1)
   │   └─ Generate DB + admin credentials
   └─ Cloudflare Domain Setup
       ├─ Wait for server IP + WordPress ready (from steps 1–2)
       ├─ Generate origin certificate
       └─ Deploy certificate to server via SSH

3. Credential Consolidation
   ├─ Collect all auto-gen passwords + usernames
   ├─ Collect Cloudflare origin cert details
   ├─ Capture API responses for manual steps
   └─ Format for delivery

4. Setup Report Generation
   ├─ Write setup instructions (domain, nameservers, etc.)
   ├─ List all auto-generated credentials
   ├─ Include manual follow-up steps (email config, SMTP, etc.)
   ├─ Provide verification checklist
   └─ Save to vps-reports/ directory

5. Cleanup
   ├─ Clear all credentials from memory
   ├─ Clear SSH connections
   └─ Confirm setup complete
```

**Credential Capture:**
- MySQL root password from VPS setup
- Database name, username, password from WordPress setup
- WordPress admin username, password from WordPress setup
- Cloudflare origin cert + key from Cloudflare setup
- All credentials returned in final setup report (single-session only)

**Error Handling:**
- Partial failure: Mark failed step, report skipped steps
- Rollback: Suggest deletion of VPS files + Cloudflare zone if user wants to retry
- Clear all credentials on error (before cleanup)

**Output:** Domain setup report with all credentials, manual steps, and verification checklist

---

## Data Flow

### Credential Handling

```
User Input
   ↓
Encrypt in memory (not persisted)
   ↓
Pass to skill inline
   ↓
Use for single audit cycle
   ↓
Clear from memory
```

**Security Properties:**
- No `.env` files created
- No credential files written
- Memory cleared after skill execution
- SSH keys passed via `paramiko` (no sshpass fallback that writes to filesystem)

---

### Report Storage

#### Audit Reports

```
vps-reports/
├── domain-review-{domain}-{YYYYMMDD}.md
│   ├── Executive Summary
│   ├── Health Dashboard (table)
│   ├── Cross-Reference Issues (validation grid)
│   ├── Critical Actions (prioritized list)
│   ├── Warnings (secondary issues)
│   ├── Recommendations (enhancements)
│   └── Links to individual reports
│
├── vps-audit-{domain}-{YYYYMMDD}.md
│   ├── OS & Kernel
│   ├── PHP Configuration
│   ├── MySQL/MariaDB
│   ├── Web Server
│   ├── SSL Certificates
│   ├── Firewall & Ports
│   └── fail2ban Status
│
├── cloudflare-audit-{domain}-{YYYYMMDD}.md
│   ├── DNS Records
│   ├── SSL/TLS Configuration
│   ├── Page Rules
│   ├── Firewall Rules
│   ├── Security Settings
│   └── Performance Settings
│
└── wordpress-audit-{domain}-{YYYYMMDD}.md
    ├── WordPress Core
    ├── Plugins (with vulnerabilities)
    ├── Themes
    ├── Users & Roles
    ├── Database Integrity
    ├── Malware Scan
    └── Backup Status
```

#### Setup Reports

```
vps-reports/
├── domain-setup-{domain}-{YYYYMMDD}.md
│   ├── Setup Overview (domain, dates, status)
│   ├── VPS Server Setup
│   │   ├─ Installed packages (Nginx, PHP-FPM, MariaDB, firewall)
│   │   ├─ MySQL root password (auto-generated)
│   │   └─ Connection details
│   ├── WordPress Site Setup
│   │   ├─ Database credentials (auto-generated)
│   │   ├─ WordPress admin credentials (auto-generated)
│   │   └─ Installation path
│   ├── Cloudflare Domain Setup
│   │   ├─ Zone ID
│   │   ├─ DNS records (A + www CNAME)
│   │   ├─ SSL Mode (Full Strict)
│   │   └─ Security settings
│   ├── Manual Follow-Up Steps
│   │   ├─ Update nameservers
│   │   ├─ Configure email/SMTP
│   │   ├─ Enable WordPress updates
│   │   └─ Configure backups
│   ├── Verification Checklist
│   │   ├─ Check DNS propagation
│   │   ├─ Test HTTPS redirect
│   │   ├─ Verify WordPress login
│   │   └─ Test email delivery
│   └── Credential Summary (all generated passwords)
```

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **SSH Transport** | Python `paramiko` or `sshpass` | VPS server connection |
| **API Client** | `requests` / curl | Cloudflare API calls |
| **CLI Execution** | WP-CLI (Bash/PHP) | WordPress data extraction |
| **Markdown Generation** | Template + string building | Report formatting |
| **Orchestration** | Claude Agent (multi-turn) | Workflow coordination |

---

## Error Handling & Resilience

### Transient Failures
- **SSH timeout:** Retry 3x with 5-second exponential backoff
- **API rate limit:** Wait 60s, retry once
- **WP-CLI timeout:** Reduce query scope or query subsystems individually

### Fatal Failures
- **Wrong credentials:** Fail fast with clear error message
- **Domain not found:** Verify domain spelling, check if on Cloudflare
- **WP-CLI not installed:** Suggest installation or remote execution

### Partial Data
- If VPS audit fails: Continue with Cloudflare and WordPress
- If Cloudflare API fails: Continue with VPS and WordPress (note DNS validation unavailable)
- If WordPress audit fails: Continue with VPS and Cloudflare (note security score incomplete)
- Generate report with "partial audit" flag

---

## Performance Characteristics

| Audit | Typical Duration | Factors |
|-------|------------------|---------|
| VPS Server | 1–2 min | SSH latency, disk I/O for log scanning |
| Cloudflare | 30–60 sec | API latency, number of DNS records |
| WordPress | 2–3 min | Site size, number of plugins, DB queries |
| **Total** | **4–6 min** | Network, server load, complexity |

**Optimization opportunities:**
- Batch API calls (Cloudflare)
- Parallel subsystem queries (WP-CLI)
- Cache known server configurations

---

## Scalability Considerations

### Current (V1.0)
- Single domain per run
- Credential handling: single-session
- Report storage: local filesystem

### Phase 2+
- Batch queue for multiple domains
- Persistent credential vault (encrypted)
- Database storage for audit history
- Async job processing

---

**Last Updated:** 2026-03-19
**Version:** 1.1
