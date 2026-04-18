---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: bmad-testarch-atdd
mode: create
storyId: 6-7-document-download-api
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-7-document-download-api.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_document_upload.py
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.7 — Document Download API

**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)
**Date:** 2026-04-17
**Story:** `eusolicit-docs/implementation-artifacts/6-7-document-download-api.md`
**Stack:** Backend (Python / FastAPI / pytest-asyncio / httpx ASGITransport)
**Generation Mode:** AI Generation (sequential)

---

## Step 1: Preflight & Context Summary

### Stack Detection

- **Detected stack:** `backend`
- **Indicators:** `pyproject.toml` at `eusolicit-app/services/client-api/pyproject.toml`; pytest + pytest-asyncio test suite; `conftest.py` with session-factory pattern and RSA JWT fixtures
- **Test framework:** pytest + pytest-asyncio + httpx AsyncClient (ASGITransport)
- **No frontend indicators** (`package.json` / `playwright.config.*` / `next.config.*` not present at root)

### Prerequisites

- [x] Story 6.7 approved with clear acceptance criteria (Status: `ready-for-dev`)
- [x] pytest configuration present (`pyproject.toml`)
- [x] Existing test patterns confirmed:
  - `test_document_upload.py` (immediate predecessor — mirrors the download pattern)
  - `test_opportunity_detail.py` (DB fixture + httpx pattern)
- [x] `client_api_session_factory` fixture confirmed in `conftest.py`
- [x] `rsa_test_key_pair` / `rsa_env_setup` (autouse) JWT fixtures confirmed in `conftest.py`
- [x] `app` fixture (with `get_db_session` override + rollback) confirmed in `conftest.py`
- [x] `patch("client_api.services.document_service._get_s3_client")` mock pattern documented in story Dev Notes and used in `test_document_upload.py`
- [x] `AuditLog` ORM model import path confirmed: `client_api.models.audit_log`
- [x] `documents_table` import path confirmed: `client_api.models.client_document`
- [x] Epic-level test design loaded from `test-design-epic-06.md`

### Story Context Loaded

- **Endpoint:** `GET /api/v1/opportunities/{opportunity_id}/documents/{document_id}/download`
- **New service function:** `download_document` in `document_service.py`
- **New schema:** `DocumentDownloadResponse` in `schemas/documents.py`
- **New config field:** `document_presigned_get_expiry: int` (default 600) in `ClientApiSettings`
- **New route:** `GET /{doc_id}/download` registered on the documents sub-router
- **No new Alembic migration** — reuses `client.documents` table from S06.06
- **Security invariants:**
  - Requires valid Bearer JWT (AC1)
  - Owner (`user_id` match) OR same-company paid-tier member (AC2)
  - `scan_status = 'clean'` required; `pending` → 422, `infected`/`failed` → 422 (AC4, AC5)
  - Presigned GET URL with `ResponseContentDisposition: attachment` (AC6)
  - Audit log written via `write_audit_entry` on success (AC7)

### Epic Test Design Context (test-design-epic-06.md)

| Risk ID | Score | Category | Relation to S06.07 |
|---------|-------|----------|--------------------|
| E06-R-003 | 6 | SEC | **ClamAV pre-scan gap** — scan_status must gate download; this story closes the HTTP-level gate (S06.06 only verified DB state) |
| E06-R-008 | 4 | SEC | ClamAV timeout → `failed` status; download gate for `failed` tested here |

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (full test file)

**Rationale:** Story 6.7 adds a single new endpoint with no existing test coverage in
`test_document_download.py`. All ACs are well-specified with exact HTTP status codes,
error body schemas, and presigned URL parameter requirements. The mock pattern
(`_get_s3_client`) is established in `test_document_upload.py`. Sequential AI generation
is appropriate — one test file, no parallel subagent overhead.

---

## Step 3: Test Strategy

### Coverage Strategy (from test-design-epic-06.md + AC mapping)

| Priority | Risk / AC | Test Scenario | Test Level |
|----------|-----------|---------------|------------|
| P0 | E06-R-003 / AC4 | `scan_status=pending` → 422 scan_pending (HTTP-level gate) | API |
| P0 | E06-R-003 / AC5 | `scan_status=infected` → 422 scan_failed (HTTP-level gate) | API |
| P1 | E06-P1-018 / AC6 | Owner downloads clean doc; presigned URL + expires_in + filename in response | API |
| P1 | E06-P1-018 / AC6 | Same-company paid-tier member downloads clean doc | API |
| P1 | E06-P1-019 / AC2 | Different-company user → 403 access_denied | API |
| P1 | AC2 (implicit) | Same-company free-tier non-owner → 403 access_denied | API |
| P1 | AC3 | Non-existent document_id → 404 not_found | API |
| P1 | AC3 | Soft-deleted document → 404 not_found | API |
| P1 | E06-P1-020 / AC7 | Audit log entry present in shared.audit_log after success | API |
| P1 | AC1 | Unauthenticated request → 401 | API |
| P1 | AC5 (implicit) | `scan_status=failed` (timeout) → 422 scan_failed | API |
| P2 | AC9 | Endpoint appears in /openapi.json with 200/401/403/404/422 responses | API |

### Mock Strategy

| Component | Mock Approach | Rationale |
|-----------|--------------|-----------|
| boto3 S3 client | `patch("client_api.services.document_service._get_s3_client", return_value=MagicMock())` | No real AWS credentials in CI; reuses established pattern from `test_document_upload.py` |
| DB session | `download_app` fixture overrides `get_db_session` with committing session factory | Committed sessions needed so seeded documents are visible to the endpoint |
| Audit log | Direct `AuditLog` ORM query via `client_api_session_factory` | Verifies `write_audit_entry` actually persisted the row |

### Fixtures Required

| Fixture | Source | Purpose |
|---------|--------|---------|
| `download_app` | New (function-scoped) | Sets env vars, clears settings cache, overrides `get_db_session` with committing sessions |
| `cleanup_documents` | New (autouse, function-scoped) | Deletes all `client.documents` rows for `_TEST_OPP_ID` after each test |
| `_seed_document()` | New helper (not fixture) | Inserts a `client.documents` row with configurable scan_status/deleted |
| `_make_jwt()` | New helper | Issues RS256 JWT using session-scoped `rsa_test_key_pair` from conftest |
| `rsa_test_key_pair` | `conftest.py` (session-scoped) | RSA key pair for test JWT signing |
| `client_api_session_factory` | `conftest.py` | For seeding and direct DB assertions |
| `app` | `conftest.py` | Base FastAPI app with standard overrides |

### Red Phase Requirements

All 11 test functions are decorated with:
```python
@pytest.mark.skip(reason="RED PHASE: S06.07 not yet implemented")
```

Tests will fail with `404 Not Found` (route not registered) or `ImportError`
(schema/service not yet implemented) until S06.07 is fully implemented.

---

## Step 4: Tests Generated

**Output file:** `eusolicit-app/services/client-api/tests/api/test_document_download.py`

All tests are decorated with `@pytest.mark.skip(reason="RED PHASE: S06.07 not yet implemented")`.
They will fail with `ImportError` or `404 Not Found` until S06.07 is fully implemented.

### Test Inventory

| Test Function | Test ID | Priority | AC(s) | Skip Reason |
|---------------|---------|----------|-------|-------------|
| `test_owner_downloads_clean_document` | E06-P1-018 | P1 | AC1, AC2, AC6 | RED PHASE |
| `test_same_company_paid_member_downloads` | E06-P1-018 (pt.2) | P1 | AC2, AC6 | RED PHASE |
| `test_different_company_denied` | E06-P1-019 | P1 | AC2 | RED PHASE |
| `test_same_company_free_tier_denied` | (Implicit) | P1 | AC2 | RED PHASE |
| `test_pending_scan_returns_422` | E06-P0-006 | P0 | AC4 | RED PHASE |
| `test_infected_scan_returns_422` | E06-P0-007 | P0 | AC5 | RED PHASE |
| `test_failed_scan_returns_422` | (Implicit) | P1 | AC5 | RED PHASE |
| `test_nonexistent_document_returns_404` | (Implicit) | P1 | AC3 | RED PHASE |
| `test_soft_deleted_document_returns_404` | (Implicit) | P1 | AC3 | RED PHASE |
| `test_audit_log_written_on_download` | E06-P1-020 | P1 | AC7 | RED PHASE |
| `test_unauthenticated_returns_401` | (Implicit) | P1 | AC1 | RED PHASE |
| `test_download_endpoint_in_openapi` | (Implicit) | P2 | AC9 | RED PHASE |

**Total tests:** 12 test cases
**P0:** 2 | **P1:** 9 | **P2:** 1

---

## Step 5: Validation & Completion

### AC Coverage Map

| AC | Description | Tests Covering |
|----|-------------|----------------|
| AC1 | Bearer JWT required; 401 if absent/invalid | `test_unauthenticated_returns_401`, `test_owner_downloads_clean_document` |
| AC2 | Owner OR same-company paid-tier member; 403 with `access_denied` otherwise | `test_different_company_denied`, `test_same_company_free_tier_denied`, `test_same_company_paid_member_downloads`, `test_owner_downloads_clean_document` |
| AC3 | 404 with `not_found` for absent or soft-deleted documents | `test_nonexistent_document_returns_404`, `test_soft_deleted_document_returns_404` |
| AC4 | 422 with `scan_pending` when scan_status=pending | `test_pending_scan_returns_422` |
| AC5 | 422 with `scan_failed` when scan_status=infected or failed | `test_infected_scan_returns_422`, `test_failed_scan_returns_422` |
| AC6 | 200 with `download_url`, `expires_in=600`, `filename`; `ResponseContentDisposition: attachment` in S3 params | `test_owner_downloads_clean_document`, `test_same_company_paid_member_downloads` |
| AC7 | Audit log entry written via `write_audit_entry` on success | `test_audit_log_written_on_download` |
| AC8 | `document_presigned_get_expiry` config field (default 600) | Implicitly verified by `expires_in=600` assertion in P1-018 tests |
| AC9 | Endpoint in `/openapi.json` with 200/401/403/404/422 responses | `test_download_endpoint_in_openapi` |
| AC10 | Integration tests in `tests/api/test_document_download.py` covering all 10 scenarios | This checklist + generated test file |

### Risk Coverage Map

| Risk ID | Score | Covered By |
|---------|-------|-----------|
| E06-R-003 (ClamAV pre-scan gap) | 6 | `test_pending_scan_returns_422` (P0), `test_infected_scan_returns_422` (P0) |
| E06-R-008 (ClamAV timeout → failed) | 4 | `test_failed_scan_returns_422` (P1) |

### TDD Red Phase Verification

- [x] All 12 tests decorated with `@pytest.mark.skip(reason="RED PHASE: S06.07 not yet implemented")`
- [x] All tests assert EXPECTED behavior (not placeholder assertions)
- [x] No passing tests generated (correct — this is red phase)
- [x] Tests will fail with `404 Not Found` (route not registered) or `AttributeError`/`ImportError` when skip is removed and S06.07 is not implemented

### Green Phase Checklist

When implementing S06.07, remove `@pytest.mark.skip` decorators **one group at a time**:

- [ ] **Group 1 — Schema + Config (AC8, AC9 partial):** Remove skip from `test_download_endpoint_in_openapi`
  - Implement: `DocumentDownloadResponse` schema, `document_presigned_get_expiry` config field, register `GET /{doc_id}/download` route (empty handler returning 501 is enough to pass this test)

- [ ] **Group 2 — Auth gate (AC1):** Remove skip from `test_unauthenticated_returns_401`
  - Implement: `get_current_user` dependency on the route handler

- [ ] **Group 3 — Not Found + Soft-Delete (AC3):** Remove skip from `test_nonexistent_document_returns_404`, `test_soft_deleted_document_returns_404`
  - Implement: DB query with `WHERE deleted_at IS NULL`, raise `AppException(status_code=404, error="not_found")`

- [ ] **Group 4 — Access Control (AC2):** Remove skip from `test_different_company_denied`, `test_same_company_free_tier_denied`, `test_owner_downloads_clean_document`, `test_same_company_paid_member_downloads`
  - Implement: `is_owner` / `is_company_member` / `is_paid` logic, raise `AppException(status_code=403, error="access_denied")`

- [ ] **Group 5 — Scan Gates (AC4, AC5):** Remove skip from `test_pending_scan_returns_422`, `test_infected_scan_returns_422`, `test_failed_scan_returns_422`
  - Implement: `scan_status` checks before presigned URL generation

- [ ] **Group 6 — Happy Path (AC6):** Remove skip from `test_owner_downloads_clean_document`, `test_same_company_paid_member_downloads` (if not already unblocked)
  - Implement: `s3.generate_presigned_url("get_object", Params={..., "ResponseContentDisposition": "attachment; filename=..."}, ExpiresIn=600)`, return `DocumentDownloadResponse`

- [ ] **Group 7 — Audit (AC7):** Remove skip from `test_audit_log_written_on_download`
  - Implement: `await write_audit_entry(db, user_id=..., action_type="document.download", entity_type="document", entity_id=document_id)`

### Known Limitations / Notes

1. **`_make_jwt` uses `rsa_test_key_pair` from conftest** — the `rsa_env_setup` autouse fixture (session-scoped) populates `core.security._private_key_cache` before any test runs, so `create_access_token` will work without additional setup.

2. **`download_app` calls `get_settings.cache_clear()`** — needed because env vars are set via `os.environ.setdefault` after the session-level DB URL fixture may have already cached settings. This mirrors the pattern in `test_document_upload.py::upload_app`.

3. **Audit log cleanup** — `cleanup_documents` only cleans `client.documents` rows. The `shared.audit_log` table may accumulate entries across tests. This is acceptable for tests that *read* the audit log (`test_audit_log_written_on_download`) because they query by `entity_id` (the specific `doc_id` seeded in that test, which is a random UUID).

4. **OpenAPI test (AC9)** is scoped as P2. It verifies the route is registered but does not validate full request/response schema shape — that is implicitly validated by the other integration tests.

5. **`document_presigned_get_expiry` env var name** — the `ClientApiSettings` field uses `env="DOCUMENT_PRESIGNED_GET_EXPIRY"` (per AC8). In the `download_app` fixture we set `CLIENT_API_DOCUMENT_PRESIGNED_GET_EXPIRY` to follow the pydantic-settings v2 `env_prefix="CLIENT_API_"` convention. Verify the actual pydantic field alias in `config.py` (some fields use bare aliases like `INTERNAL_SERVICE_SECRET` without the `CLIENT_API_` prefix — check and adjust env var name if needed during green phase).

6. **`AuditLog.entity_id` type** — assumed to be `UUID` (same type as `documents_table.c.id`). Verify the column type on the `AuditLog` ORM model; if it is `str`, adjust the assertion in `test_audit_log_written_on_download` accordingly.

7. **`DOCUMENT_PRESIGNED_GET_EXPIRY` env var name** — per AC8, the field uses `env="DOCUMENT_PRESIGNED_GET_EXPIRY"` directly (no `CLIENT_API_` prefix). The `download_app` fixture sets both `CLIENT_API_DOCUMENT_PRESIGNED_GET_EXPIRY` and should also set the bare `DOCUMENT_PRESIGNED_GET_EXPIRY` if the field does not use the prefix. Confirm and update during green phase.

---

## Change Log

| Date | Actor | Change |
|------|-------|--------|
| 2026-04-17 | BMad ATDD (bmad-testarch-atdd) | Initial ATDD checklist generated for Story 6.7; test file written to `eusolicit-app/services/client-api/tests/api/test_document_download.py`; 12 test cases covering E06-P0-006, P0-007, P1-018, P1-019, P1-020 and implicit auth/tier/scan/404 scenarios; all tests in RED phase with skip markers |
