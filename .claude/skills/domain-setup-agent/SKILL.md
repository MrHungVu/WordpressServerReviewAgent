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

# Domain Setup Agent

## Purpose
Orchestrate all 3 setup skills to deliver a complete WordPress site from bare VPS to live domain. This is the single entry point users interact with for new site provisioning.

## Trigger Phrases
- "setup domain [domain]"
- "install WordPress on [domain]"
- "set up [domain]"
- "configure [domain]"
- "deploy WordPress to [domain]"

## Step 1: Collect Credentials
Ask user for ALL of these (use AskUserQuestion tool). NEVER save any of these to files:

### Required
1. **Domain name** (e.g., example.com)
2. **SSH host** (IP address)
3. **SSH user** (typically root)
4. **SSH password**
5. **SSH port** (default: 22)
6. **Cloudflare email**
7. **Cloudflare Global API Key**
8. **WordPress admin email**

## Step 2: Verify Connectivity
Before running setup, verify both connections succeed:
```bash
# Test SSH connection
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p PORT USER@HOST 'echo "SSH OK"'

# Test Cloudflare API
curl -s "https://api.cloudflare.com/client/v4/user" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.success'
```

If either fails, inform user and ask to verify credentials. Do NOT proceed until both pass.

If `sshpass` is not available: `brew install hudochenkov/sshpass/sshpass` (macOS) or `apt install sshpass`.
If `jq` is not available: `brew install jq` (macOS) or `apt install jq`.

## Step 3: Run Setup Skills (Sequential)
Each skill depends on outputs from the previous. Run in strict order.

### 3a. Activate `vps-server-setup` skill
Pass: SSH credentials, domain name.
Capture from output:
- Server IP
- Nginx version
- PHP-FPM version and socket path
- MariaDB version and generated root password

### 3b. Activate `wordpress-site-setup` skill
Pass: SSH credentials, domain, MariaDB root password (from 3a), WordPress admin email.
Capture from output:
- WordPress install path
- WordPress version
- WP admin URL, username, password
- DB name, DB user, DB password

### 3c. Activate `cloudflare-domain-setup` skill
Pass: Cloudflare credentials, domain, server IP (from 3a), SSH credentials (for origin certificate install).
Capture from output:
- Nameserver 1 and Nameserver 2
- Zone ID
- SSL mode applied
- Settings applied

## Step 4: Generate Final Report
Save to: `PROJECT_ROOT/vps-reports/domain-setup-{domain}-{YYYYMMDD}.md`

```markdown
# Domain Setup Report
**Domain:** {domain}
**Date:** YYYY-MM-DD HH:MM
**Status:** COMPLETE / PARTIAL

## SAVE THESE CREDENTIALS
| Item | Value |
|------|-------|
| WP Admin URL | https://{domain}/wp-admin |
| WP Admin User | {from wordpress-site-setup} |
| WP Admin Password | {from wordpress-site-setup} |
| DB Name | {from wordpress-site-setup} |
| DB User | {from wordpress-site-setup} |
| DB Password | {from wordpress-site-setup} |
| MariaDB Root Password | {from vps-server-setup} |

## What Was Installed
| Component | Version |
|-----------|---------|
| Nginx | {from vps-server-setup} |
| PHP-FPM | {from vps-server-setup} |
| MariaDB | {from vps-server-setup} |
| WordPress | {from wordpress-site-setup} |

## Cloudflare Configuration
- Zone ID: {from cloudflare-domain-setup}
- SSL Mode: Full (Strict)
- Origin Certificate: Installed
- Settings: Always HTTPS, TLS 1.2+, HSTS, Brotli

## ACTION REQUIRED — Manual Steps
1. Go to your domain registrar
2. Change nameservers to:
   - {nameserver1 from cloudflare-domain-setup}
   - {nameserver2 from cloudflare-domain-setup}
3. Wait 24-48 hours for DNS propagation
4. Log in to https://{domain}/wp-admin and change your password
5. Verify setup: Run "review domain {domain}" to audit

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
- [x] fail2ban active
- [x] SSH hardening applied

## Post-Setup Recommendations
- [ ] Install backup plugin (UpdraftPlus)
- [ ] Install security plugin (Wordfence)
- [ ] Configure SMTP for email delivery
- [ ] Set up automated backups
```

## Partial Failure Handling
- If `vps-server-setup` fails: report what succeeded, do NOT run subsequent skills, set Status to PARTIAL
- If `wordpress-site-setup` fails: report VPS setup success, skip `cloudflare-domain-setup`, set Status to PARTIAL
- If `cloudflare-domain-setup` fails: report VPS + WP success, note Cloudflare needs manual setup, set Status to PARTIAL
- Always produce a report with whatever data is available

## Important Rules
- Credentials exist only in conversation context during the session
- The setup report is a one-time credential delivery mechanism — warn user to save credentials from it immediately
- The report file is saved to `vps-reports/` (already in .gitignore) — user should delete after saving credentials
- Sequential execution only — each skill depends on previous outputs
- After setup completes, suggest running "review domain {domain}" to verify the installation
- Health indicators: COMPLETE = all 3 skills succeeded; PARTIAL = one or more failed
