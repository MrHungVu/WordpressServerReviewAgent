# Phase 3: cloudflare-domain-setup v2 (MODERATE UPDATE)

## Context Links

- Current skill: `.claude/skills/cloudflare-domain-setup/SKILL.md`
- Brainstorm: [Gap Analysis](../reports/brainstorm-260323-1703-ols-stack-upgrade.md)
- Depends on: [Phase 1](phase-01-vps-server-setup.md) (server ready), [Phase 2](phase-02-wordpress-site-setup.md) (site exists)

## Overview

- **Priority:** P1
- **Status:** complete
- **Effort:** 4h
- **Description:** Moderate update to existing Cloudflare setup skill. Changes origin cert path from `/etc/ssl/cloudflare/` to `/usr/local/lsws/conf/cert/`. Removes Nginx SSL config section (OLS handles it in Phase 2). Adds WAF managed rules, Bot Fight Mode, rate limiting for wp-login/xmlrpc, WooCommerce page rules (bypass cart/checkout/my-account), HTTP/3. Keeps all existing zone/DNS/SSL logic.

## Key Insights

- Origin cert path changes from Nginx location to OLS location -- only path difference
- No more Nginx config update step (OLS vhost SSL already configured by Phase 2)
- WAF managed rulesets API: `PUT /zones/{id}/rulesets/{ruleset_id}/rules`
- Bot Fight Mode: `PUT /zones/{id}/bot_management` (free plan uses `fight_mode`)
- Rate limiting uses Cloudflare Rules API (not deprecated rate limiting API)
- Free plan has 3 page rules; WooCommerce needs 1 extra -> may exhaust limit
- HTTP/3 is a zone setting toggle

## Requirements

### Functional
- F1: Keep existing: zone create/detect, DNS A+CNAME proxied, SSL Full Strict, Always HTTPS, HSTS, security settings
- F2: Change origin cert deploy path to `/usr/local/lsws/conf/cert/{domain}.crt` and `.key`
- F3: Remove Nginx SSL config update section entirely
- F4: NEW: Enable WAF managed rules (Cloudflare + OWASP rulesets) via API
- F5: NEW: Enable Bot Fight Mode via API
- F6: NEW: Rate limiting rules for wp-login.php and xmlrpc.php via API
- F7: NEW: WooCommerce page rules -- bypass cache for `/cart/*`, `/checkout/*`, `/my-account/*`
- F8: NEW: Enable HTTP/3 (QUIC) via zone settings API
- F9: Keep existing page rules for wp-admin + wp-login

### Non-Functional
- Idempotent: check existing before create/update
- All CF API via curl + jq
- Server-side commands (cert deploy) via sshpass SSH
- No credentials saved to files

## Architecture

No architectural change. Same Cloudflare API flow, different cert paths on server.

```
Cloudflare API Calls (new additions marked with +):
  POST /zones                           # Zone create/detect
  POST /zones/{id}/dns_records          # A + CNAME
  PATCH /zones/{id}/settings/ssl        # Full Strict
  POST /certificates                    # Origin cert
  PATCH /zones/{id}/settings/*          # Security settings
  POST /zones/{id}/pagerules            # wp-admin, wp-login
+ POST /zones/{id}/pagerules            # WooCommerce bypass
+ PATCH /zones/{id}/settings/http3      # Enable QUIC
+ PUT /zones/{id}/bot_management        # Bot Fight Mode
+ POST /zones/{id}/rulesets             # WAF managed rules
+ POST /zones/{id}/rulesets/{id}/rules  # Rate limiting
```

## Related Code Files

- **Modify:** `.claude/skills/cloudflare-domain-setup/SKILL.md`

## Implementation Steps

### Changes to Existing Steps

#### Step 5 (Origin Cert Install): Change paths
**Before (v1):**
```bash
sshpass ... 'mkdir -p /etc/ssl/cloudflare/'
echo "$CERTIFICATE" | sshpass ... 'cat > /etc/ssl/cloudflare/DOMAIN.pem'
echo "$PRIVATE_KEY" | sshpass ... 'cat > /etc/ssl/cloudflare/DOMAIN.key'
chmod 644 /etc/ssl/cloudflare/DOMAIN.pem
chmod 600 /etc/ssl/cloudflare/DOMAIN.key
chown root:root /etc/ssl/cloudflare/*
```

**After (v2):**
```bash
sshpass ... 'mkdir -p /usr/local/lsws/conf/cert/'
echo "$CERTIFICATE" | sshpass ... 'cat > /usr/local/lsws/conf/cert/${DOMAIN}.crt'
echo "$PRIVATE_KEY" | sshpass ... 'cat > /usr/local/lsws/conf/cert/${DOMAIN}.key'
sshpass ... '
  chmod 644 /usr/local/lsws/conf/cert/${DOMAIN}.crt
  chmod 600 /usr/local/lsws/conf/cert/${DOMAIN}.key
  chown nobody:nogroup /usr/local/lsws/conf/cert/${DOMAIN}.*
'
```

#### Step 6 (Nginx SSL Config): REMOVE ENTIRELY
Delete the entire "Update Nginx with SSL" section. OLS vhost SSL block already configured in Phase 2.

After cert deploy, restart OLS instead:
```bash
sshpass ... '/usr/local/lsws/bin/lswsctrl restart'
```

### New Steps

#### NEW Step 9: Enable WAF Managed Rules
```bash
# Get zone rulesets
RULESETS=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json")

# Enable Cloudflare Managed Ruleset (if available on plan)
CF_MANAGED_ID=$(echo "$RULESETS" | jq -r '.result[] | select(.name | contains("Cloudflare Managed")) | .id // empty')
if [ -n "$CF_MANAGED_ID" ]; then
  curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets/${CF_MANAGED_ID}" \
    -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" \
    --data '{"enabled": true}'
fi

# Enable OWASP ModSecurity Core Ruleset (if available)
OWASP_ID=$(echo "$RULESETS" | jq -r '.result[] | select(.name | contains("OWASP")) | .id // empty')
if [ -n "$OWASP_ID" ]; then
  curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets/${OWASP_ID}" \
    -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" \
    --data '{"enabled": true}'
fi
```
Note: WAF managed rules may require Pro+ plan. If 403 returned, log as "WAF requires Pro plan" and continue.

#### NEW Step 10: Enable Bot Fight Mode
```bash
curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/bot_management" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" \
  --data '{"fight_mode": true}'
```
If returns error (free plan may not support full bot management), try the simpler endpoint:
```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/browser_check" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" \
  --data '{"value":"on"}'
```

#### NEW Step 11: Rate Limiting Rules
Create rate limiting rules for wp-login.php and xmlrpc.php using Cloudflare custom rules:
```bash
# Rate limit wp-login.php: 5 requests per 10 seconds
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets/phases/http_ratelimit/entrypoint" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" \
  --data '{
    "rules": [
      {
        "description": "Rate limit wp-login.php",
        "expression": "(http.request.uri.path eq \"/wp-login.php\")",
        "action": "block",
        "ratelimit": {
          "characteristics": ["cf.colo.id", "ip.src"],
          "period": 10,
          "requests_per_period": 5,
          "mitigation_timeout": 600
        }
      },
      {
        "description": "Block xmlrpc.php",
        "expression": "(http.request.uri.path eq \"/xmlrpc.php\")",
        "action": "block"
      }
    ]
  }'
```
If rate limiting API not available on plan, log and continue.

#### NEW Step 12: WooCommerce Page Rules
Check remaining page rule slots (free plan = 3 total):
```bash
RULES_COUNT=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" | jq '.result | length')

if [ "$RULES_COUNT" -lt 3 ]; then
  # WooCommerce: bypass cache for cart/checkout/my-account
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
    -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" \
    --data '{
      "targets":[{"target":"url","constraint":{"operator":"matches","value":"*'${DOMAIN}'/(cart|checkout|my-account)/*"}}],
      "actions":[{"id":"cache_level","value":"bypass"}],
      "status":"active"
    }'
else
  echo "WARN: Page rules limit reached (${RULES_COUNT}/3). WooCommerce bypass rule NOT created. Consider Pro plan."
fi
```

#### NEW Step 13: Enable HTTP/3
```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/http3" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" \
  --data '{"value":"on"}'
```

### Updated Output
```
## Cloudflare Setup Results (v2.0)
- **Zone ID:** {zone_id}
- **Nameservers:** {ns1}, {ns2}
- **SSL Mode:** Full (Strict)
- **Origin Certificate:** /usr/local/lsws/conf/cert/{domain}.crt (expires ~2041)
- **DNS:** A @ -> {SERVER_IP} (proxied), CNAME www -> {domain} (proxied)
- **Settings:** Always HTTPS, TLS 1.2+, HSTS, Brotli, Medium Security
- **Page Rules:** wp-admin bypass, wp-login bypass, WooCommerce bypass (if slot available)
- **WAF:** Managed rules enabled (or "requires Pro plan")
- **Bot Fight Mode:** enabled (or "browser_check enabled as fallback")
- **Rate Limiting:** wp-login.php (5/10s), xmlrpc.php (blocked)
- **HTTP/3:** enabled
```

## Todo List

- [x] Change origin cert deploy path to `/usr/local/lsws/conf/cert/`
- [x] Change cert ownership to `nobody:nogroup`
- [x] Remove entire Nginx SSL config update section
- [x] Add `lswsctrl restart` after cert deploy
- [x] Implement WAF managed rules enable (with Pro plan fallback)
- [x] Implement Bot Fight Mode enable (with free plan fallback)
- [x] Implement rate limiting rules for wp-login + xmlrpc
- [x] Implement WooCommerce page rules (with slot check)
- [x] Implement HTTP/3 enable
- [x] Update output format with new settings

## Success Criteria

- Origin cert exists at `/usr/local/lsws/conf/cert/{domain}.crt`
- `curl -sI https://{domain}` shows Cloudflare headers + valid SSL
- SSL mode shows "strict" via API check
- HTTP/3 setting shows "on" via API check
- Page rules count includes WooCommerce bypass (or warn if full)
- OLS restarts without error after cert deploy

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| WAF managed rules require Pro plan | LOW | Gracefully skip with log message; Cloudflare free WAF still provides basic protection |
| Page rules limit exhausted (3 on free) | MEDIUM | Check count before creating; warn user to upgrade or consolidate rules |
| Rate limiting API not on free plan | LOW | Skip with warning; fail2ban on server provides equivalent protection |
| Bot Fight Mode blocks legitimate bots | LOW | Browser check fallback is less aggressive; can be toggled off |
| Origin cert deploy fails -> SSL broken | HIGH | Always restart OLS after deploy; verify with curl |

## Security Considerations

- Origin cert private key: 600 permissions, owned by nobody
- WAF rules add defense-in-depth beyond server-level protections
- Rate limiting at Cloudflare edge reduces load on origin
- Bot Fight Mode reduces automated attacks before they reach server
- xmlrpc.php blocked both at CF (rate limit) and OLS (rewrite rule)

## Next Steps

After this phase: Phase 4 (domain-setup-agent) orchestrates Phases 1-3 in sequence with multi-site loop.
