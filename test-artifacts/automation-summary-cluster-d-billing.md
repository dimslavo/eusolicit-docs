---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-identify-targets
  - step-03-generate-tests
  - step-04-validate-and-summarize
lastStep: step-04-validate-and-summarize
lastSaved: '2026-05-09'
workflowType: bmad-testarch-automate
mode: bmad-integrated
scope: cluster-D-billing-api
batch: P4-b
storyKeys:
  - 8-4-stripe-webhook-endpoint
  - 8-6-checkout-session-endpoint
  - 8-7-customer-portal-endpoint
  - 8-12-public-tier-policies
  - 8-13-invoices-endpoint
detectedStack: backend
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/services/client-api/src/client_api/api/v1/billing.py
  - eusolicit-app/services/client-api/src/client_api/core/security.py
  - eusolicit-app/services/client-api/tests/unit/test_billing_webhook_endpoint.py
  - eusolicit-app/services/client-api/tests/unit/test_billing_usage_endpoint.py
---

# Automation Summary: P4-b — billing.py route contract (cluster D, sub-run 2)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P4-b (cluster D, gap-fill API coverage, sub-run 2)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

Per the rollout-plan inventory:

> "service-layer (Stripe webhook, VAT, portal, addon checkout) covered in
>  unit; no request-level test for billing routes
>  → API-level happy path + 401/403 + tier-gate negative"

The existing billing service-layer suite is comprehensive
(`test_checkout_session.py`, `test_portal_session.py`,
`test_addon_checkout_service.py`, `test_billing_service_per_bid.py`,
`test_usage_meter_service.py`, etc.). But the two **route-level** files
that already exist on disk —
`tests/unit/test_billing_webhook_endpoint.py` and
`tests/unit/test_billing_usage_endpoint.py` — are entirely
`@pytest.mark.skip()`-decorated RED-phase placeholders dating to
Story 8.4 / 8.8 implementation. Six other billing endpoints
(`/tiers`, `/subscription`, `/checkout/session`, `/portal/session`,
`/addon/checkout/session`, `/invoices`) had **zero** request/response
coverage.

P4-b ships a **GREEN** route-contract spec that exercises 5 of those 6
through `ASGITransport`, with all external side effects (Stripe SDK,
Redis, DB session, `require_auth`) patched at the documented
module-level aliases.

---

## Files Created

### `services/client-api/tests/api/test_billing_route_contract.py` (NEW)

9 tests across 5 endpoints, marked `@pytest.mark.api` per the project's
"needs running services" definition (in this case the patched in-process
FastAPI app):

| # | Endpoint | Scope | Description |
|---|---|---|---|
| 1 | GET `/billing/tiers` | Public list | No auth required. Mock 5 shuffled `TierAccessPolicy` rows; assert canonical order `free → starter → professional → pro_plus → enterprise`; verify the 7 most important fields in `TierPolicyResponse` are present. |
| 2 | GET `/billing/subscription` | Auth happy path | Mock `require_auth` + DB session returning a `Subscription` row; assert 200 with `tier`, `status`, `is_trial`, `stripe_customer_id` keys. |
| 3 | GET `/billing/subscription` | No row | Session returns `None`; expect 404 with body `{"detail": "subscription_not_found"}`. |
| 4 | GET `/billing/subscription` | Missing bearer | No `Authorization` header; assert 401 or 403 (HTTPBearer auto-error variance). |
| 5 | POST `/billing/checkout/session` | Admin happy path | Patch service to return `{checkout_url, session_id}`; expect 200 with the same payload. |
| 6 | POST `/billing/checkout/session` | Non-admin RBAC | User role = `bid_manager` → `ForbiddenError` → 403. |
| 7 | POST `/billing/checkout/session` | Service error | Service returns `{"error": "stripe_not_configured", "message": …}` → 422; assert top-level error/message (NOT wrapped in `detail`). |
| 8 | POST `/billing/portal/session` | Non-admin RBAC | User role = `contributor` → `ForbiddenError` → 403 (AC6). |
| 9 | GET `/billing/invoices` | Empty list | Session returns `[]`; expect 200 with body `[]` (NOT 404). |

### Helpers (test-local, ~25 lines)

- `_make_user(role)` — `MagicMock` shaped like `CurrentUser` (`user_id`,
  `company_id`, `role`, `subscription_tier`).
- `_make_session_with_scalar_one_or_none(value)` — `AsyncMock` whose
  `execute().scalar_one_or_none()` returns `value`.
- `_make_session_with_scalars(values)` — same but for `.scalars().all()`.
- `_build_app()` — defers `from client_api.main import app` so the import
  doesn't run at module load (matches the pattern in
  `test_billing_webhook_endpoint.py`).

---

## Validation

| Check | Result |
|---|---|
| `ruff check` filtered for new file | ✅ All checks passed |
| `pytest --collect-only` (Python 3.13.5) | ✅ **9 tests collected** in 1 module |
| Active vs skip split | 9 active / 0 skip / 0 xfail |
| Local runtime | ❌ Host venv lacks service deps (per saved memory `project_test_execution_environment.md`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| `GET /api/v1/billing/tiers` (no auth) | `billing.py:575` | ✅ |
| `_TIER_ORDER` canonical ordering | `billing.py:572` | ✅ |
| `TierPolicyResponse` 11 fields | `billing.py:551-569` | ✅ |
| `GET /api/v1/billing/subscription` returns `{tier, status, is_trial, trial_end, current_period_end, stripe_customer_id}` | `billing.py:194-226` | ✅ |
| `subscription_not_found` 404 body | `billing.py:217` | ✅ |
| `POST /api/v1/billing/checkout/session` admin-only RBAC | `billing.py:283-292` (`ROLE_HIERARCHY` rank check) | ✅ |
| Service error → 422 with top-level `error`/`message` (NOT wrapped in `detail`) | `billing.py:301-305` | ✅ |
| `POST /api/v1/billing/portal/session` admin-only RBAC | `billing.py:325-332` | ✅ |
| `GET /api/v1/billing/invoices` returns `[]` not 404 | `billing.py:625-660` | ✅ |
| Module-level patch points (`require_auth`, `get_db_session`, service aliases) | `billing.py:90, 97, 66-79` | ✅ |
| `_billing_session()` accepts both async-gen and direct mock values | `billing.py:104-120` (the `inspect.isasyncgen` branch) | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up                            # full stack — only required if a future
                                   # test exercises a real service call
pytest services/client-api/tests/api/test_billing_route_contract.py -v -m api
```

The new spec runs in-process via `ASGITransport`; no infra required for
the 9 tests as written. CI should be able to execute them in the
`make test-service SVC=client-api` pass that already runs the existing
RED-phase webhook/usage tests.

### Risks / open questions for the CI pass

1. **HTTPBearer auto-error variance** — test #4 accepts 401 OR 403 for
   the missing-bearer case (same as P4-a NPS #8). FastAPI's `HTTPBearer`
   surfaces 403 by default when `auto_error=True`; 401 if a custom
   handler is installed. Until the project pins this behaviour, the
   dual-accept assertion stays.

2. **`require_auth` patch lifecycle** — the production code aliases
   `require_auth = get_current_user` at module import (`billing.py:90`),
   then calls it via `await require_auth(credentials)` inside each
   endpoint. Patching the alias works because the route reads it
   dynamically; if a future refactor inlines `get_current_user` calls
   directly, every patch would need updating.

3. **`get_db_session` mock returns the session directly** — the
   `_billing_session()` context manager (`billing.py:104-120`) explicitly
   handles both the async-generator (production) and the direct-value
   (test) shape. Returning `AsyncMock()` from the patch satisfies the
   non-asyncgen branch. Watch for any future refactor that removes the
   `inspect.isasyncgen` fork.

4. **Missing endpoint coverage** — five endpoints in `billing.py` remain
   without GREEN route-level tests:
   - `POST /billing/webhooks/stripe` — the existing
     `test_billing_webhook_endpoint.py` is RED (`@pytest.mark.skip`); a
     parallel GREEN file (or removal of the skip decorators) is the
     follow-up.
   - `POST /billing/addon/checkout/session` — admin/bid_manager RBAC,
     422 on `addon_not_configured`.
   - `GET /billing/addon/status` — purchased/not-purchased branches.
   - `POST /billing/vat/validate` — needs `vies_service.validate_vat_number`
     mock + Stripe `Customer.modify` mock + audit-write mock. (3 branches:
     valid/invalid/pending.)
   - `GET /subscription/usage` — needs `_get_redis` patch + tier policy
     mock; the existing `test_billing_usage_endpoint.py` is RED.

5. **Tier-gate negative not exercised here** — `billing.py` endpoints
   guard on RBAC role but NOT on subscription tier (the tier-gate is
   applied to *consumer* endpoints like NPS or analytics, not to billing
   itself). The plan's "tier-gate negative" line item is therefore
   irrelevant for the billing module — flagged for the rollout-plan
   author. Consumer-side tier-gate negatives are already covered by
   `test_nps_feedback_workspace_isolation.py` (P3-style coverage from
   prior work).

6. **`POST /vat/validate` — DB write under test** — the route mutates
   `companies.vat_validation_status` and conditionally calls Stripe.
   This is the most fixture-heavy endpoint to add and was deliberately
   deferred to a follow-up; the existing service-layer test
   (`vies_service` direct call coverage) plus the manual smoke test
   keeps the operational risk acceptable for now.

7. **`/billing/checkout/session` Pydantic literal validation** — the
   `tier` field accepts `"starter" | "professional" | "pro_plus" | "enterprise"`
   only. A future test for "POST with invalid tier returns 422" would
   complete the validation surface but adds little signal beyond what
   FastAPI's standard 422 path already does.

8. **Deferred follow-ups for this surface** (in priority order):
   - Lift `test_billing_webhook_endpoint.py` from RED → GREEN (remove the
     module-wide `@pytest.mark.skip`; verify the patches still resolve).
   - Lift `test_billing_usage_endpoint.py` from RED → GREEN (same).
   - VAT validate happy-path + invalid + Stripe-sync skip-when-no-customer.
   - Addon checkout session + addon status branches.

---

## Coverage Delta

- Before P4-b: route-level coverage for `billing.py` was effectively
  **zero** (all existing route tests were RED-phase
  `@pytest.mark.skip()`). The 9 ship-critical endpoints relied on
  service-layer tests which exercise the business logic but never bind
  it to the FastAPI request/response surface.
- After P4-b: **9 active** route-contract tests covering 5 of those 9
  endpoints (`/tiers`, `/subscription`, `/checkout/session`,
  `/portal/session`, `/invoices`) with happy paths, RBAC negatives, and
  service-error envelope verification. Webhook + usage + VAT + addon
  remain as scoped follow-ups.

---

## Cluster D (P4) running totals (through P4-b)

| Sub-run | Spec file | Active | Skip / xfail |
|---|---|---:|---:|
| P4-a | `test_nps_feedback_route_contract.py` | 8 | 0 |
| P4-b | `test_billing_route_contract.py` | 9 | 0 |
| **Total** | **2 spec files** | **17** | **0** |

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P4-c** — Cross-service notification flow (largest of the three —
  needs `make up` not just `make infra`; client-api emits event →
  notification consumer → outbox row + delivery side-effect with mocked
  SendGrid).

After P4-b runs green in CI, update the rollout-plan checklist line.
