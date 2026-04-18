---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: atdd
storyId: 7-2-proposal-crud-api
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-2-proposal-crud-api.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/services/client-api/tests/api/test_espd_profile.py
  - eusolicit-app/services/client-api/tests/api/test_company_profile.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.2 — Proposal CRUD API

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED (failing tests generated — feature not implemented)
**Story:** `7-2-proposal-crud-api`
**Test File:** `services/client-api/tests/api/test_proposals.py`

---

## TDD Red Phase Status

| Metric | Value |
|--------|-------|
| **Phase** | 🔴 RED — All tests will fail until implementation is complete |
| **Test File Created** | ✅ `services/client-api/tests/api/test_proposals.py` |
| **Total Tests** | 41 |
| **Test Classes** | 8 |
| **Execution Mode** | Sequential (Python pytest) |
| **Why Tests Fail** | Router not registered in `main.py`; schemas/service/router files not created |

---

## Stack Detection

- **Detected Stack:** `fullstack` (Python FastAPI backend + Next.js frontend)
- **Story Target:** Backend only (pytest integration tests)
- **Test Framework:** `pytest` + `pytest-asyncio` + `httpx.ASGITransport`
- **Pattern Source:** `test_espd_profile.py` (mirror of `TestAC6CrossCompanyRLS` for E07-P0-005)

---

## Acceptance Criteria Coverage

### AC1 — POST /api/v1/proposals (Create)
**Epic Link:** E07-P1-001

| Test | Method | Expected | Coverage |
|------|--------|----------|----------|
| `test_post_creates_proposal_with_opportunity_id_returns_201` | POST | 201 + all fields | ✅ Nominal path |
| `test_post_creates_proposal_with_title_only_returns_201` | POST | 201 + opportunity_id=null | ✅ Optional field |
| `test_post_creates_proposal_minimal_returns_201` | POST `{}` | 201 + nulls | ✅ Empty body |
| `test_post_initialises_first_version` | POST + GET | 201 + version sections=[] | ✅ Two-phase FK insert |
| `test_post_company_id_in_body_is_ignored` | POST + body injection | 201 + JWT company_id | ✅ Security: body injection ignored |
| `test_post_title_over_500_chars_returns_422` | POST `title` x501 | 422 | ✅ Validation |
| `test_unauthenticated_post_returns_401` | POST (no auth) | 401 | ✅ Auth guard |

**Gaps:** None — all AC1 sub-items covered.

---

### AC2 — GET /api/v1/proposals (List)
**Epic Link:** E07-P1-002

| Test | Method | Expected | Coverage |
|------|--------|----------|----------|
| `test_get_list_empty_returns_200` | GET | 200 `{proposals:[], total:0}` | ✅ Empty state |
| `test_get_list_returns_own_company_proposals` | GET | 200 total=2 | ✅ Nominal list |
| `test_get_list_excludes_archived_by_default` | GET (no filter) | 200 archived excluded | ✅ AC7 default filter |
| `test_get_list_filter_by_opportunity_id` | GET `?opportunity_id=X` | 200 total=1 | ✅ Opportunity filter |
| `test_get_list_filter_by_status` | GET `?status=active` | 200 only active | ✅ Status filter |
| `test_get_list_pagination` | GET `?limit=2&offset=N` | 200 paged | ✅ Pagination |
| `test_get_list_ordered_by_created_at_desc` | GET | B before A | ✅ Sort order |
| `test_unauthenticated_list_returns_401` | GET (no auth) | 401 | ✅ Auth guard |

**Gaps:** None — all AC2/AC7 list behaviors covered.

---

### AC3 — GET /api/v1/proposals/{proposal_id} (Detail)
**Epic Link:** E07-P1-003, E07-P2-001

| Test | Method | Expected | Coverage |
|------|--------|----------|----------|
| `test_get_detail_returns_200_with_current_version_content` | GET `/{id}` | 200 + `current_version_content` | ✅ Joined content |
| `test_get_nonexistent_proposal_returns_404` | GET random UUID | 404 | ✅ E07-P2-001 |
| `test_unauthenticated_get_returns_401` | GET (no auth) | 401 | ✅ Auth guard |

**Gaps:** None.

---

### AC4 — PATCH /api/v1/proposals/{proposal_id}
**Epic Link:** E07-P2-001

| Test | Method | Expected | Coverage |
|------|--------|----------|----------|
| `test_patch_title_only_returns_200` | PATCH `{title}` | 200 title updated, list-item shape | ✅ Partial update |
| `test_patch_status_to_active_returns_200` | PATCH `{status: active}` | 200 status=active | ✅ Status update |
| `test_patch_both_fields_returns_200` | PATCH `{title, status}` | 200 both updated | ✅ Both fields |
| `test_patch_status_archived_returns_422` | PATCH `{status: archived}` | 422 | ✅ Archived excluded from PATCH |
| `test_patch_invalid_status_returns_422` | PATCH `{status: unknown}` | 422 | ✅ Enum validation |
| `test_patch_nonexistent_proposal_returns_404` | PATCH random UUID | 404 | ✅ E07-P2-001 |
| `test_unauthenticated_patch_returns_401` | PATCH (no auth) | 401 | ✅ Auth guard |

**Gaps:** None.

---

### AC5 — DELETE /api/v1/proposals/{proposal_id} (Soft-archive)
**Epic Link:** E07-P1-004, E07-P2-001

| Test | Method | Expected | Coverage |
|------|--------|----------|----------|
| `test_delete_archives_proposal_returns_200` | DELETE | 200 + status=archived | ✅ Soft-archive (not 204) |
| `test_delete_archived_proposal_not_in_default_list` | DELETE + GET list | archived excluded | ✅ AC7 default exclusion |
| `test_delete_archived_proposal_visible_with_status_filter` | DELETE + GET `?status=archived` | returned | ✅ AC7 filter |
| `test_delete_nonexistent_proposal_returns_404` | DELETE random UUID | 404 | ✅ E07-P2-001 |
| `test_unauthenticated_delete_returns_401` | DELETE (no auth) | 401 | ✅ Auth guard |

**Gaps:** None.

---

### AC6 — Company-Scoped RLS (E07-P0-005)
**Epic Link:** E07-P0-005 (P0 — Critical)

| Test | Method | Expected | Coverage |
|------|--------|----------|----------|
| `test_company_b_cannot_get_company_a_proposal_returns_404` | GET cross-company | 404 (not 403) | ✅ UUID enumeration prevention |
| `test_company_b_cannot_patch_company_a_proposal_returns_404` | PATCH cross-company | 404 (not 403) | ✅ Cross-company PATCH |
| `test_company_b_cannot_delete_company_a_proposal_returns_404` | DELETE cross-company | 404 (not 403) | ✅ Cross-company DELETE |
| `test_company_b_cannot_view_company_a_versions_endpoint_returns_404` | GET detail cross-company | 404 | ✅ Version data no-leak |
| `test_list_only_returns_own_company_proposals` | GET list (2 companies) | A sees 2, not 3 | ✅ Company-scoped list isolation |
| `test_post_company_id_body_ignored_uses_jwt_company` | POST + body injection | JWT company wins | ✅ Body company_id ignored |

**Note:** Full E07-P0-005 requires 5 endpoint types. S07.02 covers: GET detail, PATCH, DELETE,
and list isolation. `GET /{id}/versions` is S07.03; `POST /{id}/generate` is S07.05.

---

### AC8 — Role Enforcement

| Test | Method | Expected | Coverage |
|------|--------|----------|----------|
| `test_low_privilege_roles_cannot_post` (×3) | POST contributor/reviewer/read_only | 403 | ✅ Parametrized |
| `test_low_privilege_roles_cannot_patch` (×3) | PATCH contributor/reviewer/read_only | 403 | ✅ Parametrized |
| `test_low_privilege_roles_cannot_delete` (×3) | DELETE contributor/reviewer/read_only | 403 | ✅ Parametrized |
| `test_bid_manager_can_post_patch_delete` | POST+PATCH+DELETE bid_manager | 201/200/200 | ✅ Permitted role |
| `test_all_roles_can_get_list_and_detail` (×3) | GET contributor/reviewer/read_only | 200 | ✅ Read-only access |

---

## AC7 (Embedded in AC2/AC5)
AC7 (archived exclusion from default list; filter-based access) is tested via:
- `test_get_list_excludes_archived_by_default`
- `test_get_list_filter_by_status`
- `test_delete_archived_proposal_not_in_default_list`
- `test_delete_archived_proposal_visible_with_status_filter`

---

## Test Count Summary

| Class | Tests | AC | Priority |
|-------|-------|----|----------|
| `TestAC1CreateProposal` | 7 | AC1, AC8 | P1 (E07-P1-001) |
| `TestAC2ListProposals` | 8 | AC2, AC7, AC8 | P1 (E07-P1-002) |
| `TestAC3GetProposal` | 3 | AC3, AC8 | P1 (E07-P1-003) |
| `TestAC4PatchProposal` | 7 | AC4, AC8 | P1/P2 |
| `TestAC5ArchiveProposal` | 5 | AC5, AC7, AC8 | P1 (E07-P1-004) |
| `TestAC6CrossCompanyRLS` | 6 | AC6 | **P0 (E07-P0-005)** |
| `TestAC8RoleEnforcement` | 5+3×3=14 parametrized | AC8 | P1 |
| **Total** | **50 (41 unique + 9 parametrized expansions)** | AC1–AC9 | P0–P1 |

> Note: 3 parametrized test methods each expand to 3 test instances (roles: contributor,
> reviewer, read_only), giving 50 actual test runs for the full AC8 parametrize set.

---

## Why These Tests Will Fail (Red Phase)

The following files do not exist yet:

```
services/client-api/src/client_api/schemas/proposals.py     ← Not created (Task 1)
services/client-api/src/client_api/services/proposal_service.py  ← Not created (Task 2)
services/client-api/src/client_api/api/v1/proposals.py      ← Not created (Task 3)
```

And in `services/client-api/src/client_api/main.py`:
```python
# Missing (Task 4):
from client_api.api.v1 import proposals as proposals_v1
api_v1_router.include_router(proposals_v1.router)
```

When the test file is run before implementation, all requests to `/api/v1/proposals` will
return **404** (route not found), causing every assertion to fail.

---

## Implementation Files (for Dev Reference)

| File | Status |
|------|--------|
| `src/client_api/schemas/proposals.py` | ❌ Not created |
| `src/client_api/services/proposal_service.py` | ❌ Not created |
| `src/client_api/api/v1/proposals.py` | ❌ Not created |
| `src/client_api/main.py` | ❌ Router not registered |
| `tests/api/test_proposals.py` | ✅ **Created (ATDD RED phase)** |

---

## TDD Green Phase Checklist

After implementing the feature (Story 7.2 Tasks 1–5):

- [ ] Run `cd eusolicit-app/services/client-api && pytest tests/api/test_proposals.py -v`
- [ ] Verify all 41+ tests **PASS** (green phase)
- [ ] If any test fails → either fix the implementation OR file a test-correction issue
- [ ] Verify `pytest tests/api/test_proposals.py -v --tb=short` completes within 60s
- [ ] Check coverage: `pytest --cov=client_api.api.v1.proposals --cov=client_api.services.proposal_service tests/api/test_proposals.py`
- [ ] Coverage for proposal router: ≥80% (from epic exit criteria)
- [ ] No P0 failures allowed before Sprint 8 close (Demo Milestone)

---

## Epic Test Coverage Map

| Epic Test ID | Priority | Covered By |
|-------------|----------|-----------|
| E07-P0-005 | **P0** | `TestAC6CrossCompanyRLS` (6 tests — GET detail, PATCH, DELETE, list isolation, body injection) |
| E07-P1-001 | P1 | `TestAC1CreateProposal.test_post_creates_proposal_with_opportunity_id_returns_201` |
| E07-P1-002 | P1 | `TestAC2ListProposals` (8 tests — all list scenarios) |
| E07-P1-003 | P1 | `TestAC3GetProposal.test_get_detail_returns_200_with_current_version_content` |
| E07-P1-004 | P1 | `TestAC5ArchiveProposal` (5 tests — soft-archive full behavior) |
| E07-P2-001 | P2 | `test_get_nonexistent_proposal_returns_404`, `test_patch_nonexistent_proposal_returns_404`, `test_delete_nonexistent_proposal_returns_404` |

**Not covered in S07.02 (deferred to later stories):**

| Epic Test ID | Story | Reason |
|-------------|-------|--------|
| E07-P0-001 (SSE persistence) | S07.05 | Proposal generation stream — not in S07.02 |
| E07-P0-002 (cleanup job) | S07.05 | generation_status cleanup — not in S07.02 |
| E07-P0-003 (concurrent PATCH race) | S07.04 | Optimistic locking — not in S07.02 |
| E07-P0-005 (GET /versions, POST /generate) | S07.03, S07.05 | 2 of 5 endpoint types deferred |

---

## Related Documents

- Story: `eusolicit-docs/implementation-artifacts/7-2-proposal-crud-api.md`
- Epic test design: `eusolicit-docs/test-artifacts/test-design-epic-07.md`
- Pattern reference: `services/client-api/tests/api/test_espd_profile.py`
- Architecture constraints: Story Dev Notes (AC6, circular FK, structlog, flush/refresh pattern)

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Story:** 7-2-proposal-crud-api
**Version:** Create mode (Steps 1–4C)
