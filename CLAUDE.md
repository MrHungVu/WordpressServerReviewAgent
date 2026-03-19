# ReviewServerConfigAgent

WordPress domain review & setup agent with 4 audit skills + 4 setup skills.

## Audit Skills (in `.claude/skills/`)
- `vps-server-audit` — SSH into VPS, audit server config (OS, PHP, MySQL, Nginx, SSL, firewall, security)
- `cloudflare-domain-audit` — Cloudflare API audit (DNS, SSL/TLS, caching, security, performance)
- `wordpress-site-audit` — WP-CLI + source code audit (core, plugins, themes, users, malware scan, code review)
- `domain-review-agent` — Orchestrator that runs all 3 audits and produces combined report

## Setup Skills (in `.claude/skills/`)
- `vps-server-setup` — SSH into fresh VPS, install Nginx + PHP-FPM + MariaDB, configure firewall + SSH hardening
- `wordpress-site-setup` — Install WP-CLI + WordPress, create DB, configure wp-config.php, Nginx server block, security hardening
- `cloudflare-domain-setup` — Cloudflare API setup (zone, DNS, SSL Full Strict + Origin Cert, security settings, page rules)
- `domain-setup-agent` — Orchestrator that runs all 3 setup skills sequentially and produces credential report

## Usage
- **Audit:** "review domain example.com" — audits existing site, produces health report
- **Setup:** "setup domain example.com" — sets up new WordPress site from scratch on fresh VPS

## Reports
- Audit reports go to `vps-reports/` (e.g., `domain-review-example.com-20260319.md`)
- Setup reports go to `vps-reports/` (e.g., `domain-setup-example.com-20260319.md`)
- Never commit credentials

## Security
- Credentials are NEVER saved to files
- Credentials are single-session only (passed inline each time)
- Setup reports contain generated credentials — save immediately, they won't be shown again
- Reports may contain server IPs — review before sharing
