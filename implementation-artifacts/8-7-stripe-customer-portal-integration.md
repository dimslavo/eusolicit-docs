# Story 8.7: Stripe Customer Portal Integration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a company admin,
I want to access the Stripe Customer Portal directly from the subscription management page,
so that I can self-service plan changes, subscription cancellation, and payment method updates without contacting support.

## Acceptance Criteria

1. **AC1 — Portal Session Endpoint (Backend)** — `POST /api/v1/billing/portal/session` looks up the authenticated company's `stripe_customer_id` from the `subscriptions` table, calls `stripe.billing_portal.Session.create(customer=..., return_url=...)` via `asyncio.to_thread()`, and returns `{"portal_url": "https://billing.stripe.com/..."}`. Requires admin authentication. Returns HTTP 200.

2. **AC2 — Missing Customer Guard** — If the company has no `stripe_customer_id` in the subscriptions table (Stripe provisioning from Story 8.1 failed or was skipped), return HTTP 422 with `{"error": "billing_not_configured", "message": "Stripe customer not found — contact support"}`. Do NOT call the Stripe API.

3. **AC3 — Stripe Not Configured Guard** — If `stripe.api_key` is not set at runtime, return HTTP 422 with `{"error": "billing_not_configured", "message": "Stripe not configured — contact support"}` without calling the Stripe API.

4. **AC4 — Frontend Portal Button** — The billing settings page (`/settings/billing`) renders a "Manage Subscription" button. On click, the button calls `POST /api/v1/billing/portal/session` and, on success, opens the portal URL in a new browser tab via `window.open(portal_url, "_blank")`. The button shows a loading/spinner state while the API call is in flight and is disabled when `stripe_customer_id` is null.

5. **AC5 — Subscription State Refresh on Return** — When the user switches back to the billing settings page from the portal tab, the subscription state automatically refreshes by re-fetching `GET /api/v1/billing/subscription`. TanStack Query's `refetchOnWindowFocus: true` (default) handles this without any additional polling code.

6. **AC6 — Admin Only** — Only company admins may generate a portal session. Non-admin authenticated users receive HTTP 403.

7. **AC7 — Audit Trail** — On successful portal session creation, write an audit entry with `action_type="billing.portal_session_created"`, `entity_type="subscription"`, `entity_id=subscription.id`, `after={"session_id": "<session_id>"}`.

8. **AC8 — i18n** — All user-facing strings on the billing settings page use `useTranslations("billing")`. The portal button uses the existing `manageBilling` key ("Manage Subscription"). New loading-state and error-toast strings (`portalOpening`, `portalError`) are added to both BG and EN message files. No hardcoded English strings.

## Tasks / Subtasks

- [x] Task 1 — Add `create_portal_session()` to `billing_service.py` (AC: 1, 2, 3, 7)
  - [x] 1.1 Add the function after `create_checkout_session()` in `services/client-api/src/client_api/services/billing_service.py`:
    ```python
    async def create_portal_session(
        company_id: UUID,
        session: AsyncSession,
    ) -> dict:
        """Create a Stripe Customer Portal session for self-service billing management.

        AC1: Calls stripe.billing_portal.Session.create() with customer ID and return URL.
        AC2: Returns error dict if stripe_customer_id is missing.
        AC3: Returns error dict if stripe.api_key is not configured.
        AC7: Writes audit entry after session creation.

        Parameters
        ----------
        company_id:
            UUID of the company requesting the portal session.
        session:
            Caller-provided AsyncSession (managed externally; function only flushes).

        Returns
        -------
        dict
            On success: {"portal_url": "https://billing.stripe.com/p/session/..."}
            On error:   {"error": "<code>", "message": "<human text>"}
        """
        # AC3: Guard against unconfigured Stripe
        if not stripe.api_key:
            logger.warning(
                "portal_session_stripe_not_configured",
                company_id=str(company_id),
            )
            return {
                "error": "billing_not_configured",
                "message": "Stripe not configured — contact support",
            }

        settings = get_settings()

        # AC2: Look up subscription row for stripe_customer_id
        result = await session.execute(
            select(Subscription).where(Subscription.company_id == company_id)
        )
        sub = result.scalar_one_or_none()

        if sub is None or not sub.stripe_customer_id:
            logger.warning(
                "portal_session_no_stripe_customer",
                company_id=str(company_id),
            )
            return {
                "error": "billing_not_configured",
                "message": "Stripe customer not found — contact support",
            }

        # Build return URL — send user back to billing settings page after portal
        return_url = f"{settings.frontend_url}/bg/settings/billing"

        try:
            portal_session = await asyncio.to_thread(
                stripe.billing_portal.Session.create,
                customer=sub.stripe_customer_id,
                return_url=return_url,
            )
        except stripe.error.StripeError as exc:
            logger.error(
                "portal_session_creation_failed",
                company_id=str(company_id),
                error=str(exc),
                error_type=type(exc).__name__,
            )
            return {
                "error": "stripe_error",
                "message": "Failed to create portal session — please try again",
            }

        logger.info(
            "portal_session_created",
            company_id=str(company_id),
            session_id=portal_session.id,
        )

        # AC7: Audit trail
        await write_audit_entry(
            session,
            action_type="billing.portal_session_created",
            entity_type="subscription",
            entity_id=sub.id,
            after={"session_id": portal_session.id},
            company_id=company_id,
        )
        await session.flush()

        return {"portal_url": portal_session.url}
    ```

- [x] Task 2 — Add `POST /api/v1/billing/portal/session` endpoint to `billing.py` (AC: 1, 2, 3, 6)
  - [x] 2.1 Update the import from `billing_service` to include `create_portal_session`:
    ```python
    from client_api.services.billing_service import (
        create_checkout_session as create_checkout_session_svc,
        create_portal_session as create_portal_session_svc,
    )
    ```
  - [x] 2.2 Add the endpoint after `create_checkout_session_endpoint`:
    ```python
    @router.post("/portal/session", status_code=200)
    async def create_portal_session_endpoint(
        request: Request,
    ) -> JSONResponse:
        """Create a Stripe Customer Portal session for the authenticated company.

        Returns {"portal_url": "https://billing.stripe.com/..."} on success.
        Returns HTTP 422 with top-level {"error": ..., "message": ...} if
        stripe_customer_id or stripe.api_key is not configured (AC2/AC3).

        Auth: company admin only (AC6).
        """
        credentials = await http_bearer(request)
        current_user: CurrentUser = await require_auth(credentials)

        # AC6: admin-only — same inline rank-check pattern as POST /checkout/session
        user_rank = ROLE_HIERARCHY.get(current_user.role, 0)
        admin_rank = ROLE_HIERARCHY.get("admin", 0)
        if user_rank < admin_rank:
            raise ForbiddenError("Admin role required to access billing portal")

        async with _billing_session() as session:
            svc_result = await create_portal_session_svc(
                company_id=current_user.company_id,
                session=session,
            )

        if "error" in svc_result:
            # Top-level error body — NOT nested under FastAPI's default "detail" key
            return JSONResponse(status_code=422, content=svc_result)
        return JSONResponse(status_code=200, content=svc_result)
    ```

- [x] Task 3 — Add new i18n keys for portal loading/error states (AC: 8)
  - [x] 3.1 Add to `frontend/apps/client/messages/en.json` inside the `"billing"` object (after `"trialEndsOn"`):
    ```json
    "portalOpening": "Opening billing portal…",
    "portalError": "Could not open billing portal. Please try again."
    ```
  - [x] 3.2 Add matching keys to `frontend/apps/client/messages/bg.json`:
    ```json
    "portalOpening": "Отваряне на портала за фактуриране…",
    "portalError": "Порталът за фактуриране не може да бъде отворен. Моля, опитайте отново."
    ```

- [x] Task 4 — Frontend: add "Manage Subscription" portal button to billing settings page (AC: 4, 5, 8)
  - [x] 4.1 In `frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx`, add a portal session mutation after the existing `checkoutMutation`:
    ```typescript
    const portalMutation = useMutation({
      mutationFn: async () => {
        const res = await apiClient.post<{ portal_url: string }>(
          '/api/v1/billing/portal/session',
          {}
        );
        return res.data;
      },
      onSuccess: (data) => {
        // AC4: open portal in a new tab — NOT window.location.href
        window.open(data.portal_url, '_blank');
      },
      onError: () => {
        toast.error(t('portalError'));
      },
    });
    ```
  - [x] 4.2 Add the "Manage Subscription" button in the billing page JSX (below the tier selection cards section):
    ```tsx
    <div className="mt-6 border-t pt-6">
      <button
        onClick={() => portalMutation.mutate()}
        disabled={portalMutation.isPending || !subscription?.stripe_customer_id}
        data-testid="manage-subscription-btn"
        className="..."
      >
        {portalMutation.isPending ? t('portalOpening') : t('manageBilling')}
      </button>
    </div>
    ```
  - [x] 4.3 AC5 — Confirm the existing `useQuery` for subscription does NOT have `refetchOnWindowFocus: false`. TanStack Query defaults `refetchOnWindowFocus: true`, which triggers an automatic re-fetch when the user returns to this tab from the portal. No additional polling code is needed.
  - [x] 4.4 Add `useToast` import if not already present. The `toast` function is called on portal errors via `onError`.

- [x] Task 5 — Unit tests (AC: 1, 2, 3, 6, 7)
  - [x] 5.1 Create `services/client-api/tests/unit/test_portal_session.py`:

    **Test class: `TestCreatePortalSession`** (service layer)
    - `test_create_portal_session_returns_portal_url` — Mock `asyncio.to_thread` to return `MagicMock(id="bps_test_001", url="https://billing.stripe.com/p/session/bps_test_001")`. Seed subscription with `stripe_customer_id="cus_test"`. Call `await create_portal_session(company_id, session)`. Assert result `== {"portal_url": "https://billing.stripe.com/p/session/bps_test_001"}`. Assert `asyncio.to_thread` called with `stripe.billing_portal.Session.create`.
    - `test_create_portal_session_missing_customer_returns_error` — Seed subscription with `stripe_customer_id=None`. Assert result `== {"error": "billing_not_configured", "message": "Stripe customer not found — contact support"}`. Assert `asyncio.to_thread` NOT called.
    - `test_create_portal_session_no_subscription_row_returns_error` — No subscription row in DB. Assert result `== {"error": "billing_not_configured", ...}`. Assert Stripe NOT called.
    - `test_create_portal_session_stripe_not_configured_returns_error` — Temporarily set `stripe.api_key = None`. Assert result `== {"error": "billing_not_configured", "message": "Stripe not configured — contact support"}`. Assert Stripe NOT called.
    - `test_create_portal_session_stripe_error_returns_error` — Mock `asyncio.to_thread` to raise `stripe.error.StripeError("internal_server_error")`. Assert result `== {"error": "stripe_error", "message": "Failed to create portal session — please try again"}`. Assert no exception propagates.
    - `test_create_portal_session_writes_audit_entry` — Mock `asyncio.to_thread` (success). Mock `write_audit_entry`. Assert `write_audit_entry` called with `action_type="billing.portal_session_created"`, `entity_type="subscription"`, `after={"session_id": "bps_test_001"}`.
    - `test_create_portal_session_return_url_contains_billing_path` — Assert the `return_url` kwarg passed to `stripe.billing_portal.Session.create` (via `asyncio.to_thread`) contains `/settings/billing`.

    **Test class: `TestCreatePortalSessionEndpoint`** (endpoint layer)
    - `test_post_portal_session_returns_portal_url` — Mock `create_portal_session_svc` returning `{"portal_url": "https://billing.stripe.com/..."}`. Authenticated POST (admin user, `role="admin"`) to `/api/v1/billing/portal/session`. Assert 200 + `portal_url` in response body at top level.
    - `test_post_portal_session_returns_422_with_top_level_error_body` — Mock service returning `{"error": "billing_not_configured", "message": "Stripe customer not found — contact support"}`. Assert 422. Assert response body has top-level `"error"` key (NOT nested under `"detail"`).
    - `test_post_portal_session_returns_401_without_auth` — No Authorization header. Assert 401.
    - `test_post_portal_session_returns_403_for_non_admin` — Authenticated as contributor role (`role="contributor"`). Assert 403. Assert service function NOT called.

- [x] Task 6 — Integration test (AC: P1 8.7-API-001)
  - [x] 6.1 Create `services/client-api/tests/integration/test_portal_session_flow.py`:
    - **`test_portal_session_returns_url_for_authenticated_admin`** (mirrors `8.7-API-001`): Seed subscription with `stripe_customer_id="cus_test"`, `tier="professional"`, `status="active"`, `is_trial=False`. Mock `stripe.billing_portal.Session.create` (via `asyncio.to_thread`) returning `MagicMock(id="bps_int_001", url="https://billing.stripe.com/p/session/bps_int_001")`. Auth as company admin. POST to `/api/v1/billing/portal/session`. Assert 200. Assert `response.json()["portal_url"] == "https://billing.stripe.com/p/session/bps_int_001"`. Assert no `webhook_events` row created (portal creation does not touch the webhook deduplication table).

- [x] Task 7 — E2E test scaffold (AC: 8.7-API-001 future)
  - [x] 7.1 Add a `test.skip` scenario to `frontend/e2e/specs/billing-checkout.spec.ts`:
    ```typescript
    test.skip('manage subscription portal button opens Stripe portal (8.7)', async ({ page }) => {
      // Activation story: bmad-testarch-atdd for Epic 8
      // Steps:
      // 1. Log in as company admin with active subscription
      // 2. Navigate to /settings/billing
      // 3. Assert "Manage Subscription" button is visible (data-testid="manage-subscription-btn")
      // 4. Mock POST /api/v1/billing/portal/session to return a test portal URL
      // 5. Click the button
      // 6. Assert window.open called with portal URL and '_blank'
    });
    ```

## Dev Notes

### Critical Architecture Constraints

- **`billing.py` router is already registered** via `api_v1_router.include_router(billing_v1.router)` in `main.py`. Do NOT add another `include_router`. The new `POST /portal/session` endpoint just extends the existing router object in `billing.py`.
- **Stripe SDK is synchronous** — ALL Stripe API calls (`stripe.billing_portal.Session.create`, etc.) MUST be wrapped in `await asyncio.to_thread(...)`. Never call Stripe SDK methods directly in an async context. `asyncio.to_thread()` is the Python 3.9+ canonical idiom — NOT `loop.run_in_executor(None, ...)`.
- **`_billing_session()` context manager** — The new endpoint MUST use the module-level `_billing_session()` async context manager (defined at `billing.py:67-83`), NOT `Depends(get_db_session)`. This makes the session patchable in unit tests. See the existing `POST /checkout/session` endpoint for the pattern.
- **`require_auth` alias** — Call `await require_auth(credentials)` at runtime inside the endpoint body (not as a `Depends()` parameter). The module-level alias `require_auth = get_current_user` is already defined at `billing.py:60`. This makes it patchable in unit tests via `patch("client_api.api.v1.billing.require_auth", ...)`.
- **Admin RBAC** — Use the same inline `ROLE_HIERARCHY` rank-check pattern as `POST /checkout/session` (lines 219–225). Raise `ForbiddenError("Admin role required to access billing portal")`. `ForbiddenError` from `eusolicit_common.exceptions` is already imported at `billing.py:36`.
- **422 response body** — Use `JSONResponse(status_code=422, content=svc_result)` — NOT `raise HTTPException(status_code=422, detail=svc_result)`. FastAPI's default exception handler nests under `"detail"`, breaking the `{"error": ..., "message": ...}` contract. The existing checkout endpoint demonstrates the correct pattern at `billing.py:237`.
- **No new Alembic migration needed** — No schema changes. `stripe_customer_id` column already exists from Story 8.1.
- **No new config settings needed** — `return_url` uses `settings.frontend_url` (already in `ClientApiSettings`). No `stripe_portal_configuration_id` needed for default portal configuration usage.
- **Stripe API version** — Pinned at `"2024-06-20"` in `config.py:114`. The `billing_portal.Session` resource is stable since 2022; no version-specific behavior differences.

### Stripe Customer Portal API Details

```python
# stripe.billing_portal.Session.create() — canonical call for this story:
portal_session = await asyncio.to_thread(
    stripe.billing_portal.Session.create,
    customer="cus_...",                     # from subscriptions.stripe_customer_id
    return_url="https://frontend/bg/settings/billing",  # where Stripe sends user after portal
    # No 'configuration' param needed — uses the default Stripe Dashboard portal config
)
portal_session.url   # "https://billing.stripe.com/p/session/..."
portal_session.id    # "bps_..." or "bpses_..."
```

**Manual prerequisite (one-time, before first use in staging/production):**
The Stripe Customer Portal must be configured in the Stripe Dashboard:
- Billing → Customer Portal → Settings
- Enable: Plan changes (link to product/price IDs), Subscription cancellation, Payment method updates
- This is documented as Assumption #3 in `test-design-epic-08.md` and is NOT automated by this story.

**Stripe Portal vs. Checkout:**
- `stripe.billing_portal.Session` — for existing customers to manage their subscription (plan changes, cancel, update payment)
- `stripe.checkout.Session` — for new subscription creation or guided upgrade (Story 8.6)
- These are different Stripe resources. Do not confuse their namespaces.

### Existing Code to Reuse (Do NOT Reinvent)

| Component | Location | Use |
|-----------|----------|-----|
| `create_checkout_session()` | `billing_service.py` | Direct template for `create_portal_session()` — same error patterns, same `asyncio.to_thread()` wrapping, same audit trail call |
| `POST /checkout/session` endpoint | `billing.py:202-238` | Direct template for `POST /portal/session` — copy the `require_auth`, RBAC check, `_billing_session()`, and `JSONResponse(422)` patterns verbatim |
| `_billing_session()` | `billing.py:67-83` | Already defined — reuse for new endpoint |
| `require_auth` alias | `billing.py:60` | Already defined — call at runtime in endpoint body |
| `ROLE_HIERARCHY` inline rank check | `billing.py:219-225` | Copy exactly for `POST /portal/session` |
| `ForbiddenError` | `eusolicit_common.exceptions` | Already imported at `billing.py:36` |
| `write_audit_entry()` | `src/client_api/services/audit_service.py` | Same pattern as all billing stories |
| `get_settings()` | `src/client_api/config.py` | `settings.frontend_url` → `return_url` construction |
| `Subscription` ORM | `src/client_api/models/subscription.py` | `stripe_customer_id`, `id` fields |
| `asyncio` (module) | Already imported in `billing_service.py` | No new import |
| `apiClient` | `packages/ui/src/lib/api-client.ts` | Frontend HTTP with JWT singleton refresh-lock |
| `useToast` / `toast` | `packages/ui/src/lib/hooks/useToast.ts` or `@eusolicit/ui` | Error notification on portal failure |
| `manageBilling` i18n key | `messages/en.json:920` | Already exists — "Manage Subscription" — DO NOT add a duplicate key |

### Subscription ORM Model (relevant fields after Stories 8.1–8.6)

```
client.subscriptions:
  id                      UUID PK
  company_id              UUID NOT NULL FK→client.companies.id
  stripe_customer_id      String(255) nullable  ← KEY: needed for portal session creation
  stripe_subscription_id  String(255) nullable
  tier                    String(50) NOT NULL default "free"
  status                  String(50) nullable
  is_trial                Boolean NOT NULL default false
  trial_end               DateTime(tz) nullable
  current_period_end      DateTime(tz) nullable
```

`GET /api/v1/billing/subscription` already returns `stripe_customer_id` in the response — the frontend can check `subscription?.stripe_customer_id` to conditionally disable the portal button.

### Frontend Implementation Notes

**Portal opens in a new tab** (epic spec says "opens the portal in a new tab"):
```typescript
window.open(data.portal_url, '_blank');
// NOT window.location.href = data.portal_url  ← that replaces the current tab
```

**Auto-refresh on return (AC5) — no code needed:**
TanStack Query defaults `refetchOnWindowFocus: true`. When the user switches back to the billing settings tab from the portal, the `['billing', 'subscription']` query automatically refetches `GET /api/v1/billing/subscription`. Verify the existing `useQuery` in `billing/page.tsx` does NOT have `refetchOnWindowFocus: false` set (confirmed: the current implementation uses the default).

**Portal button disable logic:**
```typescript
disabled={portalMutation.isPending || !subscription?.stripe_customer_id}
```
Free-tier users or users whose Stripe provisioning failed (null `stripe_customer_id`) should see the button greyed out — there is no portal to open without a Stripe customer.

**i18n keys — what already exists vs. what is new:**
| Key | Status | Value (EN) |
|-----|--------|------------|
| `manageBilling` | ✅ EXISTS (Story 8.6) | "Manage Subscription" |
| `portalOpening` | 🆕 NEW (this story) | "Opening billing portal…" |
| `portalError` | 🆕 NEW (this story) | "Could not open billing portal. Please try again." |

**Button placement:** Add a "Manage your subscription" section below the tier selection cards, above the billing history (which is Story 8.13). A simple `div` with a border-top divider works — no need for a full card component for 2 points.

### Test Patterns — Inherit from Story 8.6

```python
# Pattern for mocking stripe.billing_portal.Session.create (service layer):
from unittest.mock import AsyncMock, MagicMock, patch

mock_portal = MagicMock()
mock_portal.id = "bps_test_001"
mock_portal.url = "https://billing.stripe.com/p/session/bps_test_001"

with patch("asyncio.to_thread", return_value=mock_portal) as mock_thread:
    result = await create_portal_session(company_id, session)
    assert result == {"portal_url": "https://billing.stripe.com/p/session/bps_test_001"}
    # Verify the correct Stripe function was passed
    mock_thread.assert_called_once_with(
        stripe.billing_portal.Session.create,
        customer="cus_test",
        return_url=ANY,
    )
```

```python
# Pattern for seeding subscription row (reuse from test_checkout_session.py / test_webhook_service.py):
def _make_subscription_row(
    company_id: UUID,
    tier: str = "professional",
    status: str = "active",
    is_trial: bool = False,
    stripe_customer_id: str | None = "cus_test",
) -> Subscription:
    sub = Subscription()
    sub.id = uuid4()
    sub.company_id = company_id
    sub.tier = tier
    sub.status = status
    sub.is_trial = is_trial
    sub.stripe_customer_id = stripe_customer_id
    return sub
```

```python
# Pattern for admin-auth mock in endpoint unit tests (from test_checkout_session.py):
mock_current_user = MagicMock()
mock_current_user.company_id = company_id
mock_current_user.role = "admin"  # CRITICAL: admin-only endpoint

with patch("client_api.api.v1.billing.require_auth", new=AsyncMock(return_value=mock_current_user)):
    with patch("client_api.api.v1.billing.get_db_session", return_value=mock_session):
        response = test_client.post("/api/v1/billing/portal/session")
```

**Key learnings from Story 8.6 completion notes** (prevent repeating mistakes):
1. `require_auth` and `get_db_session` are called at runtime inside the endpoint body — patch at the module level (`"client_api.api.v1.billing.require_auth"`), not globally.
2. `_billing_session()` handles both real async generators (production) and direct mock values (tests) via `inspect.isasyncgen()`.
3. `ForbiddenError` from `eusolicit_common.exceptions` maps to HTTP 403 — not 422.
4. Unit tests must set `mock_current_user.role = "admin"` on all admin-only endpoint tests, plus add a specific `role="contributor"` test for the 403 path.
5. 422 response body must use `JSONResponse(status_code=422, content=svc_result)` — not `raise HTTPException(...)` — to avoid the FastAPI `detail` wrapper.

### Test Coverage from Epic Test Design

| Test ID | Level | This Story's Implementation |
|---------|-------|-----------------------------|
| **8.7-API-001** (P1) | API | Service unit tests (7) + endpoint unit tests (4) + 1 integration test |

**P1 note:** Must pass on PR-to-main. The integration test is the primary P1 target for `8.7-API-001`. Unit tests validate all edge cases.

### Project Structure Notes

**New files:**
- `services/client-api/tests/unit/test_portal_session.py` — 11 unit tests
- `services/client-api/tests/integration/test_portal_session_flow.py` — 1 integration test

**Modified files:**
- `services/client-api/src/client_api/services/billing_service.py` — add `create_portal_session()`
- `services/client-api/src/client_api/api/v1/billing.py` — add `POST /portal/session` endpoint + import alias
- `frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx` — add portal mutation + button
- `frontend/apps/client/messages/en.json` — add `portalOpening`, `portalError` keys under `billing` namespace
- `frontend/apps/client/messages/bg.json` — add `portalOpening`, `portalError` keys under `billing` namespace (BG)
- `frontend/e2e/specs/billing-checkout.spec.ts` — add `test.skip` portal scenario

**No changes to:**
- `webhook_service.py` — no webhook handling needed for portal sessions
- `main.py` — billing router already registered
- Any Alembic migration files — no schema changes
- `packages/eusolicit-models` — no new shared DTOs needed (response is `{"portal_url": "..."}`)

**Regression risk:** `billing.py` and `billing_service.py` modified. Run full billing test suite after changes:
```bash
cd eusolicit-app
make test-service SVC=client-api
# or targeted:
pytest services/client-api/tests/unit/test_billing_service.py \
       services/client-api/tests/unit/test_billing_webhook_endpoint.py \
       services/client-api/tests/unit/test_checkout_session.py \
       services/client-api/tests/unit/test_portal_session.py -v
```

### References

- Epic story spec: [Source: eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md#S08.07]
- Epic test design (P1 test 8.7-API-001): [Source: eusolicit-docs/test-artifacts/test-design-epic-08.md#P1]
- Story 8.6 (billing.py full content, `_billing_session`, `ROLE_HIERARCHY` check, `JSONResponse(422)` pattern, unit test patterns): [Source: eusolicit-docs/implementation-artifacts/8-6-stripe-checkout-upgrade-downgrade-flow.md]
- Story 8.1 (`provision_stripe_customer`, `asyncio.to_thread` pattern, `write_audit_entry` usage): [Source: eusolicit-docs/implementation-artifacts/8-1-stripe-customer-provisioning-on-company-registration.md]
- billing.py (current live state): [Source: eusolicit-app/services/client-api/src/client_api/api/v1/billing.py]
- billing_service.py (current live state): [Source: eusolicit-app/services/client-api/src/client_api/services/billing_service.py]
- ClientApiSettings (Stripe settings, frontend_url): [Source: eusolicit-app/services/client-api/src/client_api/config.py]
- Frontend billing page (existing): [Source: eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx]
- i18n billing namespace (existing 20 keys): [Source: eusolicit-app/frontend/apps/client/messages/en.json#billing]
- Project context — asyncio.to_thread (rule 9), audit trail (rules 44–45), RBAC (rules 41–42), i18n (rule 29), QueryGuard/apiClient (rules 21, 26): [Source: eusolicit-docs/project-context.md]
- Stripe Billing Portal Sessions API: `stripe.billing_portal.Session.create(customer, return_url)`

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

- ATDD test `test_create_portal_session_returns_portal_url` failed with `is` identity comparison for bound classmethod objects. Fixed by changing `is` to `==` — Python classmethods create a new bound method object on each attribute access, so `stripe.billing_portal.Session.create is stripe.billing_portal.Session.create` is always False, while `==` correctly compares `__func__` and `__self__` attributes.
- Ruff linter auto-split the two `billing_service` imports into separate `from` blocks; left as-is since further consolidation triggered another ruff reformat cycle.

### Completion Notes List

- ✅ `create_portal_session()` added to `billing_service.py` after `create_checkout_session()`. Follows identical error-guard pattern (AC3 stripe.api_key check first, then AC2 stripe_customer_id check). Uses `asyncio.to_thread()` per project rule 9. Writes audit entry with `action_type="billing.portal_session_created"` (AC7).
- ✅ `POST /api/v1/billing/portal/session` added to `billing.py` using exact same patterns as `POST /checkout/session`: runtime `require_auth`, inline `ROLE_HIERARCHY` rank-check (AC6 admin-only), `_billing_session()` context manager, `JSONResponse(422)` for error body without `detail` wrapper (AC2/AC3).
- ✅ i18n keys `portalOpening` and `portalError` added to both `en.json` and `bg.json` under `billing` namespace (AC8).
- ✅ Frontend billing page updated: `useToast` imported from `@eusolicit/ui`, `portalMutation` added before `checkoutMutation`, "Manage Subscription" button added below tier grid with `data-testid="manage-subscription-btn"`, disabled when `!subscription?.stripe_customer_id` or mutation is pending (AC4).
- ✅ AC5 confirmed — `useQuery` for subscription has no `refetchOnWindowFocus: false`; TanStack Query default (true) handles tab-focus refresh automatically.
- ✅ 11 unit tests pass (7 service layer + 4 endpoint layer), covering AC1/2/3/6/7.
- ✅ 1 integration test scaffold activated (skip removed) for P1 test 8.7-API-001.
- ✅ E2E `test.skip` scenarios already present in `e2e/specs/billing-checkout.spec.ts` (4 portal-related tests added by ATDD phase).
- ✅ 546 total unit tests pass; 0 regressions.

### File List

- `services/client-api/src/client_api/services/billing_service.py` — added `create_portal_session()` function
- `services/client-api/src/client_api/api/v1/billing.py` — added `create_portal_session_svc` import + `POST /portal/session` endpoint
- `frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx` — added `useToast` import, `portalMutation`, and "Manage Subscription" button
- `frontend/apps/client/messages/en.json` — added `portalOpening`, `portalError` keys in `billing` namespace
- `frontend/apps/client/messages/bg.json` — added `portalOpening`, `portalError` keys in `billing` namespace (BG)
- `services/client-api/tests/unit/test_portal_session.py` — activated (removed 11 `@pytest.mark.skip` decorators); fixed `is` → `==` for bound method comparison
- `services/client-api/tests/integration/test_portal_session_flow.py` — activated (removed 1 `@pytest.mark.skip` decorator)
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — updated 8-7 status: ready-for-dev → in-progress → review

## Change Log

- 2026-04-19: Story 8.7 implemented — Stripe Customer Portal integration complete.
  - Added `create_portal_session()` to billing_service.py (AC1, AC2, AC3, AC7)
  - Added `POST /api/v1/billing/portal/session` endpoint to billing.py (AC1, AC2, AC3, AC6)
  - Added `portalOpening`/`portalError` i18n keys to en.json + bg.json (AC8)
  - Added `portalMutation` + "Manage Subscription" button to billing settings page (AC4, AC5, AC8)
  - Activated 11 unit tests + 1 integration test (ATDD skip decorators removed)
  - Fixed ATDD test assertion: `is` → `==` for Python classmethod bound method comparison
  - 546 unit tests pass; 0 regressions

## Senior Developer Review

**Reviewer:** Claude Sonnet 4.6 (bmad-code-review autopilot)
**Date:** 2026-04-19
**Verdict:** ✅ **Approve**

### Scope verified

- `services/client-api/src/client_api/services/billing_service.py` — new `create_portal_session()` (lines 490–581)
- `services/client-api/src/client_api/api/v1/billing.py` — new `POST /billing/portal/session` (lines 248–278) + import alias
- `frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx` — `portalMutation` + `manage-subscription-btn`
- `frontend/apps/client/messages/{en,bg}.json` — `portalOpening`, `portalError` keys added under `billing` namespace
- `services/client-api/tests/unit/test_portal_session.py` — 11 tests, all pass locally (1.44s)
- `services/client-api/tests/integration/test_portal_session_flow.py` — P1 8.7-API-001 scaffold activated

### Acceptance criteria coverage

| AC | Covered | Evidence |
|----|---------|----------|
| AC1 Portal session endpoint | ✅ | `test_create_portal_session_returns_portal_url`, `test_post_portal_session_returns_portal_url`, integration `test_portal_session_returns_url_for_authenticated_admin` |
| AC2 Missing customer guard | ✅ | `test_create_portal_session_missing_customer_returns_error`, `test_create_portal_session_no_subscription_row_returns_error`, `test_post_portal_session_returns_422_with_top_level_error_body` |
| AC3 Stripe not configured guard | ✅ | `test_create_portal_session_stripe_not_configured_returns_error` |
| AC4 Frontend portal button | ✅ | `page.tsx:156-166`, `data-testid="manage-subscription-btn"`, `window.open(url, "_blank")`, disabled when no `stripe_customer_id` |
| AC5 Refetch on window focus | ✅ | `useQuery` on line 30 does not override `refetchOnWindowFocus`; TanStack default (true) applies |
| AC6 Admin only | ✅ | `billing.py:264-267` inline `ROLE_HIERARCHY` rank check → `ForbiddenError` (403); `test_post_portal_session_returns_403_for_non_admin` |
| AC7 Audit trail | ✅ | `test_create_portal_session_writes_audit_entry` asserts `action_type="billing.portal_session_created"`, `entity_type="subscription"`, `after={"session_id": ...}` |
| AC8 i18n | ✅ | `portalOpening`/`portalError` present in both `en.json` and `bg.json` (line 924–925); no hardcoded English strings in page.tsx; `manageBilling` reused from 8.6 |

### Strengths

- Pattern reuse from Story 8.6 is exact: `require_auth` runtime call, `_billing_session()` context manager, `ROLE_HIERARCHY` inline check, `JSONResponse(422, content=...)` to preserve top-level `error`/`message` contract — no drift.
- `asyncio.to_thread()` correctly wraps the synchronous Stripe SDK (project rule 9).
- Error-guard ordering is safe: AC3 (no API key) short-circuits *before* any DB query, so a misconfigured environment never hits Postgres.
- Audit entry is written via the caller's session per the audit_service contract, with `company_id` set correctly for multi-tenant tracing.
- Frontend disables the button when `!subscription?.stripe_customer_id` — graceful for free-tier accounts or Story 8.1 provisioning failures.
- Integration test asserts the negative space (no `webhook_events` row created) — small but correct: portal sessions are synchronous API calls, not webhook-driven.

### Findings (non-blocking)

1. **[Low — Known issue, pre-existing] Hardcoded `/bg/` locale in `return_url`.**
   `billing_service.py:544` — `return_url = f"{settings.frontend_url}/bg/settings/billing"`. An English-locale user returning from Stripe's portal will land on the Bulgarian version of the billing page. This mirrors the pre-existing hardcoded `/bg/` in `create_checkout_session` (lines 435 & 438) and the story Dev Notes explicitly prescribe this literal. Not introduced by this story; recommend a follow-up story to thread the active locale through the session-creation call (e.g., accept `locale` as a function argument, or resolve it from a request context / Accept-Language header in the endpoint).

2. **[Low — Cosmetic] Split `from client_api.services.billing_service import ...` blocks.**
   `billing.py:46-51` — ruff auto-split `create_checkout_session_svc` and `create_portal_session_svc` into two consecutive `from` blocks with identical paths. The Debug Log notes further consolidation triggers another ruff reformat cycle; acceptable, but worth a `noqa` comment or a ruff rule carve-out if it recurs for the third time in a future billing story.

3. **[Low — Test assertion is tautological] Integration webhook_events filter.**
   `test_portal_session_flow.py:202-204` — the query `WebhookEvent.stripe_event_id.like(f"%{company_id.hex[:8]}%")` will never match anything because `stripe_event_id` is populated from Stripe's own `evt_...` IDs, which bear no relationship to the local `company_id`. The test passes trivially. The intent (prove portal creation doesn't touch the webhook dedup table) is correct — a simpler formulation would be `select(WebhookEvent)` with `.count()` pre/post and assert the count is unchanged, or just drop the assertion since it's covered by the AC1 contract.

4. **[Info] Empty POST body.**
   `page.tsx:50` posts `{}` against the endpoint. The endpoint ignores the body, which is fine. No Pydantic body model is declared (unlike `create_checkout_session_endpoint` which uses `CheckoutSessionRequest`), so no 422 validation risk.

### Architecture alignment

- ✅ No Celery worker introduced (rule: BackgroundTasks only).
- ✅ SQLAlchemy async only.
- ✅ structlog used throughout.
- ✅ Audit emitted via `write_audit_entry`.
- ✅ Endpoint under `/api/v1/` via the already-registered `billing` router.
- ✅ Pydantic v2 DTOs — not applicable (no request body), no inline model drift.
- ✅ Two-tier RBAC — company role (admin via `ROLE_HIERARCHY`) enforced at the endpoint; no entity-level check needed (portal session is company-scoped by `company_id` lookup).

### Regression risk

Low. Only additive changes to `billing.py` and `billing_service.py`; existing endpoints untouched. Dev Agent reports 546 total unit tests pass with 0 regressions, which I did not re-verify end-to-end but which is consistent with the additive nature of the diff.

### Verdict rationale

All 8 acceptance criteria are met with tests that assert the specified behaviour, the implementation reuses the 8.6 pattern verbatim, and the 3 findings above are either pre-existing (hardcoded locale), cosmetic (import split), or non-functional (tautological webhook assertion). No blocking issues. **Approve.**

## Status

review
