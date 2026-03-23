---
title: "OLS v2.0 Stack Upgrade with Multi-Site Support"
description: "Rewrite all 8 WordPress skills from Nginx/LEMP to OpenLiteSpeed + LSPHP + Redis + OPcache with multi-site"
status: complete
priority: P1
effort: 40h
branch: main
tags: [openlitespeed, multi-site, rewrite, v2.0, wordpress, redis, lsphp]
created: 2026-03-23
---

# OLS v2.0 Stack Upgrade Plan

## Summary

Full rewrite of 4 setup skills + update of 4 audit skills from Nginx/LEMP to OpenLiteSpeed CLI-only stack. Adds multi-site support (1 VPS -> N WordPress sites), Redis object cache per site, OPcache JIT, adaptive memory tuning, Cloudflare-only access, custom wp-admin URL, WooCommerce, and local backups.

## Research

- [Brainstorm & Gap Analysis](../reports/brainstorm-260323-1703-ols-stack-upgrade.md)
- [OLS Production Setup](../reports/researcher-260323-1708-openlitespeed-wordpress-production.md)
- [Multi-Site Best Practices](../reports/researcher-260323-1708-multi-site-openlitespeed-best-practices.md)

## Phases

| # | Phase | Type | Effort | Status | File |
|---|-------|------|--------|--------|------|
| 1 | vps-server-setup v2 | MAJOR REWRITE | 10h | complete | [phase-01](phase-01-vps-server-setup.md) |
| 2 | wordpress-site-setup v2 | MAJOR REWRITE | 10h | complete | [phase-02](phase-02-wordpress-site-setup.md) |
| 3 | cloudflare-domain-setup v2 | MODERATE UPDATE | 4h | complete | [phase-03](phase-03-cloudflare-domain-setup.md) |
| 4 | domain-setup-agent v2 | ORCHESTRATOR UPDATE | 4h | complete | [phase-04](phase-04-domain-setup-agent.md) |
| 5 | vps-server-audit v2 | UPDATE | 3h | complete | [phase-05](phase-05-vps-server-audit.md) |
| 6 | wordpress-site-audit v2 | UPDATE | 3h | complete | [phase-06](phase-06-wordpress-site-audit.md) |
| 7 | cloudflare-domain-audit v2 | MINOR UPDATE | 2h | complete | [phase-07](phase-07-cloudflare-domain-audit.md) |
| 8 | docs + domain-review-agent v2 | UPDATE | 4h | complete | [phase-08](phase-08-docs-and-review-agent.md) |

## Execution Order

Phases 1-4 are strictly sequential (each depends on prior). Phases 5-7 can parallelize after Phase 1. Phase 8 is last.

## Key Decisions

- OLS CLI-only, no CyberPanel/WebAdmin
- LSPHP 8.2 from LiteSpeed repo
- Shared binary + separate worker pools per site
- Single Redis, DB per site (0-15) + key prefix
- Adaptive memory: 2GB/4GB/8GB+ profiles
- WooCommerce always installed
- ModSecurity deferred to v2.1
- Local backup + cron, 30-day retention

## Validation Log

### Session 1 — 2026-03-23
**Trigger:** Initial plan creation validation
**Questions asked:** 8

#### Questions & Answers

1. **[Scope]** Phase 1 assumes a FRESH VPS each time. Should vps-server-setup v2 also handle upgrading an existing Nginx v1 VPS to OLS in-place?
   - Options: Fresh VPS only | Support both fresh + migration | Fresh only + migration guide
   - **Answer:** Fresh VPS only, but detect if OLS/WordPress already installed and install new site (multi-site)
   - **Custom input:** "Fresh vps only, detect if OLS and wordpress installed then installed new site, because remember we use mutiple site"
   - **Rationale:** vps-server-setup must be idempotent — first run installs stack, subsequent runs detect existing OLS and skip install. Critical for multi-site: user adds site #2 later on same VPS.

2. **[Config Safety]** Phase 2 appends to httpd_config.conf for each site. If a site setup fails mid-append, config may be corrupted. How to handle?
   - Options: Backup + validate + rollback | Atomic write via temp file | Validate-only, no rollback
   - **Answer:** Backup + validate + rollback + never crash old sites
   - **Custom input:** "backup + validate + roolback + never crash old site, site running real, find a way"
   - **Rationale:** Production sites running on VPS. Any config corruption takes ALL sites down. Must use staging config approach.

3. **[Login Security]** Blocking direct /wp-login.php breaks password reset emails. Is partial workaround (allow rp/resetpass/logout) acceptable?
   - Options: Allow rp/resetpass/logout actions | Block all, rewrite emails | Rate-limit instead of blocking
   - **Answer:** Allow rp/resetpass/logout actions (Recommended)
   - **Rationale:** Balances security (blocks scanners) with functionality (password resets work). No extra mu-plugin needed.

4. **[OS Support]** CentOS/RHEL support — how much effort?
   - Options: Ubuntu/Debian only | Ubuntu primary + basic CentOS | Ubuntu only, fail on CentOS
   - **Answer:** Ubuntu primary + basic CentOS
   - **Rationale:** Some VPS providers still default to CentOS. Need yum equivalents for each apt command. Doubles testing but expands compatibility.

5. **[Idempotent]** vps-server-setup should be idempotent across runs — detect existing stack and skip installation?
   - Options: Yes, detect + skip | Always fresh install (wipe)
   - **Answer:** Yes, detect + skip installed (Recommended)
   - **Rationale:** Essential for multi-site. Second site added weeks later — must not re-install OLS/MariaDB/Redis. Detect and return existing config.

6. **[Config Safety]** Staging config approach for safe multi-site appending?
   - Options: Staging file + validate + swap | Backup + direct append + validate | Include-based isolation
   - **Answer:** Staging file + validate + swap (Recommended)
   - **Rationale:** Zero downtime risk. Changes validated before touching production config. Failed validation = no change.

7. **[Backup Creds]** Backup script embeds DB password in plaintext. Alternative?
   - Options: Use .my.cnf per-site | Keep plaintext, chmod 700 | Use wp-cli db export
   - **Answer:** Use .my.cnf per-site (Recommended)
   - **Rationale:** /var/www/{domain}/.my.cnf with [mysqldump] creds, chmod 600. No plaintext password visible in script. Standard MySQL pattern.

8. **[Credentials]** Adding site #2 to existing VPS — how to get MariaDB root password?
   - Options: Ask user each time | Store encrypted on server | Skip root pw, use manager user
   - **Answer:** Ask user each time (Recommended)
   - **Rationale:** User must have saved root pw from initial setup report. No credential persistence on disk. Simple, secure.

#### Confirmed Decisions
- **Fresh VPS + idempotent stack detect:** vps-server-setup detects existing OLS, skips install, returns config — supports multi-site additions
- **Staging config approach:** httpd_config.conf changes via staging file → validate → atomic swap. Zero risk to running sites
- **Login URL security:** Allow rp/resetpass/logout query params through, block scanners
- **OS support:** Ubuntu primary + basic CentOS (yum equivalents)
- **Backup creds:** Per-site .my.cnf files instead of plaintext in scripts
- **Multi-site credential flow:** Ask user for MariaDB root pw each time (no server persistence)

#### Action Items
- [ ] Phase 1: Add idempotent detection logic (check `systemctl is-active lsws`, skip install if active)
- [ ] Phase 1: Add CentOS/RHEL `yum` equivalents for all `apt` commands
- [ ] Phase 2: Replace direct httpd_config.conf append with staging file + validate + swap
- [ ] Phase 2: Replace plaintext backup script creds with per-site .my.cnf
- [ ] Phase 4: Add MariaDB root password to credential collection (for existing VPS)

#### Impact on Phases
- **Phase 1:** Add idempotent stack detection at start (check OLS/MariaDB/Redis running, skip install if present). Add CentOS `yum` branches for all package installs. Add step to ask for existing MariaDB root pw if detected.
- **Phase 2:** Replace Step 11 direct append with staging config pattern: copy → append → validate → swap. Replace Step 15 backup script plaintext password with .my.cnf approach.
- **Phase 4:** Update credential collection to ask for MariaDB root password when VPS is not fresh (detected as existing setup)
