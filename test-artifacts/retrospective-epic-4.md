---
epic: 4
epicTitle: 'AI Gateway Service'
date: '2026-04-14'
stories: 10
points: 34
sprints: '3-4'
overall_verdict: 'SUCCESS'
---

# Retrospective — Epic 4: AI Gateway Service

**Date:** 2026-04-14
**Epic:** E04 — AI Gateway Service (10 stories, 34 points, Sprints 3–4)
**Verdict:** SUCCESS — All 10 stories DONE. Traceability gate PASS. No integration test failures.

---

## Executive Summary

Epic 4 delivered the complete AI Gateway service — a FastAPI-based internal ClusterIP proxy that decouples all EU Solicit backend services from the KraftData Agentic AI platform. The service provides synchronous and streaming execution endpoints for agents, workflows, and teams; per-agent circuit breaking with exponential-backoff retry; HMAC-SHA256 signed webhook reception and Redis Stream publishing; execution audit logging; concurrency control via asyncio Semaphore; and a comprehensive 10-scenario integration test suite using testcontainers + respx.

This epic is the **critical platform dependency for all AI-assisted features** — ESPD auto-fill (E11), grant eligibility, budget builder, consortium finder, and logframe generator all route through this gateway. Delivery on schedule and to spec is a significant milestone.

**Key Metrics:**

| Metric | Value |
|--------|-------|
| Stories complete | 10/10 (100%) |
| Traceability gate | PASS — P0 100% · P1 96% · P2 73% · P3 100% |
| Overall AC full coverage | 91.5% (54/59) — 100% any coverage |
| High risks mitigated (Score ≥6) | 3/3 (webhook bypass, SSE reliability, Redis publish gap) |
| Code review verdicts | All APPROVE |
| ATDD checklists | 0/10 — **NOT CREATED** (critical gap) |
| NFR assessment | **Not performed** for Epic 4 |
| sprint-status.yaml | `epic-4: in-progress` — not updated to `done` |
| Carry-forward security hardening | Still not addressed (5th consecutive epic) |

---

## What Went Well

### 1. [PATTERN] All Three High-Priority Risks Fully Mitigated
`IMPACT: standards_update | SEVERITY: medium`

The three Score-6 risks from the E04 test design were all addressed in implementation:

- **E04-R-001 (webhook signature bypass):** `hmac.compare_digest()` used for constant-time signature comparison. Unit test `test_constant_time_comparison_used` inspects source for the import. Timing assertion verifies < 1ms difference between valid and off-by-one-byte signatures. **E04-P0-012 FULL.**
- **E04-R-002 (SSE stream proxy edge-case fragility):** Queue-based async generator architecture with separate `upstream_task` and `heartbeat_task`; partial SSE frame buffering via `\n\n` delimiter; `asyncio.CancelledError` propagation cancels both tasks on client disconnect, preventing orphaned connections. Frozen-clock unit tests confirm heartbeat, idle timeout, and total timeout paths. **E04-P1-009/010/011, E04-P2-004/005/006 FULL.**
- **E04-R-003 (Redis publish gap):** `XADD` wrapped in try/except with ERROR-level structured log on failure; `webhook_log.processed=False` populated on failure; response still returns 200 to KraftData to avoid duplicate retries. Compensating monitoring added to backlog. **E04-P0-010 FULL.**

### 2. [PATTERN] Two-Layer Resilience Composition Pattern
`IMPACT: standards_update | SEVERITY: low`

The circuit breaker + retry composition pattern (`circuit.call(lambda: with_retry(factory, agent_name))`) is clean and reusable. Circuit breaker is the outer gate (CLOSED/OPEN/HALF_OPEN state machine with asyncio.Lock for coroutine safety); retry is the inner mechanism (exponential backoff 1s → 2s → 4s, ±25% jitter, max 3 retries). Non-retryable classes (`CircuitOpenError`, 4xx) propagate immediately through both layers without delay. This two-layer pattern should be the standard resilience template for any outbound service call in future epics.

### 3. [PATTERN] YAML-Driven Agent Registry Decouples Business Logic from External IDs
`IMPACT: standards_update | SEVERITY: low`

`config/agents.yaml` maps logical names (`executive-summary`, `budget-builder`) to KraftData UUIDs, agent types, and optional per-agent concurrency limits. Registry hot-reload via `POST /admin/registry/reload` updates entries without service restart; `asyncio.Lock` protects the replacement from concurrent `resolve()` reads. This pattern means KraftData UUID rotation is a config change + hot reload, never a code deploy.

### 4. [PATTERN] testcontainers + respx Integration Test Stack
`IMPACT: standards_update | SEVERITY: low`

The S04.10 integration test architecture — testcontainers PostgreSQL + Redis for the real stack, `respx.MockRouter` for KraftData outbound calls, signed `webhook_payload_factory` fixture for HMAC validation — proves all 10 critical scenarios without real KraftData credentials. The entire suite is CI-safe. This stack (testcontainers + respx + pytest-asyncio) is now the proven template for all backend integration test suites in feature epics.

### 5. [PATTERN] Traceability TRACE_GATE PASS on First Run
`IMPACT: standards_update | SEVERITY: low`

Unlike Epic 3 which required a Run 2 to fix a P0 gap, Epic 4's traceability matrix passed on the first run with 91.5% overall FULL coverage and 100% P0 coverage. The test design was thorough — 55 planned tests across 13 P0, 20 P1, 17 P2, and 5 P3 scenarios. Every AC maps to at least one planned test (100% any coverage).

### 6. [PATTERN] Fire-and-Forget Async DB Writes via asyncio.create_task
`IMPACT: standards_update | SEVERITY: low`

Execution logging and webhook log writes use `asyncio.create_task()` to decouple the audit write from the request/response path. A DB failure is swallowed with an ERROR-level structured log (including `execution_id` for recovery queries) but never propagates a 500 to the caller. This pattern ensures the gateway remains operational even during brief database outages.

### 7. [PATTERN] Structured Admin Observability Endpoints
`IMPACT: standards_update | SEVERITY: low`

`GET /admin/circuits` (per-agent state, failure count, last failure time), `GET /admin/rate-limit` (active/queued/rejected counts), `GET /admin/executions` (filtered execution log), and `GET /admin/registry/reload` form a coherent internal observability surface. These endpoints are ClusterIP-only (no Ingress) and are consumed by ops tooling. Pattern: every service with stateful resilience mechanisms should expose admin introspection endpoints.

---

## What Could Be Improved

### 1. [ANTI-PATTERN] No ATDD Checklists for Any Epic 4 Stories — 1st Occurrence for a Backend Epic
`IMPACT: standards_update | SEVERITY: critical`

```
[ACTION] Create ATDD checklists for all E04 stories before closing the epic
IMPACT: standards_update
SEVERITY: critical
```

The `test-artifacts/` directory contains zero `atdd-checklist-4-*.md` files. This means no formal ATDD RED baseline was tracked for any of the 10 Epic 4 stories. Contrast with E01 (7/10 stories), E02 (12/12), E03 (12/12), and E11 (7/16). The test design is excellent (55 scenarios, TRACE_GATE PASS) but the per-story ATDD cycle was not executed.

**Impact:** The ATDD checklist is the per-story gate between test design and implementation. Without it, there is no documented evidence that tests were written in RED state before implementation, which is the core TDD discipline. Post-hoc checklists are less valuable but should still be created from the implementation artifacts.

**Action:** Run `bmad-testarch-atdd` on each completed E04 story to generate retrospective ATDD checklists, or document the gap explicitly and enforce ATDD checklists in E05 from day one.

### 2. [ANTI-PATTERN] No NFR Assessment for Epic 4
`IMPACT: standards_update | SEVERITY: high`

```
[ACTION] Run NFR assessment for Epic 4 before closing
IMPACT: standards_update
SEVERITY: high
```

The `test-artifacts/nfr-report.md` is the Epic 2 NFR report. No `nfr-report-epic-04.md` was created. Epic 4 has significant NFR concerns that went unassessed:
- **Performance:** No p95 latency measurement for agent proxy calls (SSE latency < 100ms is a story AC with no timing test — see residual concern #1 in traceability matrix).
- **Reliability:** Circuit breaker is in-memory (per-instance) — E04-R-005. Acceptable for single replica but must be documented as a known pre-scale limitation.
- **Security:** Webhook signature validation is implemented correctly, but no formal security assessment was performed.
- **Observability:** `GET /admin/circuits` and `GET /admin/rate-limit` provide operational visibility, but no Prometheus metrics endpoint exists (carry-forward from E01).

**Action:** Run `bmad-testarch-nfr` for Epic 4, particularly focusing on the Redis publish gap (E04-R-003), semaphore starvation under streaming load (E04-R-004), and the circuit breaker per-instance limitation (E04-R-005).

### 3. [ANTI-PATTERN] Sprint Status Not Updated — 4th Consecutive Epic
`IMPACT: config_tuning | SEVERITY: low`

```
[ACTION] Update sprint-status.yaml to set epic-4: done
IMPACT: config_tuning
SEVERITY: low
```

`sprint-status.yaml` shows `epic-4: in-progress` despite all 10 stories having `done` status. This is the fourth consecutive epic (E01, E02, E03, E04) where the epic-level status field was not updated when all stories completed. The E03 retrospective explicitly flagged this and recommended a CI check. That recommendation was not acted on.

**Action:** Immediately update `epic-4: done` in `sprint-status.yaml`. Add a CI check (GitHub Actions or pre-commit hook) that validates: if all `{epic-N}-*: done`, then `epic-N: done`.

### 4. [ANTI-PATTERN] S03.11 TEA Still In-Progress — Carried into 2nd Epic
`IMPACT: story_injection | SEVERITY: medium`

```
[ACTION] Complete S03.11 TEA automation review — blocked across 2 epics
IMPACT: story_injection
SEVERITY: medium
```

`tea_status: 3-11-toast-notification-system: in-progress` in `sprint-status.yaml`. This was flagged as a Must-Do carry-forward in the E03 retrospective ("Complete S03.11 TEA automation review before E04 development starts"). It entered E04 unresolved and exits E04 still unresolved. The E03-P1-014 and E03-P2-017 (toast hover-pause E2E) items remain untested.

### 5. [ANTI-PATTERN] Security and Observability Hardening — 5th Consecutive Epic Without Action
`IMPACT: story_injection | SEVERITY: critical`

```
[ACTION] Inject security hardening stories into Epic 5 backlog — no further deferral acceptable
IMPACT: story_injection
SEVERITY: critical
```

The E03 retrospective listed 8 "Must-Do During E04 Sprint 1" security hardening items. None appear to have been addressed in E04:

1. **Prometheus /metrics endpoint** — first flagged E01 NFR, now 4 epics overdue
2. **Dockerfile USER directive** — first flagged E01 NFR, now 4 epics overdue
3. **Dependabot** — first flagged E01 NFR, now 4 epics overdue
4. **`User.is_active` check in login + refresh** — carry-forward E02
5. **Wrap bcrypt in run_in_executor** — carry-forward E02 (blocks event loop 200–400ms at cost=12)
6. **Fix Redis rate limiter to fail-open** — carry-forward E02 (Redis outage blocks all logins)
7. **JWT audience/issuer claim verification** — carry-forward E02
8. **pnpm audit in CI** — frontend security

Now that E04 introduces the AI Gateway, two additional items compound the list:
9. **Circuit breaker state in Redis for multi-replica** (E04-R-005 pre-scale requirement)
10. **SSE latency benchmark** — no wall-clock measurement for the < 100ms per-event requirement

EU Solicit now has **10 open security/reliability issues** across 4 epics. As a pre-production platform, this is approaching blocker status for staging deployment.

### 6. [ANTI-PATTERN] SSE Latency Acceptance Criterion Not Tested
`IMPACT: prompt_adjustment | SEVERITY: medium`

```
[ACTION] Add wall-clock timing assertion to E04-P1-009 SSE latency test
IMPACT: prompt_adjustment
SEVERITY: medium
```

Traceability matrix residual concern #1: S04.05-AC3 requires "latency < 100ms per event after KraftData sends." E04-P1-009 verifies events are forwarded in order but contains no timing measurement. The concern was flagged and deferred with a recommendation to "add a timing assertion or P3 benchmark." This was not done before closing the epic.

### 7. [ANTI-PATTERN] Config Negative-Path Tests Missing — Environment Variable Validation
`IMPACT: prompt_adjustment | SEVERITY: low`

```
[ACTION] Add pydantic-settings negative-path test for missing required environment variables
IMPACT: prompt_adjustment
SEVERITY: low
```

Traceability matrix residual for S04.01-AC4: No test verifies behavior when `KRAFTDATA_API_KEY` is absent or `DATABASE_URL` is malformed. The gap was documented (E04-P2-NEW-001 recommendation) but not implemented. The missing env var produces an implicit pydantic-settings `ValidationError` — no test confirms the error message names the missing field clearly.

---

## Process Learnings

### 1. [PROCESS_CHANGE] Two-Layer Resilience Pattern Is the Standard for Outbound Service Calls

Circuit breaker (outer) wrapping retry (inner) — both composable, both configurable — is now the proven pattern for any EU Solicit service calling an external API. Future epics (E05 data pipeline calling scrapers, E08 billing calling Stripe, E09 notifications calling email providers) must apply this pattern rather than building ad-hoc retry logic.

### 2. [PROCESS_CHANGE] testcontainers + respx Is the Mandatory Backend Integration Test Stack

The S04.10 architecture is now the template: testcontainers for real database/Redis instances, respx for mocking outbound HTTP calls, `ASGITransport` for in-process FastAPI testing. All backend feature epics (E05, E06, E07, E08) must produce integration tests using this stack before closing their test story.

### 3. [PROCESS_CHANGE] Admin Introspection Endpoints Are Mandatory for Stateful Services

Every service with stateful resilience mechanisms (circuit breaker, rate limiter, connection pool, queue) must expose ClusterIP-only admin endpoints. This is now a standard acceptance criterion for backend services. Future stories implementing stateful components must include an introspection endpoint in the story's acceptance criteria.

### 4. [PROCESS_CHANGE] ATDD Checklist Must Be Created Before Story Development Starts

Epic 4 demonstrates what happens when the ATDD cycle is skipped: the test design is thorough, the implementation is correct, but there is no formal documentation that tests were written in RED state before implementation. Starting with E05, ATDD checklists are **mandatory before the development story begins**, not after.

### 5. [PROCESS_CHANGE] SSE Streaming Requires Dedicated Performance Benchmarks

SSE proxy latency (< 100ms per event) cannot be verified by functional tests alone. Future epics building real-time streaming features (E07 proposal generation, E11 agent executions) must include a P3 benchmark test with wall-clock timing assertions using a controlled event source with known send timestamps.

---

## Findings Summary

| ID | Type | Severity | Description | Recommended Action |
|----|------|----------|-------------|-------------------|
| F-001 | PATTERN | — | Webhook HMAC-SHA256 constant-time comparison pattern — `hmac.compare_digest()` required for all webhook signature validation | Apply to any future webhook receiver endpoint |
| F-002 | PATTERN | — | Two-layer resilience composition: `circuit_breaker(retry(http_factory))` — reusable for all outbound service calls | Standard pattern for E05+ outbound HTTP clients |
| F-003 | PATTERN | — | YAML-driven registry pattern: decouple business logical names from external provider IDs | Apply to any third-party integration with swappable identifiers |
| F-004 | PATTERN | — | testcontainers + respx integration test stack — CI-safe real-stack testing without live credentials | Mandatory for all backend integration test stories |
| F-005 | PATTERN | — | Fire-and-forget async DB writes (`asyncio.create_task`) for audit/logging paths | Apply to all audit writes in backend services |
| F-006 | PATTERN | — | Admin introspection endpoints for all stateful service mechanisms | Mandatory acceptance criterion for stateful backend components |
| F-007 | ANTI_PATTERN | **critical** | Zero ATDD checklists for E04 (10 stories, 0 checklists) — ATDD cycle not executed | `[ACTION]` Run retrospective ATDD checklists; enforce before-dev gate in E05 |
| F-008 | ANTI_PATTERN | **high** | No NFR assessment for Epic 4 | `[ACTION]` Run `bmad-testarch-nfr` for E04 before closing |
| F-009 | ANTI_PATTERN | low | sprint-status.yaml `epic-4: in-progress` — 4th consecutive occurrence | `[ACTION]` Update to `done`; add CI enforcement |
| F-010 | ANTI_PATTERN | medium | S03.11 TEA still `in-progress` — carried across 2 epics | `[ACTION]` Must close before E05 development starts |
| F-011 | ANTI_PATTERN | **critical** | 10 open security/reliability items across E01–E04 — no hardening pass yet | `[ACTION]` Inject as non-negotiable stories into E05 planning |
| F-012 | ANTI_PATTERN | medium | SSE latency < 100ms requirement has no timing test | `[ACTION]` Add P3 benchmark test with wall-clock assertion |
| F-013 | ANTI_PATTERN | low | Config negative-path test for missing env vars not implemented | `[ACTION]` Add E04-P2-NEW-001 unit test |
| F-014 | ACTION | **critical** | Security hardening stories must be injected into E05 backlog immediately | `story_injection` — SEVERITY: critical |
| F-015 | ACTION | **high** | NFR assessment for E04 must be run before starting E05 | `standards_update` — SEVERITY: high |
| F-016 | ACTION | medium | Add timing assertion to SSE latency test (E04-P1-009) | `prompt_adjustment` — SEVERITY: medium |
| F-017 | ACTION | medium | Circuit breaker state migration to Redis (pre-scale) must be planned before E04 service goes to multi-replica | `story_injection` — SEVERITY: medium |
| F-018 | PROCESS_CHANGE | — | Two-layer resilience pattern is the standard for all outbound HTTP in EU Solicit | Document in architecture; reference in E05+ story specs |
| F-019 | PROCESS_CHANGE | — | ATDD checklist creation is a BEFORE-DEV gate, not optional | Enforce via sprint planning; no story starts without checklist scaffolded |
| F-020 | PROCESS_CHANGE | — | Admin introspection endpoints are mandatory for stateful backend components | Add to story template as standard acceptance criterion |

---

## Metrics Deep Dive

### Story Velocity

| Story | Title | Points | Code Review | Key Tests |
|-------|-------|--------|-------------|-----------|
| S04.01 | FastAPI Scaffold & Health Probes | 2 | APPROVE | 7 tests (3 unit, 4 integration) |
| S04.02 | httpx Async Client & KraftData Auth | 3 | APPROVE | Unit: auth header, timeout, pool limits, clean shutdown |
| S04.03 | Agent Registry | 2 | APPROVE | Unit: load/resolve/not-found/duplicate/hot-reload |
| S04.04 | Sync Execution Endpoints | 5 | APPROVE | Unit: routing, UUID passthrough, type mismatch |
| S04.05 | SSE Stream Proxy | 5 | APPROVE | Unit: events, disconnect, client cancel, heartbeat, timeout, partial frame |
| S04.06 | Circuit Breaker & Retry | 5 | APPROVE | 76 unit tests total (17 new): state machine, backoff, non-retryable paths |
| S04.07 | Webhook Receiver & Redis Publish | 3 | APPROVE | Unit: HMAC, routing, idempotency, constant-time comparison |
| S04.08 | Execution Logging & DB Schema | 3 | APPROVE | Unit: row creation, circuit-open logging, DB failure swallowed |
| S04.09 | Rate Limit & Concurrency Control | 3 | APPROVE | Unit: semaphore, queue timeout, per-agent limit, streaming hold |
| S04.10 | Integration Tests & E2E Validation | 3 | APPROVE | 10 integration scenarios, ≥85% coverage, testcontainers |
| **Total** | | **34** | **10/10 APPROVE** | — |

### Traceability Coverage

| Priority | FULL | PARTIAL | NONE | % FULL |
|----------|-----:|--------:|-----:|-------:|
| P0 | 17 | 0 | 0 | **100%** |
| P1 | 24 | 1 | 0 | **96%** |
| P2 | 11 | 4 | 0 | **73%** |
| P3 | 2 | 0 | 0 | **100%** |
| **Overall** | **54** | **5** | **0** | **91.5%** |

### Risk Mitigation Status

| Risk ID | Score | Category | Status | Notes |
|---------|-------|----------|--------|-------|
| E04-R-001 | 6 | SEC | ✅ FULLY MITIGATED | `hmac.compare_digest()` + constant-time unit test |
| E04-R-002 | 6 | TECH | ✅ FULLY MITIGATED | Queue-based SSE proxy, CancelledError propagation, frozen-clock tests |
| E04-R-003 | 6 | DATA | ✅ MITIGATED | try/except on XADD, ERROR log, processed=False; monitoring backlog item added |
| E04-R-004 | 4 | PERF | ⚠️ DOCUMENTED | Semaphore starvation under streaming load; separate pools noted as Phase 2 |
| E04-R-005 | 3 | TECH | ⚠️ DOCUMENTED | Per-instance circuit state; Redis migration backlog for multi-replica |
| E04-R-006 | 4 | TECH | ✅ MITIGATED | asyncio.Lock on registry replacement; rollback on validation failure |
| E04-R-007 | 4 | OPS | ✅ MITIGATED | ERROR log on DB write failure; execution_id retained for recovery |
| E04-R-008 | 2 | OPS | ✅ MONITORED | testcontainers health retry configured; @pytest.mark.slow tagging |
| E04-R-009 | 2 | DATA | ✅ MONITORED | Startup logs all loaded entries; reload endpoint documented |

---

## Carry-Forward Items for Epic 5

### Must-Do Before E05 Development Starts

1. **[SECURITY — CRITICAL] Inject security hardening stories into E05 sprint** — 1 day
   - Prometheus /metrics endpoint (E01 carry-forward, 4 epics overdue)
   - Dockerfile USER directive (E01 carry-forward)
   - Dependabot configuration (E01 carry-forward)
   - `User.is_active` check in login + refresh (E02 carry-forward)
   - bcrypt executor (E02 carry-forward — 200-400ms event loop block)
   - Redis rate limiter fail-open (E02 carry-forward)
   - JWT audience/issuer claim verification (E02 carry-forward)
   - pnpm audit in CI (E03 carry-forward)

2. **[QUALITY] Complete S03.11 TEA automation review** — 2 hours
   - Toast hover-pause E2E (E03-P1-014, E03-P2-017) still untested
   - `tea_status: 3-11-toast-notification-system: in-progress` must close before E05

3. **[PROCESS] Update sprint-status.yaml: epic-4: done** — 5 minutes
   - All 10 stories are done; epic status must reflect this

4. **[QUALITY] Run NFR assessment for Epic 4** — 4 hours
   - Focus areas: SSE proxy performance (< 100ms), circuit breaker per-instance limitation, Redis publish reliability

5. **[QUALITY] Create retrospective ATDD checklists for E04 stories** — 4 hours
   - Or formally document the gap and enforce ATDD before-dev gate in E05

### Must-Do During E05 (Pre-Production Hardening)

1. **Migrate circuit breaker state to Redis** — 4 hours
   - Required before ai-gateway runs as multiple replicas
   - E04-R-005 pre-scale backlog item

2. **Add SSE latency benchmark (P3)** — 2 hours
   - Wall-clock timing assertion for < 100ms per-event requirement (E04 residual concern #1)
   - Use controlled SSE mock with known send timestamps + `time.monotonic()` comparison

3. **Add `agents.yaml` full 29-entry validation to CI** — 1 hour
   - E04-P3-005 (nightly only) verifies 29-entry count; promote to PR gate

4. **Document circuit breaker per-instance limitation in ADR** — 1 hour
   - E04-R-005: in-memory state is per-process; document scaling implications

5. **Config negative-path test for missing env vars** — 1 hour
   - E04-P2-NEW-001: pydantic-settings ValidationError with field name in message

---

## Conclusion

Epic 4 is a **strong technical delivery**. The AI Gateway is the most architecturally complex piece built so far — it combines async HTTP proxying, SSE streaming, circuit breaking, retry logic, webhook security, Redis event publishing, and execution audit logging in a single service. All 10 stories were completed, all code reviews approved, and the traceability gate passed on the first run.

The **primary concern entering Epic 5 is the compounding security and quality debt**: 10 open HIGH/CRITICAL items spanning 4 epics, zero ATDD checklists for E04, no NFR assessment, and no performance baseline after 4 epics. These are not design failures — the test design, implementation quality, and resilience patterns are all strong. But the process gates that enforce discipline (ATDD checklists, NFR assessments, security hardening sprints) have been consistently deferred. At the scale of E05–E07 (data pipeline, opportunity discovery, proposal generation), this debt becomes a credible production risk.

Three non-negotiables before Epic 5 closes:
1. All 8 carry-forward security items shipped as E05 stories (no further deferral)
2. ATDD checklists enforced as a before-development gate for every story
3. NFR assessment run for both E04 and E05 before the Demo milestone

---

**Generated:** 2026-04-14
**Workflow:** bmad-retrospective v1.0
**Input sources:**
- `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md`
- `eusolicit-docs/test-artifacts/test-design-epic-04.md`
- `eusolicit-docs/test-artifacts/traceability-matrix.md`
- `eusolicit-docs/implementation-artifacts/4-{1..10}-*.md` (all 10 stories)
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml`
- `eusolicit-docs/implementation-artifacts/deferred-work.md`
- `eusolicit-docs/test-artifacts/retrospective-epic-3.md` (carry-forward context)

<!-- Powered by BMAD-CORE -->
