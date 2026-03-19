# Code Standards & Project Structure

## Project Structure

```
ReviewServerConfigAgent/
├── .claude/
│   └── skills/
│       ├── vps-server-audit/
│       │   ├── instructions.md      # Skill system prompt & capabilities
│       │   ├── tests.md              # Test scenarios & expected outputs
│       │   └── examples.md           # Usage examples
│       ├── cloudflare-domain-audit/
│       ├── wordpress-site-audit/
│       ├── domain-review-agent/
│       ├── vps-server-setup/
│       ├── wordpress-site-setup/
│       ├── cloudflare-domain-setup/
│       └── domain-setup-agent/
│
├── docs/
│   ├── README.md                     # Quick start & overview
│   ├── project-overview-pdr.md       # Project purpose & requirements
│   ├── system-architecture.md        # Technical design & data flows
│   ├── code-standards.md             # This file
│   ├── project-changelog.md          # Release notes & history
│   └── development-roadmap.md        # Phases & milestones
│
├── vps-reports/                      # Audit & setup output directory
│   ├── domain-review-*.md            # Audit reports
│   ├── vps-audit-*.md
│   ├── cloudflare-audit-*.md
│   ├── wordpress-audit-*.md
│   └── domain-setup-*.md             # Setup reports
│
├── CLAUDE.md                         # Project instructions & usage
├── .gitignore                        # Never commit reports or credentials
└── README.md                         # GitHub repository README

```

**Key Directory Rules:**
- `.claude/skills/` — Each skill is self-contained (instructions + tests)
- `docs/` — All documentation files live here
- `vps-reports/` — Generated audit reports (temporary, not version-controlled)
- No source code repository (skills are Claude-native, not code files)

---

## Skill Implementation Standards

### Skill Structure

Each skill has a dedicated directory with:

```
.claude/skills/{skill-name}/
├── instructions.md                   # System prompt & core definitions
├── tests.md                          # Test cases & validation
└── examples.md                       # Usage examples & sample outputs
```

### Skill Instructions Template

**Required Sections in `instructions.md`:**

1. **Purpose** — One-sentence skill objective
2. **Inputs** — What user provides (domain, credentials, parameters)
3. **Outputs** — What skill returns (report format, data structure)
4. **Detailed Implementation** — Step-by-step audit logic
5. **Error Handling** — How to handle connectivity/permission failures
6. **Assumptions** — Server type, OS, software versions
7. **Security** — Credential handling, data sensitivity
8. **Fallbacks** — Alternative approaches for different server types

### Skill Testing Standards

**Test Cases in `tests.md`:**

- [ ] Happy path: Valid domain, all systems responding
- [ ] Missing credentials: No SSH key, invalid Cloudflare token
- [ ] Server unreachable: SSH timeout, API 5xx error
- [ ] Partial data: WP-CLI fails but VPS/Cloudflare succeed
- [ ] Edge cases: Domain not on Cloudflare, WordPress not installed
- [ ] Performance: Measure audit duration, verify < 5 min

**Test Validation:**
- Actual test runs against real domains (post-development)
- Example outputs in `examples.md` with before/after scenarios
- Error message clarity (users understand what went wrong)

---

## Audit Report Standards

### Report Format

All reports use **Markdown with consistent structure:**

```markdown
# Report Title

**Domain:** example.com
**Date:** YYYY-MM-DD HH:MM TZ
**Status:** Initial / Post-Fix

## Executive Summary
- 2-3 sentence overview
- Overall health score
- Key improvements/concerns

## Health Dashboard
| Category | Status | Issues | Score |
|----------|--------|--------|-------|
| Layer    | HEALTHY / ATTENTION / CRITICAL | N | X/10 |

## Critical Issues
1. [Issue Title]
   - Description
   - Impact
   - Recommendation

## Warnings
- [Secondary concerns with lower priority]

## Recommendations
- [Enhancements and best practices]

## Appendix
- Raw data, command outputs, detailed logs
```

### Naming Convention

#### Audit Reports
```
{type}-audit-{domain}-{YYYYMMDD}.md
├─ domain-review-
├─ vps-audit-
├─ cloudflare-audit-
└─ wordpress-audit-
```

#### Setup Reports
```
domain-setup-{domain}-{YYYYMMDD}.md
```

**Examples:**
- Audit: `domain-review-example.com-20260313.md`
- Audit: `vps-audit-example.com-20260313.md`
- Audit: `cloudflare-audit-example.com-20260313.md`
- Audit: `wordpress-audit-example.com-20260313.md`
- Setup: `domain-setup-example.com-20260319.md`

### Report Content Standards

**Writing Style:**
- Technical but accessible (assume reader is WordPress admin, not sysadmin)
- Use tables for structured data
- Code blocks for commands and configurations
- Lists for multi-item findings
- Action-oriented language ("Fix X by doing Y")

**Data Sensitivity:**
- Flag reports as containing server IPs
- Remind users not to commit to version control
- Remove credentials from all outputs

**Cross-References:**
- Link between related findings across layers
- Example: "See Cloudflare audit for SSL mode details"
- Index findings by severity for easy navigation

---

## Finding Classification

### Severity Levels

| Severity | Definition | Examples | Action Timeline |
|----------|-----------|----------|-----------------|
| **Critical** | Immediate threat to security or functionality | Malware, missing HTTPS, SQL injection | Fix within 24 hours |
| **Warning** | Issue reducing security/performance | Outdated plugins, weak PHP config | Fix within 7 days |
| **Recommendation** | Best practice improvement | SEO plugin, auto-updates, hardening | Plan next month |

### Finding Template

Each finding should include:

```
[Icon] **[Title]** [Severity Badge]
- **What:** Description of the issue
- **Why:** Why it matters
- **Impact:** What could happen if not fixed
- **Fix:** Specific steps to resolve (code examples if applicable)
- **Verification:** How to confirm it's fixed
```

**Example:**

```markdown
🔴 **HTTP Site URL on HTTPS Domain** [CRITICAL]
- **What:** WordPress siteurl and home are http:// while Cloudflare enforces HTTPS
- **Why:** Users may see SSL warnings, mixed content errors, or insecure connections
- **Impact:** Loss of customer trust, potential cart abandonment for ecommerce
- **Fix:**
  ```bash
  wp option update siteurl "https://example.com"
  wp option update home "https://example.com"
  ```
- **Verification:** Visit https://example.com, check no SSL warnings, verify admin login works
```

---

## Cross-Layer Validation Standards

### Validation Grid

When auditing multiple layers, create a validation matrix:

| Check | VPS Layer | Cloudflare Layer | WordPress Layer | Status |
|-------|-----------|------------------|-----------------|--------|
| DNS A Record | xxx.xxx.xxx.xxx | xxx.xxx.xxx.xxx | - | ✅ Match |
| SSL Mode | Full cert (2040) | Full Strict | Enforced in WP | ✅ Consistent |
| HTTPS Enforcement | Apache rewrite | Always Use HTTPS | siteurl = https:// | ✅ Triple-checked |
| Cache Rules | None | `/cart/*` bypass | WooCommerce cart | ✅ Aligned |

**Rules for Cross-Validation:**
1. DNS A record must match server IP
2. SSL mode (Cloudflare) must match origin certificate
3. Cache bypass rules must align with dynamic content paths
4. HTTPS enforcement must be consistent across all 3 layers
5. Email MX records must point to actual mail provider
6. Firewall rules must allow legitimate admin traffic

---

## Health Score Calculation

### Formula

```
Score = max(0, 10 - (critical_deductions × 2 + warning_deductions × 1))

Critical Deduction: -2 points each
- Malware detected
- SSL misconfig / missing HTTPS
- Unprotected brute-force vectors
- SQL injection / RCE vulnerability
- Expired SSL certificate

Warning Deduction: -1 point each
- Outdated plugins (3+ versions behind)
- Missing security headers
- Weak PHP configuration
- Pending system updates
- Missing backups
- Stale sessions

Example:
  Base: 10
  - 2 critical issues: 10 - (2×2) = 6
  - 5 warnings: 6 - (5×1) = 1
  Final Score: 1/10 (CRITICAL)

Example:
  Base: 10
  - 0 critical issues: 10
  - 5 warnings: 10 - (5×1) = 5
  Final Score: 5/10 (ATTENTION)
```

### Score Bands

| Score | Status | Action |
|-------|--------|--------|
| 8–10 | HEALTHY | Monitor; apply recommendations when possible |
| 6–7 | ATTENTION | Fix warnings within 7–30 days |
| 3–5 | CONCERNS | Fix warnings immediately; escalate critical issues |
| 0–2 | CRITICAL | Immediate remediation required; consider taking site offline |

---

## Credential Handling Standards

### Never Do (Audit & Setup)
- ❌ Save passwords/tokens to `credentials.md` or `.env.example`
- ❌ Store credentials in report markdown files permanently
- ❌ Write SSH keys to disk (use paramiko in-memory)
- ❌ Include API tokens in example outputs
- ❌ Commit reports containing credentials to version control

### Always Do (Audit)
- ✅ Ask user for credentials at runtime
- ✅ Pass credentials inline to skill execution
- ✅ Clear credentials from memory after use
- ✅ Log only sanitized credential type: "SSH key auth used"
- ✅ Remove credentials from audit reports before saving

### Always Do (Setup)
- ✅ Generate passwords using `openssl rand -base64 32`
- ✅ Pass auto-generated passwords to orchestrator only (in-memory)
- ✅ Include auto-generated credentials ONLY in final setup report (single-session display)
- ✅ Never persist setup credentials to files after report generation
- ✅ Clear all credentials from memory after report delivery

### Credential Types

| Type | Input Method | Storage | Notes |
|------|--------------|---------|-------|
| SSH Password (audit) | Prompted at runtime | Memory only, cleared after session | Requires paramiko or sshpass |
| SSH Key (audit) | Passed as string | Memory only, cleared after session | Preferred; more secure |
| Cloudflare API Token (audit) | Prompted at runtime | Memory only, cleared after session | Never API key (legacy) |
| MySQL Password (setup) | Auto-generated | Setup report only, cleared after | Use openssl rand -base64 32 |
| WordPress Admin (setup) | Auto-generated | Setup report only, cleared after | Generate secure username + password |
| Cloudflare Origin Cert (setup) | Auto-generated | Deploy to server, cleared from memory | 15-year validity, Cloudflare-signed |

---

## Testing & Validation

### Pre-Audit Checklist
- [ ] User provided valid domain name (DNS resolvable)
- [ ] User provided SSH credentials (password or key)
- [ ] User provided Cloudflare API token
- [ ] Verify Cloudflare token is API token (not API key)
- [ ] Test SSH connectivity before main audit

### Post-Audit Checklist
- [ ] All 3 audits completed successfully (or partial with reason)
- [ ] Reports generated and saved to `vps-reports/`
- [ ] Cross-validation completed and findings consolidated
- [ ] Health scores calculated for all layers
- [ ] Findings prioritized by severity
- [ ] No credentials left in reports or memory

### Example Validation Test
```
Domain: test.example.com (real site with test server)
Expected Results:
- VPS Audit: 3–5 min, returns OS + PHP + MySQL details
- Cloudflare Audit: 30–60 sec, validates DNS + SSL
- WordPress Audit: 2–3 min, lists plugins, themes, users
- Domain Review: Cross-validates all layers, assigns score 6–8/10
- Reports: 4 files created in vps-reports/
```

---

## Documentation Standards for Skills

### Skill Instructions Must Include

1. **Capability Statement** — What this skill does
2. **Input Parameters** — Domain, credentials, optional flags
3. **Output Specification** — Report format, JSON structure if applicable
4. **Step-by-Step Implementation** — Exact commands to run
5. **Credential Handling** — How credentials are passed and cleared
6. **Error Scenarios** — Common failures and recovery
7. **Server Type Variations** — How to handle Apache, Nginx, OpenLiteSpeed, etc.
8. **Known Limitations** — What this skill cannot audit
9. **Examples** — Sample input/output for successful audit

### Examples in Each Skill

Provide 2–3 realistic examples showing:
- Healthy system audit output
- System with warnings
- System with critical issues
- Error scenario (SSH timeout, API rate limit)

---

## Future Extensibility

### Adding New Audit Skills

**Pattern:**
1. Create `skill-name/` directory under `.claude/skills/`
2. Write `instructions.md` with implementation details
3. Write `tests.md` with test cases
4. Write `examples.md` with sample outputs
5. Integrate into `domain-review-agent` orchestrator
6. Update `docs/system-architecture.md` with new subsystem
7. Update roadmap with phase/milestone

**Example:** Planned future skill for "SSL Transparency Audit" (cert history, issuance logs)

---

## Version Management

**Versioning Scheme:** `MAJOR.MINOR.PATCH`

- **MAJOR:** Breaking changes to skill interface or report format
- **MINOR:** New features, non-breaking enhancements
- **PATCH:** Bug fixes, documentation updates

**Release Process:**
1. Update version in CLAUDE.md
2. Document changes in project-changelog.md
3. Tag commit with version
4. Update roadmap with next phase

---

**Last Updated:** 2026-03-19
**Version:** 1.1
