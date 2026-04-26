# Retrospective: Epic 10 — Collaboration, Tasks & Approvals

**Date:** 2026-04-22
**Epic:** 10
**Status:** RED Phase Complete / NFR Critical Failure

## Executive Summary
Epic 10 successfully established a comprehensive TDD foundation with 100% traceability of all 14 Acceptance Criteria across 1,520+ planned tests. However, the epic suffers from a critical non-functional breakdown: the systematic omission of epic-level test design and the continued neglect of TEA review scoring (7th consecutive epic). Most severely, a multi-tenant security gap remains open due to the delayed implementation of proposal-level RBAC middleware.

## TEA Quality Signal
- **TEA Reviews Scored:** 0 / 16 stories.
- **TEA Coverage Gap:** Systematic under-execution continues. No independent signal for test isolation, maintainability, or performance exists for Epic 10.
- **TEA Verdict:** FAIL (Gate Rule 5: "TEA review is a per-story gate, not an epic-level activity").

## Critical Findings & Action Plan

### [ANTI-PATTERN] Missing Epic-Level Test Design
- **Finding:** Epic 10 was skipped in the test-design pipeline. No `test-design-epic-10.md` exists.
- **Impact:** Formal NFR verification (Performance, Scalability, Reliability) is undefined. No load testing strategy for concurrent collaborator editing.
- **SEVERITY:** HIGH
- **IMPACT:** standards_update
- **[ACTION]** Retroactively author `test-design-epic-10.md` before GREEN phase completion.

### [ANTI-PATTERN] Security Isolation Gap (NFR4)
- **Finding:** Proposal workspace endpoints lack per-proposal RBAC. Delayed S10.02 allows UUID-based existence leakage/access between unrelated companies.
- **Impact:** Violates fundamental multi-tenant security architecture.
- **SEVERITY:** CRITICAL
- **IMPACT:** story_injection
- **[ACTION]** S10.02 (Proposal-Level RBAC Middleware) is now a blocking gate for all Epic 10 feature merges.

### [PATTERN] 100% RED-Phase Traceability
- **Finding:** All 14 Epic ACs are mapped to specific failing tests across all 16 story ATDD checklists.
- **Impact:** Robust TDD foundation. Prevents "done" status without behavioral verification.
- **SEVERITY:** low
- **[PATTERN]** Continue the "Parametrized Permission Matrix" and "Cross-Tenant Negative Test" patterns in all future epics.

### [PATTERN] i18n and Audit Consistency
- **Finding:** 100% of backend stories include audit assertions; 100% of frontend stories include EN/BG parity checks.
- **Impact:** High compliance and localization reliability.
- **SEVERITY:** low
- **[PATTERN]** Reinforce these as mandatory structural tests for all stories.

## Learnings for Subsequent Epics
1. **The "N+1" Debt:** Deferring TEA reviews and NFR design is now a compounding project-level risk. The Orchestrator must enforce the TEA gate at the story level.
2. **Middleware First:** Foundational security middleware (like S10.02) must be implemented *before* the features that depend on it, or the features must remain disabled.
3. **Traceability Rigor:** The BMad TEA Agent's `bmad-testarch-trace` workflow is highly effective at identifying AC coverage gaps during the RED phase.

## Actionable Markers for Orchestrator
- `[ACTION]` Create `test-design-epic-10.md` retroactively. IMPACT: standards_update | SEVERITY: high
- `[ACTION]` Prioritize S10.02 as blocking dependency. IMPACT: story_injection | SEVERITY: critical
- `[ACTION]` Implement `review -> done` gate on TEA score >= 80. IMPACT: config_tuning | SEVERITY: high
- `[PATTERN]` Parametrized Permission Matrix for RBAC.
- `[PATTERN]` EN/BG Parity Assertions for i18n.
- `[ANTI-PATTERN]` Skipping epic-level test design.
- `[ANTI-PATTERN]` Cumulative TEA review under-execution.
