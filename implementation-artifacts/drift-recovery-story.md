# Story: drift-recovery-story (Epic 13 — Drift Recovery & Hardening Coordinator)

Status: ready-for-dev

<!-- This is the coordinator/parent story for Epic 13 hardening. Sub-stories (inj-01, inj-02, inj-03, dw-01, dw-02, dw-03) deliver narrow technical slices; this story owns end-to-end acceptance and verification across all of them. -->

## Story

As a **platform operator**,
I want **the architectural drift accumulated across Epics 1–12 (Dependabot, k6 baseline, billing observability, Stripe resilience, TEA review backlog) recovered and verified before the Beta production-release gate**,
so that **EU Solicit ships with the security posture, performance baseline, observability surface, and quality coverage promised in PRD §8 (NFR1–NFR15) and the §12 open-question register (OQ-1, OQ-2, OQ-3) — instead of the 6+ epic carry-forward debt currently catalogued by Epic 8 NFR assessment and the 2026-04-25 Implementation Readiness HALT verdict**.

## Story Requirements

**Acceptance Criteria (BDD)**

- **GIVEN** the application has accumulated technical drift documented in `eusolicit-docs/test_artifacts/nfr-report.md` (4 HIGH issues, 6 CONCERNS) and the IR HALT report `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-04-25.md`,
- **WHEN** the drift-recovery work below is executed and merged,
- **THEN** Dependabot must be configured for `pip`, `npm`, and `github-actions` ecosystems with weekly cadence and grouped security PRs (closes 6th-consecutive-epic carry-forward; PRD §12 OQ-2),
- **AND** a k6 performance baseline must be established, executed, and committed under `test_artifacts/`, with real `p50/p95/p99` numbers populating `eusolicit-docs/implementation-artifacts/load-test-results.md` — the empty-stub state that has persisted for 6 epics is no longer acceptable (PRD §8 NFR2/NFR3, §12 OQ-1),
- **AND** the TEA review backlog (Epic 8 + Epic 9, 6th-consecutive-epic gap of 0 scored reviews) must have at least 3 stories per epic with TEA review scores ≥ 80/100 recorded in `sprint-status.yaml` (`tea_status` field) (Epic 8 retro action ACT-V8-04),
- **AND** all outbound Stripe SDK calls in `services/client-api/src/client_api/services/billing_service.py` and `vies_service.py` must be wrapped in the E04 two-layer resilience pattern `circuit_breaker(retry(stripe_call))` — never bare `try/except stripe.error.StripeError` (NFR4),
- **AND** the five billing Prometheus metrics (`billing_webhook_processing_duration_seconds`, `billing_usage_sync_drift_total`, `billing_stripe_api_errors_total`, `billing_active_subscriptions_total{tier=…}`, `billing_trial_to_paid_conversions_total`) must be registered against `SERVICE_METRICS_REGISTRY` and visible at `/metrics` (NFR13, PRD §12 OQ-3),
- **AND** the planning-artifact integrity gaps called out by the 2026-04-25 IR report — corrupted `epics.md` and missing `epics/E13-drift-recovery-and-hardening.md` — must be tracked as out-of-scope follow-ups but **not block** this story (they live in the planning track, not the codebase track).

## Sub-Story Coordination Map

This story is the **coordinator**. The actual code changes are landed by the following sub-stories already on the Sprint 13 board (`sprint-status.yaml`):

| Sub-story | Status (at story creation) | Slice owned | This story's dependency |
|---|---|---|---|
| `inj-01-dependabot-configuration` | ready-for-dev | `.github/dependabot.yml` for `pip` + `npm` + `github-actions` | AC1 |
| `inj-02-k6-performance-baseline` | ready-for-dev | k6 scripts + `load-test-results.md` populated | AC2 |
| `inj-03-tea-review-backlog-epic8-epic9` | ready-for-dev | TEA scored reviews for ≥ 3 stories in E08 + E09; `tea_status` updates | AC3 |
| `dw-01-proposal-backend-schema-alignment` | in-progress | Backend DTO ↔ frontend type alignment (E07 retro lesson) | (cross-cutting; not gated by this story but must be done before E13 close) |
| `dw-02-celery-task-infrastructure-fixes` | ready-for-dev | `_RUN_ID_REGISTRY` prefork bug + non-atomic transactions (E05 retro) | (cross-cutting) |
| `dw-03-ui-breakpoint-hook-fix` | ready-for-dev | `useBreakpoint` 1024 vs 1280 px alignment (E07 retro lesson) | (cross-cutting) |

**Net new work owned directly by this coordinator story** (i.e., not in any sub-story above):

- AC4 — Stripe outbound circuit-breaker wrap (`billing_service.py`, `vies_service.py`)
- AC5 — Five billing Prometheus metrics (registered + emitted + scrapeable at `/metrics`)
- AC6 — Coordinator gate: verify all sub-stories are GREEN, evidence committed, and update sprint-status `epic-13` → `done` only when all six ACs below pass.

## Acceptance Criteria

### AC1 — Dependabot Configured (delegated to `inj-01`)

**Files:**
- `.github/dependabot.yml` (new)

**Required content (minimum):**
```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/eusolicit-app"
    schedule:
      interval: "weekly"
    groups:
      security:
        applies-to: security-updates
  - package-ecosystem: "npm"
    directory: "/eusolicit-app/frontend"
    schedule:
      interval: "weekly"
    groups:
      security:
        applies-to: security-updates
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

**Verification:** A first weekly Dependabot PR cycle must complete; no critical/high CVEs unresolved on `stripe`, `vies-python`, `weasyprint`, `python-docx`, `tiptap/*`, `recharts`, `@dnd-kit/sortable`, `boto3`, `pyclamd`, `fakeredis`, `testcontainers`, `clamav` clients (per Epic 4–8 retro inventories).

**Why directories are explicit:** Repo layout is `eusolicit-app/` for Python services (each service has its own `pyproject.toml` — Dependabot scans recursively from `directory:` root) and `eusolicit-app/frontend/` for the pnpm/Turborepo monorepo. Setting `directory: "/"` would miss both.

**This is the 6th consecutive epic with this gap. Do not defer again.**

### AC2 — k6 Performance Baseline Established (delegated to `inj-02`)

**Files:**
- `eusolicit-app/tests/perf/k6/*.js` (new — scripts)
- `eusolicit-docs/implementation-artifacts/load-test-results.md` (currently empty stub — populate with real numbers)
- `test_artifacts/k6-{date}.html` (k6 HTML report committed as evidence)

**Required test scenarios (minimum P0 coverage):**

| Endpoint | Target | Why |
|---|---|---|
| `GET /api/v1/opportunities` (FTS list) | p95 < 200 ms at 50 VUs | PRD §8 NFR2 — REST p95; FTS over 10K+ opportunities unvalidated for 4 epics |
| `GET /api/v1/opportunities/{id}/summary` (SSE) | TTFB p95 < 500 ms | PRD §8 NFR3 — SSE TTFB; SSE concurrency cap (10/pod) unvalidated |
| `POST /api/v1/billing/webhooks/stripe` | 100 req/s sustained | NFR backstop for webhook idempotency under replay storms |
| `POST /api/v1/billing/checkout/session` | p95 < 500 ms at 20 VUs | Checkout latency dominated by Stripe — under degradation, p95 could exceed 200 ms; circuit-breaker fail-fast is the mitigation (see AC4) |
| `GET /api/v1/subscription/usage` | p95 < 200 ms at 50 VUs | Redis INCR + DB row read; should be fast — verify |
| **Redis usage counter scalability test (8.8-PERF-001)** | 10,000 concurrent INCR → final count == 10,000 | Epic 8 NFR `R-003` mitigation; only architecture, never load-tested |

**Output evidence committed:**
1. `load-test-results.md` populated with: p50, p95, p99 per endpoint; pass/fail vs threshold; date; environment.
2. `test_artifacts/k6-{YYYY-MM-DD}.html` HTML report.
3. PR description includes a quote of headline numbers (so the historical record survives even if k6 artifacts are pruned).

**This is the 6th consecutive epic with this gap.** Per Epic 8 retro: "S12.17 hard gate; no further deferrals." S12.17 is marked `done` but the load-test results file was reported empty at retrospective time — verify it is **not** still empty before this AC passes.

### AC3 — TEA Review Backlog Cleared (delegated to `inj-03`)

**Scope:** Epic 8 (Subscription & Billing) and Epic 9 (Notifications/Calendar). Both have 0/14 TEA scored reviews per `nfr-report.md` and `traceability-matrix.md`.

**Required minimum:** ≥ 3 TEA-scored reviews per epic, score ≥ 80/100, recorded in `sprint-status.yaml` under a `tea_status:` block (or equivalent; check `_bmad/bmm/sprint-status.yaml` schema). Priority candidates:

**Epic 8 (revenue-critical):**
- S08.04 webhook HMAC + idempotency (R-004 mitigation — security-critical)
- S08.08 usage metering with Redis counters (R-003 — accuracy-critical)
- S08.10 EU VAT/VIES (only confirmed-GREEN story; reference for review)

**Epic 9 (per traceability-matrix.md, all 14 stories at FULL coverage):**
- S09.04 alert matching + immediate dispatch (Redis stream consumer correctness)
- S09.08 Google Calendar OAuth2 sync (token encryption / Fernet pattern)
- S09.10 task/approval notification consumer (lifecycle event handling)

**Per Epic 7 retro lesson:** TEA review catches contract drift and constant mismatches invisible to ATDD (e.g., `useBreakpoint` 1024 vs 1280 px; `ProposalResponse` missing fields). Apply that lens here.

**Per Epic 8 retro action ACT-V8-04 (open):** Configure TEA review score ≥ 80/100 as a blocking gate `review → done` in the agent config. That config change is **out of scope for this story** but listed so the developer knows it is the structural fix for the 7-consecutive-epic gap.

### AC4 — Stripe Outbound Circuit-Breaker (NEW work owned by this story)

**Files (definitive list — confirm via Grep before edit):**
- `eusolicit-app/services/client-api/src/client_api/services/billing_service.py` (every `asyncio.to_thread(stripe.*)` call site)
- `eusolicit-app/services/client-api/src/client_api/services/vies_service.py` (VIES SOAP calls — same pattern, different upstream)

**Pattern (from Epic 4 — `project-context.md` confirms this is the platform standard):**

```python
# Two-layer composition: circuit_breaker is the OUTER wrapper, retry is INNER.
# Order matters — circuit_breaker must see retry exhaustion as a single failure,
# otherwise retry will burn the circuit-breaker's failure budget on a single request.

from eusolicit_common.resilience import circuit_breaker, retry

stripe_call = circuit_breaker(
    retry(
        lambda: asyncio.to_thread(stripe.checkout.Session.create, **kwargs),
        max_attempts=3,
        backoff="exponential",
        retry_on=(stripe.error.APIConnectionError, stripe.error.RateLimitError),
    ),
    failure_threshold=5,          # open after 5 consecutive failures
    recovery_timeout_seconds=30,  # half-open probe after 30s
    expected_exception=stripe.error.StripeError,
)
result = await stripe_call()
```

**Critical reuse rule:** Do **not** invent a new circuit-breaker. Use the existing implementation in `eusolicit-common` (or wherever the Epic 4 AI Gateway client lives). If the implementation is currently coupled to the AI Gateway client, **extract it** into `eusolicit-common/resilience/` (or wherever `circuit_breaker` already lives) so billing and VIES can import it. Reference the Epic 4 retrospective and `project-context.md` Epic-4 patterns ("Two-layer resilience composition: `circuit_breaker(retry(http_factory))` — now the standard for all outbound HTTP").

**Gotcha — the OBS-001 bug from Epic 5 retro (still open in `project-context.md`):** The current circuit-breaker implementation increments the failure counter on **4xx responses** as well as 5xx, in both the E04 AI Gateway service and the E05 pipeline client. Same logical bug in two places. **If you extract the circuit-breaker for reuse here, fix that bug at the same time** — only 5xx and connection/timeout errors should count as failures. Stripe 4xx (e.g., `card_declined`) is a domain error, not a resilience signal.

**Test obligations (per Epic 4 ATDD pattern):**
- Unit: feed mock 5 consecutive Stripe `APIError` (5xx) → assert circuit opens.
- Unit: feed 4 mock 5xx → reset → assert circuit stays closed.
- Unit: feed 5xx 5x → wait `recovery_timeout_seconds` → assert half-open probe sent.
- Unit: assert 4xx (e.g., `card_declined`) does **not** increment failure counter (OBS-001 fix verification).
- Integration (`testcontainers` + `respx` per E04 standard): mock Stripe API at the HTTP layer; verify circuit-breaker behavior end-to-end through `billing_service.py`.

### AC5 — Billing Prometheus Metrics (NEW work owned by this story)

**Files:**
- `eusolicit-app/services/client-api/src/client_api/observability/metrics.py` (existing per Epic 5; add 4 metrics here)
- `eusolicit-app/services/notification/src/notification/observability/metrics.py` (add `billing_usage_sync_drift_total` here — sync runs in notification service per `tasks/billing_usage_sync.py`)
- `eusolicit-app/services/client-api/src/client_api/services/billing_service.py` (emission sites)
- `eusolicit-app/services/client-api/src/client_api/services/webhook_service.py` (webhook latency timing)
- `eusolicit-app/services/notification/src/notification/workers/tasks/billing_usage_sync.py` (drift gauge emission)

**Five metrics (verbatim names — per Epic 8 retro action items):**

| Metric | Type | Labels | Source |
|---|---|---|---|
| `billing_webhook_processing_duration_seconds` | Histogram | `event_type`, `outcome` (`success` / `idempotent` / `error`) | `webhook_service.py` — wrap event processing in a timer |
| `billing_usage_sync_drift_total` | Gauge | `tier` | `billing_usage_sync.py` — diff Redis counter vs Stripe usage record after sync; emit absolute drift |
| `billing_stripe_api_errors_total` | Counter | `endpoint` (`checkout`, `portal`, `subscription`, `usage_record`), `error_type` (`api_connection`, `rate_limit`, `card_declined`, `other`) | Inside the `try/except` wrapping each Stripe SDK call (after AC4 wrapper) |
| `billing_active_subscriptions_total` | Gauge | `tier` (`free`, `starter`, `professional`, `enterprise`) | Periodic refresh via Celery task or subscription event consumer; do not query DB on every scrape |
| `billing_trial_to_paid_conversions_total` | Counter | `tier` (target tier user upgraded to) | Increment in the subscription event handler when status transitions `trialing → active` AND `tier ≠ free` |

**Pattern adherence:** Follow the existing `metrics.py` + `SERVICE_METRICS_REGISTRY` pattern from Epic 5. Per Epic 5 retro anti-pattern: **never** wrap metric emission in `try/except Exception: pass` — that hides operational errors. If a metric emission can fail (e.g., label cardinality), let it raise in CI tests so the bug is caught.

**Test obligations:**
- Unit per metric: emit value → scrape `/metrics` via FastAPI test client → assert metric line present with correct labels.
- Integration (one happy path through webhook service): fire mock `customer.subscription.updated` → assert `billing_webhook_processing_duration_seconds_count{outcome="success"}` incremented by 1.
- Smoke (manual or CI): `curl http://localhost:8001/metrics | grep ^billing_` lists all 5 metric families.

**Grafana panels (out of scope for this story but worth listing for the next story):** Webhook latency p95; usage-sync drift > 0 alert; Stripe error rate by endpoint; tier distribution stacked bar; trial-to-paid conversion week-over-week.

### AC6 — Coordinator Gate

This story moves to `done` **only when all of the following are GREEN**:

- [ ] AC1: `.github/dependabot.yml` merged; first PR cycle observed within 7 days.
- [ ] AC2: `load-test-results.md` populated with real numbers; `test_artifacts/k6-*.html` committed; thresholds met or documented exceptions raised.
- [ ] AC3: ≥ 3 TEA-scored reviews per E08 + E09 with score ≥ 80/100 in `sprint-status.yaml` (or `tea_status` block).
- [ ] AC4: Both `billing_service.py` and `vies_service.py` use `circuit_breaker(retry(...))`; OBS-001 bug fixed in the shared implementation; ≥ 4 unit tests + 1 integration test GREEN.
- [ ] AC5: All 5 metrics registered in `SERVICE_METRICS_REGISTRY`; visible in `/metrics`; ≥ 5 unit tests + 1 integration test GREEN.
- [ ] Sub-stories closed: `inj-01`, `inj-02`, `inj-03` → `done`. (`dw-01`/`dw-02`/`dw-03` are cross-cutting tech-debt items that may close in parallel; this story does not block on them.)

## Tasks / Subtasks

- [ ] **T1 (AC1) — Dependabot:** Coordinate `inj-01-dependabot-configuration`. Verify `.github/dependabot.yml` exists with the three ecosystems above; first PR cycle observed.
- [ ] **T2 (AC2) — k6 baseline:** Coordinate `inj-02-k6-performance-baseline`. Verify scripts exist, results committed, `load-test-results.md` populated.
- [ ] **T3 (AC3) — TEA backlog:** Coordinate `inj-03-tea-review-backlog-epic8-epic9`. Verify ≥ 3 reviews per epic with score ≥ 80/100.
- [ ] **T4 (AC4) — Stripe circuit-breaker (owned by this story):**
  - [ ] T4.1 — Locate the existing E04 circuit-breaker implementation; if not in a shared package, extract to `eusolicit-common/resilience/`.
  - [ ] T4.2 — Fix OBS-001: failure counter must NOT increment on 4xx responses (only 5xx + connection/timeout). Apply fix once at the source.
  - [ ] T4.3 — Wrap every `asyncio.to_thread(stripe.*)` call in `billing_service.py` with `circuit_breaker(retry(...))`.
  - [ ] T4.4 — Wrap VIES SOAP calls in `vies_service.py` with the same pattern (already has fail-open; CB layer is additive).
  - [ ] T4.5 — Unit tests: 5 scenarios (5x 5xx → open, 4x 5xx → closed, half-open probe, 4xx does not increment, recovery).
  - [ ] T4.6 — Integration test (`testcontainers` + `respx` per Epic 4 stack): end-to-end through `billing_service.create_checkout_session()`.
  - [ ] T4.7 — ATDD checklist for billing endpoints: assert circuit-breaker opens after 5 consecutive Stripe 5xx; 503 returned with `Retry-After` during open.
- [ ] **T5 (AC5) — Billing Prometheus metrics (owned by this story):**
  - [ ] T5.1 — Add 4 metrics to `client-api/observability/metrics.py` (webhook duration histogram, Stripe error counter, active subscriptions gauge, conversion counter).
  - [ ] T5.2 — Add 1 metric to `notification/observability/metrics.py` (`billing_usage_sync_drift_total`).
  - [ ] T5.3 — Emit webhook duration in `webhook_service.py` (wrap event processing in timer; capture `outcome` label).
  - [ ] T5.4 — Emit Stripe error counter in `billing_service.py` (inside `except` blocks of AC4 wrappers).
  - [ ] T5.5 — Emit active subscriptions gauge via Celery periodic task (refresh every 5 min) — never on every `/metrics` scrape.
  - [ ] T5.6 — Emit trial→paid counter in subscription event consumer.
  - [ ] T5.7 — Emit usage sync drift gauge in `billing_usage_sync.py` after each sync run.
  - [ ] T5.8 — Unit tests: 5 metrics × 1 emission test each.
  - [ ] T5.9 — Integration test: fire `customer.subscription.updated` mock → scrape `/metrics` → assert webhook histogram count incremented.
- [ ] **T6 (AC6) — Coordinator gate:** Verify all checklist items in AC6 are GREEN; update `sprint-status.yaml` to mark this story `done` and `epic-13` → `done`.

## Dev Notes

### Architecture Compliance (Persistent Facts from `project-context.md`)

The drift-recovery story exists precisely because patterns codified in `project-context.md` were **not extended** to the billing service. Quoting the patterns this story must conform to:

- **Epic 4 pattern:** "Two-layer resilience composition: `circuit_breaker(retry(http_factory))` — now the standard for all outbound HTTP."
- **Epic 4 pattern:** "`testcontainers` + `respx` is the mandatory backend integration test stack."
- **Epic 5 pattern:** "E04 two-layer resilience pattern reused successfully in E05 pipeline AI Gateway client without redesign." — Same expectation here for billing.
- **Epic 5 anti-pattern (OBS-001):** "4xx errors incorrectly increment circuit breaker failure counter in both E04 AI Gateway service and E05 pipeline client; same logical bug in two places; fix must be applied simultaneously." — **Apply the same simultaneous fix** when extending to billing/VIES.
- **Epic 5 anti-pattern:** "Silent `except Exception: pass` on all Prometheus metrics instrumentation across 4 pipeline files; hides operational errors without any log signal." — Do not repeat this in `billing_service.py`.
- **Epic 8 pattern:** "`asyncio.to_thread()` is mandatory for synchronous I/O SDKs inside `async def` FastAPI handlers — Stripe Python SDK, VIES SOAP clients, ... MUST be wrapped." — Already satisfied by `billing_service.py`; preserve when adding the CB wrapper.
- **Epic 8 anti-pattern (now closing):** "Stripe outbound circuit-breaker absent — E04 two-layer resilience pattern not extended to billing." — This story closes that finding.
- **Epic 8 anti-pattern (now closing):** "No Prometheus billing metrics — revenue-critical path completely unobservable; missing webhook processing latency histogram, `billing_usage_sync_drift_total` gauge, Stripe API error counter, active tier distribution gauge, trial-to-paid conversion counter." — This story closes that finding (5 metrics).
- **Epic 8 anti-pattern (closing across project):** "k6 performance baseline absent — 6th consecutive epic carry-forward (E03→E05→E06→E07→E08); `load-test-results.md` remains an unfilled template." — Closed by AC2/`inj-02`.
- **Epic 8 anti-pattern (closing across project):** "Dependabot not configured — 6th consecutive epic carry-forward." — Closed by AC1/`inj-01`.

### File-Structure Requirements

- Python services live under `eusolicit-app/services/<service>/src/<service>/`.
- Shared resilience and metrics primitives belong in `eusolicit-app/packages/eusolicit-common/`. **If the existing circuit-breaker is not yet here, extract it as part of T4.1.**
- Tests follow service-local convention: `eusolicit-app/services/client-api/tests/{unit,integration,api}/test_billing_*.py`.
- Per-service Prometheus registry: `services/<svc>/src/<svc>/observability/metrics.py` calls `SERVICE_METRICS_REGISTRY` — pattern established in Epic 5.
- Performance scripts: `eusolicit-app/tests/perf/k6/*.js` (per `inj-02`).
- Performance evidence: `eusolicit-docs/implementation-artifacts/load-test-results.md` and `test_artifacts/k6-{date}.html`.

### Library / Framework Versions

| Library | Constraint | Why pinned |
|---|---|---|
| `stripe` (Python) | follow Epic 8 pin (do not bump) | API version pinning prevents Stripe breaking changes (Epic 8 retro R-007) |
| `prometheus-client` | match existing `eusolicit-common` constraint | All services share `SERVICE_METRICS_REGISTRY` — single source of version truth |
| k6 | latest stable (use Docker image `grafana/k6:latest` for CI) | No app-side coupling |
| `pytest` / `respx` / `testcontainers` | match `pyproject.toml` of `client-api` and `notification` | Mandatory integration stack per E04 retro |

**Do not introduce new HTTP client libraries.** The Stripe SDK already wraps `urllib3`; the circuit-breaker layer wraps the SDK call, not the underlying HTTP. Do not reach for `httpx` here.

### Testing Requirements (Epic-Level Test Design)

The epic-level test design lives in `test_artifacts/` (project root). Loaded for this story:

**From `test_artifacts/nfr-report.md` (Epic 8 NFR Assessment, dated 2026-04-24):**
- Overall: PASS (with CONCERNS); 4 PASS / 6 CONCERNS / 0 FAIL across 8 ADR categories; 20/29 criteria met (69%).
- 4 HIGH issues all map to this story's ACs (k6, circuit-breaker, billing metrics, Dependabot).
- All four high-risk mitigations (R-001 webhook idempotency, R-002 VIES fallback, R-003 Redis atomicity, R-004 HMAC verification) are **already implemented and verified** at unit/integration level — do not regress them while adding the CB wrapper or metrics emission.
- Specific test IDs to extend or activate:
  - **8.8-PERF-001** — 10K concurrent INCR → Redis count == 10K. Currently *not written*. **Must be written and executed under AC2.**
  - **8.4-API-004** — `invoice.payment_failed → past_due`. Currently unit-only. Per `nfr-report.md` quick-win #3, add an integration test via `test_stripe_webhook_flow.py` using ASGI test client. **Recommended companion test for AC4 work** (since the CB wrapper touches the same call paths).
- Recommended monitoring hooks (8 listed in §"Monitoring Hooks") map to the metrics in AC5; ensure metric names match so Grafana panels and alerts can reference them directly.

**From `test_artifacts/traceability-matrix.md` (Epic 9 Traceability):**
- Gate: PASS. 14/14 stories at FULL coverage across unit/integration/API/UI.
- Recommended action verbatim: "Run /bmad:tea:test-review to assess test quality." → That action is `inj-03` (AC3).
- Patterns codified for Epic 9 (test-quality-relevant): dual-layer idempotency (DB constraint + Redis SETNX), `asyncio.to_thread` for Celery `.delay()` from async consumers, 404 (not 403) for cross-user scoped resources, narrow `except` clauses (never swallow `celery.exceptions.Retry`), Fernet for OAuth token encryption, ECDSA webhook validation on raw body fail-closed. **TEA review under AC3 should sample these patterns** in S09.04, S09.08, S09.10.

**From `test_artifacts/gate-decision.json`:** Epic 9 PASS at 100% / 100% / 100% (P0/P1/overall). No critical gaps. Use it as the reference shape for the gate-decision artifact this drift-recovery story should produce on close (`test_artifacts/gate-decision-epic-13.json`, schema-compatible).

**From `eusolicit-docs/test-artifacts/test-design-epic-08.md`:**
- **R-003 (Redis usage metering drift) & R-004 (Stripe webhook signature bypass)** are marked as High-Priority risks (Score 6).
- Test **8.8-PERF-001 (10,000 concurrent INCR)** is explicitly defined as a P3 load test. This directly supports AC2.
- Test **8.4-API-004 (invoice.payment_failed marks past_due)** is defined as a P0 API test. Use this as a companion test for the Stripe circuit-breaker (AC4).

**From `eusolicit-docs/test-artifacts/test-design-epic-09.md`:**
- **E09-R-004 (Stripe usage counter atomicity)** is marked as a High-Priority risk (Score 6). The mitigation strategy explicitly enforces operation order: read counter → call Stripe with idempotency key → `GETDEL` only on confirmed 200 response. This must be validated in AC4/AC5 testing.
- The epic test design mandates that crash-before-ACK scenarios do not produce duplicate emails (E09-R-002) and that Stripe counters are intact on failure but cleared on success. Ensure the TEA reviews (AC3) check for coverage of these explicit risk mitigations.

**From the wider `test_artifacts/` directory:** ATDD checklists for `4-9`, `6-5`, `9-3`, `10-1`, `10-2`, `10-8` already exist as templates. **No ATDD checklist exists for any drift-recovery sub-story** — generate one for AC4 and AC5 work as part of T4.7 and T5.8/T5.9. Follow the structure of the existing 10-x checklists.

### Previous Story Intelligence

The closest prior reference is **`12-17-performance-optimization-load-testing-security-audit.md`** (status: `done`). It covers GZip middleware, CORS, Kubernetes NetworkPolicies, OWASP audit. Reading it before starting this story is mandatory: it shows the platform-standard pattern for cross-cutting hardening stories (per-AC file lists, exact code snippets per file, middleware order rationale). Mirror that level of specificity in your sub-task notes.

Two failure modes documented in the Epic 7 retrospective directly affect this story's quality bar:

1. **Schema drift between backend Pydantic and frontend types** caught only in TEA review (`ProposalResponse` missing `current_version_number` / `generation_status`). When you add new metrics or new error responses (AC4 503 with `Retry-After`), update both backend DTOs and any frontend types in the same PR. Codegen is the long-term fix; this story is too narrow to introduce codegen but should not regress the manual sync.
2. **Numeric constants drift between AC and implementation** (1024 px vs 1280 px). All numeric thresholds in this story (5 consecutive failures, 30 s recovery, 1000-byte gzip floor, 200 ms p95, 500 ms p99, 30 s SSE TTFB) **must appear verbatim in code as named constants**, not magic numbers. Reviewers should grep for the numbers and find a constant.

### Project Structure Notes

- `epic-13` exists in `sprint-status.yaml` but **no `eusolicit-docs/planning-artifacts/epics/E13-*.md` scope document exists** (Implementation Readiness Report 2026-04-25 §6.2 C2). This is a planning-track gap, not a code-track gap; do **not** block dev work on it. The dev work has all the context it needs in this story file plus `nfr-report.md`.
- `eusolicit-docs/planning-artifacts/epics.md` is corrupted (duplicate body, unsubstituted templates, FR15 contradicts PRD). Do **not** read `epics.md` for FR/AC context. Read `eusolicit-docs/planning-artifacts/PRD.md` (v2.0) and the per-epic files in `eusolicit-docs/planning-artifacts/epics/E0n-*.md` instead. This is the single most important "do not be misled" warning in this story.
- `eusolicit-docs/planning_artifacts/` (underscore — note hyphen-vs-underscore!) is a stale regression directory with old PRD/architecture/ux-spec files. Some skill is still writing to it. Do not read or write there.

### Git Intelligence (Recent Patterns)

Recent commits in the story-creation window touched proposal/billing services and frontend. The pattern for hardening PRs in this repo:

- One PR per AC (or per coherent slice) — six PRs total here (one each for inj-01/02/03 already on the board, then this story's AC4 + AC5 + final coordinator close-out). Do not bundle.
- ATDD checklist file added in the same PR that adds the test code.
- Small dev-notes update at the end of each PR for `project-context.md` candidates (any new pattern or anti-pattern observed).
- `make lint && make type-check && make test` must pass locally before push (per `CLAUDE.md` Commands section).

### References

- [Source: `eusolicit-docs/planning-artifacts/PRD.md` §8 NFR2/NFR3/NFR4/NFR13 — performance, resilience, observability targets]
- [Source: `eusolicit-docs/planning-artifacts/PRD.md` §12 OQ-1/OQ-2/OQ-3 — open questions this story closes]
- [Source: `test_artifacts/nfr-report.md` §"Quick Wins", §"Recommended Actions", §"Evidence Gaps", §"Monitoring Hooks"]
- [Source: `test_artifacts/traceability-matrix.md` Epic 9 PASS gate; coverage heuristics inventory]
- [Source: `test_artifacts/gate-decision.json` schema for Epic 13 close-out artifact]
- [Source: `eusolicit-docs/planning-artifacts/project-context.md` §"Patterns" Epic 4 (two-layer resilience), Epic 5 (E04 reuse), Epic 8 (asyncio.to_thread)]
- [Source: `eusolicit-docs/planning-artifacts/project-context.md` §"Anti-Patterns" Epic 5 OBS-001 (4xx miscount); Epic 8 Stripe CB absent / no billing metrics / Dependabot 6 epics / k6 6 epics]
- [Source: `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-04-25.md` §3.3 Missing/weak coverage; §6.2 critical issues C7 (k6/Dependabot/metrics carry-forward); §6.3 step 7 (Promote E13 hardening to blocking-gate status)]
- [Source: `eusolicit-docs/implementation-artifacts/12-17-performance-optimization-load-testing-security-audit.md` — pattern reference for cross-cutting hardening stories]
- [Source: `eusolicit-app/services/client-api/src/client_api/services/billing_service.py` — every `asyncio.to_thread(stripe.*)` call is an AC4 wrap site]
- [Source: `eusolicit-app/services/client-api/src/client_api/services/vies_service.py` — VIES SOAP wrap site]
- [Source: `eusolicit-app/services/notification/src/notification/workers/tasks/billing_usage_sync.py` — `billing_usage_sync_drift_total` emission site]
- [Source: `CLAUDE.md` §Commands — `make lint`, `make type-check`, `make test`, `make test-integration`, coverage minimum 80%]

### Out-of-Scope (explicit, to prevent scope creep)

The 2026-04-25 IR HALT report flagged additional gaps. These are **NOT** part of this story:

- Corrupted `epics.md` regeneration (planning track, not code).
- Missing `E13-*.md` scope document (planning track, not code).
- Duplicate UX docs (`ux-spec.md` vs `ux-design-specification.md`) (UX/Plan owner).
- `planning_artifacts/` (underscore) regression (orchestrator-team / writer identification).
- Legacy `EU_Solicit_*.md` archival (Plan owner).
- PRD §6.5/§6.6/§6.8 FR-numbering revision and NFR11/12 reconciliation (PRD owner).
- TEA review score blocking-gate config change (Test Architect / config tuning, ACT-V8-04).

These are tracked elsewhere; do not let them hold up this story's GREEN gate.

## Dev Agent Record

### Status

ready-for-dev

### Agent Model Used

(populated by dev-story)

### Debug Log References

(populated during implementation)

### Completion Notes List

- Ultimate context engine analysis completed 2026-04-25: comprehensive coordinator story drafted from PRD v2.0 §8 + §12, `test_artifacts/nfr-report.md`, `traceability-matrix.md`, `gate-decision.json`, `project-context.md` Epic 4–9 patterns/anti-patterns, and 2026-04-25 Implementation Readiness Report.
- Story scope confirmed: AC1–AC3 delegated to `inj-01`/`inj-02`/`inj-03` sub-stories (already `ready-for-dev`); AC4 (Stripe circuit-breaker) and AC5 (5 Prometheus metrics) are net-new work owned by this coordinator. AC6 is the close-out gate.
- Out-of-scope items from the IR HALT report explicitly fenced off so dev does not stall on planning-track work.

### File List

(populated during implementation)

## Project Context Reference

See `eusolicit-docs/planning-artifacts/project-context.md` for the full pattern/anti-pattern catalogue. The pattern-and-anti-pattern citations in §"Architecture Compliance" above are extracted verbatim from that file. When this story closes, append the new pattern observed: **"E04 circuit-breaker pattern extended to all outbound payment SDK calls (Stripe + VIES) with simultaneous OBS-001 fix; revenue-critical path now observable via 5 billing metrics."** under §Patterns / Epic 13.
