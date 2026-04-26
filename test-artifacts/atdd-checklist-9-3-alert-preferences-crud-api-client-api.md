---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate']
lastStep: 'step-04c-aggregate'
lastSaved: '2026-04-24'
workflowType: 'testarch-atdd'
storyId: '9.3'
storyKey: '9-3-alert-preferences-crud-api-client-api'
storyFile: '/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/9-3-alert-preferences-crud-api-client-api.md'
atddChecklistPath: '/home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-9-3-alert-preferences-crud-api-client-api.md'
generatedTestFiles:
  - '/home/debian/Projects/eusolicit/eusolicit-app/tests/api/v1/test_alert_preferences_atdd.py'
inputDocuments:
  - '/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/9-3-alert-preferences-crud-api-client-api.md'
  - '/home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design-epic-09.md'
---

# ATDD Checklist - Epic 9, Story 3: Alert Preferences CRUD API (Client API)

**Date:** 2026-04-24
**Author:** BMad TEA Agent
**Primary Test Level:** API

---

## Story Summary

As a **user**, I want **to manage my alert preferences via an API**, so that I can configure my CPV sectors, regions, budget, and alert frequency (immediate/daily/weekly) to receive relevant opportunity notifications.

---

## Acceptance Criteria

1. **CRUD Endpoints exist** — Implement endpoints under `/api/v1/alerts/preferences`: `POST`, `GET`, `PUT /{id}`, `DELETE /{id}`, `PATCH /{id}/toggle`.
2. **CPV Validation** — `cpv_sectors` (text[]) validated against known CPV codes. Invalid codes return 422.
3. **Region Validation** — `regions` (text[]) validated against EU member states. Invalid regions return 422.
4. **Budget Range** — `budget_min` and `budget_max` (decimal) enforced: `min >= 0`, `max > min`.
5. **Deadline Proximity** — `deadline_days_ahead` (int, 1-90).
6. **Limit Enforcement** — Maximum 5 alert preferences per user. A 6th create attempt returns 409.
7. **Toggle Endpoint** — `PATCH /{id}/toggle` flips `is_active` boolean without requiring full payload.
8. **Auth & Scoping** — Endpoints require authentication. Users can only access their own preferences. Accessing another user's preference returns 404.
9. **Database** — Alembic migration for `client.alert_preferences` with composite index on `(user_id, is_active)`.

---

## Story Integration Metadata

- **Story ID:** `9.3`
- **Story Key:** `9-3-alert-preferences-crud-api-client-api`
- **Story File:** `/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/9-3-alert-preferences-crud-api-client-api.md`
- **Checklist Path:** `/home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-9-3-alert-preferences-crud-api-client-api.md`
- **Generated Test Files:** `/home/debian/Projects/eusolicit/eusolicit-app/tests/api/v1/test_alert_preferences_atdd.py`

---

## Red-Phase Test Scaffolds Created

### API Tests (11 tests)

**File:** `/home/debian/Projects/eusolicit/eusolicit-app/tests/api/v1/test_alert_preferences_atdd.py` (56 lines)

- ✅ **Test:** `test_create_alert_preference_success`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** Valid data creates an alert preference successfully (201).
- ✅ **Test:** `test_create_alert_preference_cpv_validation_failure`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** Invalid CPV codes return 422.
- ✅ **Test:** `test_create_alert_preference_region_validation_failure`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** Non-EU regions return 422.
- ✅ **Test:** `test_create_alert_preference_budget_validation_failure`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** Inverted budget ranges return 422.
- ✅ **Test:** `test_create_alert_preference_deadline_boundary_failure`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** Deadline boundary limits (0 and 91) return 422.
- ✅ **Test:** `test_create_alert_preference_max_limit_enforced`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** Creating a 6th preference returns 409.
- ✅ **Test:** `test_list_alert_preferences_success`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** GET lists user's preferences successfully.
- ✅ **Test:** `test_get_alert_preference_ownership_isolation`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** Accessing another user's preference returns 404.
- ✅ **Test:** `test_update_alert_preference_success`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** PUT updates preference successfully.
- ✅ **Test:** `test_delete_alert_preference_success`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** DELETE removes preference successfully.
- ✅ **Test:** `test_toggle_alert_preference_success`
  - **Status:** RED - ATDD Scaffold: TDD Red Phase
  - **Verifies:** PATCH /toggle flips is_active boolean successfully.

---

## Implementation Checklist

### Test: `test_create_alert_preference_success`
**File:** `/home/debian/Projects/eusolicit/eusolicit-app/tests/api/v1/test_alert_preferences_atdd.py`
**Tasks to make this test pass:**
- [ ] Create Alembic migration for `client.alert_preferences` table with composite index `(user_id, is_active)`.
- [ ] Create Pydantic Request/Response schemas in `schemas/alert_preferences.py`.
- [ ] Implement `POST /api/v1/alerts/preferences` endpoint.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_create_alert_preference_success`
- [ ] ✅ Test passes (green phase)

### Test: `test_create_alert_preference_cpv_validation_failure`
**Tasks to make this test pass:**
- [ ] Add CPV code validation to Pydantic schema.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_create_alert_preference_cpv_validation_failure`
- [ ] ✅ Test passes (green phase)

### Test: `test_create_alert_preference_region_validation_failure`
**Tasks to make this test pass:**
- [ ] Add EU Region validation to Pydantic schema.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_create_alert_preference_region_validation_failure`
- [ ] ✅ Test passes (green phase)

### Test: `test_create_alert_preference_budget_validation_failure`
**Tasks to make this test pass:**
- [ ] Add budget range validation (`min >= 0`, `max > min`) to Pydantic schema.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_create_alert_preference_budget_validation_failure`
- [ ] ✅ Test passes (green phase)

### Test: `test_create_alert_preference_deadline_boundary_failure`
**Tasks to make this test pass:**
- [ ] Add `deadline_days_ahead` validation (1-90) to Pydantic schema.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_create_alert_preference_deadline_boundary_failure`
- [ ] ✅ Test passes (green phase)

### Test: `test_create_alert_preference_max_limit_enforced`
**Tasks to make this test pass:**
- [ ] Check existing preferences count for the user before creation. Return 409 if 5 or more.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_create_alert_preference_max_limit_enforced`
- [ ] ✅ Test passes (green phase)

### Test: `test_list_alert_preferences_success`
**Tasks to make this test pass:**
- [ ] Implement `GET /api/v1/alerts/preferences` to list user's preferences.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_list_alert_preferences_success`
- [ ] ✅ Test passes (green phase)

### Test: `test_get_alert_preference_ownership_isolation`
**Tasks to make this test pass:**
- [ ] Ensure all individual preference endpoints (GET, PUT, DELETE, PATCH) return 404 if the preference doesn't belong to the authenticated user.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_get_alert_preference_ownership_isolation`
- [ ] ✅ Test passes (green phase)

### Test: `test_update_alert_preference_success`
**Tasks to make this test pass:**
- [ ] Implement `PUT /api/v1/alerts/preferences/{id}` to completely update a preference.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_update_alert_preference_success`
- [ ] ✅ Test passes (green phase)

### Test: `test_delete_alert_preference_success`
**Tasks to make this test pass:**
- [ ] Implement `DELETE /api/v1/alerts/preferences/{id}` to delete a preference.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_delete_alert_preference_success`
- [ ] ✅ Test passes (green phase)

### Test: `test_toggle_alert_preference_success`
**Tasks to make this test pass:**
- [ ] Implement `PATCH /api/v1/alerts/preferences/{id}/toggle` to switch `is_active` status.
- [ ] Run test: `pytest tests/api/v1/test_alert_preferences_atdd.py::test_toggle_alert_preference_success`
- [ ] ✅ Test passes (green phase)

---

## Running Tests

```bash
# Run all tests for this story
pytest tests/api/v1/test_alert_preferences_atdd.py

# Run specific test
pytest tests/api/v1/test_alert_preferences_atdd.py::test_create_alert_preference_success
```
