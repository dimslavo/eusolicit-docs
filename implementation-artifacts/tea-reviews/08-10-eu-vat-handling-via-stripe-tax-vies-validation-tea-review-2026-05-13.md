# TEA Review — S08.10 eu-vat-handling-via-stripe-tax-vies-validation

**Reviewer:** 🧪 Murat (TEA), bmad-tea audit-style retrospective dispatch under inj-03 coordinator
**Date:** 2026-05-13
**Story status at review time:** done (shipped under Epic 8; refreshed today under drift-recovery P2 close)
**Review type:** Retrospective quality audit for inj-03 AC3 closure (≥80/100 required)

## Verdict

**Score: 86/100 — PASS (≥80 threshold)**

Tax-compliance surface is well-covered with strong VIES outage handling, fire-and-forget Redis enqueue testing (P2 added 2 tests today), and dual-call-site coverage (vies_service.py + api/v1/billing.py). One material weakness: no end-to-end test asserting "VAT-validated but Stripe-not-yet-synced" reconciliation actually drains the `vat.sync.pending` Redis Stream — the consumer side of P2 is implicit.

## Risk Scoring

| Risk | Cat | Probability | Impact | Score | Status |
|---|---|---|---|---|---|
| Stripe customer tax-id misaligned → wrong VAT charged | BUS+SEC | 2 (medium — circuit-open scenario can leave async drift) | 3 (B2B EU customer billed VAT incorrectly = legal/refund exposure) | 6 | MITIGATED (with caveat — see below) |
| VIES outage → false `invalid` rejection blocking signup | BUS | 1 (low — `vies_503_returns_pending` test covers) | 2 | 2 | MITIGATED |
| Fire-and-forget Redis failure swallows VAT sync silently | OPS | 1 (low — P2 fire-and-forget test asserts non-raise contract) | 2 | 2 | MITIGATED |
| Cross-tenant VAT data leak | SEC | 1 (low — company_id scoped) | 3 | 3 | MITIGATED |
| Stripe circuit open during VAT update → silent compliance gap | DATA | 2 (medium — drift-recovery P2 added reconciliation enqueue; consumer not tested end-to-end) | 3 | 6 | PARTIALLY MITIGATED |

One score=6 risk PARTIALLY mitigated. Per `risk-governance.md`, score≥6 demands mitigation plan — present (consumer worker exists per E28 webhook receiver design + sibling reconciler). Acceptable for CONCERNS-grade pass, but **flag for follow-up test** below.

## Test Coverage Analysis

**Quantitative signal:**
- 3 test files; 34 `def test_*` functions; ~2195 LOC
- Strong endpoint-level coverage (`test_vat_validate_endpoint.py` 817 LOC)
- Service-level coverage with VIES mock scenarios (`test_vies_service.py` 770 LOC — includes 2 P2 tests added today)
- Integration test for full flow (`test_vat_validation_flow.py` 608 LOC)

**Coverage by priority:**
- **P0** (revenue-critical/tax-compliance): valid VAT → Stripe customer.tax_ids set ✓; invalid VAT → 422 with structured error ✓; circuit-open → DB updated + Stripe sync enqueued ✓
- **P1** (high-risk): VIES 503/timeout/connection-error → 422 retry-later ✓; malformed VAT → 422 without calling VIES ✓ (early validation prevents external roundtrip)
- **P2** (edge): no stripe_customer_id case skipped cleanly ✓

**Test-quality discipline:**
- ✅ VIES external call mocked with explicit timeout + 503 + connection-error scenarios — proper external-dependency isolation
- ✅ Retry-with-backoff scenario tested
- ✅ P2 fire-and-forget contract asserts `validate_and_sync_company_vat` NEVER raises (`test_never_raises_on_exception` + `test_circuit_open_does_not_raise_to_caller`)
- ⚠️ Stripe customer.modify is mocked — no contract test against real Stripe schema for tax_ids payload shape

## Quality Findings

### Strengths

1. **Defensive validation order** — malformed VAT format rejected BEFORE hitting VIES (saves external roundtrip; `test_malformed_vat_returns_invalid_without_calling_vies` locks this contract). Cost-efficient + tax-compliance-conservative.
2. **Fire-and-forget contract explicit** — `test_never_raises_on_exception` asserts the function CANNOT propagate exceptions. Critical because it's invoked via `asyncio.create_task()` from the registration flow; a raised exception would silently kill the background task without surfacing.
3. **VIES outage paths comprehensive** — 503 + timeout + connection-error each have dedicated tests with explicit retry-after semantics.
4. **P2 today's additions:** `test_enqueues_vat_sync_pending_on_stripe_circuit_open` validates the Redis Stream enqueue is the reconciliation handoff.

### Weaknesses

1. **No consumer-side test for `vat.sync.pending` Redis Stream** — P2 enqueues to this stream when Stripe circuit is open, but I found no test asserting "a queued VAT-sync event eventually drains and updates the Stripe customer". This is the partially-mitigated risk above. **Recommended follow-up:** add integration test in `data-pipeline` or `client-api` worker (whichever consumer owns the stream) that asserts: (a) enqueue → consumer pickup → Stripe customer.modify called with correct tax_ids → ack.
2. **Stripe customer.modify schema brittle** — `tax_ids=[{"type": "eu_vat", "value": vat_number}]` is the canonical Stripe schema as of `stripe<9`. No contract test against Stripe's published schema. SDK upgrade risk (similar to S08.04 weakness #1).
3. **No load test on bulk-VAT-update scenarios** — companies updating VAT mid-billing-period could trigger a high-volume Stripe call burst. No k6 baseline. Low-priority — bulk updates are user-rare.

## Recommendations

1. **Follow-up test (P1):** consumer-side test for `vat.sync.pending` Redis Stream drain → Stripe customer.modify. This closes the score=6 risk above and would lift this story to 92/100.
2. **Follow-up coverage (P2):** Pact-style consumer-driven contract test against Stripe staging webhook payload schema for `customer.modify` response.
3. **Spec alignment:** the spec mentions "Stripe Tax" enablement as an operator-side prerequisite (Dashboard config). Add a deployment readiness check to ops runbook: "Verify Stripe Tax enabled at deploy time before this code runs in prod" — not a code test, but tracked in `deferred-work.md`.

## Closure

**inj-03 AC3: PASS at 86/100.** Tax-compliance posture is solid for code-side coverage. Consumer-side reconciliation drain test is the highest-value follow-up.
