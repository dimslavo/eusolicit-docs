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
storyId: 6-6-document-upload-api-s3-clamav
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-6-document-upload-api-s3-clamav.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_opportunity_listing.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.6 — Document Upload API (S3 + ClamAV)

**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)
**Date:** 2026-04-17
**Story:** `eusolicit-docs/implementation-artifacts/6-6-document-upload-api-s3-clamav.md`
**Stack:** Backend (Python / FastAPI / pytest-asyncio)
**Generation Mode:** AI Generation (sequential)

---

## Step 1: Preflight & Context Summary

### Stack Detection

- **Detected stack:** `backend`
- **Indicators:** `pyproject.toml` at `eusolicit-app/services/client-api/pyproject.toml`; pytest-asyncio test suite; `conftest.py` with `client_api_session_factory`, `superuser_session_factory`, `starter_user_token`, `free_user_token` fixtures
- **Test framework:** pytest + pytest-asyncio + httpx AsyncClient

### Prerequisites

- [x] Story 6.6 approved with clear acceptance criteria (Status: ready-for-dev)
- [x] pytest configuration present (`pyproject.toml`)
- [x] Existing test patterns established (`test_opportunity_listing.py`, `test_opportunity_detail.py`)
- [x] `superuser_session_factory` and `client_api_session_factory` patterns confirmed in `conftest.py`
- [x] `starter_user_token` and `free_user_token` JWT fixtures confirmed in `conftest.py`
- [x] `get_db_session` override pattern confirmed from S06.04/S06.05
- [x] `patch("client_api.services.document_service._get_s3_client")` mock pattern documented in story Dev Notes
- [x] Epic-level test design loaded from `test-design-epic-06.md`

### Story Context Loaded

- **Endpoints:**
  - `POST /api/v1/opportunities/{opportunity_id}/documents/upload` — presigned PUT URL
  - `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/confirm` — dispatch ClamAV
  - `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/scan-result` — internal callback
- **New table:** `client.documents` (Alembic migration `019_documents`)
- **New service:** `document_service.py` with `request_upload`, `confirm_upload`, `record_scan_result`
- **New Celery dispatch:** `celery_producer.dispatch_document_scan`
- **Security invariant:** `scan_status` defaults to `pending`; download (S06.07) requires `scan_status = clean`

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (full test file)

**Rationale:** Story 6.6 introduces an entirely new module with no existing test coverage.
Three endpoints, one new DB table, S3 presigned URL generation, ClamAV async flow, and an
internal callback auth mechanism all require a dedicated test file from scratch.

---

## Step 3: Test Strategy

### Coverage Strategy (from test-design-epic-06.md)

| Priority | Risk | Coverage Approach |
|----------|------|-------------------|
| P0 | E06-R-003 (ClamAV pre-scan gap, score 6) | `test_pending_document_not_downloadable_state` (DB state), `test_infected_scan_marks_deleted` (scan_status + deleted_at + S3 tag) |
| P0 | E06-R-006 (Concurrent package size race, score 4) | `test_concurrent_package_size_race` (asyncio.gather + SELECT FOR UPDATE) |
| P1 | E06-P1-016 (MIME type validation) | Parametrized allowed (4 types) + rejected (2 types) |
| P1 | E06-P1-017 (Presigned URL + confirm flow) | End-to-end: upload → presigned URL → confirm → DB assert |
| P2 | E06-P2-005 (File > 100MB rejected) | Single assert on size boundary |
| Implicit | Auth / tier gates | 401 unauthenticated, 403 free-tier, 401 missing/wrong internal key |
| Implicit | Clean scan flow | scan_status=clean, deleted_at=NULL |

### Mock Strategy

| Component | Mock Approach | Rationale |
|-----------|--------------|-----------|
| boto3 S3 client | `patch("client_api.services.document_service._get_s3_client")` returns `MagicMock` | No real AWS credentials in CI; mock scoped to module level as documented in Dev Notes |
| Celery dispatch | `patch("client_api.services.document_service.dispatch_document_scan")` | Celery broker unavailable in CI; confirmed safe per story Dev Notes (dispatch failure is logged, not re-raised) |
| DB session | `upload_app` fixture overrides `get_db_session` with `client_api_session_factory` | Same override pattern used in all prior epic-6 tests |

### Fixtures Required

| Fixture | Source | Purpose |
|---------|--------|---------|
| `upload_app` | New (module-scoped) | Sets env vars, clears settings cache, overrides `get_db_session` |
| `seeded_upload_opportunity` | New (module-scoped) | Seeds `pipeline.opportunities` row for `_TEST_OPP_ID` |
| `starter_headers` | Derived from `starter_user_token` (conftest) | Paid-tier auth header |
| `free_headers` | Derived from `free_user_token` (conftest) | Free-tier auth header |
| `internal_headers` | New (function-scoped) | `X-Internal-Service-Key` header for scan-result endpoint |
| `test_client` | `conftest.py` | `httpx.AsyncClient` bound to the app |
| `client_api_session_factory` | `conftest.py` | For direct DB assertions post-endpoint call |
| `superuser_session_factory` | `conftest.py` | For seeding `client.documents` rows in race condition test |

---

## Step 4: Tests Generated

**Output file:** `eusolicit-app/services/client-api/tests/api/test_document_upload.py`

All tests are decorated with `@pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")`.
They will fail with `ImportError` or `404 Not Found` until S06.06 is fully implemented.

### Test Inventory

| Test Function | Test ID | Priority | AC | Skip Reason |
|---------------|---------|----------|----|-------------|
| `test_upload_rejects_oversized_file` | E06-P2-005 | P2 | AC3 | RED PHASE |
| `test_upload_allowed_mime_types[application/pdf]` | E06-P1-016 | P1 | AC3 | RED PHASE |
| `test_upload_allowed_mime_types[…docx]` | E06-P1-016 | P1 | AC3 | RED PHASE |
| `test_upload_allowed_mime_types[…xlsx]` | E06-P1-016 | P1 | AC3 | RED PHASE |
| `test_upload_allowed_mime_types[application/zip]` | E06-P1-016 | P1 | AC3 | RED PHASE |
| `test_upload_rejected_mime_types[image/jpeg]` | E06-P1-016 | P1 | AC3 | RED PHASE |
| `test_upload_rejected_mime_types[application/x-msdownload]` | E06-P1-016 | P1 | AC3 | RED PHASE |
| `test_upload_requires_authentication` | (Implicit) | — | AC2 | RED PHASE |
| `test_upload_free_tier_blocked` | (Implicit) | — | AC2 | RED PHASE |
| `test_upload_presigned_url_and_confirm` | E06-P1-017 | P1 | AC5, AC6 | RED PHASE |
| `test_pending_document_not_downloadable_state` | E06-P0-006 | P0 | AC6 | RED PHASE |
| `test_infected_scan_marks_deleted` | E06-P0-007 | P0 | AC7 | RED PHASE |
| `test_clean_scan_marks_downloadable` | (Implicit) | — | AC7 | RED PHASE |
| `test_scan_result_requires_internal_key` | (Implicit) | — | AC7 | RED PHASE |
| `test_concurrent_package_size_race` | E06-P2-004 | P2 | AC4 | RED PHASE |

**Total tests:** 15 (11 named + 4 parametrized = 15 test cases)

---

## Step 5: Validation & Completion

### AC Coverage Map

| AC | Description | Tests Covering |
|----|-------------|----------------|
| AC1 | Alembic migration `019_documents` | Covered implicitly by all DB-asserting tests (table must exist) |
| AC2 | JWT auth + paid-tier requirement | `test_upload_requires_authentication`, `test_upload_free_tier_blocked` |
| AC3 | MIME type + per-file size validation (400) | `test_upload_allowed_mime_types[*]`, `test_upload_rejected_mime_types[*]`, `test_upload_rejects_oversized_file` |
| AC4 | Package size `SELECT FOR UPDATE` + 500MB limit | `test_concurrent_package_size_race` |
| AC5 | Presigned PUT URL, `expires_in=900`, doc record inserted | `test_upload_presigned_url_and_confirm` (Step 1) |
| AC6 | Confirm: ownership check, pending state, Celery dispatch | `test_upload_presigned_url_and_confirm` (Step 2+3), `test_pending_document_not_downloadable_state` |
| AC7 | scan-result: internal key auth, scan_status update, S3 soft-delete | `test_infected_scan_marks_deleted`, `test_clean_scan_marks_downloadable`, `test_scan_result_requires_internal_key` |
| AC8 | `dispatch_document_scan` Celery pattern | `test_upload_presigned_url_and_confirm` (mock dispatch called assert) |
| AC9 | OpenAPI schema | Not covered by integration tests — verified manually or via `/openapi.json` smoke test |
| AC10 | Integration tests in correct file | This checklist + generated test file |

### Risk Coverage Map

| Risk ID | Score | Covered By |
|---------|-------|-----------|
| E06-R-003 (ClamAV pre-scan gap) | 6 | `test_pending_document_not_downloadable_state`, `test_infected_scan_marks_deleted` |
| E06-R-006 (Package size race) | 4 | `test_concurrent_package_size_race` |
| E06-R-008 (ClamAV timeout) | 4 | Not directly covered — background timeout job out of scope for S06.06; `ix_documents_pending` index supports future job |

### Green Phase Checklist

When implementing S06.06, remove `@pytest.mark.skip` decorators **one group at a time**:

- [ ] **Group 1 — Validation (AC2, AC3):** Remove skip from `test_upload_requires_authentication`, `test_upload_free_tier_blocked`, `test_upload_rejects_oversized_file`, `test_upload_rejected_mime_types[*]`
  - Implement: migration, model, schemas, config, `request_upload` MIME+size validation, route handler with tier check
- [ ] **Group 2 — Upload flow (AC5):** Remove skip from `test_upload_allowed_mime_types[*]`, `test_upload_presigned_url_and_confirm`
  - Implement: S3 presigned URL generation, document record INSERT, `DocumentUploadResponse`
- [ ] **Group 3 — Confirm flow (AC6):** Remove skip from `test_pending_document_not_downloadable_state`
  - Implement: `confirm_upload` service, `dispatch_document_scan`, confirm route handler
- [ ] **Group 4 — Scan result (AC7):** Remove skip from `test_infected_scan_marks_deleted`, `test_clean_scan_marks_downloadable`, `test_scan_result_requires_internal_key`
  - Implement: `record_scan_result`, `verify_scan_callback_auth`, `_soft_delete_s3_object`, scan-result route handler
- [ ] **Group 5 — Package size race (AC4):** Remove skip from `test_concurrent_package_size_race`
  - Implement: `SELECT SUM(size) … FOR UPDATE` in `request_upload`; requires real PostgreSQL for locking semantics

### Known Limitations / Notes

1. **`test_concurrent_package_size_race`** requires a real PostgreSQL connection with proper transaction isolation. In CI environments backed by SQLite or in-memory fakes, this test will not correctly verify the `SELECT … FOR UPDATE` locking behaviour. Use `testcontainers-postgres` or the dev DB.

2. **S3 mock scope:** The `_mock_s3_client()` helper returns a fresh `MagicMock` per call. Tests that need to assert `put_object_tagging` or `delete_object` was called must hold a reference to the same mock instance (as done in `test_infected_scan_marks_deleted`).

3. **`application/x-zip-compressed`** is in `ALLOWED_MIME_TYPES` but not in `_ALLOWED_TYPES` for parametrize — added as the 5th allowed type in `schemas/documents.py`. The parametrize list covers the 4 most common types per AC10; `x-zip-compressed` is implicitly covered by the MIME constant test.

4. **OpenAPI schema (AC9)** is not verified by integration tests — it should be spot-checked manually or via a dedicated `/openapi.json` test that asserts the three document endpoint paths are present.

5. **S06.07 download gate (E06-P0-006 full coverage)** — the `test_pending_document_not_downloadable_state` test verifies DB state only. The HTTP-level 422 from the download endpoint is verified in `test_document_download.py` (S06.07 scope).

---

## Change Log

| Date | Actor | Change |
|------|-------|--------|
| 2026-04-17 | BMad ATDD (bmad-testarch-atdd) | Initial ATDD checklist generated for Story 6.6; test file written to `eusolicit-app/services/client-api/tests/api/test_document_upload.py`; 15 test cases covering E06-P0-006, P0-007, P1-016, P1-017, P2-004, P2-005 and implicit auth/tier/scan scenarios; all tests in RED phase with skip markers |
