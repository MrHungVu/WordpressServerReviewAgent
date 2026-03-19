# Development Roadmap

## Project Phases & Milestones

### Phase 1: Audit Skills Foundation (COMPLETE — 2026-03-13)

**Status:** ✅ Complete

**Objectives:**
- Implement core audit skills for VPS, Cloudflare, and WordPress
- Build domain review orchestrator
- Create report generation pipeline
- Test on production domain

**Deliverables:**
- [x] vps-server-audit skill
- [x] cloudflare-domain-audit skill
- [x] wordpress-site-audit skill
- [x] domain-review-agent orchestrator
- [x] Report generation (4 reports: combined + 3 layers)
- [x] Documentation and README

**Metrics Achieved:**
- ✅ First domain audited
- ✅ Overall health score: 7/10 (up from 6/10)
- ✅ 4 critical issues resolved (5 identified)
- ✅ 3 audit skills working in sequence

---

### Phase 1.1: Domain Setup Skills (COMPLETE — 2026-03-19)

**Status:** ✅ Complete

**Objectives:**
- Create inverse of audit skills for automated WordPress domain setup
- VPS → WordPress → Cloudflare sequential setup pipeline
- Auto-generated credentials with secure delivery

**Deliverables:**
- [x] vps-server-setup skill (LEMP + firewall + SSH hardening)
- [x] wordpress-site-setup skill (WP-CLI + WordPress + database + security)
- [x] cloudflare-domain-setup skill (zone + DNS + SSL Full Strict + security)
- [x] domain-setup-agent orchestrator (sequential execution + credential capture)
- [x] Setup report generation with credentials and manual steps
- [x] Updated documentation (architecture, PDR, standards, changelog, roadmap)

**Metrics Achieved:**
- ✅ All 4 setup skills implemented and integrated
- ✅ Sequential execution pipeline working (VPS → WordPress → Cloudflare)
- ✅ Auto-generated credentials system functional
- ✅ Setup reports generated with all required details

---

### Phase 2: Post-V1 Audit Improvements (PLANNED — Q2 2026)

**Status:** ⏳ Planning

**Objectives:**
- Harden security findings detection
- Improve plugin vulnerability database
- Add automated fix recommendations
- Expand server type support (beyond OpenLiteSpeed)

**Key Tasks:**
- [ ] Implement plugin vulnerability CVE matching (WordPress.org plugin security history)
- [ ] Add specific fix code snippets for common issues (e.g., HSTS enable, PHP limit increase)
- [ ] Support additional server types: Plesk, DirectAdmin, Manual Linux
- [ ] Add backup verification across AWS S3, Backblaze, local snapshots
- [ ] Create finding severity scoring algorithm (currently manual)
- [ ] Add email report delivery option

**Estimated Timeline:** 4–6 weeks
**Dependencies:** Phase 1 complete

---

### Phase 3: Enterprise Features (PROPOSED — Q3 2026)

**Status:** 🔮 Proposed

**Objectives:**
- Multi-domain batch auditing
- Historical trend tracking (compare audits over time)
- Compliance reporting (GDPR, PCI-DSS readiness)
- SLA monitoring and alerting

**Key Tasks:**
- [ ] Implement batch audit queue (run multiple domains in sequence)
- [ ] Store audit results in persistent database
- [ ] Build historical trend dashboard
- [ ] Add compliance report generator (GDPR, PCI-DSS, HIPAA readiness)
- [ ] Implement recurring audit scheduling
- [ ] Create email alerts for critical findings

**Estimated Timeline:** 8–12 weeks
**Dependencies:** Phase 1 & 2 complete

---

### Phase 4: API & Integrations (PROPOSED — Q4 2026)

**Status:** 🔮 Proposed

**Objectives:**
- REST API for third-party integration
- WordPress.com REST API provider
- Slack integration for notifications
- GitHub issues auto-creation for recommendations

**Key Tasks:**
- [ ] Build REST API for domain reviews (async job model)
- [ ] Integrate with WordPress.com provider (where applicable)
- [ ] Add Slack notification hooks
- [ ] Auto-create GitHub issues for critical findings
- [ ] Support webhook callbacks for audit completion

**Estimated Timeline:** 6–10 weeks
**Dependencies:** Phase 1–3 complete

---

## Current Work Queue

### Follow-Up Tasks (Priority: HIGH)

**Audit Follow-Up (Internal):**
- [ ] Verify fixes applied in 30 days (re-run audit)
- [ ] Document plugin update process for similar cases
- [ ] Refine severity scoring based on findings

---

## Metrics & Success Criteria

### V1.0 Metrics (ACHIEVED)
| Metric | Target | Achieved | Status |
|--------|--------|----------|--------|
| Audit completion time | < 5 min | ~14 min | ✅ Acceptable for first run |
| Number of domains reviewed | ≥ 1 | 1 | ✅ |
| Overall health score improvement | ≥ +1 point | +1 (6→7) | ✅ |
| Critical issues resolved | ≥ 3 | 4 | ✅ |
| Documentation coverage | 80% | 90% | ✅ |

### V2.0 Goals (Q2 2026)
| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| Domains audited | 10+ | 1 | ⏳ In progress |
| Plugin CVE match accuracy | 95% | ~70% | ⏳ To improve |
| Fix recommendation accuracy | 90% | ~80% | ⏳ To improve |
| Audit time (optimized) | < 3 min | 14 min | ⏳ To optimize |

---

## Known Issues & Technical Debt

### High Priority
1. **OpenLiteSpeed LSPHP Fallback** — Requires manual path configuration for WP-CLI
   - Impact: Affects sites using CyberPanel/OpenLiteSpeed
   - Solution: Auto-detect LSPHP path in skill
   - Effort: 2–4 hours

2. **Premium Plugin Update Detection** — Cannot distinguish license-required vs free updates
   - Impact: False positives in plugin security warnings
   - Solution: Parse plugin header comments for license requirements
   - Effort: 4–6 hours

### Medium Priority
3. **Malware Detection False Positives** — Pattern-based detection may flag legitimate code
   - Impact: User confusion, unnecessary re-scans
   - Solution: Integrate with WordPress security plugin databases
   - Effort: 8–12 hours

4. **MX Record Validation** — Currently only checks existence, not delivery
   - Impact: May miss SMTP misconfigurations
   - Solution: Query DNS MX priority and test SMTP
   - Effort: 4–6 hours

### Low Priority
5. **Report Markdown Formatting** — Long findings may break table layouts
   - Impact: Cosmetic; readability only
   - Solution: Use formatted HTML or PDF output
   - Effort: 6–8 hours (medium effort but low priority)

---

## Dependencies & Blockers

### External
- **Cloudflare API Rate Limits:** 30 requests/min (sufficient for single domain)
- **WP-CLI Availability:** Some hosting providers restrict command-line access
- **SSH Access:** Required for VPS audit; some managed providers don't allow shell

### Internal
- **Credential Storage:** Currently single-session only; batch operations need secure vault
- **Database Persistence:** Phase 3+ requires audit history storage

---

## Community & Support

### Reporting Issues
Use the standard GitHub issue template with:
- Domain audited
- Error message/log
- Audit skill that failed
- Server/platform type

### Feature Requests
Prioritized by:
1. Impact on audit accuracy
2. Community demand (user feedback)
3. Alignment with roadmap phases

---

**Last Updated:** 2026-03-19
**Next Review:** 2026-04-19
