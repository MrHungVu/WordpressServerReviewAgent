---
name: cloudflare-domain-setup
description: Configure Cloudflare for a domain via API. Sets up zone, DNS records, SSL Full Strict with Origin Certificate, WAF managed rules, Bot Fight Mode, rate limiting for wp-login/xmlrpc, WooCommerce cache bypass rules, HTTP/3 (QUIC), security settings, and cache page rules. Use when user wants to set up Cloudflare for a WordPress domain.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---

# Cloudflare Domain Setup Skill

## Purpose
Configure Cloudflare for a WordPress domain: zone creation, DNS records, SSL Full (Strict) with Origin Certificate deployed to OLS cert directory, WAF managed rules, Bot Fight Mode, rate limiting for wp-login/xmlrpc, WooCommerce page rules, HTTP/3 (QUIC), security settings, and page rules.

## Required Input (from user/orchestrator, NEVER save to files)
- Cloudflare email
- Cloudflare Global API Key
- Domain name
- Server IP (from vps-server-setup or user)
- SSH host, user, password, port (for installing origin cert on server)

## API Authentication
All Cloudflare API calls use:
```bash
curl -s -X METHOD "https://api.cloudflare.com/client/v4/ENDPOINT" \
  -H "X-Auth-Email: EMAIL" \
  -H "X-Auth-Key: API_KEY" \
  -H "Content-Type: application/json" \
  --data 'JSON_BODY'
```

Ensure `jq` is available: `command -v jq &>/dev/null || apt install jq -y` (or `brew install jq` on macOS).

## SSH Connection (for server-side operations)
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

If `sshpass` not installed locally:
- macOS: `brew install hudochenkov/sshpass/sshpass`
- Linux: `apt install sshpass -y`

## Setup Checklist

### 1. Check/Create Zone
```bash
# Check existing zone
ZONE_RESPONSE=$(curl -s "https://api.cloudflare.com/client/v4/zones?name=DOMAIN" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json")
ZONE_ID=$(echo "$ZONE_RESPONSE" | jq -r '.result[0].id // empty')

# Create if not exists
if [ -z "$ZONE_ID" ]; then
  ZONE_RESPONSE=$(curl -s -X POST "https://api.cloudflare.com/client/v4/zones" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"name":"DOMAIN","jump_start":true}')
  ZONE_ID=$(echo "$ZONE_RESPONSE" | jq -r '.result.id')
fi

# Extract nameservers
NS1=$(echo "$ZONE_RESPONSE" | jq -r '.result.name_servers[0] // .result[0].name_servers[0]')
NS2=$(echo "$ZONE_RESPONSE" | jq -r '.result.name_servers[1] // .result[0].name_servers[1]')
```

### 2. DNS Records
Check existing before creating. Update if exists, create if not.

**A Record (@ → Server IP, proxied):**
```bash
# Check existing A record
EXISTING_A=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records?type=A&name=DOMAIN" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json")
RECORD_ID=$(echo "$EXISTING_A" | jq -r '.result[0].id // empty')

if [ -n "$RECORD_ID" ]; then
  # Update existing
  curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records/${RECORD_ID}" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"A","name":"@","content":"SERVER_IP","proxied":true}'
else
  # Create new
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"A","name":"@","content":"SERVER_IP","proxied":true}'
fi
```

**www CNAME (www → domain, proxied):**
```bash
# Check existing www CNAME
EXISTING_WWW=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records?type=CNAME&name=www.DOMAIN" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json")
WWW_ID=$(echo "$EXISTING_WWW" | jq -r '.result[0].id // empty')

if [ -n "$WWW_ID" ]; then
  curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records/${WWW_ID}" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"CNAME","name":"www","content":"DOMAIN","proxied":true}'
else
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"CNAME","name":"www","content":"DOMAIN","proxied":true}'
fi
```

### 3. SSL Mode — Full (Strict)
```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/ssl" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{"value":"strict"}'
```

### 4. Generate Origin Certificate
**NOTE:** The Origin CA API uses a different auth header than the zone API. Use `X-Auth-User-Service-Key` with the Global API Key (same key value, different header name). If that fails, try the Origin CA Key from the Cloudflare dashboard (API Tokens → Origin CA Key).
```bash
CERT_RESPONSE=$(curl -s -X POST "https://api.cloudflare.com/client/v4/certificates" \
  -H "X-Auth-User-Service-Key: KEY" -H "Content-Type: application/json" \
  --data '{
    "hostnames": ["DOMAIN", "*.DOMAIN"],
    "requested_validity": 5475,
    "request_type": "origin-rsa",
    "csr": ""
  }')

CERTIFICATE=$(echo "$CERT_RESPONSE" | jq -r '.result.certificate')
PRIVATE_KEY=$(echo "$CERT_RESPONSE" | jq -r '.result.private_key')
```
**IMPORTANT:** Never log or output the private key content.

### 5. Install Origin Cert on Server (via SSH)
Deploy cert to OLS cert directory. OLS vhost SSL block already references these paths (configured in Phase 2).
```bash
# Create OLS cert directory
sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'mkdir -p /usr/local/lsws/conf/cert/'

# Write certificate
echo "$CERTIFICATE" | sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'cat > /usr/local/lsws/conf/cert/DOMAIN.crt'

# Write private key
echo "$PRIVATE_KEY" | sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'cat > /usr/local/lsws/conf/cert/DOMAIN.key'

# Set permissions and ownership (www-data = OLS runtime user)
sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  chmod 644 /usr/local/lsws/conf/cert/DOMAIN.crt
  chmod 600 /usr/local/lsws/conf/cert/DOMAIN.key
  chown www-data:www-data /usr/local/lsws/conf/cert/DOMAIN.*
'

# Restart OLS to load new cert
sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  '/usr/local/lsws/bin/lswsctrl restart'
```

### 6. Security Settings (API)
Apply each setting via PATCH (use full API URL with zone_id):
```bash
BASE="https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings"
AUTH='-H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json"'

# Always Use HTTPS
curl -s -X PATCH "${BASE}/always_use_https" ${AUTH} --data '{"value":"on"}'

# Minimum TLS 1.2
curl -s -X PATCH "${BASE}/min_tls_version" ${AUTH} --data '{"value":"1.2"}'

# Auto HTTPS Rewrites
curl -s -X PATCH "${BASE}/automatic_https_rewrites" ${AUTH} --data '{"value":"on"}'

# Brotli compression
curl -s -X PATCH "${BASE}/brotli" ${AUTH} --data '{"value":"on"}'

# Security level
curl -s -X PATCH "${BASE}/security_level" ${AUTH} --data '{"value":"medium"}'

# Browser integrity check
curl -s -X PATCH "${BASE}/browser_check" ${AUTH} --data '{"value":"on"}'

# HSTS
curl -s -X PATCH "${BASE}/security_header" ${AUTH} \
  --data '{"value":{"strict_transport_security":{"enabled":true,"max_age":31536000,"include_subdomains":true,"nosniff":true}}}'
```

### 7. Page Rules (wp-admin + wp-login)
Check existing rules count first (free plan = 3 max):
```bash
RULES_COUNT=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.result | length')
```

If rules available, create:
```bash
# wp-admin: bypass cache + high security
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{
    "targets":[{"target":"url","constraint":{"operator":"matches","value":"*DOMAIN/wp-admin/*"}}],
    "actions":[{"id":"cache_level","value":"bypass"},{"id":"security_level","value":"high"}],
    "status":"active"
  }'

# wp-login: bypass cache + high security
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{
    "targets":[{"target":"url","constraint":{"operator":"matches","value":"*DOMAIN/wp-login.php*"}}],
    "actions":[{"id":"cache_level","value":"bypass"},{"id":"security_level","value":"high"}],
    "status":"active"
  }'
```

If page rules limit reached, warn user and skip.

### 8. WooCommerce Page Rules (bypass cache)
Check remaining page rule slots. Free plan = 3 total; wp-admin and wp-login above use 2 slots.
```bash
RULES_COUNT=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.result | length')

if [ "$RULES_COUNT" -lt 3 ]; then
  # WooCommerce: bypass cache for cart, checkout, and my-account pages
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{
      "targets":[{"target":"url","constraint":{"operator":"matches","value":"*DOMAIN/(cart|checkout|my-account)/*"}}],
      "actions":[{"id":"cache_level","value":"bypass"}],
      "status":"active"
    }'
else
  echo "WARN: Page rules limit reached (${RULES_COUNT}/3). WooCommerce bypass rule NOT created. Consider Pro plan."
fi
```

### 9. Enable WAF Managed Rules
WAF managed rulesets may require Pro+ plan. If unavailable, log and continue — free plan still provides basic protection.
```bash
# Get zone rulesets
RULESETS=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json")

# Enable Cloudflare Managed Ruleset (if available on plan)
CF_MANAGED_ID=$(echo "$RULESETS" | jq -r '.result[] | select(.name | contains("Cloudflare Managed")) | .id // empty')
if [ -n "$CF_MANAGED_ID" ]; then
  WAF_RESULT=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets/${CF_MANAGED_ID}" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"enabled": true}')
  echo "$WAF_RESULT" | jq -r 'if .success then "WAF Cloudflare Managed: enabled" else "WAF Cloudflare Managed: requires Pro plan" end'
else
  echo "WAF Cloudflare Managed: not available on this plan"
fi

# Enable OWASP ModSecurity Core Ruleset (if available)
OWASP_ID=$(echo "$RULESETS" | jq -r '.result[] | select(.name | contains("OWASP")) | .id // empty')
if [ -n "$OWASP_ID" ]; then
  OWASP_RESULT=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets/${OWASP_ID}" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"enabled": true}')
  echo "$OWASP_RESULT" | jq -r 'if .success then "WAF OWASP: enabled" else "WAF OWASP: requires Pro plan" end'
else
  echo "WAF OWASP: not available on this plan"
fi
```

### 10. Enable Bot Fight Mode
```bash
BOT_RESULT=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/bot_management" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{"fight_mode": true}')

# If bot_management endpoint fails (free plan), fall back to browser_check
if echo "$BOT_RESULT" | jq -e '.success == false' > /dev/null 2>&1; then
  echo "Bot Fight Mode: not available, enabling browser_check as fallback"
  curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/browser_check" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"value":"on"}'
else
  echo "Bot Fight Mode: enabled"
fi
```

### 11. Rate Limiting Rules
Create rate limiting rules for wp-login.php and xmlrpc.php using Cloudflare custom rules API.
If rate limiting API is not available on plan, log and continue (fail2ban on server provides equivalent protection).
```bash
RATELIMIT_RESULT=$(curl -s -X POST \
  "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets/phases/http_ratelimit/entrypoint" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
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
  }')

echo "$RATELIMIT_RESULT" | jq -r 'if .success then "Rate limiting: enabled (wp-login 5/10s, xmlrpc blocked)" else "Rate limiting: \(.errors[0].message // "not available on this plan")" end'
```

### 12. Enable HTTP/3 (QUIC)
```bash
HTTP3_RESULT=$(curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/http3" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{"value":"on"}')

echo "$HTTP3_RESULT" | jq -r 'if .success then "HTTP/3: enabled" else "HTTP/3: \(.errors[0].message // "failed")" end'
```

## Output
Report these results to the orchestrator:
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

## Manual Step Required
```
## ACTION REQUIRED — Change Nameservers
Go to your domain registrar and change nameservers to:
1. {nameserver1}
2. {nameserver2}
DNS propagation takes 24-48 hours. Site will work after propagation completes.
```

## Error Handling
- If zone already exists → reuse it (extract zone_id)
- If DNS records exist → update (PUT) instead of create (POST)
- If page rules limit reached → warn user, skip additional page rules
- If origin cert generation fails → check API key has zone permissions
- If OLS restart fails after cert deploy → check OLS logs: `/usr/local/lsws/logs/error.log`
- If WAF managed rules return 403 → requires Pro plan, log and continue
- If bot_management endpoint returns error → fall back to browser_check setting
- If rate limiting API unavailable → log and continue (fail2ban on server covers it)
- If API returns 403 → API key lacks permissions, inform user
- If API returns 429 → rate limited, wait 60s and retry once

## Important Rules
- NEVER save API keys or credentials to any file
- NEVER log or output the private key content
- Origin cert private key file must be 600 permissions (owner-only)
- Cert files owned by www-data:www-data (OLS runtime user)
- Use `jq` for JSON parsing
- Run all server-side commands via SSH remotely
- Every step checks existing state before acting (idempotent)
- WAF, Bot Fight, rate limiting steps are best-effort — skip gracefully on free plan
- Always restart OLS after deploying cert: `lswsctrl restart`
