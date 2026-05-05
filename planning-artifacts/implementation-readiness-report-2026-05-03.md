---
stepsCompleted:
  - step-01-document-discovery
  - step-02-prd-analysis
  - step-03-epic-coverage-validation
  - step-04-ux-alignment
  - step-05-epic-quality-review
  - step-06-final-assessment
inputDocuments:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/ux-spec.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E17-crm-integrations.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E18-trust-center-compliance.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-03.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-04-28-v2.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/sprint-status.yaml"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/project-context.md"
focus: "Pre-kickoff [IR] for Epic 18 (Trust Center & Compliance Posture); checkpoint Epic 17 close-out (17-3 in review); reconcile open CRs from IR-v2 (2026-04-28)"
date: 2026-05-03
assessor: "PM (BMAD autopilot, bmad-check-implementation-readiness)"
supersedes: "implementation-readiness-report-2026-04-28-v2.md"
---

# Implementation Readiness Assessment Report — IR-v3

**Date:** 2026-05-03
**Project:** EU Solicit
**Mode:** BMAD autopilot (no operator prompts)
**Assessor role:** Product Manager — requirements traceability + planning-hygiene critique
**Trigger:** Operator BMAD-stream guidance, "Before starting any epic, run [IR] Implementation Readiness." Epic 17 is closing on 17-3 Approve verdict; **Epic 18 (Trust Center) is the next epic at `backlog`**. IR-v2 (2026-04-28) recorded 8 open CRs; this IR-v3 reconciles 5 working days of delta.

---

## Executive Verdict

**Overall Readiness for Epic 18 kickoff: NEEDS WORK — proceed with named caveats.**

**HALT determination: NO HALT.** The Epic 18 spec is coherent across PRD-amendment + architecture + epic-AC. The blockers are **coverage gaps and process drift**, not spec incoherence. However, **CR-7 (E18 UX gap, 6 surfaces, now 5 working days aged)** remains a hard prerequisite for `bmad-create-story` on `18-0` to produce a story file with grounded ATDD source-inspection assertions. **Operator decision required:** schedule `bmad-agent-ux-designer` (Sally) before invoking `bmad-create-story` for `18-0`, OR accept ATDD-grounding risk for the M2-deadline frontend story.

| Dimension | Verdict | Notes |
|---|---|---|
| PRD coverage of Epic 18 (FR10) | 🛑 Split-source unchanged | Verified by `grep`: `PRD.md` returns **0 matches** for `FR10` / `Trust Center` / `sub.processor` / `FR8` / `FR9` / `FR11`. All FR10.x lives only in `prd-amendment-2026-04-25.md`. CR-4 unresolved for 5+ working days. |
| Architecture coverage of Epic 18 | ✅ Strong (unchanged from IR-v2) | ADR-011, public-route ingress rule, S3 artefact pipeline, sub-processor change → notification flow all documented. |
| UX coverage of Epic 18 | 🛑 **Critical gap, aged 5 working days** | `ux-spec.md` (last modified 2026-04-28) — verified by `grep`: only 3 incidental matches for "trust" (semantic uses in §7.9 "Audit and trust signals", §6.5 narrative, regulatory copy). **Zero matches** for `/trust`, `MDX`, `sub.processor`, public-route, public AppShell variant, compliance posture. CR-7 unaddressed. |
| Epic 18 spec quality | ✅ Adequate | 3 stories sized (5/5/3 = 13 pts), dependencies clear (E03/E07/E09 all `done`), 9 ACs testable, M2 hard deadline + GDPR Art. 28 30-day rule explicit. |
| Epic 17 close-out readiness | ⚠ Partial | 17-0/17-1/17-2 `done` (with AP13-03 caveat — see CR-9). 17-3 in `review` awaiting `bmad-code-review` pass-3. Epic 17 closes on 17-3 Approve verdict. |
| Planning-artifact hygiene (CR-1) | 🛑 Unresolved | `planning-artifacts/_archive/` and `planning-artifacts/epics/_archive/` both verified non-existent. 80+ stale epic files persist across 6+ naming conventions. |
| Planning-view drift (CR-2) | 🛑 Unresolved | `epics.md` (modified 2026-04-30) still shows the 9-epic legacy view (Epics 1–9). E10–E21 invisible from this view. |
| PRD source unification (CR-4) | 🛑 Unresolved | `PRD.md` (modified 2026-04-27) — base PRD still missing FR8/FR9/FR10/FR11. |
| UX gap for E17 surfaces (CR-5) | ⚠ Unresolved (now low-impact, Epic 17 nearly closed) | No CRM connection card / conflict log / stage-mapping UX in `ux-spec.md`. Stories shipped via inline ATDD source-inspection regardless. |
| Epic 13 carry-forwards (CR-6) | ⚠ Unresolved — **12th consecutive epic boundary** | All 7 stories (drift-recovery, dw-01..03, inj-01..03) still `ready-for-dev`. `inj-01` (Dependabot) is named E18 dependency-scan gate (R-018-4). |
| **NEW: AP13-03 triple-fire (CR-9 evolved)** | 🛑 **CRITICAL — structural** | Sprint-status lines 292/293/294 (17-0, 17-1, 17-2) each show `development_status: done` with inline trailing note `"Status: review (awaiting bmad-code-review pass N)"`. Three production-deployed stories carry no senior-developer-review Approve verdict on file — Beta/GA-gate audit-trail liability. Epic 15 retrospective named this CRITICAL ACTION #1; AP16 register named it 5th recurrence; v18 of `bmad-correct-course` (sprint-change-proposal-2026-05-03.md) records the 5th, 6th, and 7th recurrences in a single 72-hour window. |

**Net change since IR-v2 (2026-04-28-v2):** 0 of 8 prior CRs resolved. CR-7 aged from 0 → 5 working days. CR-9 evolved from "single-instance review-pass dispatch lag" to "structural triple-fire AP13-03 anti-pattern". CR-3 (Epic 16 status) remains the only CR closed since IR-v1 (2026-04-28).

---

## Step 1 — Document Discovery

### Canonical documents (operative source of truth)

| Document | Path | Last modified | Status |
|---|---|---|---|
| PRD (base) | `planning-artifacts/PRD.md` | 2026-04-27 | Operative; **missing FR8–FR11** |
| PRD amendment | `planning-artifacts/prd-amendment-2026-04-25.md` | 2026-04-25 | Operative; **sole source of FR10** |
| Architecture | `planning-artifacts/architecture.md` | 2026-04-28 | Operative |
| UX spec | `planning-artifacts/ux-spec.md` | 2026-04-28 | Operative; **zero coverage of E17/E18 surfaces** |
| Epics roadmap (legacy view) | `planning-artifacts/epics.md` | 2026-04-30 | **Drifted** — still shows 9-epic legacy view; E10–E21 invisible |
| Epic 17 spec | `planning-artifacts/epics/E17-crm-integrations.md` | (per-epic file authoritative) | Operative |
| Epic 18 spec | `planning-artifacts/epics/E18-trust-center-compliance.md` | (per-epic file authoritative) | Operative |
| Sprint status | `implementation-artifacts/sprint-status.yaml` | 2026-05-03 | Operative |
| project-context | `planning-artifacts/project-context.md` | 2026-04-27 | Operative; **no Epic 17 retro entries yet** (Epic 17 still in-progress) |
| Latest correct-course proposal | `planning-artifacts/sprint-change-proposal-2026-05-03.md` | 2026-05-03 | v18 of bmad-correct-course; same recommendations as v17, same brief, **18th consecutive fire** |

### Hygiene status (verified 2026-05-03)

```
$ ls planning-artifacts/_archive/        → does not exist
$ ls planning-artifacts/epics/_archive/  → does not exist
```

**CR-1 and CR-2 actions from IR-v1 (2026-04-28) have not been executed.** Risk profile is unchanged: any agent loading from `planning-artifacts/epics/` will see ~80 stale epic files across 6+ naming conventions. Per-epic E**-prefixed files are authoritative; legacy `epic-NN-...md` files are stale; `epics.md` (the consolidated roadmap view) is also stale.

### New since IR-v2

- `implementation-artifacts/sprint-status.yaml` — multiple updates (Stories 17-1, 17-2, 17-3 created; 17-0/17-1/17-2 transitioned to `done` with AP13-03 caveat; 17-3 in `review`).
- `implementation-artifacts/17-1-...`, `17-2-...`, `17-3-...md` — three new story files.
- `planning-artifacts/sprint-change-proposal-2026-04-30.md` (v17) and `2026-05-03.md` (v18) — two further bmad-correct-course fires.
- No new IR report between v2 (2026-04-28) and this IR-v3 (2026-05-03). The 2026-04-30 stub (`implementation-readiness-report-2026-04-30.md`, 1KB, 1039 bytes) has no operative content — IR-v2 remains the prior-IR baseline.

---

## Step 2 — PRD Analysis (Epic 18 focus)

### FR10 (Trust Center & Compliance Posture) — present **only in amendment**

`prd-amendment-2026-04-25.md` lines 156–177 define the full FR10 surface:

- **FR10.1** — Public `/trust` page lists GDPR (compliant), ISO 27001 (in progress with target M12), SOC 2 (planned/N-A — secondary in EU), data residency (EU only), encryption (AES-256 at rest, TLS 1.3 in transit).
- **FR10.2** — Downloadable PDF artefacts: GDPR DPA, Security Overview, Sub-Processor List, latest pen-test summary (redacted), BCP summary, Data Residency confirmation.
- **FR10.3** — Sub-Processor List auto-updates from `infra/sub-processors.yaml`.
- **FR10.4** — Versioned artefacts with change-log entries; sub-processor change → email to active customer DPAs (GDPR Art. 28).

`PRD.md` (base) — **0 matches** (verified) for any of: `FR10`, `Trust Center`, `sub.processor`, `FR8`, `FR9`, `FR11`. Same fragmentation footgun called out for E17 in IR-v2; **no progress in 5 working days**.

### NFR coverage for Epic 18

| NFR | Relevance to E18 | Coverage |
|---|---|---|
| NFR-5/6 (auth, encryption) | `/trust/*` is the **only unauthenticated route** — needs explicit ingress allow-list and confirmation no auth-required middleware leaks | ✅ Architecture §11 line 569 (Routing & Path Architecture); E18 AC bullet 1 |
| NFR-7 (data isolation) | E18 must not leak any tenant data via the public route | ✅ E18 AC bullet 9 ("cross-tenant negative test: unauthenticated request returns full content with no user data") |
| NFR-9 (third-party deps — Dependabot) | WeasyPrint + new MDX pipeline deps (`@next/mdx`, `gray-matter`) need scanning | 🛑 Carry-forward `inj-01-dependabot-configuration` still `ready-for-dev` (CR-6, 12th deferral). E18 ships unscanned MDX/PDF deps unless `inj-01` lands first. R-018-4 |
| NFR-12 (auditability — ISO 27001) | E18 IS the ISO 27001 evidence vehicle | ✅ Epic goal narrative + ADR-011 explicitly tie static-rendered architecture to ISO change-management evidence |
| NFR-22 (notification reliability — Art. 28 emails) | Sub-processor change → email to all paid DPAs is GDPR-mandated | ✅ E18 S18.02 reuses Epic 9 `notification` service + canonical fire-and-forget audit write (project-context Epic 13 Rule 45) |

### PRD completeness assessment for Epic 18

✅ **Functional scope is fully specified when amendment is read alongside base PRD.**
🛑 **Source-of-truth fragmentation persists.** Same blocker as Epic 17 — CR-4 must land before either E17.x or E18.x story creation routinely cites PRD sources from a single file. **5 working days aged with no movement.**

---

## Step 3 — Epic Coverage Validation

### Sprint state delta since IR-v2 (2026-04-28-v2)

| Epic | Status (was, 2026-04-28) | Status (now, 2026-05-03) | Delta |
|---|---|---|---|
| 13 | `done` (with 6 carry-forwards) | unchanged | All 6 carry-forwards still `ready-for-dev`; `epic-13-retrospective: done` |
| 14 | `done` | unchanged | retrospective `done` |
| 15 | `done` | unchanged | retrospective `done` |
| 16 | `done` | unchanged | retrospective `done` |
| 17 | `in-progress` (17-0 in review) | `in-progress` (17-3 in review; 17-0/17-1/17-2 `done` with AP13-03 caveat) | 4/4 stories created; 3/4 dev-complete; 1/4 in `review`; **on track to close on 17-3 Approve verdict** |
| 18 | `backlog` | unchanged | **Next epic to kick off**; CR-7 unaddressed |
| 19/20/21 | `backlog` | unchanged | — |

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
| M2 deadline tracked in `eusolicit-docs/` | Amendment §11.4 (locked decision) | ⚠ No tracking artefact specified — see CR-8 |
| Cross-tenant negative test (unauthenticated → no user data leak) | E18 AC bullet 9 | ✅ |

**E18 ACs map cleanly to PRD-amendment + architecture sources.** No coverage gap at the epic-AC ↔ requirements level. Gap is at the UX-spec layer (Step 4) and the planning-hygiene layer (CR-1/CR-2/CR-4).

### Epic 17 — close-out checkpoint (17-3 the only remaining lever)

Story 17-3 (Salesforce Adapter — Daily-Quota-Aware) is in `review` since 2026-05-03. Pass-2 `bmad-code-review` verdict was Changes Requested (B-1..B-6 + A-1..A-8). Pass-3 `bmad-dev-story` review-fix landed today, closing B-1..B-6 + A-1, A-5, A-7, A-8 + A-4 verified. Pending pass-3 `bmad-code-review` Approve verdict for the epic to close at 4/4 done.

**Risk:** if the AP13-03 anti-pattern triple-fires for a fourth time on 17-3 (i.e., 17-3 transitions `review` → `done` via `bmad-dev-story` review-fix loop instead of code-review Approve), the entire Epic 17 fan-out closes without a single senior-developer-review Approve verdict on file. **This MUST not happen.** v18 of `bmad-correct-course` calls this out as the single most important sequencing intervention.

---

## Step 4 — UX Alignment

### Critical finding: Epic 18 has **zero meaningful UX spec coverage** (verified 2026-05-03)

`ux-spec.md` (modified 2026-04-28) was searched for: `trust`, `sub.processor`, `MDX`, `/trust`, `public route`, `unauthenticated page`, `public shell`, `compliance posture`. **Result: 3 incidental matches**, all semantic / regulatory in nature:

- Line 65: "**Trust through transparency.** Every AI conclusion is sourced." (Design principle, not E18 surface)
- Line 273: "Downgrade and cancellation are **as discoverable as upgrade** ... a regulatory and trust requirement." (Billing UX)
- Line 484: "### 7.9 Audit and trust signals" (Audit UX)

**Zero matches** for the actual E18 customer-facing surfaces. The customer-facing layer for Epic 18 is entirely unspecified at the UX-spec level:

1. **Public `/trust` page layout** — There is no public variant of `<AppShell>` specified. AC bullet 4 of S18.00 says "Layout reuses existing `<AppShell>` patterns minus authenticated topbar (public variant)" — but the public variant does not exist.
2. **Compliance posture cards** (GDPR / ISO 27001 / SOC 2 / residency / encryption) — visual treatment, status-badge design, "in progress" indicator UX not specified.
3. **ISO 27001 roadmap component** (M3/M9–10/M12 milestones with current-stage indicator) — pure E18 surface, no precedent in spec.
4. **Downloadable artefact list** — file-card design, signed-URL handling, error states, locale handling for legal artefacts (BG/EN versions of DPA?) not specified.
5. **Change-log section** — version-history UX, diff-display semantics, subscriber-only vs public visibility not specified.
6. **Sub-processor change email template** (BG/EN) — copy + i18n keys not drafted.

### Cumulative UX debt

| Epic | UX surfaces unspecified | Status |
|---|---|---|
| E17 (CRM) | 5 surfaces (CR-5) | Unresolved since IR-v2; **low operational impact now** — Epic 17 stories shipped via inline ATDD source-inspection without UX-spec grounding (acceptable but technical debt) |
| E18 (Trust Center) | 6 surfaces (CR-7) | **Unresolved 5 working days; HIGH operational impact** — S18.00 (`Type: frontend`) cannot be created with grounded ATDD until resolved; M2 hard deadline blast radius widening |

**Total UX surfaces unspecified for in-flight + next-up epics: 11** (unchanged from IR-v2). A `bmad-agent-ux-designer` (Sally) pass for E18 must precede `bmad-create-story` for `18-0`. **This is the single highest-leverage planning action available to the operator right now.**

---

## Step 5 — Epic Quality Review

### E18 spec quality scorecard (unchanged from IR-v2)

| Dimension | Score | Evidence |
|---|---|---|
| Goal clarity | A | "mid-large EU consulting-firm deals stall 4–8 weeks at the procurement gate" — concrete deal-blocker framing tied to M2 hard deadline |
| Acceptance criteria specificity | A− | 9 ACs, all testable; M2 deadline + GDPR Art. 28 30-day rule + AES-256 / TLS 1.3 explicit |
| Story decomposition | A | 3 stories (S18.00 frontend MDX/route, S18.01 PDF pipeline, S18.02 notification flow); 5/5/3 = 13 pts matches header |
| Dependency declaration | A | E03 (frontend shell), E07 (WeasyPrint), E09 (notification) — all `done` per sprint-status; no upstream blockers |
| Test design | B | Cross-tenant negative test, i18n parity, ATDD source-inspection (MDX vs runtime Markdown), 30-day-future lint test, signed-URL TTL test, audit log delivery test all called out. **No `test-design-epic-18.md` exists** — story files will need to fill the gap (mirror S15.1 / S17.0 inline pattern). |
| Source-of-truth citations | A− | Cites "PRD v1.1 §6 FR10 + §8 US10", "architecture-evaluation §2 Change-4", "§11 locked decision §11.4 (M2 deadline)" — verifiable in amendment + architecture-evaluation file |
| Net-new fence (BMM Rule) | A | Each story is single-concern; living-vs-legal artefact split is explicit |
| Anti-pattern guard rails | B+ | WeasyPrint `run_in_executor` requirement (project-context Epic 7) called out; canonical fire-and-forget audit write (Rule 45) cited; **missing**: explicit "no production-side test routes" (S15-0 B1), "no raw text() INSERTs" (E14.2 BLOCKING #3), "canonical ORM seeding only" — not as relevant for E18 (mostly content + CI), but should be enumerated in story-file Anti-Pattern fences for consistency with E14/15/16/17 carry-forwards |

### E18 risks flagged for the dev pass (carry-forward from IR-v2)

- **R-018-1:** Public-route ingress rule is the first **unauthenticated** route in the platform; any misconfiguration leaks the auth-redirect into the public flow OR allows authenticated routes to leak. Reviewer must include path-traversal probe (`/trust/../api/v1/...`).
- **R-018-2:** S18.02's "30-day-future lint test" blocks PRs adding sub-processors with <30-day notice. ADR or playbook needed for legal-emergency override path.
- **R-018-3:** S18.01's signed-URL-1-hour-TTL — does a 1-hour signed URL satisfy "publicly downloadable" per amendment FR10.2? **Story should pick (a) stable public-read S3 URL with bucket-policy gating OR (b) fresh signed URLs server-side per page load before dev.**
- **R-018-4:** WeasyPrint dependency proliferation. `inj-01-dependabot` still `ready-for-dev` (CR-6, 12th deferral); E18 ships unscanned MDX + WeasyPrint + Next.js MDX deps unless Dependabot lands first.
- **R-018-5:** Locale handling for legal artefacts. EU consulting-firm deals require BG-language DPA for Bulgarian counterparties. AC bullet 3 lists artefacts in English; no BG-language equivalent specified. Confirm legal-counsel position OR scope BG variant for each legal artefact.

### Epic 17 in-flight checkpoint (closed-loop verification needed)

- **17-0 / 17-1 / 17-2:** All marked `done` in sprint-status; all carry inline `"awaiting bmad-code-review pass N"` notes. **AP13-03 5th, 6th, 7th recurrences in 72h.** Three production-deployed stories with no senior-developer-review Approve verdict on file. **Beta/GA-gate audit-trail liability.**
- **17-3:** In `review` since today (2026-05-03); pass-3 dev-fix landed; awaiting `bmad-code-review` pass-3 Approve verdict. **The single in-flight item with a clean dispatch path.**

---

## Step 6 — Final Assessment

### Summary

Epic 18 is **specification-ready at the PRD-amendment + architecture + epic-AC level**. The blockers are:

1. **CR-7 (E18 UX gap, 6 surfaces, 5 working days aged)** — operational blocker for `bmad-create-story` on `18-0`.
2. **CR-9 evolved (AP13-03 triple-fire)** — process-integrity blocker for Epic 17 close-out integrity; must not extend to 17-3.
3. **CR-1 + CR-2 + CR-4 (planning-artifact hygiene + PRD source unification)** — discoverability and source-of-truth blockers; **5 working days aged with zero movement**.
4. **CR-6 (Epic 13 carry-forwards, 12th deferral)** — `inj-01` (Dependabot) is the named E18 NFR-9 gate (R-018-4); shipping E18 before `inj-01` means MDX/WeasyPrint/Next.js MDX deps land unscanned.

### Critical Issues Requiring Immediate Action (ranked)

1. **CR-9 (AP13-03 triple-fire) — 🛑 CRITICAL — NEW since IR-v2.**
   Sprint-status lines 292/293/294 (17-0, 17-1, 17-2) each show `development_status: done` with inline `"awaiting bmad-code-review pass N"`. **Action:** dispatch `bmad-code-review` re-passes for 17-0, 17-1, 17-2 sequentially after 17-3 ships, to retroactively obtain Approve verdicts. Without this, three production-deployed stories have no senior-developer-review evidence on file.
   **Also:** 17-3 must NOT mirror this pattern. On 17-3 pass-3 review, the verdict MUST be set explicitly (Approve / Changes-Requested / Reject) and recorded in story file §7. Do not transition `done` without Approve.

2. **CR-7 (E18 UX gap) — 🛑 CRITICAL — aged 5 working days from IR-v2.**
   6 E18 surfaces unspecified (public `<AppShell>` variant, compliance posture cards, ISO 27001 roadmap component, artefact-download UI, change-log section, sub-processor email template). **Action:** dispatch `bmad-agent-ux-designer` (Sally) for E18 surfaces BEFORE invoking `bmad-create-story` for `18-0`. Output should be append-only supplements to `ux-spec.md` covering the 6 surfaces. M2 hard deadline blast radius widening.

3. **CR-1 (planning-artifact hygiene) — 🛑 HIGH — unchanged from IR-v1 (now 8 working days aged).**
   Move 80+ stale epic files to `planning-artifacts/epics/_archive/`; move PRD/UX backups to `planning-artifacts/_archive/`. 30–60 min hygiene pass.

4. **CR-2 (planning view drift) — 🛑 HIGH — unchanged from IR-v1.**
   Reconcile `epics.md` (9-epic legacy view) vs operational E10–E21 set. Pick Option A (rename + new full-roadmap file) or Option B (regenerate to merge).

5. **CR-4 (PRD source unification) — 🛑 HIGH — unchanged from IR-v1.**
   Merge `prd-amendment-2026-04-25.md` into `PRD.md` and stamp v1.1 (or v2.0). Same blocker now applies to **both E17 and E18** — every BMAD agent loading `PRD.md` cannot see FR8/FR9/FR10/FR11.

6. **CR-6 (Epic 13 carry-forwards) — ⚠ HIGH — 12th consecutive epic boundary deferral.**
   7 stories at `ready-for-dev` (drift-recovery, dw-01..03, inj-01..03). **`inj-01` (Dependabot) is the named E18 NFR-9 gate (R-018-4).** Without `inj-01`, E18 ships unscanned MDX/WeasyPrint/Next.js MDX deps. Recommend gating S18.00 PR-merge on `inj-01` being merged first.

7. **CR-8 (E18 design questions) — ⚠ MEDIUM — unchanged from IR-v2.**
   - R-018-3: Signed URL vs public-read S3 for legal artefacts — pick one before S18.01 enters dev.
   - R-018-5: BG-language legal artefacts — confirm legal position or scope.
   - R-018-2: Sub-processor <30-day-future override path — playbook needed.

8. **CR-5 (UX gap for E17 surfaces) — ⚠ LOW (now low-impact) — unchanged from IR-v2.**
   5 E17 surfaces unspecified. Epic 17 stories shipped via inline ATDD source-inspection regardless. Treat as backlog technical debt, not a blocker.

### Recommended Next Steps (in dispatch order, per Operator BMAD-stream guidance)

1. **Dispatch `bmad-code-review` for Story 17-3** — pass-3 verdict closes Epic 17 fan-out (4/4 stories). Reviewer must verify pass-2 closures B-1..B-6 + A-1/A-5/A-7/A-8 + A-4 verified + A-2 deviation #26 + standard reviewer pass per story §4.6 24-row anti-pattern fence. **Critical reviewer instruction:** verdict MUST be set explicitly (Approve / Changes-Requested / Reject); do not transition `done` without Approve.
2. **In parallel, dispatch `bmad-agent-ux-designer` (Sally) for E18 surfaces (CR-7).** Output: append-only ux-spec.md supplements covering the 6 named surfaces. Resolve R-018-3 + R-018-5 inline. **Gates `bmad-create-story` for 18-0.**
3. **After 17-3 closes, dispatch `bmad-code-review` re-passes for 17-0, 17-1, 17-2 sequentially** (retroactive AP13-03 closure — CR-9). Each re-pass either issues retroactive Approve (closing the AP13-03 register) OR raises net-new findings forcing return to `review`.
4. **Resolve CR-1 + CR-2 + CR-4** in a single 30–60 min planning-hygiene pass. Same recommendation as IR-v1 + IR-v2; still not done. **8 working days aged.**
5. **Dispatch `inj-01-dependabot-configuration`** in parallel with E18 work — closes both CR-6 (12th-deferral) and R-018-4 (NFR-9 for E18 deps). Single `.github/dependabot.yml` file, <30 minutes per Epic 8 retrospective sizing.
6. **Begin E18 kickoff** with `bmad-create-story` for `18-0` (Public /trust Route + MDX Pipeline + Sub-Processor YAML + Change-Log Generator), **only after CR-7 (UX) lands**. Per operator workflow guidance, this MUST be followed by `[VS] Validate Story` (non-negotiable) before dev. Epic 18 is multi-story → run `[SR] Story Review` after each story completes; run `[PR] Post-Review` after code review of each story; consider `[ER] Epic Review` after S18.02 closes (3 stories with content + CI + notification interdependencies).
7. **Before E18 kickoff, also confirm:** Epic 17 retrospective (currently `optional`) — operator decision; if Epic 17 is the first epic to close with all 4 provider stories shipped under the AP13-03 anti-pattern, the retro is **operationally important** for codifying the structural pattern observation (orchestrator dispatch routing biased toward `dev-pass-completes → set-done` rather than `code-review-Approve → set-done`).

### CR Summary Table (consolidated, 2026-05-03)

| CR | Severity | Title | Aged (working days) | Action |
|---|---|---|---|---|
| CR-1 | 🛑 HIGH | Planning-artifact hygiene (80+ stale epic files) | 8 | Hygiene pass; move to `_archive/` |
| CR-2 | 🛑 HIGH | Planning view drift (epics.md 9-epic legacy view) | 8 | Rename or regenerate epics.md |
| CR-3 | ✅ CLOSED | Epic 16 status integrity | — | Closed in IR-v2 |
| CR-4 | 🛑 HIGH | PRD source unification (FR8–FR11 missing from PRD.md) | 8 | Merge amendment into PRD.md |
| CR-5 | ⚠ LOW | UX gap for E17 surfaces (5 unspecified) | 5 | Backlog tech debt; ship inline ATDD |
| CR-6 | ⚠ HIGH | Epic 13 carry-forwards (12th deferral) | 12 epic boundaries | Dispatch inj-01 (Dependabot) at minimum |
| CR-7 | 🛑 CRITICAL | UX gap for E18 surfaces (6 unspecified) | 5 | Dispatch Sally BEFORE bmad-create-story for 18-0 |
| CR-8 | ⚠ MEDIUM | E18 design questions (R-018-2/3/5) | 5 | Resolve during story creation |
| CR-9 | 🛑 CRITICAL — evolved | AP13-03 triple-fire (17-0/17-1/17-2 done without Approve) | 5 → 0 (just landed) | Sequential bmad-code-review re-passes after 17-3 closes |

---

## HALT determination

**No HALT.** Epic 18 spec is coherent and aligned with PRD-amendment + architecture. None of the open issues constitute a "spec is broken or incoherent" condition. The blockers are:

- **Coverage gaps** (CR-7 → UX) — actionable via single Sally dispatch.
- **Process-integrity** (CR-9 → AP13-03 triple-fire) — actionable via 4 sequential `bmad-code-review` dispatches (17-3 forward, then 17-0/17-1/17-2 retroactive).
- **Source-of-truth fragmentation** (CR-1/2/4) — actionable via single 30–60 min planning-hygiene pass.
- **NFR-9 dependency posture** (CR-6 → R-018-4) — actionable via single `inj-01` (Dependabot) dispatch (<30 min per Epic 8 sizing).

**Operator decision points:**

1. **Schedule Sally (`bmad-agent-ux-designer`) pass for E18 surfaces (CR-7)** before invoking `bmad-create-story` for `18-0`, OR explicitly accept the risk that S18.00's frontend ATDD will be weakly grounded against an undefined UX surface for the M2 hard-deadline epic.
2. **On 17-3 `bmad-code-review` pass-3 dispatch:** reviewer MUST explicitly set verdict (Approve / Changes-Requested / Reject) and record in story file §7. Do NOT allow inline "awaiting bmad-code-review pass N" notes to substitute for verdict closure (AP13-03 8th recurrence would be the single most damaging signal in the project's audit trail).
3. **Retroactive `bmad-code-review` re-passes for 17-0/17-1/17-2 (CR-9):** dispatch sequentially after 17-3 closes. Without retroactive closure, three production-deployed stories carry no senior-developer-review evidence on file — Beta/GA-gate audit-trail liability.

---

*Generated by `bmad-check-implementation-readiness` skill in BMAD autopilot mode (no operator prompts) on 2026-05-03. This IR-v3 supersedes implementation-readiness-report-2026-04-28-v2.md for Epic 18 kickoff readiness; v2's Epic 17 specification findings remain in force where not explicitly updated here. The 2026-04-30 stub (1KB) was not operative content and is superseded by reference.*
