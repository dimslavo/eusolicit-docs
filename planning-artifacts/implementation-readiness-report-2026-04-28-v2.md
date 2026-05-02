---
stepsCompleted: ["step-01-document-discovery", "step-02-prd-analysis", "step-03-epic-coverage-validation", "step-04-ux-alignment", "step-05-epic-quality-review", "step-06-final-assessment"]
inputDocuments:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/ux-spec.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E17-crm-integrations.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E18-trust-center-compliance.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/sprint-status.yaml"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/17-0-crm-connection-model-fernet-token-vault-sync-engine-scaffold-conflict-resolution-framework.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-04-28.md"
focus: "Readiness for Epic 18 (Trust Center & Compliance Posture) kickoff + verification of Epic 17 in-flight remaining stories + reconciliation of prior IR open items"
date: 2026-04-28
assessor: "PM (autopilot)"
---

# Implementation Readiness Assessment Report — v2

**Date:** 2026-04-28
**Project:** eusolicit
**Mode:** BMAD autopilot (no operator prompts)
**Assessor role:** Product Manager — requirements traceability + planning-hygiene critique
**Focus:** (1) Validate readiness for the **next epic to kick off (Epic 18 — Trust Center)**, per operator workflow guidance "Before starting any epic, run [IR] Implementation Readiness". (2) Verify status of prior IR open items (CR-1..CR-6 from 2026-04-28-v1). (3) Spot-check readiness of the still-pending Epic 17 stories (17-1, 17-2, 17-3) since the orchestrator may dispatch them in parallel.

---

## Executive Verdict

**Overall Readiness for Epic 18 kickoff: NEEDS WORK — proceed with named caveats**

| Dimension | Verdict | Notes |
|---|---|---|
| PRD coverage of Epic 18 (FR10) | ⚠ Split-source | FR10.x lives only in `prd-amendment-2026-04-25.md`. Base `PRD.md` contains zero references to FR10/Trust Center/sub-processor. Same gap as flagged for E17 in prior IR (CR-4 unresolved). |
| Architecture coverage of Epic 18 | ✅ Strong | ADR-011 (`Trust Center as static-rendered Next.js, not a CMS`), public-route ingress rule, S3 artefact pipeline, `infra/trust/artefacts/` repo layout, sub-processor change → notification flow — all documented. |
| UX coverage of Epic 18 | 🛑 **Critical gap** | `ux-spec.md` has **zero** references to `/trust`, MDX rendering, sub-processor list, public-page variant of `<AppShell>`, change-log section, or downloadable-artefact UI. The entire customer-facing surface is unspecified. |
| Epic 18 spec quality | ✅ Adequate | 3 stories sized (5/5/3 = 13 pts matches header), dependencies (E03/E07/E09) clear, ACs testable, ISO 27001 evidence rationale explicit. |
| Epic 17 in-flight readiness | ⚠ Partial | 17-0 in `review`. 17-1/17-2/17-3 still `backlog`; story files not yet created. CR-5 (UX gap for E17 surfaces) from prior IR remains unaddressed. |
| Planning-artifact hygiene (CR-1) | 🛑 **Unresolved** | No `_archive/` directory created; 80+ epic files in `planning-artifacts/epics/` across 6+ naming conventions persist. |
| Planning-view drift (CR-2) | 🛑 **Unresolved** | `epics.md` (10-epic view) still erases visibility of E11–E21. |
| Epic 16 status integrity (CR-3) | ✅ **Resolved** | `sprint-status.yaml` now shows `epic-16: done`. |
| PRD source unification (CR-4) | 🛑 **Unresolved** | Base PRD.md still has no FR8/FR9/FR10/FR11. |
| UX gap for E17 surfaces (CR-5) | ⚠ **Unresolved** | No CRM connection card / conflict log / stage-mapping UX added to `ux-spec.md`. |
| Epic 13 carry-forwards (CR-6) | ⚠ **Unresolved** | 6 stories still at `ready-for-dev` (drift-recovery, dw-01..03, inj-01..03). |

**Net change since 2026-04-28-v1:** 1 of 6 prior CRs resolved (CR-3). 5 remain. 1 new critical gap surfaces for Epic 18 (UX completely missing). 1 net-new architecture-vs-epic alignment concern (sub-processor change notification → existing `notification` service extension is well-scoped but introduces a CI→Redis Streams hop that the architecture diagram on line 70 of `architecture.md` does not depict).

---

## Step 1 — Document Discovery

### Canonical documents (operative source of truth)

Same inventory as prior IR (2026-04-28-v1). No additions; no deletions; no archive moves performed.

**New since prior IR:**
- `implementation-artifacts/17-0-crm-connection-model-fernet-token-vault-sync-engine-scaffold-conflict-resolution-framework.md` (Story 17-0 file, status `review`).
- `implementation-readiness-report-2026-04-28.md` (prior IR — this v2 report supersedes the planning-readiness sections; the 17-0-specific findings still stand).

### Hygiene status (verified 2026-04-28)

```
$ ls planning-artifacts/epics/_archive/  → does not exist
$ ls planning-artifacts/_archive/        → does not exist
```

CR-1 and CR-2 actions from the prior IR have not been executed. Risk profile is unchanged.

---

## Step 2 — PRD Analysis (Epic 18 focus)

### FR10 (Trust Center & Compliance Posture) — present **only in amendment**

`prd-amendment-2026-04-25.md` lines 156–177 define the full FR10 surface:
- FR10.1 — Public `/trust` page lists GDPR (compliant), ISO 27001 (in progress with target date), SOC 2 (planned/N-A), data residency (EU only), encryption (AES-256 at rest, TLS 1.3 in transit).
- FR10.2 — Downloadable PDF artefacts: GDPR DPA, Security Overview, Sub-Processor List, latest pen-test summary (redacted), BCP summary, Data Residency confirmation.
- FR10.3 — Sub-Processor List auto-updates from `infra/sub-processors.yaml`.
- FR10.4 — Versioned artefacts with change-log entries; sub-processor change → email to active customer DPAs (GDPR Art. 28).

`PRD.md` (base) — zero matches for any of: `FR10`, `Trust Center`, `sub.processor`, `/trust`, `WeasyPrint`, `MDX`. **Same fragmentation footgun as for E17 (FR11) called out in prior IR; no progress.**

### NFR coverage for Epic 18

| NFR | Relevance | Coverage |
|---|---|---|
| NFR-5/6 (auth, encryption) | `/trust/*` is the only **unauthenticated** route in the platform — needs explicit ingress allow-list and confirmation no auth-required middleware leaks | ✅ Architecture line 569 (Routing & Path Architecture table) and AC bullet 1 of E18 both call this out |
| NFR-7 (data isolation) | E18 must not leak any tenant data via the public route | ✅ E18 AC bullet 9 ("cross-tenant negative test: unauthenticated request returns full content with no user data") |
| NFR-9 (third-party deps — Dependabot) | WeasyPrint + new MDX pipeline deps (e.g. `@next/mdx`, `gray-matter`) need scanning | ⚠ Carry-forward `inj-01-dependabot-configuration` still ready-for-dev (CR-6). E18 ships unscanned MDX/PDF deps unless inj-01 lands first |
| NFR-12 (auditability — ISO 27001) | E18 IS the ISO 27001 evidence vehicle | ✅ Epic goal narrative + ADR-011 explicitly tie the static-rendered architecture to ISO change-management evidence |
| NFR-22 (notification reliability — Art. 28 emails) | Sub-processor change → email to all paid DPAs is GDPR-mandated | ✅ E18 S18.02 reuses Epic 9 `notification` service + canonical fire-and-forget audit write (project-context Epic 13 Rule 45) |

### PRD completeness assessment for Epic 18

✅ **Functional scope is fully specified when amendment is read alongside base PRD.**
🛑 **Source-of-truth fragmentation persists.** Any agent that loads `PRD.md` and not the amendment will be unable to ground Epic 18 ACs to a PRD source. This is the same blocker as Epic 17 — CR-4 must land before either E17.x or E18.x story creation routinely cites PRD sources from a single file.

---

## Step 3 — Epic Coverage Validation

### Sprint state delta since 2026-04-28-v1

| Epic | Status (was) | Status (now) | Delta |
|---|---|---|---|
| 13 | done (with 6 carry-forwards) | unchanged | 6 carry-forwards still `ready-for-dev` |
| 16 | in-progress (suspect) | **done** | ✅ CR-3 resolved |
| 17 | backlog (next) | **in-progress** | 17-0 created + dev pass complete (status: `review`); 17-1/17-2/17-3 still `backlog` |
| 18 | backlog | unchanged | **Next epic to kick off per operator workflow guidance** |
| 19/20/21 | backlog | unchanged | — |

### Epic 18 — FR coverage map

| AC reference | PRD-amendment / architecture source | Coverage |
|---|---|---|
| Public `/trust` route accessible without auth | Amendment FR10.1; architecture §11 line 569; ADR-011 line 757 | ✅ |
| Compliance posture display (GDPR, ISO 27001, SOC 2 N-A, residency, encryption) | Amendment FR10.1 | ✅ |
| Downloadable PDF artefacts (DPA, SecOverview, Sub-Proc List, Pen-Test, BCP, Residency) | Amendment FR10.2 | ✅ |
| Sub-Processor List auto-generated from `infra/sub-processors.yaml` | Amendment FR10.3; architecture §16 line 925 | ✅ |
| Versioned artefacts with change-log | Amendment FR10.4 | ✅ |
| Sub-processor change → DPA email (GDPR Art. 28) | Amendment FR10.4; architecture line 160 (`notification` service rw); ADR-011 line 760 | ✅ |
| ISO 27001 roadmap (M3 Stage 1 / M9–10 Stage 2 / M12 cert) | Amendment §156 (Change 4) + amendment §165 PRD §3 update | ✅ |
| M2 deadline tracked in `eusolicit-docs/` | Amendment §11.4 (locked decision) | ⚠ No tracking artefact specified — see CR-7 below |
| Cross-tenant negative test (unauthenticated → no user data leak) | E18 AC bullet 9 | ✅ |

### Epic 17 — readiness of remaining stories (17-1/17-2/17-3)

Story files **do not yet exist** in `implementation-artifacts/`. Per operator guidance "Before each story, ALWAYS run [VS] Validate Story", each must pass `bmad-create-story` then [VS] before dev. Spec quality is inherited from `E17-crm-integrations.md` (covered favourably in prior IR §5).

**Pre-flight items before each story file is created:**
- 17-1 (HubSpot): needs HubSpot sandbox account + OAuth app credentials in vault (procurement lead time risk). Carry-forward from prior IR R-002.
- 17-2 (Pipedrive): same procurement note for Pipedrive sandbox; lower complexity per locked decision §11.6.
- 17-3 (Salesforce): **highest complexity**; Salesforce sandbox + dev-org procurement lead time is a real schedule risk (prior IR R-002/R-004). Confirm sandbox is provisioned **before** kicking off `bmad-create-story` for 17-3.

---

## Step 4 — UX Alignment

### Critical finding: Epic 18 has **zero UX spec coverage**

`ux-spec.md` (2026-04-28) was searched for: `trust`, `sub.processor`, `MDX`, `/trust`, `public route`, `unauthenticated`. **Zero matches.**

The customer-facing surface for Epic 18 is entirely unspecified at the UX-spec level:

1. **Public `/trust` page layout** — There is no public variant of `<AppShell>` specified. AC bullet 4 of S18.00 says "Layout reuses existing `<AppShell>` patterns minus authenticated topbar (public variant)" — but the public variant does not exist in `ux-spec.md`.
2. **Compliance posture cards** (GDPR / ISO 27001 / SOC 2 / residency / encryption) — visual treatment, status-badge design, "in progress" indicator UX not specified.
3. **ISO 27001 roadmap component** (M3/M9–10/M12 milestones with current-stage indicator) — pure E18 surface, no precedent in spec.
4. **Downloadable artefact list** — file-card design, signed-URL handling, error states, locale handling for legal artefacts (BG/EN versions of DPA?) not specified.
5. **Change-log section** — version-history UX, diff-display semantics, subscriber-only vs public visibility not specified.
6. **Sub-processor change email template** (BG/EN) — copy + i18n keys not drafted.

### Cumulative UX debt

| Epic | UX gap | Status |
|---|---|---|
| E17 (CRM) | 5 surfaces unspecified (CR-5) | Unresolved since 2026-04-28-v1 |
| E18 (Trust Center) | 6 surfaces unspecified (new) | New finding |

**Total UX surfaces unspecified for in-flight + next-up epics: 11.** A `bmad-create-ux-design` or `bmad-agent-ux-designer` pass should be scheduled before either E17.1 or E18.0 enters `bmad-create-story`. The frontend stories (S18.00 in particular is `Type: frontend`) cannot have meaningful ATDD source-inspection assertions if the UX surface is undefined.

---

## Step 5 — Epic Quality Review (focus: E18, with E17 in-flight checkpoint)

### E18 spec quality scorecard

| Dimension | Score | Evidence |
|---|---|---|
| Goal clarity | A | "mid-large EU consulting-firm deals stall 4–8 weeks at the procurement gate without downloadable security/compliance evidence" — concrete deal-blocker framing tied to M2 hard deadline |
| Acceptance criteria specificity | A− | 9 ACs, all testable; M2 deadline + GDPR Art. 28 30-day rule + AES-256 / TLS 1.3 explicit |
| Story decomposition | A | 3 stories (S18.00 frontend MDX/route, S18.01 PDF pipeline, S18.02 notification flow); 5/5/3 = 13 pts matches header |
| Dependency declaration | A | E03 (frontend shell), E07 (WeasyPrint), E09 (notification) — all `done` per sprint-status; no upstream blockers |
| Test design | B | Cross-tenant negative test, i18n parity, ATDD source-inspection (MDX vs runtime Markdown), 30-day-future lint test, signed-URL TTL test, audit log delivery test all called out. **No `test-design-epic-18.md` exists** — story files will need to fill the gap (mirror S15.1 / S17.0 inline pattern). |
| Source-of-truth citations | A− | Cites "PRD v1.1 §6 FR10 + §8 US10", "architecture-evaluation §2 Change-4", "§11 locked decision §11.4 (M2 deadline)" — verifiable in amendment + architecture-evaluation file |
| Net-new fence (BMM Rule) | A | Each story is single-concern; living-vs-legal artefact split is explicit; CI integration scope (S18.02) is bounded to a single GitHub Actions step |
| Anti-pattern guard rails | B+ | WeasyPrint `run_in_executor` requirement (project-context Epic 7) called out; canonical fire-and-forget audit write (Rule 45) cited; **missing**: explicit "no production-side test routes" (S15.0 B1), "no raw text() INSERTs" (E14.2 BLOCKING #3) — these are not as relevant for E18 (mostly content + CI), but should still be enumerated in story-file Anti-Pattern fences for consistency |

### E18 risks flagged for the dev pass

- **R-018-1:** Public-route ingress rule (`nginx ingress + Cloudflare WAF allow `/trust/*` without JWT`) is infrastructure work that touches Helm chart + WAF config. This is the first **unauthenticated** route in the platform; any misconfiguration leaks the auth-redirect into the public flow OR allows authenticated routes to leak. AC bullet 9 covers the negative test, but the ingress reviewer must include a path-traversal probe (`/trust/../api/v1/...`).
- **R-018-2:** S18.02's "30-day-future lint test" is a CI-side rule. If a sub-processor needs to be added with <30-day notice (legal emergency), the lint blocks the PR. ADR or playbook needed for the override path. **Recommend adding to S18.02 ACs.**
- **R-018-3:** S18.01's signed-URL-1-hour-TTL applies to artefacts served from S3. Legal-team review: does a 1-hour signed URL satisfy "publicly downloadable" per amendment FR10.2? An unauthenticated user landing on the page expects a stable link, not one that expires. Either:
  - (a) Use a stable public-read S3 URL for legal artefacts (DPA, Pen-Test) with bucket-policy gating (`Principal: "*"`, `Action: "s3:GetObject"`, `Resource: "arn:aws:s3:::eusolicit-trust-public/*"`), OR
  - (b) Generate fresh signed URLs server-side on each page load (acceptable, but adds a server hop per visit).
  - **Story should pick one before dev.**
- **R-018-4:** WeasyPrint dependency proliferation. `inj-01-dependabot` is still `ready-for-dev`; if E18 ships before Dependabot, MDX pipeline + WeasyPrint + Next.js MDX deps land unscanned. **Recommend gating S18.00 PR-merge on Dependabot config being merged.**
- **R-018-5:** Locale handling for legal artefacts. EU consulting-firm deals require BG-language DPA for Bulgarian counterparties. AC bullet 3 lists the artefacts in English; no BG-language equivalent is specified. **Either confirm legal-counsel position (English-only acceptable) or scope a BG variant for each legal artefact.**

### Epic 17 in-flight checkpoint

- **17-0** is in `review`. The story file is high-quality (11 ACs, explicit out-of-scope fence, anti-pattern enumeration carrying forward Epic 14/15/16 retro learnings). The dev pass reportedly delivered 166 ATDD tests passing and 5 Prometheus metrics. **Recommendation:** Senior code review (not yet done per sprint-status) should validate:
  - Cross-schema FK string-form (per S16 reviewer M2)
  - Claim-on-success not claim-before-dispatch (per S16 M6)
  - Outer circuit-breaker added or shared helper extended (S16 L3)
  - Startup Fernet-key validation in production (S16 L4)
- **17-1/17-2/17-3** story files do not exist. Each must pass `bmad-create-story` → [VS] Validate Story before dev. Sandbox procurement (R-002/R-004 from prior IR) remains the schedule risk for 17-3.

---

## Step 6 — Final Assessment

### Summary

The **next epic (E18 — Trust Center) is largely ready to kick off** at the spec level. PRD coverage exists (in the amendment), architecture coverage is strong (ADR-011 + S3 layout + ingress decision), and the epic spec itself is high quality with a hard M2 deadline driving urgency.

However, three categories of issues remain:

1. **Inherited debt from prior IR (CR-1, CR-2, CR-4, CR-5, CR-6) — 5 of 6 prior CRs unresolved.** Only CR-3 (Epic 16 status) was closed.
2. **Net-new critical gap: Epic 18 has zero UX spec coverage.** This blocks meaningful ATDD on the frontend story (S18.00) and risks the M2 hard deadline if a UX pass slips.
3. **Net-new design questions for Epic 18 dev pass (R-018-1..R-018-5).** None are blocking spec correctness, but each warrants story-creation-time resolution.

### Critical Issues Requiring Immediate Action (consolidating prior + new)

1. **CR-1 (planning-artifact hygiene) — 🛑 CRITICAL — unchanged from prior IR.**
   Move 80+ stale epic files to `planning-artifacts/epics/_archive/`; move PRD/UX backups to `planning-artifacts/_archive/`. 30–60 min hygiene pass.

2. **CR-2 (planning view drift) — 🛑 CRITICAL — unchanged from prior IR.**
   Reconcile `epics.md` (10-epic view) vs operational E11–E21 set. Pick Option A (rename + new full-roadmap file) or Option B (regenerate to merge).

3. **CR-4 (PRD source unification) — 🛑 HIGH — unchanged from prior IR.**
   Merge `prd-amendment-2026-04-25.md` into `PRD.md` and stamp v1.1 (or v2.0). Same blocker now applies to **both E17 and E18** — every BMAD agent loading `PRD.md` cannot see FR8/FR9/FR10/FR11.

4. **CR-5 (UX gap for E17 surfaces) — ⚠ HIGH — unchanged from prior IR.**
   5 E17 surfaces unspecified. Schedule `bmad-agent-ux-designer` pass before 17-1 enters `bmad-create-story`.

5. **CR-7 (NEW) — UX gap for E18 surfaces — 🛑 CRITICAL.**
   6 E18 surfaces unspecified (public `<AppShell>` variant, compliance posture cards, ISO 27001 roadmap component, artefact-download UI, change-log section, sub-processor email template). Schedule `bmad-agent-ux-designer` pass before S18.00 enters `bmad-create-story`. **This is potentially the M2-deadline-blocker risk — recommend resolving in next 2 working days.**

6. **CR-8 (NEW) — Resolve E18 design questions before story creation — ⚠ HIGH.**
   - R-018-3: Signed URL vs public-read S3 for legal artefacts — pick one before S18.01 enters dev.
   - R-018-5: BG-language legal artefacts — confirm legal position or scope.
   - R-018-2: Sub-processor <30-day-future override path — playbook needed.

7. **CR-6 (Epic 13 carry-forwards) — ⚠ MEDIUM (background) — unchanged from prior IR.**
   6 stories at `ready-for-dev`. `inj-01` (Dependabot) is now blocking E18 dependency-scan posture (R-018-4).

8. **CR-9 (NEW) — Epic 17 senior code review pending — ⚠ MEDIUM.**
   17-0 is at `review` with no senior code review verdict yet recorded. Per operator workflow guidance, [PR] Post-Review must run after code review. Recommend dispatching `bmad-code-review` for 17-0 as next orchestrator action.

### Recommended Next Steps (in priority order)

1. **Resolve CR-7 first.** Schedule a `bmad-agent-ux-designer` (Sally) pass for E18 surfaces — this is the M2-deadline blocker.
2. **In parallel, dispatch `bmad-code-review` for Story 17-0** so it can transition from `review` → `done` and unblock 17-1 story creation.
3. **Resolve CR-1 + CR-2 + CR-4** in a single 30–60 min planning-hygiene pass. Same recommendation as prior IR; still not done.
4. **Resolve CR-8 (E18 design questions)** during S18.0x story creation in `bmad-create-story` (i.e. these are "fill in the blanks" rather than "halt and clarify").
5. **Begin E18 kickoff** with `bmad-create-story` for `18-0` (Public /trust Route + MDX Pipeline + Sub-Processor YAML + Change-Log Generator). Per operator workflow guidance, this MUST be followed by `[VS] Validate Story` (non-negotiable) before dev.
6. **Dispatch `inj-01-dependabot-configuration`** in parallel with E18 work — closes both CR-6 and R-018-4 (NFR-9 for E18 deps).

### Final Note

This v2 assessment confirms that **Epic 18 is specification-ready at the PRD + Architecture + Epic-AC level**, but that the **UX-spec layer is the critical blocker** for the M2 hard deadline. Without a UX pass, the frontend story (S18.00) cannot be created with meaningful ATDD source-inspection assertions, and the resulting customer-facing surface risks ad-hoc design decisions during dev that diverge from the established design system.

**Recommendation:** **Proceed to `bmad-agent-ux-designer` for E18 surfaces immediately, then `bmad-create-story` for 18-0 once UX-spec supplement is in place.** All other CRs (CR-1, CR-2, CR-4, CR-5, CR-6, CR-8, CR-9) can be resolved in parallel or during story creation rather than blocking E18 kickoff specifically.

---

## HALT determination

**No HALT.** Epic 18 spec is coherent and aligned with PRD-amendment + architecture. None of the open issues constitute a "spec is broken or incoherent" condition. The blockers are:
- Hygienic (CR-1, CR-2, CR-4) — file organisation, not spec correctness.
- Coverage gaps (CR-5, CR-7) — UX-spec layer needs supplements before frontend stories.
- Process (CR-9) — pending senior code review for 17-0.

**Operator decision point:** Schedule a `bmad-agent-ux-designer` pass for E18 surfaces (CR-7) before invoking `bmad-create-story` for `18-0`, OR accept the risk that S18.00's frontend ATDD will be weakly grounded.

---

*Generated by `bmad-check-implementation-readiness` skill in BMAD autopilot mode (no operator prompts) on 2026-04-28. This v2 supersedes implementation-readiness-report-2026-04-28.md (v1) for E18 kickoff readiness; v1's E17 specification findings remain in force where not explicitly updated here.*
