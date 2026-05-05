---
date: 2026-05-05
project: eusolicit
stepsCompleted: ["step-01", "step-02", "step-03", "step-04", "step-05", "step-06"]
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-spec.md
  - eusolicit-docs/planning-artifacts/epics.md
  - eusolicit-docs/planning-artifacts/epics/ (E01-E21 canonical)
verdict: READY-WITH-CAVEATS
---

# EU Solicit — Implementation Readiness Report (2026-05-05)

## Executive Summary

EU Solicit's planning corpus (PRD v2.0, Architecture v2.0, UX Spec v3.0, 21 canonical epics E01–E21) is **implementation-ready with caveats**. All 44 PRD functional requirements (FR-1..FR-44) trace to at least one epic; all 23 NFRs are addressed across infrastructure (E01), security/auth (E02), AI gateway (E04), platform reliability (E13, E21), and observability work. The planning artifacts are detailed enough for sprint-level story generation — Epics E01–E12 already include sized stories (3–5 pts) with explicit ACs; E13–E21 carry sized stories or coordinator/PE-style outlines.

The HALT triggers checked (missing PRD, missing architecture, FRs with zero epic coverage on must-have features) are not present — implementation is in flight (Epic 14 done, Epic 21 PE.02 done) and the canonical artifacts validate the pipeline.

The only blockers are **housekeeping**: the `epics/` directory contains 50+ legacy `epic-*` draft files alongside the canonical E01–E21, and `epics.md` summary is stale (declares only 9 epics, vs. the canonical 21). Neither blocks orchestrator execution; both are flagged for cleanup.

## Document Inventory

| Document | Path | Lines | Status |
|---|---|---|---|
| PRD | `eusolicit-docs/planning-artifacts/PRD.md` | 398 | Present, complete (v2.0, 2026-04-27) |
| Architecture | `eusolicit-docs/planning-artifacts/architecture.md` | 1070 | Present, complete (v2.0 living doc, last validated 2026-05-04) |
| UX Spec | `eusolicit-docs/planning-artifacts/ux-spec.md` | 705 | Present, complete (Draft v3.0, 2026-05-05) |
| Epics summary | `eusolicit-docs/planning-artifacts/epics.md` | 645 | Present but **STALE** — declares 9 epics; canonical set is E01–E21 |
| Canonical epics | `eusolicit-docs/planning-artifacts/epics/E01..E21*.md` | 21 files | Present, complete |
| Sprint status | `eusolicit-docs/implementation-artifacts/sprint-status.yaml` | n/a | Present, orchestrator-managed |
| Project context | `eusolicit-docs/planning-artifacts/project-context.md` | n/a | Present |

**HOUSEKEEPING ISSUE — Duplicate / Stale Epic Files (NOT a HALT):**
The `epics/` directory contains the canonical `E01..E21*.md` set plus **~52 legacy `epic-*.md` drafts** from prior planning iterations (e.g. `epic-1-onboarding.md`, `epic-01-core-platform.md`, `epic-1-user-onboarding-and-identity.md`, `epic-01-workspace-and-access-management.md`, etc.). These are duplicate/stale clutter and should be removed or moved to an archive subdirectory to prevent accidental ingestion by automated tooling.

**STALE epics.md:** The summary file `epics.md` lists only Epics 1–9 (the original MVP slice) and predates the 2026-04-25 PRD amendment that injected E14–E21. It also predates the renaming/expansion that produced E01–E13. Recommend regenerating from the canonical E01–E21 set or deprecating it in favour of the per-epic files.

## PRD Analysis

**Functional Requirements:** 44 FRs (FR-1..FR-44), grouped into 7 sections — User & Tenant Management (FR-1..9), Billing (FR-10..14), Data Pipeline & Discovery (FR-15..20), AI Analysis (FR-21..25), Proposal Generation (FR-26..33), Compliance & Workflow (FR-34..39), Notifications & Admin (FR-40..44). Post-MVP markers are explicit on FR-31, FR-32, FR-37, FR-38, FR-39, FR-41.

**Non-Functional Requirements:** 23 NFRs (NFR-1..NFR-23) across Performance (1–4), Security (5–9), Scalability (10–13), Reliability (14–17), Accessibility (18–20), Maintainability (21–23). All have measurable thresholds.

**Domain-Specific:** GovTech requirements explicit — ZOP/EU directives, GDPR, EU data residency (eu-central-1), AES-256 + TLS 1.3, immutable audit log, ESPD XML format, WCAG 2.1 AA, AOP/TED integration.

**Quality:** Requirements are well-formed, atomic, and testable. **Minor ambiguity:** FR-44 ("secure REST API") leaves API key vs OAuth2 unspecified — resolved in E12 S12.15 (X-API-Key header). FR-12's trial-without-credit-card behaviour is explicit; FR-14's "one-time add-on" surface is later expanded by E15 per-bid SKU.

## Epic Coverage Matrix (FR → Epic)

| FR | Description (short) | Primary Epic(s) | Secondary |
|---|---|---|---|
| FR-1 | Email/password registration | E02 (S02.02) | E03 (UI) |
| FR-2 | Google OAuth registration | E02 (S02.06) | E03 (UI) |
| FR-3 | Login/logout | E02 (S02.03, S02.04, S02.05) | E03 (UI) |
| FR-4 | Email verification | E02 (S02.02) | E03 |
| FR-5 | Company profile management | E02 (S02.08) | E03 (S03.09 wizard) |
| FR-6 | Invite users to workspace | E02 (S02.09), E10 (S10.01) | E03 |
| FR-7 | Assign roles | E02 (S02.09, S02.10), E10 (S10.02) | — |
| FR-8 | Multi-client workspaces | E14 (S14.00–S14.04) | — |
| FR-9 | User in multiple workspaces | E14 (S14.02, S14.03) | — |
| FR-10 | Stripe subscription | E08 (S08.06) | — |
| FR-11 | Tier-based feature gating | E08 (S08.14), E06 (S06.02, S06.03) | E15 (Pro+) |
| FR-12 | 14-day trial Professional | E08 (S08.03, S08.05) | — |
| FR-13 | Self-service subscription mgmt | E08 (S08.07) | — |
| FR-14 | One-time add-on purchases | E08 (S08.09), E15 (S15.01) | — |
| FR-15 | Auto-ingest AOP/TED | E05 (S05.04, S05.05, S05.06) | — |
| FR-16 | Search + faceted filters | E06 (S06.01, S06.10) | — |
| FR-17 | AI relevance score | E05 (S05.07), E06 (S06.04) | — |
| FR-18 | Listing sorted by relevance/deadline | E06 (S06.04, S06.09) | — |
| FR-19 | Opportunity detail view | E06 (S06.05, S06.11) | — |
| FR-20 | Save / star opportunities | E06 (implied; E09 calendar tracks) | needs explicit story |
| FR-21 | One-page executive summary | E06 (S06.08, S06.13), E07 indirectly | — |
| FR-22 | Requirements checklist extraction | E07 (S07.06) | — |
| FR-23 | High-risk clause flagging | E07 (S07.07 clause-risk) | — |
| FR-24 | Upload PDF/DOCX | E06 (S06.06, S06.12) | — |
| FR-25 | Score simulation | E07 (S07.07 scoring-simulation, S07.15) | — |
| FR-26 | Create proposal | E07 (S07.02, S07.11) | — |
| FR-27 | AI first-draft generation | E07 (S07.05, S07.13) | — |
| FR-28 | Rich text editor | E07 (S07.12) | — |
| FR-29 | Version history | E07 (S07.03, S07.16) | — |
| FR-30 | Restore previous version | E07 (S07.03 rollback, S07.16) | — |
| FR-31 | Section locking (Post-MVP) | E10 (S10.03, S10.12) | — |
| FR-32 | Comments / mentions (Post-MVP) | E10 (S10.04, S10.13) | — |
| FR-33 | Export PDF/DOCX | E07 (S07.10, S07.16) | E12 (S12.09 reuse) |
| FR-34 | Compliance validation | E07 (S07.07 compliance-check, S07.14) | E11 (framework reuse) |
| FR-35 | Compliance frameworks (admin) | E11 (S11.08, S11.09, S11.14) | — |
| FR-36 | ESPD generation | E11 (S11.02, S11.03, S11.13) | E02 (S02.12 ESPD profile schema) |
| FR-37 | Task management (Post-MVP) | E10 (S10.05, S10.14) | — |
| FR-38 | Task dependencies (Post-MVP) | E10 (S10.06) | — |
| FR-39 | Approval workflow (Post-MVP) | E10 (S10.08, S10.09, S10.16) | — |
| FR-40 | Email digests | E09 (S09.03, S09.04, S09.05, S09.06, S09.12) | — |
| FR-41 | Calendar sync (Post-MVP) | E09 (S09.07 iCal, S09.08 Google, S09.09 MS, S09.13) | — |
| FR-42 | Admin portal | E12 (S12.11, S12.12, S12.13, S12.14) | E03 (admin shell) |
| FR-43 | Immutable audit trail | E02 (S02.11), E14 (audit ext) | cross-cutting |
| FR-44 | Enterprise REST API | E12 (S12.15, S12.16) | — |

**Coverage:** 44/44 FRs covered. **Soft gap:** FR-20 ("save/star opportunities") is implied by E09 calendar tracking and listing-page saved-filter UI, but does not have a dedicated story in E06 (S06.01–S06.14). Recommend adding a `SAVED OPPORTUNITIES` story or explicit AC under E06 S06.05 / S06.09.

## NFR Coverage

| NFR | Addressed by |
|---|---|
| NFR-1 (API p95 < 200ms) | E13 S13.03 (k6 baseline), E21 PE.01, architecture §1.2 |
| NFR-2 (AI TTFB < 500ms) | E04 S04.05 (SSE proxy), E13 S13.03, E21 PE.01 |
| NFR-3 (Lighthouse 90+) | UX Spec §10 Performance budgets; not explicitly covered by an epic story — **gap** |
| NFR-4 (100 concurrent users MVP) | E13 S13.03, E21 PE.01 |
| NFR-5 (JWT short-lived + refresh) | E02 S02.03, S02.05 |
| NFR-6 (TLS 1.3 + AES-256) | Architecture §1.2; infra in E21 implicit |
| NFR-7 (data isolation) | E01 S01.03, E02 S02.10, E14 S14.02 |
| NFR-8 (admin IP allowlist + MFA) | E12 S12.11–S12.14, architecture |
| NFR-9 (no critical CVEs) | E13 S13.02 (Dependabot) |
| NFR-10/11 (vertical/horizontal scale) | E01 (stateless services), E21 PE.04 |
| NFR-12 (read replicas) | E21 PE.02 (Multi-AZ + replicas) |
| NFR-13 (10K companies / 1M opps) | E13 S13.03, E21 PE.01, PE.02 (FTS GIN index amendment) |
| NFR-14 (99.5% MVP / 99.9% post) | E21 (entire epic) |
| NFR-15 (zero data loss) | E21 PE.02 (35d PITR) |
| NFR-16 (idempotency) | E08 S08.04 (webhook dedup) |
| NFR-17 (DR within 4h) | E21 PE.02, PE.06 (runbooks) |
| NFR-18 (WCAG 2.1 AA) | UX Spec §11; cross-cutting AC; **no explicit epic story — gap** (orchestrator should mandate per-story AC) |
| NFR-19 (keyboard navigation) | UX Spec §11; cross-cutting |
| NFR-20 (screen reader) | UX Spec §11; cross-cutting |
| NFR-21 (ruff/mypy/eslint/tsc) | E01 S01.08 (CI matrix) |
| NFR-22 (80% unit / 90% E2E coverage) | CLAUDE.md project standard, E13 S13.11 (E2E suite), E13 done-gate AC |
| NFR-23 (structured logs + Prometheus) | E01 S01.06, E13 S13.04 (Prometheus bootstrap), E21 PE.05 |

**NFR Soft Gaps:**
- **NFR-3 Lighthouse score 90+** — UX Spec mentions the target but no epic owns the budget enforcement. Recommend adding to E13 done-gate or E03 acceptance.
- **NFR-18/19/20 WCAG/keyboard/screen-reader** — UX Spec §11 codifies the requirement and the principle (P7) but no epic story explicitly tests WCAG conformance. Recommend a cross-cutting AC ("a11y audit GREEN") or a dedicated story under E12 S12.18 (launch polish).

## UX Alignment

UX Spec defines **6 explicit UX-DR requirements** (per epics.md restatement) plus extensive screen catalogues (§5), interaction patterns (§6), AI states (§7), accessibility (§11). Mapping:

| UX-DR | Spec Section | Epic Coverage |
|---|---|---|
| UX-DR1 (multi-proposal pipeline, deadline-first) | §5.3 Dashboard, §3.1 IA | E07 (workspace), E10 (kanban S10.14), E12 (analytics) |
| UX-DR2 (split-pane editor + inspector panels) | §5.6 Proposal Workspace, §6 | E07 (S07.11–S07.15), E10 (S10.12, S10.13) |
| UX-DR3 (Cmd+K command menu) | §6.1 | **No explicit story** — gap; add to E03 backlog |
| UX-DR4 (real-time AI state machine) | §7 | E04 (SSE), E07 (S07.13 streaming UI) |
| UX-DR5 (per-section locking) | §5.6, Journey D | E10 (S10.03, S10.12) |
| UX-DR6 (one-page exec summary < 90s) | §3.4 Journey A step 8 | E06 (S06.08), E04 (S04.05) |

UI epics (E03, E06, E07, E10, E11, E12, E14, E16, E17, E19) reference UX patterns (`<AppShell>`, `<QueryGuard>`, `useZodForm`, `<FormField>`, design-system component compliance) consistent with UX Spec §6 and CLAUDE.md frontend patterns. Source-inspection ATDD asserting design-system primitives (no native `<select>`/`<dialog>`) is mandated in E13 S13.11, E14 S14.03, E16, E19 — strong alignment.

**UX Gap:** Cmd+K global command palette (UX-DR3) has no owning story. Recommend adding to E03 or E12 polish.

## Epic Quality Review

| Epic | Sprint | Pts | Goal | Stories | ACs | Deps Declared | Quality |
|---|---|---|---|---|---|---|---|
| E01 Infrastructure & Monorepo | 1–2 | 34 | clear | 10, sized 2–5 | per-story explicit | None | Strong |
| E02 Auth & Identity | 1–2 | 34 | clear | 12, sized 2–3 | E01 | strong | Strong |
| E03 Frontend Shell & Design System | 1–2 | 37 | clear | 12, sized 2–5 | None (integrates with E02) | strong | Strong |
| E04 AI Gateway | 3–4 | 34 | clear | 10, sized 2–5 | E01 | strong | Strong |
| E05 Data Pipeline & Ingestion | 3–4 | 34 | clear | 12, sized 2–5 | E01, E04 | strong | Strong |
| E06 Opportunity Discovery | 5–6 | 55 | clear | 14, sized 3–5 | E02, E03, E05 | strong | Strong; soft gap on FR-20 |
| E07 Proposal Generation | 7–8 | 55 | clear | 16, sized 2–3 | E04, E06 | strong | Strong |
| E08 Subscription & Billing | 9–10 | 55 | clear | 14, sized 2–5 | E02, E06 | strong | Strong |
| E09 Notifications, Alerts & Calendar | 9–10 | 55 | clear | 14, sized 2–5 | E01, E05, E06 | strong | Strong |
| E10 Collaboration, Tasks & Approvals | 11–12 | 55 | clear | 16, sized 2–5 | E02, E07 | strong | Strong |
| E11 Grants & Compliance | 11–12 | 55 | clear | 16, sized 2–3 | E04, E06, E07 | strong | Strong |
| E12 Analytics, Reporting & Admin | 13–14 | 55 | clear | 18, sized 2–4 | E05, E06, E07, E08 | strong | Strong; closes MVP |
| E13 Hardening & Drift Recovery | 13 | 34 | clear (carry-forward closure) | 12, sized 1–5 | E01–E12 | strong | Strong; coordinator pattern |
| E14 Multi-Client Workspace | 14–16 | 34 | clear | 5, sized 3–13 | E01, E02, E06 | strong | Strong; S14.03 oversized at 13pt — consider split |
| E15 Per-Bid SKU + Pro+ | 16–17 | 21 | clear | 3, sized 5–8 | E08, E14 | strong | Adequate; small story count, large per-story scope |
| E16 Slack & Teams Notifications | 17–18 | 8 | clear | 1 mega-story (S16.00, 8pt) | E14, E15 | strong | **Adequate but consolidated** — single 8pt story spans service scaffold + UI + alert routing; may benefit from split |
| E17 CRM Integrations | 18–22 | 55 | clear | 4, sized 13–16 | E14, E15, E16 | strong | Adequate; large per-story scope (16pt Salesforce) — 6-week add disclosed |
| E18 Trust Center | 14–15 | 13 | clear (M2 hard deadline) | 3, sized 3–5 | E03, E07, E09 | strong | Strong |
| E19 Outcome Telemetry | 17–18 | 21 | clear | 3, sized 5–8 | E07, E12, E14 | strong | Strong |
| E20 NPS & Reviews (residual) | 18 | 5 | clear | 1, 5pt | E14, E19 | strong | Adequate (deliberately scoped down) |
| E21 Platform Reliability 99.9% SLA | 14–17 (parallel) | 34 | clear | 6 PE.01–PE.06, sized 3–8 | E13 | strong | Strong; PE.02 amendment 2026-05-04 documented |

**Story sizing observations:**
- E14 S14.03 is 13pt — large but justified (Zustand migration + workspace switcher + URL refactor + 2 sprints of ATDD).
- E17 S17.03 (Salesforce) is 16pt — explicitly disclosed as ~6-week add and tagged as third-priority.
- E16 has only one 8pt story bundling backend scaffold + frontend + alert routing; would benefit from split into 2–3 stories for risk isolation but is internally consistent.

**No epic missing critical sections.** Every epic has Goal, Acceptance Criteria, Stories with per-story ACs and (in most cases) Implementation Notes. E13–E21 follow a slightly different shape (Source line, denser implementation notes) reflecting their amendment-driven origin, but they remain implementation-ready.

**Cross-epic dependency graph is consistent.** No circular dependencies detected. The MVP slice (E01–E12) closes at MVP Launch; E13 hardens; E14–E20 expand for the consulting-firm ICP; E21 runs parallel to E14–E17 to gate the 99.9% SLA.

## Critical Issues

**None HALT-worthy.** No missing PRD/architecture/UX docs; no FR with zero coverage; no NFR critically unaddressed.

## Non-Critical Issues (Housekeeping & Soft Gaps)

1. **HOUSEKEEPING — Legacy `epic-*.md` clutter:** `eusolicit-docs/planning-artifacts/epics/` contains ~52 historical drafts alongside the canonical E01–E21. Move to `epics/_archive/` or delete to prevent ingestion confusion.
2. **HOUSEKEEPING — Stale `epics.md`:** Only declares Epics 1–9; predates the E14–E21 amendment. Regenerate from canonical set or deprecate with a redirect note.
3. **SOFT GAP — FR-20 "save/star":** Implied but no dedicated E06 story. Add as AC under S06.05 / S06.09 or a small new story.
4. **SOFT GAP — UX-DR3 Cmd+K command menu:** No owning story; add to E03 backlog or E12 polish.
5. **SOFT GAP — NFR-3 Lighthouse 90+:** No explicit story owns the budget; add to E13 done-gate or E12 S12.17.
6. **SOFT GAP — NFR-18/19/20 (WCAG/keyboard/screen reader):** UX Spec mandates compliance but no story tests it. Add cross-cutting a11y audit AC (E13 done-gate or E12 S12.18).
7. **SOFT — E16 single mega-story:** S16.00 spans service scaffold + UI + alert routing in 8pt. Consider splitting for risk isolation when story is created.
8. **SOFT — `epics.md` `FR Coverage Map` placeholder unfilled:** Line 107 has `{{requirements_coverage_map}}` template literal not interpolated. The fragment at lines 147–174 is a stale, partially-corrupted partial coverage map. Reconcile or regenerate.

## Recommendations

1. Move legacy `epic-*.md` files into `epics/_archive/` and update orchestrator workflow to read only canonical `E[0-9][0-9]*.md` files.
2. Regenerate `epics.md` from the canonical E01–E21 set OR deprecate it (the per-epic files are the source of truth).
3. Add story or AC for FR-20 (save/star) under E06.
4. Add story or AC for UX-DR3 (Cmd+K) under E03 or E12 polish.
5. Add cross-cutting NFR-3 (Lighthouse) and NFR-18/19/20 (a11y) audit AC to E13 done-gate (existing TEA AC pattern is the natural extension point).
6. When creating Story 16-0, split into (a) service scaffold, (b) UI configuration, (c) alert routing — three 3pt stories sum to current 8pt and isolate failure modes.
7. Continue current orchestrator-driven cadence: E14 closed; E15.0/E21.PE.01–PE.02 done; E15.1, E21.PE.03 next per sprint-status.

## Final Verdict

**READY-WITH-CAVEATS.**

All 44 FRs and all 23 NFRs have epic coverage; planning documents are complete, internally consistent, and consumed by the orchestrator's BMAD pipeline (verified by sprint-status.yaml progress through Epic 14 closure and Epic 21 PE.02 done). The caveats are housekeeping (duplicate/stale epic files, stale epics.md summary) and a small set of soft gaps (FR-20 explicit story, UX-DR3, NFR-3, a11y AC) that do not block implementation but should be addressed in the next planning pass or absorbed into existing done-gates.

No HALT condition. Implementation may proceed.
