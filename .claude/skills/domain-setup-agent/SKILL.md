---
name: domain-setup-agent
description: Orchestrating agent that runs vps-server-setup ONCE then loops wordpress-site-setup and cloudflare-domain-setup per domain. Supports multi-site provisioning on a single VPS — collects a domain list, admin IP, optional SSH key/port, per-site login slugs, computes adaptive LSAPI_CHILDREN based on RAM profile and site count, handles partial failures, and produces a consolidated credential report. Use when user says "setup domain", "setup domains [list]", "install WordPress", or wants automated OLS + WordPress + Cloudflare multi-site setup.
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

# Domain Setup Agent (v2 — Multi-Site)

## Purpose
Orchestrate all 3 setup skills to deliver one or more complete WordPress sites from bare VPS to live domains. vps-server-setup runs ONCE (OLS foundation), then wordpress-site-setup + cloudflare-domain-setup run per domain. This is the single entry point for new site provisioning.

Maximum 16 sites per VPS (Redis DB 0–15 limit). Warn user and stop if domain list exceeds 16.

## Trigger Phrases
- "setup domain [domain]"
- "setup domains [domain1, domain2, ...]"
- "install WordPress on [domain]"
- "set up [domain]"
- "configure [domain]"
- "deploy WordPress to [domain]"
- "provision [domain]"

---

## Step 1: Collect Credentials

Ask user for ALL of the following via AskUserQuestion tool. NEVER save any credentials to files.

### Required
1. **Domain list** — comma-separated or one per line (e.g., site1.com, site2.com)
2. **SSH host** — VPS IP address
3. **SSH user** — default: root
4. **SSH password**
5. **SSH port** — default: 22
6. **Cloudflare email**
7. **Cloudflare Global API Key**
8. **WordPress admin email** — shared across all sites

### Conditional
10. **MariaDB root password** — ask ONLY if VPS already has OLS + MariaDB installed (vps-server-setup detects existing install and skips re-install; in that case the orchestrator needs the existing root password to pass downstream)

### Optional
11. **SSH public key** — for key-based auth setup (paste full public key or leave blank)
12. **Custom SSH port** — port to change SSH to (e.g., 2222); leave blank to keep current port
13. **Custom login slug per site** — e.g., `site1.com=my-login, site2.com=admin-portal`; leave blank to auto-generate `login-{hex4}` per site

---

## Step 2: Verify Connectivity

Test both connections before running any setup. Do NOT proceed until both pass.

```bash
# Test SSH
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p PORT USER@HOST 'echo "SSH OK"'

# Test Cloudflare API
curl -s "https://api.cloudflare.com/client/v4/user" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.success'
```

If `sshpass` not available: `brew install hudochenkov/sshpass/sshpass` (macOS) or `apt install sshpass`.
If `jq` not available: `brew install jq` (macOS) or `apt install jq`.

If SSH fails: ask user to verify host/user/password/port. If CF API returns `false`: ask user to verify email and API key. Do NOT continue until both return success.

---

## Step 3: Activate vps-server-setup (ONCE)

Run once for the entire VPS. Pass: SSH credentials, first domain (for hostname context), optional SSH public key, optional custom SSH port.

```
Activate skill: vps-server-setup
Pass:
  - SSH creds (host, user, password, port)
  - primary_domain = domain_list[0]
  - ssh_public_key  (optional)
  - custom_ssh_port (optional)
```

Capture from output — these values are required for all downstream steps:

| Variable | Description |
|----------|-------------|
| `SERVER_IP` | VPS public IP address |
| `RAM_PROFILE` | "2GB", "4GB", or "8GB+" |
| `LSAPI_BASE` | Base LSAPI children value set by vps-server-setup |
| `MARIADB_ROOT_PASS` | MariaDB root password (generated or confirmed existing) |
| `LSPHP_PATH` | Path to lsphp binary (e.g., /usr/local/lsws/lsphp82/bin/lsphp) |
| `OLS_VERSION` | OpenLiteSpeed version string |
| `ACTIVE_SSH_PORT` | Effective SSH port after setup (may differ if custom port was set) |

If vps-server-setup fails: stop, report error, set Status to FAILED. Do NOT attempt per-site setup.

---

## Step 4: Compute Per-Site Resources

Calculate resource allocation based on RAM profile and number of sites:

```
num_sites = len(domain_list)

# Enforce maximum
if num_sites > 16:
  STOP — warn user: "Maximum 16 sites per VPS (Redis DB 0–15 limit). Please reduce domain list."

# LSAPI workers per site
LSAPI_CHILDREN = max(4, floor(LSAPI_BASE / num_sites))

# Warn if low
if LSAPI_CHILDREN < 6:
  Warn: "LSAPI_CHILDREN = {LSAPI_CHILDREN} (low for {num_sites} sites — performance may be limited)"

# PHP memory limits per LSPHP ext app
if RAM_PROFILE == "2GB":
  PHP_MEM_SOFT = "350M"
  PHP_MEM_HARD = "400M"
  REDIS_MAXMEM = "64"    # MB

if RAM_PROFILE == "4GB":
  PHP_MEM_SOFT = "500M"
  PHP_MEM_HARD = "600M"
  REDIS_MAXMEM = "128"

if RAM_PROFILE == "8GB+":
  PHP_MEM_SOFT = "800M"
  PHP_MEM_HARD = "1000M"
  REDIS_MAXMEM = "256"
```

---

## Step 5: Per-Site Setup Loop

Run sequentially. For each site: WP setup first, then CF setup, then smoke test.

```
for site_number, domain in enumerate(domain_list):   # site_number starts at 0

  # Resolve login slug
  login_slug = user_provided_slug[domain] if provided else "login-" + random_hex(4)

  # ── 5a. Activate wordpress-site-setup ──────────────────────────────
  Activate skill: wordpress-site-setup
  Pass:
    - SSH creds (host, user, password, ACTIVE_SSH_PORT)
    - domain
    - mariadb_root_pass = MARIADB_ROOT_PASS
    - admin_email
    - site_number          (integer, determines Redis DB assignment: site0→DB0, site1→DB1, ...)
    - lsapi_children       = LSAPI_CHILDREN
    - ram_profile          = RAM_PROFILE
    - login_slug
    - lsphp_path           = LSPHP_PATH
    - php_mem_soft         = PHP_MEM_SOFT
    - php_mem_hard         = PHP_MEM_HARD

  Capture:
    - WP_ADMIN_USER
    - WP_ADMIN_PASS
    - DB_NAME
    - DB_USER
    - DB_PASS
    - DB_PREFIX
    - REDIS_DB  (should equal site_number)
    - REDIS_PREFIX  (slug derived from domain)
    - BACKUP_SCRIPT path

  If wordpress-site-setup fails:
    → log_error(domain, error_message)
    → mark site as FAILED
    → SKIP Step 5b and 5c for this domain
    → continue loop to next domain

  # ── 5b. Activate cloudflare-domain-setup ───────────────────────────
  Activate skill: cloudflare-domain-setup
  Pass:
    - CF creds (email, api_key)
    - domain
    - server_ip = SERVER_IP
    - SSH creds (for origin certificate installation on VPS)

  Capture:
    - CF_ZONE_ID
    - CF_NS1
    - CF_NS2
    - SSL_MODE  (should be "Full (Strict)")
    - ORIGIN_CERT_PATH

  If cloudflare-domain-setup fails:
    → log_error(domain, "CF setup failed: " + error_message)
    → mark site as PARTIAL (WP installed, CF not configured)
    → note in report: user must set up Cloudflare manually
    → continue loop to next domain

  # ── 5c. Smoke test ──────────────────────────────────────────────────
  Run:
    curl -sk -H "Host: {domain}" https://{SERVER_IP}/ | head -5

  Check: response contains HTML or HTTP 200/301
  If smoke test fails: log warning but do NOT mark site as FAILED (DNS not propagated yet — expected)
  Record SMOKE_TEST result as "PASS" or "PENDING (DNS not propagated)"

  # ── Record per-site result ──────────────────────────────────────────
  Store all captured values for report generation (Step 6)
```

---

## Step 6: Generate Final Report

Save to: `PROJECT_ROOT/vps-reports/domain-setup-{YYYYMMDD}.md`

Display full report in conversation AND save to file. Use plaintext format so user can copy the entire block.

```
═══════════════════════════════════════════════════════════════════
  MULTI-SITE DOMAIN SETUP REPORT
  Date: YYYY-MM-DD HH:MM UTC
  Status: COMPLETE / PARTIAL ({N}/{total} sites)
  COPY AND SAVE ALL INFORMATION BELOW IMMEDIATELY
═══════════════════════════════════════════════════════════════════

══ VPS SERVER ═════════════════════════════════════════════════════
  Server IP:              {SERVER_IP}
  SSH Port:               {ACTIVE_SSH_PORT}
  SSH User:               {ssh_user}
  OS:                     {os_version}
  RAM:                    {total_ram}MB (Profile: {RAM_PROFILE})
  OpenLiteSpeed:          {OLS_VERSION}
  LSPHP:                  8.2 ({LSPHP_PATH})
  MariaDB:                {mariadb_version}
  MariaDB Root Password:  {MARIADB_ROOT_PASS}
  Redis:                  active (maxmemory={REDIS_MAXMEM}mb)
  OPcache:                {opcache_mem}MB, JIT=tracing
  Firewall:               UFW (SSH open + fail2ban, 80+443 CF only)
  fail2ban:               active (sshd + openlitespeed-auth)
  WebAdmin:               disabled
  Config:                 /usr/local/lsws/conf/httpd_config.conf
  LSAPI Workers/Site:     {LSAPI_CHILDREN}

══ SITE {N}: {domain} [COMPLETE] ══════════════════════════════════

  ── LOGIN ────────────────────────────────────────────────────
  Site URL:               https://{domain}
  Admin Login URL:        https://{domain}/{login_slug}
  Admin Username:         {WP_ADMIN_USER}
  Admin Password:         {WP_ADMIN_PASS}
  Admin Email:            {admin_email}

  ── DATABASE ─────────────────────────────────────────────────
  DB Name:                {DB_NAME}
  DB User:                {DB_USER}
  DB Password:            {DB_PASS}
  DB Prefix:              {DB_PREFIX}

  ── SERVER CONFIG ────────────────────────────────────────────
  Install Path:           /var/www/{domain}/html
  OLS Vhost:              /usr/local/lsws/conf/vhosts/{domain}.conf
  LSPHP Workers:          {LSAPI_CHILDREN}
  Redis DB:               {REDIS_DB}
  Redis Prefix:           {REDIS_PREFIX}:
  Backup Script:          {BACKUP_SCRIPT}
  Backup Schedule:        Daily at 02:00 UTC

  ── CLOUDFLARE ───────────────────────────────────────────────
  CF Zone ID:             {CF_ZONE_ID}
  CF Nameserver 1:        {CF_NS1}
  CF Nameserver 2:        {CF_NS2}
  SSL Mode:               Full (Strict)
  Origin Cert:            /usr/local/lsws/conf/cert/{domain}.crt

══ SITE {N}: {domain} [PARTIAL — CF NOT CONFIGURED] ══════════════
  WP installed but Cloudflare setup failed.
  Error: {error_message}
  Action: Re-run "setup domain {domain}" to retry CF setup only.
  Manual CF: Point {domain} A record → {SERVER_IP} and configure SSL.

══ SITE {N}: {domain} [FAILED] ════════════════════════════════════
  Error: {error_message}
  Action: Re-run "setup domain {domain}" to retry from scratch.

══ SECURITY APPLIED (ALL SITES) ══════════════════════════════════
  [x] OLS with WebAdmin disabled
  [x] Cloudflare-only firewall (no direct IP access to 80/443)
  [x] Custom login URLs per site (/wp-login.php blocked)
  [x] SSL Full Strict with Cloudflare origin certificates
  [x] Redis object cache (isolated DB per site)
  [x] LSCache + WooCommerce cache exclusions
  [x] fail2ban (SSH + OLS brute-force jails)
  [x] Per-site backup scripts with 30-day retention
  [x] DB credentials in .my.cnf (not plaintext scripts)
  [x] Non-default DB prefix per site

══ ACTION REQUIRED ════════════════════════════════════════════════
  1. Change nameservers at your domain registrar:
     {domain1} → {CF_NS1}, {CF_NS2}
     {domain2} → {CF_NS1}, {CF_NS2}
  2. Wait 24–48 hours for DNS propagation
  3. Test each site: visit https://{domain}/
  4. Log in and change admin passwords immediately
  5. Run "review domain {domain}" to verify each site

══ POST-SETUP RECOMMENDATIONS ═════════════════════════════════════
  [ ] Configure SMTP for email delivery (per site)
  [ ] Install security monitoring plugin (Wordfence or equivalent)
  [ ] Test backup restore: run {BACKUP_SCRIPT}
  [ ] Set up uptime monitoring (UptimeRobot etc.)
  [ ] Review Cloudflare WAF rules for false positives

═══════════════════════════════════════════════════════════════════
  THIS REPORT IS SAVED TO: vps-reports/domain-setup-{YYYYMMDD}.md
  CREDENTIALS WILL NOT BE SHOWN AGAIN — SAVE NOW
═══════════════════════════════════════════════════════════════════
```

Report filename format: `domain-setup-{YYYYMMDD}.md` (no domain name in filename for multi-site).

---

## Partial Failure Handling

| Failure Point | Action |
|---------------|--------|
| vps-server-setup fails | STOP entire flow. Report error. No per-site setup attempted. |
| wordpress-site-setup fails for site N | Mark site N as FAILED. Skip CF setup for site N. Continue with site N+1. |
| cloudflare-domain-setup fails for site N | Mark site N as PARTIAL (WP installed). Log error. Continue with site N+1. |
| Smoke test fails for site N | Log as PENDING (DNS not propagated — expected). Do NOT mark as FAILED. |
| All sites fail | Report FAILED status. Suggest re-running after diagnosing root cause. |

Always produce a report with whatever data was captured before the failure.

---

## Important Rules

- Credentials exist only in conversation context. NEVER write credentials to files other than the final report.
- The setup report is a one-time credential delivery — warn user explicitly to save it immediately.
- The report file is saved to `vps-reports/` (already .gitignore'd) — user should delete after saving credentials.
- vps-server-setup runs ONCE regardless of site count. Do not invoke it per domain.
- Sequential execution within each site: wordpress-site-setup BEFORE cloudflare-domain-setup.
- Redis DB assignment is deterministic: site_number maps directly to Redis DB index (site0→DB0, site1→DB1...).
- LSAPI_CHILDREN minimum is 4. Never pass a lower value to wordpress-site-setup.
- Maximum 16 sites per VPS. Validate before starting setup loop.
- After setup completes, suggest running "review domain {domain}" for each successfully provisioned site.
- Health status: COMPLETE = vps + all sites succeeded; PARTIAL = some sites failed; FAILED = vps-server-setup failed.
