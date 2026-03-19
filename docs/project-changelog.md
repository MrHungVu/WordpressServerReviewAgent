# Project Changelog

All notable changes to ReviewServerConfigAgent are documented here.

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
| 1.1.0 | 2026-03-19 | Released | Setup skills (VPS, WordPress, Cloudflare, Orchestrator) |
| 1.0.0 | 2026-03-13 | Released | Audit skills (VPS, Cloudflare, WordPress, Orchestrator) |

---

**Last Updated:** 2026-03-19
**Maintainer:** ReviewServerConfigAgent Project
