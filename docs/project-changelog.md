# Project Changelog

All notable changes to ReviewServerConfigAgent are documented here.

## [2.0.0] — 2026-03-23

### Major Changes — OLS Stack with Multi-Site Support

- **VPS Server Setup v2** (`vps-server-setup`) — MAJOR REWRITE
  - Replaced Nginx + PHP-FPM with OpenLiteSpeed + LSPHP 8.2
  - Added Redis installation + adaptive configuration (DBs 0-15)
  - Added OPcache JIT tuning (adaptive by RAM)
  - Added Cloudflare-only UFW firewall (auto-fetch CF IPs)
  - Added optional: sudo user creation, SSH key auth, custom SSH port
  - Added fail2ban OLS access log jail
  - Added adaptive memory profiling (2GB/4GB/8GB+)
  - WebAdmin console disabled by default

- **WordPress Site Setup v2** (`wordpress-site-setup`) — MAJOR REWRITE
  - Multi-site aware: call N times for N sites on same VPS
  - OLS vhost config with isolated LSPHP worker pool per site
  - Redis object cache per site (DB 0-15 isolation)
  - LiteSpeed Cache plugin auto-configured
  - WooCommerce always installed with LSCache exclusions
  - Custom wp-admin URL via OLS rewrite rules
  - Per-site backup script with 30-day retention cron
  - Per-site php.ini with adaptive memory limits

- **Cloudflare Domain Setup v2** (`cloudflare-domain-setup`) — MODERATE UPDATE
  - Origin cert path: `/usr/local/lsws/conf/cert/` (was `/etc/ssl/cloudflare/`)
  - Removed Nginx SSL config section (OLS handles SSL natively)
  - Added WAF managed rules enable
  - Added Bot Fight Mode enable
  - Added rate limiting for wp-login.php + xmlrpc.php
  - Added WooCommerce page rules (cart/checkout/my-account bypass)
  - Added HTTP/3 (QUIC) enable

- **Domain Setup Agent v2** (`domain-setup-agent`) — ORCHESTRATOR UPDATE
  - Multi-site flow: VPS setup once, then loop per domain
  - Adaptive LSAPI_CHILDREN calculation based on RAM + site count
  - Redis DB assignment (sequential: site0=DB0, site1=DB1...)
  - Partial failure handling (continue other sites if one fails)
  - Consolidated multi-site setup report

- **VPS Server Audit v2** (`vps-server-audit`) — UPDATE
  - OLS detection + LSPHP auto-detect
  - OLS config parsing (listeners, vhosts, ext apps)
  - Redis status + per-DB key counts
  - OPcache stats via LSPHP
  - UFW Cloudflare-only rules validation
  - WebAdmin disabled check + CF-Connecting-IP check

- **WordPress Site Audit v2** (`wordpress-site-audit`) — UPDATE
  - LSPHP auto-detect for WP-CLI
  - LSCache plugin + Redis config validation
  - Custom login URL check
  - WooCommerce audit with LSCache exclusion check
  - Per-site backup status check

- **Cloudflare Domain Audit v2** (`cloudflare-domain-audit`) — MINOR UPDATE
  - WAF managed rules status check
  - Bot Fight Mode check
  - WooCommerce page rules validation
  - HTTP/3 enabled check
  - Rate limiting rules check

- **Domain Review Agent v2** (`domain-review-agent`) — UPDATE
  - OLS SSL cert path cross-validation
  - CF-Connecting-IP cross-validation
  - LSPHP ↔ WordPress compatibility check
  - Redis DB assignment validation (no duplicates)
  - LSAPI_CHILDREN RAM budget validation
  - Backup coverage cross-check

- **Documentation Updates**
  - Updated system-architecture.md (OLS stack, multi-site architecture)
  - Updated code-standards.md (SKILL.md unified structure, OLS config notes)
  - Updated project-overview-pdr.md (F5-F8 requirements, known limitations)
  - Updated development-roadmap.md (Phase 1.2 complete, Phase 2 updated)
  - Updated CLAUDE.md (stack description)

### Technical Changes
- Unified skill file structure: `instructions.md` + `tests.md` + `examples.md` → single `SKILL.md`
- SKILL.md includes: Purpose, Stack Notes, Multi-Site Awareness, Implementation, Tests, Examples
- All skills OLS/LSPHP/Redis aware
- Memory profiling: auto-detect RAM and tune worker processes
- Redis: 16 isolated DBs per site, no sharing across sites
- LSPHP: Explicit path `/usr/local/lsws/lsphp82/bin/php` for WP-CLI invocation

---

## [1.1.0] — 2026-03-19

### Added
- **VPS Server Setup Skill** (`vps-server-setup`)
  - SSH-based LEMP stack installation (Nginx + PHP-FPM + MariaDB)
  - UFW/firewalld firewall configuration with common ports (80, 443, SSH)
  - SSH hardening (disable root login, optional port change)
  - fail2ban installation and configuration for brute force protection
  - Auto-generated MySQL root password (openssl rand -base64 32)
  - Idempotent operations — safe to re-run

- **WordPress Site Setup Skill** (`wordpress-site-setup`)
  - WP-CLI installation and WordPress core download with checksum verification
  - Automated database creation with auto-generated credentials
  - wp-config.php security hardening (DISALLOW_FILE_EDIT, non-default table prefix)
  - Proper file permissions (755 for directories, 644 for files, 600 for config)
  - Nginx server block creation with security rules
  - XML-RPC blocking, user enumeration protection, REST API abuse prevention
  - Auto-generated database and WordPress admin credentials

- **Cloudflare Domain Setup Skill** (`cloudflare-domain-setup`)
  - Zone creation or detection via Cloudflare API
  - DNS A record setup (root domain to server IP)
  - CNAME record for www subdomain (both proxied through Cloudflare)
  - Origin Certificate generation and installation (15-year validity)
  - SSL Mode configuration (Full Strict with HSTS)
  - TLS minimum version 1.2, Brotli compression, Always HTTPS enforcement
  - Page rules for wp-admin and wp-login.php cache bypass
  - Security settings (DDoS protection, rate limiting, firewall defaults)

- **Domain Setup Orchestrator** (`domain-setup-agent`)
  - Sequential execution pipeline (VPS → WordPress → Cloudflare)
  - Credential collection and management across all skills
  - Auto-generated credential capture and consolidation
  - Origin certificate deployment to VPS via SSH
  - Final setup report with all credentials and manual follow-up steps
  - Partial failure handling with clear status reporting
  - Verification checklist for post-setup validation

- **Documentation Updates**
  - Updated system-architecture.md with setup skill architecture and workflow diagrams
  - Updated project-overview-pdr.md with F5-F8 functional requirements for setup skills
  - Updated code-standards.md with setup report naming and credential handling standards
  - Updated development-roadmap.md with Phase 1.1 completion

---

## [1.0.0] — 2026-03-13

### Initial Release
Complete implementation of 4-skill WordPress domain review system with VPS, Cloudflare, and WordPress audit capabilities.

### Added
- **VPS Server Audit Skill** (`vps-server-audit`)
  - SSH-based server reconnaissance
  - OS, PHP, MySQL, and web server detection
  - SSL certificate validation
  - Firewall and port analysis
  - fail2ban and rate limiting checks
  - Generates VPS audit report with 0-10 health score

- **Cloudflare Domain Audit Skill** (`cloudflare-domain-audit`)
  - API-based Cloudflare zone analysis
  - DNS record validation against server IP
  - SSL/TLS mode and certificate inspection
  - Page rule and cache configuration review
  - Firewall rule and DDoS protection audit
  - Security headers validation
  - Generates Cloudflare audit report with 0-10 health score

- **WordPress Site Audit Skill** (`wordpress-site-audit`)
  - WP-CLI-based site reconnaissance
  - WordPress core version and security patch status
  - Plugin vulnerability and update scanning
  - Theme integrity and listing
  - User account enumeration and role validation
  - Malware detection via pattern matching
  - Database integrity checks
  - Generates WordPress audit report with 0-10 health score

- **Domain Review Orchestrator** (`domain-review-agent`)
  - Multi-layer audit sequencing
  - Cross-validation of infrastructure layer consistency
  - Health score consolidation (VPS + Cloudflare + WordPress)
  - Severity-based finding prioritization
  - Combined domain review report generation

- **Documentation Structure**
  - README.md with project overview and usage
  - System Architecture documentation
  - Code Standards and implementation guidelines
  - Project Changelog (this file)
  - Development Roadmap

### Technical Notes

#### Credential Handling
- SSH authentication via paramiko (sshpass unavailable on this system)
- Cloudflare API token authentication
- WP-CLI executed via LSPHP path (required due to system PHP limitation)

---

## Version Timeline

| Version | Date | Status | Key Features |
|---------|------|--------|--------------|
| 2.0.0 | 2026-03-23 | Released | OLS stack, multi-site support, all 8 skills v2 |
| 1.1.0 | 2026-03-19 | Released | Setup skills (VPS, WordPress, Cloudflare, Orchestrator) |
| 1.0.0 | 2026-03-13 | Released | Audit skills (VPS, Cloudflare, WordPress, Orchestrator) |

---

**Last Updated:** 2026-03-23
**Maintainer:** ReviewServerConfigAgent Project
