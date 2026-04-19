---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-19'
workflowType: 'atdd'
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-7-stripe-customer-portal-integration.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/services/client-api/tests/unit/test_checkout_session.py
  - eusolicit-app/services/client-api/tests/integration/test_checkout_upgrade_flow.py
  - eusolicit-app/e2e/specs/billing-checkout.spec.ts
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 8.7 — Stripe Customer Portal Integration

**Date:** 2026-04-19
**Author:** BMad TEA Agent
**Story:** 8-7-stripe-customer-portal-integration
**Status:** 🔴 RED PHASE — Failing tests generated, implementation pending
**Epic Test Design ID:** 8.7-API-001 (P1)

---

## Summary

| Category | Count |
|---|---|
| Unit Tests (service layer) | 7 |
| Unit Tests (endpoint layer) | 4 |
| Integration Tests | 1 |
| E2E Tests (skip scaffold) | 3 |
| **Total** | **15** |
| All tests marked skip (RED phase) | ✅ Yes |
| All tests assert expected behavior | ✅ Yes |

---

## TDD Red Phase Status

🔴 **CURRENT PHASE: RED** — All tests are decorated with `@pytest.mark.skip` (Python) or
`test.skip(...)` (TypeScript). They define the expected behavior but will NOT run until
the skip decorators are removed after implementation.

**Implementation required before removing skips:**
- Task 1: `create_portal_session()` in `billing_service.py`
- Task 2: `POST /api/v1/billing/portal/session` endpoint in `billing.py`
- Task 3: i18n keys in `en.json` + `bg.json`
- Task 4: Frontend portal mutation + "Manage Subscription" button

---

## Acceptance Criteria Coverage

| AC | Description | Test(s) | Priority | File |
|---|---|---|---|---|
| AC1 | Portal session endpoint returns `portal_url` | `test_create_portal_session_returns_portal_url`, `test_post_portal_session_returns_portal_url`, `test_portal_session_returns_url_for_authenticated_admin` | P1 | unit + integration |
| AC1 (return_url) | `return_url` passed to Stripe contains `/settings/billing` | `test_create_portal_session_return_url_contains_billing_path` | P1 | unit |
| AC2 (null customer) | `stripe_customer_id=None` → 422 `billing_not_configured` | `test_create_portal_session_missing_customer_returns_error`, `test_post_portal_session_returns_422_with_top_level_error_body` | P1 | unit |
| AC2 (no row) | No subscription row → 422 `billing_not_configured` | `test_create_portal_session_no_subscription_row_returns_error` | P1 | unit |
| AC3 | `stripe.api_key` not set → 422 `billing_not_configured` | `test_create_portal_session_stripe_not_configured_returns_error` | P1 | unit |
| AC4 | Portal button visible, calls API, opens `window.open(..., '_blank')` | `8.7-AC4: manage subscription portal button opens...` | P2 | E2E (skip) |
| AC4 (disabled) | Button disabled when `stripe_customer_id` is null | `8.7-AC4: manage subscription button is disabled...` | P2 | E2E (skip) |
| AC5 | Window focus triggers subscription refetch (TanStack default) | Verified via code review (no polling code needed) | P2 | N/A |
| AC6 | Admin-only: non-admin → 403, service NOT called | `test_post_portal_session_returns_403_for_non_admin` | P1 | unit |
| AC6 (auth) | No auth header → 401/403 | `test_post_portal_session_returns_401_without_auth` | P1 | unit |
| AC7 | Audit entry written on success | `test_create_portal_session_writes_audit_entry` | P1 | unit |
| AC8 | i18n: button uses `manageBilling` key (no hardcoded English) | `8.7-AC8: portal button label uses manageBilling...` | P3 | E2E (skip) |

**Resilience coverage (not in ACs but important):**

| Scenario | Test | File |
|---|---|---|
| `stripe.error.StripeError` caught, returns `stripe_error` dict | `test_create_portal_session_stripe_error_returns_error` | unit |
| 422 body at top level (NOT nested under FastAPI `detail`) | `test_post_portal_session_returns_422_with_top_level_error_body` | unit |
| No `webhook_events` row created by portal session | `test_portal_session_returns_url_for_authenticated_admin` (step 7) | integration |

---

## Test Files Generated

### Python Unit Tests (RED PHASE — all skipped)

**File:** `services/client-api/tests/unit/test_portal_session.py`

**Class: `TestCreatePortalSession`** (service layer — 7 tests)

| # | Test Name | AC | Expected Behavior |
|---|---|---|---|
| 1 | `test_create_portal_session_returns_portal_url` | AC1 | Returns `{"portal_url": "..."}` on success |
| 2 | `test_create_portal_session_missing_customer_returns_error` | AC2 | `stripe_customer_id=None` → error dict, Stripe NOT called |
| 3 | `test_create_portal_session_no_subscription_row_returns_error` | AC2 | No row → error dict, Stripe NOT called |
| 4 | `test_create_portal_session_stripe_not_configured_returns_error` | AC3 | `stripe.api_key=None` → error dict, Stripe NOT called |
| 5 | `test_create_portal_session_stripe_error_returns_error` | AC1 | `StripeError` caught → `{"error": "stripe_error", ...}` |
| 6 | `test_create_portal_session_writes_audit_entry` | AC7 | `write_audit_entry` called with `action_type="billing.portal_session_created"` |
| 7 | `test_create_portal_session_return_url_contains_billing_path` | AC1 | `return_url` kwarg contains `/settings/billing` |

**Class: `TestCreatePortalSessionEndpoint`** (endpoint layer — 4 tests)

| # | Test Name | AC | Expected Behavior |
|---|---|---|---|
| 8 | `test_post_portal_session_returns_portal_url` | AC1 | 200 + `portal_url` at top level |
| 9 | `test_post_portal_session_returns_422_with_top_level_error_body` | AC2 | 422 + `{"error": ..., "message": ...}` at top level (no `detail` wrapper) |
| 10 | `test_post_portal_session_returns_401_without_auth` | Auth | 401/403 without `Authorization` header |
| 11 | `test_post_portal_session_returns_403_for_non_admin` | AC6 | 403 for `role="contributor"`, service NOT called |

### Python Integration Test (RED PHASE — skipped)

**File:** `services/client-api/tests/integration/test_portal_session_flow.py`

| # | Test Name | Epic Test ID | Expected Behavior |
|---|---|---|---|
| 12 | `test_portal_session_returns_url_for_authenticated_admin` | **8.7-API-001 (P1)** | Seeds subscription, mocks Stripe, asserts 200 + `portal_url`, no `webhook_events` row |

### TypeScript E2E Tests (RED PHASE — all test.skip)

**File:** `e2e/specs/billing-checkout.spec.ts` _(appended to existing file)_

| # | Test Name | AC | Expected Behavior |
|---|---|---|---|
| 13 | `8.7-AC4: manage subscription portal button opens Stripe portal in new tab` | AC4 | Button visible, enabled, calls API, `window.open(..., '_blank')` |
| 14 | `8.7-AC4: manage subscription button is disabled when stripe_customer_id is null` | AC4 | Button visible but `disabled` when `stripe_customer_id=null` |
| 15 | `8.7-AC8: portal button label uses manageBilling i18n key (no hardcoded English)` | AC8 | BG locale shows translated text, not raw English "Manage Subscription" |

---

## Assumptions and Constraints

1. **Stripe Customer Portal must be configured in Stripe Dashboard** (manual one-time setup — not automated by this story). See `test-design-epic-08.md` Assumption #3.
2. **`asyncio.to_thread()`** is the canonical wrapper for all synchronous Stripe SDK calls — never call Stripe directly in async context.
3. **422 response body contract**: `JSONResponse(status_code=422, content=svc_result)` — NOT `raise HTTPException(422, detail=svc_result)` — to avoid FastAPI's `detail` nesting.
4. **`require_auth` patching**: patch at `"client_api.api.v1.billing.require_auth"` (module-level alias), not globally.
5. **`_billing_session()` context manager**: patch `"client_api.api.v1.billing.get_db_session"` in unit tests.
6. **`stripe_customer_id` already exists** in `client.subscriptions` (from Story 8.1) — no schema migration needed.
7. **AC5** (subscription refresh on window focus) is satisfied by TanStack Query's default `refetchOnWindowFocus: true` — no test needed beyond code review confirming the option is not disabled.

---

## Implementation Guidance

### Backend (Python)

**New function:** `billing_service.create_portal_session(company_id: UUID, session: AsyncSession) -> dict`

```python
# AC3: Guard
if not stripe.api_key:
    return {"error": "billing_not_configured", "message": "Stripe not configured — contact support"}

# AC2: Lookup
sub = await session.execute(select(Subscription).where(Subscription.company_id == company_id))
sub = result.scalar_one_or_none()
if sub is None or not sub.stripe_customer_id:
    return {"error": "billing_not_configured", "message": "Stripe customer not found — contact support"}

# AC1: Create session
portal_session = await asyncio.to_thread(
    stripe.billing_portal.Session.create,
    customer=sub.stripe_customer_id,
    return_url=f"{settings.frontend_url}/bg/settings/billing",
)

# AC7: Audit
await write_audit_entry(session, action_type="billing.portal_session_created",
    entity_type="subscription", entity_id=sub.id, after={"session_id": portal_session.id}, ...)

return {"portal_url": portal_session.url}
```

**New endpoint:** `POST /api/v1/billing/portal/session` in `billing.py`

Pattern: copy `POST /checkout/session` exactly — `require_auth`, `ROLE_HIERARCHY` rank check, `_billing_session()`, `JSONResponse(422, content=svc_result)`.

**New import alias in `billing.py`:**

```python
from client_api.services.billing_service import (
    create_checkout_session as create_checkout_session_svc,
    create_portal_session as create_portal_session_svc,  # ← add this
)
```

### Frontend (TypeScript)

**Portal mutation** (in `billing/page.tsx`):

```typescript
const portalMutation = useMutation({
  mutationFn: async () => {
    const res = await apiClient.post<{ portal_url: string }>('/api/v1/billing/portal/session', {});
    return res.data;
  },
  onSuccess: (data) => { window.open(data.portal_url, '_blank'); },
  onError: () => { toast.error(t('portalError')); },
});
```

**Button JSX:**

```tsx
<button
  onClick={() => portalMutation.mutate()}
  disabled={portalMutation.isPending || !subscription?.stripe_customer_id}
  data-testid="manage-subscription-btn"
>
  {portalMutation.isPending ? t('portalOpening') : t('manageBilling')}
</button>
```

---

## Green Phase Activation Instructions

After implementing Story 8.7 Tasks 1–4:

### 1. Remove skip decorators from Python tests

```bash
# Unit tests (11 skips to remove)
sed -i '/@pytest.mark.skip/d' \
  eusolicit-app/services/client-api/tests/unit/test_portal_session.py

# Integration test (1 skip to remove)
sed -i '/@pytest.mark.skip/d' \
  eusolicit-app/services/client-api/tests/integration/test_portal_session_flow.py
```

### 2. Run the test suite (green phase verification)

```bash
cd eusolicit-app

# Unit tests only (fast)
pytest services/client-api/tests/unit/test_portal_session.py -v

# Integration test (requires DB)
pytest services/client-api/tests/integration/test_portal_session_flow.py -v

# Full billing suite (regression check)
pytest services/client-api/tests/unit/test_billing_service.py \
       services/client-api/tests/unit/test_billing_webhook_endpoint.py \
       services/client-api/tests/unit/test_checkout_session.py \
       services/client-api/tests/unit/test_portal_session.py \
       services/client-api/tests/integration/test_portal_session_flow.py -v

# Coverage gate (must stay ≥ 80%)
make coverage
```

### 3. E2E tests (activate when frontend is implemented)

Remove `test.skip` from the three 8.7 scenarios in `billing-checkout.spec.ts`:
- `8.7-AC4: manage subscription portal button opens Stripe portal in new tab`
- `8.7-AC4: manage subscription button is disabled when stripe_customer_id is null`
- `8.7-AC8: portal button label uses manageBilling i18n key (no hardcoded English)`

---

## Next Recommended Workflows

1. **`bmad-agent-dev`** — Implement Story 8.7 Tasks 1–4 (remove skips, verify green)
2. **`bmad-testarch-automate`** — Expand test automation coverage after implementation
3. **`bmad-testarch-trace`** — Generate traceability matrix after all Epic 8 stories are green
