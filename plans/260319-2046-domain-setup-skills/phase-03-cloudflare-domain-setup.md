# Phase 3: Cloudflare Domain Setup Skill

## Context
- Parent plan: [plan.md](./plan.md)
- Depends on: [Phase 1](./phase-01-vps-server-setup.md) (server IP), [Phase 2](./phase-02-wordpress-site-setup.md) (WordPress installed)
- Reference: `.claude/skills/cloudflare-domain-audit/SKILL.md` (inverse of this)

## Overview
- **Priority:** P1
- **Status:** pending
- **Description:** Create `cloudflare-domain-setup` skill that configures Cloudflare for the domain via API: zone setup, DNS records, SSL Full Strict with Origin Certificate, security settings, and cache page rules.

## Key Insights
- Zone may already exist on Cloudflare — detect and reuse, don't create duplicate
- Origin Certificate generated via Cloudflare API (not Let's Encrypt) — 15 years validity
- Origin cert must be installed on Nginx via SSH (this skill crosses both CF API + SSH)
- Nginx server block from Phase 2 needs SSL directives added
- Nameserver change is the ONE thing user must do manually at their registrar
- Page rules for /wp-admin/* and /wp-login.php* bypass cache + higher security

## Requirements

### Functional
- Add zone or detect existing zone via Cloudflare API
- Create DNS A record for `@` → server IP (proxied)
- Create DNS CNAME record for `www` → domain (proxied)
- Set SSL mode to Full (Strict)
- Generate 15-year Origin Certificate via CF API
- Install origin cert + private key on server via SSH
- Update Nginx server block with SSL (listen 443 ssl, cert paths)
- Add HTTP→HTTPS redirect in Nginx
- Enable: Always Use HTTPS, min TLS 1.2, HSTS, Brotli, auto HTTPS rewrites
- Set security level to Medium
- Create page rules for wp-admin and wp-login.php (bypass cache, high security)
- Return Cloudflare nameservers for user to set at registrar

### Non-Functional
- Idempotent: check existing zone/records before creating
- Handle both Global API Key and API Token auth
- Graceful error if domain not yet pointed to CF (expected — user sets NS later)

## Architecture

### SKILL.md Structure
```
---
name: cloudflare-domain-setup
description: Configure Cloudflare for a domain via API. Sets up zone, DNS records, SSL Full Strict with Origin Certificate, security settings, and cache page rules. Use when user wants to set up Cloudflare for a WordPress domain.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---
```

### Cloudflare API Pattern
```bash
curl -s -X METHOD "https://api.cloudflare.com/client/v4/ENDPOINT" \
  -H "X-Auth-Email: EMAIL" \
  -H "X-Auth-Key: API_KEY" \
  -H "Content-Type: application/json" \
  --data 'JSON_BODY'
```

### Setup Sequence
1. Check existing zone: `GET /zones?name={domain}`
2. Create zone if not exists: `POST /zones` with `{name: domain, jump_start: true}`
3. Extract zone ID + nameservers from response
4. Create A record: `POST /zones/{id}/dns_records` — `@` → server IP, proxied
5. Create www CNAME: `POST /zones/{id}/dns_records` — `www` → domain, proxied
6. Set SSL Full Strict: `PATCH /zones/{id}/settings/ssl` — `{value: "strict"}`
7. Generate Origin Cert: `POST /certificates` — hostnames, 5475 days
8. SSH: create `/etc/ssl/cloudflare/` dir, write cert + key files
9. SSH: update Nginx server block with SSL config
10. SSH: `nginx -t && systemctl reload nginx`
11. Enable settings via PATCH: always_use_https, min_tls_version, brotli, security_level, automatic_https_rewrites
12. Create page rules: wp-admin bypass cache, wp-login bypass cache
13. Output: nameservers, zone ID, settings applied

### Origin Certificate API
```bash
# Generate origin cert
curl -s -X POST "https://api.cloudflare.com/client/v4/certificates" \
  -H "X-Auth-Email: EMAIL" \
  -H "X-Auth-Key: KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "hostnames": ["domain.com", "*.domain.com"],
    "requested_validity": 5475,
    "request_type": "origin-rsa",
    "csr": ""
  }'
```

### Nginx SSL Addition
```nginx
server {
    listen 443 ssl;
    server_name {domain} www.{domain};

    ssl_certificate /etc/ssl/cloudflare/{domain}.pem;
    ssl_certificate_key /etc/ssl/cloudflare/{domain}.key;

    # ... rest of config from Phase 2 ...
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name {domain} www.{domain};
    return 301 https://$host$request_uri;
}
```

## Related Code Files
- **Create:** `.claude/skills/cloudflare-domain-setup/SKILL.md`
- **Modify (via SSH):** `/etc/nginx/sites-available/{domain}` (add SSL)
- **Create (via SSH):** `/etc/ssl/cloudflare/{domain}.pem`, `/etc/ssl/cloudflare/{domain}.key`
- **Reference:** `.claude/skills/cloudflare-domain-audit/SKILL.md`

## Implementation Steps

1. Create directory `.claude/skills/cloudflare-domain-setup/`
2. Write `SKILL.md` with frontmatter
3. Write Purpose, Required Input (CF email, API key, domain, server IP, SSH creds)
4. Write Zone Setup section:
   - Check existing: `GET /zones?name={domain}`
   - Create if not exists: `POST /zones`
   - Extract zone_id and nameservers
5. Write DNS Records section:
   - Check existing A record before creating
   - Create A record (@ → IP, proxied)
   - Create www CNAME (www → domain, proxied)
6. Write SSL Configuration section:
   - Set SSL mode Full Strict
   - Generate Origin Certificate via API
   - Parse cert and key from JSON response
7. Write Origin Cert Installation section (SSH):
   - `mkdir -p /etc/ssl/cloudflare/`
   - Write cert to `{domain}.pem`, key to `{domain}.key`
   - Set permissions: 644 cert, 600 key
8. Write Nginx SSL Update section:
   - Replace Phase 2's HTTP-only server block with SSL block
   - Add HTTP→HTTPS redirect block
   - Test and reload: `nginx -t && systemctl reload nginx`
9. Write Security Settings section (API PATCH calls)
10. Write Page Rules section (API POST for wp-admin, wp-login)
11. Write Output section — nameservers + zone ID + settings list
12. Write Manual Steps section — nameserver change instructions

## Todo List
- [ ] Create `.claude/skills/cloudflare-domain-setup/` directory
- [ ] Write SKILL.md with frontmatter
- [ ] Zone detection/creation via API
- [ ] DNS A record creation (with existing record check)
- [ ] DNS www CNAME creation
- [ ] SSL Full Strict mode setting
- [ ] Origin Certificate generation via API
- [ ] Origin cert installation on server via SSH
- [ ] Nginx SSL configuration update
- [ ] HTTP→HTTPS redirect in Nginx
- [ ] Security settings (HTTPS, TLS 1.2, Brotli, HSTS)
- [ ] Page rules for wp-admin and wp-login
- [ ] Nameserver output for user manual step
- [ ] Error handling for API failures

## Success Criteria
- Cloudflare zone exists with correct DNS records
- SSL mode set to Full (Strict)
- Origin Certificate installed on Nginx, serving HTTPS
- `nginx -t` passes after SSL config
- All security settings enabled (HTTPS, TLS 1.2, Brotli)
- Page rules bypass cache for wp-admin and wp-login
- Cloudflare audit skill scores 8+/10

## Risk Assessment
| Risk | Impact | Mitigation |
|------|--------|------------|
| Zone already exists with conflicting records | MEDIUM | Check existing records, update instead of create |
| Origin cert API response format changes | LOW | Parse carefully, validate cert content |
| Nginx SSL config syntax error | HIGH | Always `nginx -t` before reload |
| Domain not yet pointed to CF | EXPECTED | This is normal — user sets NS after setup |
| Page rules limit on free plan (3 max) | MEDIUM | Check existing rules count, warn if limit reached |

## Security Considerations
- Origin cert private key permissions must be 600 (owner only)
- SSL directory must be owned by root
- Never log or output the private key content
- HSTS enables long-term HTTPS commitment (inform user)

## Next Steps
→ Phase 4: `domain-setup-agent` orchestrator
