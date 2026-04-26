---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
lastStep: step-04c-aggregate
lastSaved: '2026-04-19'
workflowType: bmad-testarch-atdd
mode: story
storyKey: 9-8-google-calendar-oauth2-sync
detectedStack: fullstack
generationMode: AI (backend)
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/9-8-google-calendar-oauth2-sync.md
  - eusolicit-docs/test-artifacts/test-design-epic-09.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/notification/tests/conftest.py
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-app/services/notification/pyproject.toml
  - eusolicit-app/pyproject.toml
  - eusolicit-app/services/client-api/tests/api/test_calendar_ical_api.py
  - eusolicit-app/services/client-api/tests/integration/test_033_migration.py
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 9.8 — Google Calendar OAuth2 & Sync

**Date:** 2026-04-19
**Story:** `9-8-google-calendar-oauth2-sync`
**Epic:** E09 — Notifications, Alerts & Calendar
**Type:** Backend (split: Client API + Notification Service)
**TDD Phase:** 🔴 RED — All tests are failing (feature not yet implemented)

---

## Summary

| Category | Count | Status |
|---|---|---|
| Unit tests (client-api) | 4 | 🔴 Skip (Red Phase) |
| Unit tests (notification) | 4 + 8 = 12 | 🔴 Skip (Red Phase) |
| API tests (client-api) | 11 | 🔴 Skip (Red Phase) |
| Migration integration tests | 9 | 🔴 Skip (Red Phase) |
| Sync integration tests | 3 | 🔴 Skip (Red Phase) |
| **Total** | **39** | **🔴 All Red** |

---

## TDD Red Phase Status

✅ Failing tests generated across all test levels
✅ All tests decorated with `@pytest.mark.skip(reason=SKIP_REASON)`
✅ All tests assert EXPECTED behavior (not placeholders)
✅ Test files follow project conventions (async_mode=auto, SKIP_REASON variable, lazy imports)
✅ E09-R-001 and E09-R-007 mitigation tests included (P0 gate)
✅ Cross-tenant isolation test included (project-context rule #38)

---

## Generated Test Files

### Client API Service

| File | Test IDs | Priority | AC Coverage |
|---|---|---|---|
| `services/client-api/tests/unit/test_token_crypto.py` | S09.8-U-01 to U-04 | P0/P1 | AC7 |
| `services/client-api/tests/api/test_calendar_google_oauth_api.py` | S09.8-A-01 to A-11 | P1/P2 | AC1, AC2, AC3 |
| `services/client-api/tests/integration/test_034_migration.py` | S09.8-M-01 to M-09 | P1 | AC6 |

### Notification Service

| File | Test IDs | Priority | AC Coverage |
|---|---|---|---|
| `services/notification/tests/unit/test_token_crypto.py` | S09.8-UN-01 to UN-04 | P0/P1 | AC7 |
| `services/notification/tests/unit/test_calendar_sync_google_unit.py` | S09.8-WU-01 to WU-08 | P1/P2 | AC4, AC5, AC8 |
| `services/notification/tests/integration/test_calendar_sync_google.py` | S09.8-I-01 to I-03 | P1/P2 | AC4, AC8 |

---

## Acceptance Criteria Coverage

| AC | Description | Test IDs | Priority |
|---|---|---|---|
| AC1 | OAuth2 connect endpoint (302, scope, access_type, state) | S09.8-A-01, A-02 | P1 |
| AC2 | OAuth2 callback (token exchange, Fernet, UPSERT, audit, redirect) | S09.8-A-03, A-04, A-05, A-06 | P0/P1 |
| AC3 | Disconnect (revoke, delete, 204, 404, error tolerance) | S09.8-A-07, A-08, A-09, A-10, A-11 | P2 |
| AC4 | Periodic sync (diff: creates, updates, deletes) | S09.8-WU-01 to WU-04, I-01, I-02 | P1 |
| AC5 | Token refresh (transparent, invalid_grant handling) | S09.8-WU-06, WU-07 | P1 |
| AC6 | calendar_events migration 034 (schema, constraints, grants) | S09.8-M-01 to M-09 | P1 |
| AC7 | Fernet encryption (roundtrip, ciphertext != plaintext, error on missing key) | S09.8-U-01 to U-04, UN-01 to UN-04 | P0 |
| AC8 | Idempotency (partial failure safety, iCalUID determinism) | S09.8-WU-05, I-01, I-03 | P1 |

---

## Risk Mitigation Tests (E09-R-001 and E09-R-007)

| Risk | Test ID | Description | Gate Level |
|---|---|---|---|
| E09-R-001 | S09.8-U-02 | `encrypt_token(t) != t` — ciphertext differs from plaintext | **P0 exit gate** |
| E09-R-001 | S09.8-A-03 | Stored `encrypted_access_token` in DB != plaintext bytes | **P0 exit gate** |
| E09-R-001 | S09.8-A-06 | No plaintext token appears in any structlog record during callback | **P0 exit gate** |
| E09-R-007 | S09.8-WU-05 | Partial batch failure: successful events persisted, failures captured in sync_log | **P0 exit gate** |

All four risk tests must be **GREEN** before the story is considered done (see Epic 09 exit criteria).

---

## Acceptance Criteria Not Yet Covered by Tests

| Item | Reason | Next Action |
|---|---|---|
| AC9: Rate-limit retry with `self.retry(countdown=...)` | Complex Celery mock; lower priority (P2) | Add after P1 tests pass; covered by Epic E09-R-008 pattern |
| AC10: Observability (structured log events) | Structlog output assertions can be brittle; deferred | Add `structlog_capture` fixture assertions in automate phase |
| AC11: Docker Compose / .env.example | Infrastructure config; not testable via pytest | Manual verification + CI env var check |
| P3: E2E "connect Google → opportunity appears in calendar" | Manual / `@skip-ci` per AC12; not a sprint exit blocker | Tag as `@pytest.mark.slow @pytest.mark.skip` when added |

---

## Acceptance Criteria Mapping (from Story AC12)

| Priority | Epic Test Design Description | Story Task | Test ID(s) |
|---|---|---|---|
| **P0** | Fernet token encryption roundtrip (2 tests) | Task 2.3 | S09.8-U-01, U-02 |
| **P1** | Google periodic sync diff create/update/delete (5 tests) | Tasks 7.1–7.5, 9.1 | S09.8-WU-01 to WU-05, I-01 |
| **P1** | Token refresh transparent flow (2 tests) | Tasks 7.6, 7.7 | S09.8-WU-06, WU-07 |
| **P2** | `sync_log` row written with correct counts (2 tests) | Tasks 7.8, 9.1 | S09.8-WU-08, I-02 |
| **P2** | Google disconnect revokes + deletes row (2 tests) | Tasks 8.5, 8.6 | S09.8-A-07, A-08 |
| **P3** | E2E "connect Google → opportunity in calendar" | Deferred (manual) | Not generated (skip-CI) |

---

## Test Infrastructure Requirements

### Fixtures Required (already exist in conftest.py)

- `client_api_session_factory` — async DB session factory (client-api)
- `migration_session_factory` — migration-role DB session
- `rsa_test_key_pair` + `rsa_env_setup` — RS256 JWT for auth
- `test_redis_client` — Redis test client (DB index 1)
- `notification_engine` / `notification_session_factory` — notification DB

### Fixtures Required (new — added in test files or conftest)

- `calendar_authed_client` — seeded user + authenticated HTTPX client (in test file)
- `postgres_container` — testcontainers PostgreSQL for integration tests (in integration test file)
- `integration_engine` / `integration_session_factory` — connected to testcontainer

### Environment Variables Required

| Variable | Purpose | Test Setup |
|---|---|---|
| `CALENDAR_ENCRYPTION_KEY` | Fernet key for token crypto | Auto-generated in unit tests via `monkeypatch` |
| `CLIENT_API_DATABASE_URL` | Client API DB connection | Already in conftest default |
| `TEST_DATABASE_URL` | Migration-role DB URL | Already in conftest default |
| `TEST_DATABASE_NOTIFICATION_URL` | Notification-role DB URL | Already in conftest default |

### External Mocking Strategy

| External System | Mock Method | Used In |
|---|---|---|
| Google OAuth2 token exchange | `patch("client_api.core.oauth.oauth.google_calendar.authorize_access_token")` | API tests |
| Google token revocation endpoint | `patch("httpx.AsyncClient.post")` | API tests |
| Google token refresh endpoint | `patch("notification.core.google_oauth.refresh_access_token")` | Unit + integration |
| Google Calendar API (build) | `patch("notification.core.google_calendar_client.build_calendar_service")` | Unit + integration |

**Never use real Google credentials in CI.** All OAuth and Calendar API calls are mocked.

---

## TDD Green Phase Instructions

Once the feature is implemented, follow this sequence to enable tests:

### Step 1 — Verify migration 034 applied
```bash
cd eusolicit-app
make migrate-service SVC=client-api
```

### Step 2 — Remove skip markers (per task completion order)

| When Task Completes | Remove Skips From |
|---|---|
| Task 2 (token_crypto.py) | `test_token_crypto.py` (both services) |
| Task 3 (migration 034) | `test_034_migration.py` |
| Task 5.2 (OAuth endpoints) | `test_calendar_google_oauth_api.py` |
| Task 6 (calendar_sync.py) | `test_calendar_sync_google_unit.py`, `test_calendar_sync_google.py` |

### Step 3 — Run tests in order
```bash
# P0 — Fernet encryption gate
cd eusolicit-app/services/client-api
pytest tests/unit/test_token_crypto.py -v -m unit

# P0 — Callback never logs refresh token
pytest tests/api/test_calendar_google_oauth_api.py::test_google_callback_never_logs_refresh_token -v

# P1 — Sync diff logic
cd ../../services/notification
pytest tests/unit/test_calendar_sync_google_unit.py -v -m unit

# P1 — OAuth endpoints
cd ../client-api
pytest tests/api/test_calendar_google_oauth_api.py -v -m api

# P1 — Migration 034
pytest tests/integration/test_034_migration.py -v -m integration

# Integration — full round-trip
cd ../notification
pytest tests/integration/test_calendar_sync_google.py -v -m integration
```

### Step 4 — Verify coverage gates
```bash
cd eusolicit-app
make coverage  # must stay ≥80% on:
               # token_crypto.py (both services)
               # calendar_sync.py
               # calendar.py (new endpoints)
```

---

## Exit Criteria for Story 9.8

- [ ] `test_ciphertext_differs_from_plaintext` PASSES — stored token bytes differ from plaintext (E09-R-001)
- [ ] `test_google_callback_never_logs_refresh_token` PASSES — no plaintext token in logs (E09-R-001)
- [ ] `test_google_callback_encrypts_and_stores_tokens` PASSES — DB row exists + decrypts correctly
- [ ] `test_calendar_sync_partial_failure_preserves_successes` PASSES — (E09-R-007)
- [ ] `test_calendar_sync_refresh_token_on_expiry` PASSES — transparent token refresh
- [ ] `test_google_sync_integration_create_update_delete_cycle` PASSES — full round-trip
- [ ] Coverage ≥80% on `token_crypto.py`, `calendar_sync.py`, new endpoints in `calendar.py`
- [ ] `make lint` and `make type-check` pass for both services

---

## Notes for Developer

1. **Import lazy pattern**: All test imports from the production modules are inside the test
   function body (e.g., `from client_api.core.token_crypto import ...`). This avoids
   `ImportError` collection failures when the module doesn't exist yet.

2. **`asyncio_mode = "auto"`**: No `@pytest.mark.asyncio` needed on `async def test_*` functions.

3. **Token crypto tests**: `importlib.reload()` is used in S09.8-U-03 and U-04 to force
   the module to pick up the updated `monkeypatch`'d settings. Remove the reload if the
   implementation reads settings lazily on each call.

4. **Notification worker tests**: The sync task uses **synchronous** psycopg2 sessions (Celery
   worker context). The unit tests mock the session entirely to avoid the sync/async boundary.
   The integration tests use `sync_calendars.apply()` (Celery eager mode) which runs synchronously.

5. **S09.8-WU-05 is the E09-R-007 gate test**: The anti-pattern it prevents is rolling back
   ALL local `calendar_events` state on any API failure — which causes duplicate Google events
   on the next 15-minute cycle. The correct behavior: persist confirmed successes, capture
   failures in `sync_log.error_message`.

6. **cross-tenant test (S09.8-A-11)**: Returns 404 (not 403) per project-context rule #38.
   User A can't tell whether User B has a connection — 404 is the correct status.

---

**Generated by:** BMad TEA Agent — ATDD Workflow (bmad-testarch-atdd)
**Execution Mode:** Sequential (backend project, AI generation)
**Version:** BMad v6
