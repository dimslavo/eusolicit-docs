---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04a-api-failing-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-19'
workflowType: bmad-testarch-atdd
storyKey: 8-11-enterprise-custom-invoicing
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-11-enterprise-custom-invoicing.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/services/admin-api/tests/conftest.py
  - eusolicit-app/services/admin-api/tests/api/test_tenants.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/unit/test_webhook_service.py
  - eusolicit-app/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 8.11 — Enterprise Custom Invoicing

**Date:** 2026-04-19
**Author:** BMad TEA Agent (bmad-testarch-atdd)
**TDD Phase:** 🔴 RED — All tests written before implementation
**Story:** `eusolicit-docs/implementation-artifacts/8-11-enterprise-custom-invoicing.md`
**Epic Test Design:** `eusolicit-docs/test-artifacts/test-design-epic-08.md` (8.11-API-001, R-008)

---

## TDD Red Phase Status

✅ Failing acceptance tests generated — all decorated with `@pytest.mark.skip`

| File | Tests | Phase |
|------|-------|-------|
| `services/admin-api/tests/unit/test_enterprise_invoice_service.py` | 12 | 🔴 RED |
| `services/admin-api/tests/api/test_enterprise_invoice_api.py` | 9 | 🔴 RED |
| `services/client-api/tests/unit/test_enterprise_invoice_webhook.py` | 4 | 🔴 RED |
| **Total** | **25** | **All skipped** |

---

## Preflight: Stack Detection & Configuration

- **Detected stack:** `fullstack` (Next.js 14 frontend + Python/FastAPI backend)
- **Generation mode:** AI generation (backend tests only — story has no UI changes)
- **Test framework:** `pytest` + `pytest-asyncio` (Python 3.12)
- **Mocking pattern:** `unittest.mock` (`AsyncMock`, `MagicMock`, `patch`)
- **Auth pattern:** `generate_platform_admin_token()` from `eusolicit_test_utils.auth`
- **HTTP client:** `httpx.AsyncClient` + `ASGITransport`
- **Session mocking:** `app.dependency_overrides[get_client_db_session]` (admin-api pattern)

---

## Acceptance Criteria Coverage Map

| AC | Description | Test ID(s) | Priority | Test File |
|----|-------------|-----------|----------|-----------|
| AC1 | `client.enterprise_invoices` Alembic migration 031 | (integration — manual migration verify) | P3 | — |
| AC2 | `POST /api/v1/admin/enterprise-invoicing/` endpoint | `test_create_invoice_*` (6 tests) | P2 | api/test_enterprise_invoice_api.py |
| AC3 | Stripe four-step flow via asyncio.to_thread | `test_8_11_api_001_*`, `test_create_invoice_net60_*`, `test_create_invoice_adds_stripe_*`, `test_create_invoice_finalizes_*` | P2 | unit/test_enterprise_invoice_service.py |
| AC4 | `GET /api/v1/admin/enterprise-invoicing/` list endpoint | `test_list_*` (7 tests) | P2 | unit/test_enterprise_invoice_service.py + api/test_enterprise_invoice_api.py |
| AC5 | `invoice.paid` webhook updates enterprise invoice status | `test_invoice_paid_*` (4 tests) | P2 | unit/test_enterprise_invoice_webhook.py |
| AC6 | Audit trail via structlog (`billing.enterprise_invoice_created`) | (covered implicitly by service test: structlog.info called) | P3 | — |
| AC7 | Tier verification before Stripe API calls | `test_create_invoice_rejects_non_enterprise_tier`, `_rejects_missing_stripe_customer_id`, `_rejects_unknown_company` | P2 | unit/test_enterprise_invoice_service.py |

### Coverage Gaps (out of scope / deferrable)

| Item | Reason | Mitigation |
|------|--------|-----------|
| AC1 Migration schema | Requires live DB; verified by `make migrate-service SVC=client-api` | Task 1.2 manual verify |
| AC6 Structlog audit | structlog is tested at INFO level; caplog integration deferred | Add caplog assertion in green phase |
| AC3 Stripe metadata | metadata fields in Invoice.create not explicitly asserted | Extend test in green phase |

---

## Test Strategy

### Level Selection (Backend-focused)

| Level | Rationale | Tests |
|-------|-----------|-------|
| **Unit** | Service function isolation; mock DB + Stripe; fast feedback | 12 (service) + 4 (webhook) |
| **API** | Router integration; mock service; test HTTP contract (codes, error shapes) | 9 |
| **Integration** | DB migration schema; skipped (manual `make migrate-service`) | 0 (deferred) |

### Priority Assignment

| Priority | Tests | Criteria |
|----------|-------|---------|
| P2 | All 25 tests | Secondary feature, low risk (R-008 score=2); edge cases |

Note: All Story 8.11 tests are P2 per epic test design (R-008: score 2, Monitor).

---

## Generated Test Files

### File 1: `services/admin-api/tests/unit/test_enterprise_invoice_service.py`

**Purpose:** Unit tests for `enterprise_invoice_service.create_enterprise_invoice` and `list_enterprise_invoices` service functions. Mocks DB session and `asyncio.to_thread` (Stripe SDK calls).

| Test | AC | Epic Test ID | Assertion Focus |
|------|----|-------------|----------------|
| `test_8_11_api_001_create_invoice_net30_sets_days_until_due_30` | AC3 | **8.11-API-001** | `days_until_due=30`, `collection_method='send_invoice'`, `auto_advance=False` |
| `test_create_invoice_net60_sets_days_until_due_60` | AC3 | — | `days_until_due=60` for NET_60 |
| `test_create_invoice_adds_stripe_invoice_item_per_line_item` | AC3 | — | `InvoiceItem.create` called N times with correct `price_data` |
| `test_create_invoice_finalizes_and_sends_invoice_after_items` | AC3 | — | `finalize_invoice` index < `send_invoice` index in call sequence |
| `test_create_invoice_persists_db_record_with_status_open` | AC2 | — | `session.add(record)` with `status='open'`, `sent_at` set |
| `test_create_invoice_rejects_non_enterprise_tier` | AC7 | — | `ValueError('not_enterprise_tier:...')`, Stripe calls NOT made |
| `test_create_invoice_rejects_missing_stripe_customer_id` | AC7 | — | `ValueError('no_stripe_customer:...')`, Stripe calls NOT made |
| `test_create_invoice_rejects_unknown_company` | AC7 | — | `ValueError('company_not_found:...')`, Stripe calls NOT made |
| `test_list_returns_all_invoices_without_filters` | AC4 | — | `total==3`, `len(items)==3`, limit/offset echoed |
| `test_list_filters_by_company_id` | AC4 | — | `session.execute` called twice; `total==1` |
| `test_list_filters_by_status_paid` | AC4 | — | `session.execute` called twice; `total==1` |
| `test_list_applies_limit_and_offset` | AC4 | — | `result.limit==10`, `result.offset==20`, `total==100` |

**Key mock strategy:**
```python
# asyncio.to_thread side effects (in call order):
[{"id": "in_test_001"},  # Invoice.create
 MagicMock(),            # InvoiceItem.create × N
 MagicMock(),            # finalize_invoice
 MagicMock()]            # send_invoice

# DB session for create:
result.one_or_none.return_value = row_mock  # row with tier='enterprise'

# DB session for list (two calls):
session.execute.side_effect = [count_result, data_result]
```

---

### File 2: `services/admin-api/tests/api/test_enterprise_invoice_api.py`

**Purpose:** HTTP-level tests for the enterprise invoicing router. Mocks service layer (`enterprise_invoice_service.*`) and injects mock `get_client_db_session`. Tests HTTP status codes and error response shapes.

| Test | AC | Assertion Focus |
|------|----|----------------|
| `test_create_invoice_requires_platform_admin_auth` | AC2 | 401 (no token), 403 (member token) |
| `test_create_invoice_returns_201_with_invoice_response` | AC2 | 201, `stripe_invoice_id`, `status='open'`, `company_id` |
| `test_create_invoice_returns_422_for_non_enterprise_tier` | AC2 | 422, `detail.error=='not_enterprise_tier'` |
| `test_create_invoice_returns_422_for_missing_stripe_customer` | AC2 | 422, `detail.error=='no_stripe_customer'` |
| `test_create_invoice_returns_404_for_unknown_company` | AC2 | 404 |
| `test_create_invoice_returns_502_on_stripe_error` | AC2 | 502, `detail.error=='stripe_error'` |
| `test_list_invoices_returns_200_with_paginated_results` | AC4 | 200, `items`, `total`, `limit`, `offset` |
| `test_list_invoices_filters_by_company_id_query_param` | AC4 | Service called with `company_id==UUID` |
| `test_list_invoices_filters_by_status_query_param` | AC4 | Service called with `status=='paid'` |

**Key mock strategy:**
```python
# Override DB session dependency
app.dependency_overrides[get_client_db_session] = _make_mock_client_session_override(session)

# Mock service at router import path
with patch("admin_api.api.v1.enterprise_invoicing.enterprise_invoice_service.create_enterprise_invoice",
           new_callable=AsyncMock, return_value=mock_response):
    ...
```

---

### File 3: `services/client-api/tests/unit/test_enterprise_invoice_webhook.py`

**Purpose:** Unit tests for the ADDITIVE `_handle_enterprise_invoice_paid` extension to `webhook_service._handle_invoice_event`. Verifies no-op behavior, paid status update, and non-regression of subscription handler.

| Test | AC | Assertion Focus |
|------|----|----------------|
| `test_invoice_paid_updates_enterprise_invoice_to_paid` | AC5 | `session.execute` called; `scalar_one_or_none` returns updated_id |
| `test_invoice_paid_noop_when_no_enterprise_invoice_match` | AC5 | No exception; `session.execute` called (WHERE returns nothing) |
| `test_invoice_paid_does_not_break_subscription_handler` | AC5 | `_handle_enterprise_invoice_paid` called once from `_handle_invoice_event` |
| `test_enterprise_invoice_handler_not_called_for_other_event_types` | AC5 | `_handle_enterprise_invoice_paid` NOT called for `invoice.payment_failed` |

**Key implementation note for Task 6.1:**
```python
# In _handle_invoice_event(), after existing subscription update:
# ADDITIVE NEW CODE:
if event_type == "invoice.paid":
    stripe_inv_id = event_obj.get("id")
    if stripe_inv_id:
        await _handle_enterprise_invoice_paid(session, stripe_inv_id)

# New function (add below _handle_invoice_event):
async def _handle_enterprise_invoice_paid(session: AsyncSession, stripe_invoice_id: str) -> None:
    from client_api.models.enterprise_invoice import EnterpriseInvoice
    from sqlalchemy import update as sa_update
    result = await session.execute(
        sa_update(EnterpriseInvoice)
        .where(EnterpriseInvoice.stripe_invoice_id == stripe_invoice_id)
        .values(status="paid", paid_at=datetime.now(tz=timezone.utc))
        .returning(EnterpriseInvoice.id)
    )
    updated_id = result.scalar_one_or_none()
    if updated_id:
        logger.info("enterprise_invoice_paid_via_webhook", ...)
```

---

## Next Steps (TDD Green Phase)

After implementing Story 8.11 (Tasks 1–6):

### Phase Transition Checklist

- [ ] **Task 1** — `031_enterprise_invoices.py` migration created + applied (`make migrate-service SVC=client-api`)
- [ ] **Task 2** — `EnterpriseInvoice` ORM model created in `client-api`
- [ ] **Task 3** — Pydantic schemas (`LineItemSchema`, `CreateEnterpriseInvoiceRequest`, `EnterpriseInvoiceResponse`, `EnterpriseInvoiceListResponse`) created in `admin-api`
- [ ] **Task 4** — `enterprise_invoice_service.py` created in `admin-api`
- [ ] **Task 5** — `enterprise_invoicing.py` router created + registered in `admin-api/main.py`
- [ ] **Task 6** — `_handle_enterprise_invoice_paid` added to `webhook_service.py` (additive only)

### Remove skip decorators (green phase):

```bash
# Remove @pytest.mark.skip from all 25 tests:
grep -n "mark.skip" \
  services/admin-api/tests/unit/test_enterprise_invoice_service.py \
  services/admin-api/tests/api/test_enterprise_invoice_api.py \
  services/client-api/tests/unit/test_enterprise_invoice_webhook.py
```

### Run tests (verify green phase):

```bash
# Unit tests (no external deps)
make test-unit

# Or run specific service tests:
cd eusolicit-app
pytest services/admin-api/tests/unit/test_enterprise_invoice_service.py -v
pytest services/admin-api/tests/api/test_enterprise_invoice_api.py -v
pytest services/client-api/tests/unit/test_enterprise_invoice_webhook.py -v
```

### Green Phase Success Criteria

- [ ] All 25 tests PASS (no skips, no errors)
- [ ] `8.11-API-001` (P2) passing: NET_30 → `days_until_due=30` asserted
- [ ] R-008 mitigation passing: `invoice.paid` webhook updates enterprise invoice status
- [ ] No regression in existing `test_webhook_service.py` (run full suite)
- [ ] `make quality-check` passes (ruff + mypy)
- [ ] Test coverage ≥ 80% for new files

---

## Risk Coverage

| Risk ID | Risk | Test Coverage |
|---------|------|---------------|
| R-008 | Enterprise invoice delayed sending or amount mismatch | `test_invoice_paid_updates_enterprise_invoice_to_paid` (webhook sync); `test_list_invoices_returns_200_with_paginated_results` (admin review visibility) |

---

## Implementation Guidance for Dev Agent

### Files to create (new):
1. `services/client-api/alembic/versions/031_enterprise_invoices.py` (migration)
2. `services/client-api/src/client_api/models/enterprise_invoice.py` (ORM model)
3. `services/admin-api/src/admin_api/schemas/enterprise_invoice.py` (Pydantic schemas)
4. `services/admin-api/src/admin_api/services/enterprise_invoice_service.py` (service)
5. `services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py` (router)

### Files to modify (additive):
6. `services/client-api/src/client_api/services/webhook_service.py` (+~15 lines additive)
7. `services/admin-api/src/admin_api/main.py` (+2 lines: import + include_router)

### Critical implementation rules (from project-context.md):
- All Stripe SDK calls: `await asyncio.to_thread(stripe.*, ...)` — NEVER call sync Stripe SDK directly in async functions
- Admin-API DB: use `get_client_db_session()` for `client.enterprise_invoices` writes
- Auth: `get_admin_user()` dependency validates `platform_admin` JWT role
- Migration: every `op.*` call must include `schema="client"` — violation puts DDL in `public` schema
- Webhook extension: purely additive — NEVER modify existing subscription handler logic

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** BMad v6
