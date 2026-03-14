# ReviewServerConfigAgent

WordPress domain review agent with 3 audit skills + 1 orchestrator.

## Skills (in `.claude/skills/`)
- `vps-server-audit` — SSH into VPS, audit server config (OS, PHP, MySQL, Nginx, SSL, firewall, security)
- `cloudflare-domain-audit` — Cloudflare API audit (DNS, SSL/TLS, caching, security, performance)
- `wordpress-site-audit` — WP-CLI + source code audit (core, plugins, themes, users, malware scan, code review)
- `domain-review-agent` — Orchestrator that runs all 3 and produces combined report with cross-reference

## Usage
Tell Claude: "review domain example.com" — it will ask for credentials and run all audits.

## Reports
Output goes to `vps-reports/` directory. Never commit credentials.

## Security
- Credentials are NEVER saved to files
- Credentials are single-session only (passed inline each time)
- Reports may contain server IPs — review before sharing
