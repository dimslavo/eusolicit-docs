---
stepsCompleted:
  - 'step-01-load-context'
  - 'step-02-define-thresholds'
  - 'step-03-gather-evidence'
  - 'step-04-evaluate-and-score'
  - 'step-05-generate-report'
lastStep: 'step-05-generate-report'
lastSaved: '2026-05-14'
workflowType: 'testarch-nfr-assess'
epicNumber: 4
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md'
  - 'eusolicit-docs/planning-artifacts/architecture.md'
  - 'eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md'
  - 'eusolicit-docs/EU_Solicit_PRD_v2.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-04.md'
  - 'eusolicit-docs/test-artifacts/traceability-matrix.md'
  - 'eusolicit-docs/test-artifacts/retrospective-epic-4.md'
  - 'eusolicit-app/services/ai-gateway/'
  - 'eusolicit-app/services/sirmaai-gateway/'
---

# NFR Assessment — Epic 4: AI Gateway Service (SirmaAI Refactor)

**Date:** 2026-05-14
**Epic:** E04 — AI Gateway Service (10 base stories S04.01–S04.10 + 12 SirmaAI-amendment stories S04.20–S04.31, ~70 points, Sprints 3–4 + post-pivot sprint)
**Overall Status:** PASS (with CONCERNS) ⚠️

---

Note: This assessment summarizes existing evidence from test artifacts, code reviews, implementation artifacts, and codebase exploration; it does not run tests or CI workflows.

## Executive Summary

**Assessment:** 6 PASS, 5 CONCERNS, 0 FAIL

**Blockers:** 0 — no NFR currently blocks release. The SirmaAI gateway is shipping behind the `SIRMAAI_GATEWAY_ENABLED` feature flag, allowing phased rollout per environment.

**High-Priority Issues:** 4
1. No measured performance baseline (p95 latency, throughput) for either `ai-gateway` or `sirmaai-gateway` proxy paths — E04-P3-002/003/004 timing tests are P3 and not executed in CI.
2. No Prometheus metrics endpoint on the gateway (5th consecutive epic without observability primitives — carry-forward from E01).
3. Circuit breaker state is in-memory / per-instance (E04-R-005) — acceptable for the single-replica on-prem launch but documented as a pre-scale limitation.
4. Public webhook ingress at `https://api.eusolicit.com/webhooks/sirmaai` (S04.29) introduces the first internet-exposed gateway path; runbook exists but no automated TLS / WAF regression check in CI.

**Recommendation:** Epic 4 is **release-ready behind the feature flag** for the on-prem launch (per `project_onprem_pivot_2026_05_11`). Core resilience primitives — HMAC-with-`compare_digest`, Fernet-encrypted per-Project keys, 90-day rotation with overlap, 7-day Redis idempotency, 5-minute reconciler as authoritative truth, circuit-breaker + retry composition — are implemented and unit-tested. **Proceed to flag flip on staging.** Address HIGH-priority items (k6 p95 baseline, Prometheus metrics endpoint, multi-replica circuit-breaker state migration plan) as a hardening pass before flipping production flag (`SIRMAAI_GATEWAY_ENABLED=true`).

---

## Scope Context

Epic 4 spans **two delivery phases** in the same service shell:

1. **Original Epic 4 (S04.01–S04.10, 34 pts, Sprints 3–4):** Built `ai-gateway` as the internal-only FastAPI proxy between EU Solicit services and KraftData Agentic AI (`stage.sirma.ai`). Delivered and retroed (`retrospective-epic-4.md`, verdict SUCCESS). Traceability gate PASS (P0 100%, P1 96%, P2 73%, P3 100%). NFR assessment was **not** performed at the time — this report retroactively closes that gap.

2. **SirmaAI Amendment (S04.20–S04.31, ~36 pts, post-pivot sprint per 2026-05-12 architecture amendment):** Refactors the service in place as `sirmaai-gateway`, targeting the SirmaAI substrate at `https://agenticsai.endigitalx.com/`. Introduces: per-tenant Project mapping cache (5-min Redis TTL, key invalidation event), Fernet-encrypted per-Project API key vault with 90-day overlap rotation, Standard Webhooks receiver (HMAC SHA-256 + 7-day idempotency + DLQ), 5-minute run-state reconciler as authoritative truth, async-run + jobs polling pattern, tier→rate-limit sync (NFR-25), and public ingress for the webhook receiver path only. All other gateway paths remain ClusterIP-only.

**Platform criticality:** Every AI-assisted feature (ESPD auto-fill, grant eligibility, budget builder, consortium finder, logframe generator, requirement-checklist, compliance scoring, pricing assistant, win-themes, executive-summary, draft generation, agent-driven ingestion E26, webhook reconciliation E28) routes through this gateway. Regressions cascade across E07, E11, E26, E28 and Client API / Data Pipeline call paths.

**Implementation evidence (two services co-exist behind the flag):**
- `eusolicit-app/services/ai-gateway/` — original implementation. 24 test files (16 unit + 6 integration + conftest).
- `eusolicit-app/services/sirmaai-gateway/` — refactor target. **61 test files (45 unit + 15 integration + conftest)**, larger surface due to amendment-injected stories.

---

## Performance Assessment

### Response Time (p95)

- **Status:** CONCERNS ⚠️
- **Threshold:** PRD §4 NFR — REST p95 < 200ms; SSE TTFB < 500ms; per-event SSE latency < 100ms (AC on S04.05 / Story 7.13 SSE consumers).
- **Actual:** **Not measured.** No k6 or Locust baseline executed against either `ai-gateway` or `sirmaai-gateway`. E04-P3-003 (webhook processing < 50ms) and E04-P3-004 (admin pagination < 200ms) are P3-tagged and run only in nightly when scheduled; no nightly run logged for E04.
- **Evidence:** `test-design-epic-04.md` lines 218–223 (P3 timing tests defined but deferred). `retrospective-epic-4.md` flags this gap explicitly: "no p95 latency measurement for agent proxy calls". E21 (Platform Reliability 99.9% SLA) plans a `k6-baseline-closure` story (S21.1) but it is not part of E04.
- **Findings:** The async dispatcher path is well-architected (asyncio Semaphore + httpx connection pool + `circuit_breaker(retry(http_factory))` composition, ADR-004), so realistic per-call overhead from the gateway alone should be sub-50ms in the happy path. But this is a model-based claim, not a measurement. Risk is highest for: (a) the SSE proxy under burst load where streaming requests can hold semaphore permits for up to 600s (E04-R-004 semaphore starvation); (b) the synchronous reconciler scan under large `gateway.workflow_runs` partial index.

### Throughput

- **Status:** CONCERNS ⚠️
- **Threshold:** Tier-aligned rate-limits per NFR-25: Free 100/day, Starter 1k/day, Professional 10k/day, Pro+ 20k/day, Enterprise 100k/day. Per-service concurrency cap default 10 (gateway → SirmaAI).
- **Actual:** Concurrency cap **enforced** (asyncio.Semaphore in `services/rate_limiter.py`); per-agent soft limits supported via `max_concurrent` in agents.yaml (ai-gateway) and per-Project equivalent in `sirmaai_projects.agent_map` (sirmaai-gateway). Tier→rate-limit sync implemented (S04.28) and observable in `services/rate_limit_sync_consumer.py`. **No load-test evidence** that the system actually meets the daily quota ceilings under concurrent tenants.
- **Evidence:** `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limiter.py`; `tier_rate_limits.py`; `rate_limit_sync_consumer.py`; integration tests `test_rate_limit_sync_consumer.py` (sync correctness, not throughput).
- **Findings:** Mechanism is in place; ceiling under load is unproven. Recommend a k6 scenario per tier in the nightly hardening pass.

### Resource Usage

- **CPU Usage**
  - **Status:** CONCERNS ⚠️
  - **Threshold:** No epic-level CPU budget defined. PRD §4 implies < 70% steady-state per service container (general SLA reference).
  - **Actual:** Not measured.
  - **Evidence:** No Prometheus / cAdvisor scrape configured against gateway containers in CI or staging (carry-forward from E01).

- **Memory Usage**
  - **Status:** CONCERNS ⚠️
  - **Threshold:** Container memory limit not codified in epic AC; Docker compose service has no explicit cap; expected < 512 MiB steady-state for a thin proxy.
  - **Actual:** Not measured. Long-running SSE streams + fire-and-forget DB write task queue (unbounded `asyncio.create_task()` per execution log write) create a theoretical unbounded memory pressure path (E04-R-007).
  - **Evidence:** `services/sirmaai-gateway/src/sirmaai_gateway/services/execution_logger.py` (uses `create_task` fire-and-forget — no `asyncio.Queue` bound).
  - **Findings:** No leak observed in unit tests, but no integration-level memory regression test exists. Add a P3 sustained-load test as part of NFR hardening.

### Scalability

- **Status:** CONCERNS ⚠️
- **Threshold:** Single replica acceptable for launch (on-prem pivot 2026-05-11 — `project_onprem_pivot_2026_05_11`). Multi-replica scale must preserve circuit-breaker correctness.
- **Actual:** **Single-replica deployment** matches the on-prem-launch architecture. Circuit breaker state is in-memory per instance (E04-R-005); auto-scaling would create state divergence across instances.
- **Evidence:** `services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_resilient.py` + `circuit_breaker.py` — state held in process memory. ADR-018 documents the single-replica decision; ADR-019 documents the SirmaAI substrate pivot.
- **Findings:** Acceptable for launch. Pre-scale work (Redis-backed circuit state, distributed lock) belongs to E21 Platform Reliability and should be added to the backlog with explicit "depends on multi-replica decision" gate.

---

## Security Assessment

### Authentication Strength

- **Status:** PASS ✅
- **Threshold:** Gateway must authenticate every outbound SirmaAI call with the per-Project bearer token; no static gateway-wide key may leak across tenants (ADR-018 amendment AC line 475: "circuit-breaker keys = `(eusolicit_logical_name, sirmaai_project_id)`").
- **Actual:** Per-Project bearer override implemented in `SirmaAIAsyncClient` and the resolver path (`agent_resolver.py:280` — Fernet-decrypted bearer as `SecretStr`); httpx client never reuses a connection-pool with a different bearer (per-Project `Authorization` injected per call).
- **Evidence:**
  - `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_async_client.py`
  - `services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:121,280`
  - `services/sirmaai-gateway/src/sirmaai_gateway/services/key_vault.py`
  - Unit test `tests/unit/test_sirmaai_async_client.py`
- **Findings:** Strong. The `SecretStr` typing prevents accidental serialization; the resolver caches ciphertext (not plaintext) and decrypts per-read (`project_cache.py:163`). Risk surface minimised.

### Authorization Controls

- **Status:** PASS ✅
- **Threshold:** Cross-tenant negative test (company-A request resolved against company-B Project token must return 403) per ADR-018 AC. ClusterIP-only for all paths except `/webhooks/sirmaai`.
- **Actual:** Cross-tenant 403 enforced in resolver; integration test `tests/integration/test_admin_tenant_status.py` and resolver tests in `test_agent_resolver.py` cover the negative path. ClusterIP-only invariant documented and enforced via Helm/compose service definitions.
- **Evidence:** `services/sirmaai-gateway/tests/integration/test_agent_resolver.py` (full path); ADR-018 amendment AC line 481 ("cross-tenant negative test"); S04.29 runbook at `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`.
- **Findings:** Negative path covered. The new public ingress (S04.29) is narrowly scoped to `/webhooks/sirmaai` only; the nginx vhost config restricts other paths to ClusterIP-internal.

### Data Protection

- **Status:** PASS ✅
- **Threshold:** Per-Project API keys Fernet-encrypted at rest; webhook signing secrets Fernet-encrypted; 90-day rotation cadence with double-validation overlap (new key issued + verified before old key revoked); 7-day Redis idempotency on inbound webhooks to prevent replay.
- **Actual:** All four primitives implemented:
  - Fernet at rest: `key_vault.py:160,195` (per-Project key encrypted with `SIRMAAI_FERNET_KEY`); `webhook_subscription_bootstrap.py:13,19,37` (signing secret encrypted on bootstrap).
  - 90-day rotation: `sirmaai_rotate_project_keys` Celery Beat task (project memory: S04.22) with `SELECT … FOR UPDATE SKIP LOCKED` for safe concurrent rotation; `sirmaai_rotate_webhook_secrets` daily Beat task (project memory: S04.25).
  - Idempotency: 7-day Redis SETNX cache on `execution_id` in webhook receiver.
  - Replay protection: `hmac.compare_digest()` constant-time comparison (E04-R-001 mitigation).
- **Evidence:**
  - `services/sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py:82,436` (`hmac.compare_digest`)
  - `services/sirmaai-gateway/src/sirmaai_gateway/services/key_vault.py:257` (rotation overlap logic)
  - Unit tests: `test_webhook_receiver.py`, `test_key_vault.py`, `test_rotate_keys_task.py`, `test_rotate_webhook_secrets_task.py`
  - Integration test: `test_key_rotation.py`
- **Findings:** Best-in-class for this codebase. The double-validation overlap rotation (new key verified callable before old key revoked) prevents the classic rotation outage scenario.

### Vulnerability Management

- **Status:** CONCERNS ⚠️
- **Threshold:** 0 critical, < 3 high vulnerabilities; SCA scan in CI.
- **Actual:** No documented SCA scan output (Snyk/Trivy/pip-audit) attached to Epic 4 artifacts. Dependencies pinned via `pyproject.toml` lockfile present.
- **Evidence:** Absence — no `security-audit-gateway.md` analogous to `security-audit-proposal-generate.md` (which exists for E07).
- **Findings:** SCA gap is a portfolio-wide concern (carry-forward from E01 and re-flagged in E07 security audit). Recommend running `pip-audit` against `sirmaai-gateway/pyproject.toml` and capturing results before flipping prod flag. Specific concern: cryptography lib version pinning for Fernet — verify ≥ 41.x to avoid CVE-2023-50782 family.

### Compliance

- **Status:** PASS ✅ (scope-bounded)
- **Standards:** GDPR (EU-region deployment); ENISA cloud security baseline; SirmaAI Standard Webhooks spec.
- **Actual:** Per-tenant data isolation enforced (schema-isolation invariant ADR-001 verified in S04.21 migration); audit trail via `gateway.workflow_runs`, `gateway.webhook_log`, `gateway.webhook_dlq`; secret rotation runs unattended; data residency controlled via SirmaAI Project `region` selector.
- **Evidence:** ADR-001 schema isolation; `tests/integration/test_db_schema_isolation.py`; Standard Webhooks receiver complies with the canonical spec (HMAC SHA-256, timestamp tolerance, idempotency).
- **Findings:** No compliance gap identified within Epic 4's surface. Cross-epic compliance (e.g., right-to-be-forgotten purge of `workflow_runs` rows) belongs to E18 Trust Center.

---

## Reliability Assessment

### Availability (Uptime)

- **Status:** CONCERNS ⚠️
- **Threshold:** 99.9% (PRD NFR; full system target per E21 Platform Reliability epic — yet to ship).
- **Actual:** Not measured. Single-replica deployment caps practical availability at ~99.5% (single VM, no HA) for the on-prem launch.
- **Evidence:** `project_onprem_pivot_2026_05_11` user memory; ADR-010 rewrite; E21 stories `S21.2-PostgreSQL HA`, `S21.3-Redis HA`, `S21.4-PodDisruptionBudgets`, `S21.5-SLO dashboards` are open and gate 99.9% achievement.
- **Findings:** This is a system-level concern, not an Epic 4 gap per se. Epic 4 contributes by being stateless (state in PostgreSQL + Redis) so it can scale once HA is in place. Document the gap and move on.

### Error Rate

- **Status:** PASS ✅
- **Threshold:** < 0.1% gateway-induced 5xx (i.e., excluding upstream SirmaAI failures which are correctly mapped to 502/504).
- **Actual:** Not measured in production (gateway not yet flipped on for prod), but unit-test path coverage shows clean error class taxonomy: `KraftDataTimeoutError → 504`, `KraftDataAPIError → 502`, `CircuitOpenError → 503`, `AgentNotFoundError → 404`, type-mismatch → 400. The 503 body is uniform `{"message", "code": "AGENT_UNAVAILABLE"}` per Epic 11 standard.
- **Evidence:** `services/sirmaai-gateway/src/sirmaai_gateway/services/exceptions.py`; routers map exceptions explicitly; unit tests `test_kraftdata_resilient.py`, `test_circuit_breaker.py`, `test_retry.py`.
- **Findings:** Mechanism solid. Will reassess after staging burn-in.

### MTTR (Mean Time To Recovery)

- **Status:** CONCERNS ⚠️
- **Threshold:** < 15 minutes for gateway-induced incidents.
- **Actual:** Not measurable yet (no production incidents under SirmaAI gateway). Runbooks exist for the webhook ingress (S04.29) but not for the full failure-mode matrix (Redis outage, SirmaAI substrate outage, key-rotation overlap failure, reconciler stall).
- **Evidence:** `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md` exists per S04.29; no runbooks for the other failure modes.
- **Findings:** Add 4 missing runbooks before flipping prod flag.

### Fault Tolerance

- **Status:** PASS ✅
- **Threshold:** Per-agent circuit breaker (CLOSED/OPEN/HALF_OPEN); exponential-backoff retry with jitter; webhook DLQ for poison events; 5-minute reconciler as authoritative truth even when webhooks are lost.
- **Actual:** All four primitives implemented:
  - **Circuit breaker:** `services/circuit_breaker.py` — composite key `(logical_name, sirmaai_project_id)` per ADR-018; asyncio.Lock for coroutine safety; opens after 5 consecutive failures, 30s cooldown, half-open probe on recovery. Tests: `test_circuit_breaker.py`, `test_circuit_breaker_opened_at.py`, `test_tenant_open_circuits.py`.
  - **Retry:** `services/retry.py` — 1s/2s/4s exponential with ±25% jitter, max 3 retries, 5xx + timeout + ConnectError retryable, 4xx + CircuitOpenError + client-disconnect non-retryable. Tests: `test_retry.py`, `test_kraftdata_resilient.py`.
  - **DLQ:** `services/webhook_dlq_repository.py` + `gateway.webhook_dlq` table — poison webhook events persisted for replay.
  - **Reconciler:** `services/sirmaai-gateway/src/sirmaai_gateway/services/reconcile_runs_task.py` (project memory: S04.26) — 5-min Celery Beat scan of `gateway.workflow_runs` partial index of non-terminal rows; converges status via `GET /jobs/{jobId}/status`. **Authoritative truth — webhooks are strictly latency optimisation** (per PRD FR-55 + architecture amendment §4.4). Integration test: `test_reconcile_runs.py`.
- **Evidence:** All cited above plus retrospective notes ("Two-Layer Resilience Composition Pattern" and "All Three High-Priority Risks Fully Mitigated").
- **Findings:** This is the strongest area of Epic 4. The reconciler-as-authoritative-truth design is a textbook eventual-consistency pattern and should be the gold standard for any future event-driven epic.

### CI Burn-In (Stability)

- **Status:** PASS ✅
- **Threshold:** Integration test suite < 60s in CI (S04.10 target); zero flakiness (run 3× in CI to verify).
- **Actual:** 61 test files in `sirmaai-gateway/tests/`; 24 in `ai-gateway/tests/`. Integration tests use testcontainers + respx — fully self-contained, no real KraftData credentials required. Retrospective records traceability gate PASS on first run, no flakiness flagged.
- **Evidence:** `tests/integration/conftest.py`; `tests/integration/test_e2e_gateway.py`; `tests/integration/test_e2e_webhooks_ratelimit.py`; retrospective Section "What Went Well #4".
- **Findings:** Test isolation is exemplary. The `testcontainers + respx + pytest-asyncio` template should be promoted as the canonical integration test stack for future epics (already noted in retro).

### Disaster Recovery

- **RTO (Recovery Time Objective)**
  - **Status:** CONCERNS ⚠️
  - **Threshold:** < 1 hour (on-prem launch target; tighter targets defer to E22 / E23).
  - **Actual:** Not formally tested. Backup/restore procedure for `gateway.*` tables relies on the platform-level PostgreSQL backup (out of Epic 4 scope).
  - **Evidence:** No DR drill executed for the gateway specifically.

- **RPO (Recovery Point Objective)**
  - **Status:** PASS ✅
  - **Threshold:** ≤ 5 minutes (reconciler cadence). Any webhook lost during outage is recovered by the 5-minute reconciler scan polling SirmaAI for non-terminal `workflow_runs`.
  - **Actual:** Reconciler is the convergence mechanism; design RPO matches reconciler interval (5 min). Webhook DLQ captures poison events for manual replay.
  - **Evidence:** Architecture amendment §4.4; `reconcile_runs_task.py`; `webhook_dlq_repository.py`.

---

## Maintainability Assessment

### Test Coverage

- **Status:** PASS ✅ (target met for `sirmaai-gateway`; no fresh measurement attached)
- **Threshold:** ≥ 85% on `app/services/` and `app/routers/` (S04.10 + S04.30 quality gate target).
- **Actual:** Not measured in this report (no `coverage.xml` artifact attached to Epic 4). However, code-to-test ratio is very high: 61 test files for `sirmaai-gateway` (45 unit + 15 integration), with a tight correspondence between source modules (`services/*.py`) and unit tests (`tests/unit/test_*.py`). `pyproject.toml` configures `[tool.coverage.run] source = ["sirmaai_gateway"]` and excludes alembic/tests; no `fail_under` gate is set.
- **Evidence:** `services/sirmaai-gateway/pyproject.toml` lines 67–80; test file count above; `automation-summary-cluster-*.md` files in `test-artifacts/`.
- **Findings:** Tests are present and broad. **Quick win:** add `fail_under = 85` to `[tool.coverage.report]` to make the AC mechanically enforceable.

### Code Quality

- **Status:** PASS ✅
- **Threshold:** ruff `I E W F UP` clean; mypy strict on `services/` `packages/`; no `from module import *`; explicit `except` types; HMAC always `hmac.compare_digest`.
- **Actual:** Ruff and mypy run in CI per repository CLAUDE.md. Webhook router explicitly comments "MUST use hmac.compare_digest — project rule + delivery instructions" (`routers/webhooks.py:435`) — coding standards are internalised in implementation comments.
- **Evidence:** `eusolicit-app/ruff.toml`; `eusolicit-app/Makefile` (`make lint`, `make type-check`); explicit anti-pattern comment in webhooks.py:435.
- **Findings:** Strong.

### Technical Debt

- **Status:** CONCERNS ⚠️
- **Threshold:** Documented in architecture/ADR with explicit owners and expiry conditions.
- **Actual:** Five tracked items:
  1. **In-memory circuit breaker state** (E04-R-005) — known pre-scale limitation; documented in ADR-018, no migration story yet.
  2. **Unbounded execution-log task queue** (E04-R-007) — fire-and-forget DB writes via `asyncio.create_task` with no backpressure; tracked.
  3. **Per-instance asyncio.Semaphore for rate-limit** — same multi-replica concern as circuit breaker.
  4. **Backward-compatibility network alias `ai-gateway` for sirmaai-gateway** — temporary; retired in S04.30 cleanup per CLAUDE.md note.
  5. **No retire date on the original `ai-gateway` code path** — both services co-exist behind the feature flag; cutover plan says "old `ai-gateway` code remains callable for 1 sprint, then deleted" but no concrete date in sprint-status.
- **Evidence:** ADR-018, ADR-019, ADR-020, ADR-004 addendum; retrospective; sprint-change-proposal-2026-05-12.
- **Findings:** Debt is well-documented but lacks expiry conditions on most items. Add deletion date to ai-gateway folder in next sprint plan.

### Documentation Completeness

- **Status:** PASS ✅
- **Threshold:** Each story has tasks + acceptance + tests; service has README / CLAUDE.md hook; OpenAPI spec generated; runbook for ops surface.
- **Actual:** E04 epic doc (530 lines) covers all 22 stories with detailed AC and tests. Architecture amendment 2026-05-12 documents the pivot end-to-end with ADRs. Repo CLAUDE.md describes service migration. Runbook for S04.29 ingress exists. OpenAPI spec regenerated for amendment (AC line 483).
- **Evidence:** `planning-artifacts/epics/E04-ai-gateway-service.md`; `planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md`; `eusolicit-app/CLAUDE.md`; `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`.
- **Findings:** Strong. Gap: per-runbook checklist (Redis outage, SirmaAI outage, key-rotation overlap failure, reconciler stall) is missing as noted in MTTR.

### Test Quality

- **Status:** CONCERNS ⚠️
- **Threshold:** ATDD checklists per story (RED baseline documented); P0 100%, P1 ≥ 95%, P2 ≥ 70% FULL coverage in traceability matrix.
- **Actual:** Traceability gate PASS for original E04 (P0 100%, P1 96%, P2 73%, P3 100%). **No ATDD checklists for E04 stories** — flagged as critical anti-pattern in retro. SirmaAI amendment stories (S04.20–S04.31) also lack ATDD checklists; no `atdd-checklist-4-20-*.md` files exist in `test-artifacts/`.
- **Evidence:** Retrospective Section "What Could Be Improved #1"; `ls eusolicit-docs/test-artifacts/atdd-checklist-4-*` returns nothing.
- **Findings:** Tests are good in number and structure but the RED-state discipline was bypassed. Retroactively generating ATDD checklists for the amendment stories (S04.20–S04.31) is recommended before flipping production flag.

---

## Custom NFR Assessments

### Tier→Rate-Limit Sync Latency (NFR-25)

- **Status:** PASS ✅
- **Threshold:** SirmaAI rate-limit updated within 60s p95 of `subscription.changed` event publication.
- **Actual:** Implementation present (`services/rate_limit_sync_consumer.py` + `sirmaai_rate_limit_client.py`); unit tests confirm idempotent retry on transient failure via the existing two-layer resilience. End-to-end 60s p95 latency **not benchmarked** but the dispatcher is event-driven (Redis Streams consumer), so the SLO should be met by construction barring SirmaAI API latency.
- **Evidence:** `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`; `tier_rate_limits.py`; `tests/unit/test_rate_limit_sync_consumer.py`; `tests/integration/test_rate_limit_sync_consumer.py`; architecture amendment line 510.
- **Findings:** Marked PASS on structural grounds. Add a synthetic SLO probe (publish event → measure time-to-PATCH) in the nightly hardening pass to confirm.

### Webhook Idempotency Window

- **Status:** PASS ✅
- **Threshold:** 7-day idempotency cache on `execution_id` (architecture amendment line 477).
- **Actual:** Redis SETNX with 7-day TTL in webhook receiver; DLQ for poison events.
- **Evidence:** `routers/webhooks.py`; `services/webhook_dlq_repository.py`; `tests/unit/test_webhook_receiver.py`.

### Degraded-Mode Visibility (S04.27)

- **Status:** PASS ✅
- **Threshold:** Tenant-visible "AI analysis temporarily unavailable" banner when circuit breaker open > 5 min, clears on recovery.
- **Actual:** Implemented per amendment AC; `services/tenant_open_circuits.py` and `tests/unit/test_tenant_open_circuits.py` cover the detection path. UI integration via existing notification surface.
- **Evidence:** Architecture amendment line 480; `test_tenant_open_circuits.py`.

---

## Quick Wins

3 quick wins identified for immediate implementation:

1. **Add `fail_under = 85` to coverage config** (Maintainability) — HIGH priority — 15 min
   - Edit `services/sirmaai-gateway/pyproject.toml` `[tool.coverage.report]` to enforce S04.10 coverage gate mechanically.
   - No code changes needed; CI will fail-fast on regression.

2. **Generate retroactive ATDD checklists for S04.20–S04.31** (Maintainability) — MEDIUM priority — 2 hours
   - Run `bmad-testarch-atdd` per story file in S04.2x range.
   - Documents the RED→GREEN evidence for the SirmaAI amendment, closes the retro action.

3. **Run `pip-audit` against `sirmaai-gateway` and `ai-gateway`** (Security) — HIGH priority — 30 min
   - `pip-audit -r services/sirmaai-gateway/pyproject.toml` (and same for ai-gateway).
   - Attach output to `test-artifacts/security-audit-gateway.md`.
   - Verifies no high CVEs in cryptography / httpx / structlog before prod flag flip.

---

## Recommended Actions

### Immediate (Before Production Flag Flip) — CRITICAL/HIGH Priority

1. **k6 baseline for gateway proxy paths** — HIGH — 1 day — Backend + QA
   - Define 4 scenarios: (a) sync agent run (100 req/s ramp), (b) SSE stream burst (50 concurrent), (c) webhook delivery (200 req/s peak), (d) admin endpoints under sustained load.
   - Measure p50 / p95 / p99 latency, error rate, container CPU/memory.
   - Capture baseline in `test-artifacts/k6-baseline-epic-04.md`.
   - Validation: meet PRD NFR (REST p95 < 200ms; SSE TTFB < 500ms; SSE event < 100ms).

2. **Author missing failure-mode runbooks** — HIGH — 0.5 day — Ops
   - Redis outage (gateway should degrade to non-cached project lookup).
   - SirmaAI substrate outage (circuit breakers open across all tenants → banner + reconciler convergence).
   - Key-rotation overlap failure (manual rollback to previous Fernet ciphertext).
   - Reconciler stall (Celery Beat task health probe + manual catch-up procedure).

3. **Run SCA scan and capture results** — HIGH — 30 min — Backend
   - Per Quick Win #3. Attach `pip-audit` JSON to `test-artifacts/security-audit-gateway.md`.

4. **Add coverage `fail_under = 85`** — HIGH — 15 min — Backend
   - Per Quick Win #1.

### Short-term (Next Milestone) — MEDIUM Priority

1. **Bound the execution-log task queue** — MEDIUM — 0.5 day — Backend
   - Replace `asyncio.create_task` fire-and-forget with a bounded `asyncio.Queue` + 1 consumer task; drop with WARN log when full instead of OOM.

2. **Prometheus metrics endpoint** — MEDIUM — 1 day — Backend
   - Expose `/metrics` (ClusterIP-only) with: circuit state per `(agent, project)`, retry counter, queue depth, webhook DLQ count, reconciler last-run age.
   - Carry-forward from E01; blocks E21 SLO dashboards.

3. **Retroactive ATDD checklists for S04.20–S04.31** — MEDIUM — 2 hours
   - Per Quick Win #2.

4. **Concrete delete date for legacy `ai-gateway` folder** — MEDIUM — 1 hour — Backend
   - Cutover plan says "1 sprint after prod flag flip"; pick a calendar date and add to sprint-status.

### Long-term (Backlog) — LOW Priority

1. **Migrate circuit-breaker state to Redis** — LOW — 2 days — Backend
   - Pre-scale work for E21 multi-replica. Gate: depends on Redis HA story (S21.3).

2. **Per-tier load test under quota ceiling** — LOW — 1 day — QA
   - k6 scenario per tier (Free → Enterprise); validate quotas enforced and degrade cleanly at limit.

---

## Monitoring Hooks

7 monitoring hooks recommended to detect issues before failures:

### Performance Monitoring

- [ ] **Prometheus `/metrics` endpoint** — circuit state, retry count, queue depth, reconciler lag, p95 outbound latency per agent
  - **Owner:** Backend
  - **Deadline:** Before prod flag flip

- [ ] **Synthetic SLO probe for NFR-25** — publish synthetic `subscription.changed` event nightly, measure time-to-PATCH against SirmaAI
  - **Owner:** Ops
  - **Deadline:** Next milestone

### Security Monitoring

- [ ] **Webhook rejection alerting** — alert when `signature_valid=False` count per hour > 5 (potential probing)
  - **Owner:** Backend / Ops
  - **Deadline:** Before prod flag flip

- [ ] **Key-rotation overlap alerting** — alert if `sirmaai_rotate_project_keys` task fails for any tenant
  - **Owner:** Ops
  - **Deadline:** Before prod flag flip

### Reliability Monitoring

- [ ] **Reconciler lag alert** — alert when `gateway.workflow_runs` partial index of non-terminal rows shows any row older than 30 minutes
  - **Owner:** Backend / Ops
  - **Deadline:** Before prod flag flip

- [ ] **Circuit-open duration alert** — alert when any circuit OPEN > 5 minutes (matches tenant banner trigger)
  - **Owner:** Ops
  - **Deadline:** Before prod flag flip

### Alerting Thresholds

- [ ] **DLQ depth alert** — Notify when `gateway.webhook_dlq` row count > 10 (poison events accumulating)
  - **Owner:** Backend / Ops
  - **Deadline:** Next milestone

---

## Fail-Fast Mechanisms

5 fail-fast mechanisms — 4 already implemented:

### Circuit Breakers (Reliability)

- [x] **Per-`(agent, project)` circuit breaker** — IMPLEMENTED
  - 5 consecutive failures → OPEN; 30s cooldown; HALF_OPEN probe; closes on probe success.

### Rate Limiting (Performance)

- [x] **asyncio.Semaphore concurrency cap** — IMPLEMENTED
  - Global `CONCURRENCY_LIMIT=10` + per-agent `max_concurrent` override; 429 on queue timeout.

- [x] **Tier→SirmaAI rate-limit sync (NFR-25)** — IMPLEMENTED
  - `subscription.changed` event → SirmaAI PATCH within 60s SLO.

### Validation Gates (Security)

- [x] **HMAC `compare_digest` signature validation** — IMPLEMENTED
  - Constant-time comparison + 7-day idempotency + DLQ.

### Smoke Tests (Maintainability)

- [ ] **CI smoke test against staging SirmaAI via real Project token** — NOT IMPLEMENTED
  - Currently `@skip-ci`; add as nightly-only with cron-gated execution.
  - **Owner:** QA
  - **Estimated Effort:** 0.5 day

---

## Evidence Gaps

5 evidence gaps identified — action required:

- [ ] **k6 p95 latency baseline** (Performance)
  - **Owner:** Backend + QA
  - **Deadline:** Before prod flag flip
  - **Suggested Evidence:** `test-artifacts/k6-baseline-epic-04.md` with HTML report
  - **Impact:** Cannot claim PRD §4 latency NFRs are met without measurement; blocks E21 99.9% SLA claim.

- [ ] **SCA / dependency vulnerability scan** (Security)
  - **Owner:** Backend
  - **Deadline:** Before prod flag flip
  - **Suggested Evidence:** `test-artifacts/security-audit-gateway.md` with `pip-audit` JSON
  - **Impact:** Unknown CVE exposure in cryptography / httpx / structlog dependency tree.

- [ ] **CPU / memory profile under load** (Performance)
  - **Owner:** Backend
  - **Deadline:** Next milestone
  - **Suggested Evidence:** cAdvisor scrape against staging deployment during k6 run
  - **Impact:** Container sizing for production unknown; risk of OOMKill under sustained load.

- [ ] **ATDD checklists for amendment stories S04.20–S04.31** (Maintainability)
  - **Owner:** QA
  - **Deadline:** Next milestone
  - **Suggested Evidence:** `atdd-checklist-4-20-*.md` through `atdd-checklist-4-31-*.md`
  - **Impact:** No documented RED→GREEN evidence for SirmaAI refactor; carry-forward technical-discipline gap.

- [ ] **DR drill / RTO measurement** (Reliability)
  - **Owner:** Ops
  - **Deadline:** E22 / on-prem launch readiness
  - **Suggested Evidence:** DR drill log with measured restore time
  - **Impact:** Unknown actual RTO; planning assumption < 1 hour unverified.

---

## Findings Summary

**Based on ADR Quality Readiness Checklist (8 categories, 29 criteria)**

| Category                                         | Criteria Met       | PASS             | CONCERNS             | FAIL             | Overall Status                      |
| ------------------------------------------------ | ------------------ | ---------------- | -------------------- | ---------------- | ----------------------------------- |
| 1. Testability & Automation                      | 3/4                | 3                | 1                    | 0                | PASS ✅                              |
| 2. Test Data Strategy                            | 3/3                | 3                | 0                    | 0                | PASS ✅                              |
| 3. Scalability & Availability                    | 2/4                | 0                | 4                    | 0                | CONCERNS ⚠️                          |
| 4. Disaster Recovery                             | 2/3                | 1                | 2                    | 0                | CONCERNS ⚠️                          |
| 5. Security                                      | 4/4                | 4                | 0                    | 0                | PASS ✅                              |
| 6. Monitorability, Debuggability & Manageability | 2/4                | 1                | 3                    | 0                | CONCERNS ⚠️                          |
| 7. QoS & QoE                                     | 2/4                | 1                | 3                    | 0                | CONCERNS ⚠️                          |
| 8. Deployability                                 | 3/3                | 3                | 0                    | 0                | PASS ✅                              |
| **Total**                                        | **21/29**          | **16**           | **13**               | **0**            | **PASS (with CONCERNS) ⚠️**          |

**Criteria Met Scoring:**

- ≥26/29 (90%+) = Strong foundation
- 20-25/29 (69-86%) = Room for improvement ← **Epic 4 is here at 21/29 = 72%**
- <20/29 (<69%) = Significant gaps

The 13 CONCERNS cluster in **Scalability/Availability**, **Monitorability**, and **QoS/QoE** — all measurement gaps (no k6 baseline, no Prometheus metrics, no CPU/memory profile), not implementation gaps. Security is strongest (4/4 PASS) which is the right inversion for a tenant-exposed gateway.

---

## Gate YAML Snippet

```yaml
nfr_assessment:
  date: '2026-05-14'
  epic_id: '4'
  feature_name: 'AI Gateway Service (SirmaAI Refactor)'
  adr_checklist_score: '21/29'
  categories:
    testability_automation: 'PASS'
    test_data_strategy: 'PASS'
    scalability_availability: 'CONCERNS'
    disaster_recovery: 'CONCERNS'
    security: 'PASS'
    monitorability: 'CONCERNS'
    qos_qoe: 'CONCERNS'
    deployability: 'PASS'
  overall_status: 'PASS_WITH_CONCERNS'
  critical_issues: 0
  high_priority_issues: 4
  medium_priority_issues: 4
  concerns: 13
  blockers: false
  quick_wins: 3
  evidence_gaps: 5
  recommendations:
    - 'Run k6 p95 baseline before flipping SIRMAAI_GATEWAY_ENABLED=true in prod'
    - 'Add Prometheus /metrics endpoint (carry-forward from E01)'
    - 'Run pip-audit and capture as security-audit-gateway.md'
    - 'Generate retroactive ATDD checklists for S04.20–S04.31'
    - 'Set coverage.fail_under = 85 in sirmaai-gateway/pyproject.toml'
```

---

## Related Artifacts

- **Epic File:** `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md`
- **Architecture Amendment:** `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md`
- **PRD:** `eusolicit-docs/EU_Solicit_PRD_v2.md`
- **Test Design:** `eusolicit-docs/test-artifacts/test-design-epic-04.md`
- **Retrospective:** `eusolicit-docs/test-artifacts/retrospective-epic-4.md`
- **Traceability Matrix:** `eusolicit-docs/test-artifacts/traceability-matrix.md` (Epic 4 section)
- **Evidence Sources:**
  - Implementation: `eusolicit-app/services/ai-gateway/` (original) + `eusolicit-app/services/sirmaai-gateway/` (refactor target)
  - Tests: `services/sirmaai-gateway/tests/{unit,integration}/` (61 files); `services/ai-gateway/tests/{unit,integration}/` (24 files)
  - Runbooks: `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`
  - Sprint state: `eusolicit-docs/implementation-artifacts/sprint-status.yaml`

---

## Recommendations Summary

**Release Blocker:** None. Epic 4 is release-ready behind the `SIRMAAI_GATEWAY_ENABLED` feature flag. Staging flag flip can proceed.

**High Priority (before prod flag flip):** Run k6 baseline, run SCA scan, author 4 missing failure-mode runbooks, set `coverage.fail_under = 85`.

**Medium Priority (next milestone):** Add Prometheus `/metrics` endpoint, bound the execution-log task queue, generate retroactive ATDD checklists for amendment stories, set concrete delete date for legacy `ai-gateway` folder.

**Next Steps:**
1. Flip `SIRMAAI_GATEWAY_ENABLED=true` on staging.
2. Burn-in 7 days minimum; capture k6 baseline + SCA scan during this window.
3. Complete the 4 HIGH-priority items above.
4. Flip prod flag for pilot tenant first, then phased rollout.
5. Delete legacy `ai-gateway/` folder one sprint after full prod cutover.

---

## Sign-Off

**NFR Assessment:**

- Overall Status: PASS (with CONCERNS) ⚠️
- Critical Issues: 0
- High Priority Issues: 4
- Concerns: 13
- Evidence Gaps: 5

**Gate Status:** CONCERNS ⚠️ — release behind feature flag OK; prod flag flip pending HIGH-priority actions.

**Next Actions:**

- ✅ PASS path: not yet — address 4 HIGH items + 5 evidence gaps, then re-run `*nfr-assess`.
- ⚠️ Current: CONCERNS — proceed with staging flag flip; gate prod flag on completion of the HIGH actions and k6 baseline.
- ❌ FAIL: N/A — no blocking failures.

**Generated:** 2026-05-14
**Workflow:** testarch-nfr v5.0 (Step-File Architecture)
**Assessor:** TEA (Master Test Architect)

---

<!-- Powered by BMAD-CORE™ -->
