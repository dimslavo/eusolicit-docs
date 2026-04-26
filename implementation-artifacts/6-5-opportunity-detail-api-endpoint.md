# Story 6.5: Opportunity Detail API Endpoint

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer building the opportunity detail feature**,
I want **`GET /api/v1/opportunities/{opportunity_id}` to return complete tender data for paid-tier users — including contracting authority, full description, evaluation criteria, mandatory documents, submission deadline with timezone, CPV codes with labels, linked AI analysis summary (if previously generated), a submission guide reference from `pipeline.submission_guides`, and up to 5 related opportunities by CPV/region overlap — after TierGate scope validation, while free-tier users always receive 403 with an upgrade prompt**,
so that **the frontend detail page (S06.11) has all the data needed to render its five tabs (Overview, Documents, Requirements, AI Analysis, Submission Guide), and tier enforcement is consistently applied at the single-item level using `check_scope()` from the `OpportunityTierGateContext` established in S06.02**.

## Acceptance Criteria

1. [x] `GET /api/v1/opportunities/{opportunity_id}` is registered in the opportunities router at path `/{opportunity_id}` (prefix `/api/v1/opportunities`), is distinct from `/search` and `""` (listing), requires a valid Bearer JWT (returns 401 if absent/invalid), and appears in `/openapi.json` with correct path, path parameter schema, and response description.

2. [x] Free-tier users (`subscription_tier = free`) always receive 403 with body `{"error": "tier_limit", "upgrade_url": "<billing_upgrade_url>"}` regardless of which `opportunity_id` they request; no opportunity data is returned to free-tier users on the detail endpoint.

3. [x] Paid-tier users (starter, professional, enterprise) receive TierGate scope validation via `tier_gate.check_scope(item)` after the opportunity is fetched; if the opportunity is outside the user's tier scope (e.g. Starter user requests a non-Bulgaria opportunity or a budget > 500K EUR), 403 is returned with body `{"error": "tier_limit", "upgrade_url": "..."}`.

4. [x] Returns 404 with `{"error": "not_found", "detail": "Opportunity not found."}` for a non-existent `opportunity_id` (UUID not present in `pipeline.opportunities` or where `deleted_at IS NOT NULL`).

5. [x] For an in-scope paid-tier user, the response JSON contains all `OpportunityFullResponse` fields (id, title, description, status, opportunity_type, deadline, budget_min, budget_max, currency, country, region, contracting_authority, cpv_codes, evaluation_criteria, mandatory_documents, relevance_scores, published_at, updated_at) plus three additional top-level keys: `related_opportunities` (list, max 5 items), `submission_guide` (object or null), `ai_summary` (object or null).

6. [x] `related_opportunities` is a list of up to 5 `RelatedOpportunityItem` objects for opportunities that share at least one CPV code with the requested opportunity OR are in the same region; ordered by deadline ASC (soonest-deadline relevant opportunities first); the requested opportunity itself is excluded; each item is scope-checked and out-of-scope items are silently excluded from the list (not 403'd); contains fields: `id`, `title`, `region`, `cpv_codes`, `status`, `deadline`, `opportunity_type`.

7. [x] `submission_guide` when present is the most recently created non-deleted row from `pipeline.submission_guides` for this `opportunity_id`; includes fields: `id`, `source_portal`, `reviewed`, `steps` (full JSONB steps array); when no submission guide exists, `submission_guide` is `null`.

8. [x] `ai_summary` when present is the most recently generated row from `client.ai_summaries` for `(opportunity_id, user_id)` matching the requesting user; includes fields: `id`, `content`, `model`, `tokens_used`, `generated_at`; when no prior summary exists, `ai_summary` is `null`; fetching the summary via this endpoint does NOT decrement the AI usage counter.

9. [x] The `client.ai_summaries` Alembic migration (`018_ai_summaries.py`) creates the table in the `client` schema with columns: `id UUID PK DEFAULT gen_random_uuid()`, `opportunity_id UUID NOT NULL`, `user_id UUID NOT NULL REFERENCES client.users(id) ON DELETE CASCADE`, `company_id UUID NOT NULL REFERENCES client.companies(id) ON DELETE CASCADE`, `content TEXT NOT NULL`, `model VARCHAR(100)`, `tokens_used INTEGER`, `generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`; with composite index on `(opportunity_id, user_id)` and single-column index on `company_id`.

10. [x] Integration tests in `tests/api/test_opportunity_detail.py` cover: free-tier → 403 with `tier_limit` body (E06-P0-001); paid-tier in-scope → 200 with all `OpportunityFullResponse` fields present including `evaluation_criteria`, `mandatory_documents`, deadline timezone-aware (E06-P1-012); `related_opportunities` → up to 5 by CPV/region (E06-P1-013); 404 for non-existent UUID (E06-P1-014); pre-seeded AI summary returned without counter decrement (E06-P1-015); paid-tier out-of-scope → 403 (E06-P0-003 partial); unauthenticated → 401.

## Tasks / Subtasks

- [x] Task 1: Create `client.ai_summaries` Alembic migration (AC: 9)
  - [x] 1.1 Create `eusolicit-app/services/client-api/alembic/versions/018_ai_summaries.py`
- [x] Task 2: Create `pipeline_submission_guide.py` Core table definition (AC: 7)
  - [x] 2.1 Create `eusolicit-app/services/client-api/src/client_api/models/pipeline_submission_guide.py`
- [x] Task 3: Create `client_ai_summary.py` Core table definition (AC: 8)
  - [x] 3.1 Create `eusolicit-app/services/client-api/src/client_api/models/client_ai_summary.py`
- [x] Task 4: Add `OpportunityDetailResponse` and supporting schemas to `schemas/opportunities.py` (AC: 5, 6, 7, 8)
  - [x] 4.1 Edit `eusolicit-app/services/client-api/src/client_api/schemas/opportunities.py`
- [x] Task 5: Add `get_opportunity_detail()` service function to `opportunity_service.py` (AC: 4, 5, 6, 7, 8)
  - [x] 5.1 Add the following imports to the top of `opportunity_service.py`
  - [x] 5.2 Add the following import to the imports block in `opportunity_service.py`
  - [x] 5.3 Add `get_opportunity_detail()` function to `opportunity_service.py` AFTER `list_opportunities()`
- [x] Task 6: Add `GET /{opportunity_id}` route handler to `api/v1/opportunities.py` (AC: 1, 2, 3, 4)
  - [x] 6.1 Add the following imports to `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py`
  - [x] 6.2 Edit `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py`
- [x] Task 7: Write integration tests (AC: 10)
  - [x] 7.1 Create `eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py`

## Dev Agent Record

- **Implemented by:** Gemini 2.0 Flash + session-64d24845-8d28-4902-b1bf-14e8f6eedc49
- **File List:**
  - Modified: `eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py` (unskipped and fixed SQL/dependencies)
  - Modified: `eusolicit-app/services/client-api/src/client_api/services/opportunity_service.py` (hardened search result parsing for missing types)
- **Test Results:** `10 passed, 7 warnings in 2.65s` (Scoped run: `tests/api/test_opportunity_detail.py`)
- **Known Deviations:**
  - Alembic migration `018` was already present with ID `018` instead of `018_ai_summaries`, but fulfills all AC9 requirements.
  - Core implementation was already present; verified correctness and fixed test infrastructure to pass.

### Known Deviation (AC 9)
The Alembic migration ID is `018` instead of `018_ai_summaries`. This matches the project's numerical revision convention and was already merged. AC9 is fully satisfied by the schema content.

## Senior Developer Review

- **Status:** Approved
- **Reviewer:** bmad-code-review
- **Findings:**
  - Implementation correctly handles tier gating, scope checks, and gracefully builds the payload per the specification.
  - All tests passing.
  - Resolved minor linting errors and added missing type hints in `opportunity_service.py` and `opportunities.py` during review to ensure clean `ruff` and `mypy` checks.
  - The deviation regarding migration `018` is acceptable and aligns with conventions.
