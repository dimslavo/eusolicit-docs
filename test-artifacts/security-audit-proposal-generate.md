# Security Audit Evidence — Story 7-17: Proposal Generate Endpoint Hardening

**Story:** 7-17 — Proposal Generate Endpoint: Security Error Sanitization & Audit Trail Hardening
**Artifact type:** Security audit evidence (NFR4 / NFR12)
**Generated:** 2026-04-18
**Status:** COMPLETE — all ATDD tests GREEN

---

## Summary

Story 7-17 implements two security hardening requirements for the `POST /api/v1/proposals/{id}/generate` SSE endpoint:

- **NFR4 (Security):** Single-point error sanitization preventing raw AI Gateway stack traces, API keys, and internal details from reaching clients
- **NFR12 (Compliance):** Complete audit trail rows written to `shared.audit_log` on every invocation with ip_address, correlation_id, company_id, and version snapshot

---

## Acceptance Criteria Coverage

| AC | Description | Status | Test(s) |
|----|-------------|--------|---------|
| AC1 | `POST /generate` returns HTTP 200 for SSE (not 500 on handled errors) | ✅ PASS | `test_gateway_error_event_emitted_sanitized`, `test_stream_end_without_terminal_emits_error` |
| AC2 | All error paths emit sanitized fixed-string error events (no raw exc details) | ✅ PASS | `test_gateway_error_event_emitted_sanitized`, `test_generic_runtime_error_emitted_sanitized`, `test_timeout_error_emitted_sanitized`, `test_unavailable_error_emitted_sanitized`, `test_cancelled_error_handled` |
| AC3 | `write_audit_entry` failure never propagates to client (fail-open) | ✅ PASS | `test_audit_write_failure_stream_still_completes`, `test_audit_write_exception_not_propagated_to_client` |
| AC4 | Structured ERROR log emitted on audit write failure | ✅ PASS | `test_audit_write_failure_logged_as_structured_error` |
| AC5 | `X-Content-Type-Options: nosniff` header on StreamingResponse | ✅ PASS | `test_sse_response_has_nosniff_header` |
| AC6 | `sanitize_content_block_body()` strips control chars, raises 422 on >100 KiB | ✅ PASS | `test_sanitize_content_block_body_*` (5 unit tests) |
| AC7 | `_sanitize_generation_error()` single-point sanitizer for all error paths | ✅ PASS | `test_sanitize_generation_error_*` (7 unit tests) |

---

## Test Results (2026-04-18)

### Unit Tests (27 passing)

```
services/client-api/tests/unit/test_sanitize_generation_error.py  — 7 passed
services/client-api/tests/unit/test_sanitize_content_block_body.py — 5 passed
(+ 15 passing in other unit modules)
```

### Integration Tests (14 passing)

```
services/client-api/tests/integration/test_generate_error_sanitization.py  — 7 passed
services/client-api/tests/integration/test_generate_audit_log.py           — 4 passed
services/client-api/tests/integration/test_generate_audit_failopen.py      — 3 passed
```

**Total Story 7-17 tests: 41 passing, 0 failing**

---

## Implementation Changes

### 1. `_sanitize_generation_error()` — Single sanitization point (NFR4)

**File:** `services/client-api/src/client_api/services/proposal_service.py`

Replaces 3 inline error construction sites with a single helper that:
- Maps `AiGatewayTimeoutError` / `AiGatewayUnavailableError` → `{"error": "AI Gateway unavailable or timed out", "code": "gateway_unavailable"}`
- Maps gateway `error` SSE events → `{"error": "AI Gateway reported a generation error", "code": "gateway_error"}`
- Maps all other `Exception` → `{"error": "An internal error occurred during generation", "code": "internal_error"}`
- Returns stable, fixed strings — no `str(exc)` or traceback forwarded to client

### 2. `asyncio.CancelledError` handling (AC2)

**File:** `services/client-api/src/client_api/services/proposal_service.py`

Added explicit `except asyncio.CancelledError` block before `except Exception` in `_run_generation_task`. In Python 3.8+, `CancelledError` is a `BaseException` (not `Exception`) and was silently falling through, leaving proposals stuck in `generating` status. Now:
- Sets `generation_status = 'failed'` before re-raising
- Emits sanitized error event to queue
- Re-raises so asyncio task is properly cancelled

### 3. `sanitize_content_block_body()` — Content block sanitization (AC6)

**File:** `services/client-api/src/client_api/services/proposal_service.py`

New helper that:
- Strips control characters in range `\x00–\x1F` except `\n`, `\t`, `\r` via regex `[\x00-\x08\x0b\x0c\x0e-\x1f]`
- Raises `HTTPException(422)` with `{"error": "content_block_too_large", "block_id": ...}` if body exceeds 100 KiB (102,400 bytes)
- Wired into `build_generation_payload()` so every content block is sanitized before being sent to AI Gateway

### 4. `X-Content-Type-Options: nosniff` header (AC5)

**File:** `services/client-api/src/client_api/api/v1/proposals.py`

Added to `StreamingResponse` `headers` dict alongside existing `Cache-Control: no-cache` and `X-Accel-Buffering: no`.

### 5. Comprehensive `write_audit_entry` call (NFR12)

**File:** `services/client-api/src/client_api/api/v1/proposals.py`

Replaced minimal audit call with full-fidelity write:
- `ip_address`: extracted from `X-Forwarded-For` header or `request.client.host`
- `correlation_id`: from `X-Request-ID` → `X-Correlation-ID` → `uuid4()` fallback
- `company_id`: from JWT `current_user.company_id`
- `before`: `{"version_number": <N>}` snapshot of pre-generation version
- Wrapped in `try/except` with `_audit_log.error("audit_write_failed", ...)` — fail-open (AC3, AC4)

### 6. Audit write fail-open (AC3, AC4)

```python
try:
    await write_audit_entry(session, user_id=current_user.user_id,
        action_type="proposal.generate", entity_type="proposal",
        entity_id=proposal_id, company_id=current_user.company_id,
        ip_address=_ip_address, correlation_id=_correlation_id, before=_before)
except Exception as _audit_exc:
    _audit_log.error("audit_write_failed", proposal_id=str(proposal_id),
                    correlation_id=_correlation_id, exc_type=type(_audit_exc).__name__)
```

### 7. Database migration 025 (NFR12)

**File:** `services/client-api/alembic/versions/025_audit_log_company_correlation.py`

Adds to `shared.audit_log`:
- `company_id UUID NULLABLE` — company scope of the event (no FK — cross-schema rule)
- `correlation_id VARCHAR(128) NULLABLE` — from `X-Request-ID` or generated

### 8. `AuditLog` ORM model + `write_audit_entry` signature

**Files:**
- `services/client-api/src/client_api/models/audit_log.py` — added two new `Mapped` fields
- `services/client-api/src/client_api/services/audit_service.py` — added `company_id` and `correlation_id` keyword-only params

### 9. Test infrastructure: structlog→stdlib routing (conftest)

**File:** `services/client-api/tests/conftest.py`

Added session-scoped autouse fixture `configure_structlog_stdlib` that reconfigures structlog to use `structlog.stdlib.LoggerFactory()` in tests. This ensures structlog `.error()` calls are captured by pytest's `caplog` fixture (AC4 test requirement). Production code continues to use `PrintLoggerFactory`.

---

## Security Properties Verified

| Property | Verification method |
|----------|---------------------|
| No raw exc messages in SSE stream | `test_generic_runtime_error_emitted_sanitized` asserts `"audit_write_failed_test"` not in SSE body |
| No API keys / stack traces in SSE stream | `test_gateway_error_event_emitted_sanitized` asserts sensitive fields absent from error event |
| Fixed-string error codes only | Unit tests verify exact `{"error": ..., "code": ...}` shape |
| Audit written on success AND failure | `test_generate_writes_audit_log_on_success`, `test_generate_writes_audit_log_on_failure` |
| Exactly 1 audit row per invocation | `test_exactly_one_audit_row_per_invocation` |
| Audit row shape: correlation_id, ip_address, before, company_id | `test_audit_log_row_has_correct_shape` |
| Audit failure does not degrade SSE | `test_audit_write_failure_stream_still_completes`, `test_audit_write_exception_not_propagated_to_client` |
| Control chars stripped from content blocks | `test_sanitize_content_block_body_strips_control_characters` |
| Oversized blocks rejected 422 | `test_sanitize_content_block_body_raises_422_on_oversized_body` |
| CancelledError sets status=failed | `test_cancelled_error_handled` |

---

## Deviations

**DEVIATION: CONTRADICTORY_SPEC** — Story 7-17 stated "No schema migration needed" but the ATDD tests required `company_id` and `correlation_id` columns on `shared.audit_log`. Migration 025 was created to resolve this. Deferred item "Error SSE event forwards raw AI Gateway data" in `deferred-work.md` is resolved by this story.
