# Phase 4: domain-setup-agent v2 (ORCHESTRATOR UPDATE)

## Context Links

- Current skill: `.claude/skills/domain-setup-agent/SKILL.md`
- Brainstorm: [Gap Analysis - Orchestrator](../reports/brainstorm-260323-1703-ols-stack-upgrade.md)
- Depends on: [Phase 1](phase-01-vps-server-setup.md), [Phase 2](phase-02-wordpress-site-setup.md), [Phase 3](phase-03-cloudflare-domain-setup.md)

## Overview

- **Priority:** P1
- **Status:** complete
- **Effort:** 4h
- **Description:** Update orchestrator for multi-site flow. Collects domain list (not just single domain), admin IP, optional SSH key/port, custom login slugs per site. Runs vps-server-setup ONCE, then loops wordpress-site-setup + cloudflare-domain-setup per domain. Computes adaptive LSAPI_CHILDREN per site based on RAM profile and site count. Handles partial failures -- if one site fails, continues with others.

## Key Insights

- vps-server-setup runs ONCE per VPS (foundation)
- wordpress-site-setup + cloudflare-domain-setup run PER DOMAIN
- RAM profile from Phase 1 determines per-site LSAPI_CHILDREN: `floor(LSAPI_BASE / num_sites)`
- Redis DB assignment: site1=DB0, site2=DB1, site3=DB2...
- Partial failure: if site2 fails, site3 still gets set up; report shows per-site status
- Consolidated report includes ALL sites' credentials

## Requirements

### Functional
- F1: Collect expanded credentials: domain list, admin IP, SSH key (opt), custom SSH port (opt), login slug per site
- F2: Run vps-server-setup ONCE, capture RAM profile + MariaDB root pw + LSPHP path
- F3: Calculate LSAPI_CHILDREN per site: `floor(LSAPI_BASE / num_sites)`, minimum 4
- F4: FOR EACH domain: run wordpress-site-setup (pass site_number, LSAPI_CHILDREN, RAM profile)
- F5: FOR EACH domain: run cloudflare-domain-setup (pass domain, server IP, SSH creds)
- F6: Verification smoke test per site after all setups complete
- F7: Consolidated setup report with per-site credentials and statuses
- F8: Handle partial failure gracefully (continue others, report per-site status)

### Non-Functional
- Sequential within each site (WP before CF for that domain)
- Sites can be set up sequentially (safe, simpler)
- Single report file for all sites

## Architecture

```
Orchestration Flow:

1. Collect Credentials
   ├── SSH creds, CF creds, admin email
   ├── Domain list: [site1.com, site2.com, site3.com]
   ├── Admin IP for UFW
   ├── SSH key (optional)
   └── Login slugs: {site1: "my-login", site2: "admin-portal", ...}

2. Phase 1: vps-server-setup (ONCE)
   └── Outputs: RAM profile, LSAPI_BASE, MariaDB root pw, LSPHP path

3. Compute: LSAPI_CHILDREN = max(4, floor(LSAPI_BASE / num_sites))

4. FOR site_number=0; site_number < num_sites; site_number++:
   ├── Phase 2: wordpress-site-setup(domain, site_number, LSAPI_CHILDREN, ...)
   │   └── Outputs: WP admin creds, DB creds, Redis DB#
   ├── Phase 3: cloudflare-domain-setup(domain, server_ip, ...)
   │   └── Outputs: Zone ID, nameservers
   └── Smoke test: curl -sk https://{domain}/ | head

5. Consolidated Report -> vps-reports/domain-setup-{date}.md
```

## Related Code Files

- **Modify:** `.claude/skills/domain-setup-agent/SKILL.md`

## Implementation Steps

### Step 1: Updated Credential Collection
Ask user for ALL of these via AskUserQuestion:

**Required:**
1. Domain list (comma-separated or one per line)
2. SSH host (IP address)
3. SSH user (default: root)
4. SSH password
5. SSH port (default: 22)
6. Cloudflare email
7. Cloudflare Global API Key
8. WordPress admin email
9. Admin IP (for UFW SSH whitelist)

**Conditional (ask if existing VPS detected):**
<!-- Updated: Validation Session 1 - MariaDB root pw needed for existing VPS -->
10. MariaDB root password (if VPS already has OLS+MariaDB installed; vps-server-setup detects this and skips install)

**Optional:**
11. SSH public key (for key auth setup)
12. Custom SSH port (to change to)
13. Custom login slug per site (default: random `login-{hex4}`)

### Step 2: Verify Connectivity
```bash
# Test SSH
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p PORT USER@HOST 'echo "SSH OK"'

# Test Cloudflare API
curl -s "https://api.cloudflare.com/client/v4/user" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.success'
```
If either fails, inform user and ask to verify. Do NOT proceed until both pass.

### Step 3: Activate vps-server-setup (ONCE)
Pass: SSH creds, first domain (for context), admin IP, optional SSH key, optional custom port.

Capture from output:
- Server IP
- RAM profile (2GB/4GB/8GB+)
- LSAPI_BASE value
- MariaDB root password
- LSPHP path
- OLS version
- Active SSH port

### Step 4: Compute Per-Site Resources
```
num_sites = len(domain_list)
LSAPI_CHILDREN = max(4, floor(LSAPI_BASE / num_sites))

# Memory limits per LSPHP ext app (for memSoftLimit/memHardLimit in OLS config)
if RAM_PROFILE == "2GB":  PHP_MEM_SOFT="350M", PHP_MEM_HARD="400M"
if RAM_PROFILE == "4GB":  PHP_MEM_SOFT="500M", PHP_MEM_HARD="600M"
if RAM_PROFILE == "8GB+": PHP_MEM_SOFT="800M", PHP_MEM_HARD="1000M"
```

### Step 5: Per-Site Setup Loop
```
for site_number, domain in enumerate(domain_list):
  login_slug = user_provided_slug[domain] or "login-" + random_hex(4)

  # 5a. wordpress-site-setup
  activate wordpress-site-setup(
    SSH creds, domain, MariaDB root password, admin email,
    site_number, LSAPI_CHILDREN, RAM_PROFILE, login_slug, LSPHP path
  )
  -> capture: WP admin creds, DB creds, Redis DB#

  # 5b. cloudflare-domain-setup
  activate cloudflare-domain-setup(
    CF creds, domain, server_ip, SSH creds
  )
  -> capture: zone ID, nameservers

  # 5c. Smoke test
  curl -sk -H "Host: ${domain}" https://${SERVER_IP}/ | head -5
  # Check for WordPress HTML or 200 status

  # Record per-site status: COMPLETE or FAILED (with error)
```

**Partial Failure:** If wordpress-site-setup fails for site2:
- Log error for site2
- Skip cloudflare-domain-setup for site2
- Continue with site3
- Mark site2 as FAILED in report

### Step 6: Generate Consolidated Report
Save to: `PROJECT_ROOT/vps-reports/domain-setup-{YYYYMMDD}.md`

**IMPORTANT:** Output ALL information in copy-friendly plaintext format so user can copy-paste entire block.

```
═══════════════════════════════════════════════════════════════════
  MULTI-SITE DOMAIN SETUP REPORT
  Date: YYYY-MM-DD HH:MM UTC
  Status: COMPLETE / PARTIAL ({N}/{total} sites)
  COPY AND SAVE ALL INFORMATION BELOW IMMEDIATELY
═══════════════════════════════════════════════════════════════════

══ VPS SERVER ═════════════════════════════════════════════════════
  Server IP:            {server_ip}
  SSH Port:             {ssh_port}
  SSH User:             {ssh_user}
  OS:                   {os_version}
  RAM:                  {total_ram}MB (Profile: {profile})
  OpenLiteSpeed:        {ols_version}
  LSPHP:                8.2 (/usr/local/lsws/lsphp82/bin/lsphp)
  MariaDB:              {mariadb_version}
  MariaDB Root Password:{mariadb_root_pass}
  Redis:                active (maxmemory={redis_mem}mb)
  OPcache:              {opcache_mem}MB, JIT=tracing
  Firewall:             UFW (SSH from {admin_ip}, 80+443 CF only)
  fail2ban:             active (sshd + openlitespeed-auth)
  WebAdmin:             disabled
  Config:               /usr/local/lsws/conf/httpd_config.conf

══ SITE 1: {domain1} [COMPLETE] ═══════════════════════════════════

  ── LOGIN ────────────────────────────────────────────────────
  Site URL:             https://{domain1}
  Admin Login URL:      https://{domain1}/{login_slug1}
  Admin Username:       {wp_admin_user1}
  Admin Password:       {wp_admin_pass1}
  Admin Email:          {admin_email}

  ── DATABASE ─────────────────────────────────────────────────
  DB Name:              {db_name1}
  DB User:              {db_user1}
  DB Password:          {db_pass1}
  DB Prefix:            {prefix1}

  ── SERVER CONFIG ────────────────────────────────────────────
  Install Path:         /var/www/{domain1}/html
  OLS Vhost:            /usr/local/lsws/conf/vhosts/{domain1}.conf
  LSPHP Workers:        {lsapi_children}
  Redis DB:             0
  Redis Prefix:         {slug1}:
  Backup Script:        /usr/local/bin/backup-{slug1}.sh
  Backup Schedule:      Daily at 02:00 UTC

  ── CLOUDFLARE ───────────────────────────────────────────────
  CF Zone ID:           {zone_id1}
  CF Nameserver 1:      {ns1}
  CF Nameserver 2:      {ns2}
  SSL Mode:             Full (Strict)
  Origin Cert:          /usr/local/lsws/conf/cert/{domain1}.crt

══ SITE 2: {domain2} [COMPLETE] ═══════════════════════════════════
  (same format as Site 1)

══ SITE 3: {domain3} [FAILED] ═════════════════════════════════════
  Error: {error message}
  Action: Re-run "setup domain {domain3}" to retry

══ SECURITY APPLIED (ALL SITES) ═══════════════════════════════════
  [x] OLS with WebAdmin disabled
  [x] Cloudflare-only firewall (no direct IP access)
  [x] Custom login URLs (per site, default /wp-login.php blocked)
  [x] SSL Full Strict with Cloudflare origin certificates
  [x] Redis object cache (isolated DB per site)
  [x] LSCache + WooCommerce cache exclusions
  [x] fail2ban (SSH + OLS brute-force jails)
  [x] Per-site backup scripts with 30-day retention
  [x] DB credentials in .my.cnf (not plaintext scripts)

══ ACTION REQUIRED ════════════════════════════════════════════════
  1. Change nameservers at your domain registrar:
     {domain1} → {ns1}, {ns2}
     {domain2} → {ns1}, {ns2}
  2. Wait 24-48 hours for DNS propagation
  3. Test each site: visit https://{domain}/
  4. Log in and change admin passwords immediately
  5. Run "review domain {domain}" to verify each site

══ POST-SETUP RECOMMENDATIONS ════════════════════════════════════
  [ ] Configure SMTP for email delivery (per site)
  [ ] Install security monitoring plugin
  [ ] Test backup restore: /usr/local/bin/backup-{slug}.sh
  [ ] Set up uptime monitoring (UptimeRobot etc.)
  [ ] Review Cloudflare WAF rules for false positives

═══════════════════════════════════════════════════════════════════
  THIS REPORT IS SAVED TO: vps-reports/domain-setup-{YYYYMMDD}.md
  CREDENTIALS WILL NOT BE SHOWN AGAIN — SAVE NOW
═══════════════════════════════════════════════════════════════════
```

## Todo List

- [x] Update credential collection with domain list + admin IP + SSH key/port + login slugs
- [x] Update connectivity verification
- [x] Implement vps-server-setup activation (once)
- [x] Implement per-site resource computation (LSAPI_CHILDREN, memory limits)
- [x] Implement per-site setup loop (wordpress-site-setup + cloudflare-domain-setup)
- [x] Implement smoke test per site
- [x] Implement partial failure handling (continue on failure, per-site status)
- [x] Implement consolidated multi-site report generation
- [x] Update trigger phrases for multi-site

## Success Criteria

- All sites in domain list attempted (even if some fail)
- Consolidated report contains credentials for all successful sites
- Failed sites clearly marked with error reason
- LSAPI_CHILDREN correctly calculated: `floor(LSAPI_BASE / num_sites)` >= 4
- Redis DB assignments sequential: site0=DB0, site1=DB1...
- Report saved to `vps-reports/domain-setup-{date}.md`

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Site 2 failure corrupts OLS config for site 1 | HIGH | `openlitespeed -t` after each site; rollback append if fails |
| LSAPI_CHILDREN too low (many sites) | MEDIUM | Enforce minimum 4; warn if <6 that performance may suffer |
| Redis DB 15 exceeded (>16 sites) | LOW | Maximum 16 sites per VPS; warn user and stop |
| Cloudflare rate limit during multi-site CF setup | LOW | Small delay between CF API calls; retry on 429 |

## Security Considerations

- All credentials in-memory only for session
- Consolidated report is one-time delivery (warn user to save)
- Report file in vps-reports/ (.gitignore'd) -- user should delete after saving creds
- Admin IP whitelist covers all sites (single VPS)
- Each site gets unique DB user with minimal privileges

## Next Steps

After this phase: Setup skills complete. Move to audit skill updates (Phases 5-8).
