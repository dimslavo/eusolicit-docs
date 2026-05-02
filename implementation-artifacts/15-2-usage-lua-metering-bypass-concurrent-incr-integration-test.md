# Story 15.2: Per-Bid Usage Metering Bypass & Concurrent INCR Integration Test

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a Bid Manager with an active Per-Bid SKU,
I want to generate AI summaries and drafts without hitting tier limits, and ensure atomic concurrent usage metering,
so that I am not blocked during a critical pursuit, and the platform accurately counts parallel usage via Lua scripts without race conditions.

## Acceptance Criteria

1. [x] **Given** an opportunity has `per_bid_sku_active=true`
   **When** the user invokes a metered AI feature for that opportunity
   **Then** the usage metering check bypasses the monthly subscription tier limit
   **And** the action proceeds successfully
2. [x] **Given** 10,000 concurrent metered AI requests for an opportunity without Per-Bid SKU
   **When** the usage is tracked via Redis INCR
   **Then** the atomicity is maintained via Lua script or strict atomic operations
   **And** the final usage count is exactly 10,000 (resolving NFR 8.8-PERF-001)
3. [x] **Given** the usage middleware
   **When** checking limits
   **Then** it first checks for the active per-bid SKU in Redis (`add_on_active:{workspace_or_company_id}:{opportunity_id}`)
   **And** if present, it completely bypasses the standard tier-based usage increment and limit check.

## Tasks / Subtasks

- [x] Task 1: Implement Lua script for atomic usage increment (AC: 2)
  - [x] Subtask 1.1: Replace simple `INCR` with a Lua script in `client-api/src/api/services/usage_meter_service.py` to ensure `INCR` + `EXPIRE` atomicity.
- [x] Task 2: Implement Per-Bid SKU Bypass (AC: 1, 3)
  - [x] Subtask 2.1: Check for `add_on_active:{workspace_or_company_id}:{opportunity_id}` in Redis before incrementing.
  - [x] Subtask 2.2: If key exists, bypass usage increment and limit checks completely.
- [x] Task 3: Develop Concurrent Integration Tests (AC: 2)
  - [x] Subtask 3.1: Write a pytest/k6 script to simulate 10,000 concurrent requests against the increment logic.
  - [x] Subtask 3.2: Verify the final usage count is exactly 10,000.

## Dev Notes

- **Operator Workflow Guidance:** 
  - Before starting this epic, [IR] Implementation Readiness was run to validate the specs.
  - Before starting this story, ALWAYS run [VS] Validate Story. This is non-negotiable.
  - After this story is complete, run [SR] Story Review, and since this is the final story in Epic 15, run [ER] Epic Review to validate the epic as a whole before QA.
  - After code review is complete, run [PR] Post-Review to catch any implementation gaps before QA testing begins.
- **Relevant architecture patterns and constraints**:
  - Enforce atomic `INCR` operations via Lua to prevent Redis usage metering drift under high concurrency (from `test-design-epic-08.md` R-003 / 8.8-PERF-001).
  - The middleware should handle both standard limits and the Per-Bid bypass seamlessly.
- **Source tree components to touch**:
  - Backend: `eusolicit-app/services/client-api/src/api/middlewares/usage_metering.py` (or equivalent).
  - Tests: `eusolicit-app/services/client-api/tests/integration/`
  - (Optional) `tests/perf/k6/k6-redis-incr.js` for the performance baseline if doing E2E load tests.
- **Testing standards summary**:
  - The test design requires explicitly validating 10,000 concurrent `INCR`s resulting in a count of exactly 10,000 (test design for 8.8-PERF-001 from `nfr-report.md`).
  - No commit() calls inside test bodies (isolated to fixture).
  - Full pytest summary line quoted in Test Results.

### Project Structure Notes

- Alignment with unified project structure: Ensure backend Python tests use the `asyncio` event loop properly when simulating concurrency (`asyncio.gather` for the concurrent test).

### References

- Cite all technical details with source paths and sections:
  - Epic 15 Spec: `eusolicit-docs/planning-artifacts/epics/epic-15.md#Story 15.2`
  - NFR Report (Redis INCR concurrency): `test_artifacts/nfr-report.md`
  - Subscription & Billing Test Design (R-003, 8.8-PERF-001): `eusolicit-docs/test-artifacts/test-design-epic-08.md`
  - Story 15.1 (Per-Bid SKU handoff contract): `eusolicit-docs/implementation-artifacts/15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config.md`

## Dev Agent Record

### Agent Model Used

Gemini 2.0 Flash

### Debug Log References

- `usage_gate.per_bid_bypass` log emitted when bypass is active.
- `usage_incremented` log emitted by `usage_meter_service`.
- `usage_gate.asymmetric_failure` log emitted when the per-company billing counter fails but tier counter succeeds.

### Completion Notes List

- [F1] Implemented workspace-scoped bypass key lookup by extracting `workspace_id` from JWT in `get_usage_gate`.
- [F2] Added try/except logging for asymmetric failure scenarios between per-tier and per-company billing counters.
- [F3] Added `test_concurrent_check_and_increment_enterprise` to exercise the full `check_and_increment` flow under concurrency.
- [F4] Cleaned up tests to properly delete seeded Redis keys using try/finally blocks and added `@pytest.mark.integration`.
- Implemented `_INCR_EXPIRE_LUA` in `usage_meter_service.py` for atomic per-company billing counters.
- Updated `UsageGateContext` to include `company_id`, `workspace_id`, and support `opportunity_id` for bypass checks.
- Wired `UsageGateContext.check_and_increment` to call `usage_increment` for all tiers (including Enterprise) to ensure billing tracking resolves NFR 8.8-PERF-001.
- Implemented and verified integration tests for bypass logic and 10,000-request concurrency.

### File List

- Modified: `eusolicit-app/services/client-api/src/client_api/core/usage_gate.py`
- Modified: `eusolicit-app/services/client-api/src/client_api/services/usage_meter_service.py`
- Modified: `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py`
- Modified: `eusolicit-app/services/client-api/tests/integration/test_usage_metering_bypass.py`

### Test Results

`5 passed, 7 warnings in 3.37s`

## Senior Developer Review

**Reviewer:** Claude (BMAD code review — re-review after F1-F4 fixes)
**Date:** 2026-04-27
**Outcome:** Approve

### Re-Review Summary (current)

All four findings from the prior review have been addressed and the 5
integration tests pass (`5 passed, 7 warnings in 3.37s`):

- **F1 (was BLOCKING) — RESOLVED.** `UsageGateContext` now carries
  `workspace_id` (`usage_gate.py:150`); `get_usage_gate` extracts it from
  the JWT (`usage_gate.py:330-339`); `check_and_increment` checks the
  workspace-scoped bypass key first, then falls back to the company-scoped
  key (`usage_gate.py:196-219`). New test
  `test_usage_middleware_bypasses_tier_limit_workspace_scoped` seeds the
  workspace-scoped key and asserts bypass works. Aligned with the
  webhook contract in `webhook_service.py:699-700`
  (`scoped_id = workspace_id_str or company_id_str`).
- **F2 (MINOR) — Mitigated.** Asymmetric failure between the per-tier
  counter and the per-company billing counter is now caught and logged
  via `usage_gate.asymmetric_failure` (`usage_gate.py:275-285`) so the
  daily Stripe sync task can reconcile. Joint atomicity via a single
  Lua script remains a future improvement but is deferrable.
- **F3 (MINOR) — RESOLVED.** Added
  `test_concurrent_check_and_increment_enterprise` that drives 10k
  concurrent `check_and_increment` calls through the full gate (Enterprise
  tier so the per-user branch is skipped) and asserts the per-company
  counter equals 10,000.
- **F4 (MINOR) — RESOLVED.** `pytestmark = [pytest.mark.asyncio,
  pytest.mark.integration]` is now set so `make test-unit` won't pick
  these up; all tests wrap seeded keys in `try/finally` and clean up.

### Spot checks performed

- `_USAGE_LUA` is correct: GET → conditional INCR with EXPIRE-on-first-write.
- `_INCR_EXPIRE_LUA` in `usage_meter_service.py` is the correct minimal pattern.
- Bypass branch is positioned before tier resolution and returns early — AC 3
  ("completely bypasses") is honored.
- Enterprise unlimited path still increments the per-company billing counter
  (resolves NFR 8.8-PERF-001 / AC 2 billing tracking).
- The unverified JWT decode in `get_usage_gate` is safe because the token is
  already verified upstream by `get_current_user`; it is only a redundancy
  (would be cleaner if `CurrentUser` carried `workspace_id` natively, but
  that is out of scope for this story).

### Verdict

REVIEW: Approve

### Historical Findings (resolved — kept for audit trail)

The original review flagged the items below; all have been addressed as
described above.

### Summary (original review)

The Lua-script atomicity work and the per-bid SKU bypass branch are sound and the
3 integration tests pass. However, the bypass key lookup is *not* aligned with
the contract Story 15.1 wrote in the webhook handler, which leaves a real gap
for workspace-scoped Per-Bid purchases. Two smaller issues are flagged below.

### Findings

#### F1 — [BLOCKING] Bypass lookup ignores workspace-scoped key

`client_api/core/usage_gate.py:191` only constructs

```python
bypass_key = f"add_on_active:{self.company_id}:{opportunity_id}"
```

…but the webhook in `client_api/services/webhook_service.py:699-708` writes
`add_on_active:{workspace_id_or_company_id}:{opportunity_id}` — i.e. when the
JWT carries `workspace_id`, the active key is workspace-scoped and the
company-scoped lookup will miss it. The Story 15.1 read-side endpoint in
`client_api/api/v1/per_bid.py:117-118` already does this correctly with
`workspace_or_company_id`. AC 1 and AC 3 both reference
`add_on_active:{workspace_or_company_id}:{opportunity_id}` literally, so this
is a direct AC miss for workspace users.

**Required fix:** plumb `workspace_id` into `UsageGateContext` (it is not
currently a field on `CurrentUser`, so this likely also needs a JWT-claim
extraction analogous to `per_bid.py:166-175`) and check the
workspace-scoped key first, falling back to the company-scoped key. Add a
test that seeds the workspace-scoped key and asserts bypass works.

The inline comments at lines 188-190 acknowledge this gap ("we knew
workspace_id"); the comment is not a substitute for handling it.

DEVIATION: usage gate bypass only checks company-scoped key while webhook stores workspace-scoped key when JWT has workspace_id
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

#### F2 — [MINOR] Tier check + per-company billing increment are not jointly atomic

After the `_USAGE_LUA` tier check succeeds, `check_and_increment` calls
`usage_increment(...)` (`usage_gate.py:256`) as a *separate* `EVAL` round
trip. If that second call fails (Redis hiccup, cancellation), the user's
per-tier counter has incremented but the per-company billing counter has
not, and Stripe sync will under-bill. Consider folding both writes into a
single Lua script keyed on both counter keys, or at minimum log/metric the
asymmetric-failure case so it can be reconciled by the daily sync task.

Same asymmetry on the Enterprise unlimited path (`usage_gate.py:208`): the
billing counter is incremented but there is no compensating logic if the
caller never actually consumes the AI summary downstream.

#### F3 — [MINOR] AC 2 concurrency test does not exercise the full flow

`test_concurrent_usage_incr_maintains_atomicity` calls `usage_increment`
directly. AC 2 is phrased as "10,000 concurrent metered AI requests …
tracked via Redis INCR", which implies the path reached through
`UsageGateContext.check_and_increment`. The current test verifies the inner
Lua script's atomicity (which is the right primitive) but not the wrapping
gate. Add a second concurrency test that calls
`check_and_increment("ai_summary", redis, response)` for an enterprise-tier
context (so the per-user limit branch is skipped) and asserts the
per-company counter equals N. This both raises confidence in the
integration and protects against a future refactor that drops the
`usage_increment` call from `check_and_increment`.

#### F4 — [MINOR] Tests missing pytest markers

`tests/integration/test_usage_metering_bypass.py` has only
`pytestmark = pytest.mark.asyncio`. Per `CLAUDE.md` ("Pytest markers — use
the right one"), integration tests requiring Redis should also carry
`@pytest.mark.integration`. Without it, `make test-unit` may attempt to run
these and fail when Redis is unavailable.

Also: tests do not clean up the seeded `add_on_active:*` and
`user:*:usage:*` keys; they rely on random UUIDs to avoid collisions. The
session-scoped `test_redis_client` fixture means these keys live for the
whole pytest session — fine in isolation but brittle if a future test
enumerates Redis keys.

### What's Good

- `_USAGE_LUA` correctly performs `GET` + conditional `INCR` atomically with
  proper `EXPIRE`-on-first-write semantics (E06-R-002 mitigation).
- `_INCR_EXPIRE_LUA` in `usage_meter_service.py` is the correct, minimal Lua
  pattern for an idempotent counter with TTL.
- Bypass branch is positioned before tier resolution and correctly returns
  early — AC 3's "completely bypasses" wording is honored for the
  company-scoped case.
- Enterprise-tier unlimited path now still increments the per-company
  billing counter, resolving the Stripe-sync gap called out in NFR
  8.8-PERF-001.
- The 10k `asyncio.gather` test is a strong proof of Lua atomicity:
  `len(set(results)) == 10000` is a sharper assertion than `max == 10000`
  alone.

### Verdict (original review)

REVIEW: Changes Requested

Address F1 before moving to QA. F2-F4 may be deferred but should be tracked.

## Known Deviations

### Detected by `3-code-review` at 2026-04-27T14:10:01Z (session 025af833-ba2c-49bd-967d-699d7ce1613f)

- usage gate bypass only checks company-scoped key while webhook stores workspace-scoped key when JWT has workspace_id _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- usage gate bypass only checks company-scoped key while webhook stores workspace-scoped key when JWT has workspace_id _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
