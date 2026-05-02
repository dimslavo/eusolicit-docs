---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-27'
workflowType: testarch-atdd
storyId: '15.1'
storyKey: 15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config
storyFile: /home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config.md
atddChecklistPath: /home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/atdd-checklist-15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config.md
generatedTestFiles:
  # Client-API — integration
  - eusolicit-app/services/client-api/tests/integration/test_migration_050_pricing_tiers_seed.py
  - eusolicit-app/services/client-api/tests/integration/test_pricing_tiers_list_endpoint.py
  - eusolicit-app/services/client-api/tests/integration/test_per_bid_checkout_happy_path.py
  - eusolicit-app/services/client-api/tests/integration/test_per_bid_checkout_cross_tenant.py
  - eusolicit-app/services/client-api/tests/integration/test_per_bid_webhook_redis_active_key.py
  - eusolicit-app/services/client-api/tests/integration/test_per_bid_webhook_idempotency.py
  # Client-API — unit
  - eusolicit-app/services/client-api/tests/unit/test_billing_service_per_bid.py
  # Admin-API — integration + unit
  - eusolicit-app/services/admin-api/tests/integration/test_pricing_tiers_admin_only.py
  - eusolicit-app/services/admin-api/tests/unit/test_pricing_tier_service.py
  # Frontend — Vitest component tests
  - eusolicit-app/frontend/apps/client/__tests__/PerBidPricingTierPicker.test.tsx
  - eusolicit-app/frontend/apps/admin/__tests__/PricingTierManagementPage.test.tsx
  # Frontend — Playwright E2E
  - eusolicit-app/frontend/e2e/per-bid-pricing-tier-picker.spec.ts
  - eusolicit-app/frontend/e2e/admin-pricing-tier-management.spec.ts
inputDocuments:
  - eusolicit-docs/implementation-artifacts/15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/services/client-api/tests/integration/test_pro_plus_cross_tenant_isolation.py
  - eusolicit-app/services/client-api/tests/integration/test_billing_checkout_pro_plus.py
  - eusolicit-app/services/client-api/tests/integration/test_migration_049_pro_plus_seed.py
  - eusolicit-app/services/client-api/tests/integration/test_addon_purchase_flow.py
  - eusolicit-app/tests/conftest.py
  - eusolicit-app/services/admin-api/tests/conftest.py
  - eusolicit-app/services/admin-api/tests/api/test_tenants.py
  - _bmad/bmm/config.yaml
---

# ATDD Checklist — Story 15.1: Per-Bid Pricing Tiers + Stripe Checkout Integration + Admin Config

**Date:** 2026-04-27
**Author:** BMad TEA Agent
**Primary Test Level:** Integration (backend) + Component/E2E (frontend)
**TDD Phase:** 🔴 RED — All scaffolds are skipped until implementation

---

## Story Summary

As a **Bid Manager**, I want to purchase a Per-Bid SKU at one of three customer-facing pricing tiers (€99 standard / €199 grant_module / €299 enterprise_stack) for a specific opportunity via Stripe Checkout, with the tier menu admin-configurable, so that I can pay-per-pursuit for premium AI/compliance/grant features without upgrading my entire subscription tier, and so that platform admins can adjust pricing tiers without redeployment.

This is a fullstack extension story: it introduces the `client.add_on_pricing_tiers` database table, admin CRUD endpoints, a client-facing tier-list and checkout endpoint, a webhook extension that writes the Redis active-key contract for S15.02, and a three-tier React picker UI component.

---

## Stack Detection

**Detected stack:** `fullstack` (Python 3.12 FastAPI backend + Next.js 14 TypeScript frontend)

- Backend test markers: `@pytest.mark.skip` (RED phase), `@pytest.mark.integration`, `@pytest.mark.unit`
- Frontend test skip syntax: `it.skip(...)` (Vitest), `test.skip(true, '...')` (Playwright)

---

## Generation Mode

**Mode selected:** AI Generation (sequential)
- Rationale: All backend ACs have clear, specific contracts; no browser recording needed for backend. Frontend ACs specify exact component shape, API contracts, and selector targets.

---

## Acceptance Criteria Coverage

| AC | Description | Test File(s) | Priority |
|----|-------------|--------------|----------|
| AC 1 | `client.add_on_pricing_tiers` table + migration 050 | `test_migration_050_pricing_tiers_seed.py` | P0 |
| AC 2 | `add_on_pricing_tier_id` FK + `workspace_id` on `add_on_purchases` | `test_migration_050_pricing_tiers_seed.py` | P0 |
| AC 3 | `AddOnPricingTier` ORM model | `test_migration_050_pricing_tiers_seed.py` (structural), `test_billing_service_per_bid.py` | P0 |
| AC 4 | Admin API CRUD endpoints | `test_pricing_tiers_admin_only.py`, `test_pricing_tier_service.py` | P0 |
| AC 5 | `GET /api/v1/billing/per-bid/pricing-tiers` + Redis cache | `test_pricing_tiers_list_endpoint.py` | P0 |
| AC 6 | `POST /api/v1/opportunities/{id}/per-bid/checkout` | `test_per_bid_checkout_happy_path.py`, `test_billing_service_per_bid.py` | P0 |
| AC 7 | Webhook extension for `add_on_pricing_tier_id` + `workspace_id` | `test_per_bid_webhook_idempotency.py`, `test_per_bid_webhook_redis_active_key.py` | P0 |
| AC 8 | Redis `add_on_active:{...}:{opportunity_id}` key + 90-day TTL | `test_per_bid_webhook_redis_active_key.py` | P0 |
| AC 9 | Frontend: `PerBidPricingTierPicker.tsx` component | `PerBidPricingTierPicker.test.tsx`, `per-bid-pricing-tier-picker.spec.ts` | P0 |
| AC 10 | Frontend: success return flow | `PerBidPricingTierPicker.test.tsx`, `per-bid-pricing-tier-picker.spec.ts` | P1 |
| AC 11 | Frontend: admin pricing-tier management page | `PricingTierManagementPage.test.tsx`, `admin-pricing-tier-management.spec.ts` | P1 |
| AC 12 | Cross-tenant negative test (mandatory per CLAUDE.md) | `test_per_bid_checkout_cross_tenant.py` | P0 |
| AC 13 | Webhook idempotency regression test | `test_per_bid_webhook_idempotency.py` | P0 |
| AC 14 | Admin CRUD authorization regression | `test_pricing_tiers_admin_only.py` | P0 |
| AC 15 | Migration seed idempotency | `test_migration_050_pricing_tiers_seed.py` | P0 |
| AC 16 | `alembic check` clean post-migration | Dev Agent Record (quoted output — not automated) | P0 |
| AC 17 | Full pytest suite for both services | Dev Agent Record (quoted output — not automated) | P0 |
| AC 18 | i18n BG/EN strings | `PerBidPricingTierPicker.test.tsx` (structural), `pnpm check:i18n` in Dev Agent Record | P1 |
| AC 19 | `Literal[...]` schema discipline | `test_pricing_tier_service.py` (validates name validation), `test_pricing_tiers_admin_only.py` (validates 422 on invalid name) | P1 |

---

## Story Integration Metadata

- **Story ID:** `15.1`
- **Story Key:** `15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config`
- **Story File:** `/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config.md`
- **Checklist Path:** `/home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/atdd-checklist-15-1-per-bid-pricing-tiers-stripe-checkout-integration-admin-config.md`
- **Epic Test Design Reference:** `test-design-epic-08.md` (closest analogous — R-001 webhook race, R-006 cache invalidation, 8.9-API-001 addon checkout pattern)
- **Story 15.0 Cross-Tenant Pattern:** `test_pro_plus_cross_tenant_isolation.py` (parametrised symmetry shape adopted for AC 12)

---

## Red-Phase Test Scaffolds Created

### Backend Integration Tests — client-api (7 files, 40 test functions)

---

#### `test_migration_050_pricing_tiers_seed.py` (7 tests) — AC 1, 2, 15
**File:** `eusolicit-app/services/client-api/tests/integration/test_migration_050_pricing_tiers_seed.py`

- 🔴 **`test_migration_050_applies_without_error`** [P0]
  - **Status:** RED — migration 050 does not exist yet
  - **Verifies:** `alembic upgrade head` returns exit code 0 with revision="050", down_revision="049"

- 🔴 **`test_migration_050_creates_add_on_pricing_tiers_table`** [P0]
  - **Status:** RED — table does not exist
  - **Verifies:** All 10 columns exist (id, name, display_label_en, display_label_bg, price_eur_cents, stripe_price_id, feature_stack, active, created_at, updated_at)

- 🔴 **`test_migration_050_seeds_exactly_three_rows`** [P0]
  - **Status:** RED — table doesn't exist
  - **Verifies:** `SELECT COUNT(*) FROM client.add_on_pricing_tiers` = 3

- 🔴 **`test_migration_050_seed_rows_have_correct_data`** [P0]
  - **Status:** RED — table doesn't exist
  - **Verifies:** standard=9900, grant_module=19900, enterprise_stack=29900, all active=true, correct feature_stack JSON

- 🔴 **`test_migration_050_adds_columns_to_add_on_purchases`** [P0]
  - **Status:** RED — columns not added yet
  - **Verifies:** `add_on_pricing_tier_id` (UUID nullable FK) and `workspace_id` (UUID nullable FK) exist on `client.add_on_purchases`

- 🔴 **`test_migration_050_downgrade_removes_table_and_columns`** [P0]
  - **Status:** RED — table doesn't exist
  - **Verifies:** `alembic downgrade -1` removes `add_on_pricing_tiers` and drops the two columns from `add_on_purchases`

- 🔴 **`test_migration_050_upgrade_downgrade_upgrade_round_trip_is_idempotent`** [P0]
  - **Status:** RED — migration doesn't exist
  - **Verifies:** Full round-trip (upgrade→downgrade→upgrade) leaves exactly 3 rows without `IntegrityError` from `uq_add_on_pricing_tiers_name`

---

#### `test_pricing_tiers_list_endpoint.py` (7 tests) — AC 5
**File:** `eusolicit-app/services/client-api/tests/integration/test_pricing_tiers_list_endpoint.py`

- 🔴 **`test_pricing_tiers_list_returns_only_active_tiers`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** `GET /api/v1/billing/per-bid/pricing-tiers` returns 200, all tiers have `active=true`

- 🔴 **`test_pricing_tiers_list_caches_response_in_redis`** [P0]
  - **Status:** RED — endpoint + cache logic doesn't exist
  - **Verifies:** After first GET, Redis key `pricing_tiers:active` exists

- 🔴 **`test_pricing_tiers_list_uses_cache_on_second_request`** [P1]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Second request hits Redis cache (no second DB query)

- 🔴 **`test_pricing_tiers_list_response_shape`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Response has `{"tiers": [...]}` with fields: id, name, display_label, price_eur_cents, feature_stack, active

- 🔴 **`test_pricing_tiers_list_requires_authentication`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Unauthenticated GET → 401

- 🔴 **`test_pricing_tiers_list_locale_aware_display_label`** [P1]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** `Accept-Language: bg` → display_label from `display_label_bg`; `Accept-Language: en` → from `display_label_en`

- 🔴 **`test_pricing_tiers_list_inactive_tier_not_shown`** [P1]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Seeded inactive tier is absent from response

---

#### `test_per_bid_checkout_happy_path.py` (10 tests) — AC 6
**File:** `eusolicit-app/services/client-api/tests/integration/test_per_bid_checkout_happy_path.py`

- 🔴 **`test_per_bid_checkout_standard_tier_returns_checkout_url`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** `POST /api/v1/opportunities/{id}/per-bid/checkout` with standard tier → 200 `{"checkout_url": str, "session_id": str}`

- 🔴 **`test_per_bid_checkout_grant_module_tier_returns_checkout_url`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Same for grant_module tier

- 🔴 **`test_per_bid_checkout_enterprise_stack_tier_returns_checkout_url`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Same for enterprise_stack tier

- 🔴 **`test_per_bid_checkout_requires_bid_manager_role`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** `contributor` role → 403

- 🔴 **`test_per_bid_checkout_returns_403_for_read_only_role`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** `read_only` role → 403

- 🔴 **`test_per_bid_checkout_returns_422_for_unknown_tier_id`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Non-existent `pricing_tier_id` → 422 `{"error": "tier_not_found_or_inactive"}`

- 🔴 **`test_per_bid_checkout_returns_422_for_inactive_tier`** [P1]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** `active=false` tier → 422 `{"error": "tier_not_found_or_inactive"}`

- 🔴 **`test_per_bid_checkout_returns_422_when_stripe_price_id_null`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Tier with `stripe_price_id=null` → 422 `{"error": "tier_not_configured"}`

- 🔴 **`test_per_bid_checkout_writes_audit_log_entry`** [P1]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** Audit log entry with `action_type="billing.per_bid_checkout_session_created"`

- 🔴 **`test_per_bid_checkout_requires_authentication`** [P0]
  - **Status:** RED — endpoint doesn't exist
  - **Verifies:** No auth → 401

---

#### `test_per_bid_checkout_cross_tenant.py` (7 test cases) — AC 12 **MANDATORY**
**File:** `eusolicit-app/services/client-api/tests/integration/test_per_bid_checkout_cross_tenant.py`

- 🔴 **`test_per_bid_checkout_cross_tenant_returns_403`** [P0] — PARAMETRISED: direction × tier_name (6 cases)
  - Parameters: `direction ∈ {a_to_b, b_to_a}` × `tier_name ∈ {standard, grant_module, enterprise_stack}`
  - **Status:** RED — endpoint + cross-tenant validation not implemented
  - **Verifies:** Company A's bid_manager POSTs checkout on Company B's opportunity → 403 `{"error": "opportunity_not_accessible"}` (per AC 12: NOT 404 — opportunities are platform-broadcast public-tender data)

- 🔴 **`test_per_bid_checkout_cross_workspace_within_tenant_returns_403`** [P0]
  - **Status:** RED — workspace-scoped validation not implemented
  - **Verifies:** Within company A, workspace W1 bid_manager → W2's opportunity → 403

> **Note:** Parametrised symmetry (a_to_b + b_to_a) is non-negotiable per CLAUDE.md + AC 12 + Story 15.0 review-fix B3. Single-direction tests are insufficient.

---

#### `test_per_bid_webhook_redis_active_key.py` (5 tests) — AC 8
**File:** `eusolicit-app/services/client-api/tests/integration/test_per_bid_webhook_redis_active_key.py`

- 🔴 **`test_webhook_sets_redis_active_key_with_correct_format`** [P0]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** `add_on_active:{workspace_id}:{opportunity_id}` exists in Redis after successful webhook

- 🔴 **`test_webhook_redis_key_ttl_is_90_days`** [P0]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** `await redis.ttl(key) >= 60*60*24*90 - 60` (90 days minus 60s margin)

- 🔴 **`test_webhook_uses_company_id_prefix_when_workspace_id_null`** [P1]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** When `workspace_id` absent in metadata → key = `add_on_active:{company_id}:{opportunity_id}`

- 🔴 **`test_webhook_active_key_value_matches_tier_name`** [P0]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** Redis value is one of `"standard"`, `"grant_module"`, `"enterprise_stack"`

- 🔴 **`test_webhook_key_format_exactly_matches_s15_02_contract`** [P0] **CRITICAL**
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** Key format has exactly 3 colon-separated segments: `add_on_active:{workspace_or_company_id}:{opportunity_id}` — this is the S15.02 `_USAGE_LUA KEYS[2]` handoff contract. A typo silently breaks S15.02.

---

#### `test_per_bid_webhook_idempotency.py` (4 tests) — AC 13
**File:** `eusolicit-app/services/client-api/tests/integration/test_per_bid_webhook_idempotency.py`

- 🔴 **`test_duplicate_webhook_creates_only_one_add_on_purchase_row`** [P0]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** Two identical POSTs → `COUNT(*) FROM client.add_on_purchases WHERE stripe_checkout_session_id='cs_test_xxx'` = 1

- 🔴 **`test_duplicate_webhook_sets_redis_key_idempotently`** [P0]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** Two POSTs → Redis key exists, value unchanged (SET idempotency)

- 🔴 **`test_duplicate_webhook_writes_audit_log_exactly_once`** [P1]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** Audit log entry count for `action_type="billing.per_bid_purchased"` == 1 after two deliveries

- 🔴 **`test_webhook_dedup_uses_existing_event_dedup_guard`** [P0]
  - **Status:** RED — webhook extension not implemented
  - **Verifies:** `webhook_events` table has exactly 1 row for `stripe_event_id` after two deliveries (Epic 8 dedup pattern)

---

### Backend Unit Tests — client-api (1 file, 7 tests)

#### `test_billing_service_per_bid.py` (7 tests) — AC 6
**File:** `eusolicit-app/services/client-api/tests/unit/test_billing_service_per_bid.py`

- 🔴 **`test_create_per_bid_checkout_session_calls_stripe_with_correct_params`** [P0]
  - **Verifies:** `stripe.checkout.Session.create` called with `mode="payment"`, `customer`, `line_items=[{"price": tier.stripe_price_id, "quantity": 1}]`, `automatic_tax={"enabled": True}`

- 🔴 **`test_create_per_bid_checkout_session_includes_metadata`** [P0]
  - **Verifies:** Stripe metadata has `type="addon"`, `company_id`, `workspace_id`, `opportunity_id`, `add_on_pricing_tier_id`, `user_id`

- 🔴 **`test_create_per_bid_checkout_session_raises_when_tier_not_found`** [P0]
  - **Verifies:** Unknown tier_id → raises or returns `{"error": "tier_not_found_or_inactive"}`

- 🔴 **`test_create_per_bid_checkout_session_raises_when_tier_inactive`** [P0]
  - **Verifies:** `tier.active=False` → raises or 422

- 🔴 **`test_create_per_bid_checkout_session_raises_when_stripe_price_id_none`** [P0]
  - **Verifies:** `tier.stripe_price_id is None` → raises or `{"error": "tier_not_configured"}`

- 🔴 **`test_create_per_bid_checkout_session_uses_asyncio_to_thread`** [P0]
  - **Verifies:** Stripe call wrapped in `asyncio.to_thread` (NEVER direct sync call in async context)

- 🔴 **`test_create_per_bid_checkout_session_returns_checkout_url_and_session_id`** [P0]
  - **Verifies:** Returns `{"checkout_url": str, "session_id": str}`

---

### Backend Integration Tests — admin-api (1 file, 22 tests)

#### `test_pricing_tiers_admin_only.py` (22 tests) — AC 4, AC 14
**File:** `eusolicit-app/services/admin-api/tests/integration/test_pricing_tiers_admin_only.py`

**Auth Enforcement (AC 14):** [P0]
- 🔴 `test_get_pricing_tiers_requires_admin_bearer_token` — no auth → 401
- 🔴 `test_get_pricing_tiers_rejects_client_api_jwt` — company-role JWT → 401/403
- 🔴 `test_post_pricing_tier_requires_admin_bearer_token` — no auth → 401
- 🔴 `test_patch_pricing_tier_requires_admin_bearer_token` — no auth → 401
- 🔴 `test_delete_pricing_tier_requires_admin_bearer_token` — no auth → 401

**CRUD Happy Path (AC 4):** [P0/P1]
- 🔴 `test_list_pricing_tiers_returns_all_seeded_tiers` [P0] — 200 with standard/grant_module/enterprise_stack
- 🔴 `test_get_pricing_tier_by_id_returns_correct_tier` [P0] — GET single tier by ID
- 🔴 `test_get_pricing_tier_by_id_returns_404_for_unknown_id` [P1] — random UUID → 404
- 🔴 `test_create_pricing_tier_returns_201` [P0] — POST valid → 201 with full body
- 🔴 `test_create_pricing_tier_rejects_invalid_name` [P0] — name="invalid" → 422 (AC 19: Literal[...] enforcement)
- 🔴 `test_create_pricing_tier_rejects_zero_price` [P0] — price_eur_cents=0 → 422
- 🔴 `test_create_pricing_tier_rejects_negative_price` [P0] — price_eur_cents=-100 → 422
- 🔴 `test_patch_pricing_tier_updates_price_and_labels` [P0] — PATCH → 200 with updated values
- 🔴 `test_patch_pricing_tier_name_is_immutable` [P0] — PATCH with `name` field → 422 `{"error": "name_immutable"}`
- 🔴 `test_delete_pricing_tier_soft_deletes_when_not_in_use` [P0] — DELETE → 204; tier still in GET list with `active=false`
- 🔴 `test_delete_pricing_tier_returns_409_when_in_use` [P0] — purchase reference → 409 `{"error": "tier_in_use"}`

**Audit Log:** [P1]
- 🔴 `test_create_pricing_tier_writes_audit_log` — `action_type="admin.pricing_tier.created"`
- 🔴 `test_patch_pricing_tier_writes_audit_log` — `action_type="admin.pricing_tier.updated"`
- 🔴 `test_delete_pricing_tier_writes_audit_log` — `action_type="admin.pricing_tier.deleted"`

**Cache Invalidation:** [P0]
- 🔴 `test_create_pricing_tier_invalidates_client_cache` — POST → Redis `pricing_tiers:active` deleted
- 🔴 `test_patch_pricing_tier_invalidates_client_cache` — PATCH → key deleted
- 🔴 `test_delete_pricing_tier_invalidates_client_cache` — DELETE → key deleted

---

### Backend Unit Tests — admin-api (1 file, 12 tests)

#### `test_pricing_tier_service.py` (12 tests) — AC 4
**File:** `eusolicit-app/services/admin-api/tests/unit/test_pricing_tier_service.py`

- 🔴 `test_soft_delete_succeeds_when_no_purchases_reference_tier` [P0]
- 🔴 `test_soft_delete_blocked_when_purchases_reference_tier` [P0] → raises TierInUseError
- 🔴 `test_create_tier_validates_name_must_be_valid_enum` [P0] — "invalid" → error
- 🔴 `test_create_tier_validates_price_must_be_positive` [P0] — 0 → error
- 🔴 `test_create_tier_validates_price_negative_rejected` [P0] — -1 → error
- 🔴 `test_update_tier_rejects_name_change` [P0] → NameImmutableError
- 🔴 `test_update_tier_allows_price_update` [P1]
- 🔴 `test_update_tier_allows_stripe_price_id_update` [P1]
- 🔴 `test_update_tier_allows_feature_stack_update` [P1]
- 🔴 `test_update_tier_allows_active_flag_toggle` [P1]
- 🔴 `test_list_tiers_includes_inactive_tiers` [P1] — admin sees ALL (no active=true filter)
- 🔴 `test_get_tier_returns_none_for_unknown_id` [P1]

---

### Frontend Component Tests — Vitest (2 files, 20 tests)

#### `PerBidPricingTierPicker.test.tsx` (11 tests) — AC 9, AC 10, AC 18
**File:** `eusolicit-app/frontend/apps/client/__tests__/PerBidPricingTierPicker.test.tsx`

- 🔴 Static assertions verify component file, API wrappers, and i18n key existence before implementation
- 🔴 Behaviour tests (marked `it.skip`) verify rendering, selection, checkout call, loading state, error toast, success flow, URL cleanup, enterprise gate

#### `PricingTierManagementPage.test.tsx` (9 tests) — AC 11, AC 18
**File:** `eusolicit-app/frontend/apps/admin/__tests__/PricingTierManagementPage.test.tsx`

- 🔴 Static assertions for page file, API wrappers, i18n keys
- 🔴 Behaviour tests verify DataTable columns, Add dialog, edit Sheet, CRUD mutations, 422/409 error handling, RBAC redirect

---

### Frontend E2E Tests — Playwright (2 files, 17 tests)

#### `per-bid-pricing-tier-picker.spec.ts` (9 tests) — AC 9, AC 10
**File:** `eusolicit-app/frontend/e2e/per-bid-pricing-tier-picker.spec.ts`

All tests use `test.skip(true, '🔴 RED PHASE: ...')`:
- [P0] Tier cards display on opportunity detail page
- [P0] Stripe redirect on purchase click
- [P1] Success toast + unlocked badge on return from Stripe
- [P1] URL param cleanup after success
- [P1] Error toast on 422 checkout failure
- [P1] Loading/disabled state during checkout
- [P1] Feature stack tooltip visible
- [P2] Enterprise users do not see picker
- [P2] Picker hidden when already purchased (badge shown)

#### `admin-pricing-tier-management.spec.ts` (8 tests) — AC 11
**File:** `eusolicit-app/frontend/e2e/admin-pricing-tier-management.spec.ts`

All tests use `test.skip(true, '🔴 RED PHASE: ...')`:
- [P0] DataTable columns + all 3 seed tiers visible
- [P1] Create tier via dialog → POST + row in table
- [P1] Edit tier via Sheet → PATCH + updated table
- [P1] Name field disabled in edit Sheet (immutable)
- [P1] Soft-delete → tier active=false, row remains
- [P2] 409 → "Tier in use" error toast
- [P2] Non-platform-admin → redirect to `/admin/403`
- [P2] Platform-admin can access page (smoke)

---

## Test Count Summary

| Category | Files | Tests | Priority |
|----------|-------|-------|----------|
| client-api integration | 6 | 40 (incl. 6 parametrised cases) | P0/P1 |
| client-api unit | 1 | 7 | P0 |
| admin-api integration | 1 | 22 | P0/P1 |
| admin-api unit | 1 | 12 | P0/P1 |
| Frontend Vitest (component) | 2 | 20 | P0/P1/P2 |
| Frontend Playwright E2E | 2 | 17 | P0/P1/P2 |
| **Total** | **13** | **118** | |

All 118 tests are in RED phase (skip-decorated). Activated tests will fail until implementation is complete.

---

## Mock Requirements

### Stripe SDK (`stripe.checkout.Session.create`)
```python
from unittest.mock import MagicMock, patch

@patch("client_api.services.billing_service.stripe.checkout.Session.create")
def test_example(mock_stripe_create):
    mock_stripe_create.return_value = MagicMock(
        id="cs_test_15_1_xxx",
        url="https://checkout.stripe.com/pay/cs_test_15_1_xxx",
    )
```

### Stripe Webhook Signature Verification
```python
@patch("stripe.Webhook.construct_event")
def test_webhook(mock_construct):
    mock_construct.return_value = {
        "id": "evt_test_15_1_xxx",
        "type": "checkout.session.completed",
        "data": {
            "object": {
                "id": "cs_test_15_1_xxx",
                "payment_intent": "pi_test_xxx",
                "customer": "cus_test_xxx",
                "amount_total": 9900,
                "currency": "eur",
                "metadata": {
                    "type": "addon",
                    "company_id": str(company_id),
                    "workspace_id": str(workspace_id),
                    "opportunity_id": str(opportunity_id),
                    "add_on_pricing_tier_id": str(tier_id),
                    "user_id": str(user_id),
                },
            }
        },
    }
```

---

## Fixture Infrastructure

### Canonical ORM seeding pattern (from test_billing_checkout_pro_plus.py)
```python
async def _seed_company_bid_manager(session, *, tier="professional", stripe_customer_id="cus_test"):
    from client_api.models.company import Company
    from client_api.models.company_membership import CompanyMembership
    from client_api.models.subscription import Subscription
    from client_api.models.user import User
    from datetime import UTC, datetime

    suffix = uuid.uuid4().hex[:8]
    company = Company(name=f"PerBidTest-{suffix}")
    session.add(company); await session.flush()
    user = User(email=f"bid-mgr-{suffix}@eusolicit-test.example.com",
                full_name="Bid Mgr", hashed_password="irrelevant",
                email_verified=True, is_active=True)
    session.add(user); await session.flush()
    membership = CompanyMembership(user_id=user.id, company_id=company.id,
                                    role="bid_manager", accepted_at=datetime.now(UTC))
    subscription = Subscription(company_id=company.id, tier=tier,
                                  status="active", stripe_customer_id=stripe_customer_id)
    session.add_all([membership, subscription]); await session.flush()
    return company.id, user.id
```

### Root conftest fixtures used
- `db_session` — per-test rollback session
- `clean_redis` — Redis DB1 flush before/after
- `create_company_pair` — seed two companies for cross-tenant tests
- `register_and_verify_with_role` — canonical user registration helper
- `OpportunityFactory` — opportunity seeding
- `rsa_test_key_pair` — for JWT minting in cross-tenant tests

---

## Risk Mitigations Applied

| Risk | Source | Mitigation in Tests |
|------|--------|---------------------|
| Webhook race (R-001) | Epic 8 test-design | `test_per_bid_webhook_idempotency.py` covers dedup + single row + single audit entry |
| Cache invalidation failure (R-006) | Epic 8 test-design | `test_pricing_tiers_admin_only.py` cache invalidation class + `test_pricing_tiers_list_endpoint.py` cache hit class |
| Redis key format mismatch for S15.02 | AC 8 + story dev notes | `test_webhook_key_format_exactly_matches_s15_02_contract` is a P0 structural assertion |
| Cross-tenant purchase (AC 12) | CLAUDE.md mandatory | 6-case parametrised symmetry (direction × tier_name) + workspace-scoped variant |
| Stripe SDK sync/async violation | Story dev notes | `test_create_per_bid_checkout_session_uses_asyncio_to_thread` unit test |
| Name immutability regression | AC 4 / AC 19 | `test_patch_pricing_tier_name_is_immutable` (integration) + `test_update_tier_rejects_name_change` (unit) |
| Admin-api auth regression (AC 14) | CLAUDE.md + AC 14 | `TestAuthEnforcement` class in `test_pricing_tiers_admin_only.py` |
| Migration seed uniqueness (AC 15) | AC 15 + migration 049 pattern | `test_migration_050_upgrade_downgrade_upgrade_round_trip_is_idempotent` |

---

## Implementation Checklist (Activation Order for Dev Agent)

Activate scaffolds task-by-task, verifying RED → GREEN for each batch before moving on:

### Task 1 — Migration 050 + ORM model (AC 1, 2, 3, 15, 16)
- [ ] Activate `test_migration_050_*` tests → confirm they FAIL (table missing)
- [ ] Implement `050_add_on_pricing_tiers.py` migration
- [ ] Implement `add_on_pricing_tier.py` ORM model
- [ ] Update `add_on_purchase.py` ORM + `models/__init__.py`
- [ ] Run tests → all 7 pass (GREEN)
- [ ] Run `alembic check` → quote "No new upgrade operations detected." in Dev Agent Record

### Task 2 — Admin API CRUD (AC 4, 14, 19)
- [ ] Activate `test_pricing_tier_service.py` unit tests → confirm FAIL
- [ ] Implement `pricing_tier_service.py`
- [ ] Run unit tests → GREEN
- [ ] Activate `test_pricing_tiers_admin_only.py` → confirm FAIL
- [ ] Implement `pricing_tiers.py` router + `pricing_tier.py` schemas + mount in `main.py`
- [ ] Run integration tests → all 22 pass (GREEN)

### Task 3 — Client API endpoints (AC 5, 6, 19)
- [ ] Activate `test_pricing_tiers_list_endpoint.py` → confirm FAIL
- [ ] Activate `test_billing_service_per_bid.py` unit tests → confirm FAIL
- [ ] Activate `test_per_bid_checkout_happy_path.py` (happy path tests only first) → confirm FAIL
- [ ] Implement `GET /billing/per-bid/pricing-tiers` in `billing.py`
- [ ] Implement `create_per_bid_checkout_session()` in `billing_service.py`
- [ ] Implement `POST /opportunities/{id}/per-bid/checkout` in `per_bid.py`
- [ ] Run all 3 files → GREEN

### Task 4 — Cross-tenant validation (AC 12)
- [ ] Activate `test_per_bid_checkout_cross_tenant.py` → confirm FAIL
- [ ] Implement `opportunity_service.is_opportunity_accessible()` check in checkout endpoint
- [ ] Run cross-tenant tests → all 7 cases GREEN
- [ ] Run `test_per_bid_checkout_happy_path.py` again → still GREEN

### Task 5 — Webhook extension + Redis active key (AC 7, 8, 13)
- [ ] Activate `test_per_bid_webhook_redis_active_key.py` → confirm FAIL
- [ ] Activate `test_per_bid_webhook_idempotency.py` → confirm FAIL
- [ ] Implement `_handle_addon_checkout_completed` extension in `webhook_service.py`
- [ ] Run both files → all 9 tests GREEN
- [ ] Verify S15.02 key format contract test specifically (`test_webhook_key_format_exactly_matches_s15_02_contract`) passes

### Task 6 — Frontend picker + success flow (AC 9, 10, 18)
- [ ] Remove `it.skip` from `PerBidPricingTierPicker.test.tsx` → confirm FAIL
- [ ] Implement `PerBidPricingTierPicker.tsx`, `lib/api/billing.ts`, `lib/queries/use-billing.ts`
- [ ] Add i18n strings to `messages/{bg,en}.json`
- [ ] Run Vitest → GREEN; run `pnpm check:i18n` → PASS
- [ ] Activate `per-bid-pricing-tier-picker.spec.ts` → confirm FAIL
- [ ] Wire picker into opportunity detail page; implement success return flow
- [ ] Run Playwright E2E → GREEN

### Task 7 — Admin pricing-tier page (AC 11, 18)
- [ ] Activate `PricingTierManagementPage.test.tsx` → confirm FAIL
- [ ] Activate `admin-pricing-tier-management.spec.ts` → confirm FAIL
- [ ] Implement admin page, components, API wrappers
- [ ] Add admin i18n strings; run `pnpm check:i18n`
- [ ] Run Vitest + Playwright → GREEN

### Task 8 — Full validation (AC 16, 17)
- [ ] `make test-service SVC=client-api` → quote final summary line in Dev Agent Record
- [ ] `make test-service SVC=admin-api` → quote final summary line in Dev Agent Record
- [ ] `pnpm test` in both `apps/client` and `apps/admin` → quote both summaries
- [ ] `pnpm lint`, `pnpm type-check` → quote outputs
- [ ] `make lint`, `make type-check` → no new errors on touched files
- [ ] `alembic check` → "No new upgrade operations detected."
- [ ] Set `Status: review` in story file BEFORE updating sprint-status.yaml

---

## Running Tests

```bash
# Backend — client-api integration (needs postgres + redis running)
make test-service SVC=client-api

# Backend — specific test files (integration)
pytest eusolicit-app/services/client-api/tests/integration/test_migration_050_pricing_tiers_seed.py -v -s
pytest eusolicit-app/services/client-api/tests/integration/test_per_bid_checkout_cross_tenant.py -v -s
pytest eusolicit-app/services/client-api/tests/integration/test_per_bid_webhook_redis_active_key.py -v -s

# Backend — unit tests (no external deps)
pytest eusolicit-app/services/client-api/tests/unit/test_billing_service_per_bid.py -v
pytest eusolicit-app/services/admin-api/tests/unit/test_pricing_tier_service.py -v

# Backend — admin-api integration
make test-service SVC=admin-api
pytest eusolicit-app/services/admin-api/tests/integration/test_pricing_tiers_admin_only.py -v -s

# Frontend — Vitest component tests
cd eusolicit-app/frontend && pnpm test --filter client
cd eusolicit-app/frontend && pnpm test --filter admin

# Frontend — i18n check
cd eusolicit-app/frontend && pnpm check:i18n

# Frontend — E2E Playwright
make test-e2e-chromium
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**

- ✅ 13 test scaffold files generated (9 Python + 4 TypeScript)
- ✅ 118 tests all in skip/RED state
- ✅ All tests assert expected behavior (not placeholder assertions)
- ✅ Parametrised cross-tenant symmetry (direction × tier_name = 6 cases)
- ✅ S15.02 Redis key contract test included (P0)
- ✅ Mock requirements documented (Stripe SDK + webhook HMAC)
- ✅ Fixture infrastructure documented (canonical ORM seeding pattern)
- ✅ ATDD checklist saved to test_artifacts/

### GREEN Phase (Dev Agent — next)

1. **Pick one task from the Implementation Checklist** (start with Task 1 — migration)
2. **Remove `@pytest.mark.skip` / `it.skip`** for that task's tests → confirm they FAIL
3. **Implement the feature** for that task
4. **Run tests** → verify GREEN
5. **Move to next task** and repeat
6. **Never skip the validation gate** (Task 8) — quoted pytest output is mandatory per AC 17

---

## Test Design Provenance

This checklist was derived from:
1. **`test-design-epic-08.md`** — R-001 (webhook race), R-006 (cache invalidation), 8.9-API-001/002 add-on checkout pattern
2. **`test_pro_plus_cross_tenant_isolation.py`** (Story 15.0) — canonical parametrised symmetry shape for AC 12
3. **`test_billing_checkout_pro_plus.py`** (Story 15.0) — Stripe mock + canonical ORM seeding for checkout tests
4. **`test_migration_049_pro_plus_seed.py`** (Story 15.0) — alembic CLI round-trip test pattern
5. **`test_addon_purchase_flow.py`** (Story 8.9) — closest sibling for webhook + idempotency patterns
6. **Story 15.1 Dev Notes** — S15.02 Redis key format contract, asyncio.to_thread requirement, cache invalidation via direct DELETE

---

## Known Gaps and Deferred Items

| Item | Reason | Owner |
|------|--------|-------|
| AC 16 — `alembic check` output | Requires live DB; quoted in Dev Agent Record manually | Dev Agent |
| AC 17 — Full pytest suite output | Quoted in Dev Agent Record manually (verbatim summary lines) | Dev Agent |
| AC 18 — `pnpm check:i18n` output | Quoted in Dev Agent Record after i18n strings added | Dev Agent |
| k6 / NFR-OB-5 load gate | Out of scope for this story per Dev Notes (Epic 13 carry-forward `inj-02`) | Future |
| `_USAGE_LUA` metering bypass | Owned by S15.02 — this story only writes the Redis active-key contract | S15.02 |
| UX spec gap for per-bid picker | Non-blocking per implementation readiness report; flag in Dev Agent Record | UX |

---

**Generated by BMad TEA Agent** — 2026-04-27
