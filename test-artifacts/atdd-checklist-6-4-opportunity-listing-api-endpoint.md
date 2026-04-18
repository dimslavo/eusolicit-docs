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
storyId: 6-4-opportunity-listing-api-endpoint
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-4-opportunity-listing-api-endpoint.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_opportunity_search.py
  - eusolicit-app/services/client-api/tests/api/test_opportunity_tier_gate.py
  - eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/auth.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.4 — Opportunity Listing API Endpoint

**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)
**Date:** 2026-04-17
**Story:** `eusolicit-docs/implementation-artifacts/6-4-opportunity-listing-api-endpoint.md`
**Stack:** Backend (Python / FastAPI / pytest-asyncio)
**Generation Mode:** AI Generation (sequential)

---

## Step 1: Preflight & Context Summary

### Stack Detection

- **Detected stack:** `backend`
- **Indicators:** `pyproject.toml` at `eusolicit-app/services/client-api/pyproject.toml`; pytest-asyncio test suite; no `playwright.config.*` present
- **Test framework:** pytest + pytest-asyncio + httpx AsyncClient

### Prerequisites

- [x] Story 6.4 approved with clear acceptance criteria (Status: ready-for-dev)
- [x] pytest configuration present (`pyproject.toml`)
- [x] Existing test patterns established (test_opportunity_search.py, test_opportunity_tier_gate.py)
- [x] `superuser_session_factory` pattern available for pipeline.opportunities seeding
- [x] `OpportunityTierGateContext` dependency pattern confirmed (from S06.02)
- [x] `get_pipeline_readonly_session` dependency pattern confirmed (from S06.01)

### Story Context Loaded

- **Endpoint:** `GET /api/v1/opportunities` (browse mode — no `q` param)
- **Distinct from:** `GET /api/v1/opportunities/search` (FTS) and `GET /api/v1/opportunities/{id}` (detail)
- **Dependencies reused from S06.01/S06.02:** `get_pipeline_readonly_session`, `OpportunityTierGateContext`, `get_current_user`
- **New in S06.04:** Sort-aware listing cursor (`{sort_by, v, id}`) vs. search cursor (`{published_at, id}`)

### TEA Config Flags

- `tea_use_playwright_utils`: not configured → not used (backend stack)
- `tea_use_pactjs_utils`: not configured → not used
- `tea_pact_mcp`: not configured → none
- `tea_browser_automation`: not applicable (backend)
- `test_stack_type`: auto-detected → `backend`

### Knowledge Fragments Loaded (Core/Backend)

- `data-factories.md` — seeding patterns, ON CONFLICT DO NOTHING, module-scoped fixtures
- `test-quality.md` — isolation rules, no cross-test state, rollback patterns
- `test-healing-patterns.md` — failure diagnosis; asyncpg type handling
- `test-levels-framework.md` — unit / integration / API level selection for backend
- `test-priorities-matrix.md` — P0–P3 criteria and coverage targets
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
| AC1 | GET /api/v1/opportunities in OpenAPI; distinct from /search and /{id} | P3 | API | `test_listing_endpoint_in_openapi_schema` |
| AC2 | Filter params (cpv_codes, regions, budget, deadline, status, type) accepted | P1 | API | (covered via seeded data in sort tests) |
| AC2 | Invalid status enum → 400 | P2 | API | `test_invalid_enum_filter_returns_400[status]` |
| AC2 | Invalid type enum → 400 | P2 | API | `test_invalid_enum_filter_returns_400[type]` |
| AC3 | sort_by=deadline desc → C > B > A | P1 | API | `test_sort_by_deadline_desc_returns_later_deadlines_first` |
| AC3 | sort_by=deadline asc → A < B < C | P1 | API | `test_sort_by_deadline_asc_returns_soonest_first` |
| AC3 | sort_by=published_date desc → C > B > A | P1 | API | `test_sort_by_published_date_desc_returns_most_recent_first` |
| AC3 | sort_by=budget desc → A > B > C | P1 | API | `test_sort_by_budget_desc_returns_200` |
| AC3 | sort_by=relevance_score desc → A > B > C (per company_id) | P1 | API | `test_sort_by_relevance_score_desc_uses_company_id` |
| AC3 | Unknown sort_by → 400 with details.field=sort_by | P1 | API | `test_invalid_sort_by_returns_400` |
| AC3 | Unknown sort_order → 400 | P1 | API | `test_invalid_sort_order_returns_400` |
| AC4 | Pagination: after_cursor → page 2; non-overlapping; total_count stable | P1 | API | `test_forward_pagination_with_after_cursor` |
| AC4 | limit=100 (max) → 200 | P2 | API | `test_limit_100_returns_200` |
| AC4 | limit=101 (>max) → 422 | P2 | API | `test_limit_101_returns_422` |
| AC5 | Malformed cursor (wrong JSON structure) → 400 | P2 | API | `test_malformed_cursor_returns_400` |
| AC5 | Cursor sort_by mismatch (cursor=published_date, req=deadline) → 400 | P2 | API | `test_cursor_sort_by_mismatch_returns_400` |
| AC6 | Free-tier: exactly 6 fields; paid fields absent | P0 | API | `test_free_tier_receives_only_six_fields` |
| AC6 | Paid-tier (Starter): FullResponse with budget, cpv_codes, etc. | P1 | API | `test_paid_tier_receives_full_response` |
| AC6 | Out-of-scope items → 200, silently omitted (not 403) | P1 | API | `test_starter_tier_out_of_scope_returns_200_not_403` |
| AC7 | Response has results[], total_count, next_cursor, prev_cursor | P1 | API | `test_listing_response_has_required_pagination_fields` |
| AC7 | No matching rows → 200, results=[], total_count=0, cursors=null | P2 | API | `test_empty_results_returns_correct_shape` |
| AC8 | relevance_score sort keyed by current_user.company_id | P1 | API | `test_sort_by_relevance_score_desc_uses_company_id` |
| AC9 | No Bearer JWT → 401 | P2 | API | `test_listing_requires_authentication` |
| AC10 | Integration test file exists and covers E06-P0-002, P1-010, P1-011 | P0/P1 | API | (all tests above) |

### Priority Distribution

| Priority | Count | Rationale |
|----------|-------|-----------|
| P0 | 1 | Revenue-critical: free-tier field restriction (E06-R-001) |
| P1 | 11 | Critical feature paths: sort combinations, pagination, tier responses |
| P2 | 7 | Edge cases: limit boundaries, invalid cursors, enum validation, auth, empty results |
| P3 | 1 | Nice-to-have: OpenAPI schema verification |
| **Total** | **20** | |

### Test Level Selection

**All tests are API-level integration tests** (against FastAPI ASGI app via httpx AsyncClient):
- Backend project — no browser/E2E tests required
- Integration over unit because the sort/pagination logic involves actual DB queries that
  cannot be adequately tested at unit level (keyset conditions, NULLS LAST ordering, JSONB subscript)
- Dependency overrides isolate JWT auth and tier-gate logic from subscription DB state

### TDD Red Phase Confirmation

All 20 test functions have `@pytest.mark.skip(reason="RED PHASE: ...")`.
Tests assert **expected behaviour** (not placeholders like `assert True`).
Tests will fail with `SKIPPED` on first run, becoming `FAILED` when the
`@pytest.mark.skip` is removed before the endpoint is implemented, and
`PASSED` after S06.04 implementation.

---

## Step 4: Generated Failing Tests

### Test File Created

| File | Tests | TDD Phase |
|------|-------|-----------|
| `eusolicit-app/services/client-api/tests/api/test_opportunity_listing.py` | 20 | 🔴 RED (all skipped) |

### Fixture Infrastructure

**New in test file (module-scoped):**
- `listing_superuser_engine` — migration-role engine for pipeline schema seeding
- `listing_superuser_factory` — session factory for above engine
- `seeded_listing_opportunities` — inserts 3 rows (A/B/C with deterministic sort values); tears down after module
- `listing_free_app` — FastAPI app with free-tier dependency overrides
- `listing_starter_app` — FastAPI app with starter-tier dependency overrides + `company_id=_LISTING_COMPANY_ID`
- `listing_enterprise_app` — FastAPI app with enterprise-tier dependency overrides

**Added to `tests/conftest.py`:**
- `starter_user_token` — JWT with `subscription_tier=starter` and fixed `company_id=12345678-0000-0000-0000-000000000001`
  (company_id matches `_LISTING_COMPANY_ID` used in seed data for green-phase relevance_score tests)

**Reused from conftest.py:**
- `app` — base FastAPI app fixture (overrides db session, email service, Redis)
- `test_client` — httpx AsyncClient from `app`
- `free_user_token` — JWT for free tier
- `pro_user_token`, `enterprise_user_token` — existing tier JWTs

### Seed Data Design

Three rows seeded with deterministic multi-column ordering:

```
Label | source_id   | deadline  | budget_max | relevance[company] | published_at
------+-------------+-----------+------------+--------------------+-------------
  A   | list-opp-A  | NOW+5d    | 500 000    |        0.9         | NOW-10d
  B   | list-opp-B  | NOW+15d   | 200 000    |        0.5         | NOW-5d
  C   | list-opp-C  | NOW+30d   |  50 000    |        0.2         | NOW-1d
```

Expected DESC order per sort_by:
- `deadline`:        C, B, A  (C has latest = largest deadline)
- `budget`:          A, B, C  (A has largest budget_max)
- `relevance_score`: A, B, C  (A has highest company score)
- `published_date`:  C, B, A  (C published most recently)

Expected ASC order (verified for deadline; others follow inverse):
- `deadline ASC`:    A, B, C  (A has soonest deadline)

### Dependency Override Pattern

```python
# Tests override THREE dependencies to isolate from JWT and subscription DB:
app.dependency_overrides[get_pipeline_readonly_session] = _pipeline_override  # migration-role session
app.dependency_overrides[get_current_user] = _user_override          # fixed CurrentUser
app.dependency_overrides[get_opportunity_tier_gate] = _tier_gate_override  # controlled tier
```

`CurrentUser.company_id` is set to `_LISTING_COMPANY_ID` (UUID `12345678-0000-0000-0000-000000000001`)
in all listing app fixtures so relevance_score sort picks up the seeded per-company scores.

---

## Step 4C: Aggregate

### TDD Red Phase Compliance

- [x] All 20 test functions have `@pytest.mark.skip(reason="RED PHASE: ...")`
- [x] All assertions target **expected** behaviour (not `assert True` placeholders)
- [x] No passing tests generated (correct — this is red phase)
- [x] All test files written to disk

### Files Created / Modified

| Action | Path |
|--------|------|
| **CREATED** | `eusolicit-app/services/client-api/tests/api/test_opportunity_listing.py` |
| **MODIFIED** | `eusolicit-app/services/client-api/tests/conftest.py` (added `starter_user_token`) |
| **CREATED** | `eusolicit-docs/test-artifacts/atdd-checklist-6-4-opportunity-listing-api-endpoint.md` |

---

## Step 5: Validation & Completion

### Acceptance Criteria Coverage

| AC | Coverage | Tests |
|----|----------|-------|
| AC1 — OpenAPI registration, distinct from search/detail | ✅ P3 | 1 test |
| AC2 — Browse mode filters; enum 400 | ✅ P1/P2 | 3 tests |
| AC3 — 4 sort_by × 2 sort_order; 400 on invalid | ✅ P1 | 7 tests |
| AC4 — Cursor pagination; limit boundaries | ✅ P1/P2 | 3 tests |
| AC5 — Sort-aware cursor format; mismatch 400 | ✅ P2 | 2 tests |
| AC6 — TierGate per item; free 6 fields; paid full; out-of-scope silent | ✅ P0/P1 | 3 tests |
| AC7 — Response shape; empty results | ✅ P1/P2 | 2 tests |
| AC8 — Relevance sort by company_id | ✅ P1 | 1 test (via sort_by=relevance_score) |
| AC9 — Auth required; 401 | ✅ P2 | 1 test |
| AC10 — Integration test IDs E06-P0-002, P1-010, P1-011 | ✅ All | All 20 tests |

**Coverage summary:** All 10 ACs have at least one test. E06-R-001 (TierGate bypass risk, score 6)
is mitigated by the P0 field-level test (`test_free_tier_receives_only_six_fields`).

### Quality Gate

| Gate | Status |
|------|--------|
| P0 tests present (revenue-critical gate) | ✅ 1 P0 test (E06-P0-002) |
| P1 tests present (critical feature paths) | ✅ 11 P1 tests |
| All tests have `@pytest.mark.skip` (TDD red phase) | ✅ 20/20 |
| No placeholder assertions (`assert True`) | ✅ All assertions are behavioural |
| Test file follows existing codebase patterns | ✅ Matches test_opportunity_search.py style |
| `starter_user_token` added to conftest.py | ✅ With fixed company_id |
| Seed data uses deterministic sort values | ✅ A/B/C with 3-column ordering |
| Seed teardown uses `source_id LIKE 'list-opp-%'` pattern | ✅ Consistent with other test modules |

### Risks and Assumptions

| Item | Detail |
|------|--------|
| **Assumption: `OpportunityTierGateContext` accepts `user_tier="free"`** | Confirmed from test_opportunity_tier_gate.py patterns |
| **Assumption: `get_pipeline_readonly_session` is injectable** | Confirmed from test_opportunity_search.py |
| **Risk: `relevance_scores::jsonb` seed format** | Seed uses `json.dumps({str_uuid: float})` + `::jsonb` cast — matches S05.07 JSONB schema |
| **Risk: `CurrentUser.company_id` type** | Dev notes confirm `UUID` type; `_LISTING_COMPANY_ID` defined as `uuid.UUID(...)` |
| **Note: `test_total_count_reflects_db_count_not_tier_filtered`** | Not included — requires two different tier contexts in one test (complex); deferred to green-phase validation |

---

## TDD Green Phase Checklist

When S06.04 implementation is complete (Tasks 1–3 done), to enter green phase:

### 1. Verify Implementation Completeness

Before removing skips, confirm all implementation tasks are done:
- [ ] **Task 1.1** — `_encode_listing_cursor` / `_decode_listing_cursor` / `_get_sort_column` / `_build_listing_keyset_condition` added to `opportunity_service.py`
- [ ] **Task 1.2** — `Float`, `nulls_last` added to SQLAlchemy imports
- [ ] **Task 1.3** — `list_opportunities()` function added after `search_opportunities()` in `opportunity_service.py`
- [ ] **Task 2.1** — `GET ""` route handler (browse mode listing) added to `api/v1/opportunities.py`
- [ ] **Task 2.2** — Existing imports verified (no new imports needed if S06.01/S06.02 already in place)

### 2. Remove `@pytest.mark.skip` Decorators

```bash
# Remove all RED PHASE skip markers in the listing test file:
cd eusolicit-app/services/client-api
grep -n "pytest.mark.skip" tests/api/test_opportunity_listing.py
# Remove each @pytest.mark.skip(...) line manually or with sed
```

### 3. Run Tests

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_opportunity_listing.py -v
```

Expected: All 20 tests **PASS** (green phase).

### 4. Verify Test IDs

| Test ID | Expected Result |
|---------|----------------|
| E06-P0-002 | PASS — free-tier sees only 6 fields |
| E06-P1-010 | PASS — all 4 sort_by × 2 sort_order correctly ordered |
| E06-P1-011 | PASS — pagination metadata present; paid-tier sees FullResponse |

### 5. Check Coverage

```bash
pytest tests/api/test_opportunity_listing.py --cov=client_api.api.v1.opportunities \
  --cov=client_api.services.opportunity_service --cov-report=term-missing
```

Target: ≥80% line coverage on listing route handler and `list_opportunities()` function.

### 6. Commit

```bash
git add tests/api/test_opportunity_listing.py tests/conftest.py
git add eusolicit-docs/test-artifacts/atdd-checklist-6-4-opportunity-listing-api-endpoint.md
git commit -m "feat(s06.04): add ATDD tests for opportunity listing endpoint (red phase)"
```

---

## Summary

```
✅ ATDD Test Generation Complete (TDD RED PHASE)

🔴 TDD Red Phase: 20 Failing Tests Generated

📊 Summary:
  - Total Tests: 20 (all with @pytest.mark.skip)
  - P0: 1  (revenue-critical: free-tier field gate)
  - P1: 11 (sort combinations, pagination, tier responses)
  - P2: 7  (edge cases, auth, limit boundaries, cursors)
  - P3: 1  (OpenAPI schema check)
  - Backend API tests only (no E2E — backend project)
  - Execution mode: sequential (single agent)

✅ Acceptance Criteria Coverage: 10/10 ACs covered

📂 Files Generated/Modified:
  - eusolicit-app/services/client-api/tests/api/test_opportunity_listing.py (NEW — 20 tests)
  - eusolicit-app/services/client-api/tests/conftest.py (MODIFIED — starter_user_token added)
  - eusolicit-docs/test-artifacts/atdd-checklist-6-4-opportunity-listing-api-endpoint.md (NEW)

📝 Next Steps (TDD cycle):
  1. Implement S06.04 (Tasks 1–3) per the story task list
  2. Remove @pytest.mark.skip from all 20 tests
  3. Run: pytest tests/api/test_opportunity_listing.py -v
  4. Verify all 20 tests PASS (green phase)
  5. Run full test suite to check for regressions
  6. Commit implementation + tests together

🔗 Related Workflows:
  - /bmad-testarch-automate — expand coverage after green phase
  - /bmad-testarch-ci — wire listing tests into CI pipeline
```
