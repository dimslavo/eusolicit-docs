---
epic: 5
epicTitle: 'Data Pipeline & Opportunity Ingestion'
date: '2026-04-17'
stories: 12
points: 34
sprints: '3-4'
overall_verdict: 'SUCCESS_WITH_CONCERNS'
nfr_score: '16/29 (PASS with CONCERNS)'
trace_gate: 'PASS (98.1% FULL, 100% P0/P1)'
---

# Retrospective — Epic 5: Data Pipeline & Opportunity Ingestion

**Date:** 2026-04-17
**Epic:** E05 — Data Pipeline & Opportunity Ingestion (12 stories, 34 points, Sprints 3–4)
**Verdict:** SUCCESS WITH CONCERNS — All 12 stories DONE. TRACE_GATE PASS. NFR PASS with 4 HIGH priority reliability items deferred.

---

## Executive Summary

Epic 5 delivered the complete Celery-based asynchronous data pipeline for EU Solicit: schema migrations (S05.01), Beat scheduler (S05.02), AI Gateway client with circuit breaker (S05.03), three crawler tasks for AOP, TED, and EU Grants (S05.04–S05.06), relevance scoring (S05.07), submission guide generation (S05.08), Redis Streams event publishing (S05.09), expired opportunity cleanup (S05.10), enrichment queue worker (S05.11), and E2E integration tests + observability (S05.12).

All 12 stories are `done` in `sprint-status.yaml`. The TRACE_GATE passed (98.1% FULL coverage, 100% P0/P1). The NFR assessment returned PASS with CONCERNS (16/29 ADR criteria, 0 blockers, 4 HIGH priority items). Epic 4 carry-forward items were partially addressed — ATDD checklists were created (S05.06–S05.12 at minimum), NFR assessment was completed, and `epic-4: done` and `epic-5: done` are both reflected in sprint-status.yaml.

**The primary concern entering Epic 6 is the compounding production-readiness debt in the data pipeline itself:** four HIGH reliability items, the continued absence of a k6 throughput baseline, and the first generation of TEA test automation scores for E05 stories still absent from sprint-status.yaml.

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Stories complete | 12/12 (100%) |
| Sprint status | `epic-5: done` ✅ |
| TRACE_GATE | PASS — P0 100% · P1 100% · P2 91.7% · P3 100% |
| Overall AC coverage | 98.1% FULL (51/52), 100% any coverage |
| High risks mitigated (Score ≥6) | 3/3 (dedup race FULL, AI Gateway cascade PARTIAL, Redis publish SUBSTANTIAL) |
| Code review verdicts | 12/12 APPROVE |
| ATDD checklists | 7/12 — S05.06–S05.12 created; S05.01–S05.05 still DESIGN phase only |
| NFR assessment | PASS with CONCERNS (16/29, 0 blockers, 4 HIGH) |
| TEA test review scores | **Not populated for any E05 story** (gap carried from E04) |
| Deferred work items | 20+ items across 6 story sections |

---

## What Went Well

### 1. [PATTERN] All Three Top-Ranked Risks Mitigated with Dedicated P0 Tests
`IMPACT: standards_update | SEVERITY: medium`

All three Score-6 risks from the E05 test design received dedicated P0 test coverage:

- **E05-R-001 (Dedup race condition):** Atomic PostgreSQL `INSERT ... ON CONFLICT DO UPDATE` eliminates the check-then-insert race. DB-level UNIQUE constraint on `(source_id, source_type)`. E05-P0-001 (sequential) and E05-P0-002 (concurrent) both FULL. **FULLY MITIGATED.**
- **E05-R-002 (AI Gateway cascade / orphaned runs):** Circuit breaker + Celery autoretry confirmed. E05-P0-003/004 test coverage exists. **PARTIALLY MITIGATED** — `_RUN_ID_REGISTRY` prefork incompatibility remains HIGH priority deferred.
- **E05-R-003 (Redis publish-or-lose):** Isolated retry (publish-only retry, not re-triggering crawl), ERROR log with `run_id` + `opportunity_ids`, data persisted in PostgreSQL before publish. E05-P0-007/008 FULL. **SUBSTANTIALLY MITIGATED.**

Pattern: dedicating P0 test slots specifically to the top-ranked risks from the test design is the correct gate signal. Continuing this discipline in E06+.

### 2. [PATTERN] TRACE_GATE PASS with 100% P0/P1 Coverage
`IMPACT: standards_update | SEVERITY: low`

The traceability matrix returned 100% P0 and 100% P1 FULL coverage (39/39 combined) and 98.1% overall FULL. The sole PARTIAL (S05.12-AC4, `cleanup_expired` missing `crawler_type` log binding) is tracked with a RED stub test. Every story AC maps to at least one planned test.

This is the second consecutive TRACE_GATE PASS (after E04) on a backend service epic. The test design quality is consistent.

### 3. [PATTERN] Comprehensive, Structured Deferred Work Documentation
`IMPACT: standards_update | SEVERITY: low`

20+ deferred items across 6 E05 story sections in `deferred-work.md` — each with file location, line references, rationale, and severity classification. No unacknowledged debt. The NFR assessment directly maps its HIGH-priority findings to specific deferred-work entries (S05.06, S05.11, S05.12).

This structured debt tracking enables the next engineer to immediately understand scope, location, and rationale without code archaeology.

### 4. [PATTERN] NFR Assessment Completed for Epic 5 (Process Improvement from E04)
`IMPACT: standards_update | SEVERITY: medium`

The E04 retrospective flagged missing NFR assessment as HIGH anti-pattern. Epic 5 delivered `nfr-report-epic-05.md` at epic close. This closes the carry-forward and restores the quality gate sequence: test design → traceability → ATDD → code review → NFR assessment → retrospective.

### 5. [PATTERN] Celery Eager Mode + testcontainers Integration Test Architecture
`IMPACT: standards_update | SEVERITY: low`

The E05 integration test stack extends the E04 testcontainers + respx template with:
- Celery `task_always_eager` for synchronous task testing (CI-safe, no real workers)
- `freezegun` for back-off timing assertions
- testcontainers PostgreSQL + Redis at session scope (shared across modules)
- Factory fixtures (OpportunityFactory, CrawlerRunFactory, CompanyProfileFactory, EnrichmentQueueFactory)
- Mock fixture sets per crawler type (AOP, TED, EU Grants) with paginated and error variants

This architecture is now the proven template for all backend async/worker service integration tests.

### 6. [PATTERN] Circuit Breaker Applied at Pipeline Client Layer (E04 Pattern Reused)
`IMPACT: standards_update | SEVERITY: low`

The E04 two-layer resilience pattern (circuit breaker outer, retry inner) was correctly applied in S05.03 for the AI Gateway client in the pipeline service. Correlation ID propagation from Beat task context through all AI Gateway calls. This validates the E04 pattern as portable across service boundaries without redesign.

---

## What Could Be Improved

### 1. [ANTI-PATTERN] `_RUN_ID_REGISTRY` Module-Level Dict Is Prefork-Incompatible
`IMPACT: prompt_adjustment | SEVERITY: critical`

```
[ACTION] Replace _RUN_ID_REGISTRY with Redis-backed task run ID tracking before production deployment
IMPACT: standards_update
SEVERITY: critical
```

The `_RUN_ID_REGISTRY` module-level Python dict is not shared across Celery prefork worker processes. When `on_failure` fires in a different process than the task that ran, it cannot retrieve the `run_id` to mark `crawler_runs.status = failed`. Combined with the fact that each Celery retry creates a new `CrawlerRun` record (and `on_failure` only marks the last), previous runs remain permanently in `retrying` state.

All current tests run in Celery eager mode — this masks the prefork incompatibility entirely. A production deployment with the default `prefork` pool silently breaks `CrawlerRun` lifecycle management.

**Fix:** Store `run_id` in Redis with `SETEX task_id run_id 3600`. The `on_failure` handler reads from Redis. Alternatively, migrate the pipeline service to `gevent`/`eventlet` pool, which eliminates all cross-process state issues.

### 2. [ANTI-PATTERN] Non-Atomic DB Transaction Patterns Across Three Crawlers
`IMPACT: standards_update | SEVERITY: high`

```
[ACTION] Combine opportunity upsert and crawler_runs status update into a single transaction in crawl_aop, crawl_ted, crawl_eu_grants
IMPACT: standards_update
SEVERITY: high
```

All three crawler tasks use separate DB sessions for (1) opportunity upsert commit and (2) `CrawlerRun` status update. A process crash between the two commits leaves opportunities persisted but `crawler_runs.status` permanently `running`. The enrichment queue worker has the same pattern: enrichment result commit and item deletion in separate transactions — an item can get permanently stuck in `processing` with no recovery mechanism if the second commit fails.

**Fix:** `async with session.begin()` wrapping both operations, plus a `finally` block updating the run status as a safety net.

### 3. [ANTI-PATTERN] S05.01–S05.05 ATDD Checklists Never Created
`IMPACT: standards_update | SEVERITY: high`

```
[ACTION] Generate retrospective ATDD checklists for S05.01–S05.05 before E06 development starts
IMPACT: standards_update
SEVERITY: high
```

Five of twelve E05 stories have implementation artifacts but no ATDD checklists. The traceability matrix explicitly notes these are "DESIGN phase only" — no TDD RED state was documented before implementation. This is a partial regression from the improvement made in E05 vs E04 (E04 had 0/10; E05 has 7/12).

The ATDD checklist serves as the documented proof that tests were specified before code. Without it for S05.01–S05.05, these stories lack the formal per-story quality gate.

### 4. [ANTI-PATTERN] TEA Test Automation Review Scores Not Populated for Any E05 Story
`IMPACT: config_tuning | SEVERITY: medium`

```
[ACTION] Run bmad-testarch-test-review on at minimum S05.03 (AI Gateway client) and S05.12 (E2E integration) before E06 closes
IMPACT: standards_update
SEVERITY: medium
```

`tea_status` in `sprint-status.yaml` has no entries for any Epic 5 story. The E04 retrospective did not call out this gap explicitly, but it has now carried forward across two backend epics (E04 has 0 TEA scores; E05 has 0 TEA scores). The review report (`review-report.md`) was last generated for story 11.7 — a completely different epic.

TEA scores provide independent test quality signal (determinism, isolation, maintainability, performance). Given the known silent exception swallowing in E05 metric instrumentation and the log field test masking identified in the NFR report, a TEA review of the S05.12 integration suite is particularly warranted.

### 5. [ANTI-PATTERN] Silent `except Exception: pass` on All Metrics Recording Points
`IMPACT: prompt_adjustment | SEVERITY: medium`

```
[ACTION] Replace all silent metrics exception swallowing with at-minimum debug-level logging across crawl_aop.py, crawl_ted.py, crawl_eu_grants.py, process_enrichment_queue.py
IMPACT: standards_update
SEVERITY: medium
```

Four files use `except Exception: pass` on Prometheus metrics recording — this aids test stability (no test failures from metric side-effects) but completely silences operational errors. A broken Prometheus client or misconfigured metric label will produce zero signal in production.

**Fix:** `except Exception as exc: log.debug("metrics_recording_failed", exc=str(exc))` — three-character change per site that restores observability without impacting test behaviour.

### 6. [ANTI-PATTERN] 4xx Errors Increment Circuit Breaker Failure Counter (OBS-001)
`IMPACT: prompt_adjustment | SEVERITY: medium`

```
[ACTION] Restrict circuit breaker failure counting to 5xx and transport errors only (OBS-001, S05.03)
IMPACT: prompt_adjustment
SEVERITY: medium
```

The E05 AI Gateway client circuit breaker uses a generic `except Exception` catch-all that increments the failure counter for all errors, including 4xx (client-side) responses. Five consecutive bad requests (misconfigured caller, wrong agent name) can trip the circuit open, causing all real agent calls to be rejected until the half-open window (60s).

This is carried forward from the same bug documented in E04 (`circuit_breaker.py` for the AI Gateway service). The fix is identical: inspect HTTP status before incrementing the counter; count only 5xx/transport errors.

### 7. [ANTI-PATTERN] No k6 Pipeline Throughput Baseline — `load-test-results.md` Still a Template
`IMPACT: story_injection | SEVERITY: medium`

```
[ACTION] Inject k6 pipeline throughput measurement as a concrete Sprint 5 task — do not defer to staging
IMPACT: story_injection
SEVERITY: medium
```

The `load-test-results.md` file is an unfilled template. The NFR report explicitly calls out the absence of any crawl cycle completion time measurement. The 24h data freshness SLA (PRD §3.1) cannot be verified without a throughput baseline.

This gap was flagged in E03 (no load testing) and again in E04, and again in E05. At Demo milestone (Sprint 8), the platform must demonstrate that the crawl pipeline can process 100+ opportunities per cycle well within the 24h SLA.

### 8. [ANTI-PATTERN] Health Endpoint Exposing Exception Details in 503 Response Body
`IMPACT: prompt_adjustment | SEVERITY: low`

```
[ACTION] Replace str(exc) in pipeline_health 503 response with generic message; log full exception server-side
IMPACT: prompt_adjustment
SEVERITY: low
```

`main.py:43` returns `str(exc)` in the HTTP 503 response body. While the health endpoint has no public ingress (ClusterIP only), this is a defence-in-depth violation — DB connection strings and hostnames could leak to internal health-check callers, monitoring systems, or log aggregators that capture HTTP response bodies.

Quick win: `"Service temporarily unavailable"` as the response body; `log.error("health_check_failed", exc_info=True)` server-side.

---

## Process Learnings

### 1. [PROCESS_CHANGE] Celery Prefork Pool Incompatibility Requires Explicit Test Coverage in Non-Eager Mode

Module-level Python state (dicts, lists, counters) is not safe across Celery prefork worker processes. Any state used in Celery signal handlers (`on_failure`, `on_retry`, `on_success`) must be stored in an external system (Redis or DB). All E05 tests run in `task_always_eager` mode which masks this. Future pipeline epics (or hardening passes) must include at least one integration test that runs the pipeline with a real Celery prefork pool.

### 2. [PROCESS_CHANGE] DB Transaction Atomicity Must Be Story AC, Not Code Review Finding

The non-atomic commit pattern (separate sessions for data write + status update) appeared independently in S05.04, S05.05, S05.06, and S05.11. Because it was not an explicit story acceptance criterion, the reviewer deferred it rather than blocking the review. Starting with E06, any story that writes to multiple tables as part of a single logical operation must include an AC: "writes are committed in a single transaction or have explicit rollback handling."

### 3. [PROCESS_CHANGE] ATDD Checklists for Infrastructure Stories Are Non-Negotiable

The rationale for skipping ATDD checklists on S05.01–S05.05 was likely "infrastructure/schema stories don't need TDD RED stubs." This is incorrect: S05.01 (schema) has 4 ACs, some P0 (unique constraint) — exactly the kind that benefits from a RED stub before migration is written. The rule from E04 retrospective stands: **ATDD checklist must be created before the development story begins, regardless of story type.**

### 4. [PROCESS_CHANGE] Circuit Breaker Failure Classification Is a Shared Bug — Fix Once, Propagate

OBS-001 (4xx responses incrementing circuit breaker failure count) exists in both the E04 AI Gateway service `circuit_breaker.py` and the E05 data pipeline AI Gateway client. This is one logical bug in two places. The fix must be applied to both files simultaneously and verified by a shared test.

### 5. [PROCESS_CHANGE] k6 Throughput Baseline Is a Sprint Deliverable, Not a Backlog Item

Three consecutive epics have deferred load testing to "when staging is available." The E05 NFR report explicitly flags this as an evidence gap for the 24h freshness SLA. From E06 onward, each epic that adds a new data-path must include a throughput measurement task in the sprint, even if against a local Docker Compose environment with synthetic data.

---

## Findings Summary

| ID | Type | Severity | Description | Recommended Action |
|----|------|----------|-------------|-------------------|
| F-001 | PATTERN | — | All 3 top-ranked risks mitigated with P0 test coverage | Maintain per-epic risk-to-P0-test traceability |
| F-002 | PATTERN | — | TRACE_GATE PASS with 100% P0/P1 FULL (39/39) | Continue test design quality discipline |
| F-003 | PATTERN | — | Structured deferred work with location refs and rationale | Enforce in all epics going forward |
| F-004 | PATTERN | — | NFR assessment completed (E04 carry-forward closed) | Run NFR for every epic at close |
| F-005 | PATTERN | — | Celery eager + testcontainers integration test architecture | Extend template for E06 async services |
| F-006 | PATTERN | — | E04 two-layer resilience pattern correctly reused without redesign | Reference E04 circuit breaker in E06 story specs |
| F-007 | ANTI-PATTERN | **critical** | `_RUN_ID_REGISTRY` prefork incompatibility — silent in eager mode, breaks production | `[ACTION]` Redis-backed run ID storage before production load |
| F-008 | ANTI-PATTERN | **high** | Non-atomic DB transactions in all 3 crawlers + enrichment worker | `[ACTION]` Single-transaction AC required for multi-table writes |
| F-009 | ANTI-PATTERN | **high** | S05.01–S05.05 ATDD checklists not created (5/12 missing) | `[ACTION]` Generate retrospective checklists; enforce before-dev gate |
| F-010 | ANTI-PATTERN | medium | No TEA test review scores for any E05 story | `[ACTION]` Run test review on S05.03 and S05.12 minimum |
| F-011 | ANTI-PATTERN | medium | Silent `except Exception: pass` on all metrics instrumentation | `[ACTION]` Add `log.debug` to all 4 metric exception handlers |
| F-012 | ANTI-PATTERN | medium | OBS-001: 4xx errors trip circuit breaker (same bug as E04) | `[ACTION]` Fix in both E04 and E05 files simultaneously |
| F-013 | ANTI-PATTERN | medium | No k6 throughput baseline — `load-test-results.md` still a template | `[ACTION]` Inject as Sprint 5 deliverable, not backlog item |
| F-014 | ANTI-PATTERN | low | Health endpoint exposes `str(exc)` in 503 body | `[ACTION]` Quick win — generic message + server-side log |
| F-015 | ACTION | **critical** | Fix `_RUN_ID_REGISTRY` with Redis or gevent pool before pipeline runs at scale | `IMPACT: standards_update` `SEVERITY: critical` |
| F-016 | ACTION | **high** | Combine upsert + status update into single transaction in all 3 crawlers | `IMPACT: standards_update` `SEVERITY: high` |
| F-017 | ACTION | **high** | Generate ATDD checklists for S05.01–S05.05 (retrospective) | `IMPACT: standards_update` `SEVERITY: high` |
| F-018 | ACTION | medium | Add stuck-in-processing recovery to enrichment queue worker | `IMPACT: standards_update` `SEVERITY: medium` |
| F-019 | ACTION | medium | Restrict circuit breaker to 5xx/transport errors (OBS-001) — fix in E04 + E05 files | `IMPACT: prompt_adjustment` `SEVERITY: medium` |
| F-020 | ACTION | medium | Inject k6 throughput measurement task into Sprint 5 | `IMPACT: story_injection` `SEVERITY: medium` |
| F-021 | ACTION | medium | Run TEA test review on S05.03 (AI Gateway client) and S05.12 (E2E suite) | `IMPACT: standards_update` `SEVERITY: medium` |
| F-022 | ACTION | low | Add debug-level log to all `except Exception: pass` metric handlers | `IMPACT: prompt_adjustment` `SEVERITY: low` |
| F-023 | ACTION | low | Replace `str(exc)` in health endpoint 503 with generic message | `IMPACT: prompt_adjustment` `SEVERITY: low` |
| F-024 | PROCESS_CHANGE | — | Celery prefork non-eager tests mandatory for pipeline services | Document as architecture standard |
| F-025 | PROCESS_CHANGE | — | DB transaction atomicity must be a story AC, not a code review finding | Add to story template for E06+ |
| F-026 | PROCESS_CHANGE | — | ATDD checklists are non-negotiable for all story types including infrastructure | Enforce before-dev gate unconditionally |

---

## Carry-Forward Items for Epic 6

### Must-Do Before E06 Development Starts

1. **[RELIABILITY — CRITICAL] Fix `_RUN_ID_REGISTRY` prefork incompatibility** — 4h, Backend Lead
   - Redis `SETEX task_id run_id 3600`; `on_failure` reads from Redis
   - Validate with real prefork pool integration test

2. **[RELIABILITY — HIGH] Mark previous CrawlerRun records `superseded` on retry** — 2h, Backend Lead
   - `UPDATE crawler_runs SET status='superseded' WHERE task_id=:tid AND status IN ('running', 'retrying')`
   - Validate 3-retry scenario: 3 records, last = `failed`, others = `superseded`

3. **[RELIABILITY — HIGH] Combine upsert + `crawler_runs` status update into single transaction** — 3h, Backend Lead
   - Affects `crawl_aop.py`, `crawl_ted.py`, `crawl_eu_grants.py`
   - Add `finally` block as safety net

4. **[RELIABILITY — HIGH] Add stuck-in-processing recovery to enrichment queue worker** — 2h, Backend Lead
   - Periodic sweep: reset items with `status='processing'` and `updated_at < now() - interval '10 minutes'` to `pending`

5. **[QUALITY — HIGH] Generate retrospective ATDD checklists for S05.01–S05.05** — 2h
   - Run `bmad-testarch-atdd` for each story; document gap formally if not done

6. **[QUALITY — MEDIUM] Run TEA test review on S05.03 and S05.12** — 2h
   - Review AI Gateway client unit tests (S05.03) and E2E integration suite (S05.12) for determinism, isolation, maintainability

### Must-Do During E06 Sprint

1. **[SECURITY/RELIABILITY — MEDIUM] Fix OBS-001 circuit breaker 4xx miscounting** — 1h
   - Update `circuit_breaker.py` in both E04 AI Gateway service and E05 pipeline client
   - Test: 5× 400 responses do not open circuit; 5× 503 responses do

2. **[PERFORMANCE — MEDIUM] Execute k6 pipeline throughput baseline** — 4h, QA / Backend Lead
   - Measure crawl cycle completion time with 100 synthetic opportunities per source
   - Target: < 5 minutes per cycle; establish SLO for Demo milestone

3. **[OBSERVABILITY — LOW] Add debug log to all metrics exception handlers** — 30min
   - Four files: `crawl_aop.py:179`, `crawl_ted.py:206`, `crawl_eu_grants.py:131`, `process_enrichment_queue.py:176`

4. **[SECURITY — LOW] Replace `str(exc)` in health endpoint 503** — 30min
   - `main.py:43` → generic message + `log.error("health_check_failed", exc_info=True)`

5. **[OBSERVABILITY — MEDIUM] Add `max(1, batch_size)` guard on `ENRICHMENT_BATCH_SIZE`** — 15min
   - `batch_size = max(1, int(os.getenv("ENRICHMENT_BATCH_SIZE", "20")))`

### Ongoing Backlog (Pre-Demo Milestone)

1. **Migrate pipeline to Celery `gevent` pool** — eliminates all prefork state issues
2. **Add Beat task overlap prevention via Redis distributed lock** — prevents concurrent crawl invocations
3. **Add chunking to `score_opportunities` group dispatch** — 50-opportunity chunks prevent Celery queue saturation at scale
4. **Savepoint-based `db_session` fixture** — replace rollback-only isolation (cross-test pollution risk)

---

## Metrics Deep Dive

### Story Velocity

| Story | Title | Points | Code Review | ATDD Phase |
|-------|-------|--------|-------------|-----------|
| S05.01 | Pipeline Schema Migration & Model Layer | 3 | APPROVE | DESIGN only |
| S05.02 | Celery App Bootstrap & Beat Schedule | 2 | APPROVE | DESIGN only |
| S05.03 | AI Gateway Client Module with Retry | 3 | APPROVE | DESIGN only |
| S05.04 | AOP Crawler Task | 3 | APPROVE | DESIGN only |
| S05.05 | TED Crawler Task | 3 | APPROVE | DESIGN only |
| S05.06 | EU Grants Crawler Task | 3 | APPROVE | RED (14 GREEN / 4 RED) |
| S05.07 | Relevance Scoring Post-Processing | 3 | APPROVE | RED (all stubs RED) |
| S05.08 | Submission Guide Generation Task | 3 | APPROVE | RED (all stubs RED) |
| S05.09 | Redis Streams Event Publishing | 2 | APPROVE | RED (all stubs RED) |
| S05.10 | Expired Opportunity Cleanup Task | 2 | APPROVE | RED (all stubs RED) |
| S05.11 | Enrichment Queue Worker | 4 | APPROVE | RED (all stubs RED) |
| S05.12 | E2E Integration Tests & Observability | 3 | APPROVE | RED (3 GREEN / 9 RED) |
| **Total** | | **34** | **12/12 APPROVE** | **7/12 checklists** |

### Traceability Coverage

| Priority | FULL | PARTIAL | NONE | % FULL |
|----------|-----:|--------:|-----:|-------:|
| P0 | 13 | 0 | 0 | **100%** |
| P1 | 26 | 0 | 0 | **100%** |
| P2 | 11 | 1 | 0 | **91.7%** |
| P3 | 1 | 0 | 0 | **100%** |
| **Overall** | **51** | **1** | **0** | **98.1%** |

### NFR Assessment Summary

| Category | Criteria Met | Status |
|---|---|---|
| 1. Testability & Automation | 4/4 | ✅ PASS |
| 2. Test Data Strategy | 2/3 | ⚠️ CONCERNS |
| 3. Scalability & Availability | 1/4 | ⚠️ CONCERNS |
| 4. Disaster Recovery | 0/3 | ⚠️ CONCERNS |
| 5. Security | 2/4 | ⚠️ CONCERNS |
| 6. Monitorability | 3/4 | ⚠️ CONCERNS |
| 7. QoS & QoE | 2/4 | ⚠️ CONCERNS |
| 8. Deployability | 2/3 | ⚠️ CONCERNS |
| **Total** | **16/29** | **⚠️ PASS (with CONCERNS)** |

### Risk Mitigation Status

| Risk ID | Score | Category | Status |
|---------|-------|----------|--------|
| E05-R-001 | 6 | DATA | ✅ FULLY MITIGATED — atomic upsert, DB unique constraint, E05-P0-001/002 |
| E05-R-002 | 6 | TECH | ⚠️ PARTIALLY MITIGATED — circuit breaker + retry confirmed; `_RUN_ID_REGISTRY` prefork incompatibility HIGH |
| E05-R-003 | 6 | DATA | ✅ SUBSTANTIALLY MITIGATED — isolated retry, ERROR log, E05-P0-007/008 |
| E05-R-005 | 4 | PERF | ⚠️ DOCUMENTED — chord starvation risk at 500+ opportunities; chunking deferred |
| E05-R-006 | 4 | DATA | ✅ MITIGATED — `__mapper_args__` default criterion, E05-P1-004 test |
| E05-R-007 | 4 | TECH | ⚠️ DOCUMENTED — stuck `processing` items; recovery sweep deferred |

---

## Conclusion

Epic 5 is a **solid functional delivery**. The Celery pipeline, crawler tasks, scoring chain, and event publishing all work correctly under test conditions. The test design quality is strong (60 tests, 100% P0/P1 FULL coverage), ATDD checklists improved from the E04 baseline (7/12 vs 0/10), and the NFR assessment was completed — closing two E04 carry-forwards.

The **critical production risk** is the `_RUN_ID_REGISTRY` prefork incompatibility. This is not a theoretical concern — it will silently break `CrawlerRun` lifecycle management in any deployment using the default Celery prefork pool. Combined with the non-atomic commit patterns in three crawlers and the enrichment queue worker, the pipeline has four HIGH-priority reliability issues that must be resolved before it processes real data at scale.

**Three non-negotiables before Epic 6 closes:**
1. `_RUN_ID_REGISTRY` replaced with Redis-backed storage (or Celery gevent pool adopted)
2. DB transaction atomicity enforced in crawlers via single-transaction AC in story specs
3. k6 throughput baseline executed (not deferred to staging)

---

**Generated:** 2026-04-17
**Workflow:** bmad-retrospective v2.0
**Input sources:**
- `eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md`
- `eusolicit-docs/planning-artifacts/epics/E05-data-pipeline-ingestion.md`
- `eusolicit-docs/test-artifacts/test-design-epic-05.md`
- `eusolicit-docs/test-artifacts/traceability-matrix.md`
- `eusolicit-docs/test-artifacts/nfr-report-epic-05.md`
- `eusolicit-docs/test-artifacts/atdd-checklist-5-{6..12}.md` (7 checklists)
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml`
- `eusolicit-docs/implementation-artifacts/deferred-work.md`
- `eusolicit-docs/test-artifacts/retrospective-epic-4.md` (carry-forward context)

<!-- Powered by BMAD-CORE -->
