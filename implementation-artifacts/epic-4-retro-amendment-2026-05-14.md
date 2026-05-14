---
epic: 4
epicTitle: 'AI Gateway Service — SirmaAI Refactor Amendment (S04.20–S04.31)'
date: '2026-05-14'
stories_total: 12
stories_done: 11
stories_in_progress: 1
points: 38
overall_verdict: 'PARTIAL_SUCCESS_WITH_BLOCKER'
previous_retro: 'epic-4-retro-2026-04-23.md (original AI Gateway scope S04.01–S04.10)'
partial_retrospective: true
pending_stories:
  - '4-20-service-rename-and-flag-scaffold (in-progress)'
nfr_report: 'eusolicit-docs/test-artifacts/nfr-report-epic-04.md (2026-05-14, PASS_WITH_CONCERNS, 21/29)'
traceability_matrix: 'eusolicit-docs/test-artifacts/traceability-matrix.md (Epic 22 — Epic 4 trace was carried in retrospective-epic-4.md)'
---

# Retrospective — Epic 4 SirmaAI Amendment: ai-gateway → sirmaai-gateway Refactor

**Date:** 2026-05-14
**Epic phase:** E04 SirmaAI Amendment — Stories S04.20–S04.31 (12 stories, ~38 points)
**Scope source:** `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md`; sprint-change-proposal-2026-05-12-sirmaai.md §3.2
**Verdict:** PARTIAL SUCCESS — 11 of 12 stories `done`; S04.20 (service rename + flag scaffold) remains `in-progress`. Amendment scope is **release-ready behind `SIRMAAI_GATEWAY_ENABLED` flag** per NFR assessment, but flag flip is gated on closing S04.20 + 4 HIGH NFR items.

> This is a **partial retrospective** covering only the SirmaAI amendment phase. The original Epic 4 scope (S04.01–S04.10, AI Gateway against KraftData) was retroed on 2026-04-23 (`epic-4-retro-2026-04-23.md`). Per the orchestrator playbook, S04.20's pending status means this retro should be re-opened once S04.20 closes.

---

## Pivot Context

On 2026-05-12 the platform pivoted from KraftData (`stage.sirma.ai`) to the SirmaAI substrate (`agenticsai.endigitalx.com`). The amendment refactored `ai-gateway` → `sirmaai-gateway` in-place behind a feature flag, plus introduced four net-new operational primitives:

1. **Per-tenant Project mapping cache** (S04.21) — Redis-backed 5-min TTL with key-invalidation event.
2. **Fernet-encrypted per-Project API key vault** (S04.22) — 90-day rotation with double-validation overlap.
3. **Standard Webhooks receiver** (S04.25) — HMAC SHA-256 + 7-day Redis idempotency + DLQ.
4. **5-minute run-state reconciler** (S04.26) — authoritative truth per architecture amendment §4.4.

The implementation now lives in `eusolicit-app/services/sirmaai-gateway/` with **61 test files** (45 unit + 15 integration); legacy `ai-gateway/` co-exists behind the flag and is scheduled for deletion one sprint after prod flag flip.

---

## E04 Original-Scope Retro Follow-Through

The 2026-04-23 retrospective committed to four actions before closing the amendment phase. Outcome:

| # | Commitment from 2026-04-23 retro | Status | Notes |
|---|----------------------------------|--------|-------|
| 1 | Run `bmad-testarch-nfr` for Epic 4 | ✅ Done | `nfr-report-epic-04.md` written 2026-05-14: PASS_WITH_CONCERNS, 21/29. |
| 2 | Patch S04.10 mock to assert `Authorization: Bearer` header | ⚠️ Carry-forward | Original `ai-gateway` mock still does not assert auth header format. New `sirmaai-gateway` tests explicitly assert per-Project bearer override end-to-end. Original-service patch deferred (legacy code path slated for deletion). |
| 3 | Add timing assertion to E04-P1-009 (SSE latency) | ❌ Not addressed | Carried forward to E21 k6 baseline (S21.1). |
| 4 | Pre-scale story for circuit-breaker → Redis migration | ❌ Not addressed | Documented in NFR report as backlog item; no story file exists. |
| 5 | Inject security-hardening stories (Prometheus, USER, dependabot, etc.) | ❌ Not addressed | 6th consecutive epic — escalates to terminal carry-forward (see Anti-Pattern E04A-AP01). |

```
[ANTI-PATTERN] Security & observability carry-forward debt — 6th consecutive epic
IMPACT: story_injection
SEVERITY: critical
```

The Prometheus `/metrics` endpoint and Dockerfile USER directive remain unstarted from E01 (6 epic deferrals). The NFR report flags this as a HIGH-priority blocker for prod flag flip.

---

## What Went Well

### [PATTERN] Cross-Tenant Negative Tests Embedded By Default
`IMPACT: standards_update | SEVERITY: medium`

Every amendment story from S04.23 onward shipped with an explicit cross-tenant negative test. The project-memory rule ("All cross-tenant endpoints must have negative tests") is no longer a review-time discovery — it is structurally embedded in AC templates:

- **S04.23 AC 10:** `test_resolver_does_not_leak_across_companies`
- **S04.24 AC 9:** `test_get_status_does_not_leak_across_companies`
- **S04.25 AC 7:** Webhook signed with subscription B's secret but POSTed to A's endpoint must fail HMAC.

The resolver's composite key `(eusolicit_company_id, sirmaai_project_id)` makes cross-tenant leakage structurally impossible. **This is the single strongest tenant-isolation discipline yet demonstrated.**

### [PATTERN] Composite Circuit-Breaker Keys for Tenant Isolation
`IMPACT: standards_update | SEVERITY: medium`

S04.24 introduces `circuit_key = f"sirmaai_run:{logical_name}:{sirmaai_project_id}"` — the caller (router endpoint) composes the key from business semantics, the resilience layer does not auto-derive it from the client library. One outage at one (Project × logical agent) trips that one circuit; sibling tenants stay green.

S04.28 (rate-limit sync) initially used `circuit_key=exc.agent_name` (bare) and was caught in code review — would have collapsed all SirmaAI Project outages into a single circuit. The amendment's discipline is: **circuit-breaker keys must be caller-composed, not library-auto-derived.**

### [PATTERN] Idempotency-By-SQL-Guard Across Three Independent Callers
`IMPACT: standards_update | SEVERITY: high`

`WorkflowRunRepository.converge()` (introduced in S04.24, reused in S04.25, S04.26) holds the canonical SQL guard:

```sql
UPDATE gateway.workflow_runs
   SET status = :new_status, ...
 WHERE eusolicit_run_id = :run_id
   AND status IN ('pending','running')   -- terminal rows are never re-written
```

All three callers (foreground async poll, inbound webhook, 5-min reconciler) write through this single method. The amendment's architecture invariant ("webhooks are latency optimisation; reconciler is authoritative truth" — amendment §4.4) is correct-by-design because the guard is uniform.

### [PATTERN] `SecretStr` End-to-End with Structured-Log Redaction Tests
`IMPACT: standards_update | SEVERITY: high`

After S04.22 Round-1 review flagged `ProjectMapping.api_key_plaintext: str` as a leak risk, the amendment converged on a project-wide pattern:

1. **Type:** Pydantic `SecretStr` from DB read to HTTP-call site.
2. **Unwrap point:** `.get_secret_value()` called exactly once, inside the HTTP helper, never reassigned to a `str` binding.
3. **Log assertion test:** `structlog.testing.capture_logs()` + explicit assertion that no captured log entry contains the plaintext.

Same pattern applied to webhook HMAC secrets (S04.25 AC 1 step 4: 32-byte bytes value, never `.decode()`d, never logged at any level — even DEBUG).

### [PATTERN] Standard Webhooks Spec Compliance with Multi-Value Signature Tolerance
`IMPACT: standards_update | SEVERITY: medium`

S04.25 receiver is fully compliant with the Standard Webhooks spec:
- Raw bytes read via `await request.body()` BEFORE JSON parsing.
- Canonical HMAC input: `f"{webhook_id}.{webhook_timestamp}.".encode() + raw_body` (length-prefixed bytes to prevent collision attacks).
- `hmac.compare_digest()` comparison (project rule 48; explicitly commented in source).
- Multi-value signature tolerance during rotation overlap: `webhook-signature: v1,<sig1> v1,<sig2>` split on space then comma.
- 7-day Redis SETNX idempotency cache on `webhook-id`.
- DLQ table `gateway.webhook_dlq` for poison events.

No code-review findings on the signature implementation itself — discipline was correct from Round 1.

### [PATTERN] Reconciler-As-Authoritative-Truth Eventual Consistency
`IMPACT: standards_update | SEVERITY: high`

The 5-min Celery Beat reconciler scans `gateway.workflow_runs` partial index of non-terminal rows and converges status via `GET /jobs/{jobId}/status`. Webhooks short-circuit the latency to seconds; if a webhook is lost, the reconciler catches it within 5 minutes (RPO ≤ 5 min).

This is the **textbook eventual-consistency pattern** and should be the gold standard for E26 (Agent-Driven Ingestion), E28 (Webhook & Run-State Reconciliation), and any future event-driven epic.

### [PATTERN] Public-Ingress Path Narrowing as a Breaking Cleanup
`IMPACT: standards_update | SEVERITY: medium`

S04.29 nginx vhost reduces the gateway's public surface to a single literal path: `/webhooks/sirmaai`. The pre-amendment `/ai/` location prefix (which exposed all gateway endpoints publicly, contradicting the ClusterIP-only architecture) is deleted in the same change. The runbook + cert SAN expansion are co-located.

**Pattern lesson:** When fixing a security mis-architecture, delete the dangerous surface in the same commit as the new narrow surface — don't leave both available.

---

## What Could Be Improved

### [ANTI-PATTERN] S04.26 Reconciler Stuck in 4+ Code-Review Rounds with Persistent HIGH Findings
`IMPACT: prompt_adjustment | SEVERITY: critical`

```
[ACTION] Block S04.20 amendment closure and prod flag flip on S04.26 final review pass; require single-shot final patch with no working-tree-uncommitted findings
IMPACT: prompt_adjustment
SEVERITY: critical
```

S04.26 is the most complex story in the amendment — it composes three upstream surfaces (S04.24 async-run + S04.25 webhooks + reconciler itself) across two async boundaries (HTTP poll + DB query). It has been through **four code-review rounds** with persistent HIGH findings:

| Round | HIGH Findings | Status |
|-------|--------------|--------|
| 1 | `alembic.ini` rollback not reverted; missing 4 P0/P1 integration tests; `mark_polled` not called on permanent-error branches; `map_status` KeyError uncaught | Changes Requested |
| 2 | Same three HIGH findings unresolved (dev did not address verdict before re-review) | Changes Requested (unchanged) |
| 3 | `CircuitOpenError` silently ACKs — directly violates AC 3 + docstring; breaks NFR-25 convergence during outages | Changes Requested |
| 4 | CircuitOpenError defect still found "in working tree" but uncommitted | **Changes Requested** |

**Root cause:** the dev re-submitted for review with fixes in the working tree but not in the commit, and reviewers caught it. The story file (sprint-status) still shows `done` based on an earlier verdict line; the trailing 2026-05-14 Round-4 comment marks **Changes Requested**.

This violates the **single-shot final-patch discipline**: when a story enters re-review after Round 1, all findings must be committed before re-submission. "I'll address that in the next round" is not acceptable for HIGH findings — it inflates review surface and delays closure.

### [ANTI-PATTERN] S04.20 Stuck `in-progress` While Downstream Stories Closed
`IMPACT: config_tuning | SEVERITY: high`

```
[ACTION] Audit S04.20 (service rename + flag scaffold) — it cannot legitimately be in-progress while S04.21–S04.31 are all done, since rename is the structural prerequisite
IMPACT: config_tuning
SEVERITY: high
```

`sprint-status.yaml` shows `4-20-service-rename-and-flag-scaffold: in-progress` but all 11 downstream stories (S04.21–S04.31) are `done`. The rename is by definition the structural prerequisite — every downstream story uses the new `sirmaai_gateway` import path. Two possibilities:

1. **Sprint-status drift:** The rename was actually completed inline during S04.30 (package rename) but the row was never flipped. Most likely cause given the package-rename memory note.
2. **Genuine outstanding work:** Some artifact of S04.20 (docker-compose alias, network alias, helm chart) remains unfinished and is being silently bypassed by `SIRMAAI_GATEWAY_ENABLED=false` default.

Either way, the row is wrong. Audit and flip to `done` (or escalate to backlog with specifics).

### [ANTI-PATTERN] Multi-Round Reviews Mostly Triggered by Same Three Root Causes
`IMPACT: prompt_adjustment | SEVERITY: high`

```
[ACTION] Update bmad-dev-story prompt to enforce pre-review self-checklist for (a) cross-tenant negative test, (b) SecretStr/log-redaction, (c) idempotency SQL-guard usage
IMPACT: prompt_adjustment
SEVERITY: high
```

7 of 12 amendment stories required 2–4+ review iterations. Cluster analysis of Round-1 findings shows three recurring root causes (counts across the amendment):

| Root cause | Stories affected | Detection mechanism |
|------------|------------------|---------------------|
| Bearer-token / HMAC-secret leak risk (logged via `str(exc)` or `.decode()`) | S04.22, S04.23, S04.24, S04.25 (4 stories) | Code review reading except-blocks |
| Missing cross-tenant negative test or weak isolation assertion | S04.21, S04.22 (caught at Round 1) | Code review reading test files |
| Idempotency SQL guard not used or weakened (e.g., `WHERE status != 'done'` instead of `status IN ('pending','running')`) | S04.26 (multiple rounds) | Code review reading repository methods |

A self-checklist prepended to the dev-story prompt would catch these before review submission. Reviews would then focus on subtler concurrency bugs (the actual added-value of human review), not regression-prevention sweeps.

### [ANTI-PATTERN] NFR Assessment Reactive, Not Proactive
`IMPACT: standards_update | SEVERITY: high`

```
[ACTION] Make bmad-testarch-nfr a mandatory pre-prod-flag-flip gate, executed alongside (not after) the final amendment story
IMPACT: standards_update
SEVERITY: high
```

The `nfr-report-epic-04.md` was generated on 2026-05-14, **after** 11 of 12 amendment stories closed. Five evidence gaps surfaced (k6 baseline, SCA scan, CPU/memory profile, ATDD checklists for S04.20–S04.31, DR drill) that should have been parallel-executed during the amendment, not discovered at the end. The original-scope retro (2026-04-23) already flagged "no NFR assessment" — running it 21 days later for the amendment phase replays the same pattern.

### [ANTI-PATTERN] Zero ATDD Checklists for Amendment Stories — Repeats Original-Scope Anti-Pattern
`IMPACT: standards_update | SEVERITY: high`

```
[ACTION] Generate retroactive ATDD checklists for S04.20–S04.31 OR document the gap explicitly in S04.20 closure; enforce ATDD-as-before-dev-gate from E26 onward
IMPACT: standards_update
SEVERITY: high
```

`ls eusolicit-docs/test-artifacts/atdd-checklist-4-2*` returns nothing. The 2026-04-23 retro explicitly committed to "Enforce ATDD checklist creation as a before-dev gate for every E05+ story" — and yet 12 amendment stories shipped with zero ATDD checklists. Tests exist (61 files) and pass; the discipline gap is documentation of RED→GREEN sequence, not the tests themselves.

### [ANTI-PATTERN] Legacy `ai-gateway/` Folder Has No Deletion Date
`IMPACT: config_tuning | SEVERITY: medium`

```
[ACTION] Add concrete delete date for eusolicit-app/services/ai-gateway/ to sprint-status — one sprint after SIRMAAI_GATEWAY_ENABLED=true on prod
IMPACT: config_tuning
SEVERITY: medium
```

`CLAUDE.md` says "old `ai-gateway` code remains callable for 1 sprint, then deleted." No calendar date is pinned. Without a date the deletion drifts indefinitely; co-existence of two implementations behind a flag is intentional only for the cutover window. Pick a date based on prod flag flip + 1 sprint, add to sprint-status.

---

## Process Learnings

### [PROCESS_CHANGE] Caller-Composed Circuit-Breaker Keys Are Mandatory for Multi-Tenant External Calls

When the external call has a per-tenant identity dimension (Project ID, workspace ID, OAuth installation ID), the circuit key MUST be composed by the caller as `(logical_name, tenant_id)` (or wider tuple if the call has multiple identity dimensions). Bare `logical_name` keys collapse tenant outages and create cross-tenant blast radius. This applies retroactively to E07/E11 outbound calls and prospectively to E17 (CRM via MCP) and E26 (agent-driven ingestion).

### [PROCESS_CHANGE] Idempotency Discipline = One SQL-Guarded `converge()` Method Reused by All Callers

When multiple callers (webhook + reconciler + foreground poll) write to the same row, route every write through one repository method with a `WHERE status IN (<non-terminal>)` guard. Application-layer deduplication (locks, mutex, etc.) is fragile under concurrent access; the SQL guard is correct-by-design. Pattern to be applied in E28 (Webhook & Run-State Reconciliation).

### [PROCESS_CHANGE] `SecretStr` Plus Structured-Log Redaction Test Is the Standard for All Per-Tenant Secrets

Every per-tenant secret (API key, HMAC secret, OAuth token, refresh token, MCP bearer) must be:
1. Typed `SecretStr` at the boundary it leaves the DB row.
2. Unwrapped exactly once via `.get_secret_value()` at the HTTP call site.
3. Validated by a test that captures structured logs across the entire request path and asserts no occurrence of the plaintext.

Reference implementation: `sirmaai-gateway/src/sirmaai_gateway/services/key_vault.py` + `tests/unit/test_key_vault.py`.

### [PROCESS_CHANGE] Standard Webhooks Spec is the Inbound Webhook Template

For any inbound webhook (SirmaAI now; future: SirmaAI-hosted MCP server, third-party CRM webhooks, payment-provider webhooks), the receiver must:
1. Read raw bytes BEFORE JSON parsing.
2. Build canonical HMAC input as `f"{webhook_id}.{webhook_timestamp}.".encode() + raw_body`.
3. Compare with `hmac.compare_digest()`.
4. Tolerate multi-value signatures (space-separated) during rotation overlap.
5. Store idempotency key in Redis SETNX with TTL ≥ provider's retry window.
6. DLQ poison events to a dedicated table with full payload + error_type.

Reference: `sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py`.

### [PROCESS_CHANGE] Reconciler-As-Authoritative-Truth Eventual Consistency Replaces Synchronous Polling

For long-running external operations (SirmaAI agent runs, future: workflow executions, MCP tool invocations), the canonical pattern is:
1. Submit async + persist `workflow_runs` row.
2. Inbound webhook converges row state via `repository.converge()` (latency optimisation).
3. 5-min Celery Beat reconciler scans non-terminal rows and polls source-of-truth API; also calls `repository.converge()`.
4. Foreground `GET /runs/{id}` reads the row.

The reconciler — not the webhook — is authoritative. Webhooks may be lost; the 5-min RPO is acceptable. Reference: `sirmaai_gateway/services/reconcile_runs_task.py`.

### [PROCESS_CHANGE] Public Ingress Cleanups Delete the Dangerous Surface in the Same Commit

When changing a service's public exposure, delete the old broad surface (e.g., `/ai/` location) in the SAME commit that adds the new narrow surface (e.g., literal `/webhooks/sirmaai`). Do not leave the broad surface accessible during cutover — it is the highest-risk attack window of any deploy. Reference: S04.29 nginx vhost diff.

---

## Carry-Forward to Next Phase (E26 / E28 / On-Prem Launch)

### Immediate Blockers (Before Prod Flag Flip)

1. **Close S04.26 reconciler review:** Final patch must commit all Round-4 findings; one-shot re-review. Block E28 dispatch until S04.26 is `done` with verdict APPROVE.
2. **Audit and close S04.20:** Verify rename completed (likely already done via S04.30); flip sprint-status. If genuine outstanding work, enumerate and inject as p0.
3. **k6 baseline for gateway proxy paths:** 4 scenarios (sync run, SSE burst, webhook delivery, admin endpoints). Capture in `test-artifacts/k6-baseline-epic-04.md`.
4. **SCA scan:** `pip-audit -r services/sirmaai-gateway/pyproject.toml` → `test-artifacts/security-audit-gateway.md`.
5. **4 missing failure-mode runbooks:** Redis outage, SirmaAI substrate outage, key-rotation overlap failure, reconciler stall.

### Short-Term (Next Milestone, Pre-On-Prem-Launch)

1. **Prometheus `/metrics` endpoint** — 6th consecutive epic carry-forward; blocks E21 SLO dashboards.
2. **Bound the execution-log task queue** — replace `asyncio.create_task` fire-and-forget with bounded `asyncio.Queue` + 1 consumer.
3. **Retroactive ATDD checklists for S04.20–S04.31** — close the documentation gap before prod rollout.
4. **Concrete delete date for legacy `ai-gateway/` folder.**

### Long-Term (Backlog, Pre-Multi-Replica)

1. **Migrate circuit-breaker state to Redis** — pre-scale work for E21 multi-replica. Gate: depends on Redis HA story (S21.3).
2. **Per-tier load test under quota ceiling.**

---

## Findings Summary (Structured for Orchestrator Parsing)

| ID | Type | Severity | IMPACT | Description |
|----|------|----------|--------|-------------|
| E04A-P01 | PATTERN | medium | standards_update | Cross-tenant negative tests embedded by default in AC templates |
| E04A-P02 | PATTERN | medium | standards_update | Composite circuit-breaker keys `(logical_name, tenant_id)` — caller-composed |
| E04A-P03 | PATTERN | high | standards_update | Idempotency-by-SQL-guard via shared `repository.converge()` |
| E04A-P04 | PATTERN | high | standards_update | `SecretStr` end-to-end + structured-log redaction test |
| E04A-P05 | PATTERN | medium | standards_update | Standard Webhooks spec compliance with multi-value signature tolerance |
| E04A-P06 | PATTERN | high | standards_update | Reconciler-as-authoritative-truth eventual consistency |
| E04A-P07 | PATTERN | medium | standards_update | Public-ingress cleanup deletes dangerous surface in same commit |
| E04A-AP01 | ANTI-PATTERN | critical | story_injection | Security & observability carry-forward — 6th consecutive epic |
| E04A-AP02 | ANTI-PATTERN | critical | prompt_adjustment | S04.26 reconciler 4+ review rounds; uncommitted working-tree fixes |
| E04A-AP03 | ANTI-PATTERN | high | config_tuning | S04.20 stuck `in-progress` while downstream stories closed |
| E04A-AP04 | ANTI-PATTERN | high | prompt_adjustment | Multi-round reviews from same 3 root causes (secret leak, cross-tenant test, idempotency guard) |
| E04A-AP05 | ANTI-PATTERN | high | standards_update | NFR assessment reactive, not proactive — same pattern as 2026-04-23 retro |
| E04A-AP06 | ANTI-PATTERN | high | standards_update | Zero ATDD checklists for amendment stories — repeats original-scope anti-pattern |
| E04A-AP07 | ANTI-PATTERN | medium | config_tuning | Legacy `ai-gateway/` folder has no deletion date |
| E04A-A01 | ACTION | critical | prompt_adjustment | Block S04.20 closure + prod flag flip on S04.26 single-shot final patch |
| E04A-A02 | ACTION | high | config_tuning | Audit S04.20 row; flip done or enumerate outstanding work |
| E04A-A03 | ACTION | high | prompt_adjustment | Update bmad-dev-story prompt with 3-item pre-review self-checklist |
| E04A-A04 | ACTION | high | standards_update | Make bmad-testarch-nfr a mandatory pre-prod-flag-flip gate |
| E04A-A05 | ACTION | high | standards_update | Generate retroactive ATDD checklists for S04.20–S04.31 |
| E04A-A06 | ACTION | medium | config_tuning | Add concrete delete date for legacy ai-gateway/ folder to sprint-status |
| E04A-A07 | ACTION | critical | story_injection | Inject security-hardening stories — no further deferral acceptable (6 epics) |

---

## Readiness Assessment

| Dimension | Status | Notes |
|-----------|--------|-------|
| Testing & Quality | ⚠️ CONCERNS | 61 test files; no fresh coverage measurement attached; no ATDD checklists; S04.26 missing P0/P1 integration tests |
| Deployment | ⚠️ STAGING-READY | `SIRMAAI_GATEWAY_ENABLED=false` default; staging flag flip OK; prod flag flip gated on HIGH actions |
| Stakeholder Acceptance | N/A | Internal refactor; no external stakeholder gate |
| Technical Health | ⚠️ CONCERNS | Strong on security (Fernet, HMAC, SecretStr); concerns on observability (no /metrics) and scalability (in-memory circuit state) |
| Unresolved Blockers | ❌ S04.26 review + S04.20 status | Must close before E28 dispatch |

---

## Sprint Status Update

This retrospective DOES NOT mark any sprint-status fields `done` because:
- `epic-4-retrospective: done` was already set on 2026-04-23 for the original scope.
- `epic-4: in-progress` correctly reflects S04.20 still pending.
- No `epic-4-retrospective-amendment` key exists in sprint-status (would require schema extension — out of scope for this session).

This document is the audit-trail record. The orchestrator should parse the structured findings (E04A-P01..E04A-A07) and route them per the PB-RETRO-001 feedback-loop playbook.

---

## Files Written by This Session

- `eusolicit-docs/implementation-artifacts/epic-4-retro-amendment-2026-05-14.md` (this file)
- `eusolicit-docs/project-context.md` — appended Epic 4 Amendment patterns/anti-patterns section

---

<!-- Powered by BMAD-CORE™ -->
