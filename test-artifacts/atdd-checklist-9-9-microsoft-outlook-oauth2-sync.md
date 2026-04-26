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
storyKey: 9-9-microsoft-outlook-oauth2-sync
detectedStack: fullstack
generationMode: AI (backend)
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/9-9-microsoft-outlook-oauth2-sync.md
  - eusolicit-docs/test-artifacts/test-design-epic-09.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/notification/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_calendar_google_oauth_api.py
  - eusolicit-app/services/notification/tests/unit/test_calendar_sync_google_unit.py
  - eusolicit-app/services/notification/tests/integration/test_calendar_sync_google.py
  - eusolicit-docs/test-artifacts/atdd-checklist-9-8-google-calendar-oauth2-sync.md
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-app/services/notification/pyproject.toml
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 9.9 — Microsoft Outlook OAuth2 & Sync

**Date:** 2026-04-19
**Story:** `9-9-microsoft-outlook-oauth2-sync`
**Epic:** E09 — Notifications, Alerts & Calendar
**Type:** Backend (split: Client API + Notification Service)
**TDD Phase:** 🔴 RED — All tests are failing (feature not yet implemented)

---

## Summary

| Category | Count | Status |
|---|---|---|
| API tests (client-api) | 11 | 🔴 Skip (Red Phase) |
| Unit tests (client-api) | 2 | 🔴 Skip (Red Phase) |
| Integration tests (client-api) | 2 | 🔴 Skip (Red Phase) |
| Unit tests (notification — sync logic) | 10 | 🔴 Skip (Red Phase) |
| Unit tests (notification — graph client) | 6 | 🔴 Skip (Red Phase) |
| Unit tests (notification — oauth helper) | 3 | 🔴 Skip (Red Phase) |
| Integration tests (notification) | 2 | 🔴 Skip (Red Phase) |
| **Total** | **36** | **🔴 All Red** |

---

## TDD Red Phase Status

✅ Failing tests generated across all test levels
✅ All tests decorated with `@pytest.mark.skip(reason=SKIP_REASON)`
✅ All tests assert EXPECTED behavior (not placeholders)
✅ Test files follow project conventions (async_mode=auto, SKIP_REASON variable, lazy imports)
✅ E09-R-001 mitigation tests included (P0 gate — never log tokens)
✅ E09-R-007 mitigation test included (partial-failure preserves successes)
✅ E09-R-008 mitigation tests included (429 + 503 respect Retry-After)
✅ B1 CSRF binding test included (missing session → error redirect, NOT JWT fallback)
✅ Cross-tenant isolation test included (project-context rule #38)
✅ AC3 documented deferral verified (no outbound HTTP on disconnect — Microsoft has no revoke endpoint)
✅ `transactionId` dedup key assertion included (AC8 parity with 9.8's `iCalUID`)

---

## Generated Test Files

### Client API Service

| File | Test IDs | Priority | AC Coverage |
|---|---|---|---|
| `services/client-api/tests/api/test_calendar_microsoft_oauth_api.py` | S09.9-A-01 to A-11 | P0/P1/P2 | AC1, AC2, AC3 |
| `services/client-api/tests/unit/test_oauth_registration.py` | S09.9-U-01, U-02 | P1 | Task 2.3 |
| `services/client-api/tests/integration/test_provider_enum_microsoft.py` | S09.9-M-01, M-02 | P1 | AC6 |

### Notification Service

| File | Test IDs | Priority | AC Coverage |
|---|---|---|---|
| `services/notification/tests/unit/test_outlook_calendar_sync_unit.py` | S09.9-WU-01 to WU-10 | P0/P1/P2 | AC4, AC5, AC7, AC8 |
| `services/notification/tests/unit/test_microsoft_graph_client.py` | S09.9-GU-01 to GU-06 | P1/P2 | AC7 (parse_retry_after), Task 5.2 |
| `services/notification/tests/unit/test_microsoft_oauth.py` | S09.9-OU-01 to OU-03 | P0/P1 | AC5, E09-R-001 |
| `services/notification/tests/integration/test_calendar_sync_outlook.py` | S09.9-I-01, I-02 | P1 | AC4, AC8 |

---

## Acceptance Criteria Coverage

| AC | Description | Test IDs | Priority |
|---|---|---|---|
| AC1 | OAuth2 connect endpoint (302, scope Calendars.ReadWrite+offline_access+User.Read, prompt=consent, state) | S09.9-A-01, A-02, A-11 | P1 |
| AC2 | OAuth2 callback (token exchange, Fernet encryption, UPSERT provider=microsoft, audit, redirect, error paths, B1 CSRF) | S09.9-A-03, A-04, A-05, A-06, A-10 | P0/P1 |
| AC3 | Disconnect (delete local row, NO revocation call, 204, 404 if missing, cross-tenant isolation 404) | S09.9-A-07, A-08, A-09 | P2 |
| AC4 | Periodic sync (diff: creates, updates, deletes; sync_log always written) | S09.9-WU-01 to WU-05, WU-09, WU-10, I-01, I-02 | P1/P2 |
| AC5 | Token refresh (transparent, invalid_grant handling, non-401 errors) | S09.9-WU-07, WU-08, OU-01, OU-02 | P1 |
| AC6 | No new migration — pre-existing enum supports 'microsoft' | S09.9-M-01, M-02 | P1 |
| AC7 | Rate limiting (429/503 respect Retry-After header; parse_retry_after helper) | S09.9-WU-06, WU-10, GU-01 to GU-04 | P1 |
| AC8 | Event body shape + transactionId dedup key = str(opp.id) | S09.9-WU-01, I-01 | P1 |
| AC9 | Observability (4 structured log events) | S09.9-OU-03 (token exposure), WU-08 (structured warning) | P1 |
| AC10 | Docker Compose / env vars | Not tested by pytest (manual verification) | — |
| AC11 | Test design mapping (P1/P2 coverage) | Mapped across all test IDs | — |
| AC12 | No undocumented endpoints | 3 endpoints only: connect/callback/disconnect | — |

---

## Risk Mitigation Tests

| Risk | Test ID | Description | Gate Level |
|---|---|---|---|
| E09-R-001 | S09.9-A-06 | Callback logs MUST NOT contain plaintext refresh or access token | **P0 exit gate** |
| E09-R-001 | S09.9-A-03 | Stored `encrypted_access_token` in DB != plaintext bytes | **P0 exit gate** |
| E09-R-001 | S09.9-OU-03 | `refresh_microsoft_access_token` never logs token values | **P0 exit gate** |
| E09-R-007 | S09.9-WU-05 | Partial Graph failure: successful events persisted, failures captured in sync_log | **P0 exit gate** |
| E09-R-008 | S09.9-WU-06 | HTTP 429 with Retry-After: 30 → `self.retry(countdown=30, max_retries=5)` | **P1 exit gate** |
| E09-R-008 | S09.9-WU-10 | HTTP 503 with Retry-After: 30 → same retry behavior | **P1 exit gate** |
| B1 CSRF | S09.9-A-05 | Callback without session cookie → 302 error redirect; NO JWT fallback | **P1 exit gate** |
| Rule #38 | S09.9-A-09 | User A DELETE → 404 on User B's connection (not 403) | **P1 exit gate** |
| AC3 deferred-revoke | S09.9-A-07 | No outbound HTTP call on disconnect (no revoke endpoint for our scopes) | **P2** |

All four P0 risk tests must be **GREEN** before the story is considered done (see Epic 09 exit criteria).

---

## Test Infrastructure Requirements

### Fixtures Required (already exist in conftest.py)

**Client API:**
- `client_api_session_factory` — async DB session factory (client-api)
- `migration_session_factory` — migration-role DB session
- `rsa_test_key_pair` + `rsa_env_setup` — RS256 JWT for auth
- `test_redis_client` — Redis test client (DB index 1)
- `calendar_encryption_key_setup` — autouse fixture that sets CALENDAR_ENCRYPTION_KEY
- `app` — FastAPI app fixture

**Notification:**
- `notification_session_factory` — async notification DB session
- `celery_config` — Celery eager mode config

### Fixtures Added in New Test Files

- `calendar_authed_client` (in `test_calendar_microsoft_oauth_api.py`) — seeds user + authenticated HTTPX client (mirrors `test_calendar_google_oauth_api.py`)
- `mig_engine` / `notif_engine` (in `test_provider_enum_microsoft.py`) — direct async engines for enum verification
- `su_factory`, `notif_sync_factory`, `notif_factory` (in `test_calendar_sync_outlook.py`) — module-scoped session factories for integration tests (mirrors `test_calendar_sync_google.py`)
- `ms_connection`, `mock_session`, `mock_self` fixtures (in `test_outlook_calendar_sync_unit.py`)

### Environment Variables Required

| Variable | Purpose | Test Setup |
|---|---|---|
| `CALENDAR_ENCRYPTION_KEY` | Fernet key for token crypto | Auto-generated via `calendar_encryption_key_setup` autouse fixture |
| `CLIENT_API_DATABASE_URL` | Client API DB connection | Set by `db_url_env_setup` autouse fixture |
| `NOTIFICATION_DATABASE_URL` | Notification role DB URL | Set by integration tests directly |
| `SUPERUSER_TEST_DATABASE_URL` | Superuser DB URL for seeding | Used in integration tests |
| `CLIENT_API_MICROSOFT_CLIENT_ID` | Microsoft app credentials | Not required for tests (mocked) |
| `CLIENT_API_MICROSOFT_CLIENT_SECRET` | Microsoft app credentials | Not required for tests (mocked) |

### External Mocking Strategy

| External System | Mock Method | Used In |
|---|---|---|
| Microsoft OAuth2 token exchange | `patch("client_api.core.oauth.oauth.microsoft_calendar.authorize_access_token")` | API tests (A-03 to A-06) |
| Microsoft token refresh endpoint | `patch("notification.workers.tasks.outlook_calendar_sync.refresh_microsoft_access_token")` as `AsyncMock` | Unit tests (WU-07, WU-08) |
| Microsoft Graph POST `/me/events` | `patch("notification.workers.tasks.outlook_calendar_sync.create_event")` as `AsyncMock` | Unit + integration tests |
| Microsoft Graph PATCH `/me/events/{id}` | `patch("notification.workers.tasks.outlook_calendar_sync.update_event")` as `AsyncMock` | Unit + integration tests |
| Microsoft Graph DELETE `/me/events/{id}` | `patch("notification.workers.tasks.outlook_calendar_sync.delete_event")` as `AsyncMock` | Unit + integration tests |
| `parse_retry_after` helper | Real function tested directly (no mock needed) | Unit tests (GU-01 to GU-04) |
| Microsoft refresh endpoint (direct httpx) | `respx.mock` at httpx layer | OAuth helper tests (OU-01, OU-02) |
| Graph API (direct httpx) | `respx.mock` at httpx layer | Graph client tests (GU-05, GU-06) |

**Never use real Microsoft credentials in CI.** All OAuth and Graph API calls are mocked.

**Key difference from Story 9.8:** `respx.mock` is used directly at the httpx layer for Graph API calls (unlike 9.8 which used `patch("...build_calendar_service")` for the google-api-python-client). This is cleaner and avoids the `_google_mock.py`-style helper.

---

## Acceptance Criteria NOT Covered by Generated Tests

| Item | Reason | Next Action |
|---|---|---|
| AC9: Observability — all 4 structured log events emitted correctly | `structlog_capture` fixture for all log variants deferred to automate phase | Add full structlog capture assertions in `bmad-testarch-automate` phase |
| AC10: Docker Compose / `.env.example` configuration | Infrastructure config; not testable via pytest | Manual verification + CI env var check |
| AC3: `microsoft_token_revoke_deferred` structured log emitted | A-07 verifies no outbound HTTP call; log content assertion deferred to automate | Add `caplog` assertion for `microsoft_token_revoke_deferred` in automate phase |
| P3: E2E "connect Outlook → opportunity appears in Outlook calendar" | Manual QA only; NOT a sprint-exit blocker per AC11 | Tagged `@pytest.mark.slow @pytest.mark.skip_ci` when added |
| AC7: `self.retry` not triggered more than once within a rate-limit window (thundering-herd) | Complex multi-cycle Celery simulation; deferred | Add multi-invocation timing assertion in automate phase |

---

## Acceptance Criteria Mapping (from Story AC11 Test Design)

| Priority | Epic Test Design Description | Story Task | Test ID(s) |
|---|---|---|---|
| **P1** | Microsoft Outlook OAuth2 callback — exchanges code for tokens; stores Fernet-encrypted token with `provider='microsoft'` (3 tests) | Tasks 8.1, 8.2, 8.3 | S09.9-A-01, A-02, A-03 |
| **P1** | Microsoft Graph sync creates/updates/deletes events; 429 respects `Retry-After` header (4 tests) | Tasks 7.1, 7.2, 7.3, 7.6 | S09.9-WU-01, WU-02, WU-03, WU-06 |
| **P2** | Microsoft disconnect — deletes connection row (2 tests) | Tasks 8.7, 8.8 | S09.9-A-07, A-08 |
| **Cross-cutting** | E09-R-001 never-log-token | Task 8.6 (callback), OU-03 (refresh) | S09.9-A-06, OU-03 |
| **Cross-cutting** | E09-R-007 partial-failure diff | Task 7.5 | S09.9-WU-05 |
| **Cross-cutting** | E09-R-008 Graph 429 thundering-herd | Tasks 7.6, 7.7 | S09.9-WU-06, GU-01 to GU-04 |
| **Cross-cutting** | Cross-tenant isolation (rule #38) | Task 8.9 | S09.9-A-09 |
| **P1** | Schema pre-existing invariants (AC6) | Tasks 10.1, 10.2 | S09.9-M-01, M-02 |
| **P1** | transactionId dedup key (AC8) | Tasks 7.1, 9.1 | S09.9-WU-01, I-01 |

---

## TDD Green Phase Instructions

Once the feature is implemented, follow this sequence to enable tests:

### Step 1 — Verify no new migration needed

```bash
cd eusolicit-app
# Confirm provider='microsoft' already supported (pre-existing from migrations 033/034)
make test-service SVC=client-api -- -m integration -k "test_calendar_provider_enum"
make test-service SVC=notification -- -m integration -k "test_notification_sync_log_enum"
```

### Step 2 — Remove skip markers (per task completion order)

| When Task Completes | Remove Skips From |
|---|---|
| Task 1 (config settings) + Task 2 (oauth registration) | `test_oauth_registration.py` |
| Task 3 (Microsoft OAuth endpoints in calendar.py) | `test_calendar_microsoft_oauth_api.py` |
| Task 4 (microsoft_oauth.py refresh helper) | `test_microsoft_oauth.py` |
| Task 5 (microsoft_graph_client.py + parse_retry_after) | `test_microsoft_graph_client.py` |
| Task 6 (outlook_calendar_sync.py Celery task) | `test_outlook_calendar_sync_unit.py` |
| Tasks 9 + 10 (integration + enum) | `test_calendar_sync_outlook.py`, `test_provider_enum_microsoft.py` |

### Step 3 — Run tests in priority order

```bash
# P0 — E09-R-001: never-log-token gates
cd eusolicit-app/services/client-api
pytest tests/api/test_calendar_microsoft_oauth_api.py::test_microsoft_callback_never_logs_refresh_token -v
cd ../notification
pytest tests/unit/test_microsoft_oauth.py::test_refresh_microsoft_access_token_never_logs_token -v

# P0/P1 — Callback encryption gate (E09-R-001)
cd ../client-api
pytest tests/api/test_calendar_microsoft_oauth_api.py::test_microsoft_callback_encrypts_and_stores_tokens -v

# P1 — E09-R-007: partial failure preserves successes
cd ../notification
pytest tests/unit/test_outlook_calendar_sync_unit.py::test_outlook_sync_partial_failure_preserves_successes -v

# P1 — E09-R-008: 429 respects Retry-After
pytest tests/unit/test_outlook_calendar_sync_unit.py::test_outlook_sync_429_respects_retry_after -v

# P1 — Full API test suite
cd ../client-api
pytest tests/api/test_calendar_microsoft_oauth_api.py -v -m api

# P1 — Sync unit tests
cd ../notification
pytest tests/unit/test_outlook_calendar_sync_unit.py -v -m unit

# P1 — Graph client helpers
pytest tests/unit/test_microsoft_graph_client.py -v -m unit
pytest tests/unit/test_microsoft_oauth.py -v -m unit

# P1 — Enum invariants
cd ../client-api
pytest tests/integration/test_provider_enum_microsoft.py -v -m integration

# Integration — full round-trip
cd ../notification
pytest tests/integration/test_calendar_sync_outlook.py -v -m integration

# P1 — OAuth registration
cd ../client-api
pytest tests/unit/test_oauth_registration.py -v -m unit
```

### Step 4 — Verify coverage gates

```bash
cd eusolicit-app
make coverage  # must stay ≥80% on:
               # microsoft_oauth.py (notification service)
               # microsoft_graph_client.py (notification service)
               # outlook_calendar_sync.py (notification service)
               # calendar.py (client-api — new Microsoft endpoints)
```

---

## Exit Criteria for Story 9.9

- [ ] `test_microsoft_callback_encrypts_and_stores_tokens` PASSES — stored token bytes != plaintext (E09-R-001)
- [ ] `test_microsoft_callback_never_logs_refresh_token` PASSES — no plaintext token in logs (E09-R-001)
- [ ] `test_refresh_microsoft_access_token_never_logs_token` PASSES — refresh helper safe (E09-R-001)
- [ ] `test_outlook_sync_partial_failure_preserves_successes` PASSES — (E09-R-007)
- [ ] `test_outlook_sync_429_respects_retry_after` PASSES — (E09-R-008)
- [ ] `test_microsoft_disconnect_cross_tenant_isolation` PASSES — (rule #38)
- [ ] `test_microsoft_callback_missing_session_redirects` PASSES — no JWT fallback (B1 CSRF)
- [ ] `test_microsoft_disconnect_deletes_connection` PASSES — no outbound HTTP call (AC3 deferred-revoke)
- [ ] `test_outlook_sync_integration_create_update_delete_cycle` PASSES — full round-trip (AC4, AC8)
- [ ] `test_calendar_provider_enum_supports_microsoft` PASSES — pre-existing enum verified (AC6)
- [ ] Coverage ≥80% on `microsoft_oauth.py`, `microsoft_graph_client.py`, `outlook_calendar_sync.py`, new endpoints in `calendar.py`
- [ ] `make lint` and `make type-check` pass for both services

---

## Notes for Developer

1. **Import lazy pattern**: All test imports from production modules are inside the test function body (e.g., `from notification.workers.tasks.outlook_calendar_sync import _sync_one_outlook_connection`). This prevents `ImportError` collection failures when the module doesn't exist yet.

2. **`asyncio_mode = "auto"`**: No `@pytest.mark.asyncio` needed on `async def test_*` functions.

3. **AsyncMock for Graph functions**: `create_event`, `update_event`, `delete_event` are async functions patched as `AsyncMock`. They are called inside `asyncio.run(_async_sync_one_connection(...))`. The mocked coroutines are correctly awaited inside `asyncio.run()`.

4. **`self` as MagicMock**: The Celery task uses `bind=True`. Unit tests create `mock_self = MagicMock()` and pass it as the first argument to `_sync_one_outlook_connection(mock_self, connection_id)`. This allows `mock_self.retry.assert_called_once()` assertions.

5. **`self.retry` behavior**: In production, `self.retry(exc=exc, countdown=N)` raises `celery.exceptions.Retry`. In unit tests, `mock_self.retry` is a `MagicMock` that just records the call. The test then asserts call args.

6. **AC3 disconnect deferral**: `test_microsoft_disconnect_deletes_connection` uses `respx.mock` with NO routes registered. If the implementation accidentally makes an outbound HTTP call, `respx` raises `httpx.ConnectError` (no route matched), causing the test to fail. This is intentional — it enforces the AC3 deferral.

7. **`transactionId` vs `iCalUID`**: Microsoft uses `transactionId` (not `iCalUID`) as the dedup key. The assertion checks `body["transactionId"] == str(opp.id)` (36-char UUID string, within Microsoft's 40-char limit). This is the equivalent of Story 9.8's `iCalUID == f"{opp.id}@eusolicit.com"` but shorter.

8. **Cross-service coexistence test (I-02)**: `test_outlook_sync_microsoft_provider_coexists_with_google` runs both Google and Microsoft sync tasks against the same user's connections. It asserts each task only touched its provider's `calendar_events` rows and produced separate `sync_log` rows. This is the regression guard that confirms the two tasks don't interfere.

9. **Notification unit tests use sync sessions**: The Celery worker runs sync (psycopg2) SQLAlchemy sessions. The unit tests mock `get_sync_session_factory` entirely with `MagicMock`. Integration tests inject `notif_sync_factory` (real psycopg2 session). Do NOT mix engines.

10. **`SKIP_REASON` constant**: Each test file defines `SKIP_REASON` at module level:
    - API: `"RED phase: Story 9.9 Microsoft Outlook OAuth2 endpoints not yet implemented"`
    - Sync unit: `"RED phase: Story 9.9 Microsoft Outlook sync task not yet implemented"`
    - Graph client: `"RED phase: Story 9.9 microsoft_graph_client.py not yet implemented"`
    - OAuth helper: `"RED phase: Story 9.9 microsoft_oauth.py not yet implemented"`
    - Integration: `"RED phase: Story 9.9 Outlook calendar sync integration not yet implemented"`
    - Enum: `"RED phase: Story 9.9 provider enum verification (pre-existing migrations 033/034)"`
    - OAuth reg: `"RED phase: Story 9.9 microsoft_calendar OAuth registration not yet implemented"`

---

**Generated by:** BMad TEA Agent — ATDD Workflow (bmad-testarch-atdd)
**Execution Mode:** Sequential (backend project, AI generation)
**Version:** BMad v6
