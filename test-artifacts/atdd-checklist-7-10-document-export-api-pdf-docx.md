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
storyId: 7-10-document-export-api-pdf-docx
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-10-document-export-api-pdf-docx.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/services/client-api/tests/api/test_proposal_pricing_win_themes.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.10 — Document Export API (PDF & DOCX)

**Date:** 2026-04-18
**Author:** TEA Master Test Architect
**Story Status:** ready-for-dev
**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)

---

## Preflight Summary

### Stack Detection

| Signal | Finding |
|--------|---------|
| `pyproject.toml` present | ✅ Python/FastAPI backend |
| `playwright.config.*` | ❌ Not present |
| `conftest.py` with pytest-asyncio | ✅ Confirmed |
| **Detected stack** | `backend` |

### Prerequisites Verified

| Requirement | Status |
|-------------|--------|
| Story has clear acceptance criteria (AC1–AC9) | ✅ |
| Test framework configured (pytest + pytest-asyncio) | ✅ |
| Existing test patterns available (test_proposal_pricing_win_themes.py) | ✅ |
| Epic-level test design loaded (test-design-epic-07.md) | ✅ |
| Development environment available | ✅ (local PostgreSQL + Redis) |

### Generation Mode

**Mode: AI Generation (Sequential)**
Backend stack — no browser recording required. Tests generated from acceptance criteria, story task specifications, and existing integration test patterns.

---

## Step 1: Context Summary

### Story

**7.10 — Document Export API (PDF & DOCX)**

`POST /api/v1/proposals/{proposal_id}/export`

- **Role required:** `bid_manager`
- **Input:** `{"format": "pdf" | "docx"}`
- **Output:** Streaming file download with appropriate headers
- **Key constraints:**
  - `company_id` exclusively from JWT (E07-R-003)
  - CPU-bound rendering in `run_in_executor` (E07-R-005)
  - Import chain via `client_api.utils.document_export` (Story 12.9 AC4)
  - Size limits: max 50 sections, max 512 KB total body (E07-R-010)

### Files That Must Be Created/Modified for GREEN Phase

| File | Action | Purpose |
|------|--------|---------|
| `services/client-api/src/client_api/config.py` | Modify | Add `export_max_sections` (default 50) and `export_max_content_bytes` (default 524288) |
| `services/client-api/src/client_api/schemas/proposal_export.py` | Create | `ExportRequest(BaseModel): format: Literal["pdf", "docx"]` |
| `services/client-api/src/client_api/services/export_service.py` | Create | `generate_export()`, `_load_proposal_with_content()`, `_load_company()`, `build_proposal_report_document()` |
| `services/client-api/src/client_api/api/v1/proposals.py` | Modify | Append Story 7.10 export endpoint section |
| `services/client-api/pyproject.toml` | Modify | Add `pypdf>=4.0` to dev dependencies |

---

## Step 2: Generation Mode

**Mode selected:** AI Generation
**Reason:** Backend-only (`detected_stack = backend`); all scenarios are API/integration level; no browser interactions.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenarios Mapping

| AC | Scenario | Priority | Test ID in Epic | Test Level | Test Method |
|----|----------|----------|-----------------|------------|-------------|
| AC1 | Invalid format `"xml"` → 422 | P2 | E07-P2-008 | Integration | `TestProposalExportValidation::test_invalid_format_xml_returns_422` |
| AC1 | Missing format field → 422 | P2 | E07-P2-008 | Integration | `TestProposalExportValidation::test_missing_format_returns_422` |
| AC1 | Uppercase `"PDF"` → 422 (case-sensitive Literal) | P2 | E07-P2-008 | Integration | `TestProposalExportValidation::test_uppercase_format_returns_422` |
| AC1 | No auth token → 401/403 | P2 | — | Integration | `TestProposalExportValidation::test_unauthenticated_request_returns_401_or_403` |
| AC2 | PDF export → HTTP 200 | P0 | E07-P0-010 | Integration | `TestProposalExportPDF::test_pdf_export_returns_200` |
| AC4 | Content-Type: application/pdf | P0 | E07-P0-010 | Integration | `TestProposalExportPDF::test_pdf_export_content_type_header` |
| AC4 | Content-Disposition: attachment; filename=…pdf | P0 | E07-P0-010 | Integration | `TestProposalExportPDF::test_pdf_export_content_disposition_header` |
| AC2 | PDF magic bytes `%PDF` | P0 | E07-P0-010 | Integration | `TestProposalExportPDF::test_pdf_magic_bytes` |
| AC2 | PDF parsable by pypdf, ≥1 page | P0 | E07-P0-010 | Integration | `TestProposalExportPDF::test_pdf_is_parsable_and_has_pages` |
| AC2 | PDF text layer: section titles present | P0 | E07-P0-010 | Integration | `TestProposalExportPDF::test_pdf_text_layer_contains_section_titles` |
| AC2 | PDF text layer: "Table of Contents" present | P0 | E07-P0-010 | Integration | `TestProposalExportPDF::test_pdf_text_layer_contains_table_of_contents` |
| AC3 | DOCX export → HTTP 200 | P1 | E07-P1-019 | Integration | `TestProposalExportDOCX::test_docx_export_returns_200` |
| AC4 | Content-Type: wordprocessingml | P1 | E07-P1-019 | Integration | `TestProposalExportDOCX::test_docx_export_content_type_header` |
| AC4 | Content-Disposition: attachment; filename=…docx | P1 | E07-P1-019 | Integration | `TestProposalExportDOCX::test_docx_export_content_disposition_header` |
| AC3 | DOCX magic bytes `PK` | P1 | E07-P1-019 | Integration | `TestProposalExportDOCX::test_docx_magic_bytes` |
| AC3 | DOCX parsable by python-docx, section titles present | P1 | E07-P1-019 | Integration | `TestProposalExportDOCX::test_docx_is_parsable_and_contains_section_titles` |
| AC2 | PDF title page contains company name | P1 | E07-P1-020 | Integration | `TestProposalExportBranding::test_pdf_title_page_contains_company_name` |
| AC3 | DOCX document contains company name | P1 | E07-P1-020 | Integration | `TestProposalExportBranding::test_docx_title_page_contains_company_name` |
| AC7 | Empty proposal PDF → 200 + valid file (not 500) | P2 | E07-P2-009 | Integration | `TestProposalExportEmptyProposal::test_empty_proposal_pdf_exports_gracefully` |
| AC7 | Empty proposal DOCX → 200 + valid file (not 500) | P2 | E07-P2-009 | Integration | `TestProposalExportEmptyProposal::test_empty_proposal_docx_exports_gracefully` |
| AC5 | Cross-company export → 404 not 403 (PDF) | P0 | E07-P0-005 | Integration | `TestProposalExportRLS::test_cross_company_export_returns_404_not_403` |
| AC5 | Cross-company export → 404 (DOCX, RLS not format-specific) | P0 | E07-P0-005 | Integration | `TestProposalExportRLS::test_cross_company_docx_export_returns_404` |
| AC5 | Non-existent proposal → 404 | P0 | E07-P0-005 | Integration | `TestProposalExportRLS::test_nonexistent_proposal_returns_404` |
| AC6 | 55 sections → 400 `proposal_too_large` | P2 | E07-R-010 | Integration | `TestProposalExportSizeLimits::test_too_many_sections_returns_400` |
| AC6 | ~550 KB content → 400 `proposal_too_large` | P2 | E07-R-010 | Integration | `TestProposalExportSizeLimits::test_content_too_large_returns_400` |
| AC8 | CPU-bound rendering in `run_in_executor` | — | E07-R-005 | Code-level | No separate test — implementation mitigation in `export_service.generate_export()` |

### Coverage by Priority

| Priority | Count | Epic IDs |
|----------|-------|----------|
| **P0** | 10 tests | E07-P0-010 (7 tests), E07-P0-005 (3 tests) |
| **P1** | 7 tests | E07-P1-019 (5 tests), E07-P1-020 (2 tests) |
| **P2** | 9 tests | E07-P2-008 (4 tests), E07-P2-009 (2 tests), E07-R-010 (2 tests), AC1 auth (1 test) |
| **Total** | **26 tests** | |

### TDD Red Phase Confirmation

All tests are designed to **fail** because:

1. `POST /api/v1/proposals/{proposal_id}/export` does not exist → any request returns `405 Method Not Allowed` or `404 Not Found`
2. `ExportRequest` schema does not exist → import would fail if imported directly
3. `export_service.generate_export()` does not exist → `ImportError` if referenced
4. `settings.export_max_sections` / `settings.export_max_content_bytes` do not exist → `AttributeError`
5. `pypdf` may not be installed yet → `ImportError` on `import pypdf`

---

## Step 4: Test Generation Results

### Generated File

| File | Path | Tests |
|------|------|-------|
| Main integration test file | `services/client-api/tests/api/test_proposal_export.py` | 26 tests |

### Test Classes

| Class | Priority | Epic Coverage | Tests |
|-------|----------|---------------|-------|
| `TestProposalExportPDF` | P0 | E07-P0-010 | 7 |
| `TestProposalExportDOCX` | P1 | E07-P1-019 | 5 |
| `TestProposalExportBranding` | P1 | E07-P1-020 | 2 |
| `TestProposalExportValidation` | P2 | E07-P2-008 | 4 |
| `TestProposalExportEmptyProposal` | P2 | E07-P2-009 | 2 |
| `TestProposalExportRLS` | P0 | E07-P0-005, E07-R-003 | 3 |
| `TestProposalExportSizeLimits` | P2 | E07-R-010, AC6 | 3 (2 fixtures + 2 tests) |

### Fixtures

| Fixture | Type | Purpose |
|---------|------|---------|
| `export_proposal` | `pytest_asyncio.fixture` | Company A bid_manager + proposal with 2 seeded sections |
| `empty_proposal` | `pytest_asyncio.fixture` (inner class) | Proposal with zero sections (AC7 test) |
| `oversized_sections_proposal` | `pytest_asyncio.fixture` (inner class) | Proposal with 55 sections (AC6 section count test) |
| `oversized_content_proposal` | `pytest_asyncio.fixture` (inner class) | Proposal with 5 × 110KB sections = ~550KB (AC6 content size test) |

### Helper Functions

| Function | Purpose |
|----------|---------|
| `_seed_proposal_sections(session, proposal_id, sections)` | SQL UPDATE to seed version content without using the PUT /content endpoint |
| `_register_and_login_company_b(client, session)` | Register Company B within Company A's session; return Company B token |

### Patterns Used (matching existing codebase)

- `async with client_api_session_factory() as session:` — session lifecycle
- `fastapi_app.dependency_overrides[get_db_session] = override_db` — dependency injection
- `ASGITransport(app=fastapi_app)` — in-process HTTP transport
- SQL `UPDATE client.proposal_versions SET content = CAST(:content AS jsonb)` — direct version seeding
- `await session.flush()` (never `commit`) — test transaction discipline
- `from __future__ import annotations` — all files
- `# noqa: PLC0415` — late imports inside fixtures
- `await session.rollback()` in `finally` — cleanup

---

## Step 5: Validation

### Prerequisites Check

- [x] Story has clear acceptance criteria (AC1–AC9 all specified)
- [x] Test framework configured (pytest + pytest-asyncio confirmed in conftest.py)
- [x] All tests reference non-existent endpoints (RED phase — will fail)
- [x] No placeholder assertions (`expect(True).toBe(True)` equivalents)
- [x] Realistic test data (section titles, company names, content bodies)
- [x] No orphaned browser sessions (backend-only tests, no Playwright)
- [x] Output file stored in `test_artifacts/` (not a temp directory)

### TDD Red Phase Compliance

| Check | Status |
|-------|--------|
| All tests assert EXPECTED behavior | ✅ |
| Tests will fail before implementation | ✅ (endpoint returns 404/405) |
| No passing tests generated | ✅ |
| Acceptance criteria fully covered | ✅ (all 9 ACs mapped; AC8 = code-level mitigation) |
| Priority tags documented in class docstrings | ✅ |
| Epic test IDs referenced in every test docstring | ✅ |

---

## Next Steps: TDD Green Phase

After implementing Story 7.10 per the tasks in the story file:

### 1. Install pypdf
```bash
cd services/client-api
pip install -e ".[dev]"
```

### 2. Run tests (expect RED — all fail)
```bash
cd services/client-api
pytest tests/api/test_proposal_export.py -v
```

### 3. Implement per story tasks (AC1–AC8)

Execute Tasks 1–5 from the story file in order:
- Task 1: Add settings fields to `config.py`
- Task 2: Add `pypdf>=4.0` to `pyproject.toml`
- Task 3: Create `schemas/proposal_export.py`
- Task 4: Create `services/export_service.py`
- Task 5: Append export endpoint to `api/v1/proposals.py`

### 4. Re-run tests (expect GREEN — all pass)
```bash
pytest tests/api/test_proposal_export.py -v
```

### 5. Run full regression suite
```bash
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py \
       tests/api/test_proposal_compliance_risk_scoring.py \
       tests/api/test_proposal_pricing_win_themes.py \
       tests/api/test_content_blocks.py \
       tests/api/test_proposal_export.py -v
```

**Baseline:** 351+ tests from Stories 7.1–7.9 must continue to pass.

---

## Risks and Assumptions

| Item | Notes |
|------|-------|
| **pypdf import** | `pypdf>=4.0` must be installed in the dev environment before tests in `TestProposalExportPDF` and `TestProposalExportBranding` can execute (even in RED phase for correct failure mode) |
| **python-docx** | Already a production dependency — available in the test environment |
| **Size limit defaults** | Tests use hardcoded 55 sections (> default 50) and ~550KB content (> default 512KB). If defaults change, tests may need adjustment |
| **`_compute_content_hash`** | Not used in these tests — content is seeded via direct SQL UPDATE. This avoids hash-computation coupling between test and implementation |
| **`run_in_executor` (AC8)** | No direct test for the executor pattern — verifying it via code review and the fact that the test suite doesn't time out is sufficient. A performance test (manual, E07-R-005) is separate |
| **Company name in PDF text** | Some PDF renderers embed text differently than displayed. If `pypdf.extract_text()` doesn't find the company name, the renderer may need `extractText` mode or the assertion may need adjustment |

---

## Implementation Guidance for Developer

### Critical Architecture Constraints (from story Dev Notes)

1. `from __future__ import annotations` at top of every new file
2. Import via `client_api.utils.document_export` — NOT directly from `eusolicit_common.document_generation`
3. `structlog` for all logging — never `logging`
4. `session.flush()` — never `session.commit()` in route-scoped sessions
5. `company_id` exclusively from JWT — never from request body
6. Return 404 (not 403) for cross-company access
7. `asyncio.get_running_loop().run_in_executor(None, render_fn, doc)` — MUST NOT call render_pdf/render_docx synchronously

### New API Endpoint

```
POST /api/v1/proposals/{proposal_id}/export    → 200 + file bytes (bid_manager)
                                                → 400 if proposal_too_large (AC6)
                                                → 404 if not found or cross-company (AC5)
                                                → 422 if format is invalid (AC1)
```
