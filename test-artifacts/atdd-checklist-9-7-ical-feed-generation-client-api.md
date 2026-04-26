---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-19'
workflowType: bmad-testarch-atdd
storyKey: 9-7-ical-feed-generation-client-api
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/9-7-ical-feed-generation-client-api.md
  - eusolicit-docs/test-artifacts/test-design-epic-09.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_alert_preferences.py
  - eusolicit-app/services/client-api/tests/integration/test_002_migration.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 9.7 — iCal Feed Generation (Client API)

**Date:** 2026-04-19
**Author:** TEA Master Test Architect
**Status:** 🔴 TDD RED PHASE — All tests are skipped pending implementation
**Story:** S09.7 — iCal Feed Generation (Client API)
**Epic:** E09 — Notifications, Alerts & Calendar
**Stack:** `backend` (Python 3.12 / FastAPI / pytest-asyncio)

---

## TDD Red Phase Status

✅ Failing tests generated — **22 tests** across 4 files, all decorated with `@pytest.mark.skip`

| Test File | Tests | Priority Coverage |
|-----------|-------|-------------------|
| `tests/unit/test_ical_feed_service.py` | 7 | P1×7 |
| `tests/api/test_calendar_ical_api.py` | 10 | P1×10 |
| `tests/integration/test_033_migration.py` | 8 | P2×8 |
| `tests/integration/test_ical_rfc5545.py` | 1 | P1×1 |
| **Total** | **26** | **P1×18, P2×8** |

All tests are in TDD red phase. They **will fail** when `@pytest.mark.skip` is removed
until the implementation is complete.

---

## Acceptance Criteria Coverage

| AC | Description | Test IDs | Status |
|----|-------------|----------|--------|
| **AC1** | Valid iCal output — RFC 5545 compliant `.ics` | S09.7-U-01, S09.7-U-07, S09.7-I-01 | 🔴 RED |
| **AC2** | VEVENT content: SUMMARY, DTSTART/DATE, DTEND, DESCRIPTION, URL, UID | S09.7-U-02, S09.7-U-03, S09.7-U-06, S09.7-U-07, S09.7-I-01 | 🔴 RED |
| **AC3** | Token-based unauthenticated access (no JWT on GET) | S09.7-A-04, S09.7-A-07 | 🔴 RED |
| **AC4** | Unknown/revoked token → 404 (not 401); malformed → 404 (not 422) | S09.7-A-05, S09.7-A-06 | 🔴 RED |
| **AC5** | Generate/rotate token — upsert with new UUID; old token → 404 | S09.7-A-01, S09.7-A-02, S09.7-A-03 | 🔴 RED |
| **AC6** | Response headers: Content-Type, Content-Disposition, Cache-Control, X-Content-Type-Options | S09.7-A-04, S09.7-A-08 | 🔴 RED |
| **AC7** | Freshness — no caching (recomputed on every request) | *Verified by S09.7-A-08 (no-store header) + unit design* | 🔴 RED |
| **AC8** | Tracked-opportunities scope: only open, non-deleted, with deadline; sorted ASC | S09.7-U-04, S09.7-U-05, S09.7-U-06, S09.7-A-09 | 🔴 RED |
| **AC9** | Audit trail on token generation/rotation | S09.7-A-10 | 🔴 RED |
| **AC10** | RBAC grants: client_api_role CRUD; notification_role SELECT | S09.7-M-06 | 🔴 RED |

---

## Test Inventory

### Unit Tests — `tests/unit/test_ical_feed_service.py`

| Test ID | Test Name | AC Coverage | Priority | Why It Fails (Red Phase) |
|---------|-----------|-------------|----------|--------------------------|
| S09.7-U-01 | `test_render_ical_empty_feed_is_valid_vcalendar` | AC1, AC8 | P1 | `ImportError: ical_feed_service` not created |
| S09.7-U-02 | `test_render_ical_single_event_has_all_required_fields` | AC2 | P1 | `ImportError: ical_feed_service` not created |
| S09.7-U-03 | `test_render_ical_uid_is_deterministic` | AC2 | P1 | `ImportError: ical_feed_service` not created |
| S09.7-U-04 | `test_render_ical_excludes_deleted_opportunities` | AC8 | P1 | `ImportError: ical_feed_service` not created |
| S09.7-U-05 | `test_render_ical_excludes_closed_and_no_deadline_opportunities` | AC8 | P1 | `ImportError: ical_feed_service` not created |
| S09.7-U-06 | `test_render_ical_sorted_by_deadline_ascending` | AC8 | P1 | `ImportError: ical_feed_service` not created |
| S09.7-U-07 | `test_render_ical_parses_with_icalendar_library` | AC1, AC2 | P1 | `ImportError: ical_feed_service` not created |

### API Tests — `tests/api/test_calendar_ical_api.py`

| Test ID | Test Name | AC Coverage | Priority | Why It Fails (Red Phase) |
|---------|-----------|-------------|----------|--------------------------|
| S09.7-A-01 | `test_generate_token_creates_new_ical_connection` | AC5 | P1 | 404 — route not registered |
| S09.7-A-02 | `test_generate_token_rotates_existing_connection` | AC5 | P1 | 404 — route not registered |
| S09.7-A-03 | `test_generate_token_requires_authentication` | AC5 | P1 | 404 — route not registered |
| S09.7-A-04 | `test_ical_feed_valid_token_returns_ics` | AC1, AC3, AC6 | P1 | 404 — route not registered |
| S09.7-A-05 | `test_ical_feed_unknown_token_returns_404` | AC4 | P1 | 404 (wrong reason) — route missing |
| S09.7-A-06 | `test_ical_feed_malformed_token_returns_404_not_422` | AC4 | P1 | 404 (wrong reason) — route missing |
| S09.7-A-07 | `test_ical_feed_no_auth_required` | AC3 | P1 | 404 — route not registered |
| S09.7-A-08 | `test_ical_feed_sets_no_store_cache_control_and_nosniff` | AC6 | P1 | 404 — route not registered |
| S09.7-A-09 | `test_ical_feed_includes_only_user_tracked_opportunities` | AC8 | P1 | 404 — route not registered; migration 033 missing |
| S09.7-A-10 | `test_ical_token_rotation_writes_audit_log` | AC9 | P1 | 404 — route not registered |

### Migration Tests — `tests/integration/test_033_migration.py`

| Test ID | Test Name | AC Coverage | Priority | Why It Fails (Red Phase) |
|---------|-----------|-------------|----------|--------------------------|
| S09.7-M-01 | `test_migration_033_file_exists_with_correct_revision` | Task 1 | P2 | Migration file doesn't exist |
| S09.7-M-02 | `test_upgrade_head_succeeds` | Task 1 | P2 | Migration file doesn't exist |
| S09.7-M-03 | `test_calendar_connections_has_required_columns` | AC3, AC5 | P2 | Table doesn't exist |
| S09.7-M-04 | `test_calendar_provider_enum_has_all_values` | Task 1.1 | P2 | Enum doesn't exist |
| S09.7-M-05 | `test_tracked_opportunities_pk_and_index_exist` | AC8 | P2 | Table doesn't exist |
| S09.7-M-06 | `test_grants_applied_for_client_api_role` | AC10 | P2 | Tables don't exist |
| S09.7-M-07 | `test_downgrade_removes_tables_enum_and_indexes` | Task 1.5 | P2 | Migration doesn't exist |
| S09.7-M-08 | `test_re_upgrade_after_downgrade_succeeds` | Task 1.5 | P2 | Migration doesn't exist |

### Integration / RFC 5545 Tests — `tests/integration/test_ical_rfc5545.py`

| Test ID | Test Name | AC Coverage | Priority | Why It Fails (Red Phase) |
|---------|-----------|-------------|----------|--------------------------|
| S09.7-I-01 | `test_ical_feed_parseable_by_icalendar_library_end_to_end` | AC1, AC2 | P1 | Route + migration + service all missing |

---

## Epic Test Design Coverage Mapping

From `eusolicit-docs/test-artifacts/test-design-epic-09.md`:

| Epic P1 Item | Description | Mapped Tests |
|---|---|---|
| `S09.07` P1 (4 tests) | iCal Feed — VEVENT parsing, token generation & revocation | S09.7-A-04, S09.7-A-01, S09.7-A-02, S09.7-I-01 |
| `S09.07` P2 (2 tests) | VEVENT field correctness + token rotation | S09.7-U-02, S09.7-A-02 |
| `S09.07` P2 (2 tests) | Regenerate token creates new token; old → 404 | S09.7-A-02 |

**E09-R-012 (Low):** iCal token exposed in access logs → structural mitigation documented
in story (log scrubbing). No automated test generated (manual security review item).

---

## Next Steps: TDD Green Phase

After implementation is complete (Tasks 1–10):

1. **Remove `@pytest.mark.skip`** from all 26 tests in the 4 test files
2. **Run tests:** `make test-service SVC=client-api`
3. **Expected result:** All 26 tests pass (green phase)
4. **If tests fail after removing skip:**
   - Unit tests (`test_ical_feed_service.py`): Fix `render_ical_for_user` rendering logic
   - API tests (`test_calendar_ical_api.py`): Fix router registration or endpoint behavior
   - Migration tests (`test_033_migration.py`): Fix migration SQL, enum creation, or grants
   - Integration test (`test_ical_rfc5545.py`): End-to-end fix across all layers
5. **Coverage gate:** `make coverage` — new modules must reach ≥80%

---

## Implementation Guidance for Dev Agent

### Files to Create (Story Tasks → Test Coverage)

| Story Task | New File | Tests That Unblock |
|---|---|---|
| Task 1 (Migration 033) | `alembic/versions/033_calendar_connections_tracked_opportunities.py` | S09.7-M-01 to M-08, S09.7-A-09, S09.7-I-01 |
| Task 2 (ORM `CalendarConnection`) | `src/client_api/models/calendar_connection.py` | S09.7-A-01 to A-10 |
| Task 3 (Pydantic `ICalTokenResponse`) | `src/client_api/schemas/calendar_ical.py` | S09.7-A-01 |
| Task 4 (iCal renderer service) | `src/client_api/services/ical_feed_service.py` | S09.7-U-01 to U-07, S09.7-I-01 |
| Task 5 (Router `calendar_ical.py`) | `src/client_api/api/v1/calendar_ical.py` | S09.7-A-01 to A-10, S09.7-I-01 |
| Task 5 (`main.py` registration) | Edit `src/client_api/main.py` | All A-* tests |
| Task 4.1 (dependency) | Edit `pyproject.toml` — add `icalendar>=5.0` | All U-* and I-01 tests |

### Critical Implementation Checks (Linked to Specific Tests)

- **AC4 / S09.7-A-06:** The `GET /{token}` router MUST parse `token` as UUID with
  `try: uuid.UUID(token) except ValueError: return Response(status_code=404)`.
  FastAPI path `{token: str}` — do NOT use `{token: UUID}` type hint or Pydantic will
  return 422 on malformed input.

- **AC3 / S09.7-A-07:** `GET /{token}` endpoint MUST NOT have `Depends(get_current_user)`.
  Test S09.7-A-07 explicitly sends zero auth headers and expects 200.

- **AC5 / S09.7-A-02:** `POST /generate-token` MUST use
  `INSERT ... ON CONFLICT (user_id, provider) DO UPDATE SET token = EXCLUDED.token, updated_at = now()`
  to produce a single row after multiple calls. Test S09.7-A-02 asserts row count == 1.

- **AC9 / S09.7-A-10:** Audit write must use `asyncio.create_task(...)` with
  `try/except` swallow per project-context rule #45. Test S09.7-A-10 includes a
  `asyncio.sleep(0.05)` wait for the background task to complete.

- **AC2 / S09.7-U-02:** `DTSTART` must use `icalendar.vDate(opportunity.deadline.date())`
  (not `vDatetime`) to produce `VALUE=DATE` type. Test S09.7-U-02 asserts
  `isinstance(dtstart_value, date) and not isinstance(dtstart_value, datetime)`.

---

## Risk Notes

| Risk | Test Coverage | Residual |
|------|--------------|---------|
| E09-R-012 (iCal token in access logs) | Not tested automatically — structural mitigation only | Manual security review |
| Token isolation (user A cannot use user B's token) | S09.7-A-09 covers multi-user scope | None |
| Malformed token causes 422 instead of 404 | S09.7-A-06 explicitly tests non-UUID paths | Handled by router design |

---

## Generated Files

| File | Type | Tests |
|------|------|-------|
| `services/client-api/tests/unit/test_ical_feed_service.py` | Unit | 7 |
| `services/client-api/tests/api/test_calendar_ical_api.py` | API | 10 |
| `services/client-api/tests/integration/test_033_migration.py` | Integration | 8 |
| `services/client-api/tests/integration/test_ical_rfc5545.py` | Integration | 1 |
| `eusolicit-docs/test-artifacts/atdd-checklist-9-7-ical-feed-generation-client-api.md` | Checklist | — |

---

**Generated by**: BMad TEA Agent — ATDD Workflow (`bmad-testarch-atdd`)
**TDD Phase**: RED (all tests skipped pending implementation)
**Next Workflow**: Run `bmad-dev-story` with story file `9-7-ical-feed-generation-client-api.md`
