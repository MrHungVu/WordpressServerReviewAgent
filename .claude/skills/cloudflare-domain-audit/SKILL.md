---
name: cloudflare-domain-audit
description: Audit Cloudflare configuration for a domain using Cloudflare API. Checks DNS records, SSL/TLS mode, security settings, caching, performance, page rules, firewall rules, WAF managed rules, Bot Fight Mode, WooCommerce cache bypass rules, HTTP/3, rate limiting, and edge certificates. Produces a report with status levels and fix guidance. Use when user wants to check/verify/audit Cloudflare domain config.
allowed-tools:
  - Bash
  - Read
  - Write
  - WebSearch
---

# Cloudflare Domain Audit Skill

## Purpose
Audit a domain's Cloudflare configuration via API and produce a comprehensive report.

## Required Input (from user, NEVER save to files)
- Cloudflare email
- Cloudflare Global API Key (or scoped API Token)
- Domain name to audit

## API Access
Use `curl` with Cloudflare API v4. Base: `https://api.cloudflare.com/client/v4`

### Auth Headers (Global API Key)
```bash
-H "X-Auth-Email: EMAIL" -H "X-Auth-Key: API_KEY" -H "Content-Type: application/json"
```

### Auth Headers (API Token)
```bash
-H "Authorization: Bearer TOKEN" -H "Content-Type: application/json"
```

### Step 1: Get Zone ID
```bash
curl -s "https://api.cloudflare.com/client/v4/zones?name=DOMAIN" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.result[0].id'
```

## Audit Checklist

### 1. DNS Records
- `GET /zones/{zone_id}/dns_records`
- A/AAAA/CNAME point to correct server IP
- Proxy status (orange cloud) on appropriate records
- MX, TXT (SPF, DKIM, DMARC) for email
- CAA records for SSL
- No dangling/orphan records

### 2. SSL/TLS Settings
- `GET /zones/{zone_id}/settings/ssl` — must be "full" or "strict" (CRITICAL if "flexible")
- `GET /zones/{zone_id}/settings/always_use_https` — should be "on"
- `GET /zones/{zone_id}/settings/automatic_https_rewrites` — should be "on"
- `GET /zones/{zone_id}/settings/min_tls_version` — should be "1.2" minimum
- `GET /zones/{zone_id}/settings/tls_1_3` — should be "on"

### 3. Security Settings
- `GET /zones/{zone_id}/settings/security_level` — medium or high
- `GET /zones/{zone_id}/settings/browser_check` — should be "on"
- `GET /zones/{zone_id}/settings/challenge_ttl`
- `GET /zones/{zone_id}/settings/security_header`

### 4. Performance / Speed
- `GET /zones/{zone_id}/settings/brotli` — should be "on"
- `GET /zones/{zone_id}/settings/minify` — JS, CSS, HTML
- `GET /zones/{zone_id}/settings/early_hints`
- `GET /zones/{zone_id}/settings/rocket_loader` — WARNING: can break WordPress JS
- `GET /zones/{zone_id}/settings/h2_prioritization`
- `GET /zones/{zone_id}/settings/0rtt`

### 5. Caching
- `GET /zones/{zone_id}/settings/browser_cache_ttl` — recommended 14400+
- `GET /zones/{zone_id}/settings/cache_level`
- `GET /zones/{zone_id}/settings/development_mode` — must be "off" in production
- `GET /zones/{zone_id}/settings/always_online`

### 6. Page Rules
- `GET /zones/{zone_id}/pagerules`
- Should have: cache everything for static, bypass cache for wp-admin/wp-login
- SSL rules if needed

#### WooCommerce Page Rules Check
```bash
# Get all page rules
PAGE_RULES=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json")

# Check for WooCommerce bypass rules
CART_RULE=$(echo "$PAGE_RULES" | jq '[.result[] | select(.targets[0].constraint.value | contains("cart"))] | length')
CHECKOUT_RULE=$(echo "$PAGE_RULES" | jq '[.result[] | select(.targets[0].constraint.value | contains("checkout"))] | length')
MYACCOUNT_RULE=$(echo "$PAGE_RULES" | jq '[.result[] | select(.targets[0].constraint.value | contains("my-account"))] | length')

echo "WooCommerce cache bypass rules:"
echo "  /cart/*: $([ $CART_RULE -gt 0 ] && echo 'OK' || echo 'MISSING')"
echo "  /checkout/*: $([ $CHECKOUT_RULE -gt 0 ] && echo 'OK' || echo 'MISSING')"
echo "  /my-account/*: $([ $MYACCOUNT_RULE -gt 0 ] && echo 'OK' || echo 'MISSING')"
```
Flag as WARN if WooCommerce is detected on site but cache bypass page rules are missing.

### 7. Firewall Rules
- `GET /zones/{zone_id}/firewall/rules`
- Rate limiting rules
- Bot protection
- `GET /zones/{zone_id}/settings/waf`

#### WAF Managed Rules Check
```bash
# Get zone rulesets
RULESETS=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json")

# Check Cloudflare Managed Ruleset and OWASP
CF_WAF=$(echo "$RULESETS" | jq '[.result[] | select(.name | contains("Cloudflare Managed"))] | length')
OWASP_WAF=$(echo "$RULESETS" | jq '[.result[] | select(.name | contains("OWASP"))] | length')

echo "Cloudflare Managed WAF: $([ $CF_WAF -gt 0 ] && echo 'ENABLED' || echo 'NOT FOUND')"
echo "OWASP Core Ruleset: $([ $OWASP_WAF -gt 0 ] && echo 'ENABLED' || echo 'NOT FOUND')"
```
If rulesets returns empty or 403: report "WAF managed rules: requires Pro+ plan" (INFO, not WARN).

#### Bot Fight Mode Check
```bash
BOT_MGMT=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/bot_management" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json")

FIGHT_MODE=$(echo "$BOT_MGMT" | jq -r '.result.fight_mode // "unknown"')
echo "Bot Fight Mode: ${FIGHT_MODE}"
```
If API returns error (free plan limited), fallback to browser_check setting:
```bash
BROWSER_CHECK=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/browser_check" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" | jq -r '.result.value')
echo "Browser Integrity Check (fallback): ${BROWSER_CHECK}"
```

### 8. Edge Certificates
- `GET /zones/{zone_id}/ssl/certificate_packs`
- Universal SSL active
- Certificate validity dates

### 9. Network
- `GET /zones/{zone_id}/settings/ipv6`
- `GET /zones/{zone_id}/settings/websockets`
- `GET /zones/{zone_id}/settings/http3`

#### HTTP/3 Check with Recommendation
```bash
HTTP3=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/http3" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" | jq -r '.result.value')

echo "HTTP/3 (QUIC): ${HTTP3}"
if [ "$HTTP3" != "on" ]; then
  echo "RECOMMENDATION: Enable HTTP/3 for improved performance on modern browsers"
fi
```

### 10. Rate Limiting Rules
```bash
# Check rate limiting rulesets
RATE_RULES=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets/phases/http_ratelimit/entrypoint" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json" 2>/dev/null)

if echo "$RATE_RULES" | jq -e '.result.rules' &>/dev/null; then
  WP_LOGIN_RL=$(echo "$RATE_RULES" | jq '[.result.rules[] | select(.expression | contains("wp-login"))] | length')
  XMLRPC_RL=$(echo "$RATE_RULES" | jq '[.result.rules[] | select(.expression | contains("xmlrpc"))] | length')

  echo "Rate limiting - wp-login.php: $([ $WP_LOGIN_RL -gt 0 ] && echo 'CONFIGURED' || echo 'MISSING')"
  echo "Rate limiting - xmlrpc.php: $([ $XMLRPC_RL -gt 0 ] && echo 'CONFIGURED/BLOCKED' || echo 'MISSING')"
else
  echo "Rate limiting rules: not available on plan (INFO)"
fi
```

## Report Format
Save report to: `PROJECT_ROOT/vps-reports/cloudflare-audit-DOMAIN-YYYYMMDD.md`

```markdown
# Cloudflare Audit Report
**Domain:** example.com
**Zone ID:** xxxxx
**Date:** YYYY-MM-DD HH:MM
**Plan:** Free/Pro/Business/Enterprise
**Overall Status:** HEALTHY / NEEDS ATTENTION / CRITICAL

## Summary
| Category | Status | Issues |
|----------|--------|--------|
| DNS Records | OK/WARN/CRITICAL | brief |
| SSL/TLS | OK/WARN/CRITICAL | brief |
| Security | OK/WARN/CRITICAL | brief |
| Performance | OK/WARN/CRITICAL | brief |
| Caching | OK/WARN/CRITICAL | brief |
| Page Rules | OK/WARN/CRITICAL | brief |
| Firewall | OK/WARN/CRITICAL | brief |
| WAF Rules | OK/INFO/WARN | brief |
| Bot Protection | OK/WARN | brief |
| WooCommerce Rules | OK/WARN/N/A | brief |
| Rate Limiting | OK/WARN | brief |

## DNS Records
| Type | Name | Value | Proxied | Status |
|------|------|-------|---------|--------|
| A | @ | IP | Yes/No | OK/WARN |

## Detailed Findings
### [Category]
**Status:** OK/WARN/CRITICAL
**Current:** current setting value
**Recommended:** what it should be
**How to Fix:** Cloudflare dashboard navigation OR API curl command
**Why:** explanation

## WAF & Bot Protection
| Setting | Current | Recommended | Status |
|---------|---------|-------------|--------|
| Cloudflare Managed WAF | enabled/disabled/unavailable | enabled (Pro+) | OK/INFO |
| OWASP Core Ruleset | enabled/disabled/unavailable | enabled (Pro+) | OK/INFO |
| Bot Fight Mode | on/off | on | OK/WARN |
| Browser Integrity | on/off | on | OK/WARN |

## Rate Limiting
| Target | Status | Recommended |
|--------|--------|-------------|
| wp-login.php | configured/missing | 5 req/10s |
| xmlrpc.php | blocked/missing | block entirely |

## WooCommerce Cache Rules
| Page | Cache Bypass Rule | Status |
|------|-------------------|--------|
| /cart/* | present/missing | OK/WARN |
| /checkout/* | present/missing | OK/WARN |
| /my-account/* | present/missing | OK/WARN |

## Action Items
### Critical (Fix Now)
- [ ] item

### Warnings (Should Fix)
- [ ] item

### Recommendations (Nice to Have)
- [ ] item
```

## Important Rules
- NEVER save API keys or email to any file
- Use `jq` to parse JSON (install if needed: `sudo apt-get install -y jq`)
- If API call returns 403/404, note as "unavailable on current plan" and move on
- WAF managed rules and rate limiting returning 403/empty = INFO not WARN (requires Pro+ plan)
- Rocket Loader can break WordPress — always flag this
- Provide both dashboard path AND API curl command for fixes
- Cross-reference DNS A record with actual VPS IP if available
- WooCommerce page rule check: flag WARN only if WooCommerce is detected on the site
