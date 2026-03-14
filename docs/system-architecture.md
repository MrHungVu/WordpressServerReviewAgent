# System Architecture

## Overview

ReviewServerConfigAgent is a modular, skill-based audit system that performs multi-layer WordPress domain reviews. The architecture separates concerns by audit layer (VPS, Cloudflare, WordPress) and a top-level orchestration layer.

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

**Last Updated:** 2026-03-13
**Version:** 1.0 Architecture
