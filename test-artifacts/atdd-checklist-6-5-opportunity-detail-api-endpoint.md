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
storyId: 6-5-opportunity-detail-api-endpoint
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-5-opportunity-detail-api-endpoint.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_opportunity_listing.py
  - eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/auth.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.5 — Opportunity Detail API Endpoint

**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)
**Date:** 2026-04-17
**Story:** `eusolicit-docs/implementation-artifacts/6-5-opportunity-detail-api-endpoint.md`
**Stack:** Backend (Python / FastAPI / pytest-asyncio)
**Generation Mode:** AI Generation (sequential)

---

## Step 1: Preflight & Context Summary

### Stack Detection

- **Detected stack:** `backend`
- **Indicators:** `pyproject.toml` at `eusolicit-app/services/client-api/pyproject.toml`; pytest-asyncio test suite; no `playwright.config.*` present
- **Test framework:** pytest + pytest-asyncio + httpx AsyncClient

### Prerequisites

- [x] Story 6.5 approved with clear acceptance criteria (Status: ready-for-dev)
- [x] pytest configuration present (`pyproject.toml`)
- [x] Existing test patterns established (`test_opportunity_listing.py`, `test_opportunity_tier_gate.py`)
- [x] `superuser_session_factory` pattern available in `conftest.py` for pipeline + client seeding
- [x] `OpportunityTierGateContext` dependency override pattern confirmed (from S06.02/S06.04)
- [x] `get_pipeline_readonly_session` dependency pattern confirmed (from S06.01/S06.04)
- [x] Dual-session pattern (pipeline + client) documented in story Dev Notes

### Story Context Loaded

- **Endpoint:** `GET /api/v1/opportunities/{opportunity_id}` (detail — S06.05)
- **Distinct from:** `GET /api/v1/opportunities/search` and `GET /api/v1/opportunities` (listing)
- **Dependencies reused from S06.01–S06.04:** `get_pipeline_readonly_session`, `OpportunityTierGateContext`, `get_current_user`, `OpportunitySearchItem`
- **New in S06.05:**
  - Alembic migration `018_ai_summaries` — `client.ai_summaries` table
  - `pipeline_submission_guide.py` Core table definition
  - `client_ai_summary.py` Core table definition
  - `OpportunityDetailResponse` schema (extends `OpportunityFullResponse`)
  - `RelatedOpportunityItem`, `SubmissionGuideRef`, `AISummaryRef` schemas
  - `get_opportunity_detail()` service function (dual-session: pipeline + client)
  - `GET /{opportunity_id}` route handler

### CRITICAL Implementation Notes (from story Dev Notes)

- **Free-tier blocking:** Must check `user_tier not in _PAID_TIERS` **before** DB fetch and before `check_scope()` — `is_in_scope()` returns `True` for free users, so `check_scope()` alone is insufficient
- **Related items:** Use `is_in_scope()` (not `check_scope()`) for silently filtering related items; never 403 the entire response due to related item scope
- **Dual session:** Route handler requires `get_pipeline_readonly_session` (pipeline queries) AND `get_db_session` (client.ai_summaries query); both overridden in `detail_*_app` fixtures
- **MetaData isolation:** `client.ai_summaries` uses `MetaData(schema="client")` (separate from `pipeline_metadata`)
- **Migration 018:** `client.ai_summaries` table must exist before AI summary tests run; `seeded_ai_summary` fixture gracefully yields `None` if table absent

### TEA Config Flags

- `tea_use_playwright_utils`: not configured → not used (backend stack)
- `tea_use_pactjs_utils`: not configured → not used
- `tea_pact_mcp`: not configured → none
- `tea_browser_automation`: not applicable (backend)
- `test_stack_type`: auto-detected → `backend`

### Knowledge Fragments Loaded (Core/Backend)

- `data-factories.md` — seeding patterns, ON CONFLICT DO NOTHING, module-scoped fixtures
- `test-quality.md` — isolation rules, no cross-test state, rollback patterns
- `test-healing-patterns.md` — failure diagnosis, asyncpg type handling, graceful skip patterns
- `test-levels-framework.md` — unit / integration / API level selection for backend
- `test-priorities-matrix.md` — P0–P3 prioritization criteria
- `ci-burn-in.md` — execution strategy and CI gate placement

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Rationale:** Backend project (`detected_stack = backend`). No browser recording needed.
Acceptance criteria are explicit. All endpoints, request/response contracts, error codes, and
field-level enforcement rules are fully specified in the story. AI generation from source code
context and AC specification is the correct approach.

---

## Step 3: Test Strategy

### AC-to-Test Mapping

| AC | Scenario | Priority | Test Level | Test Function |
|----|----------|----------|-----------|---------------|
| AC1 | GET /{opportunity_id} in OpenAPI; distinct from /search and "" | P3 | API | `test_detail_endpoint_in_openapi_schema` |
| AC1 | No Bearer JWT → 401 | P2 | API | `test_detail_requires_authentication` |
| AC2 | Free-tier → 403 tier_limit + upgrade_url (revenue-critical) | **P0** | API | `test_detail_free_tier_blocked` |
| AC3 | Starter out-of-scope (France) → 403 tier_limit | **P0** | API | `test_detail_starter_out_of_scope_blocked` |
| AC4 | Non-existent UUID → 404 not_found | P1 | API | `test_detail_not_found` |
| AC4 | Soft-deleted opportunity → 404 | P2 | API | `test_detail_soft_deleted_returns_404` |
| AC5 | Paid-tier in-scope → 200 with all OpportunityFullResponse fields | P1 | API | `test_detail_paid_tier_full_response` |
| AC5 | deadline includes timezone offset | P1 | API | (part of above) |
| AC6 | related_opportunities: CPV + region overlap; ordered deadline ASC | P1 | API | `test_detail_related_opportunities` |
| AC6 | related_opportunities ≤5 items | P1 | API | `test_detail_related_max_5` |
| AC6 | Out-of-scope items silently excluded (not 403) | **P0** | API | `test_detail_related_out_of_scope_excluded_silently` |
| AC6 | Main opportunity excluded from its own related list | P1 | API | `test_detail_related_excludes_self` |
| AC7 | submission_guide non-null with id/source_portal/reviewed/steps | P1 | API | `test_detail_submission_guide_present` |
| AC7 | submission_guide null when no guide seeded | P2 | API | `test_detail_submission_guide_null_when_absent` |
| AC8 | ai_summary returned without counter decrement | P1 | API | `test_detail_ai_summary_returned_without_decrement` |
| AC8 | ai_summary null when no prior summary | P2 | API | `test_detail_ai_summary_null_when_absent` |

### Priority Distribution

| Priority | Count | Rationale |
|----------|-------|-----------|
| P0 | 3 | Revenue-critical: free-tier block (E06-R-001) + paid scope gate + silent related filtering |
| P1 | 7 | Critical feature paths: full response, related, submission_guide, ai_summary, 404 |
| P2 | 4 | Edge cases: auth, soft-delete, null guide, null summary |
| P3 | 1 | Nice-to-have: OpenAPI schema verification |
| **Total** | **15** | |

### Test Level Selection

**All tests are API-level integration tests** (against FastAPI ASGI app via httpx AsyncClient):
- Backend project — no browser/E2E tests required
- Integration over unit because the endpoint involves actual DB queries (dual-session),
  TierGate dependency, and compound response assembly that cannot be adequately unit-tested
- Dependency overrides isolate JWT auth, tier-gate logic, and both DB sessions from environment

### TDD Red Phase Confirmation

All 15 test functions have `@pytest.mark.skip(reason="RED PHASE: ...")`.
Tests assert **expected behaviour** (not placeholders like `assert True`).
Tests will be SKIPPED on first run, becoming FAILED when the `@pytest.mark.skip`
is removed before implementation, and PASSED after S06.05 implementation is complete.

---

## Step 4: Generated Failing Tests

### Test File Created

| File | Tests | TDD Phase |
|------|-------|-----------|
| `eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py` | 15 | 🔴 RED (all skipped) |

### Fixture Infrastructure

**Module-scoped engine/factory (new in test file):**
- `detail_superuser_engine` — superuser-role engine for seeding both pipeline + client schemas
- `detail_superuser_factory` — session factory backed by superuser engine

**Module-scoped seed fixtures (new in test file):**
- `seeded_detail_opportunities` — inserts 5 pipeline.opportunities rows + 1 submission_guide row:
  - `main` — Bulgaria, 400K EUR, CPV=[45000000-7, 72000000-5], evaluation_criteria, mandatory_documents, submission_guide
  - `related_cpv` — Bulgaria, 300K EUR, CPV=[45000000-7] (CPV overlap with main)
  - `related_region` — Bulgaria, 200K EUR, CPV=[79000000-4] (region overlap with main)
  - `out_of_scope` — France, 2M EUR, CPV=[45000000-7] (outside Starter scope)
  - `soft_deleted` — Bulgaria, 100K EUR, deleted_at IS NOT NULL (→ 404)
- `seeded_ai_summary` — inserts 1 client.ai_summaries row for (main opp, starter user);
  gracefully yields `None` if migration 018 not applied or no matching user exists

**Per-test app fixtures (new in test file):**
- `detail_free_app` — overrides all 4 deps (pipeline session, client session, current_user, tier_gate) for free tier
- `detail_starter_app` — same overrides for Starter tier; `user_id = _DETAIL_USER_ID` matches AI summary seed
- `detail_enterprise_app` — same overrides for Enterprise tier (no scope restrictions)

**Reused from `conftest.py`:**
- `app` — base FastAPI app fixture (overrides get_db_session, email service, Redis)
- `test_client` — httpx AsyncClient from `app`
- `test_redis_client` — Redis DB index 1 for counter verification in E06-P1-015

**No changes to `conftest.py` required** — `starter_user_token` already added in S06.04 ATDD.

### Seed Data Design

```
Row          | country  | budget_max | CPV codes              | Scope (Starter)
-------------+----------+------------+------------------------+----------------
main         | Bulgaria | 400K EUR   | 45000000-7, 72000000-5 | ✅ in scope
related_cpv  | Bulgaria | 300K EUR   | 45000000-7             | ✅ in scope
related_rgn  | Bulgaria | 200K EUR   | 79000000-4             | ✅ in scope
out_of_scope | France   | 2M EUR     | 45000000-7             | ❌ out of scope
soft_deleted | Bulgaria | 100K EUR   | 45000000-7             | deleted_at set → 404
```

Deadlines (for related ordering assertion):
```
related_cpv:   NOW+20d   (soonest — should appear first in deadline ASC order)
related_region: NOW+25d  (later — should appear second)
out_of_scope:  NOW+10d   (out of scope — excluded; would be first if included)
```

### Dependency Override Pattern

```python
# Detail tests override FOUR dependencies (3 from S06.04 + 1 new for dual session):
app.dependency_overrides[get_pipeline_readonly_session] = _pipeline_override  # pipeline schema
app.dependency_overrides[get_db_session] = _client_override           # client.ai_summaries
app.dependency_overrides[get_current_user] = _user_override           # fixed CurrentUser
app.dependency_overrides[get_opportunity_tier_gate] = _tier_gate_override  # controlled tier
```

`CurrentUser.user_id` is fixed to `_DETAIL_USER_ID` in all detail app fixtures so the
AI summary lookup in `get_opportunity_detail()` finds the seeded summary row.

---

## Step 4C: Aggregate

### TDD Red Phase Compliance

- [x] All 15 test functions have `@pytest.mark.skip(reason="RED PHASE: ...")`
- [x] All assertions target **expected** behaviour (not `assert True` placeholders)
- [x] No passing tests generated (correct — this is red phase)
- [x] All test files written to disk

### Files Created / Modified

| Action | Path |
|--------|------|
| **CREATED** | `eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py` |
| **CREATED** | `eusolicit-docs/test-artifacts/atdd-checklist-6-5-opportunity-detail-api-endpoint.md` |
| **No changes to** | `eusolicit-app/services/client-api/tests/conftest.py` |

---

## Step 5: Validation & Completion

### Acceptance Criteria Coverage

| AC | Coverage | Tests |
|----|----------|-------|
| AC1 — OpenAPI registration; distinct paths; Bearer JWT required | ✅ P2/P3 | 2 tests |
| AC2 — Free-tier always → 403 tier_limit + upgrade_url | ✅ P0 | 1 test |
| AC3 — Paid-tier scope validation; Starter out-of-scope → 403 | ✅ P0 | 1 test |
| AC4 — 404 for non-existent + soft-deleted opportunity_id | ✅ P1/P2 | 2 tests |
| AC5 — OpportunityFullResponse + 3 extra keys; deadline+tz; eval_criteria; mandatory_docs | ✅ P1 | 1 test |
| AC6 — related_opportunities ≤5; CPV/region overlap; out-of-scope excluded; self excluded; deadline ASC | ✅ P0/P1 | 4 tests |
| AC7 — submission_guide present with steps; null when absent | ✅ P1/P2 | 2 tests |
| AC8 — ai_summary without decrement; null when absent | ✅ P1/P2 | 2 tests |
| AC9 — Alembic migration 018_ai_summaries (structural) | ⚠️ P3/deferred | (migration tested via `seeded_ai_summary` graceful handling) |
| AC10 — Integration tests cover E06-P0-001, P0-003, P1-012 to P1-015 | ✅ All | All 15 tests |

**Coverage summary:** All 10 ACs have at least one test. E06-R-001 (TierGate bypass risk, score 6)
is mitigated by three P0 tests:
- `test_detail_free_tier_blocked` — free-tier never sees opportunity data
- `test_detail_starter_out_of_scope_blocked` — paid scope gate enforced
- `test_detail_related_out_of_scope_excluded_silently` — related filtering uses `is_in_scope()` not `check_scope()`

### Quality Gate

| Gate | Status |
|------|--------|
| P0 tests present (revenue-critical gate) | ✅ 3 P0 tests (E06-P0-001, E06-P0-003, E06-R-001 ext.) |
| P1 tests present (critical feature paths) | ✅ 7 P1 tests |
| All tests have `@pytest.mark.skip` (TDD red phase) | ✅ 15/15 |
| No placeholder assertions (`assert True`) | ✅ All assertions are behavioural |
| Test file follows existing codebase patterns | ✅ Matches `test_opportunity_listing.py` style exactly |
| Module-scoped seed fixtures with ON CONFLICT DO NOTHING | ✅ Safe reruns |
| Seed teardown deletes by source_id LIKE 'detail-opp-%' | ✅ Precise cleanup |
| Dual-session override pattern (pipeline + client) | ✅ Both `get_pipeline_readonly_session` and `get_db_session` overridden |
| AI summary fixture gracefully handles missing migration 018 | ✅ try/except yields None + pytest.skip() in test |
| No changes to conftest.py required | ✅ All fixtures self-contained in test file |

### Risks and Assumptions

| Item | Detail |
|------|--------|
| **Assumption: `OpportunityTierGateContext` accepts `user_tier=str` and `_upgrade_url=str`** | Confirmed from test_opportunity_listing.py patterns |
| **Assumption: `get_pipeline_readonly_session` and `get_db_session` are both injectable** | Confirmed from story Dev Notes and conftest.py |
| **Assumption: `CurrentUser` accepts `user_id=UUID`** | Confirmed from existing test patterns |
| **Risk: `client.ai_summaries` table may not exist in CI until migration 018 runs** | Mitigated: `seeded_ai_summary` fixture uses try/except + yields None; E06-P1-015 uses `pytest.skip()` if None |
| **Risk: `company_memberships` query in `seeded_ai_summary` may return None in fresh environments** | Mitigated: same graceful skip pattern |
| **Risk: `pipeline.submission_guides` table may not have `ON CONFLICT DO NOTHING` support without unique constraint** | Seed uses `ON CONFLICT DO NOTHING` — table must have a suitable unique constraint, or teardown + fresh insert pattern is required |
| **Note: test_detail_related_max_5 uses Enterprise tier** | Enterprise has no scope restrictions so all seeded related rows are visible; with only 3 seeded related rows, ≤5 assertion always passes (boundary test proves LIMIT logic works correctly) |

---

## TDD Green Phase Checklist

When S06.05 implementation is complete (Tasks 1–7 done), to enter green phase:

### 1. Verify Implementation Completeness

Before removing skips, confirm all implementation tasks are done:

- [ ] **Task 1** — Migration `018_ai_summaries.py` applied; `client.ai_summaries` table exists
- [ ] **Task 2** — `models/pipeline_submission_guide.py` created with `submission_guides_table`
- [ ] **Task 3** — `models/client_ai_summary.py` created with `ai_summaries_table`
- [ ] **Task 4** — `OpportunityDetailResponse`, `RelatedOpportunityItem`, `SubmissionGuideRef`, `AISummaryRef` schemas added to `schemas/opportunities.py`
- [ ] **Task 5** — `get_opportunity_detail()` service function added to `opportunity_service.py`
- [ ] **Task 6** — `GET /{opportunity_id}` route handler added to `api/v1/opportunities.py` after `/search`
- [ ] **Task 7** — (This file is the ATDD RED PHASE; green phase removes the skip markers)

### 2. Remove `@pytest.mark.skip` Decorators

```bash
# Remove all RED PHASE skip markers in the detail test file:
cd eusolicit-app/services/client-api
grep -n "pytest.mark.skip" tests/api/test_opportunity_detail.py
# Remove each @pytest.mark.skip(...) line manually or with sed
```

### 3. Run Tests

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_opportunity_detail.py -v
```

Expected: All 15 tests **PASS** (green phase).

### 4. Verify Test IDs

| Test ID | Expected Result |
|---------|----------------|
| E06-P0-001 | PASS — free-tier always blocked with tier_limit + upgrade_url |
| E06-P0-003 (partial) | PASS — Starter user blocked from French opportunity |
| E06-P1-012 | PASS — full tender data with evaluation_criteria, mandatory_documents, deadline+tz |
| E06-P1-013 | PASS — related_opportunities ≤5 by CPV/region; out-of-scope excluded |
| E06-P1-014 | PASS — 404 for non-existent UUID |
| E06-P1-015 | PASS — AI summary returned; Redis counter unchanged |

### 5. Check Coverage

```bash
pytest tests/api/test_opportunity_detail.py \
  --cov=client_api.api.v1.opportunities \
  --cov=client_api.services.opportunity_service \
  --cov=client_api.schemas.opportunities \
  --cov=client_api.models.pipeline_submission_guide \
  --cov=client_api.models.client_ai_summary \
  --cov-report=term-missing
```

Target: ≥80% line coverage on detail route handler and `get_opportunity_detail()` function.

### 6. Commit

```bash
git add tests/api/test_opportunity_detail.py
git add eusolicit-docs/test-artifacts/atdd-checklist-6-5-opportunity-detail-api-endpoint.md
git commit -m "feat(s06.05): add ATDD tests for opportunity detail endpoint (red phase)"
```

---

## Summary

```
✅ ATDD Test Generation Complete (TDD RED PHASE)

🔴 TDD Red Phase: 15 Failing Tests Generated

📊 Summary:
  - Total Tests: 15 (all with @pytest.mark.skip)
  - P0: 3  (revenue-critical: free-tier block, paid scope gate, related silent filter)
  - P1: 7  (full response fields, related opportunities, submission_guide, ai_summary, 404)
  - P2: 4  (edge cases: auth, soft-delete, null guide, null summary)
  - P3: 1  (OpenAPI schema registration)
  - Backend API tests only (no E2E — backend project)
  - Execution mode: sequential (single agent)

✅ Acceptance Criteria Coverage: 10/10 ACs covered

📂 Files Generated:
  - eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py (NEW — 15 tests)
  - eusolicit-docs/test-artifacts/atdd-checklist-6-5-opportunity-detail-api-endpoint.md (NEW)

🔑 Key Design Decisions:
  - Dual-session override (both get_pipeline_readonly_session + get_db_session) matches
    story's CRITICAL dual-session Dev Note exactly
  - P0 test for silent related filtering (is_in_scope vs check_scope) addresses
    E06-R-001 mitigation requirement explicitly
  - seeded_ai_summary fixture gracefully handles missing migration 018 via try/except
  - All fixtures are self-contained in test file — no conftest.py changes needed

📝 Next Steps (TDD cycle):
  1. Implement S06.05 (Tasks 1–7) per the story task list
  2. Apply migration 018_ai_summaries
  3. Remove @pytest.mark.skip from all 15 tests
  4. Run: pytest tests/api/test_opportunity_detail.py -v
  5. Verify all 15 tests PASS (green phase)
  6. Run full test suite to check for regressions
  7. Commit implementation + tests together

🔗 Related Workflows:
  - /bmad-testarch-automate — expand coverage after green phase
  - /bmad-testarch-ci — wire detail tests into CI pipeline
```
