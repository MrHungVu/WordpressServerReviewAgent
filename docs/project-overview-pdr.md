# ReviewServerConfigAgent — Project Overview & PDR

## Project Purpose

ReviewServerConfigAgent is an automated audit and security review system for WordPress domains. It provides administrators, hosting providers, and security teams with comprehensive visibility into the health and security posture of WordPress sites by analyzing infrastructure, CDN, and application layers.

**Target Users:** WordPress site owners, managed hosting providers, security consultants, agency owners managing multiple client sites.

## Business Requirements

1. **Single Command Interface** — Users can run "review domain example.com" and get actionable results
2. **Multi-Layer Analysis** — Audit VPS infrastructure, Cloudflare CDN, and WordPress application simultaneously
3. **Cross-Layer Validation** — Detect inconsistencies (e.g., SSL config mismatch, cache bypass rules not aligning with site URLs)
4. **Prioritized Actions** — Surface critical issues first, then warnings, then recommendations
5. **Detailed Reports** — Generate individual reports per layer plus combined executive summary
6. **Non-Destructive** — Audits only read; fixes are manual or explicitly requested

## Functional Requirements

### F1: VPS Server Audit
- [ ] Establish SSH connection and execute server reconnaissance
- [ ] Detect OS, kernel version, and security updates available
- [ ] Audit PHP configuration (version, extensions, limits, security flags)
- [ ] Audit MySQL/MariaDB (version, users, grants, slow query log status)
- [ ] Detect web server type (Apache, Nginx, OpenLiteSpeed) and configuration
- [ ] Validate SSL certificates (expiry, chains, OCSP status)
- [ ] Check firewall rules, open ports, and IP restrictions
- [ ] Verify fail2ban/rate limiting installation and configuration
- [ ] Generate VPS audit report with 0-10 health score

### F2: Cloudflare Domain Audit
- [ ] Authenticate via Cloudflare API token
- [ ] Validate DNS A/CNAME records against known IPs
- [ ] Audit SSL/TLS configuration (mode, certificates, HSTS)
- [ ] Review page rules and cache bypass rules
- [ ] Check firewall rules, rate limiting, and DDoS protection
- [ ] Inspect security headers (CSP, X-Frame-Options, etc.)
- [ ] Generate Cloudflare audit report with 0-10 health score

### F3: WordPress Site Audit
- [ ] Execute WP-CLI commands on target server
- [ ] Verify WordPress core version and security patches
- [ ] Scan all installed plugins for updates and vulnerabilities
- [ ] Check theme integrity and list all active themes
- [ ] Enumerate user accounts and verify role assignments
- [ ] Check for malware signatures and suspicious files
- [ ] Validate database integrity and backup status
- [ ] Generate WordPress audit report with 0-10 health score

### F4: Domain Review Orchestrator
- [ ] Sequence all three audits (VPS → Cloudflare → WordPress)
- [ ] Cross-validate findings (DNS ↔ IP, SSL chain ↔ Cloudflare mode, URLs ↔ HTTPS enforcement)
- [ ] Consolidate health scores into overall rating
- [ ] Prioritize all findings by severity (critical → warning → recommendation)
- [ ] Generate combined domain review report
- [ ] Save all reports to `vps-reports/` directory

### F5: VPS Server Setup
- [ ] Install LEMP stack (Linux + Nginx + PHP-FPM + MariaDB)
- [ ] Configure and harden firewall (UFW)
- [ ] Harden SSH configuration (disable root, change port optional)
- [ ] Install and configure fail2ban for brute force protection
- [ ] Auto-generate and secure MySQL root password
- [ ] Ensure idempotent — safe to re-run

### F6: WordPress Site Setup
- [ ] Install WP-CLI and download latest WordPress
- [ ] Create database with auto-generated credentials
- [ ] Configure wp-config.php with security hardening (DISALLOW_FILE_EDIT, non-default prefix)
- [ ] Set proper file permissions (755 dirs, 644 files, 600 config)
- [ ] Create Nginx server block with security rules (block XML-RPC, user enum, REST API abuse)
- [ ] Auto-generate WordPress admin credentials

### F7: Cloudflare Domain Setup
- [ ] Create/detect Cloudflare zone for domain
- [ ] Add DNS A record pointing to server IP
- [ ] Add CNAME record for www subdomain
- [ ] Generate and install Origin Certificate (15-year validity)
- [ ] Configure SSL Mode to Full (Strict)
- [ ] Apply security settings (HSTS, TLS 1.2+, Brotli, Always HTTPS)
- [ ] Create page rules to bypass cache for wp-admin and wp-login

### F8: Domain Setup Orchestrator
- [ ] Sequence setup skills sequentially (VPS → WordPress → Cloudflare)
- [ ] Collect and manage auto-generated credentials
- [ ] Deploy Cloudflare origin certificate to VPS via SSH
- [ ] Consolidate all credentials and setup details
- [ ] Generate domain setup report with credentials and manual steps
- [ ] Handle partial failures with clear status reporting

## Non-Functional Requirements

### NFR1: Security & Privacy
- Credentials are NEVER persisted to disk
- Single-session credential handling only
- Reports flagged as sensitive (may contain IPs)
- No telemetry or external data transmission

### NFR2: Reliability
- Graceful error handling for SSH, API, and WP-CLI failures
- Fallback mechanisms (e.g., LSPHP if system PHP lacks extensions)
- Timeout handling for slow servers
- Retry logic for transient network issues

### NFR3: Usability
- Clear, actionable finding descriptions
- Organized by severity and layer
- Code examples for fixes where applicable
- Links between related findings

### NFR4: Performance
- Complete audit cycle < 5 minutes (typical)
- Lazy loading of large datasets
- Efficient API call batching
- Report generation in < 30 seconds

## Technical Architecture

### Skill-Based Design
ReviewServerConfigAgent is implemented as 8 Claude Skills (4 audit + 4 setup):

**Audit Skills:**
1. **vps-server-audit** — SSH audit engine
2. **cloudflare-domain-audit** — API audit engine
3. **wordpress-site-audit** — WP-CLI audit engine
4. **domain-review-agent** — Audit orchestrator & consolidation

**Setup Skills:**
5. **vps-server-setup** — LEMP stack installation & firewall hardening
6. **wordpress-site-setup** — WordPress + database installation & security hardening
7. **cloudflare-domain-setup** — Zone, DNS, SSL, and security configuration
8. **domain-setup-agent** — Setup orchestrator, credential capture, setup report

### Technology Stack
- **SSH Transport:** Python paramiko or sshpass
- **Cloudflare API:** REST API v4 (zone analysis)
- **WordPress CLI:** WP-CLI (WP-specific data extraction)
- **Report Generation:** Markdown with structured formatting
- **Orchestration:** Claude Agent with multi-turn reasoning

### Integration Points
- **VPS Layer:** Accepts multiple auth methods (password, key-based)
- **Cloudflare Layer:** API token authentication
- **WordPress Layer:** Direct WP-CLI or remote via SSH
- **Report Storage:** Local filesystem (`vps-reports/`)

## Success Criteria

- [x] V1.0 Feature Complete: All 4 skills implemented and tested
- [x] First Domain Audit completed (2026-03-13)
- [ ] 5+ domains successfully audited
- [ ] All findings documented and validated
- [ ] User feedback integrated

## Known Limitations

1. **OpenLiteSpeed Support:** Requires manual LSPHP path configuration
2. **Premium Plugin Updates:** Cannot auto-renew expired licenses
3. **Physical Security:** Only audits logical security (cannot assess physical server access)
4. **Code Quality Audits:** Does not perform deep static analysis on custom themes/plugins
5. **Performance Benchmarking:** Does not measure response times or load handling capacity

## Roadmap

See [Development Roadmap](./development-roadmap.md) for phases and milestones.

---

**Version:** 1.1
**Last Updated:** 2026-03-19
**Status:** Active Development (Setup skills v1.0 complete, audits stable)
