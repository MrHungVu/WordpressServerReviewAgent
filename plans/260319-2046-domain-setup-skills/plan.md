---
title: "Domain Setup Skills (Inverse of Audit Skills)"
description: "Create 4 Claude skills to automate WordPress domain setup: VPS server setup, WordPress install, Cloudflare config, and orchestrator"
status: complete
priority: P1
effort: 3h
branch: main
tags: [skills, wordpress, setup, automation, nginx, cloudflare]
created: 2026-03-19
---

# Domain Setup Skills — Implementation Plan

## Goal
Create 4 new Claude Code skills that automate WordPress domain setup — the inverse of the existing audit skills. User provides a fresh VPS + credentials → skill installs everything → delivers working WordPress site with Cloudflare SSL.

## Architecture
Mirror the existing 3+1 audit pattern:

```
domain-setup-agent (orchestrator)
  ├── vps-server-setup        → Install Nginx + PHP-FPM + MariaDB + firewall
  ├── wordpress-site-setup    → Install WP-CLI + WordPress + DB + security hardening
  └── cloudflare-domain-setup → Zone + DNS + Origin Cert + SSL Full Strict + settings
```

## Phases

| Phase | Skill | Status | Effort |
|-------|-------|--------|--------|
| [Phase 1](./phase-01-vps-server-setup.md) | `vps-server-setup/SKILL.md` | complete | 45min |
| [Phase 2](./phase-02-wordpress-site-setup.md) | `wordpress-site-setup/SKILL.md` | complete | 45min |
| [Phase 3](./phase-03-cloudflare-domain-setup.md) | `cloudflare-domain-setup/SKILL.md` | complete | 30min |
| [Phase 4](./phase-04-domain-setup-agent.md) | `domain-setup-agent/SKILL.md` (orchestrator) | complete | 30min |
| [Phase 5](./phase-05-registration-and-docs.md) | Update CLAUDE.md, docs, project files | complete | 15min |

## Execution Order
Sequential: Phase 1 → 2 → 3 → 4 → 5 (each skill builds on prior patterns)

## Key Decisions
- **Stack:** Nginx + PHP-FPM (not OpenLiteSpeed)
- **DB:** Detect existing MariaDB, install only if missing
- **SSL:** Cloudflare Origin Certificate (15-year, via API)
- **CF Config:** Full API automation (except nameserver change = manual)
- **WP Hardening:** Security baseline (DISALLOW_FILE_EDIT, XML-RPC block, file perms, etc.)
- **Idempotent:** Every step checks before acting (safe to re-run)

## References
- Brainstorm report: `plans/reports/brainstorm-260319-2046-domain-setup-agent.md`
- Existing audit skill pattern: `.claude/skills/vps-server-audit/SKILL.md`
- Project standards: `docs/code-standards.md`
