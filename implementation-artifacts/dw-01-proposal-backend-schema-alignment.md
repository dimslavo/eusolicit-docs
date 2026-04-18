# Story DW-01: Proposal Backend Schema Alignment

Status: draft

**Origin:** Deferred work from code reviews of S07.05 and S07.11 (2026-04-17/18)
**Priority:** High — blocks S07.13 (generation_status), S07.11 correct version display
**Epic:** Post-E07 cleanup (no assigned epic)
**Points:** 2 | **Type:** backend

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager**,
I want the **proposal detail API response to include all fields the frontend workspace requires** — `current_version_number`, `generation_status`, and `espd_profile_summary` in the generation payload —
so that the version indicator shows correctly, the AI generate panel can detect stuck generation on workspace load, and AI-generated drafts incorporate the company's ESPD profile.

## Acceptance Criteria

1. **`current_version_number` in `GET /proposals/:id` response** — `ProposalDetailResponse` in `schemas/proposal.py` must include `current_version_number: int | None`. The value is derived by joining `proposal_versions.version_number` via `proposals.current_version_id`. If `current_version_id` is null, the field is null. This is the value the frontend displays as "v{N}" in the toolbar.

2. **`generation_status` in `GET /proposals/:id` response** — `ProposalDetailResponse` must include `generation_status: str`. The `Proposal` ORM model already has this column (added in S07.05 migration `021_proposal_generation_status`). It is simply missing from the serialization schema.

3. **`espd_profile_summary` in generation payload** — In `services/client-api/src/client_api/services/proposal_service.py`, the `build_generation_payload()` function must include `espd_profile_summary` from the `Company` model. The `Company` model and its ESPD profile relation were implemented in E02/E11. Verify by checking `services/client-api/src/client_api/models/company.py` for the `espd_profiles` relationship and `packages/eusolicit-models/src/eusolicit_models/client/company.py` for the `EspdProfile` DTO. If `espd_profile_summary` is a computed field on the ESPD profile, use the primary ESPD profile associated with the company.

4. **`created_by` response type aligned** — `ProposalDetailResponse.created_by` must be `UUID | None` (nullable) to match the ORM column which allows null (first version row may be created by a system process). The frontend `ProposalResponse` interface must also be updated to `created_by: string | null`.

5. **`status` type annotation tightened** — `ProposalDetailResponse.status` should use a Pydantic `Literal["draft", "active", "archived"]` annotation (not bare `str`) to enable OpenAPI enum schema and prevent unknown values reaching the frontend.

6. **Existing tests remain green** — `pytest tests/api/test_proposals.py -v` must pass with no regressions after all schema changes. Add 3 new test assertions:
   - `GET /proposals/:id` response includes `current_version_number: 1` for a proposal with one version.
   - `GET /proposals/:id` response includes `generation_status: "idle"` for a freshly created proposal.
   - `build_generation_payload()` unit test asserts `"espd_profile_summary"` key present in company section of payload when an ESPD profile exists.

## Tasks / Subtasks

- [ ] Task 1: Update `ProposalDetailResponse` in `schemas/proposal.py` (AC: 1, 2, 4, 5)
  - [ ] 1.1 Open `services/client-api/src/client_api/schemas/proposal.py`
  - [ ] 1.2 Add `current_version_number: int | None = None`
  - [ ] 1.3 Add `generation_status: str = "idle"` (or use `Literal["idle","generating","completed","failed"]`)
  - [ ] 1.4 Change `created_by: UUID` to `created_by: UUID | None`
  - [ ] 1.5 Change `status: str` to `status: Literal["draft", "active", "archived"]`

- [ ] Task 2: Update `GET /proposals/:id` endpoint to populate `current_version_number` (AC: 1)
  - [ ] 2.1 Open `services/client-api/src/client_api/api/v1/proposals.py`
  - [ ] 2.2 In the detail endpoint (or `proposal_service.get_proposal_detail()`), join `proposal_versions.version_number` via `current_version_id`
  - [ ] 2.3 Option A (preferred): Add `current_version_number` as a scalar subquery on the `Proposal` ORM model using `column_property` or inline `select()` in the query
  - [ ] 2.4 Option B: Load the related `ProposalVersion` in the service layer and include `version_number` in the response builder

- [ ] Task 3: Update `build_generation_payload()` to include ESPD profile summary (AC: 3)
  - [ ] 3.1 Open `services/client-api/src/client_api/services/proposal_service.py`
  - [ ] 3.2 Locate `build_generation_payload()` function
  - [ ] 3.3 Query the company's primary ESPD profile: `SELECT * FROM client.espd_profiles WHERE company_id = :company_id AND is_primary = true LIMIT 1`
  - [ ] 3.4 If an ESPD profile exists, include `"espd_profile_summary": espd_profile.profile_summary` (or the relevant field — check `EspdProfile` model for summary field name) in the company section of the payload
  - [ ] 3.5 If no ESPD profile exists, omit the field (do not include null — the AI Gateway should handle its absence gracefully)

- [ ] Task 4: Update frontend `ProposalResponse` interface (AC: 2, 4)
  - [ ] 4.1 Open `eusolicit-app/frontend/apps/client/lib/api/proposals.ts`
  - [ ] 4.2 Add `generation_status: "idle" | "generating" | "completed" | "failed"` to `ProposalResponse`
  - [ ] 4.3 Change `created_by: string` to `created_by: string | null`
  - [ ] 4.4 Verify `ProposalWorkspacePage.tsx` — any usage of `proposal.created_by` must handle null (it is not rendered in the current workspace UI but may appear in version history)

- [ ] Task 5: Add regression + new tests (AC: 6)
  - [ ] 5.1 In `tests/api/test_proposals.py`: assert `response["current_version_number"] == 1` for a seeded proposal
  - [ ] 5.2 Assert `response["generation_status"] == "idle"` for a freshly created proposal
  - [ ] 5.3 In `tests/services/test_proposal_service.py`: add unit test for `build_generation_payload()` asserting `"espd_profile_summary"` present when ESPD profile exists; absent when no profile

## Dev Notes

### Where to Find the ESPD Profile Model

From E11 (`eusolicit-docs/implementation-artifacts/11-1-espd-profile-compliance-framework-db-schema-migrations.md`):
- Table: `client.espd_profiles`
- ORM model: `services/client-api/src/client_api/models/espd_profile.py`
- Field to use: check for `profile_summary`, `summary`, or `executive_summary` — the exact name depends on the E11 implementation. Grep for `espd_profiles` in `services/client-api/src/client_api/models/` before writing.

### Current `build_generation_payload()` Location

From S07.05 implementation artifact (`deferred-work.md` entry):
- `services/client-api/src/client_api/services/proposal_service.py` — `build_generation_payload()` function
- The company data section currently includes `name`, `description` but **omits** `espd_profile_summary`

### `current_version_number` — Scalar Subquery Pattern

```python
from sqlalchemy import select, func

# In ProposalService.get_proposal_detail():
stmt = (
    select(Proposal, ProposalVersion.version_number.label("current_version_number"))
    .outerjoin(
        ProposalVersion,
        ProposalVersion.id == Proposal.current_version_id
    )
    .where(Proposal.id == proposal_id, Proposal.company_id == company_id)
)
result = await session.execute(stmt)
row = result.first()
if not row:
    raise ProposalNotFoundError(proposal_id)
proposal, current_version_number = row
# Build response manually or use model_validate with extra field
```

Or add a `column_property` on the ORM `Proposal` model for a correlated subquery — depends on the existing pattern in the codebase.

### References

- [Source: eusolicit-docs/implementation-artifacts/deferred-work.md] — DW-01 origin items
- [Source: eusolicit-docs/implementation-artifacts/7-5-ai-draft-generation-backend-sse-integration.md#AC2] — `espd_profile_summary` requirement
- [Source: eusolicit-docs/implementation-artifacts/7-11-proposal-workspace-page-layout-navigation.md#Known Deviations] — `current_version_number` and `generation_status` deviations
- [Source: eusolicit-docs/implementation-artifacts/11-1-espd-profile-compliance-framework-db-schema-migrations.md] — ESPD profile schema
- [Source: eusolicit-app/services/client-api/src/client_api/schemas/proposal.py] — target file for schema changes
- [Source: eusolicit-app/services/client-api/src/client_api/services/proposal_service.py] — `build_generation_payload()` target
- [Source: eusolicit-app/frontend/apps/client/lib/api/proposals.ts] — frontend interface update
