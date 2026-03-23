# Phase 8: domain-review-agent v2 + Documentation Update

## Context Links

- Current skill: `.claude/skills/domain-review-agent/SKILL.md`
- Current docs: `docs/system-architecture.md`, `docs/code-standards.md`, `docs/project-overview-pdr.md`, `docs/project-changelog.md`, `docs/development-roadmap.md`
- Root: `CLAUDE.md`
- Depends on: All phases 1-7 complete

## Overview

- **Priority:** P2
- **Status:** complete
- **Effort:** 4h
- **Description:** Update domain-review-agent orchestrator with OLS-specific cross-validations. Update all documentation to reflect OLS v2.0 stack. Add changelog v2.0 entry, update system architecture diagrams, update code standards, update PDR known limitations, update roadmap phases.

## Key Insights

- Domain-review-agent cross-validation is the main value-add over individual audits
- New cross-validations needed: OLS SSL cert paths, CF-Connecting-IP config, LSPHP+WP compatibility, Redis DB assignments across sites, LSAPI_CHILDREN within RAM budget
- Documentation update is straightforward text replacement + section additions
- CLAUDE.md needs stack description updated (Nginx -> OLS)

## Requirements

### Functional (domain-review-agent updates)
- F1: Cross-validate OLS SSL cert paths match Cloudflare origin cert
- F2: Verify CF-Connecting-IP configured in OLS when behind Cloudflare
- F3: Check LSPHP version compatible with WordPress + all active plugins
- F4: Validate Redis DB assignments across multiple sites (no duplicates)
- F5: Validate total LSAPI_CHILDREN across all sites fits within RAM budget
- F6: Check per-site backup scripts exist for all detected WordPress installations

### Functional (documentation updates)
- F7: Update `docs/system-architecture.md` -- OLS stack, multi-site architecture
- F8: Update `docs/code-standards.md` -- skill structure notes for v2
- F9: Update `docs/project-overview-pdr.md` -- known limitations, requirements F5-F8
- F10: Update `docs/project-changelog.md` -- add v2.0 entry
- F11: Update `docs/development-roadmap.md` -- Phase 1.2 OLS upgrade
- F12: Update `CLAUDE.md` -- stack description

## Architecture

No structural change to review agent. Same 3-audit orchestration + cross-validation + report.

## Related Code Files

- **Modify:** `.claude/skills/domain-review-agent/SKILL.md`
- **Modify:** `docs/system-architecture.md`
- **Modify:** `docs/code-standards.md`
- **Modify:** `docs/project-overview-pdr.md`
- **Modify:** `docs/project-changelog.md`
- **Modify:** `docs/development-roadmap.md`
- **Modify:** `CLAUDE.md`

## Implementation Steps

### Part A: domain-review-agent Updates

#### Step 1: Add OLS SSL Cert Cross-Validation
In Step 4 (Cross-Reference Findings), add:
```
# OLS SSL cert path matches Cloudflare origin cert
- VPS audit: check /usr/local/lsws/conf/cert/{domain}.crt exists
- CF audit: origin cert deployed with Full Strict mode
- Cross-check: cert file exists AND SSL mode is strict
- If OLS has cert but CF SSL not strict -> WARN: downgrade risk
- If CF is strict but no cert on OLS -> CRITICAL: SSL will fail
```

#### Step 2: Add CF-Connecting-IP Cross-Validation
```
# CF-Connecting-IP configured when behind Cloudflare
- VPS audit: useClientIpInHeader setting in httpd_config.conf
- CF audit: domain proxied (orange cloud) on DNS records
- Cross-check: if proxied AND useClientIpInHeader missing -> WARN: logs show CF IPs
- If not proxied -> skip this check
```

#### Step 3: Add LSPHP + WordPress Compatibility Check
```
# LSPHP version compatible with WordPress + plugins
- VPS audit: LSPHP version detected
- WP audit: WordPress version + plugin requirements
- Cross-check:
  - PHP 8.2 supports WP 6.2+
  - Check if any plugin warns about PHP version
  - Flag if LSPHP < 8.1 (no JIT, older WP support)
```

#### Step 4: Add Redis DB Assignment Validation
```
# Redis DB assignments across sites (no duplicates, within 0-15)
- WP audit (per site): WP_REDIS_DATABASE value from wp-config.php
- VPS audit: redis-cli per-DB key counts
- Cross-check:
  - No two sites share same Redis DB number
  - All DB numbers within 0-15 range
  - DB numbers with keys match sites that have Redis configured
  - Any DB with keys but no matching site -> WARN: orphan cache
```

#### Step 5: Add LSAPI_CHILDREN RAM Budget Validation
```
# Total PHP workers within RAM budget
- VPS audit: total RAM, LSAPI_CHILDREN per ext app
- Formula: total_workers = sum(LSAPI_CHILDREN per site)
  estimated_php_mem = total_workers * 50MB (average WP worker)
  available_ram = total_ram - OS(500) - MariaDB(pool) - Redis(max) - OPcache - OLS(200)
- Cross-check:
  - estimated_php_mem < available_ram -> OK
  - estimated_php_mem >= available_ram -> WARN: memory overcommit risk
  - Suggest reducing LSAPI_CHILDREN or upgrading RAM
```

#### Step 6: Add Backup Coverage Validation
```
# All sites have backup scripts and recent backups
- WP audit (per site): backup script exists + cron + recent files
- Cross-check:
  - Every detected WordPress installation has a backup script
  - Every backup cron ran within last 48 hours
  - Any site without backup -> WARN
```

#### Step 7: Update Report Cross-Reference Section
Add to the Cross-Reference Issues table:
```markdown
## Cross-Reference Issues
| Check | VPS | Cloudflare | WordPress | Status |
|-------|-----|------------|-----------|--------|
| DNS A Record | {ip} | {ip} | - | OK/MISMATCH |
| SSL Mode | cert at OLS path | Full Strict | FORCE_SSL_ADMIN | OK/WARN |
| HTTPS | OLS listener 443 | Always HTTPS | siteurl=https | OK/WARN |
| CF-Connecting-IP | configured | proxied | - | OK/WARN |
| LSPHP ↔ WP | LSPHP 8.2 | - | WP 6.X | OK/WARN |
| Redis Isolation | DB 0,1,2... | - | WP_REDIS_DATABASE | OK/WARN |
| Memory Budget | {ram}MB avail | - | {workers}x50MB est | OK/WARN |
| Backup Coverage | scripts exist | - | per-site cron | OK/WARN |
```

### Part B: Documentation Updates

#### Step 8: Update system-architecture.md
Major changes:
- Replace "LEMP stack (Nginx + PHP-FPM + MariaDB)" with "OLS stack (OpenLiteSpeed + LSPHP + MariaDB + Redis)"
- Update Setup Workflow diagram: replace "Install LEMP + firewall" with "Install OLS + LSPHP + MariaDB + Redis + firewall"
- Update VPS Server Setup component table: replace Nginx with OLS commands
- Add multi-site architecture diagram showing N vhosts + LSPHP pools + Redis DBs
- Update data flow for multi-site credential capture
- Add Redis + OPcache to Technology Stack table
- Update "Scalability Considerations" current section for multi-site
- Bump version to 2.0

#### Step 9: Update code-standards.md
Changes:
- Update Project Structure to note SKILL.md files (not instructions.md+tests.md+examples.md)
- Add note about OLS config syntax in skill implementation
- Update "Skill Instructions Template" required sections for v2 (add: RAM profile awareness, multi-site awareness)
- Add note about LSPHP WP-CLI wrapper pattern
- Bump version to 2.0

#### Step 10: Update project-overview-pdr.md
Changes:
- Update F5 (VPS Server Setup): "LEMP stack" -> "OLS stack (OpenLiteSpeed + LSPHP 8.2 + MariaDB + Redis)"
- Update F6 (WordPress Site Setup): add multi-site, Redis, LSCache, WooCommerce, custom login URL
- Update F7 (Cloudflare Domain Setup): add WAF, Bot Fight, rate limiting, WooCommerce rules, HTTP/3
- Update F8 (Domain Setup Orchestrator): add multi-site flow, adaptive memory, site numbering
- Update Known Limitations:
  - Remove: "OpenLiteSpeed Support: Requires manual LSPHP path configuration" (now native)
  - Add: "Maximum 16 sites per VPS (Redis DB 0-15 limit)"
  - Add: "ModSecurity deferred to v2.1"
  - Add: "No per-site OS user isolation (shared www-data, planned for v2.1)"
- Update Technology Stack: add Redis, OPcache, LSPHP
- Bump version to 2.0

#### Step 11: Update project-changelog.md
Add new entry at top:

```markdown
## [2.0.0] — 2026-03-XX

### Major Changes — OLS Stack with Multi-Site Support

- **VPS Server Setup v2** (`vps-server-setup`) — MAJOR REWRITE
  - Replaced Nginx + PHP-FPM with OpenLiteSpeed + LSPHP 8.2
  - Added Redis installation + adaptive configuration
  - Added OPcache JIT tuning (adaptive by RAM)
  - Added Cloudflare-only UFW firewall (auto-fetch CF IPs)
  - Added optional: sudo user creation, SSH key auth, custom SSH port
  - Added fail2ban OLS access log jail
  - Added adaptive memory profiling (2GB/4GB/8GB+)
  - WebAdmin console disabled by default

- **WordPress Site Setup v2** (`wordpress-site-setup`) — MAJOR REWRITE
  - Multi-site aware: call N times for N sites on same VPS
  - OLS vhost config with isolated LSPHP worker pool per site
  - Redis object cache per site (DB 0-15 isolation)
  - LiteSpeed Cache plugin auto-configured
  - WooCommerce always installed with LSCache exclusions
  - Custom wp-admin URL via OLS rewrite rules
  - Per-site backup script with 30-day retention cron
  - Per-site php.ini with adaptive memory limits

- **Cloudflare Domain Setup v2** (`cloudflare-domain-setup`) — MODERATE UPDATE
  - Origin cert path: /usr/local/lsws/conf/cert/ (was /etc/ssl/cloudflare/)
  - Removed Nginx SSL config section (OLS handles SSL natively)
  - Added WAF managed rules enable
  - Added Bot Fight Mode enable
  - Added rate limiting for wp-login.php + xmlrpc.php
  - Added WooCommerce page rules (cart/checkout/my-account bypass)
  - Added HTTP/3 (QUIC) enable

- **Domain Setup Agent v2** (`domain-setup-agent`) — ORCHESTRATOR UPDATE
  - Multi-site flow: VPS setup once, then loop per domain
  - Adaptive LSAPI_CHILDREN calculation based on RAM + site count
  - Redis DB assignment (sequential: site0=DB0, site1=DB1...)
  - Partial failure handling (continue other sites if one fails)
  - Consolidated multi-site setup report

- **VPS Server Audit v2** (`vps-server-audit`) — UPDATE
  - OLS detection + LSPHP auto-detect
  - OLS config parsing (listeners, vhosts, ext apps)
  - Redis status + per-DB key counts
  - OPcache stats via LSPHP
  - UFW Cloudflare-only rules validation
  - WebAdmin disabled check + CF-Connecting-IP check

- **WordPress Site Audit v2** (`wordpress-site-audit`) — UPDATE
  - LSPHP auto-detect for WP-CLI
  - LSCache plugin + Redis config validation
  - Custom login URL check
  - WooCommerce audit with LSCache exclusion check
  - Per-site backup status check

- **Cloudflare Domain Audit v2** (`cloudflare-domain-audit`) — MINOR UPDATE
  - WAF managed rules status check
  - Bot Fight Mode check
  - WooCommerce page rules validation
  - HTTP/3 enabled check
  - Rate limiting rules check

- **Domain Review Agent v2** (`domain-review-agent`) — UPDATE
  - OLS SSL cert path cross-validation
  - CF-Connecting-IP cross-validation
  - LSPHP ↔ WordPress compatibility check
  - Redis DB assignment validation (no duplicates)
  - LSAPI_CHILDREN RAM budget validation
  - Backup coverage cross-check
```

Update version timeline table.

#### Step 12: Update development-roadmap.md
Add Phase 1.2:
```markdown
### Phase 1.2: OLS Stack Upgrade with Multi-Site (IN PROGRESS — 2026-03-XX)

**Status:** In Progress

**Objectives:**
- Migrate all 8 skills from Nginx/LEMP to OpenLiteSpeed + LSPHP + Redis
- Add multi-site support (1 VPS -> N WordPress sites)
- Add adaptive memory tuning, Cloudflare-only firewall, custom login URLs
- Add WooCommerce support with cache management

**Deliverables:**
- [ ] vps-server-setup v2 (OLS + LSPHP + MariaDB + Redis)
- [ ] wordpress-site-setup v2 (multi-site, LSCache, WooCommerce)
- [ ] cloudflare-domain-setup v2 (WAF, bot fight, rate limiting)
- [ ] domain-setup-agent v2 (multi-site orchestrator)
- [ ] vps-server-audit v2 (OLS detection)
- [ ] wordpress-site-audit v2 (LSPHP, LSCache, Redis)
- [ ] cloudflare-domain-audit v2 (WAF, WooCommerce rules)
- [ ] domain-review-agent v2 (OLS cross-validations)
- [ ] Documentation updated for v2.0
```

Update Phase 2 description to remove "Expand server type support (beyond OpenLiteSpeed)" since OLS is now primary.

#### Step 13: Update CLAUDE.md
Changes:
- Replace "4 audit skills + 4 setup skills" description
- Update Setup Skills descriptions:
  - `vps-server-setup`: "SSH into VPS, install OLS + LSPHP 8.2 + MariaDB + Redis, configure Cloudflare-only firewall + SSH hardening + adaptive memory tuning"
  - `wordpress-site-setup`: "Install WordPress via WP-CLI + LSPHP, create DB, OLS vhost with isolated LSPHP pool, Redis object cache, LSCache, WooCommerce, custom login URL, per-site backup"
  - `cloudflare-domain-setup`: "Cloudflare API setup (zone, DNS, SSL Full Strict + Origin Cert, WAF, bot fight, rate limiting, WooCommerce page rules, HTTP/3)"
  - `domain-setup-agent`: "Multi-site orchestrator: runs VPS setup once, then loops WP + CF setup per domain with adaptive resource allocation"
- Update Usage section: add multi-site example
- Keep audit skill descriptions (updated by phases 5-7)

## Todo List

### domain-review-agent
- [x] Add OLS SSL cert path cross-validation
- [x] Add CF-Connecting-IP cross-validation
- [x] Add LSPHP + WordPress compatibility check
- [x] Add Redis DB assignment validation (no duplicates across sites)
- [x] Add LSAPI_CHILDREN RAM budget validation
- [x] Add backup coverage cross-check
- [x] Update cross-reference table in report template

### Documentation
- [x] Update system-architecture.md (OLS stack, multi-site diagram)
- [x] Update code-standards.md (skill structure for v2)
- [x] Update project-overview-pdr.md (F5-F8 requirements, known limitations)
- [x] Update project-changelog.md (v2.0 entry)
- [x] Update development-roadmap.md (Phase 1.2)
- [x] Update CLAUDE.md (stack description)

## Success Criteria

- domain-review-agent correctly cross-validates OLS-specific settings
- Redis DB duplicates flagged across sites
- LSAPI_CHILDREN overcommit detected and flagged
- All 6 documentation files updated with v2.0 content
- CLAUDE.md reflects OLS stack accurately
- Changelog entry comprehensive and accurate

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Cross-validation logic too complex | LOW | Keep checks simple: compare values, flag mismatches |
| Documentation gets stale | LOW | Update docs in same PR as skill changes |
| Missing a doc file | LOW | Checklist in todo list covers all 6+1 files |

## Security Considerations

- No new security concerns for review agent (read-only)
- Documentation should NOT contain example credentials or real server IPs
- Changelog should reference features, not implementation details with secrets

## Next Steps

This is the final phase. After completion:
- Tag git as v2.0
- Archive v1 Nginx skills on separate branch (`v1-nginx-archive`)
- Run full audit on a v2-setup domain to validate end-to-end
