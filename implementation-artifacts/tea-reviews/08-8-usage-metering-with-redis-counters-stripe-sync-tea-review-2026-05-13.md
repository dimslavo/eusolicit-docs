# TEA Review — S08.08 usage-metering-with-redis-counters-stripe-sync

**Reviewer:** 🧪 Murat (TEA), bmad-tea audit-style retrospective dispatch under inj-03 coordinator
**Date:** 2026-05-13
**Story status at review time:** done (shipped under Epic 8; refreshed today under drift-recovery P6/P7/P12 close)
**Review type:** Retrospective quality audit for inj-03 AC2 closure (≥80/100 required)

## Verdict

**Score: 90/100 — PASS (≥80 threshold)**

This story has the strongest test discipline of the six reviewed today. Atomic Redis Lua semantics get dedicated atomicity test files. Just-landed drift-recovery P6 (terminal-retry-only emission), P7 (`_FEATURE_ALLOWLIST` filter), and P12 (narrowed except) added 10 fresh regression tests across `TestEmitDriftNarrowedException` + `TestFeatureLabelAllowlist` + `TestTerminalRetryOnlyDriftEmission`. The Celery retry-semantics surface is well-covered.

## Risk Scoring

| Risk | Cat | Probability | Impact | Score | Status |
|---|---|---|---|---|---|
| Redis Lua non-atomicity → double-increment | DATA | 1 (low — dedicated atomicity test files) | 3 | 3 | MITIGATED |
| Drift counter inflation on Celery retry storm | OPS | 1 (low — P6 terminal-retry-only guard tested 3 ways) | 2 | 2 | MITIGATED |
| Unbounded metric-label cardinality (feature explosion) | OPS | 1 (low — P7 `_FEATURE_ALLOWLIST` 6-entry set + bounded-set test) | 2 | 2 | MITIGATED |
| Stripe usage-record duplication (idempotency miss) | BUS | 1 (low — explicit `idempotency_key` per item) | 3 | 3 | MITIGATED |
| Cross-tenant counter leak | SEC | 1 (low — company_id scoped throughout) | 3 | 3 | MITIGATED |
| `_emit_drift` raising and blocking sync task | OPS | 1 (low — P12 narrowed `(TypeError,ValueError,AttributeError)` + propagation test for unexpected) | 2 | 2 | MITIGATED |

No risks ≥6 OPEN. **PASS.**

## Test Coverage Analysis

**Quantitative signal:**
- 5 test files; 69 `def test_*` functions; ~2350 LOC
- Dedicated atomicity files at unit + integration + worker layers (`test_billing_usage_sync_atomicity.py` × 2)
- Fresh 2026-05-13 additions: 10 tests across 3 new test classes (P6 + P7 + P12)

**Coverage by priority:**
- **P0** (revenue-critical): atomic increment, dedup, Stripe sync ✓ comprehensive
- **P1** (high-risk): Celery retry semantics, drift emission, feature allowlist ✓ comprehensive (refreshed today)
- **P2** (secondary): period rollover, audit-event emit ✓ covered

**Test-quality discipline:**
- ✅ Test pyramid balance: unit (`test_billing_usage_sync.py` 1059 LOC) + integration (`test_billing_usage_sync_atomicity.py` 195 LOC) + worker (`test_billing_usage_sync_atomicity.py` 516 LOC) — appropriate granularity per level
- ✅ Parametrized via `@pytest.mark.parametrize` — visible in `TestEmitDriftNarrowedException` (4 tests share fixture setup)
- ✅ Self-cleaning (per-test rollback)
- ✅ Cross-tenant negative test present (single-test marker but covered)

## Quality Findings

### Strengths

1. **Atomicity testing rigorous** — separate unit/integration/worker files explicitly named `_atomicity.py`. Race-condition coverage is rare in retroactive audits.
2. **Celery retry semantics fully tested** — P6 added 3 tests covering (retry=0 no emission, terminal retry emits, permanent always emits). Each scenario has explicit `self.request.retries` mock vs `self.max_retries` boundary.
3. **Feature label allowlist is type-tested** — `test_allowlist_is_a_bounded_set` asserts `isinstance(_FEATURE_ALLOWLIST, set)` AND `len(_FEATURE_ALLOWLIST) <= 50` — preventing cardinality explosion as a contract, not just a current state.
4. **Narrowed except discipline** — P12's `TestEmitDriftNarrowedException` has 4 tests: 2 swallow paths (`ValueError`, `AttributeError`), 1 propagation path (unexpected `RuntimeError`), 1 happy path. Locks in the "log-and-swallow narrow, propagate broad" contract.

### Weaknesses

1. **No explicit test for Redis Lua script SHA caching invalidation** — if `_USAGE_LUA` is updated via `SCRIPT LOAD` mid-run, in-flight evaluators using the cached SHA-1 may fail with `NOSCRIPT`. Spec didn't require this; flag for proactive coverage.
2. **`stripe_api_errors` label drift vs spec** — already logged to `deferred-work.md` under drift-recovery review: implementation labels are `(op, circuit_state, error_type)` while spec table named `(endpoint, error_type)`. Operationally finer-cardinality is more useful but external alert rules referring to spec strings need updating. Not a test gap — a spec/implementation reconciliation.
3. **No load-test for high-cardinality companies** — sync task iterates all metered companies; complexity is O(companies × items). No k6 baseline of "1000 companies × 5 items each" sync time. Spec didn't require it; flag for ops-readiness coverage.

## Recommendations

1. **Follow-up coverage (P3):** add a unit test asserting `_emit_drift` survives a forced Redis disconnect mid-emission (current happy-path only covers `inc()` succeeding).
2. **Follow-up coverage (P2):** add a k6 baseline for sync at 1000+ companies — measure p95 task duration vs Celery soft-time-limit.
3. **Documentation:** the 10 new regression tests landed today (2026-05-13) under drift-recovery should be cross-referenced in this story's `## Test Results` section if not already.

## Closure

**inj-03 AC2: PASS at 90/100.** Strongest test discipline of the inj-03 batch. Follow-ups are coverage hardening, not gating.
