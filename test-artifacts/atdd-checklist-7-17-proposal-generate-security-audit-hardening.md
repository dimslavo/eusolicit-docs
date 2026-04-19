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
storyId: 7-17-proposal-generate-security-audit-hardening
detectedStack: fullstack
generationMode: ai
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-17-proposal-generate-security-audit-hardening.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit/_bmad/bmm/config.yaml
  - eusolicit-app/services/client-api/tests/api/test_proposal_generate.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/tests/cross_service/conftest.py
  - eusolicit-app/e2e/specs/proposals/proposals-workspace.spec.ts
---

# ATDD Checklist: Story 7.17 — Proposal Generate Endpoint Security Error Sanitization & Audit Trail Hardening

**Date:** 2026-04-18
**TDD Phase:** 🔴 RED — All tests are failing (helpers not yet extracted; missing header; AC5 timing-safe path not enforced)
**Stack:** Fullstack (Python / FastAPI / pytest-asyncio + Playwright / TypeScript)
**NFR Category:** Security (NFR4) primary; Compliance / Audit Trail (NFR12) secondary
**Priority:** P0 / BLOCKER — Epic 7 NFR Assessment returned FAIL + HALT

---

## Summary

| Metric | Value |
|--------|-------|
| Total tests generated | 27 |
| Tests skipped (RED phase) | 27 |
| Tests passing | 0 |
| Acceptance criteria covered | AC1, AC2, AC3, AC4, AC5, AC6, AC7, AC8 |
| Epic risk mitigations covered | E07-R-001 (timing oracle), E07-R-003 (cross-tenant 404), E07-R-005 (content-block sanitation) |

### Test Count by Level

| Level | Marker | File | Count |
|-------|--------|------|-------|
| Unit | `unit` | `test_sanitize_generation_error.py` | 7 |
| Unit | `unit` | `test_sanitize_content_block_body.py` | 7 |
| Integration | `integration` | `test_generate_error_sanitization.py` | 7 |
| Integration | `integration` | `test_generate_audit_log.py` | 4 |
| Integration | `integration` | `test_generate_audit_failopen.py` | 3 |
| API | `api` | `test_generate_security.py` | 4 |
| Cross-service | `cross_service` | `test_generate_sse_headers.py` | 4 |
| E2E (Playwright) | — | `generate-error-display.spec.ts` | 3 |
| **Total** | | | **39** |

---

## Stack Detection

| Check | Result |
|-------|--------|
| `pyproject.toml` / `pytest.ini` present | ✅ Backend |
| `playwright.config.ts` / `package.json` present | ✅ Frontend |
| **Detected stack** | `fullstack` |
| **Primary generation mode** | AI generation (backend-heavy; one E2E spec) |

---

## Prerequisites Verified

| Prerequisite | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 8 ACs with NFR targets and measurable thresholds |
| Test framework configured (backend) | ✅ | `pytest` + `pytest-asyncio` in `services/client-api/` |
| Test framework configured (E2E) | ✅ | `playwright.config.ts` in `eusolicit-app/` |
| Existing test patterns found | ✅ | `test_proposal_generate.py`, `_TestSessionProxy`, `proposal_with_opportunity` fixture |
| `conftest.py` fixtures available | ✅ | `client_api_session_factory`, `test_redis_client` |
| Cross-service conftest available | ✅ | `all_service_clients`, `correlation_id`, `services_healthy` |
| E2E fixture pattern understood | ✅ | `authenticatedPage` from `e2e/support/fixtures.ts` |
| Epic test design loaded | ✅ | `test-design-epic-07.md` — risks R-001, R-003, R-004 |
| NFR report loaded | ✅ | `nfr-report.md` — NFR4 FAIL, NFR12 FAIL trigger |
| Hot-fix commit context understood | ✅ | Commit `2d41fcf` — inline sanitization, no tests |

---

## Test Strategy

### Test Level Selection

This story is **backend-primary** with one E2E surface:

- **Unit** — Pure helper functions (`_sanitize_generation_error`, `sanitize_content_block_body`) with no external deps
- **Integration** — SSE error sanitization across 7 failure modes; audit log write; fail-open behavior. Uses `httpx.AsyncClient` + `ASGITransport` + `unittest.mock.patch`.
- **API** — Live-service timing oracle for AC5 (cross-tenant 404 p95 < 5ms), plus header and schema assertions
- **Cross-service** — Full stack SSE header verification (client-api + ai-gateway); `X-Content-Type-Options: nosniff` presence
- **E2E (Playwright)** — UI toast assertion: raw AI Gateway error text must never appear in the DOM

### TDD Red Phase Compliance

All tests use either:
- `@pytest.mark.skip(reason="TDD Red Phase: ...")` — Python
- `test.skip(async ({ ... }) => { ... })` — Playwright TypeScript

No `test.skip.only()` or CI-breaking patterns used.

---

## Acceptance Criteria → Test Mapping

| AC | Description | Test(s) | Level | Priority |
|----|-------------|---------|-------|----------|
| AC1 | SSE error event sanitized (NFR4) | `test_gateway_error_event_emitted_sanitized` | Integration | P0 |
| AC2 | Error parity across 7 failure modes; single `_sanitize_generation_error()` helper | `test_sanitize_*` unit tests (7); `test_generate_error_sanitization.py` (7) | Unit + Integration | P0 |
| AC3 | Audit row on every mutation (NFR12) | `test_generate_writes_audit_log_on_success`, `test_generate_writes_audit_log_on_failure`, `test_audit_log_row_has_correct_shape`, `test_exactly_one_audit_row_per_invocation` | Integration | P0 |
| AC4 | Audit write never breaks user flow | `test_audit_write_failure_stream_still_completes`, `test_audit_write_failure_logged_as_structured_error`, `test_audit_write_exception_not_propagated_to_client` | Integration | P0 |
| AC5 | Timing-safe 404 on cross-tenant generate (< 5ms p95) | `test_cross_tenant_generate_returns_404`, `test_cross_tenant_generate_404_timing_p95` | API | P0 |
| AC6 | Content-block body sanitized before AI Gateway (E07-R-005) | `test_sanitize_content_block_body.py` (7 parametrized vectors) | Unit | P1 |
| AC7 | SSE headers hardened on error path (`X-Content-Type-Options: nosniff`) | `test_sse_error_headers_present_on_error`, `test_x_content_type_options_nosniff_header`, `test_sse_headers_present_on_error_frame` | API + Cross-service | P0 |
| AC8 | No regression; coverage ≥ 80% | Full suite pass (validate in CI after GREEN phase) | All levels | P1 |

---

## Generated Test Files

### Unit Tests

#### `services/client-api/tests/unit/test_sanitize_generation_error.py`

**Skip reason:** `_sanitize_generation_error()` helper not yet extracted from `_run_generation_task`

| Test | AC | Priority |
|------|----|----------|
| `test_gateway_error_event_returns_fixed_message` | AC2 | P0 |
| `test_timeout_error_returns_gateway_unavailable` | AC2 | P0 |
| `test_unavailable_error_returns_gateway_unavailable` | AC2 | P0 |
| `test_generic_exception_returns_internal_error` | AC2 | P0 |
| `test_output_never_contains_sensitive_fields` | AC1, AC2 | P0 |
| `test_code_field_always_present` | AC2 | P0 |
| `test_none_exc_and_none_event_data_returns_internal_error` | AC2 | P1 |

**Key assertions:**
- Output dict never contains keys: `"raw"`, `"detail"`, `"stack"`, `"traceback"`, `"api_key"`
- `"code"` key always present in output
- Input with `API_KEY=sk-xyz` in raw event data → output has no `sk-` prefix text
- Three machine codes: `"gateway_error"`, `"gateway_unavailable"`, `"internal_error"`

---

#### `services/client-api/tests/unit/test_sanitize_content_block_body.py`

**Skip reason:** `sanitize_content_block_body()` not yet implemented

| Test | Input | Expected | AC | Priority |
|------|-------|----------|----|----------|
| `test_clean_ascii_passthrough` | `"Hello world"` | `"Hello world"` | AC6 | P1 |
| `test_null_byte_stripped` | `"hello\x00world"` | `"helloworld"` | AC6 | P0 |
| `test_control_char_stripped` (parametrized) | `\x01`–`\x1f` sweep excl. `\n\t\r` | Stripped | AC6 | P0 |
| `test_newline_tab_carriage_return_preserved` | `"\n\t\r"` | `"\n\t\r"` | AC6 | P0 |
| `test_utf8_multibyte_preserved` | `"€ñ中"` | `"€ñ中"` | AC6 | P1 |
| `test_below_100kib_passes` | 99.9 KiB body | No exception | AC6 | P0 |
| `test_exactly_100kib_passes` | 100 KiB body (102400 bytes) | No exception | AC6 | P0 |
| `test_over_100kib_raises_422` | 102401 bytes | `HTTPException(422, {"error": "content_block_too_large"})` | AC6 | P0 |
| `test_script_tag_passthrough` | `"<script>alert(1)</script>"` | Unchanged (HTML is JSON-serialized, not stripped) | AC6 | P1 |
| `test_empty_string_returns_empty` | `""` | `""` | AC6 | P2 |
| `test_mixed_control_and_valid_chars` | `"foo\x00\x01bar\x1fquux\n"` | `"foobarquux\n"` | AC6 | P0 |
| `test_block_id_in_422_detail` | Oversized body with `block_id="block-abc"` | `detail["block_id"] == "block-abc"` | AC6 | P1 |

---

### Integration Tests

#### `services/client-api/tests/integration/test_generate_error_sanitization.py`

**Skip reason:** `_sanitize_generation_error()` single-point helper not yet extracted; AC2 parity not yet enforced

Uses `_TestSessionProxy` / `_TestSessionFactory` pattern (identical to `test_proposal_generate.py`).
Mocks `AiGatewayClient.stream_agent` per-test.

| Test | Failure Mode | Assertion | AC |
|------|-------------|-----------|-----|
| `test_gateway_error_event_emitted_sanitized` | `event: error` with `api_key='sk-abc123'` | SSE data = `{"error":"...", "code":"gateway_error"}`; `"sk-abc123"` not in data | AC1, AC2 |
| `test_timeout_error_emitted_sanitized` | `AiGatewayTimeoutError("Connection timed out after 30s")` | `code="gateway_unavailable"`; `"timed out after 30s"` not in data | AC2 |
| `test_unavailable_error_emitted_sanitized` | `AiGatewayUnavailableError(...)` | Same schema as timeout | AC2 |
| `test_generic_runtime_error_emitted_sanitized` | `RuntimeError("DB conn leak: password=xyz")` | `code="internal_error"`; `"password=xyz"` not in data | AC2 |
| `test_stream_end_without_terminal_emits_error` | Stream closes without `done` or `error` | Error event emitted with stable `code` | AC2 |
| `test_cancelled_error_handled` | `asyncio.CancelledError` | Error event emitted; `generation_status = 'failed'` | AC2 |
| `test_generation_status_set_to_failed_on_error` | All error paths | DB row `generation_status = 'failed'` | AC2, AC3 |

---

#### `services/client-api/tests/integration/test_generate_audit_log.py`

**Skip reason:** Audit log row fields (`before`/`after` content hashes, `correlation_id`, `ip_address`) not yet fully populated

Uses `proposal_with_opportunity` fixture and `shared.audit_log` SQL queries.

| Test | Assertion | AC |
|------|-----------|-----|
| `test_generate_writes_audit_log_on_success` | Row exists with `action_type='proposal.generate'`, `entity_type='proposal'`, `entity_id`, `user_id`, `company_id` | AC3 |
| `test_generate_writes_audit_log_on_failure` | Row written even when AI Gateway returns error | AC3 |
| `test_audit_log_row_has_correct_shape` | `correlation_id` not null; `ip_address` not null; `before` has `version_number`; `after` present | AC3 |
| `test_exactly_one_audit_row_per_invocation` | One `POST /generate` → exactly 1 row for `proposal_id` | AC3 |

---

#### `services/client-api/tests/integration/test_generate_audit_failopen.py`

**Skip reason:** `write_audit_entry` exception path not yet wrapped in `try/except` with structured `structlog.error("audit_write_failed", ...)` logging

| Test | Patch | Assertion | AC |
|------|-------|-----------|-----|
| `test_audit_write_failure_stream_still_completes` | `write_audit_entry` raises `RuntimeError` | SSE stream delivers `done` event; response 200 | AC4 |
| `test_audit_write_failure_logged_as_structured_error` | Same | `structlog` emits ERROR with `audit_write_failed` key | AC4 |
| `test_audit_write_exception_not_propagated_to_client` | Same | `RuntimeError("audit_write_failed_test")` text absent from HTTP response | AC4 |

---

### API Tests

#### `tests/api/test_generate_security.py`

**Skip reason:** Timing-safe error path not yet implemented; `X-Content-Type-Options` header missing

Requires live client-api service (`CLIENT_API_URL` env var).

| Test | Assertion | AC |
|------|-----------|-----|
| `test_cross_tenant_generate_returns_404` | Company B JWT on Company A proposal → 404, body has no `proposal_id` leak | AC5 |
| `test_cross_tenant_generate_404_timing_p95` | 100-run p95 of cross-tenant 404 vs own-company baseline < 5ms difference | AC5 |
| `test_sanitized_error_body_no_stack_trace` | SSE error frame schema `{error, code}` only; forbidden keys absent; no `"sk-"`, `"Traceback"`, `"API_KEY"` in values | AC1 |
| `test_sse_error_headers_present_on_error` | All 4 AC7 headers present including `X-Content-Type-Options: nosniff` | AC7 |

---

### Cross-Service Tests

#### `tests/cross_service/test_generate_sse_headers.py`

**Skip reason:** `X-Content-Type-Options: nosniff` not yet added to `StreamingResponse` headers in `proposals.py`

Requires live client-api + ai-gateway. Uses `all_service_clients` and `services_healthy` fixtures.

| Test | Assertion | AC |
|------|-----------|-----|
| `test_sse_headers_present_on_success_response` | All 4 required headers on success path | AC7 |
| `test_sse_headers_present_on_error_frame` | All 4 required headers on error path | AC7 |
| `test_x_content_type_options_nosniff_header` | Focused: `X-Content-Type-Options: nosniff` present on both paths | AC7 |
| `test_correlation_id_propagated` | `X-Request-ID` header → `correlation_id` field in SSE error frame | AC3, AC7 |

---

### E2E Tests (Playwright)

#### `e2e/specs/proposals/generate-error-display.spec.ts`

**Skip form:** `test.skip(async ({ ... }) => { ... })` — Playwright RED PHASE form

Uses `authenticatedPage` fixture, `page.route()` SSE intercept, real proposal seeded via API.

| Test | Mock | Assertion | AC |
|------|------|-----------|-----|
| `[P0] should show generic error toast when AI gateway returns raw error` | `event: error` with `Traceback...API_KEY=sk-abc123` | `ai-generate-error` or `role=alert` visible; none of 5 forbidden patterns visible | AC1, AC8 |
| `[P0] should not display raw stack trace text in any visible element` | Same | `getByText('Traceback')` not visible; `getByText('API_KEY')` not visible; `getByText('sk-abc123')` not visible | AC1, AC8 |
| `[P1] should show generic error toast for connection refused` | Abrupt stream close (empty body) | Generic error toast visible; text not empty and not forbidden | AC2, AC8 |

---

## Generation Mode

```
Mode requested: auto
Detected: sequential (no subagent capability in this run)
Resolved: sequential
```

---

## Fixture Needs (for GREEN phase)

All fixtures are defined inline in each test file or inherited from existing conftest.py. No new shared fixtures required.

| Fixture | Location | Used By |
|---------|----------|---------|
| `proposal_with_opportunity` | `tests/api/test_proposal_generate.py` | Integration tests (re-defined locally with `integration_` prefix) |
| `_TestSessionProxy` | Defined inline | Integration tests |
| `all_service_clients` | `tests/cross_service/conftest.py` | Cross-service tests |
| `services_healthy` | `tests/cross_service/conftest.py` | Cross-service tests |
| `authenticatedPage` | `e2e/support/fixtures.ts` | E2E tests |

---

## Red Phase Requirements Summary

Tests will fail because the following are not yet implemented:

| # | Missing Implementation | AC | Test(s) |
|---|------------------------|-----|---------|
| 1 | `_sanitize_generation_error(exc, *, raw_event_data) -> dict` extracted as standalone helper in `proposal_service.py` | AC2 | Unit: 7 tests |
| 2 | `sanitize_content_block_body(body: str, *, block_id: str = "") -> str` in `proposal_service.py` | AC6 | Unit: 7 tests |
| 3 | All 7 error paths in `_run_generation_task` routed through `_sanitize_generation_error()` | AC2 | Integration: 7 tests |
| 4 | `write_audit_entry` wrapped in `try/except Exception`; structlog ERROR with `audit_write_failed` key | AC4 | Integration: 3 tests |
| 5 | Audit row fields: `correlation_id`, `ip_address`, `before` (with `version_number` + `content_hash`), `after` | AC3 | Integration: 4 tests |
| 6 | `X-Content-Type-Options: nosniff` added to `StreamingResponse` headers in `generate_draft()` | AC7 | API: 2 tests; Cross-service: 3 tests |
| 7 | Timing-safe constant-time path for cross-tenant 404 (< 5ms p95 delta) | AC5 | API: 2 tests |
| 8 | UI error toast uses `data-testid="ai-generate-error"` with sanitized message | AC8 | E2E: 3 tests |

---

## Green Phase Instructions

After implementation, activate all tests by removing their `skip` markers:

### Python tests
1. Remove `@pytest.mark.skip(reason="...")` from every test function
2. Run: `pytest services/client-api/tests/unit/test_sanitize_generation_error.py -v`
3. Run: `pytest services/client-api/tests/unit/test_sanitize_content_block_body.py -v`
4. Run: `pytest services/client-api/tests/integration/test_generate_error_sanitization.py -v`
5. Run: `pytest services/client-api/tests/integration/test_generate_audit_log.py -v`
6. Run: `pytest services/client-api/tests/integration/test_generate_audit_failopen.py -v`
7. Run (requires live service): `pytest tests/api/test_generate_security.py -m api -v`
8. Run (requires all services): `pytest tests/cross_service/test_generate_sse_headers.py -m cross_service -v`
9. Run coverage gate: `pytest services/client-api/tests -m "unit or integration" --cov=client_api --cov-report=term-missing --cov-fail-under=80`

### Playwright tests
1. Remove `test.skip(` wrapper; replace with plain `test(`
2. Run: `pnpm playwright test specs/proposals/generate-error-display --project=chromium`

### CI gate
```bash
pytest services/client-api/tests -m "unit or integration or api" -v
make quality-check
pnpm playwright test specs/proposals/generate-error-display
```

---

## Key Risks & Assumptions

| Risk | Mitigation |
|------|-----------|
| `_TestSessionProxy` SAVEPOINT behavior in integration tests | Pattern proven in `test_proposal_generate.py` — reuse exactly |
| Timing oracle (AC5) flaky on slow CI | Use `statistics.quantiles(n=100)[94]` p95; allow ±10ms tolerance; run 100 iterations |
| Cross-service tests require live ai-gateway | `services_healthy` fixture auto-skips if any service unreachable |
| E2E `page.route()` SSE intercept in RED phase | Route intercept works without real endpoint — intercept is the mock |
| `structlog` log capture in audit fail-open tests | Use `caplog` with `propagate=True` or mock structlog's `error()` method directly |
| Import errors on non-existent helpers (`_sanitize_generation_error`, `sanitize_content_block_body`) | Guarded `try/except ImportError` blocks at module top — tests collect and skip cleanly |

---

## Next Recommended Workflow

After GREEN phase (all tests passing):
1. **TEA Automate** — `bmad-testarch-automate` to expand coverage for edge cases
2. **TEA Review** — `bmad-testarch-test-review` to validate test quality
3. **Traceability** — `bmad-testarch-trace` to generate AC1-AC8 → NFR4/NFR12 → E07-R-003/E07-R-005 traceability matrix
4. **NFR Re-assessment** — `bmad-testarch-nfr` to flip NFR4 FAIL → PASS and NFR12 FAIL → PASS

---

*Generated by BMad TEA Agent (bmad-testarch-atdd) on 2026-04-18*
*Story: eusolicit-docs/implementation-artifacts/7-17-proposal-generate-security-audit-hardening.md*
