# E13: Hardening & Drift Recovery

**Sprint**: 13 | **Points**: 34 | **Dependencies**: E01–E12 | **Milestone**: Beta-readiness gate
**Source:** PRD v2.0 §6 Epic 13 + §7 Open Issues; project-context Epics 4–12 carry-forwards; sprint-change-proposal-2026-04-25 / 2026-04-26.

## Goal

Close every carry-forward `[ACTION]` item that has accumulated through Epics 1–12 — Dependabot, k6 performance baselines, Prometheus `/metrics` bootstrap, TEA review backlog, Stripe outbound circuit breaker, billing observability metrics, prompt-injection sanitization, optimistic-locking GREEN verification, ClamAV stuck-document cleanup, and frontend tier-enforcement E2E coverage — so that EU Solicit can transition from in-flight feature build-out to Beta with zero rollover technical debt and verifiable NFR conformance. Epic 13 also codifies the **coordinator-with-sub-stories** pattern (project-context Epic 13) and the **NFR hot-fix + proper story** workflow (S07.17 reference) so future hardening work follows a repeatable shape.

## Acceptance Criteria

- [ ] Dependabot configured at repo root (`.github/dependabot.yml`) covering Python ecosystems (`pyproject.toml` for all 5 services + 3 packages) and Node ecosystem (`pnpm-workspace.yaml` + `frontend/package.json`); weekly schedule; PRs auto-labelled `dependencies`; first weekly run produces ≥0 PRs without errors
- [ ] k6 performance-baseline scripts checked into `tests/load/` for: REST p95 (≤200 ms target), SSE TTFB p95 (≤500 ms target for AI Gateway streaming), PostgreSQL FTS over 10 K opportunities (≤300 ms p95), Redis usage-metering 10 K concurrent INCR (no over-count), SSE concurrency cap (10/pod); scripts executed and results committed to `tests/load/load-test-results.md`
- [ ] Prometheus `/metrics` endpoint bootstrapped on every service (`client-api`, `admin-api`, `data-pipeline`, `ai-gateway`, `notification`); REST latency histogram, SSE TTFB histogram, AI Gateway error counter, billing webhook latency, tier distribution gauge, trial-to-paid counter, `billing_usage_sync_drift_total` gauge, crawler success/failure counter all emit; `/metrics` is cluster-IP-only (not public ingress) and never echoes connection strings
- [ ] TEA test-review score ≥ 80/100 recorded as an AC on every story closed in Epic 13 (and codified as a global `done`-gate AC for all subsequent epics — NOT a separate injected story)
- [ ] R-001 (collaborative-edit data loss via optimistic locking) has ≥1 GREEN integration test exercising the 409 path with `SELECT FOR UPDATE` + SHA-256 hash comparison, structured `{server_version, client_version, diff_url}` body, and frontend conflict-resolution dialog
- [ ] Content blocks sent to AI Gateway pass through a single sanitization helper (allowlist of Markdown/TipTap nodes; strip script-like tokens) **before** persistence and **before** forwarding (E07 carry-forward; closes E07-R-005)
- [ ] Stripe outbound calls in `billing_service.py` and `vies_service.py` adopt the canonical `circuit_breaker(retry(http_factory))` resilience pattern from E04; 4xx responses do NOT increment breaker failure counter; ≥1 GREEN integration test asserts breaker opens on 5 consecutive Stripe 5xx and protects the endpoint
- [ ] `invoice.payment_failed → past_due` integration test fires a mock Stripe webhook end-to-end through the webhook endpoint and asserts DB tier transitions to `past_due` (Epic 8 carry-forward)
- [ ] ClamAV stuck-document cleanup Celery Beat task transitions documents stuck in `pending` >TTL to `failed`; ≥1 GREEN integration test simulates ClamAV outage and asserts transition (Epic 6 carry-forward)
- [ ] Frontend tier-enforcement Playwright E2E exercises Free→Starter→Professional→Enterprise tier gates across opportunity discovery, proposal export, and AI summary endpoints; cross-tier deep-links return 404 (not 403); design-system component compliance (`<Select>`, `<Dialog>`, `<Sheet>`, `<Tabs>` from `@eusolicit/ui`) source-inspected at ATDD level
- [ ] Sprint-status `epic-13` transitions to `done` only when (1) all stories `done`, (2) story-file status reconciliation lint passes (no `done` in sprint-status with `review` in story file — project-context Epic 13 anti-pattern), (3) all `[ACTION] SEVERITY: critical` items from Epics 4–12 retros marked closed in retrospective verification matrix

## Stories

### S13.01: Coordinator Story — Carry-Forward Closure Gate

**Points**: 2 | **Type**: process | **Sequence**: First — gates all other E13 stories
**Pattern:** project-context Epic 13 — *Coordinator story for multi-concern hardening epics*

Owns end-to-end AC acceptance for Epic 13. Maintains a verification matrix (`docs/epic-13/carry-forward-matrix.md`) listing every `[ACTION]` item from Epics 4–12 retros, mapped to the sub-story closing it, with status (`open` / `in-sub-story` / `green` / `closed`). Coordinator cannot transition `done` until all sub-stories are `done` AND each verification-matrix row reads `closed`. Provides the canonical artifact the orchestrator's `ChangeEvaluator` can read at retrospective time to confirm the retro-to-action feedback loop is intact (project-context Epic 8 anti-pattern fix).

**Acceptance Criteria**:
- [ ] `docs/epic-13/carry-forward-matrix.md` enumerates every E04–E12 `[ACTION]` item with severity, source epic, target sub-story, status
- [ ] Verification-matrix lint runs in CI: every `closed` row must reference a merged commit hash; every `green` row must reference a passing test path
- [ ] Coordinator story carries a `Hot-Fix Context` section listing prior emergency fixes (e.g., S07.17 commit `2d41fcf`) with explicit "verify, don't re-implement" guidance
- [ ] Coordinator's `done` transition requires reviewer to quote the matrix-lint CI output (project-context Epic 13 anti-pattern: "Review approval without quoted test execution output")

---

### S13.02: Dependabot Configuration (Python + Node)

**Points**: 1 | **Type**: devops | **Severity:** Carry-forward critical (5 consecutive epics)

Add `.github/dependabot.yml` covering all 8 Python projects (5 services + 3 shared packages) and the pnpm Node workspace. Weekly cadence, group minor + patch updates, ignore major bumps until manually reviewed, auto-label `dependencies`, target `main`. Document in `docs/dependabot.md` how to triage a Dependabot PR (lint, test, merge or escalate).

**Acceptance Criteria**:
- [ ] `.github/dependabot.yml` exists with 9 update entries: 8 Python (`pip` ecosystem per project root) + 1 Node (`npm` ecosystem at frontend root, pnpm-aware)
- [ ] First scheduled run completes without configuration errors; PRs (if any) auto-labelled `dependencies` and target `main`
- [ ] CI runs lint + type-check + unit tests on every Dependabot PR via the existing matrix workflow (E01 S01.08)
- [ ] `docs/dependabot.md` provides a triage runbook
- [ ] Verification-matrix row for "Dependabot 5-epic carry-forward" transitions to `closed`

---

### S13.03: k6 Performance Baseline Suite

**Points**: 5 | **Type**: testing | **Severity:** Carry-forward critical (6 consecutive epics)

Author k6 scripts in `tests/load/` covering five mandatory scenarios (REST p95, SSE TTFB p95, FTS over 10 K opportunities, Redis usage-metering 10 K concurrent INCR, SSE concurrency cap 10/pod). Execute against the staging environment, commit raw results + summary to `tests/load/load-test-results.md`. Add `make load-baseline` and a CI release-candidate gate that re-runs the suite and fails the build on regression beyond ±10 % of committed baseline.

**Acceptance Criteria**:
- [ ] `tests/load/rest-p95.js`, `sse-ttfb.js`, `fts-search.js`, `usage-metering-concurrency.js`, `sse-concurrency-cap.js` checked in
- [ ] All 5 scripts executed against staging; raw output + summary committed to `tests/load/load-test-results.md` with timestamp + commit hash
- [ ] REST p95 ≤ 200 ms, SSE TTFB p95 ≤ 500 ms, FTS p95 ≤ 300 ms verified; if any threshold fails, story is `in-progress` with a remediation sub-task — never `done`
- [ ] `make load-baseline` documented in `Makefile` and `docs/load-testing.md`
- [ ] Release-candidate CI workflow re-executes the suite and fails on regression > 10 %
- [ ] Verification-matrix row for "k6 6-epic carry-forward" transitions to `closed`

---

### S13.04: Prometheus `/metrics` Bootstrap (All Services)

**Points**: 5 | **Type**: backend | **Severity:** Carry-forward critical (4 consecutive epics)

Add a `/metrics` endpoint to every service (`client-api`, `admin-api`, `data-pipeline`, `ai-gateway`, `notification`) using `prometheus-client`. Wire mandatory metric emitters via FastAPI middleware (REST latency histogram, error counter), via the AI Gateway client (SSE TTFB histogram, agent error counter), via the billing service (webhook latency, tier distribution, trial-to-paid, usage-sync drift), and via the pipeline (crawler success/failure counter). `/metrics` is cluster-IP-only (Helm Service `ClusterIP` annotation, NOT exposed via Ingress). Error responses on `/metrics` must never echo `str(exc)` or connection strings (NFR-OB-4 / NFR-SE-12).

**Acceptance Criteria**:
- [ ] Each of the 5 services exposes `/metrics` returning Prometheus text format
- [ ] Mandatory metrics emit non-zero values in staging: `http_request_duration_seconds`, `sse_ttfb_seconds`, `agent_calls_failed_total`, `billing_webhook_latency_seconds`, `tier_distribution`, `trial_to_paid_total`, `billing_usage_sync_drift_total`, `crawler_runs_total{status="success"|"failed"}`
- [ ] Helm `values.yaml` per service annotates the metrics port for Prometheus scraping; `/metrics` not reachable from public ingress
- [ ] Error path on `/metrics` returns generic 503 body — no internal detail leakage
- [ ] Integration tests assert all mandatory metric names exist and increment correctly under load
- [ ] Verification-matrix row for "Prometheus 4-epic carry-forward" transitions to `closed`

---

### S13.05: R-001 Optimistic-Locking GREEN Verification

**Points**: 3 | **Type**: backend + frontend | **Severity:** Carry-forward critical (E07 R-001 score 9)

Close the highest-risk Epic 7 finding: collaborative-edit data loss via optimistic locking has implementation but 0 % confirmed-passing ATDD tests. Add ≥1 GREEN integration test exercising the 409 path: client A and client B both load a content block; A saves first; B saves with stale `content_hash`; backend `SELECT FOR UPDATE` + hash comparison rejects with 409 + `{conflict: {server_version, client_version, diff_url}}`. Add ≥1 GREEN frontend RTL test asserting the conflict dialog opens, renders 3-way diff, and resolves either by reloading or by force-overwrite (which itself emits a fresh hash).

**Acceptance Criteria**:
- [ ] `tests/integration/test_proposal_optimistic_lock.py::test_409_on_stale_hash` GREEN
- [ ] `apps/client/__tests__/conflict-dialog.test.tsx::renders_three_way_diff` GREEN
- [ ] 409 body shape `{conflict: {server_version, client_version, diff_url}}` asserted; flat-vs-nested error body convention documented
- [ ] AC text in S07.04 reconciled with implementation (project-context Epic 13 anti-pattern: "AC text not updated to reflect implementation-time schema choices")
- [ ] Verification-matrix row for "R-001 0% GREEN tests" transitions to `closed`

---

### S13.06: Content-Block Prompt-Injection Sanitization Layer

**Points**: 3 | **Type**: backend (security) | **Severity:** Carry-forward high (E07-R-005)

Introduce a single sanitization helper `_sanitize_content_block(body: str) -> str` in `client_api/services/proposal/sanitizer.py` that operates on TipTap JSON / Markdown payloads, applies a node-type allowlist (paragraph, heading, list, table, code-block, link, bold, italic), and strips script-like tokens (`<script`, `javascript:`, `onerror=`, `onload=`, prompt-injection markers like `### SYSTEM`, `</prompt>`). All ingress paths (proposal save, content-block create/update, AI Gateway forwarding) call the helper. Add ≥3 GREEN unit tests for malicious payload classes; add ≥1 integration test asserting the AI Gateway request body is sanitized BEFORE outbound call.

**Acceptance Criteria**:
- [ ] `client_api/services/proposal/sanitizer.py::_sanitize_content_block` is the single entry point; no inline sanitization elsewhere (project-context Epic 13 pattern: "Single-point SSE error sanitization helper" — same shape applied to content blocks)
- [ ] Allowlist documented in `docs/security/content-block-allowlist.md`
- [ ] Unit tests cover: `<script>`, `javascript:` URI, prompt-injection sentinels, nested-node attack, encoded payloads
- [ ] Integration test asserts AI Gateway POST body is sanitized (mock `respx` capture)
- [ ] AC text + Pydantic schema enforce sanitization on every content-block-bearing endpoint
- [ ] Verification-matrix row for "E07-R-005 prompt-injection sanitization deferred" transitions to `closed`

---

### S13.07: Stripe Outbound Circuit Breaker

**Points**: 3 | **Type**: backend | **Severity:** Carry-forward high (E08)

Wrap all outbound Stripe + VIES calls in `billing_service.py` and `vies_service.py` with the canonical E04 two-layer resilience pattern: `circuit_breaker(retry(http_factory))`. Stripe Python SDK calls (synchronous) remain wrapped with `asyncio.to_thread()` from inside `async def` handlers. 4xx responses do NOT increment the breaker counter; 5xx and timeouts do. Default config: 5 consecutive failures opens breaker, 30 s open-state, half-open trial of 1 request. Add ≥1 GREEN integration test asserting breaker opens after 5 consecutive Stripe 5xx and the endpoint returns 503 fail-open while open.

**Acceptance Criteria**:
- [ ] `billing_service.py` and `vies_service.py` use the E04 canonical pattern; no bare Stripe / VIES calls remain
- [ ] 4xx-does-not-increment-breaker assertion present at unit-test level
- [ ] Integration test simulates 5 consecutive Stripe 5xx, asserts breaker open + endpoint behaviour
- [ ] Stripe SDK calls wrapped in `asyncio.to_thread()` (mandatory pattern, project-context Epic 8)
- [ ] Verification-matrix row for "Stripe outbound circuit-breaker absent" transitions to `closed`

---

### S13.08: Billing Observability — 5 Mandatory Metrics

**Points**: 2 | **Type**: backend | **Severity:** Carry-forward high (E08)

Inject five Prometheus metrics into the billing critical path (rolls into S13.04 bootstrap but owns the billing-specific instrumentation): webhook processing latency histogram, `billing_usage_sync_drift_total` gauge (Redis vs DB drift), Stripe API error counter (separate from agent error counter), active tier distribution gauge, trial-to-paid conversion counter. All metrics labelled with `tier` and `event_type` where applicable. Wire from `billing_service.py`, `webhook_service.py`, `tier_cache.py`, and the daily reconciliation job.

**Acceptance Criteria**:
- [ ] All 5 metrics emit non-zero values in staging within 24 h of deploy
- [ ] `billing_usage_sync_drift_total` is computed by the daily reconciliation job comparing Redis counters vs DB `usage_events` aggregates
- [ ] Grafana dashboard JSON checked into `infra/observability/dashboards/billing.json`
- [ ] Alert rules: drift > 5 % triggers PagerDuty warning; Stripe error rate > 1 % triggers warning
- [ ] Verification-matrix row for "No Prometheus billing metrics" transitions to `closed`

---

### S13.09: `invoice.payment_failed → past_due` Integration Test

**Points**: 1 | **Type**: backend (test) | **Severity:** Carry-forward high (E08)

Add a single missing test (`tests/integration/billing/test_stripe_webhook_flow.py::test_invoice_payment_failed_to_past_due`) that fires a mock `invoice.payment_failed` Stripe event end-to-end through the webhook endpoint, validates HMAC, asserts DB tier transitions to `past_due`, asserts tier cache DELETE, asserts Prometheus `tier_distribution{tier="past_due"}` increments by 1. Closes the Epic 8 P0 gap where this revenue-critical path had only unit-level coverage.

**Acceptance Criteria**:
- [ ] Test added and GREEN against testcontainers Postgres + Redis
- [ ] Mock event uses real Stripe webhook signature format with valid `event_id` for dedup table insertion
- [ ] Assertions cover DB state, Redis cache state, and metric emission
- [ ] Verification-matrix row for "invoice.payment_failed P0 unit-only" transitions to `closed`

---

### S13.10: ClamAV Stuck-Document Cleanup Task

**Points**: 2 | **Type**: backend | **Severity:** Carry-forward medium (E06)

Add a Celery Beat task `reset_stuck_documents_task` (analogous to E07's `reset_stuck_proposals_task`, project-context Epic 7 reference pattern) that transitions documents stuck in `pending` status >TTL (configurable, default 30 min) to `failed` with a structured `failure_reason: "clamav_timeout"`. Runs every 10 min. Emits a Prometheus counter `documents_stuck_reset_total{reason}`. Add ≥1 integration test simulating ClamAV outage (testcontainers ClamAV stopped) and asserting the task reaps stuck documents on next tick.

**Acceptance Criteria**:
- [ ] Task implemented and registered in Celery Beat schedule
- [ ] Configurable TTL via env var `CLAMAV_STUCK_DOCUMENT_TTL_SECONDS`
- [ ] `documents_stuck_reset_total` counter emits and increments under simulated outage
- [ ] Integration test GREEN
- [ ] Verification-matrix row for "ClamAV timeout-to-failed transition out of scope" transitions to `closed`

---

### S13.11: Frontend Tier-Enforcement E2E Suite

**Points**: 5 | **Type**: frontend (test) | **Severity:** Carry-forward high (E06 / E08)

Author a Playwright spec suite (`apps/client/e2e/tier-enforcement.spec.ts`) that exercises Free → Starter → Professional → Enterprise tier gates end-to-end across opportunity discovery, proposal export, AI summary, and content-blocks library. For each tier, assert the gated endpoints return the correct tier-shaped response (Free returns 6-field `OpportunityFreeResponse`; Pro+ returns full detail). Cross-tier deep-links return 404 (not 403). Source-inspect ATDD asserts no native `<select>`, `<dialog>`, `<sheet>`, `<tabs>` (must be design-system components from `@eusolicit/ui`) — project-context Epic 11 anti-pattern. Activate currently-skipped `billing-checkout.spec.ts` and `billing-vat.spec.ts` (project-context Epic 8 anti-pattern: "P0 E2E specs with ≥80 % `test.skip` are NONE coverage").

**Acceptance Criteria**:
- [ ] `tier-enforcement.spec.ts` covers all 4 tiers × 4 feature areas (16 scenarios) all GREEN
- [ ] Cross-tier deep-link 404-not-403 assertions present
- [ ] Source-inspection ATDD asserts design-system component compliance
- [ ] `billing-checkout.spec.ts` and `billing-vat.spec.ts` reactivated and GREEN; no `test.skip` lines remain in either file
- [ ] CI E2E job time budget: total suite ≤ 8 min on Chromium
- [ ] Verification-matrix rows for "P0 E2E specs ≥80 % skip" and "Frontend tier enforcement E2E gap" transition to `closed`

---

### S13.12: Story-File ↔ Sprint-Status Reconciliation Lint

**Points**: 1 | **Type**: tooling | **Severity:** Carry-forward medium (E13 self-finding)

Add a CI lint (`scripts/lint_story_status.py`) that scans `_bmad/bmm/sprint-status.yaml` and every `docs/stories/*.md` story file, fails the build if any story is `done` in sprint-status while its story-file Status is `review` / `in-progress` / `ready-for-dev`. Closes the Epic 13 anti-pattern "sprint-status `done` set without cross-checking story file status" (4th recurrence). Also adds a paired lint asserting `epic-N: in-progress` is not present in sprint-status when all sub-stories are `done` (4th-recurrence anti-pattern).

**Acceptance Criteria**:
- [ ] `scripts/lint_story_status.py` runs in CI on every PR
- [ ] Mismatch between sprint-status `done` and story-file Status fails the build with a clear diagnostic
- [ ] Closed-epic-but-in-progress detection raises a warning (not failure) — orchestrator can auto-close
- [ ] `docs/stories-lint.md` documents how to remediate
- [ ] Verification-matrix rows for both 4th-recurrence anti-patterns transition to `closed`

## Technical Notes

- **Coordinator pattern (project-context Epic 13)**: S13.01 owns the verification matrix; sub-stories (S13.02–S13.12) deliver independent slices. Coordinator cannot close until every sub-story is `done` AND its matrix row reads `closed`. Prevents mega-story scope explosion and enforces explicit closure of every retro `[ACTION]` item.
- **TEA-as-AC, not as story (project-context Epic 11)**: This epic establishes TEA review score ≥ 80/100 as a `done`-gate AC on every story going forward. The dormant `inj-03-tea-review-backlog` standalone story is closed as DUPLICATE — the AC supersedes it.
- **Hot-Fix Context section (project-context Epic 13)**: S13.01 carries a "Hot-Fix Context" section listing prior emergency fixes (S07.17 commit `2d41fcf`) so dev agents do not re-implement already-merged changes.
- **Quoted execution evidence (project-context Epic 13)**: All `done` transitions in this epic require the reviewer to quote pytest / playwright output in the review record. Approval without quoted execution is invalid (project-context Epic 13 critical anti-pattern).
- **`asyncio.CancelledError` discipline**: Every new `async def` introduced in this epic (notably S13.04 metrics middleware, S13.06 sanitizer, S13.07 circuit-breaker wrappers) MUST include `except asyncio.CancelledError: raise` or explicit handling — `CancelledError` inherits from `BaseException`, not `Exception` (project-context Epic 13 pattern).
- **Pydantic `Literal[...]` / `StrEnum` for status fields (project-context Epic 13)**: Any new status field introduced (`failure_reason` on documents, `breaker_state` on introspection endpoints) uses `Literal[...]` or `StrEnum`, never bare `str`.
- **Done-gate composition**: For every story in this epic, `done` requires (1) senior dev review APPROVED with quoted test execution, (2) ATDD tests GREEN, (3) i18n parity (n/a for backend stories), (4) no stub references for story-owned endpoints, (5) TEA review score ≥ 80/100, (6) verification-matrix row `closed`.
