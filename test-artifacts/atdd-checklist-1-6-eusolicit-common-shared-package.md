---
stepsCompleted:
  - 'step-01-preflight-and-context'
  - 'step-02-generation-mode'
  - 'step-03-test-strategy'
  - 'step-04-generate-tests'
  - 'step-04c-aggregate'
  - 'step-05-validate-and-complete'
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-6-eusolicit-common-shared-package'
detectedStack: 'backend'
generationMode: 'ai-generation'
executionMode: 'sequential'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-6-eusolicit-common-shared-package.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-app/pyproject.toml'
  - 'eusolicit-app/packages/eusolicit-common/pyproject.toml'
  - 'eusolicit-app/tests/conftest.py'
---

# ATDD Checklist: Story 1.6 — eusolicit-common Shared Package

**Date:** 2026-04-06
**Story:** 1-6-eusolicit-common-shared-package
**TDD Phase:** RED (failing tests — implementation pending)
**Stack:** Python backend (FastAPI + pytest + pytest-asyncio)

---

## Preflight Summary

| Item | Status |
|------|--------|
| Story approved with clear ACs | ✅ 9 acceptance criteria |
| Test framework configured | ✅ pytest + pytest-asyncio (asyncio_mode="auto") |
| Dev environment available | ✅ Monorepo with editable installs |
| Detected stack | `backend` (Python/FastAPI) |
| Generation mode | AI generation (no browser recording for backend) |
| Epic test design loaded | ✅ test-design-epic-01.md |
| TEA config flags | None configured (Playwright/Pact disabled — N/A for backend library) |

---

## TDD Red Phase Status

✅ **Failing tests generated**

| Test File | Test Count | Marker | AC Coverage | Priority |
|-----------|-----------|--------|-------------|----------|
| `tests/unit/test_eusolicit_common_schemas.py` | 16 | `@pytest.mark.skip` (ATDD RED) | AC 8 | P1 |
| `tests/unit/test_eusolicit_common_config.py` | 8 | `@pytest.mark.skip` (ATDD RED) | AC 1 | P1 |
| `tests/unit/test_eusolicit_common_exceptions.py` | 12 | `@pytest.mark.skip` (ATDD RED) | AC 7 | P1 |
| `tests/unit/test_eusolicit_common_logging.py` | 13 | `@pytest.mark.skip` (ATDD RED) | AC 2 | P2 |
| `tests/unit/test_eusolicit_common_health.py` | 14 | `@pytest.mark.skip` (ATDD RED) | AC 6 | P1 |
| `tests/unit/test_eusolicit_common_middleware.py` | 21 | `@pytest.mark.skip` (ATDD RED) | AC 3, 4, 5 | P2 |
| **Total** | **84** | | **AC 1–8** | |

**Note:** AC 9 (coverage target ≥80%) is a meta-criterion validated by running `pytest --cov` after implementation, not by a dedicated test file.

---

## Acceptance Criteria Coverage Matrix

| AC | Description | Test File | Scenarios | Priority |
|----|-------------|-----------|-----------|----------|
| **AC 1** | Config loader: env vars, .env fallback, Pydantic BaseSettings, env_prefix | `test_eusolicit_common_config.py` | Default values, env var reads, .env fallback, env_prefix overrides, extra vars ignored, invalid type validation | **P1** |
| **AC 2** | Logging: structlog JSON/pretty-print, correlation_id, service_name, tenant_id | `test_eusolicit_common_logging.py` | JSON renderer (production), ConsoleRenderer (dev), service_name in output, timestamp, log level filtering, context var helpers, correlation_id/tenant_id in logs | **P2** |
| **AC 3** | Auth middleware base: abstract JWT extraction, UserContext, exclude_paths | `test_eusolicit_common_middleware.py` | Valid token populates UserContext, missing auth → 401, invalid bearer → 401, invalid token → 401, excluded paths bypass, UserContext fields, sets tenant_id context var | **P2** |
| **AC 4** | Audit middleware: logs method/path/status/duration/user_id/tenant_id/correlation_id | `test_eusolicit_common_middleware.py` | Generates correlation_id, uses provided X-Request-ID, sets response header, logs all required fields, includes user_id, sets correlation_id context var | **P2** |
| **AC 5** | Rate limit middleware base: abstract with pluggable backend, per-route config | `test_eusolicit_common_middleware.py` | RateLimitResult fields, NoOpRateLimiter always allows, accepts rate_limits config, default empty config, reset_at=0 | **P2** |
| **AC 6** | Health check endpoint: GET /healthz, dependency checks, status logic | `test_eusolicit_common_health.py` | All pass → healthy, non-critical fail → degraded, critical fail → unhealthy, version included, uptime positive, latency measured, no checks → healthy, /healthz route, check factories | **P1** |
| **AC 7** | Exception handlers: domain exceptions → HTTP error envelope | `test_eusolicit_common_exceptions.py` | 7 exception classes with correct status codes, AppException hierarchy, details dict, error envelope with correlation_id, unhandled → 500 with no stack trace leak, register_exception_handlers | **P1** |
| **AC 8** | Pydantic base models: BaseSchema, PaginatedResponse[T], ErrorResponse, HealthResponse | `test_eusolicit_common_schemas.py` | camelCase serialization, snake_case deserialization, populate_by_name, from_attributes, round-trip, PaginatedResponse generic, ErrorResponse envelope, HealthResponse/HealthCheckResult | **P1** |
| **AC 9** | Unit tests ≥80% coverage | *(meta)* | Validated via `pytest --cov=packages/eusolicit-common` after implementation | **P1** |

---

## Risk Coverage

| Risk ID | Description | Score | Test Coverage |
|---------|-------------|-------|---------------|
| **E01-R-008** | eusolicit-common middleware contract misalignment | 4 (MEDIUM) | Auth, Audit, Rate Limit middleware interface tests verify abstract contracts. Exception handler tests verify error envelope contract. Health check tests verify HealthResponse contract. |

---

## Priority Distribution

| Priority | Count | Description |
|----------|-------|-------------|
| **P1** | 50 | Config loader, exception handlers, Pydantic models, health check |
| **P2** | 34 | Structured logging, auth/audit/rate-limit middleware |
| **Total** | **84** | |

---

## Test Strategy Notes

### Why Unit Tests Only

This story creates a **shared library** — not a service with API endpoints or UI. All tests are pure unit tests with mocked dependencies:

- `@pytest.mark.unit` marker on all tests
- No Docker Compose required
- No integration tests (no real DB/Redis connections)
- `unittest.mock.AsyncMock` for async dependencies
- `httpx.ASGITransport` + `httpx.AsyncClient` for middleware testing (in-process, no network)
- `FastAPI.TestClient` for synchronous endpoint verification

### Why No E2E/API Tests

- No browser UI → no E2E tests
- No service endpoints exposed → no API-level tests
- The health router (`/healthz`) is tested via `TestClient` at unit level
- Service integration deferred to E02+ stories

### Failure Mechanism

Tests fail via two mechanisms:
1. **Import-time failure**: `from eusolicit_common.config import ...` raises `ImportError` because the modules don't exist yet
2. **Skip marker**: `@pytest.mark.skip(reason="ATDD RED: ...")` ensures tests are skipped in CI until implementation begins

When the developer starts implementing:
1. Remove the `pytest.mark.skip` marker from the file under development
2. Run the tests — they should fail with `ImportError` initially
3. Implement the module — tests should start passing
4. Repeat for each module

---

## Files Generated

| File | Purpose | AC |
|------|---------|-----|
| `eusolicit-app/tests/unit/test_eusolicit_common_schemas.py` | BaseSchema, PaginatedResponse[T], ErrorResponse, HealthResponse tests | AC 8 |
| `eusolicit-app/tests/unit/test_eusolicit_common_config.py` | BaseServiceSettings, get_settings() tests | AC 1 |
| `eusolicit-app/tests/unit/test_eusolicit_common_exceptions.py` | Domain exception hierarchy + handler tests | AC 7 |
| `eusolicit-app/tests/unit/test_eusolicit_common_logging.py` | setup_logging(), get_logger(), context var tests | AC 2 |
| `eusolicit-app/tests/unit/test_eusolicit_common_health.py` | HealthCheck, create_health_router(), check factories | AC 6 |
| `eusolicit-app/tests/unit/test_eusolicit_common_middleware.py` | Auth, Audit, Rate Limit middleware tests | AC 3, 4, 5 |
| `eusolicit-docs/test-artifacts/atdd-checklist-1-6-eusolicit-common-shared-package.md` | This ATDD checklist | — |

---

## Next Steps (TDD Green Phase)

After implementing the feature modules:

1. **For each module**, remove the `pytest.mark.skip` line from the corresponding test file
2. Run tests: `cd eusolicit-app && python -m pytest tests/unit/test_eusolicit_common_<module>.py -v`
3. Verify tests PASS (green phase)
4. If any tests fail:
   - Fix implementation (feature bug) — tests define the contract
   - Or fix test (test bug) — only if the AC was misinterpreted
5. Run full coverage check: `python -m pytest --cov=packages/eusolicit-common --cov-report=term-missing tests/unit/test_eusolicit_common_*.py`
6. Verify ≥80% coverage (AC 9)
7. Run full regression: `make test` — verify no regressions in existing 1370+ tests

### Implementation Order (Recommended)

Based on dependency analysis:

1. **schemas.py** (AC 8) — no internal dependencies; other modules depend on ErrorResponse, HealthResponse
2. **config.py** (AC 1) — no internal dependencies
3. **logging.py** (AC 2) — no internal dependencies; provides context var helpers used by middleware
4. **exceptions.py** (AC 7) — depends on schemas (ErrorResponse), logging (get_correlation_id)
5. **middleware/** (AC 3, 4, 5) — depends on exceptions (UnauthorizedError), logging (set_correlation_id, set_tenant_id)
6. **health.py** (AC 6) — depends on schemas (HealthResponse, HealthCheckResult)
7. **__init__.py** exports — after all modules exist

---

## Quality Gate Criteria

| Gate | Threshold | Status |
|------|-----------|--------|
| P1 tests pass rate | 100% | ⏳ Pending implementation |
| P2 tests pass rate | ≥95% | ⏳ Pending implementation |
| Coverage on eusolicit-common | ≥80% | ⏳ Pending implementation |
| No high-risk items unmitigated | E01-R-008 covered | ✅ |
| Existing test regression | 0 new failures | ⏳ Verify after implementation |

---

## Validation Checklist

- [x] Prerequisites satisfied (story approved, pytest configured, dev env available)
- [x] All 6 test files created with correct structure and markers
- [x] Tests cover all 9 acceptance criteria (AC 9 = meta/coverage)
- [x] Tests designed to fail before implementation (ATDD RED phase)
- [x] Priority tags assigned per epic-level test design
- [x] No placeholder assertions (all assert expected behavior)
- [x] Test patterns match existing project conventions (class-based, docstrings, async)
- [x] No CLI sessions to clean up (backend-only, no browser automation)
- [x] Checklist saved to `test-artifacts/` directory
- [x] No DO NOT TOUCH files modified

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
