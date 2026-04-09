---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-09'
workflowType: testarch-atdd
storyKey: 11-3-espd-auto-fill-agent-integration
detectedStack: backend
inputDocuments:
  - eusolicit-docs/implementation-artifacts/11-3-espd-auto-fill-agent-integration.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - _bmad/bmm/config.yaml
  - eusolicit-app/services/client-api/tests/api/test_espd_profile.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/src/client_api/services/espd_service.py
  - eusolicit-app/services/client-api/pyproject.toml
---

# ATDD Checklist — Epic 11, Story 11.3: ESPD Auto-Fill Agent Integration

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**Primary Test Level:** API / Integration (pytest + pytest-asyncio + httpx + respx)
**TDD Phase:** 🔴 RED — All tests skipped pending Story 11.3 implementation

---

## Story Summary

Story 11.3 extends the ESPD Profile API (S11.02) with two new agent-backed endpoints:

1. **`POST /api/v1/espd-profiles/{profile_id}/auto-fill`** — triggers the `espd-auto-fill`
   agent via the internal AI Gateway (E04), creates a named snapshot profile with the
   pre-filled data, and returns a diff of changed ESPD Part keys.
2. **`POST /api/v1/espd-profiles/{profile_id}/export`** — exports an ESPD profile as
   well-formed XML (`format=xml`) or formatted PDF (`format=pdf`) via StreamingResponse.

Both endpoints share the S11.02 company-scoped RLS enforcement (`get_espd_profile()` helper)
and the `require_role("bid_manager")` authorization guard.

**As a** company admin or bid manager,
**I want** to trigger AI pre-fill on a saved ESPD profile and export it as XML or PDF,
**So that** I can receive a ready-to-submit ESPD document for review and download.

---

## Acceptance Criteria Coverage

| AC | Description | Test Class | Tests | Priority |
|----|-------------|------------|-------|----------|
| AC1 | `POST /auto-fill` → 200; `snapshot_profile_id`, `source_profile_id`, `opportunity_id`, `espd_data`, `changed_fields` in response | `TestAC1AutoFillHappyPath` | 1 | P0 |
| AC2 | Snapshot profile created with name "Auto-fill snapshot: …"; original profile unchanged | `TestAC1AutoFillHappyPath` | 2 | P1 |
| AC3 | Gateway payload includes `espd_data`, `opportunity_id`, `company_id`; `X-Caller-Service: client-api` header on every request | `TestAC1AutoFillHappyPath`, `TestAC3GatewayPayload` | 3 | P1 |
| AC4 | Gateway timeout / 5xx → HTTP 503 `AGENT_UNAVAILABLE`; no snapshot created on failure | `TestAC4AgentErrorHandling` | 4 | P0 |
| AC5 | `POST /export` validates `format=xml\|pdf`; missing/invalid format → 422; correct Content-Type + Content-Disposition | `TestAC5ExportXml`, `TestAC7ExportPdf`, `TestExportFormatValidation` | 6 | P1 |
| AC6 | XML export: `<ESPDResponse xmlns="urn:X-eusolicit:espd:schema:v1">`; Parts mapped as child elements; absent Parts omitted; well-formed | `TestAC5ExportXml` | 5 | P0 |
| AC7 | PDF export: `reportlab`; non-empty body starting with `%PDF-`; correct Content-Disposition | `TestAC7ExportPdf` | 3 | P1 |
| AC8 | Company-scoped RLS on both endpoints; cross-company → 404 (not 403) | `TestAC8CrossCompanyRLS` | 2 | P0 |
| AC9 | Unauthenticated → 401; contributor/reviewer/read_only → 403 | `TestAC9Authorization` | 8 | P0 |
| AC10 | Empty `espd_data` (`{}`) → 422 with "no data to auto-fill" detail | `TestAC10EmptyEspdValidation` | 1 | P1 |

---

## Epic Test ID Mapping

| Epic Test ID | Description | Test(s) | Status |
|-------------|-------------|---------|--------|
| **E11-P0-001** | ESPD company RLS: cross-company access → 404 | `test_autofill_cross_company_returns_404`, `test_export_cross_company_returns_404` | 🔴 RED |
| **E11-P0-002** | ESPD Auto-Fill timeout → structured 503 | `test_autofill_gateway_timeout_returns_503` | 🔴 RED |
| **E11-P0-003** | ESPD XML export: well-formed, correct namespace, Parts present | `test_export_xml_is_well_formed`, `test_export_xml_root_has_correct_namespace`, `test_export_xml_contains_all_parts_present_in_espd_data` | 🔴 RED |
| **E11-P0-008** | All agent endpoints return `AGENT_UNAVAILABLE` on gateway 5xx | `test_autofill_gateway_500_returns_503_not_500`, `test_autofill_gateway_503_returns_503_with_standard_body` | 🔴 RED |
| **E11-P1-008** | ESPD auto-fill: snapshot accessible via GET after auto-fill | `test_autofill_creates_new_snapshot_profile_in_db` | 🔴 RED |
| **E11-P1-009** | ESPD empty espd_data validation → 422 before agent call | `test_autofill_empty_espd_data_returns_422` | 🔴 RED |
| **E11-P2-004** | ESPD cross-company: Company A cannot export Company B profile → 404 | `test_export_cross_company_returns_404` | 🔴 RED |

---

## Test File

**File:** `eusolicit-app/services/client-api/tests/api/test_espd_autofill_export.py`

**Total test methods:** 27 (31 with `@pytest.mark.parametrize` expansion for AC9)

---

## Test Strategy

### Stack Detection

- **Detected stack:** `backend`
- **Test framework:** pytest + pytest-asyncio + httpx (matches S11.02 pattern)
- **HTTP mock:** `respx>=0.21` for intercepting `httpx.AsyncClient` calls to AI Gateway
- **No browser-based tests** — all scenarios exercisable at API level

### Test Levels Applied

| Level | Use | Tests |
|-------|-----|-------|
| **API/Integration** | All tests — endpoint validation, RLS enforcement, agent mock | 27 methods |
| **Unit** | Not needed — service functions tested indirectly via endpoints |
| **E2E** | Not in scope for S11.03 backend story; covered by E11-P3-002 (staging) |

### AI Gateway Mocking Strategy

The `AiGatewayClient.run_agent()` method creates a fresh `httpx.AsyncClient` per call
(per-call client pattern, see Dev Notes). Tests use `respx.mock` as a context manager
which intercepts all `httpx.AsyncClient` requests within the block, regardless of
when the client is instantiated.

The `aigw_env_setup` session-scoped fixture sets `CLIENT_API_AIGW_BASE_URL` to
`http://test-aigw.local:8000` before any test runs, ensuring the singleton
`AiGatewayClient` (cached via `@functools.lru_cache`) uses the interceptable test URL.

**URL intercepted:** `http://test-aigw.local:8000/agents/espd-auto-fill/run`

### Key Test Design Decisions

1. **Fixture extends S11.02 pattern**: `autofill_client_and_session` follows
   `espd_client_and_session` exactly, adding ESPD profile creation in fixture setup.
   This avoids per-test profile setup boilerplate.

2. **Cross-company fixture**: `autofill_two_companies` creates two independent companies
   in one shared session, yielding `(token_a, profile_a_id, token_b)` for RLS tests.

3. **No snapshot created on failure (AC4)**: Test counts profiles before/after failed
   auto-fill using `GET /api/v1/espd-profiles` (count comparison pattern).

4. **respx request inspection (AC3)**: `route.calls[0].request.content` exposes the
   raw request body for JSON parsing and header inspection.

5. **XML namespace verification (AC6)**: ET stores Clark notation `{namespace}LocalName`.
   Tests check `"urn:X-eusolicit:espd:schema:v1" in root.tag`.

6. **PDF magic bytes (AC7)**: `response.content[:5] == b"%PDF-"` is the canonical
   Python test for valid PDF output.

---

## TDD Red Phase Checklist

### Prerequisites

- [x] Story approved with clear acceptance criteria (10 ACs defined)
- [x] `conftest.py` at `services/client-api/tests/conftest.py` verified and loaded
- [x] `test_espd_profile.py` (S11.02) pattern verified and followed
- [x] `get_espd_profile()` helper exists in `espd_service.py` (S11.02 dependency)
- [ ] **`respx>=0.21`** added to `[project.optional-dependencies] dev` in `pyproject.toml` (Task 6.4)
- [ ] **`reportlab>=4.0`** added to `[project.dependencies]` in `pyproject.toml` (Task 6.1)

### Files Generated

| File | Status | Notes |
|------|--------|-------|
| `eusolicit-app/services/client-api/tests/api/test_espd_autofill_export.py` | ✅ Created | 27 test methods, all `@pytest.mark.skip` |
| `eusolicit-docs/test-artifacts/atdd-checklist-11-3-espd-auto-fill-agent-integration.md` | ✅ Created | This file |

### Files NOT Created (RED PHASE)

These files will be created during Story 11.3 implementation (not by ATDD):

| File | Created By | Task |
|------|-----------|------|
| `services/client-api/src/client_api/services/ai_gateway_client.py` | Dev | Task 1 |
| `services/client-api/src/client_api/schemas/espd.py` (additions) | Dev | Task 3 |
| `services/client-api/src/client_api/services/espd_service.py` (additions) | Dev | Task 4 |
| `services/client-api/src/client_api/api/v1/espd.py` (additions) | Dev | Task 5 |
| `services/client-api/src/client_api/config.py` (additions) | Dev | Task 2 |

---

## Test Summary

### By Class

| Class | AC Coverage | Methods | Phase |
|-------|------------|---------|-------|
| `TestAC1AutoFillHappyPath` | AC1, AC2, AC3 (partial), E11-P1-008 | 4 | 🔴 RED |
| `TestAC4AgentErrorHandling` | AC4, E11-P0-002, E11-P0-008 | 4 | 🔴 RED |
| `TestAC5ExportXml` | AC5, AC6, E11-P0-003 | 5 | 🔴 RED |
| `TestAC7ExportPdf` | AC7 | 3 | 🔴 RED |
| `TestAC8CrossCompanyRLS` | AC8, E11-P0-001, E11-P2-004 | 2 | 🔴 RED |
| `TestAC9Authorization` | AC9 | 4 methods (8 with parametrize) | 🔴 RED |
| `TestAC10EmptyEspdValidation` | AC10, E11-P1-009 | 1 | 🔴 RED |
| `TestExportFormatValidation` | AC5 (format validation) | 2 | 🔴 RED |
| `TestAC3GatewayPayload` | AC3 (header + payload) | 2 | 🔴 RED |
| **Total** | **AC1–AC10 (full)** | **27 methods / 31 expanded** | 🔴 **ALL RED** |

### By Priority

| Priority | Count | Epic IDs |
|----------|-------|---------|
| P0 (critical) | 12 | E11-P0-001, E11-P0-002, E11-P0-003, E11-P0-008 |
| P1 (high) | 13 | E11-P1-008, E11-P1-009 |
| P2 (medium) | 2 | E11-P2-004 |
| P3 (low) | 0 | — (covered by E11-P3-002 E2E journey in staging) |

---

## Green Phase Instructions

After Story 11.3 implementation is complete:

### Step 1 — Install dependencies
```bash
cd eusolicit-app/services/client-api
pip install -e ".[dev]"   # includes respx>=0.21
pip install reportlab>=4.0
```

### Step 2 — Remove RED PHASE markers
Remove `@pytest.mark.skip(reason="🔴 RED PHASE: Story 11.3 not yet implemented")` from
every test method in `test_espd_autofill_export.py`.

Also remove the `@pytest.importorskip("respx", ...)` call at the module top and replace
with a direct `import respx` (once Task 6.4 confirms the dependency is installed).

### Step 3 — Run tests
```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_espd_autofill_export.py -v --asyncio-mode=auto
```

### Step 4 — Expected GREEN results
All 27+ tests should pass. Typical failure modes to investigate:

| Symptom | Likely Cause |
|---------|-------------|
| `respx.exceptions.AllMockedError` | `get_ai_gateway_client` not using `AIGW_TEST_BASE_URL` — check `aigw_env_setup` fixture and LRU cache clearing |
| `ImportError: ai_gateway_client` | Task 1 not complete (`ai_gateway_client.py` not created) |
| `404` on auto-fill endpoint | Task 5.1 endpoint not registered or `espd_router` prefix wrong |
| `422` on valid auto-fill request | `ESPDAutoFillRequest` schema not imported or `opportunity_id` field type mismatch |
| `500` instead of `503` on gateway error | `AiGatewayUnavailableError`/`AiGatewayTimeoutError` not caught in service |
| PDF body doesn't start with `%PDF-` | `reportlab` not installed or `generate_espd_pdf` returns empty bytes |
| XML namespace assertion fails | `ET.register_namespace("", ESPD_NS)` not called before `ET.tostring()` |
| Snapshot profile not in DB | `session.flush()` missing after `session.add(snapshot)` in service |

### Step 5 — Verify epic test ID coverage
```bash
pytest tests/api/test_espd_autofill_export.py -v -k "cross_company or timeout or well_formed or namespace"
```
All E11-P0-00{1,2,3,8} scenarios must be GREEN before sprint gate passes.

---

## Risks and Assumptions

| Risk | Impact | Mitigation |
|------|--------|-----------|
| `respx` intercepts `httpx.AsyncClient` created inside ASGI call stack | Could fail if respx context variable does not propagate through `ASGITransport` | respx uses `contextvars` which propagate through async tasks in the same event loop; `ASGITransport` runs in the same task — should work. Validate with one test first. |
| `get_ai_gateway_client` LRU cache survives between test sessions | Tests could use wrong base URL | `aigw_env_setup` clears the cache at session start; if singleton was created before fixture runs, re-run with `--forked` or add `autouse=True` fixture ordering |
| `CLIENT_API_AIGW_BASE_URL` env var not read by `get_settings()` | `AiGatewayClient` uses default `http://ai-gateway:8000` | `aigw_env_setup` also calls `get_settings.cache_clear()` to force re-read |
| `reportlab` binary output varies by version | PDF magic bytes `%PDF-` test could be fragile | `%PDF-` is mandated by ISO 32000; test is version-stable |
| XML namespace prefix style (`ns0:tag` vs default namespace) | Root tag assertion could fail if namespace prefix not registered | Dev Notes specify `ET.register_namespace("", ESPD_NS)` — test checks `root.tag` for namespace URI substring, not prefix style |

---

## Next Steps

1. **Developer:** Implement Story 11.3 following tasks 1–6 in the story file
2. **QA:** Run `pytest tests/api/test_espd_autofill_export.py` after removing skip markers
3. **QA:** Confirm E11-P0-003 (XML XSD conformance) via separate weekly XSD validation job
4. **QA:** Trigger E11-P3-002 E2E journey (ESPD auto-fill → preview → XML export) in staging
5. **QA:** Cross-reference against `atdd-checklist-11-2-espd-profile-crud-api.md` for
   regressions in the base CRUD API (no changes expected — S11.03 adds endpoints, not modifies)
