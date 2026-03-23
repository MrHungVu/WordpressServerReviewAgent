# Phase 5: Registration & Documentation Updates

## Context
- Parent plan: [plan.md](./plan.md)
- Depends on: Phases 1-4 (all skills created)

## Overview
- **Priority:** P2
- **Status:** pending
- **Description:** Update CLAUDE.md, project documentation, and verify skills are discoverable/invocable.

## Requirements

### Functional
- Update project `CLAUDE.md` to list setup skills alongside audit skills
- Update `docs/system-architecture.md` with setup skill architecture diagram
- Update `docs/project-overview-pdr.md` with new functional requirements
- Update `docs/code-standards.md` with setup-specific standards (report naming, credential output)
- Update `docs/project-changelog.md` with v1.1 entry
- Update `docs/development-roadmap.md` with setup skills milestone
- Verify skills are invocable by directory presence

## Implementation Steps

1. Update `CLAUDE.md`:
   - Add setup skills section alongside audit skills
   - Add trigger phrase: "setup domain example.com"
   - Document credential output behavior
2. Update `docs/system-architecture.md`:
   - Add setup skill architecture diagram (mirror of audit diagram)
   - Document sequential execution flow
   - Add credential generation data flow
3. Update `docs/project-overview-pdr.md`:
   - Add F5-F8 functional requirements for setup skills
   - Update success criteria
4. Update `docs/code-standards.md`:
   - Add setup report naming convention: `domain-setup-{domain}-{YYYYMMDD}.md`
   - Add credential output standards
5. Update `docs/project-changelog.md`:
   - Add v1.1 entry with all setup skills
6. Verify skill invocation:
   - Confirm `.claude/skills/vps-server-setup/SKILL.md` exists
   - Confirm `.claude/skills/wordpress-site-setup/SKILL.md` exists
   - Confirm `.claude/skills/cloudflare-domain-setup/SKILL.md` exists
   - Confirm `.claude/skills/domain-setup-agent/SKILL.md` exists

## Todo List
- [ ] Update `CLAUDE.md` with setup skills
- [ ] Update `docs/system-architecture.md` with setup architecture
- [ ] Update `docs/project-overview-pdr.md` with setup requirements
- [ ] Update `docs/code-standards.md` with setup standards
- [ ] Update `docs/project-changelog.md` with v1.1
- [ ] Update `docs/development-roadmap.md`
- [ ] Verify all 4 skill directories exist with SKILL.md files

## Success Criteria
- All documentation reflects both audit and setup skill sets
- Skills invocable via "setup domain example.com" trigger
- Architecture diagram shows both audit and setup flows
- Changelog records v1.1 release

## Next Steps
→ Test: Run "setup domain" on a test VPS to verify end-to-end
