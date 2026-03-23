# ReviewServerConfigAgent

WordPress domain review & setup agent with 4 audit skills + 4 setup skills. OLS v2.0 stack with multi-site support.

## Audit Skills (in `.claude/skills/`)
- `vps-server-audit` — SSH into VPS, audit server config (OS, LSPHP, MySQL, OpenLiteSpeed, Redis, OPcache, SSL, firewall, security)
- `cloudflare-domain-audit` — Cloudflare API audit (DNS, SSL/TLS, WAF, bot fight, caching, security, performance)
- `wordpress-site-audit` — WP-CLI + source code audit (core, plugins, themes, users, malware scan, LSCache, Redis, custom login URL)
- `domain-review-agent` — Orchestrator that runs all 3 audits, cross-validates OLS/Redis/CF settings, produces combined report

## Setup Skills (in `.claude/skills/`)
- `vps-server-setup` — SSH into fresh VPS, install OLS + LSPHP 8.2 + MariaDB + Redis, configure Cloudflare-only firewall + SSH hardening + adaptive memory tuning
- `wordpress-site-setup` — Install WordPress via WP-CLI + LSPHP, create DB, OLS vhost with isolated LSPHP pool, Redis object cache, LSCache, WooCommerce, custom login URL, per-site backup
- `cloudflare-domain-setup` — Cloudflare API setup (zone, DNS, SSL Full Strict + Origin Cert, WAF, bot fight, rate limiting, WooCommerce page rules, HTTP/3)
- `domain-setup-agent` — Multi-site orchestrator: runs VPS setup once, then loops WP + CF setup per domain with adaptive resource allocation

## Usage
- **Audit:** "review domain example.com" — audits existing site, produces health report with OLS/Redis/CF cross-validation
- **Setup:** "setup domain example.com" — sets up new WordPress site from scratch on fresh VPS
- **Multi-site Setup:** "setup domains site1.com, site2.com" — sets up multiple WordPress sites on same VPS with adaptive memory allocation and isolated Redis DBs

## Reports
- Audit reports go to `vps-reports/` (e.g., `domain-review-example.com-20260319.md`)
- Setup reports go to `vps-reports/` (e.g., `domain-setup-example.com-20260319.md`)
- Never commit credentials

## Security
- Credentials are NEVER saved to files
- Credentials are single-session only (passed inline each time)
- Setup reports contain generated credentials — save immediately, they won't be shown again
- Reports may contain server IPs — review before sharing
