---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-25'
workflowType: bmad-testarch-atdd
mode: create
storyId: 10-10-bid-no-bid-decision-api-ai-integration
storyKey: 10-10-bid-no-bid-decision-api-ai-integration
storyFile: eusolicit-docs/implementation-artifacts/10-10-bid-no-bid-decision-api-ai-integration.md
atddChecklistPath: test_artifacts/atdd-checklist-10-10-bid-no-bid-decision-api-ai-integration.md
generatedTestFiles:
  - eusolicit-app/services/client-api/tests/integration/test_044_migration.py
  - eusolicit-app/services/client-api/tests/unit/test_bid_decision_service.py
  - eusolicit-app/services/client-api/tests/api/test_bid_decisions.py
detectedStack: fullstack (story type: backend)
generationMode: AI Generation (backend — no browser recording)
tddPhase: GREEN
inputDocuments:
  - eusolicit-docs/implementation-artifacts/10-10-bid-no-bid-decision-api-ai-integration.md
  - test_artifacts/atdd-checklist-10-8-approval-workflow-stages-crud-api.md
  - eusolicit-app/services/client-api/tests/api/test_bid_decisions.py
  - eusolicit-app/services/client-api/tests/unit/test_bid_decision_service.py
  - eusolicit-app/services/client-api/tests/integration/test_044_migration.py
  - eusolicit-app/services/client-api/tests/conftest.py
---

# ATDD Checklist: Story 10.10 — Bid/No-Bid Decision API & AI Integration

**Date:** 2026-04-25
**Author:** BMAD TEA Master Test Architect
**TDD Phase:** 🟢 GREEN — Implementation already exists and has been code-reviewed (REVIEW: Approve,
  follow-up Approve 2026-04-21). Tests were generated/extended during the dev-story and
  subsequently fixed for environment-specific issues (role naming, superuser privileges for
  seeding `pipeline.opportunities`, missing `source_id`/`source_type` columns). All tests are
  expected to PASS when the test DB runs migrations through `044_bid_decisions_runtime_columns`.
**Story Status:** draft (dev-complete, code-reviewed — two rounds, final verdict: Approve)

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `fullstack` (auto-detection: `pyproject.toml` + `playwright.config.ts` both present)
- **Story type override:** `backend` — this story has no frontend components; only backend test levels apply
- **Test framework:** pytest 8.x + pytest-asyncio + httpx AsyncClient + respx MockRouter

### Prerequisites Check

| Requirement | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 10 ACs with exhaustive scenario list in AC10 (≥ 28 scenarios specified) |
| pytest conftest with async session fixture | ✅ | `client_api_session_factory`, `superuser_session_factory` in conftest.py |
| `require_role` gate (Story 2.10) | ✅ | `bid_manager` gate; `admin` and `bid_manager` pass, lower roles → 403 |
| `get_current_user` (Story 2.4) | ✅ | Used in all auth-gated endpoints |
| `audit_service.write_audit_entry` (Story 2.11) | ✅ | Used in POST /evaluate and POST /bid-decision |
| `AiGatewayClient` (Story 11.3) | ✅ | `client_api.services.ai_gateway_client` — `run_agent`, error types |
| `pipeline.opportunities` Core Table (Story 6.1/6.5) | ✅ | Cross-schema read for opportunity context |
| `client.companies` (Story 1.3) | ✅ | Company profile payload building |
| `client.bid_outcomes` (Story 7.1) | ✅ | Historical context rollup (read-only join) |
| `BidDecision` ORM model (AC2) | ✅ | `client_api/models/bid_decision.py` |
| `schemas/bid_decisions.py` (AC3) | ✅ | All 6 schema classes implemented |
| `services/bid_decision_service.py` (AC4-AC7) | ✅ | Evaluate, decide, get, upsert, historical context |
| `/api/v1/opportunities/{id}/bid-decision` router (AC9) | ✅ | Mounted in `client_api/main.py` |
| Migration `044_bid_decisions_runtime_columns` (AC1) | ✅ | In-place ALTER TABLE on `client.bid_decisions` |
| `agents.yaml` entry `bid-no-bid-decision` (AC8) | ✅ | Added with `timeout_override: 120` |
| respx installed for AI Gateway mocking | ✅ | Module-level `pytest.importorskip("respx")` guard |

### Test Design Sources

Per Dev Notes, Epic 10 has **no epic-level test design** (only per-story ATDD checklists 10-1
through 10-9; no `test-design-epic-10.md` in `test_artifacts/`). Coverage strategy was drawn from:
- Story 11.3 `test_espd_autofill_export.py` — `respx.MockRouter` AI Gateway mock pattern
- Story 10.9 `tests/api/test_approvals.py` — arrange/act/assert format reference
- Story 10.9 `tests/integration/test_migration_043.py` — integration migration test structure
- Story 10.1 ATDD checklist — existence-leakage 404 (not 403) tenancy isolation strategy
- Story 10.8 ATDD checklist — pattern for backend-only GREEN-phase ATDD documentation

---

## Step 2: Generation Mode

**Selected mode:** AI Generation (backend — no browser recording)

**Rationale:** Pure backend API story (`type: backend` in metadata). All test patterns follow
`test_approvals.py` arrange/act/assert format with respx mocking for the AI Gateway. No browser
automation required. `bid_decision_ctx` fixture provides Company A admin/bid_manager/contributor
and Company B admin, plus live and soft-deleted pipeline opportunities.

---

## Step 3: Test Strategy

| Priority | Area | Scenarios | Harness |
|---|---|---|---|
| **P0** | Migration 044 lifecycle (AC1, AC10) | 5 (in 1 integration test method) | pytest integration + alembic subprocess |
| **P0** | ORM model correctness (AC2) | 8 | Unit (no DB — import inspection) |
| **P0** | Pydantic schema validation (AC3, AC10) | 9 | Unit (no DB — Pydantic validation) |
| **P0** | POST /evaluate happy path (AC4) | 3 | API — respx MockRouter |
| **P0** | POST /evaluate auth/gate failures (AC4) | 3 | API — 401/403/404 |
| **P0** | POST /evaluate agent failure handling (AC4) | 4 | API — respx timeout/503/malformed |
| **P0** | POST /bid-decision happy paths (AC5) | 5 | API — DB seeded rows |
| **P0** | POST /bid-decision gate failures (AC5) | 2 | API — 404/422 |
| **P1** | GET /bid-decision happy + gate paths (AC7) | 3 | API — 200/404 |
| **P1** | Upsert semantics — field isolation (AC6) | 2 | Unit — SQL inspection |
| **P1** | Historical context fallback breadcrumb (AC4, AC6) | 1 | Unit — mocked AsyncSession |
| **P1** | Agent registry entry (AC8) | 1 | Unit — YAML parse |
| **P2** | Audit trail — evaluate + decide (AC4, AC5, AC10) | 2 | API — `shared.audit_log` assertion |
| **P2** | Router wiring + no doubled prefix (AC9) | 2 | API / unit — route inspection |

**Total:** 48 tests across 3 test files (1 integration class, 23 unit, 25 API — one integration
test method encapsulates 5 migration sub-scenarios).

---

## Step 4: Generated Test Files

### Integration Tests

#### `tests/integration/test_044_migration.py` (1 test class, 5 sub-scenarios)

Migration lifecycle test — requires live DB (`@pytest.mark.integration`); uses alembic subprocess
and `migration_role` connection.

**`TestMigration044::test_upgrade_044_lifecycle`** covers:
1. `test_migration_044_adds_runtime_columns` — all 7 new columns present in `information_schema`
2. `test_migration_044_check_constraint_rejects_invalid_decision` — `decision='maybe'` raises
   `ck_bid_decisions_decision` IntegrityError
3. `test_migration_044_unique_constraint_on_company_opportunity` — duplicate `(company_id, opportunity_id)`
   raises `uq_bid_decisions_company_opportunity` IntegrityError
4. `test_migration_044_downgrade_reverses` — downgrade to 043 removes `ai_recommendation` column
5. Lifecycle completeness — upgrade/downgrade/upgrade-head round-trip returns code 0

---

### Unit Tests

#### `tests/unit/test_bid_decision_service.py` (23 tests)
Pure inspection tests — no DB, no mocking (except historical-context fallback).

**AC1 — Migration structure (pure Python):**
- `test_migration_044_file_exists` — `044_*.py` exists in `alembic/versions/`
- `test_migration_044_revision_chain` — `revision == "044"`, `down_revision == "043"`

**AC2 — BidDecision ORM model (import inspection):**
- `test_orm_bid_decision_importable` — `from client_api.models import BidDecision` succeeds
- `test_orm_bid_decision_schema_is_client` — `BidDecision.__table__.schema == "client"`
- `test_orm_bid_decision_columns` — all 12 columns present (5 original + 7 from AC1)
- `test_orm_bid_decision_check_constraint_name` — `ck_bid_decisions_decision` in `__table__.constraints`
- `test_orm_bid_decision_unique_constraint_company_opportunity` — `uq_bid_decisions_company_opportunity`
  in constraints
- `test_orm_bid_decision_tablename` — `__tablename__ == "bid_decisions"`
- `test_orm_bid_decision_removed_from_env_excluded` — `"bid_decisions"` NOT in
  `alembic/env.py::_EXCLUDED_TABLE_NAMES`; `"bid_outcomes"` and `"bid_preparation_logs"` still excluded
- `test_orm_bid_decision_in_models_init` — `BidDecision` exported from `client_api.models`

**AC3 — Pydantic schemas (Pydantic validation):**
- `test_schema_bid_decision_type_enum` — `bid | no_bid | conditional` present; `pending` absent
- `test_schema_scorecard_requires_all_five_dimensions` — missing any dimension key → `ValidationError`
- `test_schema_scorecard_score_range_1_to_10` — score=0 or score=11 → `ValidationError`; 1/5/10 valid
- `test_schema_scorecard_confidence_0_to_1` — confidence=-0.1 or 1.1 → `ValidationError`; 0.0/0.5/1.0 valid
- `test_schema_request_override_justification_max_5000` — 5001 chars → `ValidationError`; 5000 valid
- `test_schema_bid_decision_response_from_attributes` — `model_config["from_attributes"] == True`
- `test_schema_evaluate_response_fields` — `opportunity_id`, `scorecard`, `evaluated_at` in fields
- `test_schema_scorecard_dimension_rationale_min_length_1` — empty rationale `""` → `ValidationError`
- `test_schema_scorecard_rejects_extra_dimension_keys` — unknown dimension key → `ValidationError`

**AC6 — Upsert semantics (SQL inspection):**
- `test_service_upsert_evaluate_uses_on_conflict_do_update` — compiled SQL contains `ON CONFLICT`;
  service exposes `upsert_evaluate` or `_build_evaluate_upsert` helper
- `test_service_upsert_decide_uses_on_conflict_do_update` — decide helper exists; if `_build_decide_upsert`
  exposed, compiled `set_` clause must NOT contain `ai_recommendation` or `evaluated_at`

**AC4 — Historical context fallback (mocked AsyncSession):**
- `test_service_similar_opportunities_fallback_on_error` — CPV query raises → `build_historical_context`
  falls back gracefully; returns dict with `win_count`, `similar_opportunities_count`; does not re-raise

**AC8 — Agent registry:**
- `test_agent_registry_contains_bid_no_bid_decision` — `agents.yaml` loads; `agents_map` contains
  `bid-no-bid-decision`; entry has `kraftdata_id`, `type == "agent"`, `description` mentions strategic
  dimensions, `timeout_override == 120`

---

### API Tests

#### `tests/api/test_bid_decisions.py` (25 tests)
Full HTTP-layer tests via `httpx.AsyncClient` + `bid_decision_ctx` fixture + `respx.MockRouter`.

**AC4 — POST /evaluate happy paths:**
- `test_evaluate_success_returns_200_with_scorecard` — mock gateway success → 200; `opportunity_id`,
  `scorecard.recommendation`, `evaluated_at` in response; DB row has `decision='pending'`,
  `ai_recommendation` JSONB populated
- `test_evaluate_agent_called_with_correct_payload` — inspects `respx` recorded request body;
  asserts `opportunity`, `company_profile`, `historical_context` keys with correct DB-fixture values
- `test_evaluate_preserves_decision_on_re_run` — seed row with `decision='bid'`, `user_override=True`;
  POST evaluate → 200; `decision`/`decided_by`/`decided_at`/`user_override` unchanged; `ai_recommendation`
  updated to new scorecard

**AC4 — POST /evaluate gate failures:**
- `test_evaluate_unauthenticated_returns_401` — no token → 401
- `test_evaluate_contributor_role_returns_403` — `contributor` role → 403 (maps to
  `test_evaluate_technical_writer_role_returns_403` in AC10 spec; fixture uses `contributor` role per
  Known Deviations fix)
- `test_evaluate_non_existent_opportunity_returns_404` — unknown UUID → 404

**AC4 — POST /evaluate agent failure handling:**
- `test_evaluate_gateway_timeout_returns_503` — `httpx.TimeoutException` → 503
  `{"message": "AI features are temporarily unavailable...", "code": "AGENT_UNAVAILABLE"}`; DB row count
  unchanged (covers `test_evaluate_no_row_created_on_gateway_failure`)
- `test_evaluate_gateway_503_returns_503_with_standard_body` — upstream 503 → 503 with `AGENT_UNAVAILABLE`
- `test_evaluate_gateway_malformed_response_returns_503` — missing `dimensions` key → 503; no row update
- `test_evaluate_existing_row_ai_recommendation_not_mutated_on_failure` — seed row with ai_recommendation=X;
  mock timeout → 503; row.ai_recommendation still X

**AC5 — POST /bid-decision happy paths:**
- `test_decide_without_prior_evaluation_persists_decision` — no prior `/evaluate` → 201; `decision='bid'`,
  `ai_recommendation IS NULL`, `user_override=False`, `decided_by` populated
- `test_decide_aligned_with_ai_no_override` — seed row with `ai_recommendation.recommendation='bid'`;
  decide `decision='bid'` → 200/201; `user_override=False`
- `test_decide_against_ai_requires_justification` — seed row with `recommendation='bid'`; decide
  `decision='no_bid'` without justification → 422 `"Override justification is required..."`; DB row
  decision unchanged (`pending`)
- `test_decide_against_ai_with_justification_succeeds` — decide `decision='no_bid'`,
  `override_justification='Client pulled out'` → 200; `user_override=True`; justification persisted
- `test_decide_preserves_ai_recommendation_on_update` — seed row with `ai_recommendation` + fixed
  `evaluated_at='2026-01-15...'`; decide → 200; `ai_recommendation` and `evaluated_at` unchanged;
  `decision`/`decided_at` updated

**AC5 — POST /bid-decision gate failures:**
- `test_decide_soft_deleted_opportunity_returns_404` — `deleted_at` non-null → 404 on both POST evaluate
  AND POST decide
- `test_decide_invalid_decision_value_returns_422` — `decision='maybe'` → 422 Pydantic enum
- `test_decide_override_justification_over_5000_chars_returns_422` — 5001-char justification → 422

**AC7 — GET /bid-decision:**
- `test_get_returns_200_with_full_row` — seed fully-populated row → 200; all 12 `BidDecisionResponse`
  fields present (`id`, `company_id`, `opportunity_id`, `decision`, `ai_recommendation`, `user_override`,
  `override_justification`, `decided_by`, `evaluated_at`, `decided_at`, `created_at`, `updated_at`)
- `test_get_no_row_returns_404` — opportunity exists but no decision row for current company → 404
  `"No bid decision recorded for this opportunity"` (uses Company B token to avoid cross-test pollution)
- `test_get_cross_company_returns_404` — Company B GETs a row owned by Company A → 404 (existence-leakage
  safe; the row exists but `company_id` does not match caller)

**AC4/AC5 — Audit trail:**
- `test_evaluate_writes_audit_row` — `shared.audit_log` row with `entity_type='bid_decision'`,
  `action_type in ("create","update")`, `after` contains `opportunity_id`, `ai_recommendation_recommendation`,
  `evaluated_at`
- `test_decide_writes_audit_row` — audit row with `action_type` reflecting insert vs update, `after`
  contains `opportunity_id`, `decision`, `user_override`, `decided_by`

**AC9 — Router wiring:**
- `test_router_paths_exist` — all three paths return non-404 for unauthenticated requests (401 = route
  exists)
- `test_router_no_doubled_prefix` — inspects `app.routes`; asserts no `/opportunities/opportunities/`
  double-prefix (sync test — no DB required)

---

## Step 4c: Aggregate

All test files confirmed on disk under `eusolicit-app/services/client-api/tests/`.

### File List

| File | Phase | Test Count |
|---|---|---|
| `tests/integration/test_044_migration.py` | GREEN | 1 class / 5 sub-scenarios |
| `tests/unit/test_bid_decision_service.py` | GREEN | 23 |
| `tests/api/test_bid_decisions.py` | GREEN | 25 |
| **Total** | | **49** |

> **Note on TDD Phase:** The story was developed through a full dev-story + two code-review rounds
> before this ATDD checklist was generated. All tests are GREEN-phase — they are expected to PASS
> against the existing implementation. The initial test suite was generated as RED-phase scaffolds
> (no `@pytest.mark.skip` decorators were used; the story used inline docstrings instead), then the
> tests were corrected for environment-specific issues (Known Deviations). The follow-up code review
> verified that all AC contracts are implemented and returned Verdict: Approve.

### AC → Test Coverage Mapping

| AC | Test File(s) | Coverage |
|---|---|---|
| AC1 (migration 044) | `test_044_migration.py` — 5 sub-scenarios in 1 method | ✅ Full |
| AC2 (ORM model) | `test_bid_decision_service.py` × 8 import-inspection tests | ✅ Full |
| AC3 (Pydantic schemas) | `test_bid_decision_service.py` × 9 validation tests | ✅ Full |
| AC4 (POST /evaluate) | `test_bid_decisions.py` × 10; `test_bid_decision_service.py` × 3 | ✅ Full |
| AC5 (POST /bid-decision) | `test_bid_decisions.py` × 8 | ✅ Full |
| AC6 (upsert semantics) | `test_bid_decision_service.py` × 3 (2 SQL inspection + 1 fallback) | ✅ Full |
| AC7 (GET /bid-decision) | `test_bid_decisions.py` × 3 | ✅ Full |
| AC8 (agent registry) | `test_bid_decision_service.py` × 1 | ✅ Full |
| AC9 (router wiring) | `test_bid_decisions.py` × 2 | ✅ Full |
| AC10 (≥ 28 scenarios) | All files — 48+ scenarios | ✅ Far exceeds minimum |

### AC10 Explicit Scenario Coverage Matrix

| AC10 Scenario | Covered By | Status |
|---|---|---|
| `test_migration_044_adds_runtime_columns` | `test_044_migration.py::TestMigration044::test_upgrade_044_lifecycle` | ✅ |
| `test_migration_044_check_constraint_rejects_invalid_decision` | `test_upgrade_044_lifecycle` (decision='maybe' raises) | ✅ |
| `test_migration_044_unique_constraint_on_company_opportunity` | `test_upgrade_044_lifecycle` (duplicate insert raises) | ✅ |
| `test_migration_044_downgrade_reverses` | `test_upgrade_044_lifecycle` (downgrade removes columns) | ✅ |
| `test_migration_044_revision_chain` | `test_bid_decision_service.py::test_migration_044_revision_chain` | ✅ |
| `test_orm_bid_decision_importable` | `test_bid_decision_service.py::test_orm_bid_decision_importable` | ✅ |
| `test_orm_bid_decision_removed_from_env_excluded` | `test_bid_decision_service.py::test_orm_bid_decision_removed_from_env_excluded` | ✅ |
| `test_schema_bid_decision_type_enum` | `test_bid_decision_service.py::test_schema_bid_decision_type_enum` | ✅ |
| `test_schema_scorecard_requires_all_five_dimensions` | `test_bid_decision_service.py::test_schema_scorecard_requires_all_five_dimensions` | ✅ |
| `test_schema_scorecard_score_range_1_to_10` | `test_bid_decision_service.py::test_schema_scorecard_score_range_1_to_10` | ✅ |
| `test_schema_scorecard_confidence_0_to_1` | `test_bid_decision_service.py::test_schema_scorecard_confidence_0_to_1` | ✅ |
| `test_schema_request_override_justification_max_5000` | `test_bid_decision_service.py::test_schema_request_override_justification_max_5000` | ✅ |
| `test_evaluate_success_returns_200_with_scorecard` | `test_bid_decisions.py::test_evaluate_success_returns_200_with_scorecard` | ✅ |
| `test_evaluate_agent_called_with_correct_payload` | `test_bid_decisions.py::test_evaluate_agent_called_with_correct_payload` | ✅ |
| `test_decide_without_prior_evaluation_persists_decision` | `test_bid_decisions.py::test_decide_without_prior_evaluation_persists_decision` | ✅ |
| `test_decide_aligned_with_ai_no_override` | `test_bid_decisions.py::test_decide_aligned_with_ai_no_override` | ✅ |
| `test_decide_against_ai_requires_justification` | `test_bid_decisions.py::test_decide_against_ai_requires_justification` | ✅ |
| `test_decide_against_ai_with_justification_succeeds` | `test_bid_decisions.py::test_decide_against_ai_with_justification_succeeds` | ✅ |
| `test_decide_preserves_ai_recommendation_on_update` | `test_bid_decisions.py::test_decide_preserves_ai_recommendation_on_update` | ✅ |
| `test_evaluate_preserves_decision_on_re_run` | `test_bid_decisions.py::test_evaluate_preserves_decision_on_re_run` | ✅ |
| `test_get_returns_200_with_full_row` | `test_bid_decisions.py::test_get_returns_200_with_full_row` | ✅ |
| `test_get_no_row_returns_404` | `test_bid_decisions.py::test_get_no_row_returns_404` | ✅ |
| `test_evaluate_non_existent_opportunity_returns_404` | `test_bid_decisions.py::test_evaluate_non_existent_opportunity_returns_404` | ✅ |
| `test_decide_soft_deleted_opportunity_returns_404` | `test_bid_decisions.py::test_decide_soft_deleted_opportunity_returns_404` | ✅ |
| `test_evaluate_unauthenticated_returns_401` | `test_bid_decisions.py::test_evaluate_unauthenticated_returns_401` | ✅ |
| `test_evaluate_technical_writer_role_returns_403` | `test_bid_decisions.py::test_evaluate_contributor_role_returns_403` (role renamed per Known Deviations fix) | ✅ |
| `test_get_cross_company_returns_404` | `test_bid_decisions.py::test_get_cross_company_returns_404` | ✅ |
| `test_decide_invalid_decision_value_returns_422` | `test_bid_decisions.py::test_decide_invalid_decision_value_returns_422` | ✅ |
| `test_decide_override_justification_over_5000_chars_returns_422` | `test_bid_decisions.py::test_decide_override_justification_over_5000_chars_returns_422` | ✅ |
| `test_evaluate_gateway_timeout_returns_503` | `test_bid_decisions.py::test_evaluate_gateway_timeout_returns_503` | ✅ |
| `test_evaluate_gateway_503_returns_503_with_standard_body` | `test_bid_decisions.py::test_evaluate_gateway_503_returns_503_with_standard_body` | ✅ |
| `test_evaluate_gateway_malformed_response_returns_503` | `test_bid_decisions.py::test_evaluate_gateway_malformed_response_returns_503` | ✅ |
| `test_evaluate_no_row_created_on_gateway_failure` | `test_evaluate_gateway_timeout_returns_503` (includes pre/post count assertion) | ✅ |
| `test_evaluate_existing_row_ai_recommendation_not_mutated_on_failure` | `test_bid_decisions.py::test_evaluate_existing_row_ai_recommendation_not_mutated_on_failure` | ✅ |
| `test_service_upsert_evaluate_uses_on_conflict_do_update` | `test_bid_decision_service.py::test_service_upsert_evaluate_uses_on_conflict_do_update` | ✅ |
| `test_service_upsert_decide_uses_on_conflict_do_update` | `test_bid_decision_service.py::test_service_upsert_decide_uses_on_conflict_do_update` | ✅ |
| `test_service_similar_opportunities_fallback_on_error` | `test_bid_decision_service.py::test_service_similar_opportunities_fallback_on_error` | ✅ |
| `test_evaluate_writes_audit_row` | `test_bid_decisions.py::test_evaluate_writes_audit_row` | ✅ |
| `test_decide_writes_audit_row` | `test_bid_decisions.py::test_decide_writes_audit_row` | ✅ |
| `test_router_paths_exist` | `test_bid_decisions.py::test_router_paths_exist` | ✅ |
| `test_router_no_doubled_prefix` | `test_bid_decisions.py::test_router_no_doubled_prefix` | ✅ |

**All 41 AC10 explicit scenarios are covered.** ✅ (exceeds ≥ 28 minimum by 13 scenarios)

---

## Step 5: Validate & Complete

### 1. Validation

| Check | Status | Notes |
|---|---|---|
| Prerequisites satisfied | ✅ | All Story 10.10 dependencies confirmed present |
| Test files created | ✅ | 3 files: 25 API + 23 unit + 1 integration class |
| Checklist matches acceptance criteria | ✅ | All 10 ACs mapped; 41/41 AC10 scenarios covered |
| TDD phase documented | ✅ | GREEN (post-implementation, post-code-review) |
| Story metadata and handoff paths captured | ✅ | Frontmatter populated |
| CLI sessions cleaned up | N/A | Backend testing — no browser sessions |
| Temp artifacts stored in `test_artifacts/` | ✅ | Checklist at correct path |
| Code review deviations accounted for | ✅ | AC5 201/200 contract, AND semantics, racy detection, tz defaults all noted |
| All AC10 explicit scenarios covered | ✅ | 41/41 scenarios |
| respx mock pattern consistent with Story 11.3 | ✅ | `respx.mock(base_url=...)` context manager per `test_espd_autofill_export.py` |
| Cross-company isolation returns 404 not 403 | ✅ | `test_get_cross_company_returns_404` asserts 404 |
| Upsert field isolation tested | ✅ | Evaluate preserves decision fields; decide preserves ai_recommendation/evaluated_at |

### 2. Completion Summary

- **Test files created:** 3 files, 49 test scenarios (25 API + 23 unit + 1 integration class / 5 sub-scenarios)
- **Checklist output path:** `test_artifacts/atdd-checklist-10-10-bid-no-bid-decision-api-ai-integration.md`
- **Story key:** `10-10-bid-no-bid-decision-api-ai-integration`
- **Story file handoff path:** `eusolicit-docs/implementation-artifacts/10-10-bid-no-bid-decision-api-ai-integration.md`
- **TDD Phase:** 🟢 GREEN — implementation complete and code-reviewed (Verdict: Approve, 2026-04-21)
- **Key risks / assumptions:**
  - The `bid_decision_ctx` fixture uses Company A's `admin` user as the primary token for most
    evaluate/decide tests — Company A admin is the `admin` role (which passes `require_role("bid_manager")`
    as `admin` is in the allow-list). This was confirmed as correct per the Known Deviations fix
    (earlier version used `technical_writer` which fails the gate).
  - `superuser_session_factory` is required to seed `pipeline.opportunities` because `client_api_role`
    has no INSERT permission on `pipeline.opportunities`. Tests that seed fresh opportunities must use
    `ctx["su_session"]` (confirmed as the pattern in the existing test file).
  - The `test_evaluate_no_row_created_on_gateway_failure` scenario from AC10 is satisfied by
    `test_evaluate_gateway_timeout_returns_503` which includes a pre/post COUNT assertion. It is not
    a separate test function.
  - `test_decide_aligned_with_ai_no_override` and several other update-path tests assert
    `status_code in (200, 201)` rather than `== 200` specifically. The second code review noted this
    as a non-blocking follow-up opportunity: tightening to `== 200` on update paths would pin the
    AC5 201/200 contract more firmly. This is tracked as a non-blocking observation from the code review.
  - `test_service_upsert_decide_uses_on_conflict_do_update` uses a compiled-SQL string comparison to
    assert `ai_recommendation` absent from the `set_` clause. The code review flagged this as fragile
    to dialect-rendering changes; a structural `stmt.dialect_options["postgresql"]["on_conflict_update_set"]`
    assertion would be more robust in a follow-up.
  - The integration test `TestMigration044::test_upgrade_044_lifecycle` requires `migration_role`
    credentials and network access to the test DB. It is gated by `@pytest.mark.integration` and will
    be skipped in `make test-unit` runs (correct isolation).
- **Known code-review deviations resolved in implementation:**
  1. AC5 201/200 contract — fixed: `record_decision` returns `(BidDecisionResponse, was_insert)` tuple;
     route returns `JSONResponse(status_code=201 if was_insert else 200)`.
  2. `build_historical_context` AND semantics — fixed: primary query uses
     `opportunity_type = :opp_type AND cpv_codes && :cpv_codes`.
  3. Racy create/update detection — fixed: pre-fetch row existence before upsert.
  4. Timezone defaults — fixed: `default=lambda: datetime.now(UTC)` in ORM model.
  5. LIMIT 1000 on similar-opportunities scan — implemented.
  6. `logger.warning(...)` used consistently; `or_` import removed.
- **Next recommended workflow:** Story 10.11 (Bid Outcome & Lessons Learned Integration) is the next
  dependency — it assumes a `bid_decisions` row exists for `(company_id, opportunity_id)` when outcome
  recording happens, a guarantee delivered by this story's insert-on-first-call pattern.
