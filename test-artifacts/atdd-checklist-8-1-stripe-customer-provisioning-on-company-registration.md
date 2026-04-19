---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-18'
workflowType: bmad-testarch-atdd
mode: create
storyId: 8-1-stripe-customer-provisioning-on-company-registration
detectedStack: fullstack
generationMode: AI Generation (backend-only story — no E2E tests needed)
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-1-stripe-customer-provisioning-on-company-registration.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/unit/test_audit_service.py
  - eusolicit-app/services/client-api/tests/integration/test_generate_audit_log.py
  - eusolicit-app/services/client-api/tests/api/test_register.py
  - eusolicit-app/services/client-api/src/client_api/models/subscription.py
  - eusolicit-app/services/client-api/src/client_api/config.py
  - eusolicit-app/services/client-api/src/client_api/dependencies.py
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 8.1 — Stripe Customer Provisioning on Company Registration

**Date:** 2026-04-18
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated; feature not yet implemented
**Story Status:** ready-for-dev

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `fullstack`
- **Detection evidence:** `pyproject.toml` (Python/FastAPI backend) + `package.json` (Next.js frontend) both present
- **Story scope:** Pure backend — no UI components in Story 8.1; no E2E browser tests needed
- **Test framework:** pytest 8.x + pytest-asyncio + httpx AsyncClient

### Prerequisites Check

| Requirement | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 6 ACs with full task breakdown and dev notes |
| pytest conftest with async session fixture | ✅ | `client_api_session_factory` + `client_session` in tests/conftest.py |
| `stripe>=8.0` in pyproject.toml | ✅ | Per story dev notes: "stripe>=8.0 is already in services/client-api/pyproject.toml" |
| `Subscription` ORM model exists | ✅ | `src/client_api/models/subscription.py` — `client.subscriptions` (no `stripe_customer_id` yet) |
| `get_session_factory()` dependency exists | ✅ | `src/client_api/dependencies.py:51` — returns async_sessionmaker |
| `write_audit_entry()` exists | ✅ | `src/client_api/services/audit_service.py` (implemented in Story 2.11) |
| `_TestSessionFactory` pattern available | ✅ | Established in `tests/integration/test_generate_audit_log.py` |
| Epic test design loaded | ✅ | `eusolicit-docs/test-artifacts/test-design-epic-08.md` |
| `billing_service.py` exists | ❌ | **Does not exist** — blocked on Story 8.1 Task 5 |
| Migration 026 (`stripe_customer_id` column) applied | ❌ | **Not created** — blocked on Story 8.1 Task 1 |
| `register` endpoint uses BackgroundTasks | ❌ | **Not wired** — blocked on Story 8.1 Task 6 |
| `ClientApiSettings.stripe_secret_key` field exists | ❌ | **Not added** — blocked on Story 8.1 Task 3 |

### TEA Config Flags

| Flag | Value | Effect |
|---|---|---|
| `test_stack_type` | not set → `auto` → `fullstack` | Story 8.1 is pure backend; E2E skipped |
| `tea_use_playwright_utils` | not configured | Disabled; pytest-native patterns used |
| `tea_use_pactjs_utils` | not configured | Disabled; no contract tests for this story |
| `tea_pact_mcp` | not configured | None |
| `tea_browser_automation` | not configured | None |

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (sequential)

**Rationale:** Story 8.1 is entirely backend infrastructure — Stripe SDK integration,
`BackgroundTasks` wiring, DB migration, and configuration. No UI components or browser
interactions involved. Acceptance criteria are unambiguous with explicit task breakdown.
Sequential mode used (no subagent parallelism required for a pure backend story).

---

## Step 3: Test Strategy

### AC → Test Level Mapping

| Acceptance Criterion | Test Level | Priority | Test IDs |
|---|---|---|---|
| **AC1** — Stripe customer created async via BackgroundTask after POST /register | Unit + Integration | P0 | 8-1-UNIT-001, 8-1-INT-001, 8-1-INT-003 |
| **AC2** — `stripe_customer_id` stored in `client.subscriptions` row | Unit + Integration | P0 | 8-1-UNIT-002, 8-1-UNIT-007, 8-1-INT-001 |
| **AC3** — Idempotency: existing ID skips Stripe; idempotency key used | Unit | P1 | 8-1-UNIT-003, 8-1-UNIT-004, 8-1-UNIT-005 |
| **AC4** — StripeError → ERROR log; swallowed; NULL in subscriptions | Unit + Integration | P0 | 8-1-UNIT-006, 8-1-UNIT-007 (partial), 8-1-INT-002 |
| **AC5** — tax_id synced when non-null; omitted when null | Unit | P2 | 8-1-UNIT-008, 8-1-UNIT-009 |
| **AC6** — stripe_secret_key from settings; missing key → WARNING + skip | Unit | P1 | 8-1-UNIT-010, 8-1-UNIT-011 |
| **Dev Notes** — `write_audit_entry` called after provisioning | Unit | P1 | 8-1-UNIT-012 |

### Epic Test Design Alignment

From `test-design-epic-08.md`:

| Epic Test ID | Description | Covered By | Priority |
|---|---|---|---|
| Smoke: Register company | `POST /register` triggers customer creation | 8-1-INT-001 | Smoke/P0 |
| `test_provision_stripe_customer_creates_customer_and_stores_id` | Happy path: Stripe called, ID stored | 8-1-UNIT-001, 8-1-UNIT-002 | P0 |
| `test_provision_stripe_customer_idempotent_existing_id` | Existing ID → skip Stripe | 8-1-UNIT-003 | P1 |
| `test_provision_stripe_customer_uses_idempotency_key` | idempotency_key format check | 8-1-UNIT-004 | P1 |
| `test_provision_stripe_customer_stripe_error_logs_and_swallows` | Stripe error → ERROR log | 8-1-UNIT-006, 8-1-UNIT-007 | P0 |
| `test_provision_stripe_customer_no_api_key_returns_none` | No key → skip, no crash | 8-1-UNIT-010 | P1 |
| `test_register_triggers_stripe_customer_creation` | Full integration flow | 8-1-INT-001 | P0 |
| `test_register_stripe_failure_does_not_fail_registration` | Stripe fails → 201 returned | 8-1-INT-002 | P0 |

### Risk Mitigation Coverage

| Risk ID | Description | Covered By |
|---|---|---|
| R-004 (trial manipulation) | Idempotency prevents duplicate Subscription rows | 8-1-UNIT-003, 8-1-UNIT-005 |

### Test Level Rationale

- **Unit** for AC1–AC6: `provision_stripe_customer` is a pure async service function — all
  Stripe SDK calls and DB interactions are mockable. Exception handling, logging, and
  idempotency are cleanest to verify at unit level without DB or Stripe dependencies.
- **Integration** for AC1/AC2/AC4: The BackgroundTasks wiring (`register` endpoint →
  `provision_stripe_customer_bg`) requires the full ASGI stack to verify. DB state
  assertions confirm the complete flow including migration 026.
- **No E2E tests**: Story 8.1 is pure backend infrastructure with no frontend changes.

### Red Phase Design

All tests use `@pytest.mark.skip(reason=_SKIP_REASON)`. This is intentional:
- Imports deferred inside test methods (`from client_api.services.billing_service import ...`
  inside each test) — consistent with project convention (avoids collection failure when
  the module doesn't exist yet)
- Integration tests skipped to prevent false failures from missing migration/endpoint wiring
- Removing skip decorators is the signal that Story 8.1 implementation is complete

---

## Step 4: Generated Test Files

### File 1: `tests/unit/test_billing_service.py`

**Purpose:** Unit tests for `provision_stripe_customer()` function (AC1–AC6 + Dev Notes audit)

**Test classes and methods:**

| Test ID | Class | Method | AC | Priority |
|---|---|---|---|---|
| 8-1-UNIT-001 | `TestProvisionStripeCustomerHappyPath` | `test_creates_customer_with_correct_params` | AC1 | P0 |
| 8-1-UNIT-002 | `TestProvisionStripeCustomerHappyPath` | `test_stores_stripe_customer_id_in_new_subscription_row` | AC2 | P0 |
| 8-1-UNIT-003 | `TestProvisionStripeCustomerIdempotency` | `test_idempotent_existing_customer_id_skips_stripe_call` | AC3 | P1 |
| 8-1-UNIT-004 | `TestProvisionStripeCustomerIdempotency` | `test_uses_idempotency_key_format` | AC3 | P1 |
| 8-1-UNIT-005 | `TestProvisionStripeCustomerIdempotency` | `test_updates_existing_null_stripe_customer_id_row` | AC2 (upsert) | P1 |
| 8-1-UNIT-006 | `TestProvisionStripeCustomerErrorHandling` | `test_stripe_error_does_not_propagate` | AC4 | P0 |
| 8-1-UNIT-007 | `TestProvisionStripeCustomerErrorHandling` | `test_stripe_error_logged_at_error_level` | AC4 | P0 |
| 8-1-UNIT-008 | `TestProvisionStripeCustomerTaxId` | `test_includes_tax_id_data_when_vat_provided` | AC5 | P2 |
| 8-1-UNIT-009 | `TestProvisionStripeCustomerTaxId` | `test_omits_tax_id_data_when_vat_null` | AC5 | P2 |
| 8-1-UNIT-010 | `TestProvisionStripeCustomerApiKey` | `test_no_api_key_returns_none_without_stripe_call` | AC6 | P1 |
| 8-1-UNIT-011 | `TestProvisionStripeCustomerApiKey` | `test_no_api_key_logs_warning` | AC6 | P1 |
| 8-1-UNIT-012 | `TestProvisionStripeCustomerAudit` | `test_writes_audit_entry_after_successful_provisioning` | Dev Notes | P1 |

**Mocking strategy:**
- `stripe.Customer.create` patched via `unittest.mock.patch` (sync call inside executor)
- `stripe.api_key` patched to control key presence/absence
- `AsyncSession` fully mocked via `AsyncMock` (no DB required)
- `billing_module.logger` patched via `patch.object` for structlog event assertions
- `billing_service.write_audit_entry` patched via `AsyncMock` for audit call assertions

**Fixtures required:** None (all `AsyncMock`/`MagicMock` — no DB or HTTP infrastructure)

---

### File 2: `tests/integration/test_stripe_customer_provisioning.py`

**Purpose:** Integration tests for register → BackgroundTask → DB state (AC1, AC2, AC4)

**Test classes and methods:**

| Test ID | Class | Method | AC | Priority |
|---|---|---|---|---|
| 8-1-INT-001 | `TestStripeCustomerProvisioningIntegration` | `test_register_triggers_stripe_customer_creation_and_stores_id` | AC1, AC2 | P0 |
| 8-1-INT-002 | `TestStripeCustomerProvisioningIntegration` | `test_register_stripe_failure_does_not_fail_registration` | AC4 | P0 |
| 8-1-INT-003 | `TestStripeCustomerProvisioningIntegration` | `test_register_endpoint_uses_background_tasks_parameter` | AC1 (structural) | P0 |

**Fixture defined in test file:**

| Fixture | Scope | Yields | Used by |
|---|---|---|---|
| `stripe_provisioning_client_and_session` | function | `(httpx.AsyncClient, AsyncSession)` | All integration tests |

**Dependency overrides applied:**

| Dependency | Override | Purpose |
|---|---|---|
| `get_db_session` | Yields shared test session | Request-scoped DB isolation |
| `get_session_factory` | Returns `_TestSessionFactory(session)` | Background task DB writes visible in test |
| `get_email_service_dep` | `StubEmailService()` | No real email sent |
| `get_redis_client` | `test_redis_client` | Test Redis (DB index 1) |

**Additional patching in test methods:**
- `stripe.Customer.create` — returns mock with `id="cus_integration_test"` (happy path)
  or raises `stripe.error.StripeError` (failure path)
- `stripe.api_key` — set to `"sk_test_dummy"` to bypass api_key check
- `client_api.services.billing_service.get_session_factory` — backup patch to ensure
  background task uses the test session factory (covers direct import in billing_service.py)

---

## Step 5: Acceptance Criteria Coverage

### AC1 — Stripe customer created asynchronously via BackgroundTask

- [x] **8-1-UNIT-001** — `stripe.Customer.create` called with `name`, `email`, `metadata`; returns `customer.id`
- [x] **8-1-INT-001** — POST /register → 201; BackgroundTask runs; `stripe_customer_id` stored in DB
- [x] **8-1-INT-003** — `register()` has `background_tasks: BackgroundTasks` parameter (structural check)

### AC2 — `stripe_customer_id` stored on `client.subscriptions` row

- [x] **8-1-UNIT-002** — New `Subscription(company_id=..., stripe_customer_id=customer.id)` INSERTed when no row exists
- [x] **8-1-UNIT-005** — Existing row with `stripe_customer_id=NULL` → UPDATEd (not re-INSERTed)
- [x] **8-1-INT-001** — DB state verified: `subscriptions.stripe_customer_id = "cus_integration_test"` after BackgroundTask

### AC3 — Idempotency: skip Stripe; use idempotency key

- [x] **8-1-UNIT-003** — Existing non-null `stripe_customer_id` → `stripe.Customer.create` NOT called; existing ID returned
- [x] **8-1-UNIT-004** — `stripe.Customer.create` called with `idempotency_key=f"company-registration-{company_id}"`
- [x] **8-1-UNIT-005** — Upsert: existing row with NULL → UPDATE branch (not INSERT)

### AC4 — StripeError: ERROR log; swallow; NULL left in subscriptions

- [x] **8-1-UNIT-006** — `stripe.error.StripeError` raised → `provision_stripe_customer` returns `None`; no exception propagates
- [x] **8-1-UNIT-007** — `stripe.error.StripeError` raised → `logger.error(...)` called with structlog event string
- [x] **8-1-INT-002** — Stripe failure → POST /register returns 201; `subscriptions.stripe_customer_id` IS NULL

### AC5 — tax_id synced when non-null; omitted when null

- [x] **8-1-UNIT-008** — `tax_id="DE123456789"` → `tax_exempt="reverse"` and `tax_id_data=[{"type": "eu_vat", "value": "DE123456789"}]` passed to Stripe
- [x] **8-1-UNIT-009** — `tax_id=None` → `tax_id_data` omitted; `tax_exempt` not set to "reverse"; call still succeeds

### AC6 — Stripe key configuration; missing key → WARNING; skip

- [x] **8-1-UNIT-010** — `stripe.api_key=None` → `stripe.Customer.create` NOT called; returns `None`
- [x] **8-1-UNIT-011** — `stripe.api_key=None` → `logger.warning(...)` called (operator visibility)

### Dev Notes — Audit trail

- [x] **8-1-UNIT-012** — `write_audit_entry` called with `action_type="billing.stripe_customer_created"`, `entity_type="subscription"`

---

## TDD Red Phase Validation

### Compliance Checklist

- [x] **All tests marked with `@pytest.mark.skip`** — will not run until skip removed
- [x] **All tests assert EXPECTED behavior** — no placeholder `assert True`
- [x] **Imports deferred to method body** — `from client_api.services.billing_service import ...  # noqa: PLC0415` inside each test, consistent with project convention
- [x] **Error messages are actionable** — each `assert` message points to the specific Task number and AC reference
- [x] **Tests are isolated** — fixture uses `session.rollback()` in `finally`; unique UUIDs in `make_register_payload()` prevent cross-test collision
- [x] **No hard waits** — all async operations use `await`; no `asyncio.sleep` or time-based waits
- [x] **BackgroundTask pattern documented** — comment in integration test explains ASGI transport behavior (synchronous execution in tests)
- [x] **Session sharing pattern correct** — `_TestSessionFactory` wraps test session; overrides both `get_db_session` and `get_session_factory` to share transaction

### Known Design Decisions

| Decision | Rationale |
|---|---|
| `tax_id` passed as optional kwarg to `provision_stripe_customer()` | Story specifies background task args as `(company_id, company_name, user_email)` — no tax_id. AC5 tests assume function accepts `tax_id: str | None = None`. Developer may alternatively query Company.tax_id from DB inside the function; tests will need updating if signature differs. |
| Integration tests patch both `get_session_factory` dependency override AND module-level | Belt-and-suspenders: ASGI dependency override handles FastAPI-injected uses; module patch handles direct `billing_service.get_session_factory()` import |
| `@pytest.mark.integration` on integration tests | Consistent with existing test suite markers; requires DB + Redis (testcontainers) |
| Structural test (8-1-INT-003) checks `inspect.signature` | Verifies BackgroundTasks wiring without DB dependency; pure contract check |
| `test_register_stripe_failure...` asserts row is NULL *if it exists* | Implementation may or may not create a row before the StripeError; the key invariant is `stripe_customer_id IS NULL` |

---

## Next Steps (TDD Green Phase)

### Recommended Implementation Order

**1. Task 1 — Migration 026 + Subscription ORM update**
→ Remove `@pytest.mark.skip` from `TestStripeCustomerProvisioningIntegration` (after Task 6 also done)
→ Run: `make migrate-service SVC=client-api`

**2. Task 2 — SubscriptionDTO update in eusolicit-models**
→ No tests for this directly; verified implicitly via serialization in downstream stories

**3. Task 3 — `ClientApiSettings.stripe_secret_key` field**
→ No dedicated test file; AC6 unit tests verify behavior when `stripe.api_key` is None/set

**4. Task 4 — Stripe SDK configured at startup in `main.py`**
→ No dedicated test file; verified by integration tests when stripe.api_key is patched

**5. Task 5 — Implement `billing_service.py`**
→ Remove `@pytest.mark.skip` from `TestProvisionStripeCustomerHappyPath`
→ Run: `pytest tests/unit/test_billing_service.py::TestProvisionStripeCustomerHappyPath -v`
→ Expected: 2 tests GREEN ✅

→ Remove skip from `TestProvisionStripeCustomerIdempotency`
→ Run: `pytest tests/unit/test_billing_service.py::TestProvisionStripeCustomerIdempotency -v`
→ Expected: 3 tests GREEN ✅

→ Remove skip from `TestProvisionStripeCustomerErrorHandling`
→ Run: `pytest tests/unit/test_billing_service.py::TestProvisionStripeCustomerErrorHandling -v`
→ Expected: 2 tests GREEN ✅

→ Remove skip from `TestProvisionStripeCustomerTaxId`
→ Run: `pytest tests/unit/test_billing_service.py::TestProvisionStripeCustomerTaxId -v`
→ Expected: 2 tests GREEN ✅

→ Remove skip from `TestProvisionStripeCustomerApiKey`
→ Run: `pytest tests/unit/test_billing_service.py::TestProvisionStripeCustomerApiKey -v`
→ Expected: 2 tests GREEN ✅

→ Remove skip from `TestProvisionStripeCustomerAudit`
→ Run: `pytest tests/unit/test_billing_service.py::TestProvisionStripeCustomerAudit -v`
→ Expected: 1 test GREEN ✅

**6. Task 6 — Wire BackgroundTasks into register endpoint**
→ Remove `@pytest.mark.skip` from `TestStripeCustomerProvisioningIntegration`
→ Run: `pytest tests/integration/test_stripe_customer_provisioning.py -v`
→ Expected: 3 tests GREEN ✅

**Full run:**
```bash
pytest tests/unit/test_billing_service.py tests/integration/test_stripe_customer_provisioning.py -v
```
→ Expected: 15 tests GREEN ✅

### Regression Check

Run the E02 registration suite to verify Story 8.1 changes do not break existing behavior:

```bash
pytest tests/api/test_register.py -v
```

The existing test_register.py tests (11 tests, Story 2.2) must remain GREEN after adding
BackgroundTasks to the register endpoint.

---

## Summary Statistics

| Category | Count |
|---|---|
| **Unit tests** (AC1–AC6 + audit) | 12 |
| **Integration tests** (AC1, AC2, AC4 + structural) | 3 |
| **Total tests generated** | **15** |
| Tests currently passing | 0 (all skipped — TDD RED) |
| Acceptance criteria covered | 6 / 6 |
| Epic test IDs covered (test-design-epic-08.md) | All 8 Story 8.1 entries |
| Risk mitigations covered | R-004 (idempotency / no duplicate trials) |

---

## Generated Files

| File | Type | Tests |
|---|---|---|
| `eusolicit-app/services/client-api/tests/unit/test_billing_service.py` | Unit (pytest) | 12 |
| `eusolicit-app/services/client-api/tests/integration/test_stripe_customer_provisioning.py` | Integration (pytest + httpx) | 3 |
| `eusolicit-docs/test-artifacts/atdd-checklist-8-1-stripe-customer-provisioning-on-company-registration.md` | ATDD Checklist | — |

---

*Generated by TEA Master Test Architect via bmad-testarch-atdd workflow — 2026-04-18*
