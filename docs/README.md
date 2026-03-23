# ReviewServerConfigAgent Documentation

WordPress domain setup & audit agent with 8 Claude skills. Provisions complete OLS multi-site stacks and audits them across VPS, Cloudflare, and WordPress layers.

## Quick Links

- **[Project Overview & PDR](./project-overview-pdr.md)** — Purpose, architecture, and requirements
- **[System Architecture](./system-architecture.md)** — Technical design and integration points
- **[Code Standards](./code-standards.md)** — Project structure and development guidelines
- **[Project Changelog](./project-changelog.md)** — All releases, features, and fixes
- **[Development Roadmap](./development-roadmap.md)** — Phases, milestones, and progress

## Stack (v2.0)

**Server:** OpenLiteSpeed + LSPHP 8.2 + MariaDB + Redis + OPcache JIT
**CDN:** Cloudflare (SSL Full Strict, WAF, Bot Fight, Rate Limiting, HTTP/3)
**App:** WordPress + LSCache + WooCommerce + custom login URLs
**Multi-site:** 1 VPS → up to 16 WordPress sites (Redis DB 0-15 isolation)

## Setup Skills (4)

1. **VPS Server Setup** (`vps-server-setup`)
   - OLS + LSPHP 8.2 + MariaDB + Redis installation
   - Adaptive memory tuning (2GB/4GB/8GB+ profiles)
   - Cloudflare-only UFW firewall (auto-fetch CF IPs)
   - SSH hardening, fail2ban, OLS log rotation
   - Idempotent — detects existing stack, skips installed components

2. **WordPress Site Setup** (`wordpress-site-setup`)
   - Per-site: DB, OLS vhost, isolated LSPHP worker pool, Redis DB
   - LSCache plugin + WooCommerce auto-configured
   - Custom login URL via OLS rewrite (blocks /wp-login.php)
   - Per-site backup script with .my.cnf (no plaintext passwords)
   - Staging config approach — validates before touching production

3. **Cloudflare Domain Setup** (`cloudflare-domain-setup`)
   - Zone + DNS + SSL Full Strict + Origin Certificate
   - WAF managed rules, Bot Fight Mode, rate limiting
   - WooCommerce page rules (cart/checkout/my-account bypass)
   - HTTP/3 (QUIC) enabled

4. **Domain Setup Agent** (`domain-setup-agent`) — Orchestrator
   - Multi-site: VPS setup once, then loops WP + CF per domain
   - Adaptive LSAPI_CHILDREN based on RAM profile + site count
   - Partial failure handling, consolidated credential report

## Audit Skills (4)

1. **VPS Server Audit** (`vps-server-audit`)
   - Supports both Nginx (v1) and OLS (v2) stacks
   - LSPHP auto-detect, OLS config parsing, Redis status
   - OPcache stats, UFW Cloudflare-only rules validation
   - Backup infrastructure check

2. **Cloudflare Domain Audit** (`cloudflare-domain-audit`)
   - DNS, SSL/TLS, security, caching, page rules, edge certs
   - WAF managed rules, Bot Fight Mode, rate limiting checks
   - WooCommerce page rules validation, HTTP/3 check

3. **WordPress Site Audit** (`wordpress-site-audit`)
   - LSPHP auto-detect for WP-CLI on OLS servers
   - LSCache + Redis per-site DB validation
   - Custom login URL check, WooCommerce audit
   - Plugin vulnerability scanning via WebSearch
   - Per-site backup status check

4. **Domain Review Agent** (`domain-review-agent`) — Orchestrator
   - Runs all 3 audits, cross-validates across layers
   - OLS SSL cert path, CF-Connecting-IP, LSPHP↔WP compat
   - Redis DB isolation, LSAPI_CHILDREN RAM budget validation
   - Combined health score + prioritized action items

## Usage

```bash
# Setup new multi-site
"setup domains site1.com, site2.com"

# Setup single site
"setup domain example.com"

# Audit existing site
"review domain example.com"
```

## Reports

All output saved to `vps-reports/` (gitignored):
```
vps-reports/
├── domain-setup-{YYYYMMDD}.md           # Multi-site setup credentials
├── domain-review-{domain}-{YYYYMMDD}.md # Combined audit
├── vps-audit-{domain}-{YYYYMMDD}.md
├── cloudflare-audit-{domain}-{YYYYMMDD}.md
└── wordpress-audit-{domain}-{YYYYMMDD}.md
```

**Security:** Reports contain credentials and server IPs. Never commit or share.

## Credential Handling

- Credentials **NEVER saved to files** (wp-config.php and .my.cnf are operational exceptions)
- Single-session only — passed inline, cleared after use
- Setup reports are one-time credential delivery — save immediately

## Development

See [Code Standards](./code-standards.md) for skill patterns and [System Architecture](./system-architecture.md) for integration details.

---

**Last Updated:** 2026-03-23
**Version:** 2.0
