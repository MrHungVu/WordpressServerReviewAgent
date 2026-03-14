# ReviewServerConfigAgent Documentation

ReviewServerConfigAgent is a WordPress domain audit and review system that provides comprehensive security, performance, and configuration analysis across three critical layers: VPS infrastructure, Cloudflare CDN/DNS, and WordPress application.

## Quick Links

- **[Project Overview & PDR](./project-overview-pdr.md)** — Purpose, architecture, and requirements
- **[System Architecture](./system-architecture.md)** — Technical design and integration points
- **[Code Standards](./code-standards.md)** — Project structure and development guidelines
- **[Project Changelog](./project-changelog.md)** — All releases, features, and fixes
- **[Development Roadmap](./development-roadmap.md)** — Phases, milestones, and progress

## Core Audit Capabilities

### Three Integrated Skills

1. **VPS Server Audit** (`vps-server-audit` skill)
   - OS & kernel security
   - PHP configuration & extensions
   - MySQL/MariaDB hardening
   - Web server (Nginx, Apache, OpenLiteSpeed)
   - SSL/TLS certificates
   - Firewall rules & open ports
   - Fail2ban & rate limiting

2. **Cloudflare Domain Audit** (`cloudflare-domain-audit` skill)
   - DNS record validation
   - SSL/TLS configuration
   - Caching & page rules
   - Firewall rules & DDoS protection
   - Security headers & HSTS
   - Performance metrics

3. **WordPress Site Audit** (`wordpress-site-audit` skill)
   - WordPress core version & security
   - Plugin vulnerability scanning
   - Theme integrity & safety
   - User account security
   - Database integrity
   - Malware detection
   - Backup status

### Orchestrator

**Domain Review Agent** (`domain-review-agent`) runs all 3 audits and produces:
- Cross-layer validation (SSL chain, DNS/IP matching, cache/URL consistency)
- Combined health score (VPS, Cloudflare, WordPress)
- Prioritized action items (critical → warnings → recommendations)
- Individual detailed reports for each layer

## Reports Directory

All audit output is saved to `vps-reports/` with this naming convention:

```
vps-reports/
├── domain-review-{domain}-{YYYYMMDD}.md          # Combined summary
├── vps-audit-{domain}-{YYYYMMDD}.md              # VPS layer details
├── cloudflare-audit-{domain}-{YYYYMMDD}.md       # Cloudflare layer details
└── wordpress-audit-{domain}-{YYYYMMDD}.md        # WordPress layer details
```

**Security Note:** Reports may contain server IPs and configuration details. Never commit to version control or share with unauthorized users.

## Usage

```bash
# Run full domain review
# (Agent will ask for VPS SSH credentials, Cloudflare API token, and WP-CLI access)
"review domain example.com"
```

The agent will:
1. Prompt for required credentials (SSH, Cloudflare API, WP access method)
2. Execute all three audits in sequence
3. Generate individual reports
4. Produce combined report with cross-layer analysis
5. Assign health scores and prioritized recommendations

## Key Technical Considerations

### Server Variations
- Most servers run **Apache/Nginx**, but some use **OpenLiteSpeed (CyberPanel)**
- WP-CLI on OpenLiteSpeed may require LSPHP binary path: `/usr/local/lsws/lsphp{version}/bin/php`
- System PHP may lack extensions; LSPHP may be required for WP-CLI

### Credential Handling
- Credentials are **NEVER saved to files**
- Single-session only (passed inline, cleared after use)
- SSH: accepts password or key-based auth
- Cloudflare: API token (not legacy API key)
- WordPress: WP-CLI has direct DB access (no credentials needed if WP-CLI works)

### Common Issues
- MySQL `mysqli` extension missing in system PHP (fallback to LSPHP)
- `sshpass` unavailable on some systems (fallback to `paramiko` Python SSH)
- Premium WordPress plugins outdated (require manual license renewal)
- MX record misconfigurations (e.g., `smtp.google.com` instead of `aspmx.l.google.com`)

## Development

See [Code Standards](./code-standards.md) for skill implementation patterns and [System Architecture](./system-architecture.md) for integration details.

---

**Last Updated:** 2026-03-13
**Version:** 1.0
