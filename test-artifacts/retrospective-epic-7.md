---
epic: "07"
title: "Epic 7 — Proposal Generation & Document Intelligence: Retrospective"
generated: "2026-04-18"
sprint: "7–8"
stories: 17
points: 55
trace_gate: "FAIL"
nfr_overall: "FAIL→RESOLVED"
ac_coverage: "27.3% confirmed GREEN (44/161 ACs); 100% test existence (161/161 ACs backed by ATDD files)"
atdd_tests_generated: "~1,145"
---

# Retrospective — Epic 7: Proposal Generation & Document Intelligence

**Date:** 2026-04-18
**Epic:** E07 | **Sprint:** 7–8 | **Points:** 55 | **Stories:** 17 (S07.01–S07.17)
**Facilitator:** BMad Retrospective Agent (PB-RETRO-001 / PB-DECIDE-005 active)

---

## Executive Summary

Epic 7 delivers EU Solicit's **flagship capability**: an AI-powered proposal workspace with Tiptap rich-text editing, section-by-section SSE streaming from the AI Gateway, requirement checklists, compliance checking, clause risk analysis, scoring simulation, pricing recommendations, win theme extraction, content blocks library, PDF/DOCX export, and version history with diffs and rollback. It is the largest and most architecturally complex epic delivered (17 stories, 161 ACs), building on the AI Gateway (E04) and Opportunity Discovery (E06) foundations.

**Headline Results:**
- TRACE_GATE: **FAIL** — first gate failure in E04–E07 sequence. Test *existence* is 100% (all 161 ACs have ATDD test files), but only 4 of 16 testable stories reached confirmed GREEN automation (27.3% AC coverage).
- NFR: **FAIL → RESOLVED** — two P0 blockers (NFR4: raw SSE error exposure; NFR12: missing audit trail on `/generate`) were detected, surfaced immediately, and remediated via S07.17 (P0 hardening story) and commit `2d41fcf`.
- ATDD: **~1,145 tests generated** across 17 stories (100% checklist coverage for 2nd consecutive epic).
- S07.17 (Security/Audit Hardening) successfully injected and completed as a blocking story — the security hardening story pattern works.
- TEA Test Review: S07.11 scored 95/100.

**Important context:** The TRACE_GATE FAIL reflects mid-implementation readiness, not test strategy failure. 12 of 16 testable stories are in TDD RED phase with all tests written but implementation pending GREEN. This is expected for a 17-story epic in an active sprint. The gate captures real readiness: P0 optimistic locking (R-001, Score 9), PDF/DOCX export (S07.10), and E2E SSE streaming (S07.13) have 0% confirmed passing coverage.

---

## 1. What Was Delivered

| Story | Title | Type | Status | ATDD Phase | Tests |
|-------|-------|------|--------|------------|-------|
| S07.01 | Proposal + Version DB Schema Migrations | Backend | ✅ Done | GREEN (exempt) | 47 |
| S07.02 | Proposal CRUD API | Backend | ✅ Done | ✅ GREEN | 66 |
| S07.03 | Proposal Versioning API | Backend | 🔴 In-dev | 🔴 RED | 32 |
| S07.04 | Proposal Content Save API (Auto-save + Full Save) | Backend | 🔴 In-dev | 🔴 RED | 24 |
| S07.05 | AI Draft Generation Backend SSE Integration | Backend | ✅ Done | ✅ GREEN | 112 |
| S07.06 | Requirement Checklist Agent Integration | Backend | ✅ Done | ✅ GREEN | 14+ |
| S07.07 | Compliance & Risk Scoring Agent Integrations | Backend | 🔴 In-dev | 🔴 RED | 29 |
| S07.08 | Pricing Assistant & Win Theme Agent Integrations | Backend | 🔴 In-dev | 🔴 RED | 23 |
| S07.09 | Content Blocks CRUD & Search API | Backend | 🔴 In-dev | 🔴 RED | 27 |
| S07.10 | Document Export API (PDF + DOCX) | Backend | 🔴 In-dev | 🔴 RED | 26 |
| S07.11 | Proposal Workspace Page Layout & Navigation | Frontend | ✅ Done | 🔴 RED* | 223 |
| S07.12 | Tiptap Rich Text Editor (Section-Based Editing) | Frontend | ⚠️ Review | 🔴 RED | 146 |
| S07.13 | AI Draft Generation Panel (SSE Streaming) | Frontend | ⚠️ Review | 🔴 RED | 43 |
| S07.14 | Requirement Checklist & Compliance Panels | Frontend | ⚠️ Review | 🔴 RED | 24 |
| S07.15 | Scoring Simulator, Pricing & Win Themes Panels | Frontend | ⚠️ Review | ✅ GREEN | 87 |
| S07.16 | Version History, Content Blocks Library & Export Dialog | Frontend | ⚠️ Review | 🔴 RED | 151 |
| S07.17 | Proposal Generate Security Audit & Hardening | Backend | ⚠️ Review | 🔴 RED† | 39 |
| **TOTAL** | | | **5 done, 7 review, 5 in-dev** | **4 GREEN** | **~1,145** |

*S07.11 has 223 tests (159 structural + 64 behaviour); RED due to spec contradictions (breakpoint mismatch, schema drift), not missing tests.
†S07.17 ATDD checklist is RED (39 tests, 27 skipped); inline commit `2d41fcf` resolved NFR4+NFR12; formal refactored test suite pending GREEN confirmation.

---

## 2. Quality Gate Results

### TRACE_GATE: FAIL ❌

| Criterion | Required | Actual | Status |
|-----------|----------|--------|--------|
| P0 coverage (passing) | 100% | ~47% | ❌ NOT MET |
| P1 coverage (passing) | ≥80% | ~33% | ❌ NOT MET |
| Overall passing coverage | ≥80% | 27.3% | ❌ NOT MET |
| R-001 (Score 9 — data loss via optimistic locking failure) | REQUIRED mitigated | Impl exists / tests RED | 🚫 BLOCKS GATE |
| Test existence coverage | 100% | 100% | ✅ MET |

**By TDD Phase:**

| Phase | Stories | ACs | Tests |
|-------|---------|-----|-------|
| ✅ GREEN (confirmed passing) | 4 (S07.02, S07.05, S07.06, S07.15) | 44 | 338 |
| 🔴 RED (failing/skipped) | 12 (S07.03, S07.04, S07.07–S07.14, S07.16, S07.17) | 117 | 807 |
| ⚪ EXEMPT (infra) | 1 (S07.01) | 0 | 47 |
| **Total** | **17** | **161** | **~1,145** |

**P0 Coverage:**

| P0 Area | Epic Test IDs | Story | Status |
|---------|---------------|-------|--------|
| Proposal CRUD & RLS | E07-P0-001–P0-005 | S07.02 | ✅ GREEN |
| Optimistic locking (R-001) | E07-P0-003, P0-004 | S07.04 | 🔴 Impl exists / tests RED |
| AI Draft SSE backend | E07-P0-006, P0-007 | S07.05 | ✅ GREEN (+ NFR fix) |
| AI Draft SSE E2E | E07-P0-012 | S07.13 | 🔴 RED |
| Content blocks RLS | E07-P0-005 | S07.09 | 🔴 RED |
| PDF/DOCX Export | E07-P0-010 | S07.10 | 🔴 RED |
| SSE error sanitization (NFR4) | — | S07.17 | 🔴 RED (ATDD phase) / Inline resolved |

### NFR Assessment: FAIL → RESOLVED ⚠️→✅

| NFR Category | Initial Status | Final Status |
|--------------|---------------|--------------|
| Performance (NFR2, NFR5) | CONCERNS | CONCERNS — background task + threadpool mitigations exist; no k6 baseline |
| Security (NFR4) | ❌ FAIL (raw SSE error exposure) | ✅ RESOLVED — commit `2d41fcf` via S07.17 |
| Reliability (NFR1) | ✅ PASS | ✅ PASS — optimistic locking, reset_stuck task |
| Maintainability / Audit (NFR12) | ❌ FAIL (audit trail missing on /generate) | ✅ RESOLVED — commit `2d41fcf` via S07.17 |

**Critical NFR Observations:**
- NFR4+NFR12 were both deferred in S07.05 senior dev review, detected in NFR assessment, and remediated via an injected hardening story (S07.17). The security/NFR hardening story injection pattern **worked correctly**.
- No load testing baseline exists — this is the **5th consecutive epic** without k6 baseline data.
- Prompt injection sanitization (E07-R-005) for content blocks (S07.09) still relies entirely on AI Gateway — no explicit sanitization layer.

---

## 3. What Went Well

### [PATTERN] Security Hardening Story (S07.17) — Rapid NFR FAIL Response
`IMPACT: standards_update` `SEVERITY: high`

When the NFR assessment identified FAIL on two blockers (NFR4: raw error exposure; NFR12: missing audit trail), a dedicated P0/BLOCKER story (S07.17) was injected and implemented within the same sprint. The security audit artifact (`security-audit-proposal-generate.md`) documented the exact gaps, the story ACs addressed each gap precisely, and commit `2d41fcf` resolved both blockers. This is the correct pattern: **security/NFR failures detected late must be remediated via dedicated hardening stories, not deferred**.

### [PATTERN] 100% ATDD Checklist Coverage — 2nd Consecutive Epic
`IMPACT: standards_update` `SEVERITY: medium`

All 17 stories (including the injected S07.17) had ATDD checklists in RED phase. E04 had 0/10, E05 had 7/12, E06 achieved 14/14. Epic 7 maintains 100% at 17/17, now including the harder cases: a security hardening story and multiple complex multi-panel frontend stories. The discipline is now stable.

### [PATTERN] Content-Hash Optimistic Locking Prevents Data Loss (R-001)
`IMPACT: standards_update` `SEVERITY: high`

S07.04 implements `SELECT FOR UPDATE` with content-hash ETag comparison for concurrent proposal edits. When the client-sent hash mismatches the DB hash, the API returns 409 Conflict with a structured payload (current content + last modifier), enabling the frontend conflict resolution dialog (S07.12). This is the correct and complete pattern for collaborative document editing — both the conflict detection (backend) and reconciliation UI (frontend) must exist to close R-001.

### [PATTERN] Vitest Source-Inspection ATDD for Complex Frontend Components
`IMPACT: standards_update` `SEVERITY: medium`

The S07.11/S07.12/S07.15/S07.16 frontend ATDD checklists use Vitest static source-inspection (asserting file imports, component signatures, and i18n key presence via regex/AST scanning) rather than runtime rendering. This enables Red-phase test writing before components exist, avoids `act()` and hydration complexity for structural assertions, and produces 100+ tests per story that are stable in CI. S07.15 (87 tests, all GREEN) demonstrates the pattern at scale.

### [PATTERN] ThreadPoolExecutor for CPU-Bound Export Operations
`IMPACT: standards_update` `SEVERITY: medium`

S07.10 (Document Export) correctly uses `asyncio.get_event_loop().run_in_executor(None, render_fn)` (or `ThreadPoolExecutor`) to offload PDF/DOCX rendering — preventing event loop blocking during WeasyPrint/python-docx CPU-intensive operations. This is the mandatory pattern for any FastAPI endpoint invoking CPU-bound rendering libraries.

### [PATTERN] Reset-Stuck Background Task for Long-Running Async Operations
`IMPACT: standards_update` `SEVERITY: medium`

S07.05 introduces `reset_stuck_proposals_task` (Celery Beat) to mark proposals stuck in `GENERATING` status as `FAILED` after a configurable TTL. This self-healing pattern prevents orphaned UI spinners and enables user retry. It is the correct pattern for any operation that can silently hang: crawler runs (E05), generation tasks, export jobs.

### [PATTERN] TEA Test Review Provides Architecture Validation (S07.11: 95/100)
`IMPACT: standards_update` `SEVERITY: medium`

The S07.11 test review (95/100) caught two architectural contradictions before they could silently ship: (1) `useBreakpoint` threshold 1280px vs AC5 requirement of 1024px; (2) frontend `ProposalResponse` interface missing `current_version_number` and `generation_status` fields. These schema/spec mismatches are invisible at implementation time and only surface when tests are reviewed against ACs. TEA test review is not optional — it catches class-of-bug that ATDD doesn't.

---

## 4. What Needs Improvement

### [ANTI-PATTERN] TRACE_GATE FAIL — First Failure in 4-Epic Sequence
`[ACTION]` `IMPACT: standards_update` `SEVERITY: high`

E04, E05, and E06 all produced TRACE_GATE PASS on first run. E07 is the first FAIL. The cause is structural: 12 of 16 testable stories remain in TDD RED phase with implementation incomplete. Test existence is 100% — the discipline of writing tests first is working — but the gate failure exposes that 17-story epics cannot realistically achieve full GREEN confirmation within a single sprint. 

→ **Action:** For epics with >12 stories, the orchestrator should gate TRACE_GATE separately for completed stories vs. in-progress stories. Confirmed GREEN coverage on *done* stories should be the primary gate metric. The current single threshold (80% overall) penalises epic scope rather than test quality.

### [ANTI-PATTERN] R-001 (Data Loss via Optimistic Locking) Tests Never Confirmed GREEN
`[ACTION]` `IMPACT: story_injection` `SEVERITY: critical`

The highest-risk scenario in E07 (R-001, Score 9 — concurrent edits overwriting proposal versions) has implementation (S07.04 `SELECT FOR UPDATE`) but 0% confirmed passing tests. The ATDD checklist has 24 tests (all skipped). This is the most dangerous confirmed gap: the data-loss prevention mechanism is implemented but not verified under test conditions.

→ **Action:** Inject Sprint 8 story: "S07.04 Optimistic Locking: Green-phase verification — run 24 ATDD tests, confirm `SELECT FOR UPDATE` + 409 Conflict + concurrent race test passes. Block S07.12 (conflict dialog) merge on this gate."

### [ANTI-PATTERN] ATDD Checklist Internal Test Count Mismatch (S07.17: 27 stated vs 39 actual)
`[ACTION]` `IMPACT: standards_update` `SEVERITY: medium`

S07.17's ATDD checklist summary section states "27 tests" but the level-by-level breakdown counts 39. The traceability matrix used 39 (the authoritative count) but the discrepancy erodes trust in the checklist as source of truth. This is the second time count mismatches have appeared (S07.11 automation summary also had a 48-test reconciliation in Rev 4).

→ **Action:** ATDD checklists must include an auto-computed total from the level-by-level breakdown section. Checklist template: add `<!-- TOTAL: ${sum} -->` computed comment in the summary block. CI lint step should validate summary total equals computed sum.

### [ANTI-PATTERN] Frontend ProposalResponse Schema Drift — Backend/Frontend Contract Unenforced
`[ACTION]` `IMPACT: standards_update` `SEVERITY: high`

S07.11 TEA review found `current_version_number` and `generation_status` absent from the frontend `ProposalResponse` interface. These fields are present in the backend Pydantic model but the frontend type was not regenerated or manually synced. This is a cross-layer contract gap that no test caught until TEA review.

→ **Action:** All service response types used in frontend must be generated from backend Pydantic schemas (via `openapi-typescript` or equivalent codegen). Manual duplication of backend types in frontend is prohibited. Add codegen step to CI that fails on backend/frontend type divergence for shared response models.

### [ANTI-PATTERN] Breakpoint Spec Contradiction Between AC and Hook Implementation (S07.11)
`[ACTION]` `IMPACT: standards_update` `SEVERITY: medium`

AC5 for S07.11 specifies responsive collapse at 1024px, but the `useBreakpoint` implementation uses 1280px. Neither ATDD tests nor senior dev review caught this — only TEA review did. Story ACs must be the single source of truth for implementation constants.

→ **Action:** AC numeric constants (px thresholds, timeouts, limits, rates) must be explicitly listed in implementation task descriptions, not just in ACs. Story template: add "Implementation constants" subsection under each AC that includes a numeric threshold.

### [ANTI-PATTERN] P0 Areas with 0% Confirmed Passing — S07.10, S07.13, S07.04
`[ACTION]` `IMPACT: story_injection` `SEVERITY: high`

Three P0 areas (PDF/DOCX Export: S07.10; E2E SSE Streaming: S07.13; Optimistic Locking Tests: S07.04) entered Sprint 8 with 0% confirmed passing test coverage. These are the highest-risk areas (export timeout under load, streaming disconnect recovery, data loss prevention) and represent the largest risk carry-forward into Epic 8.

→ **Action:** Define minimum P0 confirmed-GREEN coverage threshold before epic can close: each P0 scenario must have at least one GREEN test before TRACE_GATE is run. Stories with P0 ACs not confirmed GREEN are `in-review`, not `done`.

### [ANTI-PATTERN] k6 Performance Baseline Absent — 5th Consecutive Epic Carry-Forward
`[ACTION]` `IMPACT: story_injection` `SEVERITY: critical`

E03, E05, E06, and now E07 NFR reports all note absent k6 baselines. Epic 7 adds ThreadPoolExecutor for PDF export (event-loop blocking mitigated) and SSE concurrency (semaphore-bounded) but zero wall-clock data exists. PDF generation at p95 under concurrent load is unvalidated. SSE TTFB for AI draft generation (multiple AI Gateway hops) is unvalidated.

→ **Action:** k6 baseline is a **hard Sprint 8 deliverable**. Gate Epic 8 TRACE_GATE on: (1) SSE `/proposals/{id}/generate` TTFB p95 < 500ms at 20 concurrent users, (2) PDF export p95 < 5s at 10 concurrent users. No further deferrals.

### [ANTI-PATTERN] Prompt Injection Sanitization for Content Blocks Deferred (E07-R-005)
`[ACTION]` `IMPACT: story_injection` `SEVERITY: high`

E07-R-005 (prompt injection via content block body) is documented but relies entirely on AI Gateway sanitization. S07.09 (Content Blocks CRUD) has no explicit input sanitization layer — content block bodies containing `{{system: ...}}` or SSRF payloads are persisted and forwarded unmodified. The AI Gateway is a necessary but not sufficient control.

→ **Action:** Inject Sprint 8 security story: "Content block body sanitization layer — strip prompt injection patterns before persistence and before forwarding to AI Gateway. Include ATDD tests with known injection payloads."

### [ANTI-PATTERN] Dependabot Not Configured — 5th Consecutive Epic Carry-Forward
`[ACTION]` `IMPACT: story_injection` `SEVERITY: critical`

Recommended in E01, E02, E05, E06, and now E07 closes without this configured. Epic 7 adds `weasyprint`, `python-docx`, `tiptap/*`, `recharts`, `@dnd-kit/sortable` — all unscanned. This is a Beta milestone blocker by any reasonable definition.

→ **Action:** Dependabot configuration is **blocked from any further deferral**. Block Epic 8 kickoff on this being configured. Single infrastructure story, no points assigned.

### [ANTI-PATTERN] S07.17 ATDD Phase Ambiguity — Security Audit vs Formal ATDD Checklist
`[ACTION]` `IMPACT: standards_update` `SEVERITY: medium`

Two artifacts in conflict: `security-audit-proposal-generate.md` claims 41 passing tests for S07.17; the ATDD checklist (`bmad-testarch-atdd` generated same day) classifies it as TDD RED with 27 skipped tests. Both artifacts are valid but describe different states (inline commit baseline vs. fully refactored form). The traceability matrix correctly uses ATDD checklist as authoritative but the ambiguity slows review.

→ **Action:** When a security audit and an ATDD checklist describe the same story, the ATDD checklist is canonical. Security audit artifacts must explicitly state which tests overlap with the ATDD checklist and which are additional. Add "ATDD Overlap" section to security audit template.

---

## 5. TEA Test Review Summary

| Story | ATDD Checklist | Tests Generated | TDD Phase | TEA Review | Senior Dev Review |
|-------|---------------|-----------------|-----------|-----------|-------------------|
| S07.01 | ✅ | 47 pytest | GREEN (exempt) | — | ✅ |
| S07.02 | ✅ | 66 pytest | ✅ GREEN | — | ✅ (automation expanded) |
| S07.03 | ✅ | 32 pytest | 🔴 RED | — | — |
| S07.04 | ✅ | 24 pytest | 🔴 RED | — | — |
| S07.05 | ✅ | 112 pytest | ✅ GREEN | — | ✅ (NFR deferred→S07.17) |
| S07.06 | ✅ | 14+ pytest | ✅ GREEN | — | ✅ |
| S07.07 | ✅ | 29 pytest | 🔴 RED | — | — |
| S07.08 | ✅ | 23 pytest | 🔴 RED | — | — |
| S07.09 | ✅ | 27 pytest | 🔴 RED | — | — |
| S07.10 | ✅ | 26 pytest | 🔴 RED | — | — |
| S07.11 | ✅ | 223 Vitest/E2E | 🔴 RED* | **95/100** | — |
| S07.12 | ✅ | 146 Vitest | 🔴 RED | — | ⚠️ Review |
| S07.13 | ✅ | 43 Vitest | 🔴 RED | — | ⚠️ Review |
| S07.14 | ✅ | 24 Vitest | 🔴 RED | — | ⚠️ Review |
| S07.15 | ✅ | 87 Vitest | ✅ GREEN | — | ⚠️ Review |
| S07.16 | ✅ | 151 Vitest/E2E | 🔴 RED | — | ⚠️ Review |
| S07.17 | ✅ | 39 pytest | 🔴 RED† | — | ⚠️ Review |
| **TOTAL** | **17/17** | **~1,145** | **4 GREEN** | **1 scored** | |

*RED due to spec contradictions (1280px vs 1024px, schema drift), not test quality issues.
†Inline fixes committed; formal refactored suite pending GREEN.

**TEA Coverage Gap:** Only 1 of 17 stories has a scored TEA review. Backend stories (S07.03–S07.10) and frontend stories S07.12–S07.16 have no TEA review scores. This is the **5th consecutive epic** with incomplete TEA review coverage.

`[ACTION]` `IMPACT: config_tuning` `SEVERITY: high`
→ TEA review must be scheduled as a blocking phase gate for all stories that reach `in-review` status. Stories cannot transition from `review` to `done` without a TEA review score ≥ 80/100.

---

## 6. Cross-Epic Carry-Forward Status

| Item | First Reported | Epics Deferred | Status |
|------|---------------|----------------|--------|
| Dependabot configuration | E01 | E01→E02→E05→E06→E07 | **CRITICAL — 5th deferral** |
| Prometheus /metrics bootstrap | E01 | E01→E02→E05→E06→E07 | **HIGH — 5th deferral** |
| k6 performance baseline | E03 | E03→E05→E06→E07 | **CRITICAL — 5th deferral** |
| OBS-001 circuit breaker 4xx miscounting | E05 | E05→E06→E07 | HIGH — E04+E05 files still unfixed |
| `_RUN_ID_REGISTRY` Celery prefork bug | E05 | E05→E06→E07 | **CRITICAL — prod silent failure** |
| Non-atomic DB transactions in crawlers | E05 | E05→E06→E07 | HIGH — prod crash risk |
| S06.05 (Opportunity Detail API) carry-forward | E06 | E06→E07 | Status unknown — needs verification |
| E06-P0-010 (Playwright E2E tier journey) | E06 | E06→E07 | Status unknown — needs verification |
| Content block prompt injection sanitization | E07 | First reported | → Sprint 8 story |

---

## 7. Structured Findings for Orchestrator (PB-RETRO-001 / PB-DECIDE-005)

### [PATTERN] Security Hardening Story Injection Is the Correct NFR FAIL Response
`IMPACT: standards_update` `SEVERITY: high`
→ When NFR assessment returns FAIL on a security or maintainability criterion, immediately inject a P0/BLOCKER hardening story targeting only those criteria. S07.17 resolved two FAIL blockers within one sprint. Template: "S{epic}.{N+1} — [Service] Security/Audit Hardening: [NFR category]" with ACs derived directly from NFR finding items.

### [PATTERN] Content-Hash Optimistic Locking as Standard for Collaborative Document Editing
`IMPACT: standards_update` `SEVERITY: high`
→ All endpoints that PATCH or PUT document content must implement: (1) `content_hash` column (SHA-256 of content JSON), (2) client sends current hash in request, (3) `SELECT FOR UPDATE` + hash comparison in transaction, (4) 409 Conflict with structured body on mismatch. Frontend must implement conflict resolution dialog. This is a two-story pattern: backend (content save API) + frontend (conflict dialog in editor).

### [PATTERN] Vitest Source-Inspection ATDD Scales to Complex Multi-Panel Frontend Components
`IMPACT: standards_update` `SEVERITY: medium`
→ For frontend stories with 15+ ACs and multiple sub-components (panels, modals, toolbars), Vitest source-inspection ATDD (regex/import assertions, no runtime rendering) is the correct approach. Runtime rendering tests (RTL/JSDOM) should be reserved for user-interaction flows. S07.15 (87 tests, all GREEN) is the reference implementation.

### [PATTERN] ThreadPoolExecutor Is Mandatory for CPU-Bound Library Calls in FastAPI
`IMPACT: standards_update` `SEVERITY: medium`
→ Any FastAPI endpoint that calls a CPU-bound library (WeasyPrint, python-docx, Pillow, lxml) MUST use `await loop.run_in_executor(executor, fn, *args)` with a dedicated `ThreadPoolExecutor`. Event loop blocking from synchronous libraries is invisible in unit tests and only manifests under load. ATDD checklists for export/render stories must include a structural assertion: "function uses `run_in_executor`".

### [PATTERN] Reset-Stuck Background Task for All Long-Running Async State Machines
`IMPACT: standards_update` `SEVERITY: medium`
→ Any resource that has an in-progress state (proposal generation, document scanning, crawler run, export job) must have a corresponding Celery Beat cleanup task that marks stale entries FAILED after a configurable TTL. S07.05's `reset_stuck_proposals_task` is the reference implementation. Add this as a required AC to all stories that introduce a `status` field with an in-progress variant.

### [ACTION] IMPACT: standards_update SEVERITY: critical
→ For epics with >12 stories, split TRACE_GATE into two checkpoints: (A) GREEN gate for *completed* stories (must pass before story moves to `done`), (B) overall coverage gate for epic close. This prevents the paradox of 100% test existence + FAIL gate.

### [ACTION] IMPACT: story_injection SEVERITY: critical
→ Inject Sprint 8 story: "S07.04 Optimistic Locking: GREEN-phase verification — confirm 24 ATDD tests pass (SELECT FOR UPDATE + 409 Conflict + concurrent race test). Block S07.12 conflict dialog merge on this gate."

### [ACTION] IMPACT: standards_update SEVERITY: critical
→ Frontend response types MUST be generated from backend Pydantic schemas via codegen (e.g., `openapi-typescript` from OpenAPI spec). Manual type duplication between backend and frontend is prohibited. Add codegen validation step to CI.

### [ACTION] IMPACT: story_injection SEVERITY: critical
→ k6 baseline is a hard Sprint 8 deliverable. Block Epic 8 kickoff on: SSE `/proposals/{id}/generate` TTFB p95 < 500ms at 20 concurrent, PDF export p95 < 5s at 10 concurrent.

### [ACTION] IMPACT: story_injection SEVERITY: critical
→ Dependabot configuration: block Epic 8 kickoff. No further deferrals. Assign as infrastructure story with no sprint points.

### [ACTION] IMPACT: story_injection SEVERITY: high
→ Inject Sprint 8 security story: "Content block body sanitization layer — strip prompt injection patterns before persistence and before forwarding to AI Gateway."

### [ACTION] IMPACT: config_tuning SEVERITY: high
→ TEA review score ≥ 80/100 required before any story transitions from `review` to `done`. Configure as orchestrator gate condition.

### [ACTION] IMPACT: standards_update SEVERITY: high
→ AC numeric constants (breakpoint thresholds, timeouts, file size limits, retry counts) must appear in story implementation task descriptions, not only in the AC text. Story template: add "Implementation constants" subsection.

### [ACTION] IMPACT: standards_update SEVERITY: high
→ Prometheus /metrics bootstrap: block Epic 8 NFR gate on /metrics being available on client-api. No further deferrals.

### [ACTION] IMPACT: standards_update SEVERITY: medium
→ ATDD checklists must auto-compute test totals. Template: add computed total comment `<!-- TOTAL: N -->` in summary block. CI lint step validates summary total matches level-by-level breakdown.

### [ACTION] IMPACT: standards_update SEVERITY: medium
→ Security audit artifacts must include "ATDD Overlap" section declaring which tests overlap with the ATDD checklist and which are additional. Prevents dual-counting and phase ambiguity.

### [ACTION] IMPACT: standards_update SEVERITY: medium
→ P0 minimum GREEN coverage gate: at least 1 confirmed-GREEN test per P0 scenario before TRACE_GATE runs. P0 ACs with 0 GREEN tests = story cannot be `done`.

### [ANTI-PATTERN] S07.04 Optimistic Locking ATDD Tests Never Confirmed GREEN — R-001 Unverified
`[ACTION]` `IMPACT: story_injection` `SEVERITY: critical`
→ Highest-risk scenario (Score 9 data loss) has implementation but 0% confirmed passing tests. Unacceptable. Sprint 8 P0 task.

### [ANTI-PATTERN] 5th Consecutive Epic Without k6 Performance Baseline
`[ACTION]` `IMPACT: story_injection` `SEVERITY: critical`
→ Hard block on Epic 8 kickoff.

### [ANTI-PATTERN] 5th Consecutive Epic Without Dependabot Configuration
`[ACTION]` `IMPACT: story_injection` `SEVERITY: critical`
→ Hard block on Epic 8 kickoff.

### [ANTI-PATTERN] Frontend Type Definitions Manually Duplicated from Backend Models (Schema Drift)
`[ACTION]` `IMPACT: standards_update` `SEVERITY: high`
→ `ProposalResponse` drift caught in TEA review — would have shipped silently without it.

### [ANTI-PATTERN] TEA Review Coverage at 1/17 Stories — 5th Consecutive Epic Under-Execution
`[ACTION]` `IMPACT: config_tuning` `SEVERITY: high`
→ Configure TEA review as blocking gate. Non-negotiable.

### [ANTI-PATTERN] OBS-001 Circuit Breaker 4xx Miscounting — 3rd Consecutive Epic Carry-Forward
`[ACTION]` `IMPACT: standards_update` `SEVERITY: high`
→ Still present in E04 and E05 files. Must be fixed before any circuit breaker code is reused in E08+.

### [ANTI-PATTERN] `_RUN_ID_REGISTRY` Celery Prefork Bug — 3rd Consecutive Epic Carry-Forward
`[ACTION]` `IMPACT: standards_update` `SEVERITY: critical`
→ Silent production failure. Must be fixed before data pipeline goes to production.

---

## 8. Retrospective Metrics

| Metric | E05 | E06 | E07 |
|--------|-----|-----|-----|
| Stories delivered (done) | 12/12 (100%) | 12/14 (86%) | 5/17 (29%)* |
| TRACE_GATE | PASS | PASS | **FAIL** |
| NFR overall | CONCERNS | CONCERNS | **FAIL→RESOLVED** |
| ATDD checklist coverage | 7/12 (58%) | 14/14 (100%) | 17/17 (100%) |
| Tests generated | ~400 | ~1,209 | ~1,145 |
| Confirmed GREEN stories | — | 12/14 | 4/17 |
| TEA reviews scored | 0 | 0 | 1/17 |
| New patterns | 5 | 9 | 6 |
| New anti-patterns | 7 | 9 | 9 |
| Actions triggered | 7 | 10 | 13 |
| Carry-forward critical items | 2 | 3 | 5 |

*Stories in `review` (7) are in-flight, not abandoned — this reflects mid-sprint state.

---

## 9. Recommendations for Epic 8

1. **Sprint 8 P0 tasks (before Epic 8 kickoff):**
   - S07.04 optimistic locking GREEN verification (R-001 mitigation confirmed)
   - k6 performance baseline for `/proposals/{id}/generate` and export endpoints
   - Dependabot configuration (hard block — no further deferrals)
   - Prometheus /metrics bootstrap on client-api
   - OBS-001 circuit breaker 4xx fix (E04+E05 files)
   - Content block prompt injection sanitization story

2. **Process gates for Epic 8:**
   - TEA review ≥ 80/100 required before `review → done` transition
   - Frontend type codegen from backend OpenAPI schema — CI validation
   - ATDD checklist auto-computed totals — CI lint
   - P0 minimum 1 GREEN test per P0 scenario before TRACE_GATE

3. **Patterns to reinforce in Epic 8 story templates:**
   - Security hardening story injection on NFR FAIL
   - Content-hash optimistic locking for any collaborative editing
   - ThreadPoolExecutor assertion in ATDD for CPU-bound endpoints
   - Reset-stuck background task as required AC for in-progress state machines
   - Negative ATDD assertions for streaming (no EventSource, no Axios buffering)
   - SSE generator lifecycle checklist (from E06 anti-pattern)
