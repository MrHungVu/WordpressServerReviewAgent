# Phase 7: cloudflare-domain-audit v2 (MINOR UPDATE)

## Context Links

- Current skill: `.claude/skills/cloudflare-domain-audit/SKILL.md`
- Brainstorm: [Audit Skills Impact](../reports/brainstorm-260323-1703-ols-stack-upgrade.md)

## Overview

- **Priority:** P2
- **Status:** complete
- **Effort:** 2h
- **Description:** Minor update. Add checks for WAF managed rules status, Bot Fight Mode, WooCommerce page rules (cart/checkout/my-account bypass), HTTP/3 enabled, and rate limiting rules for wp-login/xmlrpc. All existing checks stay unchanged.

## Key Insights

- All new checks are simple API GET calls + value validation
- WAF rulesets API: `GET /zones/{id}/rulesets` -- check if managed rulesets enabled
- Bot management: `GET /zones/{id}/bot_management` -- check fight_mode
- HTTP/3 already in the existing audit checklist (Section 9) but not validated with recommendations
- Rate limiting rules: `GET /zones/{id}/rulesets/phases/http_ratelimit/entrypoint`
- WooCommerce page rules: scan existing page rules for cart/checkout/my-account patterns

## Requirements

### Functional (NEW checks)
- F1: WAF managed rules status (Cloudflare + OWASP rulesets enabled/disabled)
- F2: Bot Fight Mode status
- F3: WooCommerce page rules check (cart/checkout/my-account bypass cache)
- F4: HTTP/3 enabled check (with recommendation)
- F5: Rate limiting rules for wp-login.php and xmlrpc.php

### Functional (KEEP existing)
- All v1 checks: DNS records, SSL/TLS settings, security settings, performance/speed, caching, page rules, firewall rules, edge certificates, network settings

## Architecture

No change. Same Cloudflare API audit flow.

## Related Code Files

- **Modify:** `.claude/skills/cloudflare-domain-audit/SKILL.md`

## Implementation Steps

### Step 1: Add WAF Managed Rules Check (extend Section 7)
```bash
# Get zone rulesets
RULESETS=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/rulesets" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json")

# Check Cloudflare Managed Ruleset
CF_WAF=$(echo "$RULESETS" | jq '[.result[] | select(.name | contains("Cloudflare Managed"))] | length')
OWASP_WAF=$(echo "$RULESETS" | jq '[.result[] | select(.name | contains("OWASP"))] | length')

echo "Cloudflare Managed WAF: $([ $CF_WAF -gt 0 ] && echo 'ENABLED' || echo 'NOT FOUND')"
echo "OWASP Core Ruleset: $([ $OWASP_WAF -gt 0 ] && echo 'ENABLED' || echo 'NOT FOUND')"
```
If rulesets empty or 403: "WAF managed rules: requires Pro+ plan" (INFO, not WARN).

### Step 2: Add Bot Fight Mode Check
```bash
BOT_MGMT=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/bot_management" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" -H "Content-Type: application/json")

FIGHT_MODE=$(echo "$BOT_MGMT" | jq -r '.result.fight_mode // "unknown"')
echo "Bot Fight Mode: ${FIGHT_MODE}"
```
If API returns error (free plan limited): check browser_check setting as fallback.
```bash
BROWSER_CHECK=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/browser_check" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" | jq -r '.result.value')
echo "Browser Integrity Check: ${BROWSER_CHECK}"
```

### Step 3: Add WooCommerce Page Rules Check (extend Section 6)
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
Flag as WARN if WooCommerce is detected on site but page rules missing.

### Step 4: Add HTTP/3 Check (extend Section 9)
HTTP/3 already listed in v1 Section 9 Network checks. Enhance with recommendation:
```bash
HTTP3=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/http3" \
  -H "X-Auth-Email: ${CF_EMAIL}" -H "X-Auth-Key: ${CF_KEY}" | jq -r '.result.value')

echo "HTTP/3 (QUIC): ${HTTP3}"
if [ "$HTTP3" != "on" ]; then
  echo "RECOMMENDATION: Enable HTTP/3 for improved performance on modern browsers"
fi
```

### Step 5: Add Rate Limiting Rules Check
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
  echo "Rate limiting rules: not available (may require higher plan)"
fi
```

### Step 6: Update Report Template
Add to Summary table:
```markdown
| WAF Rules | OK/INFO/WARN | brief |
| Bot Protection | OK/WARN | brief |
| WooCommerce Rules | OK/WARN/N/A | brief |
| Rate Limiting | OK/WARN | brief |
```

Add detailed section:
```markdown
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
```

## Todo List

- [x] Add WAF managed rules status check (Cloudflare + OWASP)
- [x] Add Bot Fight Mode check (with browser_check fallback)
- [x] Add WooCommerce page rules validation (cart/checkout/my-account)
- [x] Enhance HTTP/3 check with recommendation
- [x] Add rate limiting rules check (wp-login + xmlrpc)
- [x] Update report template with new sections
- [x] Handle plan-specific API limitations gracefully (INFO not ERROR)

## Success Criteria

- All new CF settings audited and reported
- WAF/Bot checks gracefully handle free plan limitations (INFO, not error)
- WooCommerce page rules flagged if missing
- HTTP/3 disabled flagged as recommendation
- Rate limiting status reported accurately

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| WAF API returns 403 on free plan | NONE | Report as "requires Pro plan" (INFO) |
| Bot management API changes | LOW | Fallback to browser_check |
| Page rules combined (cart+checkout in one rule) | LOW | Check for substring match, not exact |

## Security Considerations

- Read-only API calls, no changes made
- No new credential requirements
- All findings are recommendations, not automated fixes

## Next Steps

After this phase: Phase 8 (domain-review-agent + docs update) -- final phase.
