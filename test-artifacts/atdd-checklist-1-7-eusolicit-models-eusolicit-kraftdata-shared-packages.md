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
storyId: '1-7-eusolicit-models-eusolicit-kraftdata-shared-packages'
detectedStack: 'backend'
generationMode: 'ai-generation'
executionMode: 'sequential'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-app/pyproject.toml'
  - 'eusolicit-app/packages/eusolicit-models/pyproject.toml'
  - 'eusolicit-app/packages/eusolicit-kraftdata/pyproject.toml'
  - 'eusolicit-app/packages/eusolicit-common/src/eusolicit_common/schemas.py'
  - 'eusolicit-app/tests/conftest.py'
---

# ATDD Checklist: Story 1.7 -- eusolicit-models & eusolicit-kraftdata Shared Packages

**Date:** 2026-04-06
**Story:** 1-7-eusolicit-models-eusolicit-kraftdata-shared-packages
**TDD Phase:** RED (failing tests -- implementation pending)
**Stack:** Python backend (FastAPI + pytest + pytest-asyncio)

---

## Preflight Summary

| Item | Status |
|------|--------|
| Story approved with clear ACs | ✅ 11 acceptance criteria |
| Test framework configured | ✅ pytest + pytest-asyncio (asyncio_mode="auto") |
| Dev environment available | ✅ Monorepo with editable installs |
| Detected stack | `backend` (Python/FastAPI) |
| Generation mode | AI generation (no browser recording for backend) |
| Epic test design loaded | ✅ test-design-epic-01.md |
| TEA config flags | None configured (Playwright/Pact disabled -- N/A for backend library) |

---

## TDD Red Phase Status

✅ **Failing tests generated**

| Test File | Test Count | Marker | AC Coverage | Priority |
|-----------|-----------|--------|-------------|----------|
| `tests/unit/test_eusolicit_models_enums.py` | 27 | `@pytest.mark.skip` (ATDD RED) | AC 4 | P1 |
| `tests/unit/test_eusolicit_models_dtos.py` | 29 | `@pytest.mark.skip` (ATDD RED) | AC 1, 5 | P1 |
| `tests/unit/test_eusolicit_models_events.py` | 38 | `@pytest.mark.skip` (ATDD RED) | AC 2, 3, 5 | P1 |
| `tests/unit/test_eusolicit_kraftdata_requests.py` | 27 | `@pytest.mark.skip` (ATDD RED) | AC 6, 9, 10, 11 | P1 |
| `tests/unit/test_eusolicit_kraftdata_responses.py` | 38 | `@pytest.mark.skip` (ATDD RED) | AC 7, 8, 9, 10, 11 | P1 |
| `tests/unit/test_eusolicit_kraftdata_no_http_deps.py` | 5 | `@pytest.mark.skip` (ATDD RED) | AC 10 | P1 |
| **Total** | **164** | | **AC 1-11** | |

---

## Acceptance Criteria Coverage Matrix

| AC | Description | Test File | Scenarios | Priority |
|----|-------------|-----------|-----------|----------|
| **AC 1** | REST DTOs inherit from BaseSchema (camelCase, populate_by_name, from_attributes) | `test_eusolicit_models_dtos.py` | 6 DTOs: construct, camelCase serialization, deserialization, round-trip, from_attributes ORM, populate_by_name config, BaseSchema inheritance | **P1** |
| **AC 2** | Event schemas with Literal event_type discriminator | `test_eusolicit_models_events.py` | 6 events: Literal event_type value, payload fields, serialization round-trip | **P1** |
| **AC 3** | BaseEvent: event_id, timestamp, correlation_id, source_service, tenant_id | `test_eusolicit_models_events.py` | Auto-generated UUID event_id, auto-generated datetime timestamp, auto-generated UUID correlation_id, required source_service, optional tenant_id, BaseSchema inheritance, camelCase serialization | **P1** |
| **AC 4** | Enums: OpportunityStatus, AgentStatus, TaskPriority, NotificationChannel, SubscriptionTier + TaskStatus, ProposalStatus, ApprovalDecision | `test_eusolicit_models_enums.py` | 9 enums: expected values, member count, str mixin, JSON serialization, isinstance(str) check | **P1** |
| **AC 5** | Unit tests: serialization/deserialization round-trips for all DTOs and events | `test_eusolicit_models_dtos.py`, `test_eusolicit_models_events.py` | Round-trip tests for all 6 DTOs and all 6 events; enum field validation; optional field handling | **P1** |
| **AC 6** | Request models: AgentRunRequest, WorkflowRunRequest, TeamRunRequest, StorageResourceRequest, WebhookRegistrationRequest | `test_eusolicit_kraftdata_requests.py` | 5 models: construct, required/optional fields, default values, serialization | **P1** |
| **AC 7** | Response models: AgentRunResponse, WorkflowRunResponse, TeamRunResponse, StorageResourceResponse, WebhookEvent | `test_eusolicit_kraftdata_responses.py` | 5 models + WebhookEvent + StreamChunk: construct, optional fields, enum validation, serialization | **P1** |
| **AC 8** | Status enums: RunStatus, ResourceType, WebhookEventType | `test_eusolicit_kraftdata_responses.py` | 3 enums: expected values, member count, str mixin, invalid value raises ValidationError | **P1** |
| **AC 9** | Documentation: all models include docstrings | `test_eusolicit_kraftdata_requests.py`, `test_eusolicit_kraftdata_responses.py` | Docstring presence check on all 12 kraftdata models | **P2** |
| **AC 10** | Types only: no HTTP client code, no httpx/requests dependency | `test_eusolicit_kraftdata_no_http_deps.py`, `test_eusolicit_kraftdata_requests.py` | AST-scanned source files: no httpx, requests, aiohttp, urllib3 imports; pyproject.toml has only pydantic; BaseModel (not BaseSchema) inheritance | **P1** |
| **AC 11** | Unit tests: model instantiation and validation for all request/response types | `test_eusolicit_kraftdata_requests.py`, `test_eusolicit_kraftdata_responses.py` | All request/response models: construct, validate, serialize, required fields enforced | **P1** |

---

## Risk Coverage

| Risk ID | Description | Score | Test Coverage |
|---------|-------------|-------|---------------|
| **E01-R-007** | Redis Streams event serialization inconsistency | 4 (MEDIUM) | BaseEvent inherits from BaseSchema ensuring consistent camelCase envelope; all 6 events have Literal discriminator tests; ServiceEvent discriminated union validates correct subclass parsing; round-trip serialization verified for all events |

---

## Priority Distribution

| Priority | Count | Description |
|----------|-------|-------------|
| **P1** | 152 | Enums, DTOs, events, request/response models, discriminated union, no-HTTP-dep |
| **P2** | 12 | Docstring presence checks |
| **Total** | **164** | |

---

## Test Strategy Notes

### Why Unit Tests Only

Both stories create **shared type libraries** -- not services with API endpoints or UI. All tests are pure unit tests:

- `@pytest.mark.unit` marker on all tests
- No Docker Compose required
- No integration tests (no real DB/Redis connections)
- No mock dependencies needed (pure Pydantic model validation)
- `pydantic.ValidationError` assertions for invalid data
- `pydantic.TypeAdapter` for discriminated union testing

### Why No E2E/API Tests

- No browser UI -> no E2E tests
- No service endpoints exposed -> no API-level tests
- These are type definition libraries consumed by services in E02+
- Service integration deferred to later stories

### Failure Mechanism

Tests fail via two mechanisms:
1. **Import-time failure**: `from eusolicit_models.enums import ...` raises `ImportError` because the modules don't exist yet
2. **Skip marker**: `@pytest.mark.skip(reason="ATDD RED: ...")` ensures tests are skipped in CI until implementation begins

When the developer starts implementing:
1. Remove the `pytest.mark.skip` marker from the file under development
2. Run the tests -- they should fail with `ImportError` initially
3. Implement the module -- tests should start passing
4. Repeat for each module

---

## Files Generated

| File | Purpose | AC |
|------|---------|-----|
| `eusolicit-app/tests/unit/test_eusolicit_models_enums.py` | 9 domain enums: values, str mixin, member counts | AC 4 |
| `eusolicit-app/tests/unit/test_eusolicit_models_dtos.py` | 6 REST DTOs: camelCase, round-trip, from_attributes, enum validation | AC 1, 5 |
| `eusolicit-app/tests/unit/test_eusolicit_models_events.py` | BaseEvent + 6 events + ServiceEvent discriminated union | AC 2, 3, 5 |
| `eusolicit-app/tests/unit/test_eusolicit_kraftdata_requests.py` | 5 KraftData request models | AC 6, 9, 10, 11 |
| `eusolicit-app/tests/unit/test_eusolicit_kraftdata_responses.py` | 3 enums + 5 response models + WebhookEvent + StreamChunk | AC 7, 8, 9, 10, 11 |
| `eusolicit-app/tests/unit/test_eusolicit_kraftdata_no_http_deps.py` | AST-based import scan + pyproject.toml dependency check | AC 10 |
| `eusolicit-docs/test-artifacts/atdd-checklist-1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md` | This ATDD checklist | -- |

---

## Next Steps (TDD Green Phase)

After implementing the feature modules:

1. **For each module**, remove the `pytest.mark.skip` line from the corresponding test file
2. Run tests: `cd eusolicit-app && python -m pytest tests/unit/test_eusolicit_models_<module>.py -v`
3. Verify tests PASS (green phase)
4. If any tests fail:
   - Fix implementation (feature bug) -- tests define the contract
   - Or fix test (test bug) -- only if the AC was misinterpreted
5. Run full coverage check: `python -m pytest --cov=packages/eusolicit-models --cov=packages/eusolicit-kraftdata --cov-report=term-missing tests/unit/test_eusolicit_models_*.py tests/unit/test_eusolicit_kraftdata_*.py`
6. Verify >= 80% coverage (epic quality gate)
7. Run full regression: `make test` -- verify no regressions in existing tests

### Implementation Order (Recommended)

Based on dependency analysis:

**eusolicit-models (implement first -- depends on eusolicit-common):**

1. **enums.py** (AC 4) -- no internal dependencies; DTOs and events reference these enums
2. **dtos.py** (AC 1) -- depends on enums.py and eusolicit_common.schemas.BaseSchema
3. **events.py** (AC 2, 3) -- depends on eusolicit_common.schemas.BaseSchema; defines ServiceEvent union
4. **pyproject.toml** -- add eusolicit-common dependency
5. **__init__.py** -- export all DTOs, events, enums, ServiceEvent

**eusolicit-kraftdata (implement second -- standalone, no internal deps):**

6. **enums.py** (AC 8) -- standalone, only depends on pydantic
7. **requests.py** (AC 6) -- standalone pydantic BaseModel
8. **responses.py** (AC 7) -- depends on enums.py
9. **__init__.py** -- export all models

---

## Quality Gate Criteria

| Gate | Threshold | Status |
|------|-----------|--------|
| P1 tests pass rate | >= 95% | Pending implementation |
| P2 tests pass rate | >= 90% | Pending implementation |
| Coverage on eusolicit-models | >= 80% | Pending implementation |
| Coverage on eusolicit-kraftdata | >= 80% | Pending implementation |
| No high-risk items unmitigated | E01-R-007 covered | ✅ |
| Existing test regression | 0 new failures | Verify after implementation |

---

## Validation Checklist

- [x] Prerequisites satisfied (story approved, pytest configured, dev env available)
- [x] All 6 test files created with correct structure and markers
- [x] Tests cover all 11 acceptance criteria
- [x] Tests designed to fail before implementation (ATDD RED phase)
- [x] Priority tags assigned per epic-level test design
- [x] No placeholder assertions (all assert expected behavior)
- [x] Test patterns match existing project conventions (class-based, docstrings, pytestmark)
- [x] No CLI sessions to clean up (backend-only, no browser automation)
- [x] Checklist saved to `test-artifacts/` directory
- [x] No DO NOT TOUCH files modified
- [x] eusolicit-models tests verify BaseSchema inheritance (camelCase, populate_by_name, from_attributes)
- [x] eusolicit-kraftdata tests verify BaseModel inheritance (NOT BaseSchema)
- [x] ServiceEvent discriminated union tested with all 6 event types
- [x] No HTTP dependency scan uses AST parsing (not string matching)

---

**Generated by:** BMad TEA Agent -- ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
