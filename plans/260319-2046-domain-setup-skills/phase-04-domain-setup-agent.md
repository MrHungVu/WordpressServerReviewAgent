# Phase 4: Domain Setup Agent (Orchestrator)

## Context
- Parent plan: [plan.md](./plan.md)
- Depends on: Phases 1-3 (all sub-skills must be created first)
- Reference: `.claude/skills/domain-review-agent/SKILL.md` (audit orchestrator pattern)

## Overview
- **Priority:** P1
- **Status:** pending
- **Description:** Create `domain-setup-agent` orchestrator skill that collects credentials, verifies connectivity, runs the 3 setup sub-skills sequentially, and produces a final setup report with all generated credentials and manual steps.

## Key Insights
- Mirror the `domain-review-agent` orchestrator pattern closely
- Sequential execution (not parallel) — each skill depends on previous
- Must capture auto-generated credentials from each sub-skill
- Final report is the ONLY place credentials are delivered to user
- Manual steps section must be very clear — exact nameserver values
- Suggest running `review domain` afterward to verify setup

## Requirements

### Functional
- Collect all credentials via AskUserQuestion: domain, SSH (host/user/pass/port), CF (email/API key), WP admin email
- Verify SSH connectivity before proceeding
- Verify Cloudflare API access before proceeding
- Run sequentially: vps-server-setup → wordpress-site-setup → cloudflare-domain-setup
- Capture outputs: server versions, WP credentials, DB credentials, CF nameservers
- Generate setup report to `vps-reports/domain-setup-{domain}-{YYYYMMDD}.md`
- Include credential table, installed versions, manual steps, security checklist
- Handle partial failure: report what succeeded, what failed, how to retry

### Non-Functional
- Credentials NEVER saved to files — conversation context only
- Single-session credential handling
- Report clearly marked as sensitive (contains credentials)
- Total setup time target: <10 minutes

## Architecture

### SKILL.md Structure
```
---
name: domain-setup-agent
description: Orchestrating agent that runs vps-server-setup, wordpress-site-setup, and cloudflare-domain-setup skills to set up a complete WordPress site. Collects credentials once, runs all setup steps sequentially, and produces a setup report with credentials. Use when user says "setup domain", "install WordPress", or wants automated VPS + WordPress + Cloudflare setup.
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
```

### Orchestration Flow
```
1. AskUserQuestion → collect all credentials
2. Test SSH → test CF API → both OK?
3. Activate vps-server-setup → capture: PHP ver, MariaDB ver, Nginx ver, server IP
4. Activate wordpress-site-setup → capture: WP admin URL/user/pass, DB name/user/pass
5. Activate cloudflare-domain-setup → capture: nameservers, zone ID, settings applied
6. Generate final report with everything
```

### Trigger Phrases
- "setup domain [domain]"
- "install WordPress on [domain]"
- "set up [domain]"
- "configure [domain]"
- "deploy WordPress to [domain]"

### Report Template
```markdown
# Domain Setup Report
**Domain:** example.com
**Date:** YYYY-MM-DD HH:MM
**Status:** COMPLETE / PARTIAL

## ⚠️ SAVE THESE CREDENTIALS
| Item | Value |
|------|-------|
| WP Admin URL | https://example.com/wp-admin |
| WP Admin User | wp_admin_xxxx |
| WP Admin Password | [generated] |
| DB Name | wp_example_xxxx |
| DB User | wp_usr_xxxx |
| DB Password | [generated] |

## What Was Installed
| Component | Version |
|-----------|---------|
| Nginx | X.X |
| PHP-FPM | X.X |
| MariaDB | X.X |
| WordPress | X.X |

## Cloudflare Configuration
- Zone ID: xxxxx
- SSL Mode: Full (Strict)
- Origin Certificate: Installed (expires 2041)
- Settings: Always HTTPS, TLS 1.2+, HSTS, Brotli

## ✅ Manual Steps Required
1. Go to your domain registrar (GoDaddy/Namecheap/Cloudflare Registrar/etc.)
2. Change nameservers to:
   - `ns1.cloudflare.com` (actual values from API)
   - `ns2.cloudflare.com` (actual values from API)
3. Wait 24-48 hours for DNS propagation
4. Log in to https://example.com/wp-admin and change your password
5. Verify: Run "review domain example.com" to audit the setup

## Security Applied
- [x] DISALLOW_FILE_EDIT = true
- [x] WP_DEBUG = false
- [x] Non-default DB prefix
- [x] XML-RPC blocked (Nginx)
- [x] REST API user enumeration blocked
- [x] File permissions hardened (644/755/600)
- [x] UFW firewall (ports 22, 80, 443 only)
- [x] SSL Full (Strict) with Origin Certificate
- [x] Default content removed

## Post-Setup Recommendations
- [ ] Install backup plugin (UpdraftPlus or similar)
- [ ] Install security plugin (Wordfence or similar)
- [ ] Configure SMTP for email delivery
- [ ] Set up automated backups
- [ ] Consider CDN caching page rules for static content
```

## Related Code Files
- **Create:** `.claude/skills/domain-setup-agent/SKILL.md`
- **Reference:** `.claude/skills/domain-review-agent/SKILL.md` (audit orchestrator)

## Implementation Steps

1. Create directory `.claude/skills/domain-setup-agent/`
2. Write `SKILL.md` with frontmatter (include `Skill` in allowed-tools for sub-skill activation)
3. Write Purpose and Trigger Phrases sections
4. Write Step 1 — Credential Collection:
   - AskUserQuestion with all fields
   - Domain, SSH host/user/pass/port, CF email/API key, WP admin email
5. Write Step 2 — Connectivity Verification:
   - SSH test command
   - CF API test command
   - Failure → ask user to verify, don't proceed
6. Write Step 3 — Run Setup Skills:
   - Sequential: vps-server-setup → wordpress-site-setup → cloudflare-domain-setup
   - Pass credentials and outputs between skills
   - Capture generated credentials from each skill
7. Write Step 4 — Generate Final Report:
   - Report template with credential table
   - Manual steps with actual nameserver values
   - Security checklist
   - Post-setup recommendations
8. Write Partial Failure Handling section
9. Write Important Rules section (credential security)

## Todo List
- [ ] Create `.claude/skills/domain-setup-agent/` directory
- [ ] Write SKILL.md with frontmatter
- [ ] Trigger phrases section
- [ ] Credential collection via AskUserQuestion
- [ ] SSH connectivity verification
- [ ] Cloudflare API connectivity verification
- [ ] Sequential skill activation (vps → wp → cf)
- [ ] Credential capture between skills
- [ ] Final report generation with template
- [ ] Manual steps section with nameserver instructions
- [ ] Partial failure handling
- [ ] Post-setup recommendation to run audit

## Success Criteria
- User runs "setup domain example.com" → gets working WordPress site
- All credentials delivered in a single report
- Manual steps clearly explain nameserver change
- Re-running doesn't break existing setup (idempotent sub-skills)
- Partial failure produces useful report (what worked, what didn't)
- Running audit skill afterward scores 8+/10

## Risk Assessment
| Risk | Impact | Mitigation |
|------|--------|------------|
| Sub-skill fails mid-setup | HIGH | Report partial success, suggest retry |
| Credential not captured between skills | HIGH | Verify output format from each sub-skill |
| User forgets to save credentials | MEDIUM | Report clearly warns to save credentials |
| Nameserver change takes 48h | EXPECTED | Document in report, site works after propagation |

## Security Considerations
- Report contains plaintext credentials — warn user to save securely
- Credentials exist only in conversation context
- Never save credentials to any file except the one-time report
- Report file should be deleted after user saves credentials
- Remind user to change WP admin password on first login

## Next Steps
→ Phase 5: Registration & documentation updates
