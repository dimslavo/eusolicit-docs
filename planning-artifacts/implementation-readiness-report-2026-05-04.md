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
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E18-trust-center-compliance.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-04.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-03.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/sprint-status.yaml"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-17-retro-2026-05-03.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/project-context.md"
focus: "Mid-Epic 18 [IR] — overdue per BMAD-stream non-negotiable. Reconcile 18-1 in review, 18-0 done-without-Approve (AP17-C1 9th instance), 18-2 backlog gated on Sally pre-pass. Confirm CR-7 / CR-1/2/4 / CR-6 unresolved. v19 sprint-change-proposal has explicitly deferred all carry-forwards to this IR-v4."
date: 2026-05-04
assessor: "PM (BMAD autopilot, bmad-check-implementation-readiness)"
supersedes: "implementation-readiness-report-2026-05-03.md"
---

# Implementation Readiness Assessment Report — IR-v4

**Date:** 2026-05-04
**Project:** EU Solicit
**Mode:** BMAD autopilot (no operator prompts)
**Assessor role:** Product Manager — requirements traceability + planning-hygiene critique
**Trigger:** (a) Operator BMAD-stream guidance — *"Before starting any epic, run [IR] Implementation Readiness."* Epic 17 closed 2026-05-03 and Epic 18 entered without an interceding [IR]; this IR-v4 is the **overdue mid-epic readiness check** that should have fired between Epic 17 close and 18-0 dispatch. (b) v19 of `bmad-correct-course` (`sprint-change-proposal-2026-05-04.md`) explicitly defers carry-forward routing to IR-v4 (its §4 / §5 / §6 / §Out-of-scope items 1, 5, 6).

---

## Executive Verdict

**Overall Readiness for remainder of Epic 18 + Epic 18 close + Epic 19 kickoff: NEEDS WORK — proceed with named caveats.**

**HALT determination: NO HALT.** Epic 18 specification remains coherent across PRD-amendment + architecture + epic-AC. The blockers are unchanged in nature from IR-v3: **process-integrity drift, coverage gaps, source-of-truth fragmentation, and 13th-fire carry-forward deferral**, not spec incoherence. **However, the AP17-C1 anti-pattern has now fired 5 times** (17-0/17-1/17-2/17-3 + 18-0; epic-17-retro codified the first 4), and Story 18-1 has shipped 3 unscanned dependency introductions (WeasyPrint, markdown-it-py, python-frontmatter) — exactly the IR-v3 R-018-4 prediction now realised as **first concrete material harm from the inj-01 13th-deferral**. Operator decision required: dispatch the Sally pre-pass for E18 surfaces NOW (gates `bmad-create-story` for 18-2 sub-processor email template), or accept ATDD-grounding risk for the third E18 story under the M2 hard deadline.

| Dimension | Verdict | Notes (delta vs IR-v3 in **bold**) |
|---|---|---|
| PRD coverage of Epic 18 (FR10) | 🛑 Split-source unchanged | `PRD.md` (modified 2026-04-27) verified by `grep`: still **0 matches** for `FR10` / `Trust Center` / `sub.processor` / `FR8` / `FR9` / `FR11`. **CR-4 aged from 8 → 9 working days. v19 explicitly leaves CR-4 out of scope.** |
| Architecture coverage of Epic 18 | ✅ Strong (improved) | architecture.md verified by `grep` with 15 matches: ADR-011 line 764 explicit; `/trust/*` public-route ingress rule line 576; CloudFront cache rule line 633; S3 artefact registry path line 932; Trust Center sub-processor email flow line 167. **Architecture.md modified 2026-05-04 02:03** but content for E18 is unchanged from IR-v3 baseline (`revalidationOutcome` line 23 confirms "no architectural drift"). |
| UX coverage of Epic 18 | 🛑 **Critical gap, aged 6 working days** | `ux-spec.md` (modified 2026-05-04 02:06) verified by `grep` for `trust|sub.processor|MDX|/trust|public route|public.+shell|public.+variant|compliance posture|appshell|ISO 27001|FR10`: still **only 3 incidental matches** (line 65 design principle "Trust through transparency"; line 273 billing-UX trust requirement; line 484 §7.9 "Audit and trust signals"). **Zero matches** for the 6 named E18 surfaces. **CR-7 aged from 5 → 6 working days. File timestamp moved but content for E18 did not.** |
| Epic 18 spec quality | ✅ Adequate (unchanged) | E18 epic file: 9 ACs testable, 3 stories sized 5/5/3 = 13 pts, dependencies on E03/E07/E09 (all `done`), M2 hard deadline + GDPR Art. 28 30-day rule explicit. |
| Epic 18 in-flight state | ⚠ Mixed — one correct, two compromised | 18-0: `done` **without** code-review Approve (AP17-C1 9th instance, inline note in sprint-status line 308 acknowledges the violation while still setting `done`). 18-1: `review` (correct two-gate state) — **single in-flight item with clean dispatch path**, 54/54 Python ATDD tests reportedly green, awaiting `bmad-code-review` pass-1. 18-2: `backlog` — gated on (a) 18-1 Approve verdict and (b) Sally pre-pass for sub-processor email template UX (CR-7). |
| Epic 17 close-out integrity | 🛑 **Compromised — quadruple-fire** | Epic 17 = `done` (4/4 stories `done`); but **all 4 stories carry inline AP17-C1 caveat — no senior-developer-review Approve verdict on file for any of 17-0/17-1/17-2/17-3.** Epic 17 retrospective ran 2026-05-03 and codified 6 critical anti-patterns including AP17-C1 (two-gate story-close); this is the **most damaging audit-trail signal in the project so far**, with explicit Beta-gate / GA-gate liability. |
| Planning-artifact hygiene (CR-1) | 🛑 Unresolved | `planning-artifacts/_archive/` and `planning-artifacts/epics/_archive/` both verified non-existent. **80+ stale epic files persist.** Aged 8 → 9 working days. |
| Planning-view drift (CR-2) | 🛑 Unresolved | `epics.md` (modified 2026-04-30) still shows the 9-epic legacy view. **Aged 8 → 9 working days.** |
| PRD source unification (CR-4) | 🛑 Unresolved | `PRD.md` still missing FR8/FR9/FR10/FR11. **Aged 8 → 9 working days.** |
| Epic 13 carry-forwards (CR-6) | 🛑 **Unresolved — 13th deferral with first concrete harm** | All 7 stories (drift-recovery, dw-01..03, inj-01..03) still `ready-for-dev`. **`inj-01` (Dependabot) deferral has produced its first concrete material harm: Story 18-1 shipped WeasyPrint + markdown-it-py + python-frontmatter dep introductions unscanned** — exactly the IR-v3 R-018-4 prediction now realised. |
| **AP17-C1 anti-pattern (CR-9 evolved)** | 🛑 **CRITICAL — quintuple-fire (5 stories)** | 17-0 + 17-1 + 17-2 + 17-3 + 18-0 all `done` without code-review Approve verdict. epic-17-retrospective codified AP17-C1..C6 + AP17-H1..H5 + AP17-M1..M2 (13 action items, 6 critical). **Sprint-status line 308 inline note now explicitly reads `"AP17-C1 two-gate: done requires bmad-code-review Approve verdict"` while still setting status to `done`** — orchestrator self-documents the anti-pattern in the same act that fires it. |

**Net change since IR-v3 (2026-05-03):** 0 of 9 prior CRs resolved. Epic 17 closed (with AP17-C1 quadruple-fire baked in). Epic 18 entered. Story 18-0 fired AP17-C1 instance #5 within ~24h of IR-v3. Story 18-1 entered `review` in correct two-gate state — a positive signal. v19 of `bmad-correct-course` explicitly defers carry-forwards to this IR-v4; v19 also names the 19th-fire-in-<24h pattern itself as the operative meta-finding (orchestrator dispatcher consuming the verbatim drift brief without consulting prior proposals as state).

---

## Step 1 — Document Discovery

### Canonical documents (operative source of truth)

| Document | Path | Last modified | Status |
|---|---|---|---|
| PRD (base) | `planning-artifacts/PRD.md` | 2026-04-27 | Operative; **missing FR8–FR11** (CR-4) |
| PRD amendment | `planning-artifacts/prd-amendment-2026-04-25.md` | 2026-04-25 | Operative; **sole source of FR10 (Trust Center)** |
| Architecture | `planning-artifacts/architecture.md` | **2026-05-04 02:03** | Operative; revalidation note (line 23) confirms no architectural drift since 2026-04-27 |
| UX spec | `planning-artifacts/ux-spec.md` | **2026-05-04 02:06** | Operative; **timestamp moved, content for E18 surfaces unchanged — CR-7 STILL UNRESOLVED** |
| Epics roadmap (legacy view) | `planning-artifacts/epics.md` | 2026-04-30 | **Drifted** — still 9-epic legacy view; E10–E21 invisible (CR-2) |
| Epic 18 spec | `planning-artifacts/epics/E18-trust-center-compliance.md` | (per-epic file authoritative) | Operative |
| Sprint status | `implementation-artifacts/sprint-status.yaml` | 2026-05-04 | Operative |
| project-context | `planning-artifacts/project-context.md` | 2026-05-03 (likely +Epic 17 retro entries) | Operative |
| Epic 17 retrospective | `implementation-artifacts/epic-17-retro-2026-05-03.md` | 2026-05-03 | NEW since IR-v3; codifies AP17-C1..C6 |
| Latest correct-course proposal | `planning-artifacts/sprint-change-proposal-2026-05-04.md` | 2026-05-04 ~01:56 | v19 of `bmad-correct-course` — 19th consecutive fire (<24h after v18) |
| Story 18-0 file | `implementation-artifacts/18-0-public-trust-route-mdx-pipeline-sub-processor-yaml-change-log-generator.md` | 2026-05-03 | NEW since IR-v3; status `done` (AP17-C1 #5 / #9) |
| Story 18-1 file | `implementation-artifacts/18-1-pdf-artefact-pipeline-weasyprint-reuse-living-artefacts-git-managed-legal-artefacts.md` | 2026-05-04 | NEW since IR-v3; status `review` |

### Hygiene status (verified 2026-05-04)

```
$ ls planning-artifacts/_archive/        → does not exist
$ ls planning-artifacts/epics/_archive/  → does not exist
$ ls planning-artifacts/epics/ | wc -l   → 80+ files, 6+ naming conventions
```

**CR-1 and CR-2 actions from IR-v1 (2026-04-28) — 9 working days aged with zero progress.** Legacy `epic-NN-...md` files still pollute the directory; per-epic `E**`-prefixed files remain authoritative; `epics.md` consolidated roadmap still shows 9-epic legacy view.

### New since IR-v3

- `implementation-artifacts/epic-17-retro-2026-05-03.md` — Epic 17 retrospective output (8 PATTERNS / 9 ANTI-PATTERNS / 13 ACTION ITEMS).
- `implementation-artifacts/18-0-...md`, `18-1-...md` — two new E18 story files.
- `implementation-artifacts/sprint-status.yaml` — multiple updates (17-3 → done; epic-17 → done; epic-17-retrospective → done; 18-0 created → ready-for-dev → done; 18-1 created → ready-for-dev → review).
- `planning-artifacts/sprint-change-proposal-2026-05-04.md` (v19) — 19th consecutive bmad-correct-course fire.
- File timestamps on `architecture.md` and `ux-spec.md` changed (2026-05-04 02:03 / 02:06); content for Epic 18 surfaces is **unchanged** vs IR-v3 baseline (`revalidationOutcome` annotation in arch line 23 confirms no edits to E18-relevant material; ux-spec grep returns the same 3 incidental matches as IR-v3).

### Document drift (CR-1 / CR-2 / CR-4 / CR-7 unchanged)

The four document-level CRs remain in the same state IR-v1 (2026-04-28) found them. v19 of `bmad-correct-course` records this as the 9-working-day-aged operator-level decision; v19 explicitly does not re-press it (anti-pattern to repeat the recommendation that has been overridden 19 times). IR-v4 records that **this state is now an established pattern** — not a transient hygiene gap — and recommends a single batched operator-level pass to clear all four CRs in one ~30–60 min session before Epic 19 kickoff.

---

## Step 2 — PRD Analysis (Epic 18 mid-epic focus)

### FR10 (Trust Center & Compliance Posture) — present **only in amendment** (unchanged from IR-v3)

`prd-amendment-2026-04-25.md` lines 156–177 remain the sole source of truth for FR10:

- **FR10.1** — Public `/trust` page with GDPR (compliant), ISO 27001 (in progress, target M12), SOC 2 (planned/N-A — secondary in EU), data residency (EU only), encryption (AES-256 at rest / TLS 1.3 in transit). **18-0 dev pass shipped this content via inline §4.4 UX mitigation** (no Sally spec to ground against).
- **FR10.2** — Downloadable PDF artefacts (DPA, Security Overview, Sub-Processor List, Pen-Test, BCP, Residency). **18-1 dev pass implements the artefact pipeline:** 4 living artefacts (WeasyPrint-rendered) + 2 legal artefacts (Git-tracked). Currently in `review`.
- **FR10.3** — Sub-Processor List auto-updates from `infra/sub-processors.yaml`. **18-1 implements this via `scripts/render_trust_pdfs.py` + jinja2 template + slug-parity gate.**
- **FR10.4** — Versioned artefacts with change-log entries; sub-processor change → email to active customer DPAs (GDPR Art. 28 30-day notice). **18-2 (backlog) implements the DPA notification flow.**

`PRD.md` (base) — verified 2026-05-04: **0 matches** for any of `FR10`, `Trust Center`, `sub.processor`, `FR8`, `FR9`, `FR11`. **9 working days aged.** Same fragmentation footgun the project has shipped for E16/E17/E18 stories now. Every BMAD agent that loads `PRD.md` cannot see the FR10 surface; agents must know to also load `prd-amendment-2026-04-25.md`. **CR-4 must be closed before Epic 19 kickoff** to prevent the same blind spot in any FR12+ surface.

### NFR coverage for the in-flight + remaining E18 work

| NFR | Relevance to E18 stories | Coverage |
|---|---|---|
| NFR-5/6 (auth, encryption) | `/trust/*` is the **only unauthenticated route** in the platform; ingress allow-list and AppShell public-variant must avoid auth-leak | ✅ 18-0 implements `apps/client/app/(public)/trust/page.tsx` per ADR-011; 18-1 implements `GET /api/v1/trust/artefacts/{slug}` public 302-redirect with explicit AST assertion that no `Depends(get_current_user)` is wired (anti-pattern #28 from story §4.6). |
| NFR-7 (data isolation) | E18 must not leak any tenant data via `/trust/*` | ✅ 18-0 + 18-1 both include cross-tenant negative tests; 18-1 AC-11 enumerates 5 boundary cases. |
| NFR-9 (third-party deps — Dependabot) | WeasyPrint + markdown-it-py + python-frontmatter | 🛑 **First concrete harm marker.** Story 18-1 shipped 3 dep introductions to `services/integrations-api/pyproject.toml` (`weasyprint>=62,<63`, `markdown-it-py>=3.0,<4.0`, `python-frontmatter>=1.1,<2.0`) **unscanned** — `inj-01-dependabot-configuration` carry-forward is the named gate (CR-6, 13th deferral); IR-v3 R-018-4 named this as the explicit prediction. |
| NFR-12 (auditability — ISO 27001) | E18 IS the ISO 27001 evidence vehicle | ✅ ADR-011 ties static-rendered architecture to ISO change-management evidence; Trust Center renders from version-controlled MDX so every change is in Git history. |
| NFR-22 (notification reliability — Art. 28 emails) | Sub-processor change → email to all paid DPAs is GDPR-mandated; 30-day-future lint on YAML | ⚠ **18-2 backlog** — not yet implemented; gated on Sally pre-pass for sub-processor email template UX (CR-7) and 18-1 Approve verdict. |

### PRD completeness assessment for the remainder of Epic 18

✅ **Functional scope is fully specified when amendment is read alongside base PRD.**
🛑 **Source-of-truth fragmentation persists for the 9th consecutive working day.** Same blocker as E16/E17/E18.0/E18.1 — every story must cross-reference `prd-amendment-2026-04-25.md` because the base `PRD.md` does not contain FR10. Story 18-2's source-of-truth audit will be more sensitive than 18-0/18-1 because 18-2 includes GDPR Art. 28 30-day-notice law that lives only in the amendment.

---

## Step 3 — Epic Coverage Validation

### Sprint state delta since IR-v3 (2026-05-03)

| Epic | Status (was, 2026-05-03) | Status (now, 2026-05-04) | Delta |
|---|---|---|---|
| 13 | `done` (with 6 carry-forwards `ready-for-dev`) | unchanged | All 7 stories (drift-recovery, dw-01..03, inj-01..03) still `ready-for-dev`. **13th-fire AP13-05.** First concrete material harm marker (`inj-01` deferral → 18-1 shipped 3 deps unscanned). |
| 14 / 15 / 16 | `done` (with retrospective `done`) | unchanged | — |
| 17 | `in-progress` (17-3 in review) | **`done`** | 4/4 stories `done`; **all 4 stories `done` without code-review Approve verdict** — AP17-C1 quadruple-fire confirmed. Retrospective ran 2026-05-03 and codified the anti-pattern. |
| 18 | `backlog` | **`in-progress`** | 18-0: `done` without Approve (AP17-C1 #5); 18-1: `review` (correct two-gate state) — **only in-flight story with clean dispatch path**; 18-2: `backlog` (gated on 18-1 Approve + Sally pre-pass). |
| 19 / 20 / 21 | `backlog` | unchanged | — |

### Epic 18 — story-level FR coverage map (mid-epic check)

| Story | AC count | Source citations | FR coverage | Gap |
|---|---|---|---|---|
| 18-0 (`done` w/o Approve) | per dev report 6 BLOCKING + 3 NON-BLOCKING closed | PRD-amendment FR10.1, FR10.3 + arch ADR-011 + arch §11 line 576 | ✅ FR10.1 (compliance posture display) + FR10.3 (sub-processor YAML auto-update) | 🛑 **No code-review Approve verdict on file (AP17-C1 #5).** Inline §4.4 UX mitigation accepted for surfaces 1–5 (CR-7 carry). |
| 18-1 (`review`) | 11 ACs | PRD-amendment FR10.2 + arch §13 (S3 artefacts) + arch line 206 (WeasyPrint `run_in_executor` requirement) | ✅ FR10.2 (PDF artefact pipeline), FR10.3 (Sub-Processor List rendering), FR10.4 partial (versioned artefacts via S3 versioning) | ⚠ Inline §4.4 UX mitigation for surfaces 1–2 (download experience + admin trigger). 1h-TTL signed URL choice unconfirmed by Sally (R-018-3). EN-only legal PDFs unconfirmed (R-018-5). 3 deps unscanned (R-018-4). |
| 18-2 (`backlog`) | (not yet created) | PRD-amendment FR10.4 + arch line 167 (notification rw) + Epic 9 dispatch pattern | ✅ FR10.4 (sub-processor change → DPA email; 30-day notice lint) | 🛑 **Gated on Sally pre-pass for sub-processor email template UX (CR-7 Surface 6)** + 18-1 Approve verdict. M2 hard deadline blast radius: 18-2 must enter dev within ~5 working days to keep M2 in reach. |

**Epic 18 ACs map cleanly to PRD-amendment + architecture sources.** No coverage gap at the epic-AC ↔ requirements level. Gaps are at the UX-spec layer (CR-7) and process-integrity layer (CR-9 → AP17-C1).

### Epic 17 close-out integrity (NEW context: retrospective is the proper venue)

Epic 17 retrospective (`epic-17-retro-2026-05-03.md`) codified 6 critical anti-patterns:

- **AP17-C1** — Two-gate story-close violation (the structural pattern observed across 17-0/1/2/3).
- **AP17-C2** — Un-skip 257 ATDD RED-phase tests in 17.1/17.2/17.3.
- **AP17-C3** — k6 baseline for integrations-api missing (12th consecutive miss).
- **AP17-C4** — TEA review gate missing (13th consecutive miss).
- **AP17-C5** — sprint-status / story-file Status discrepancy (12th occurrence).
- **AP17-C6** — `bmad-testarch-nfr` not run for E17.

This is **the right venue** for AP17-C1 codification. IR-v3 had previously recommended "elevate to Epic 17 retro" — that recommendation was discharged. The remaining Epic-17-retro action items (AP17-C1..C6 + H1..H5 + M1..M2 — 13 total) are now operator-level backlog items; IR-v4 recommends they be enumerated in `project-context.md` against the AP register and dispatched explicitly (not implicitly carried forward).

### v19 reading of the AP17-C1 5-instance fan-out

v19 of `bmad-correct-course` (line 26): *"the orchestrator is now adding the inline note 'AP17-C1 two-gate: done requires bmad-code-review Approve verdict' while still setting status to `done`."* This is **the most diagnostically useful new signal in the project's audit trail** — the orchestrator self-documents the anti-pattern in the same act that fires it. The mitigation (per Epic 17 retrospective AP17-C1 action) is to hard-gate the `done` status-transition dispatcher on explicit Approve-verdict presence in story file §7. That is a routing-layer change, not an IR-v4 deliverable; IR-v4 records it.

---

## Step 4 — UX Alignment

### Critical finding: ux-spec.md timestamp moved 2026-04-28 → 2026-05-04 02:06, but **content for E18 surfaces is unchanged**

`ux-spec.md` was searched 2026-05-04 (post-modification timestamp) for: `trust`, `sub.processor`, `MDX`, `/trust`, `public route`, `public.+shell`, `public.+variant`, `compliance posture`, `appshell`, `ISO 27001`, `FR10`. **Result: still 3 incidental matches**, all semantic / regulatory in nature, identical to IR-v3:

- Line 65: "**Trust through transparency.** Every AI conclusion is sourced." (Design principle, not E18 surface.)
- Line 273: "Downgrade and cancellation are **as discoverable as upgrade** ... a regulatory and trust requirement." (Billing UX.)
- Line 484: "### 7.9 Audit and trust signals" (Audit UX.)

**Zero matches** for the actual E18 customer-facing surfaces. The customer-facing UX layer for Epic 18 is entirely unspecified at the UX-spec level. The 6 surfaces named in IR-v2 §4 + IR-v3 §4 are **all unaddressed**:

1. Public `/trust` page layout (no public variant of `<AppShell>` specified).
2. Compliance posture cards (GDPR / ISO 27001 / SOC 2 / residency / encryption).
3. ISO 27001 roadmap component (M3/M9–10/M12 milestones with current-stage indicator).
4. Downloadable artefact list (file-card design, signed-URL handling, error states, locale handling for legal artefacts — BG vs EN).
5. Change-log section (version-history UX, diff-display semantics, subscriber-only vs public visibility).
6. Sub-processor change email template (BG/EN copy + i18n keys) — **gates 18-2 dispatch.**

### Cumulative UX debt

| Epic | UX surfaces unspecified | Status | Operational impact |
|---|---|---|---|
| E17 (CRM) | 5 surfaces (CR-5) | Unresolved since IR-v2; Epic 17 closed `done`; treat as backlog tech debt. | LOW — stories shipped via inline ATDD source-inspection; no further story creation needed. |
| E18 (Trust Center) | 6 surfaces (CR-7) | **Unresolved 6 working days; aged from 5 → 6 since IR-v3.** | **HIGH — directly gates `bmad-create-story` for 18-2** (sub-processor email template UX is Surface 6). 18-0 + 18-1 already shipped via inline §4.4 mitigation; 18-2 cannot ship the same way because the email template requires copy + i18n keys + design that does not exist anywhere yet. |

**Total UX surfaces unspecified for in-flight + remaining-this-epic + adjacent-epic work: 11** (5 E17 backlog + 6 E18; unchanged from IR-v3 baseline).

**Single highest-leverage planning action:** dispatch `bmad-agent-ux-designer` (Sally) for the 6 E18 surfaces. v17 / v18 / v19 of `bmad-correct-course` have all recommended this; 0 dispatches. **Aged 6 working days. M2 hard-deadline blast radius widening daily.**

---

## Step 5 — Epic Quality Review

### E18 spec quality scorecard (unchanged from IR-v3)

| Dimension | Score | Evidence |
|---|---|---|
| Goal clarity | A | "mid-large EU consulting-firm deals stall 4–8 weeks at the procurement gate" — concrete deal-blocker framing tied to M2 hard deadline. |
| Acceptance criteria specificity | A− | 9 ACs in epic file, all testable; M2 deadline + GDPR Art. 28 30-day rule + AES-256 / TLS 1.3 explicit. Story-level: 18-1 has 11 ACs with AST-grounded source-inspection clauses + boundary-fence enforcement. |
| Story decomposition | A | 3 stories (S18.00 frontend, S18.01 PDF, S18.02 notification); 5/5/3 = 13 pts; sequencing 18-0 → 18-1 → 18-2 tracked correctly in sprint-status. |
| Dependency declaration | A | E03 (frontend shell), E07 (WeasyPrint patterns), E09 (notification service) — all `done`; no upstream blockers. |
| Test design | B+ | 18-1 ATDD evidence (54/54 Python tests claimed green per dev report) + AST source-inspection ×8 + pytest ×46 enumerated in sprint-status line 309. **No `test-design-epic-18.md` exists** — story files compensate with inline test-design (mirrors S15.1 / S17.0 inline pattern; explicit reviewer instruction in v19 §1 to verify the inline pattern is sufficient for 18-1). |
| Source-of-truth citations | A− | E18 epic file cites "PRD v1.1 §6 FR10 + §8 US10", "architecture-evaluation §2 Change-4", "§11 locked decision §11.4 (M2 deadline)" — verifiable. Story files cite per-story; 18-1 cites PRD-amendment line ranges + ADR-011 explicitly. |
| Net-new fence (BMM Rule) | A | Each story is single-concern; living-vs-legal artefact split is explicit; 18-1 includes anti-pattern #25 (reportlab/WeasyPrint coexistence boundary preservation) as an AST source-inspection assertion. |
| Anti-pattern guard rails | A− | 18-1 enumerates a **32-row anti-pattern fence** in §4.6 (per dev report); explicit fences for #25 (reportlab/WeasyPrint boundary), #26 (deterministic WeasyPrint output workaround), #28 (GET 302-only no proxy), #29 (POST `run_in_executor` wrap), #30 (GET no company-scope public route), #31 (build-outputs in .gitignore). Strong improvement vs E14/15/16 anti-pattern fence quality. |

### E18 risks flagged for the dev pass (carry-forward from IR-v3, with status update)

- **R-018-1 (public-route ingress allow-list path-traversal probe):** 18-0 dev pass implemented `apps/client/app/(public)/trust/page.tsx`; 18-1 dev pass implemented `GET /api/v1/trust/artefacts/{slug}` with AST assertion. **18-1 reviewer must verify path-traversal probe (`/trust/../api/v1/...`) is included in the boundary tests.**
- **R-018-2 (sub-processor <30-day-future override path ADR):** Still unaddressed; relevant for 18-2 which is `backlog`. Sally pre-pass should resolve inline.
- **R-018-3 (signed URL vs public-read S3 for legal artefacts):** **18-1 chose 1h-TTL signed URL per AC-4.** Sally pre-pass should confirm or push back. Reviewer should note this as a deviation in §6.
- **R-018-4 (WeasyPrint dependency proliferation, Dependabot/inj-01 gate):** 🛑 **First concrete material harm.** 18-1 shipped 3 dep introductions unscanned. IR-v4 elevates `inj-01` to **mandatory pre-18-2 dispatch** (was "recommended" in IR-v3).
- **R-018-5 (BG-language legal artefacts):** **18-1 chose EN-only legal PDFs per AC-9.** Sally pre-pass should confirm legal counsel position or scope BG variants. Reviewer should note this as a deviation in §6.

### Epic 18 in-flight checkpoint (closed-loop verification)

| Story | Status | Code-review verdict | Action |
|---|---|---|---|
| 18-0 | `done` (AP17-C1 #5) | None on file | **Retroactive `bmad-code-review` re-pass needed** (sequenced after 18-1 closes per v19). |
| 18-1 | `review` (correct two-gate) | None yet (pass-1 pending) | **Forward `bmad-code-review` pass-1 dispatch — single most important sequencing intervention right now.** |
| 18-2 | `backlog` | N/A | **Gated on 18-1 Approve + Sally pre-pass for sub-processor email template UX.** |

### Epic 17 retrospective verification (NEW since IR-v3)

`epic-17-retro-2026-05-03.md` — 8 PATTERNS / 9 ANTI-PATTERNS / 13 ACTION ITEMS:

- 6 critical anti-patterns: AP17-C1 (two-gate), C2 (un-skip RED-phase), C3 (k6 baseline), C4 (TEA review), C5 (sprint-status discrepancy), C6 (testarch-nfr).
- 5 high anti-patterns: AP17-H1..H5.
- 2 medium anti-patterns: AP17-M1..M2.
- project-context.md updated 2026-05-03 with 20 new E17 entries.

**The retrospective discharged the IR-v3 recommendation to codify AP17-C1 in project-context.** What remains is **dispatch of the 13 action items** — those are operator-level backlog items, not IR-v4 deliverables. IR-v4 records that AP17-C1 retrospective-codification is complete; what is NOT complete is the **retroactive `bmad-code-review` re-pass dispatch for the 5-story fan-out (17-0/1/2/3 + 18-0)** that the action item describes.

---

## Step 6 — Final Assessment

### Summary

Epic 18 is **specification-ready at the PRD-amendment + architecture + epic-AC level** for the remainder of the epic (18-1 close + 18-2 dispatch). The blockers are unchanged in nature from IR-v3:

1. **CR-7 (E18 UX gap, 6 surfaces, now 6 working days aged)** — operational blocker for `bmad-create-story` on 18-2; M2 hard deadline blast radius widening.
2. **CR-9 evolved (AP17-C1 quintuple-fire)** — process-integrity blocker; 5 stories `done` without code-review Approve verdict; epic-17-retro codified the anti-pattern but did not dispatch the retroactive re-passes.
3. **CR-1 + CR-2 + CR-4 (planning-artifact hygiene + PRD source unification)** — discoverability and source-of-truth blockers; 9 working days aged with zero movement.
4. **CR-6 (Epic 13 carry-forwards, 13th deferral, FIRST CONCRETE HARM)** — `inj-01` (Dependabot) deferral has produced its first concrete material harm via 18-1 shipping 3 dep introductions unscanned. IR-v3 R-018-4 named the prediction; IR-v4 records the realisation.

### Critical Issues Requiring Immediate Action (ranked, IR-v4)

1. **🛑 CRITICAL — Forward dispatch: `bmad-code-review` for Story 18-1 (pass-1).** Single in-flight item with clean dispatch path; 54/54 Python ATDD tests reportedly green; 11 ACs to verify; 32-row anti-pattern fence to enforce; verdict MUST be set explicitly Approve / Changes-Requested / Reject and recorded in story file §7. **Do NOT transition `done` without Approve.** This is the v19 single-most-important sequencing intervention and IR-v4 endorses it. Reviewer brief enumerated in `sprint-change-proposal-2026-05-04.md` §Path forward Action 1 (per-AC verification + Rule 39 `run_in_executor` AST assertion + R-018-3/R-018-4/R-018-5 §6 deviation reconciliation + cross-tenant negative test symmetry + i18n parity).

2. **🛑 CRITICAL — Sally pre-pass for E18 surfaces (CR-7) — aged 6 working days.** 6 surfaces unspecified (public `<AppShell>` variant, compliance posture cards, ISO 27001 roadmap component, artefact-download UI, change-log section, sub-processor email template). **Surface 6 (sub-processor email template) gates 18-2 dispatch directly.** Sally to also resolve R-018-2 (sub-processor <30-day override path), R-018-3 (signed URL vs public-read confirm), R-018-5 (BG/EN legal artefacts confirm or scope) inline. **Output:** append-only ux-spec.md supplements; do not rewrite existing sections; reconcile with 18-0/18-1 inline §4.4 mitigation. **Effort:** ~half-day. **Sequencing:** can run in parallel with Action 1 (different agent context).

3. **🛑 CRITICAL — Retroactive `bmad-code-review` re-pass dispatch for AP17-C1 5-story fan-out (17-0/17-1/17-2/17-3 + 18-0).** Five stories on disk at `development_status=done` with no senior-developer-review Approve verdict on file. Beta-gate / GA-gate audit-trail liability. epic-17-retrospective named this as critical action #1. **Sequencing:** dispatch sequentially after 18-1 closes to avoid reviewer context contention. Order: 17-0 → 17-1 → 17-2 → 17-3 → 18-0 (chronological, mirrors development order). **Effort:** 30–45 min × 5 = 150–225 min total (lower than fresh review since changes are documented). **Critical:** these are *retroactive* Approves that reconcile sprint-status with workflow contract; verdicts go in story file §7.

4. **🛑 HIGH — Dispatch `inj-01-dependabot-configuration` (Epic 13 carry-forward, 13th deferral, MATERIAL HARM REALISED).** Story 18-1 shipped WeasyPrint + markdown-it-py + python-frontmatter dep introductions **unscanned** — IR-v3 R-018-4 prediction now realised. **IR-v4 elevates `inj-01` from "recommended" to "mandatory pre-18-2 dispatch."** Single `.github/dependabot.yml` file, <30 min per Epic 8 retrospective sizing. After `inj-01` lands, retroactively scan the 3 deps and resolve any findings in a follow-up story (likely a `dw-04` or amendment to 18-1).

5. **🛑 HIGH — CR-1 (planning-artifact hygiene) — 9 working days aged.** Move 80+ stale epic files to `planning-artifacts/epics/_archive/`; move PRD/UX backups to `planning-artifacts/_archive/`. 30–60 min hygiene pass. Operator-level decision; not blocking story correctness but a permanent footgun for any agent loading from `planning-artifacts/epics/`.

6. **🛑 HIGH — CR-2 (planning view drift) — 9 working days aged.** Reconcile `epics.md` (9-epic legacy view) vs operational E10–E21 set. Pick Option A (rename + new full-roadmap file) or Option B (regenerate to merge).

7. **🛑 HIGH — CR-4 (PRD source unification) — 9 working days aged.** Merge `prd-amendment-2026-04-25.md` into `PRD.md` and stamp v1.1. Same blocker now applies to E16/E17/E18 stories. **Should land before Epic 19 kickoff to prevent FR12+ surfaces from inheriting the same fragmentation.**

8. **⚠ MEDIUM — CR-8 (E18 design questions) — Sally pre-pass to resolve.** R-018-2 (sub-processor <30-day override path), R-018-3 (signed URL — 18-1 chose 1h-TTL; confirm), R-018-5 (BG/EN legal artefacts — 18-1 chose EN-only; confirm or scope BG).

9. **⚠ LOW — CR-5 (E17 UX gap) — backlog tech debt.** 5 E17 surfaces unspecified. Epic 17 stories shipped via inline ATDD source-inspection. Treat as Sally-pass-eligible if scope permits during E18 Sally pass; not blocking.

### Recommended Next Steps (in dispatch order)

1. **Dispatch `bmad-code-review` for Story 18-1 (pass-1).** Inputs: `implementation-artifacts/18-1-pdf-artefact-pipeline-weasyprint-reuse-living-artefacts-git-managed-legal-artefacts.md`. Reviewer brief per `sprint-change-proposal-2026-05-04.md` §Path forward Action 1 (11 ACs + 32-row anti-pattern fence + R-018-3/4/5 §6 deviations). Verdict MUST be set explicitly. **Gates:** 18-1 review→done; 18-2 transitionable to ready-for-dev.

2. **In parallel, dispatch `bmad-agent-ux-designer` (Sally) for E18 surfaces (CR-7).** Output: append-only ux-spec.md supplements covering 6 named surfaces; resolve R-018-2/3/5 inline. **Gates:** `bmad-create-story` for 18-2 (sub-processor email template UX is Surface 6).

3. **In parallel, dispatch `inj-01-dependabot-configuration`** (Epic 13 carry-forward; <30 min). Closes CR-6 13th-fire AND R-018-4. After landing, retroactively scan WeasyPrint + markdown-it-py + python-frontmatter and create follow-up story for any findings.

4. **After 18-1 closes Approved, dispatch retroactive `bmad-code-review` re-passes for 17-0 → 17-1 → 17-2 → 17-3 → 18-0 sequentially.** Each verdict appended to story file §7. Closes AP17-C1 quintuple-fire audit-trail liability.

5. **Resolve CR-1 + CR-2 + CR-4 in a single 30–60 min planning-hygiene pass** — operator-level decision; recommended before Epic 19 kickoff.

6. **After 18-1 closes Approved + Sally pre-pass lands**, dispatch `[VS] Validate Story` for 18-2 (per Operator BMAD-stream non-negotiable) → `bmad-create-story` for 18-2 → dev pass → `bmad-code-review` pass-1 (NOT review-fix loop). Apply two-gate enforcement explicitly per AP17-C1 mitigation.

7. **Before Epic 18 closes**, run `[ER] Epic Review` (per Operator BMAD-stream; Epic 18 has 3 stories with content + CI + notification interdependencies). Then `bmad-retrospective` for Epic 18 (recommended given AP17-C1 dynamics; not blocking but operationally important).

8. **Before Epic 19 kickoff**, run `[IR] Implementation Readiness` v5 (this same skill) — non-negotiable per Operator BMAD-stream guidance.

### CR Summary Table (consolidated, 2026-05-04)

| CR | Severity | Title | Aged (working days) | Action |
|---|---|---|---|---|
| CR-1 | 🛑 HIGH | Planning-artifact hygiene (80+ stale epic files) | 9 | Hygiene pass; move to `_archive/` |
| CR-2 | 🛑 HIGH | Planning view drift (epics.md 9-epic legacy view) | 9 | Rename or regenerate epics.md |
| CR-3 | ✅ CLOSED | Epic 16 status integrity | — | Closed in IR-v2 |
| CR-4 | 🛑 HIGH | PRD source unification (FR8–FR11 missing from PRD.md) | 9 | Merge amendment into PRD.md before E19 kickoff |
| CR-5 | ⚠ LOW | UX gap for E17 surfaces (5 unspecified) | 6 | Backlog tech debt; Sally-pass eligible if E18 scope permits |
| CR-6 | 🛑 HIGH | Epic 13 carry-forwards (13th deferral, **FIRST CONCRETE HARM**) | 13 epic boundaries | **Mandatory `inj-01` dispatch pre-18-2** (was "recommended" in IR-v3) |
| CR-7 | 🛑 CRITICAL | UX gap for E18 surfaces (6 unspecified) | 6 | Dispatch Sally NOW; gates 18-2 |
| CR-8 | ⚠ MEDIUM | E18 design questions (R-018-2/3/5) | 6 | Resolve in Sally pass |
| CR-9 | 🛑 CRITICAL | AP17-C1 quintuple-fire (17-0/1/2/3 + 18-0 done w/o Approve) | 6 (codified in retro) | 5 sequential retroactive re-passes after 18-1 closes |

---

## HALT determination

**No HALT.** Epic 18 specification is coherent and aligned with PRD-amendment + architecture for both 18-1 (in `review`) and 18-2 (in `backlog`). None of the open issues constitute a "spec is broken or incoherent" condition. The blockers are:

- **Coverage gaps** (CR-7 → 6 E18 UX surfaces) — actionable via single Sally dispatch (~half-day).
- **Process-integrity** (CR-9 → AP17-C1 quintuple-fire) — actionable via 5 sequential `bmad-code-review` retroactive re-passes after 18-1 closes (~150–225 min total).
- **NFR-9 dependency posture** (CR-6 → R-018-4 first concrete harm) — actionable via single `inj-01` dispatch (<30 min).
- **Source-of-truth fragmentation** (CR-1 + CR-2 + CR-4) — actionable via single 30–60 min planning-hygiene pass; operator-level decision; recommended before Epic 19 kickoff.
- **Forward sequencing** (18-1 code-review pass-1) — actionable via single forward `bmad-code-review` dispatch (~30–60 min); the **single highest-priority operative item** in the project right now.

**Operator decision points:**

1. **Dispatch Sally for E18 surfaces (CR-7) NOW** before 18-2 enters `bmad-create-story`, OR explicitly accept the risk that 18-2's sub-processor email template will be created with no UX spec to ground against (worse than 18-0/18-1 because 18-2 includes externally-customer-facing email copy + i18n keys).
2. **On 18-1 `bmad-code-review` pass-1 dispatch:** reviewer MUST explicitly set verdict (Approve / Changes-Requested / Reject) and record in story file §7. **Do NOT allow inline "AP17-C1 two-gate" notes to substitute for verdict closure** (AP17-C1 6th instance would be the single most damaging signal in the project's audit trail; the orchestrator dispatcher must hard-gate on explicit Approve presence).
3. **Retroactive `bmad-code-review` re-passes for 17-0/17-1/17-2/17-3 + 18-0 (CR-9):** dispatch sequentially after 18-1 closes. Without retroactive closure, **5 production-deployed stories carry no senior-developer-review evidence on file**.
4. **Pre-18-2 dispatch of `inj-01-dependabot-configuration` (CR-6 / R-018-4):** mandatory per IR-v4 elevation. Closes the 13th-deferral material-harm marker.
5. **Pre-Epic-19 dispatch of CR-1 + CR-2 + CR-4 hygiene pass:** operator-level decision; non-blocking for 18-1/18-2 but should land before Epic 19 kickoff to prevent FR12+ surfaces from inheriting the same fragmentation.

---

*Generated by `bmad-check-implementation-readiness` skill in BMAD autopilot mode (no operator prompts) on 2026-05-04. This IR-v4 supersedes implementation-readiness-report-2026-05-03.md (IR-v3) for Epic 18 mid-epic readiness; IR-v3's specification findings remain in force where not explicitly updated here. This IR-v4 is **overdue** per Operator BMAD-stream non-negotiable workflow checkpoint "[IR] before next epic" — Epic 17 closed 2026-05-03 and Epic 18 entered without an interceding [IR]; v19 of `bmad-correct-course` (2026-05-04 ~01:56) explicitly defers carry-forward routing to this IR-v4 (its §4 / §5 / §6 / §Out-of-scope items 1, 5, 6).*
