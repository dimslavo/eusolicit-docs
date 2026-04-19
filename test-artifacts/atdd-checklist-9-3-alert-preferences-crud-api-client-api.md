---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate']
lastStep: 'step-04c-aggregate'
lastSaved: '2026-04-19'
story_id: '9-3-alert-preferences-crud-api-client-api'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/9-3-alert-preferences-crud-api-client-api.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-09.md'
---

# ATDD Checklist: 9.3 Alert Preferences CRUD API

## Preflight & Context
- Stack detected: `backend` (FastAPI/Pytest)
- Framework: `pytest` with `asyncio` and `httpx.AsyncClient`
- Test design loaded from Epic 09

## Test Strategy
- **Level:** API integration tests
- **Priorities:**
  - P0: Validation rules (CPV, Region, Budget, Deadline), Limit enforcement (5 max)
  - P1: CRUD success paths (POST, GET, PUT, DELETE, PATCH toggle)
  - P2: Scoping and isolation (404 for other user's preferences)

## TDD Red Phase Generation
Generated `test_alert_preferences_atdd.py` containing:

### 1. P0 Validation & Limits
- `test_create_preference_invalid_cpv` (422)
- `test_create_preference_invalid_region` (422)
- `test_create_preference_invalid_budget` (422)
- `test_create_preference_invalid_deadline` (422)
- `test_create_preference_limit_exceeded` (409)

### 2. P1 CRUD Operations
- `test_create_preference_success` (201)
- `test_list_preferences_success` (200)
- `test_update_preference_success` (200)
- `test_delete_preference_success` (204)
- `test_toggle_preference_success` (200)

### 3. P2 Security & Isolation
- `test_unauthenticated_access_rejected` (401)
- `test_access_other_user_preference_returns_404` (404)

All tests are marked with `@pytest.mark.skip(reason="TDD Red Phase: Endpoint not implemented yet")`.
