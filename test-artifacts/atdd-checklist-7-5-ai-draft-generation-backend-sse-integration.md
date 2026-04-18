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
storyId: 7-5-ai-draft-generation-backend-sse-integration
detectedStack: backend
generationMode: ai
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-5-ai-draft-generation-backend-sse-integration.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit/_bmad/bmm/config.yaml
  - eusolicit-app/services/client-api/tests/api/test_proposal_content_save.py
  - eusolicit-app/services/client-api/tests/api/test_ai_summary.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/src/client_api/services/ai_gateway_client.py
---

# ATDD Checklist: Story 7.5 — AI Draft Generation Backend SSE Integration

**Date:** 2026-04-17
**TDD Phase:** 🔴 RED — All tests are failing (implementation does not exist yet)
**Stack:** Backend (Python / FastAPI / pytest-asyncio)
**Test File:** `services/client-api/tests/api/test_proposal_generate.py`

---

## Summary

| Metric | Value |
|--------|-------|
| Total tests generated | 11 |
| Tests skipped (RED phase) | 11 |
| Tests passing | 0 |
| Acceptance criteria covered | AC1, AC3, AC4, AC5, AC6, AC7, AC8, AC9 |
| Epic test IDs covered | E07-P0-001, E07-P0-002, E07-P0-005, E07-P0-007, E07-P1-009, E07-P1-010, E07-P2-005 |

---

## Stack Detection

| Check | Result |
|-------|--------|
| `pyproject.toml` / `pytest.ini` present | ✅ Backend |
| `playwright.config.ts` / `package.json` (frontend) | Not relevant for this story |
| **Detected stack** | `backend` |
| **Generation mode** | AI generation (backend — no browser recording needed) |

---

## Prerequisites Verified

| Prerequisite | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 10 ACs with detailed implementation specs |
| Test framework configured | ✅ | `pytest` + `pytest-asyncio` in `services/client-api/` |
| Existing test patterns found | ✅ | `test_proposal_content_save.py`, `test_ai_summary.py` |
| `conftest.py` fixtures available | ✅ | `client_api_session_factory`, `test_redis_client`, `app` |
| AI Gateway client mock patterns understood | ✅ | `asynccontextmanager` + `unittest.mock.patch` on `stream_agent` |
| Epic test design loaded | ✅ | `test-design-epic-07.md` |

---

## Test Strategy

### Test Level Selection

This is a **backend-only** story (FastAPI + asyncpg + asyncio). All tests are **integration tests** using:
- `httpx.AsyncClient` with `ASGITransport` (real FastAPI app, no subprocess)
- `testcontainers PostgreSQL` (real DB via `client_api_session_factory` fixture)
- `unittest.mock.patch` on `AiGatewayClient.stream_agent` (respx not needed; direct `asynccontextmanager` mock)
- `asyncio.gather` for concurrency race condition tests

No E2E (Playwright) tests generated — this is backend-only. Frontend SSE tests are in Story 7.13.

### Test Priority Mapping

| Test | Class | Priority | Epic ID | Risk |
|------|-------|----------|---------|------|
| `test_disconnect_after_first_section_version_still_persisted` | `TestAC4ServerSidePersistence` | **P0** | E07-P0-001 | E07-R-001 |
| `test_reset_stuck_generating_proposals` | `TestAC7CleanupJob` | **P0** | E07-P0-002 | E07-R-001 |
| `test_generate_cross_company_returns_404` | `TestAC1AtomicClaim` | **P0** | E07-P0-005 | E07-R-003 |
| `test_concurrent_generate_exactly_one_200_one_409` | `TestAC1AtomicClaim` | **P0** | E07-P0-007 | E07-R-004 |
| `test_generate_returns_sse_stream_and_creates_version` | `TestAC1AtomicClaim` | **P1** | E07-P1-009 | E07-R-001 |
| `test_version_persisted_on_done_event` | `TestAC4ServerSidePersistence` | **P1** | E07-P1-009 | E07-R-001 |
| `test_gateway_error_event_sets_status_failed` | `TestAC6ErrorHandling` | **P1** | E07-P1-010 | — |
| `test_gateway_timeout_sets_status_failed` | `TestAC6ErrorHandling` | **P2** | E07-P2-005 | E07-R-007 |
| `test_generate_409_when_already_generating` | `TestAC1AtomicClaim` | **P1** | AC1 | E07-R-004 |
| `test_reset_stuck_does_not_reset_recent` | `TestAC7CleanupJob` | **P1** | AC7 | — |
| `test_generation_status_column_exists_with_default` | `TestAC8MigrationSchema` | **P2** | AC8 | — |
| `test_generation_status_check_constraint_rejects_invalid_value` | `TestAC8MigrationSchema` | **P2** | AC8 | — |
| `test_generate_unauthenticated_returns_401` | `TestAC1AtomicClaim` | **P2** | AC1 | — |

---

## Acceptance Criteria Coverage

### AC1 — Atomic generation_status claim (idle → generating)

| Scenario | Test | Status |
|----------|------|--------|
| Nominal claim succeeds; SSE stream starts | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |
| Already generating → 409 `generation_in_progress` | `test_generate_409_when_already_generating` | 🔴 SKIP |
| Two concurrent requests → exactly one 200, one 409 | `test_concurrent_generate_exactly_one_200_one_409` | 🔴 SKIP |
| Cross-company JWT → 404 (not 403) | `test_generate_cross_company_returns_404` | 🔴 SKIP |
| Unauthenticated → 401 | `test_generate_unauthenticated_returns_401` | 🔴 SKIP |

**Implementation required:**
- `claim_generation()` in `proposal_service.py` with atomic `UPDATE … WHERE generation_status='idle' RETURNING id`
- `require_role("bid_manager")` on endpoint
- `_assert_company_owns_proposal()` raises 404 for cross-company

### AC2 — Payload construction (opportunity, company, sections)

| Scenario | Test | Status |
|----------|------|--------|
| Payload built from proposal + company profile | Covered implicitly by `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |

**Note:** `build_generation_payload()` is exercised through the nominal stream test. Pipeline session is mocked to return empty opportunity data (graceful fallback per AC2 spec).

### AC3 — AI Gateway SSE events forwarded to client

| Scenario | Test | Status |
|----------|------|--------|
| 3 section events forwarded to client | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |
| metadata event forwarded to client | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |
| section events carry section_key, title, body | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |

### AC4 — Server-side persistence on `done` event (E07-R-001 mitigation)

| Scenario | Test | Status |
|----------|------|--------|
| New `proposal_version` created with `content.sections` | `test_version_persisted_on_done_event` | 🔴 SKIP |
| `proposals.current_version_id` updated | `test_version_persisted_on_done_event` | 🔴 SKIP |
| `generation_status = 'completed'` after done | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |
| `change_summary = 'AI generated draft'` | `test_version_persisted_on_done_event` | 🔴 SKIP |
| `done` SSE event carries `version_id` and `version_number` | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |

**Implementation required:** `persist_generated_draft()` with independent `session_factory()` session and `_version_write_locks`.

### AC5 — Client disconnect resilience (asyncio.create_task)

| Scenario | Test | Status |
|----------|------|--------|
| Disconnect after first section → version still persisted (E07-P0-001) | `test_disconnect_after_first_section_version_still_persisted` | 🔴 SKIP |

**Implementation required:**
- `asyncio.create_task(_run_generation_task(...))` — background task independent of SSE generator lifecycle
- `asyncio.Queue` as communication channel between background task and SSE generator

### AC6 — Error handling

| Scenario | Test | Status |
|----------|------|--------|
| AI Gateway `error` event → `generation_status = 'failed'`, no version | `test_gateway_error_event_sets_status_failed` | 🔴 SKIP |
| `AiGatewayTimeoutError` → `generation_status = 'failed'` | `test_gateway_timeout_sets_status_failed` | 🔴 SKIP |

**Implementation required:** `set_generation_status()` with independent session. `_run_generation_task` except clause.

### AC7 — Stuck cleanup

| Scenario | Test | Status |
|----------|------|--------|
| `generating` + updated_at > timeout → reset to `idle` (E07-P0-002) | `test_reset_stuck_generating_proposals` | 🔴 SKIP |
| Recent `generating` → NOT reset | `test_reset_stuck_does_not_reset_recent` | 🔴 SKIP |

**Implementation required:** `reset_stuck_generating_proposals(session)` in `proposal_service.py`.

### AC8 — Migration 021

| Scenario | Test | Status |
|----------|------|--------|
| `generation_status` column with default `idle` | `test_generation_status_column_exists_with_default` | 🔴 SKIP |
| CHECK constraint rejects invalid values | `test_generation_status_check_constraint_rejects_invalid_value` | 🔴 SKIP |

### AC9 — StreamingResponse headers

| Scenario | Test | Status |
|----------|------|--------|
| `Content-Type: text/event-stream` | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |
| `Cache-Control: no-cache` | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |
| `X-Accel-Buffering: no` | `test_generate_returns_sse_stream_and_creates_version` | 🔴 SKIP |

---

## Fixture Infrastructure

### New Fixtures (in test_proposal_generate.py)

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `proposal_with_opportunity` | function | Registers user/company, creates proposal with scaffold sections, clears AI gateway singleton |
| `_make_pipeline_session_override()` | helper | Mocks `get_pipeline_readonly_session` to return empty result (no real pipeline DB needed) |

### Mock Helpers

| Helper | Pattern | Purpose |
|--------|---------|---------|
| `_mock_sse_stream_success()` | `asynccontextmanager` | 3 sections + metadata + done |
| `_mock_sse_stream_error()` | `asynccontextmanager` | 1 section + error event |
| `_mock_sse_stream_single_section()` | `asynccontextmanager` | Full stream (all 3 sections + done); client disconnects after first section line |
| `_stream_agent_success` | `side_effect` function | Wrapper for `patch(..., side_effect=...)` |
| `_consume_sse_events()` | async helper | Parses SSE lines from streaming httpx response |

### Reused Fixtures (from conftest.py)

- `client_api_session_factory` — asyncpg session factory for testcontainers PostgreSQL
- `test_redis_client` — Redis DB index 1 for rate-limit isolation

---

## Implementation Checklist (TDD Green Phase)

When development is complete, remove `@pytest.mark.skip` from each test in this order:

### Phase 1 — Migration & Schema (no app code needed)
- [ ] Run `alembic upgrade head` to apply migration 021
- [ ] Remove skip from `test_generation_status_column_exists_with_default`
- [ ] Remove skip from `test_generation_status_check_constraint_rejects_invalid_value`
- [ ] Verify: 2 tests pass

### Phase 2 — Service Layer
- [ ] Implement `claim_generation()` with atomic SQL
- [ ] Implement `set_generation_status()` (independent session)
- [ ] Implement `persist_generated_draft()` (independent session + `_version_write_locks`)
- [ ] Implement `reset_stuck_generating_proposals()`
- [ ] Implement `_run_generation_task()` (background task logic)
- [ ] Remove skip from `test_generate_409_when_already_generating`
- [ ] Remove skip from `test_reset_stuck_generating_proposals`
- [ ] Remove skip from `test_reset_stuck_does_not_reset_recent`
- [ ] Verify: 3 additional tests pass

### Phase 3 — Router & SSE Endpoint
- [ ] Implement `POST /{proposal_id}/generate` in `proposals.py`
- [ ] Add `_sse_stream_from_queue()` helper
- [ ] Add `asyncio.create_task()` background task launch
- [ ] Remove skip from `test_generate_returns_sse_stream_and_creates_version`
- [ ] Remove skip from `test_generate_unauthenticated_returns_401`
- [ ] Remove skip from `test_generate_cross_company_returns_404`
- [ ] Remove skip from `test_version_persisted_on_done_event`
- [ ] Remove skip from `test_gateway_error_event_sets_status_failed`
- [ ] Remove skip from `test_gateway_timeout_sets_status_failed`
- [ ] Verify: 6 additional tests pass

### Phase 4 — Critical Path Tests (require real asyncio concurrency)
- [ ] Remove skip from `test_disconnect_after_first_section_version_still_persisted`
- [ ] Remove skip from `test_concurrent_generate_exactly_one_200_one_409`
- [ ] Verify: 2 additional tests pass (requires testcontainers PostgreSQL MVCC)

### Final Validation
- [ ] All 13 tests pass (or 11 non-schema tests if migration already applied)
- [ ] Run full regression: `pytest tests/api/test_proposals.py tests/api/test_proposal_versions.py tests/api/test_proposal_content_save.py tests/api/test_proposal_generate.py -v`
- [ ] 110+ existing tests remain green
- [ ] `generation_status` field appears in `GET /proposals/:id` response (ProposalResponse schema update)

---

## Architectural Constraints (from Story Dev Notes)

| Constraint | Enforced By |
|-----------|-------------|
| `CAST(:id AS uuid)` — never `::uuid` in `text()` SQL | `test_disconnect_after_first_section_version_still_persisted` (uses independent session) |
| `asyncio.create_task()` for background task — not inline | `test_disconnect_after_first_section_version_still_persisted` (verifies persistence after disconnect) |
| Independent `session_factory()` session for `persist_generated_draft` | `test_disconnect_after_first_section_version_still_persisted` |
| Atomic `UPDATE … WHERE generation_status='idle' RETURNING id` | `test_concurrent_generate_exactly_one_200_one_409` (PostgreSQL MVCC) |
| `company_id` from JWT only — never from request | `test_generate_cross_company_returns_404` |
| Return 404 not 403 for cross-company | `test_generate_cross_company_returns_404` |
| `structlog` for all logging | Code review gate (not directly testable) |
| `_version_write_locks` registry in `persist_generated_draft` | `test_concurrent_generate_exactly_one_200_one_409` (concurrent version write) |

---

## Risk Coverage Summary

| Risk ID | Description | Covered By | Priority |
|---------|-------------|------------|---------|
| **E07-R-001** | SSE stream persistence failure on client disconnect | `test_disconnect_after_first_section_version_still_persisted`, `test_reset_stuck_generating_proposals` | P0 |
| **E07-R-003** | Proposal RLS cross-company bypass | `test_generate_cross_company_returns_404` | P0 |
| **E07-R-004** | `generation_status` duplicate trigger race | `test_concurrent_generate_exactly_one_200_one_409`, `test_generate_409_when_already_generating` | P0 |
| **E07-R-007** | AI Gateway timeout (cascade failure) | `test_gateway_timeout_sets_status_failed` | P2 |

---

## Running Tests

```bash
# Run only Story 7.5 tests (RED phase — all skip in current state)
cd eusolicit-app/services/client-api
pytest tests/api/test_proposal_generate.py -v

# Expected output in RED phase:
#   13 skipped, 0 passed, 0 failed

# Full regression check (after implementation — GREEN phase):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py -v

# Run only P0 tests by marker (after removing @pytest.mark.skip):
pytest tests/api/test_proposal_generate.py -v \
  -k "disconnect or concurrent or cross_company or reset_stuck"
```

---

## Next Steps

1. Implement Task 1 (Alembic migration 021) → unlock schema tests
2. Implement Task 2 (ORM model) + Tasks 3–4 (config + service functions) → unlock service tests
3. Implement Tasks 5–7 (schema + router + SSE generator) → unlock endpoint tests
4. Run `test_disconnect_after_first_section_version_still_persisted` and `test_concurrent_generate_exactly_one_200_one_409` last — they need the full async task infrastructure
5. Run `/bmad-testarch-automate` to expand coverage (metadata event forwarding, heartbeat silencing, ProposalResponse schema `generation_status` field, Celery beat schedule config)

---

**Generated by:** BMad TEA Agent — Master Test Architect (`bmad-testarch-atdd`)
**Story:** 7.5 — AI Draft Generation Backend SSE Integration
**Epic:** E07 — Proposal Generation & Document Intelligence
