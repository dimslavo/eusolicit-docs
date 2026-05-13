# Epic 22 Final Retrospective — On-Prem Launch (ADR-010 Pivot)

**Date:** 2026-05-13
**Epic:** E22 — On-Prem Launch
**Facilitator:** Gemini Agent

## 1. Executive Summary

This document serves as the **final retrospective** for Epic 22, which managed the project's architectural pivot to a single-host, on-premise deployment model as defined in ADR-010. This retrospective was run after the epic's stories were moved to `review` status in `sprint-status.yaml`.

It builds upon the initial **kickoff retrospective** (`epic-22-retro-2026-05-12.md`) which was conducted at the start of the epic. The key finding of this final review is that while the epic's goals were achieved at a code level, significant process gaps identified in the kickoff retro persisted through execution, particularly regarding the failure to update story file statuses and formally execute `bmad-dev-story` cycles.

**Overall Assessment:** Epic 22 successfully delivered the technical scaffolding for the on-prem launch, but failed to follow the established BMAD process, creating a "shadow" epic where progress was tracked in `sprint-status.yaml` comments but not in the story artifacts themselves.

## 2. Context: The Kickoff Retrospective

The epic began with a comprehensive kickoff retrospective that framed the work. Its key findings were:
-   **Architectural Pivot:** The move from AWS Managed Services to a single Docker host (`www1`) was a sound decision, trading a high-availability promise for a more realistic "best-effort" posture with a defensible RTO/RPO.
-   **Nature of Stories:** E22 stories are primarily **operator playbooks and automation scaffolding**, not application code. The "live gate" (manual drills) is more important than the "code gate".
-   **Process Gaps Identified Pre-Execution:**
    -   All 6 stories were seeded by a PM agent but none had formal `bmad-dev-story` passes.
    -   A heavy backlog of manual operator actions was identified.
    -   TEA reviews were noted as being skipped for the 17th consecutive epic.
    -   `sprint-status.yaml` was known to be out of sync with story file content (Anti-Pattern `AP18-C2`).

## 3. Epic Execution Analysis: Plan vs. Actual

The execution of Epic 22 largely followed the path laid out in the kickoff retro, but with the process failures becoming reality.

### What Was Accomplished (per `sprint-status.yaml`)

The `last_updated` notes in `sprint-status.yaml` provide a narrative of the epic's execution:
-   **Stories `onprem-01` through `onprem-06` were moved to `review` status.** This was done via a bulk update, not through individual `bmad-dev-story` completions.
-   The code for these stories, which includes Ansible playbooks, backup/restore scripts, and monitoring configurations, was implemented.
-   The critical process gaps identified in the kickoff retro (TEA reviews, `2b-dev-story-verify` phase) were **not** addressed during this epic.

### The Core Process Failure: Stale Story Artifacts

The most significant finding of this final retrospective is the confirmation of the process gap warned about in the kickoff: **the individual story files remain in their initial `ready-for-dev` state.**

-   No `Dev Notes` from `bmad-dev-story` sessions were added.
-   No `Review` sections with feedback were added.
-   No `Test Results` or evidence of the manual "live gate" drills were recorded in the story files.

This confirms that Anti-Pattern `AP18-C2` (story file status is stale) and `AP22-C1` (PM seed passes are not formal completion) were not resolved. The team relied on unstructured tracking in `sprint-status.yaml` comments instead of the formal artifact-based workflow.

## 4. Synthesized Learnings & Action Items

The patterns, anti-patterns, and action items from the kickoff retrospective are still the most relevant findings. This final retro serves to confirm their impact and re-emphasize their importance.

### Confirmed Patterns (From Kickoff Retro)

-   **[E22-P01]** Architectural pivots as clean ADR rewrites + sprint-change-proposal injection.
-   **[E22-P02]** Single shared `_lib.sh` for sibling infra scripts.
-   **[E22-P03]** Redis `maxmemory-policy: noeviction` is required for non-cache state.
-   **[E22-P04]** Runbook coverage CI lint gate extends to host-layer alerts.
-   **[E22-P05]** Deploy pipeline CI-gate via `workflow_run` trigger.

### Confirmed Anti-Patterns (From Kickoff Retro)

-   **[E22-AP01]** PM sprint-change-proposal "dev passes" are not formal story completions. **(This was the core failure mode of Epic 22's execution).**
-   **[E22-AP02]** On-prem stories have a wider "code gate vs live gate" gap; ACs must be split. **(The lack of "live gate" evidence in stories confirms this).**
-   **[E22-AP03]** Pre-existing resource pressure must be resolved as a story entry criterion.
-   **[E22-AP04]** TEA review backlog continues to grow. **(Confirmed, now 18+ epics).**

### Final Action Items

The action items from the kickoff retro remain the critical path for process improvement. They are re-iterated here with urgency.

| # | Action | Owner | Priority | Status |
|---|--------|-------|----------|--------|
| A1 | Embed TEA review as mandatory AC in `bmad-dev-story` template. | Orchestrator | **CRITICAL** | **NOT DONE** |
| A2 | Implement `2b-dev-story-verify` phase to sync story file status. | Orchestrator | **CRITICAL** | **NOT DONE** |
| A3 | Execute Docker storage-root migration on www1. | Operator | **HIGH** | PENDING |
| A4 | Execute formal `bmad-dev-story` sessions for all stories. | Dev Agent | **CRITICAL** | **SKIPPED** |
| A5 | Execute all manual drills (restore, restart, rebuild) and record results. | Operator | **HIGH** | PENDING |

## 5. Conclusion & Next Steps

Epic 22 was a "successful failure." It produced the necessary code to pivot to an on-prem architecture but did so by circumventing the core BMAD development and review process. The risk carried forward is that the implemented code has not been formally reviewed, tested against ATDD, or had its "live gate" criteria (the manual drills) formally verified and documented in the story artifacts.

**The next epic MUST NOT proceed until the critical action items (A1, A2, A4) are addressed.** Failure to do so will compound the process debt and invalidate the quality assurance principles of the BMAD framework.

1.  **Update `project-context.md`:** The patterns and anti-patterns from the kickoff retro have been verified and should be enforced by the Orchestrator.
2.  **IMMEDIATE-ACTION:** A dedicated session must be run to execute the `bmad-dev-story` and `bmad-code-review` cycles for all 6 `onprem-*` stories to bring the artifacts in line with `sprint-status.yaml`.
3.  **BLOCKER:** Do not start Epic 23 until the manual operator drills are complete and the results are documented in the respective story files.
