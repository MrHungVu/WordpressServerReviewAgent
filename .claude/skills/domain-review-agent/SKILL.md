---
name: domain-review-agent
description: Orchestrating agent that runs vps-server-audit, cloudflare-domain-audit, and wordpress-site-audit skills to produce a full domain review. Collects credentials once, runs VPS + Cloudflare audits in parallel, then WordPress audit, and produces a combined final report. Use when user says "review domain", "check domain", "audit domain", or wants full site verification.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
  - Skill
---

# Domain Review Agent

## Purpose
Orchestrate all 3 audit skills to produce a comprehensive domain review. This is the main entry point users interact with.

## Trigger Phrases
- "review domain [domain]"
- "check domain [domain]"
- "audit [domain]"
- "verify [domain] setup"
- "is [domain] configured correctly?"

## Step 1: Collect Credentials
Ask user for ALL of these (use AskUserQuestion tool). NEVER save any of these to files:

### Required
1. **Domain name** (e.g., example.com)
2. **SSH host** (IP address)
3. **SSH user** (typically root)
4. **SSH password**
5. **SSH port** (default: 22)
6. **Cloudflare email**
7. **Cloudflare Global API Key** (or API Token)

### Optional
8. **WordPress install path** (auto-detected if not provided)
9. **WordPress admin URL** (for reference only)

## Step 2: Verify Connectivity
Before running audits, verify:
```bash
# Test SSH connection
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p PORT USER@HOST 'echo "SSH OK"'

# Test Cloudflare API
curl -s "https://api.cloudflare.com/client/v4/user" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.success'
```

If either fails, inform user and ask to verify credentials.

## Step 3: Run Audits (Parallel where possible)
Execute the 3 skills. VPS audit and Cloudflare audit can run in PARALLEL since they're independent. WordPress audit runs AFTER VPS audit (needs SSH + WP path from VPS findings).

### Parallel Phase (Skills 1 & 2)
1. **Activate `vps-server-audit` skill** — pass SSH credentials and domain
2. **Activate `cloudflare-domain-audit` skill** — pass Cloudflare credentials and domain

### Sequential Phase (Skill 3, after VPS audit completes)
3. **Activate `wordpress-site-audit` skill** — pass SSH credentials, domain, and WordPress path discovered during VPS audit

## Step 4: Cross-Reference Findings
After all 3 audits complete, cross-reference:
- **DNS ↔ Server**: Cloudflare A record IP matches actual VPS IP
- **SSL ↔ Cloudflare**: Cloudflare SSL mode matches server SSL config (Full Strict needs valid origin cert)
- **Cloudflare cache ↔ WordPress**: Page rules bypass cache for wp-admin
- **Server PHP ↔ WordPress**: PHP version compatible with WP + plugins
- **Firewall ↔ Cloudflare**: Server firewall allows Cloudflare IPs

## Step 5: Generate Final Report
Save to: `PROJECT_ROOT/vps-reports/domain-review-DOMAIN-YYYYMMDD.md`

```markdown
# Full Domain Review Report
**Domain:** example.com
**Date:** YYYY-MM-DD HH:MM
**Overall Health Score:** X/10

## Executive Summary
Brief 3-5 sentence overview of domain health across all 3 layers.

## Health Dashboard
| Layer | Status | Critical | Warnings | Score |
|-------|--------|----------|----------|-------|
| VPS Server | HEALTHY/ATTENTION/CRITICAL | count | count | X/10 |
| Cloudflare | HEALTHY/ATTENTION/CRITICAL | count | count | X/10 |
| WordPress | HEALTHY/ATTENTION/CRITICAL | count | count | X/10 |

## Cross-Reference Issues
Issues found by comparing findings across layers:
- [ ] issue description and fix

## Critical Actions (Fix Immediately)
Priority-ordered list from ALL 3 audits:
1. [ ] [LAYER] issue — fix command
2. [ ] [LAYER] issue — fix command

## Warnings (Fix Soon)
1. [ ] [LAYER] issue — fix
2. [ ] [LAYER] issue — fix

## Recommendations (Improve)
1. [ ] [LAYER] suggestion
2. [ ] [LAYER] suggestion

## Individual Reports
- [VPS Server Audit](./vps-audit-DOMAIN-YYYYMMDD.md)
- [Cloudflare Audit](./cloudflare-audit-DOMAIN-YYYYMMDD.md)
- [WordPress Audit](./wordpress-audit-DOMAIN-YYYYMMDD.md)
```

## Important Rules
- NEVER save credentials anywhere — keep only in conversation context
- Credentials are single-session only
- If a skill/audit fails, still produce report with available data
- Health score: 10 = perfect, deduct 2 per critical, 1 per warning
- Minimum score is 1 (never 0)
- Always cross-reference between layers — that's the value-add
- Present fixes in priority order across all layers
- Make all fix commands copy-paste ready
