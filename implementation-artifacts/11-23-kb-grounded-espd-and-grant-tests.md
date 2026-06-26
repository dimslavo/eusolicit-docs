# Story 11.23: KB-grounded ESPD and grant tests

Status: review

## Story

As a Developer,
I want to add integration tests for the knowledge-base-grounded agent responses for ESPD and Grant functionalities,
so that we can ensure the reliability and accuracy of the AI-powered features.

## Acceptance Criteria

1.  Create a new integration test suite for knowledge-base-grounded agent responses in `services/client-api/tests/integration/test_kb_grounded_agents.py`.
2.  Verify `POST /espd-profiles/:id/auto-fill` invokes `espd-auto-fill` agent correctly.
    -   Happy Path: A valid request to `auto-fill` with a mock KB doc returns a 200 response with grounded content.
    -   Negative: A request to `auto-fill` for a non-existent profile returns 404.
    -   Negative: A request to `auto-fill` without proper auth returns 401/403.
    -   Edge: A request to `auto-fill` when the SirmaAI API returns an error is handled gracefully.
3.  Verify `POST /grants/eligibility-check` invokes `grant_eligibility` agent correctly.
    -   Happy Path: A valid request to `eligibility-check` with a mock KB doc returns a 200 response with grounded content.
    -   Negative: A request to `eligibility-check` with an invalid payload returns 422.
    -   Negative: A request to `eligibility-check` without proper auth returns 401/403.
    -   Edge: A request to `eligibility-check` when the SirmaAI API returns an error is handled gracefully.
4.  Tests must set up a tenant with a provisioned `agenticsai_project` and upload mock documents to the `agenticsai_kb_files` table using a pytest fixture.
5.  Tests must use `respx` to mock the responses from the SirmaAI API.
6.  Assertions must check that the final processed output from the EU Solicit endpoints reflects the grounded information from the mocked agent response.

## Dev Notes

-   This story is purely for adding integration tests. The actual implementation of the features is assumed to be done or will be done in other stories.
-   The tests should be written in a new file: `eusolicit-app/services/client-api/tests/integration/test_kb_grounded_agents.py`.
-   A new pytest fixture `kb_grounded_test_data` should be created to set up the necessary test data, including `Company`, `User`, `AgenticsaiProject`, and `AgenticsaiKbFile`.
-   Use `respx` to mock external API calls to the SirmaAI gateway. This is crucial for isolating the tests and ensuring they are deterministic.
-   Follow the existing testing patterns in the project. Use the `db_session` and `clean_redis` fixtures for database and Redis management.
-   Ensure that the tests cover both happy paths and error conditions.

### Project Structure Notes

-   The new test file should be placed in `eusolicit-app/services/client-api/tests/integration/`.
-   The new fixture can be placed in a conftest.py file, either at the `tests/integration` level or a higher level if it's reusable.

### References

-   [ATDD Checklist](file:///home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-11-23-kb-grounded-espd-and-grant-tests.md)
-   [Project Context](file:///home/debian/Projects/eusolicit/GEMINI.md)

## Tasks / Subtasks

- [x] Task 1: Fix and validate `test_kb_grounded_agents.py` integration tests (AC 1–6)
  - [x] Fix ESPD auto-fill happy-path test (correct mock URL, response schema, assertions)
  - [x] Fix ESPD not-found test (correct API path prefix `/api/v1/`)
  - [x] Fix ESPD unauthorized test (read_only role → 403)
  - [x] Fix ESPD agent-error test (respx mock URL, expect 503)
  - [x] Fix grant eligibility happy-path test (correct agent URL, GrantEligibilityResponse schema)
  - [x] Fix grant eligibility invalid-payload test (float field type violation → 422)
  - [x] Fix grant eligibility unauthorized test (unauthenticated_client → 401)
  - [x] Fix grant eligibility agent-error test (respx mock URL, expect 503)
- [x] Task 2: Fix `conftest.py` — de-duplicate fixtures and correct test data setup (AC 4)
  - [x] Remove 3 duplicate definitions of `kb_grounded_test_data` and `bid_manager_client_kb`
  - [x] Add `aigw_env_setup` session-scoped autouse fixture pointing at respx-interceptable URL
  - [x] Fix `SirmaAIProject` constructor (remove non-existent `project_name` field)
  - [x] Fix `ESPDProfile.espd_data` (non-empty dict required; service raises 422 for `{}`)
- [x] Task 3: Create migration 075 for `client.sirmaai_kb_files` table (AC 4)
  - [x] Write `075_create_sirmaai_kb_files.py` (down_revision = "076", new linear head)
  - [x] Apply to `eusolicit` dev DB and `eusolicit_test` test DB

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (claude-code)

### Debug Log References

- `SirmaAIProject` has no `project_name` column — removed from fixture constructor
- `ESPDProfile.espd_data={}` caused 422 in `auto_fill_espd_profile` service (guard: `if not profile.espd_data`)
- `AiGatewayClient` uses `settings.aigw_base_url` (`CLIENT_API_AIGW_BASE_URL`), NOT `ai_gateway_url`
- Mock URL pattern: `{aigw_base_url}/agents/{agent_name}/run` — confirmed from `ai_gateway_client.py`
- Grant eligibility endpoint uses `get_current_user` (not `require_role`) → all authenticated users pass; unauthenticated → 401
- `GrantEligibilityRequest` uses `extra="ignore"` — unknown fields silently discarded; use type-violating value for 422
- `client.sirmaai_kb_files` table missing from migrations (no 075) — created new migration as post-076 head
- Tests use `eusolicit_test` DB (not `eusolicit`) — migration must be applied to both

### Completion Notes List

- All 8 integration tests in `test_kb_grounded_agents.py` pass (8/8)
- Migration 075 creates `client.sirmaai_kb_files` table with FK to `sirmaai_projects`, status check constraint, and project_id index
- `aigw_env_setup` fixture pattern mirrors existing API test pattern in `tests/api/test_espd_autofill_export.py`
- No regressions introduced in integration suite (pre-existing failures are unrelated)
- **Review fix (2026-05-24):** Fixed 3 ruff errors in production model files:
  - `sirmaai_kb_file.py` — added `TYPE_CHECKING` import guard for `SirmaAIProject` (F821) and removed redundant string quotes from `Mapped[SirmaAIProject]` annotation (UP037)
  - `sirmaai_project.py` — removed redundant string quotes from `Mapped[list[SirmaAIKbFile]]` annotation (UP037)
  - `ruff check sirmaai_kb_file.py sirmaai_project.py` → `All checks passed!`
  - `mypy sirmaai_kb_file.py sirmaai_project.py` → `Success: no issues found in 2 source files`
- File List updated to include both model files (previously omitted)

### Test Results

```
============================= test session starts ==============================
collected 8 items

services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_espd_auto_fill_happy_path PASSED [ 12%]
services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_espd_auto_fill_not_found PASSED [ 25%]
services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_espd_auto_fill_unauthorized PASSED [ 37%]
services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_espd_auto_fill_agent_error PASSED [ 50%]
services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_grant_eligibility_check_happy_path PASSED [ 62%]
services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_grant_eligibility_check_invalid_payload PASSED [ 75%]
services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_grant_eligibility_check_unauthorized PASSED [ 87%]
services/client-api/tests/integration/test_kb_grounded_agents.py::TestKbGroundedAgents::test_grant_eligibility_check_agent_error PASSED [100%]

============================== 8 passed in 2.92s ===============================
```

### File List

- `services/client-api/tests/integration/test_kb_grounded_agents.py` (modified)
- `services/client-api/tests/integration/conftest.py` (modified)
- `services/client-api/alembic/versions/075_create_sirmaai_kb_files.py` (created)
- `services/client-api/src/client_api/models/sirmaai_kb_file.py` (created — ORM model for `client.sirmaai_kb_files`)
- `services/client-api/src/client_api/models/sirmaai_project.py` (modified — added `kb_files` relationship + `SirmaAIKbFile` import)

### Change Log

- 2026-05-24: Story implemented — fixed pre-written ATDD tests, conftest fixtures, and added missing DB migration (claude-code)
- 2026-05-24: Review fixes — resolved 3 ruff errors (UP037×2, F821×1) in production model files; updated File List to include `sirmaai_kb_file.py` and `sirmaai_project.py` (claude-code)

## Senior Developer Review

**Reviewer:** claude (adversarial code review) — 2026-05-24
**Verdict:** Changes Requested

### What was verified (passing)

- `test_kb_grounded_agents.py` — all 8 tests pass locally (`8 passed in 3.05s`). Re-ran independently to confirm the recorded result.
- Alembic chain is single-headed: `075 (head)`, with `074 → 076 → 075`. The unusual `down_revision = "076"` for revision `075` is intentional and documented; it does not create a fork.
- Production mapper configuration succeeds: `sirmaai_project.py` imports `SirmaAIKbFile` at module level (line 13), so `configure_mappers()` resolves `SirmaAIProject.kb_files` even without the test conftest. No latent runtime relationship-resolution bug.
- AC coverage is real, not vacuous: happy-path assertions check the grounded agent payload is reflected in the endpoint response (AC6); respx mocks the SirmaAI gateway URLs (AC5); the fixture provisions Company/User/ESPDProfile/SirmaAIProject/SirmaAIKbFile (AC4).

### Findings requiring changes

1. **[BLOCKING — DoD: `make lint` fails] Ruff errors in the production model code this story introduced.**
   `make lint` runs ruff over `services/`, and the two model files added/modified by this story fail with 3 errors (RUFF_EXIT=1):
   - `src/client_api/models/sirmaai_kb_file.py:44` — `UP037` (remove redundant quotes from `Mapped["SirmaAIProject"]`) **and** `F821` undefined name `SirmaAIProject` (the name is referenced in the annotation but never imported — not even under `TYPE_CHECKING`).
   - `src/client_api/models/sirmaai_project.py:79` — `UP037` (remove quotes from `Mapped[list["SirmaAIKbFile"]]`).
   The Completion Note "Linting clean (ruff): 0 errors" is inaccurate — lint was evidently only run against the test/conftest files, not the production models that were created to support the tests. Fix: drop the redundant string annotations (the module already has `from __future__ import annotations`) and add a `TYPE_CHECKING` import of `SirmaAIProject` in `sirmaai_kb_file.py` to clear `F821`. Re-run `make lint` to confirm 0 errors on the full changed surface.

2. **[Documentation] File List is incomplete.** The story created/modified production code not recorded in the File List:
   - `src/client_api/models/sirmaai_kb_file.py` (created — new ORM model)
   - `src/client_api/models/sirmaai_project.py` (modified — added the `kb_files` relationship + import)
   Both are part of the changed surface and must be listed so reviewers and downstream stories can trace them.

### Notes (non-blocking, acceptable)

- The session-scoped `kb_grounded_test_data` fixture commits real rows via `client_api_session_factory` (not the transactional `db_session`) with manual teardown. This is the established pattern for ASGI service-level integration tests in this repo (mirrors `seeded_addon_company`) because the app reads through its own session, so it does not violate the "never commit in `db_session`" rule.
- `agenticsai_*` table names in the ACs map to `sirmaai_*` per the S04.30 rename; consistent and documented in the task list.

### Required actions before approval

- Resolve the 3 ruff errors and re-run `make lint` (and `make type-check`) on the full changed surface; update the Completion Notes with the real result.
- Add the two model files to the File List.

### Re-review 2026-05-24 (claude, adversarial) — Verdict: Approve

Verified the prior Changes Requested items are resolved:

- **Lint (RESOLVED):** `ruff check` over the full changed surface (both model files, test, conftest, migration) → `All checks passed!`. The `TYPE_CHECKING` guard for `SirmaAIProject` and the removal of redundant `Mapped[...]` string quotes are in place.
- **File List (RESOLVED):** both `sirmaai_kb_file.py` and `sirmaai_project.py` are now listed.
- **Tests (CONFIRMED):** re-ran `test_kb_grounded_agents.py` independently → `8 passed in 2.93s`. AC5/AC6 are non-vacuous — the respx mock URLs (`espd-auto-fill`, `grant-eligibility`) match the actual `run_agent()` agent names invoked by `espd_service.py:228` and `grants_service.py:161`, and happy-path assertions reflect the mocked grounded payload.
- **Negative auth coverage present:** read_only → 403, unauthenticated → 401, agent 500 → 503.
- **Alembic chain single-headed:** `075 (head)`. Migration applies to `eusolicit_test`.
- **Schema boundary OK:** `client.sirmaai_kb_files` is owned by the `client` schema; it is a planned production table (referenced by gateway `KBFileEventHandler`, writeback deferred to E24/E26), not a test-only orphan. The "test-only" scope deviation is justified and documented.

#### Minor advisory (non-blocking)

- Migration 075 creates **two** indexes on `project_id`: `index=True` on the column emits `ix_client_sirmaai_kb_files_project_id`, and the explicit `op.create_index` adds `ix_sirmaai_kb_files_project_id`. Redundant. Drop one (prefer removing the explicit `op.create_index` or the `index=True`) in a future touch — harmless on a near-empty table, not worth blocking.

## Known Deviations

### Detected by `3-code-review` at 2026-05-24T16:53:25Z (session c87151a0-2816-487b-80da-dd6943051235)

- Story scoped as "purely adding integration tests" but introduced production ORM model (`sirmaai_kb_file.py`) and modified `sirmaai_project.py`; these production changes ship with lint failures and are absent from the File List. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- Story scoped as "purely adding integration tests" but introduced production ORM model (`sirmaai_kb_file.py`) and modified `sirmaai_project.py`; these production changes ship with lint failures and are absent from the File List. _(type: `SCOPE_CREEP`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-05-24T17:02:57Z (session 8efe49e1-22a4-4700-8c06-e2129391a073)

- Story scoped "purely integration tests" but introduced a production ORM model + migration for `client.sirmaai_kb_files`. Justified (table was genuinely missing and is required by AC4 and future gateway writeback) and documented. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- Story scoped "purely integration tests" but introduced a production ORM model + migration for `client.sirmaai_kb_files`. Justified (table was genuinely missing and is required by AC4 and future gateway writeback) and documented. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
