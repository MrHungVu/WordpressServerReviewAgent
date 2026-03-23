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
- [ ] Install OLS + LSPHP 8.2 + MariaDB + Redis + OPcache
- [ ] Configure and harden Cloudflare-only UFW firewall (auto-fetch CF IPs)
- [ ] Harden SSH configuration (disable root, change port optional)
- [ ] Install and configure fail2ban with OLS access log jail
- [ ] Auto-generate and secure MySQL root password
- [ ] Auto-generate Redis initial setup with DB 0-15
- [ ] Adaptive memory tuning (2GB/4GB/8GB+ profiles)
- [ ] WebAdmin console disabled by default
- [ ] Ensure idempotent — safe to re-run

### F6: WordPress Site Setup
- [ ] Multi-site aware: call N times for N sites on same VPS
- [ ] Install WordPress via WP-CLI + LSPHP
- [ ] Create database with auto-generated credentials
- [ ] Configure OLS vhost with isolated LSPHP worker pool
- [ ] Setup Redis object cache per site (DB 0-15 isolation)
- [ ] Install and configure LiteSpeed Cache plugin
- [ ] Install WooCommerce with LSCache exclusions
- [ ] Configure custom wp-admin URL via OLS rewrite rules
- [ ] Create per-site backup script with 30-day retention cron
- [ ] Auto-generate WordPress admin credentials

### F7: Cloudflare Domain Setup
- [ ] Create/detect Cloudflare zone for domain
- [ ] Add DNS A record pointing to server IP
- [ ] Add CNAME record for www subdomain
- [ ] Generate and install Origin Certificate (15-year validity at `/usr/local/lsws/conf/cert/`)
- [ ] Configure SSL Mode to Full (Strict)
- [ ] Enable WAF managed rules
- [ ] Enable Bot Fight Mode
- [ ] Add rate limiting for wp-login.php + xmlrpc.php
- [ ] Add WooCommerce page rules (cart/checkout/my-account bypass)
- [ ] Enable HTTP/3 (QUIC)
- [ ] Apply security settings (HSTS, TLS 1.2+, Brotli, Always HTTPS)

### F8: Domain Setup Orchestrator
- [ ] Multi-site flow: VPS setup once, then loop WP + CF setup per domain
- [ ] Adaptive LSAPI_CHILDREN calculation based on RAM + site count
- [ ] Redis DB assignment (sequential: site0=DB0, site1=DB1...)
- [ ] Collect and manage auto-generated credentials
- [ ] Deploy Cloudflare origin certificate to VPS via SSH
- [ ] Consolidate all credentials and setup details
- [ ] Generate multi-site domain setup report
- [ ] Partial failure handling (continue other sites if one fails)
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
- **Web Server:** OpenLiteSpeed 1.7.x
- **PHP Runtime:** LSPHP 8.2 with OPcache JIT
- **Database:** MariaDB 10.5+
- **Cache:** Redis 6.0+ (multi-DB 0-15)
- **SSH Transport:** Python paramiko or sshpass
- **Cloudflare API:** REST API v4 (zone analysis)
- **WordPress CLI:** WP-CLI (via LSPHP)
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

1. **Maximum 16 sites per VPS:** Redis DB 0-15 hard limit
2. **ModSecurity WAF:** Deferred to v2.1 (currently CloudFlare WAF only)
3. **Per-site OS user isolation:** Not implemented (all sites use www-data, planned for v2.1)
4. **Premium Plugin Updates:** Cannot auto-renew expired licenses
5. **Physical Security:** Only audits logical security (cannot assess physical server access)
6. **Code Quality Audits:** Does not perform deep static analysis on custom themes/plugins
7. **Performance Benchmarking:** Does not measure response times or load handling capacity

## Roadmap

See [Development Roadmap](./development-roadmap.md) for phases and milestones.

---

**Version:** 2.0
**Last Updated:** 2026-03-23
**Status:** V2.0 Release (OLS stack, multi-site support, all 8 skills complete)
