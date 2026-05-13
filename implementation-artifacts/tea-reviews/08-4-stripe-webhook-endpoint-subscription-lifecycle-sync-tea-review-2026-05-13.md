# TEA Review — S08.04 stripe-webhook-endpoint-subscription-lifecycle-sync

**Reviewer:** 🧪 Murat (TEA), bmad-tea audit-style retrospective dispatch under inj-03 coordinator
**Date:** 2026-05-13
**Story status at review time:** done (shipped under Epic 8; refreshed today under drift-recovery P3/P4/P5 close)
**Review type:** Retrospective quality audit for inj-03 AC1 closure (≥80/100 required)

## Verdict

**Score: 88/100 — PASS (≥80 threshold)**

Risk-aligned coverage is strong across HMAC verification, idempotency, post-commit metric emission (just hardened today under drift-recovery P3/P4/P5), and cross-tenant isolation. Test pyramid balance is healthy: 8 unit files + 3 integration files; 64 test functions across ~3279 LOC. Refresh-by-drift-recovery left the surface in better shape than when it shipped.

## Risk Scoring (per `risk-governance.md` framework)

| Risk | Cat | Probability | Impact | Score | Status |
|---|---|---|---|---|---|
| HMAC bypass / signature spoofing | SEC | 1 (low — `compare_digest` + tests cover negative cases) | 3 | 3 | MITIGATED |
| Idempotency replay double-charge | BUS | 1 (low — dedup table + 7d window tested) | 3 | 3 | MITIGATED |
| Trial→paid ghost counter on retry | DATA | 1 (low — P3 fix landed today; post-commit emission) | 2 | 2 | MITIGATED |
| Active-subscription gauge stale on DB error | OPS | 2 (medium — try/except around refresh per P4) | 2 | 4 | MITIGATED |
| Hardcoded tier list drift on new tier | DATA | 1 (low — P5 derives from `TierAccessPolicy`) | 2 | 2 | MITIGATED |
| Cross-tenant webhook routing leak | SEC | 1 (low — company_id scoped throughout) | 3 | 3 | MITIGATED |

No risks ≥6 OPEN. Gate decision per `risk-governance.md`: **PASS**.

## Test Coverage Analysis

**Quantitative signal:**
- 8 unit test files + 3 integration files; 64 `def test_*` functions; ~3279 LOC
- HMAC + cross-tenant + idempotency markers: 5 files contain explicit coverage
- Recent additions (drift-recovery 2026-05-13): trial→paid post-commit assertion + gauge-refresh failure-doesn't-affect-webhook assertion (P3, P4)

**Coverage by priority (per `test-priorities-matrix.md`):**
- **P0** (revenue-critical, security-critical): HMAC sig verification, idempotency, dedup ✓ comprehensive
- **P1** (core flow): subscription create/update/cancel lifecycle, trial→paid transition, addon checkout ✓ covered
- **P2** (secondary): per-bid webhook, enterprise invoice paths ✓ covered

**Test-quality discipline (per `test-quality.md` DoD):**
- ✅ No hard waits (`grep "time.sleep" → 2 hits` — but in `test_webhook_billing_meta.py` they are intentional aging/clock-advance, not flow-control waits; acceptable)
- ✅ Explicit assertions (spot-check shows `assert response.status_code == ...` in test bodies)
- ✅ Self-cleaning (per-test transaction rollback via `db_session` fixture per project CLAUDE.md gold-standard)
- ✅ Cross-tenant negative tests present (per project memory rule — confirmed via grep on `cross_tenant|company_a|company_b`)

## Quality Findings

### Strengths

1. **HMAC discipline solid** — `hmac.compare_digest()` usage verified per Rule 48; signature-mismatch path tested in `test_billing_webhook_endpoint.py`
2. **Idempotency surface comprehensive** — both unit + integration coverage of dedup table, 7-day window, and concurrent-replay (`test_per_bid_webhook_idempotency.py`)
3. **Post-commit emission discipline** — drift-recovery P3 added `test_trial_to_paid_post_commit` assertion; ghost-counter regression now guarded
4. **Stream-alignment tests** — `test_webhook_service_stream_alignment.py` validates Redis Streams event-publication contract for downstream consumers (Epic 9 notification + Epic 5 pipeline)

### Weaknesses (drive follow-up coverage, not gate failure)

1. **Stripe Event.previous_attributes JSON shape brittle** — trial→paid detection at `webhook_service.py:944` relies on `previous_attributes.get("status") == "trialing"`. If Stripe's webhook payload ever ships this as an enum object instead of a string (SDK upgrade), the conversion counter silently stops firing. No regression test asserts the canonical Stripe payload shape; if the stripe-python SDK pins drift, this could silently break.
2. **No load-test against high-rate webhook bursts** — Stripe can burst-deliver after their side recovers from outage. No k6 test in `tests/integration/` exercises this. Spec didn't require it; flagging for proactive coverage.
3. **`test_stripe_webhook_flow.py` integration test asserts behaviour against mocked Stripe Event objects** — good for determinism but doesn't catch contract drift if Stripe ships a new event-type schema. Consider Pact-style consumer-driven contract test against a recorded Stripe staging webhook payload (per `contract-testing.md` knowledge fragment).

## Recommendations

1. **Follow-up coverage (P2):** add a regression test asserting Stripe `previous_attributes.status` is a string under both `stripe<9` and the next SDK target before any SDK upgrade.
2. **Follow-up coverage (P3):** add a k6 burst-replay test (50 events in 5s, all duplicates) to validate idempotency under load.
3. **Documentation:** add a §Salvaged-patterns comment in the test file noting which fixtures (`db_session`, `clean_redis`, `client_api`) provide the test isolation, so future test authors don't reinvent.

## Closure

**inj-03 AC1: PASS at 88/100.** Story closes for retrospective TEA review. Follow-up coverage tracked above is hardening, not gating.
