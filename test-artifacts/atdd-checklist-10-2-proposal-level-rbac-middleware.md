---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate']
lastStep: 'step-04c-aggregate'
lastSaved: '2026-04-24'
workflowType: 'testarch-atdd'
storyId: '10.2'
storyKey: '10-2-proposal-level-rbac-middleware'
storyFile: '/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/10-2-proposal-level-rbac-middleware.md'
atddChecklistPath: '/home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-10-2-proposal-level-rbac-middleware.md'
generatedTestFiles:
  - 'eusolicit-app/services/client-api/tests/unit/test_require_proposal_role.py'
  - 'eusolicit-app/services/client-api/tests/unit/test_proposals_router_dependency_spec.py'
  - 'eusolicit-app/services/client-api/tests/api/test_proposals_rbac_matrix.py'
  - 'eusolicit-app/services/client-api/tests/api/test_proposals_create_auto_seeds_bid_manager.py'
  - 'eusolicit-app/services/client-api/tests/api/test_proposals_audit_trail_preserved.py'
  - 'eusolicit-app/services/client-api/tests/integration/test_036_backfill_migration.py'
inputDocuments: []
---

# ATDD Checklist - Epic 10, Story 2: Proposal-Level RBAC Middleware

**Date:** 2026-04-24
**Author:** BMAD TEA Agent
**Primary Test Level:** API / Integration

---

## Story Summary

Story 10.2 implements the keystone authorization logic for Epic 10, creating `require_proposal_role` middleware to replace role-based checks with proposal-level access control.

**As** any authenticated user interacting with a proposal
**I want** my request to succeed only when my per-proposal collaborator role is in the endpoint's allowed set
**So that** Epic 10's AC13 is satisfied end-to-end and CRITICAL risk R-006 is closed.

---

## Acceptance Criteria

1. `require_proposal_role(*allowed_roles)` factory in `core/rbac.py`.
2. Role-set constants in `core/rbac.py`.
3. Rewire read endpoints in `api/v1/proposals.py`.
4. Rewire write endpoints in `api/v1/proposals.py`.
5. `POST /api/v1/proposals` retains company-role guard + atomic creator-seed.
6. `GET /api/v1/proposals` (list) adds collaborator-membership filter for non-admins.
7. Alembic migration 036: backfill `proposal_collaborators`.
8. `seed_creator_as_bid_manager` helper in `proposal_collaborator_service.py`.
9. Story 10.1's inline auth in `proposal_collaborators.py` is retained.
10. Denial audit on 403 uses `_write_denial_audit`.
11. Structured logging for every authorisation event.
12. Route-dependency snapshot test.
13. Role-boundary unit tests for `require_proposal_role`.
14. API-level integration tests for each rewired endpoint.
15. API-level tests for `POST /api/v1/proposals` auto-seed.
16. Backfill migration test.
17. OpenAPI + README documentation.
18. Audit trail coverage for proposal mutations.

---

## Story Integration Metadata

- **Story ID:** `10.2`
- **Story Key:** `10-2-proposal-level-rbac-middleware`
- **Story File:** `/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/10-2-proposal-level-rbac-middleware.md`
- **Checklist Path:** `/home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-10-2-proposal-level-rbac-middleware.md`

---

## Red-Phase Test Scaffolds Created

### Unit Tests

**Files:** 
- `eusolicit-app/services/client-api/tests/unit/test_require_proposal_role.py`
- `eusolicit-app/services/client-api/tests/unit/test_proposals_router_dependency_spec.py`

- ✅ **Test:** `test_admin_bypass_returns_immediately_without_db_hit`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC1/AC13 Admin bypass
- ✅ **Test:** `test_collaborator_roles_against_allowed_set`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC13 Role boundaries matrix
- ✅ **Test:** `test_proposals_router_dependency_spec`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC12 Route dependency snapshot

### API Tests

**Files:** 
- `eusolicit-app/services/client-api/tests/api/test_proposals_rbac_matrix.py`
- `eusolicit-app/services/client-api/tests/api/test_proposals_create_auto_seeds_bid_manager.py`
- `eusolicit-app/services/client-api/tests/api/test_proposals_audit_trail_preserved.py`

- ✅ **Test:** `test_read_endpoint_matrix`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC3/AC14 Read path enforcement
- ✅ **Test:** `test_write_endpoint_matrix`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC4/AC14 Write path enforcement
- ✅ **Test:** `test_bid_manager_only_matrix`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC4/AC14 Destructive path enforcement
- ✅ **Test:** `test_proposal_create_happy_path_auto_seeds_bid_manager`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC5/AC15 Post-create collaborator seed

### Integration Tests

**File:** `eusolicit-app/services/client-api/tests/integration/test_036_backfill_migration.py`

- ✅ **Test:** `test_036_backfill_migration_upgrade`
  - **Status:** RED - TDD Red Phase scaffold
  - **Verifies:** AC7/AC16 Database migration behavior

---

## Implementation Checklist

### Core RBAC & Helpers
- [ ] Implement `require_proposal_role` factory in `core/rbac.py`
- [ ] Add role constants in `core/rbac.py`
- [ ] Implement `seed_creator_as_bid_manager` in `proposal_collaborator_service.py`
- [ ] Run test: `pytest tests/unit/test_require_proposal_role.py`
- [ ] ✅ Test passes (green phase)

### API Route Updates
- [ ] Rewire 10 read endpoints in `api/v1/proposals.py`
- [ ] Rewire 15 write endpoints in `api/v1/proposals.py`
- [ ] Update POST `/api/v1/proposals` to seed creator
- [ ] Update GET `/api/v1/proposals` list query
- [ ] Run tests: `pytest tests/api/test_proposals_rbac_matrix.py tests/api/test_proposals_create_auto_seeds_bid_manager.py`
- [ ] ✅ Tests pass (green phase)

### Database Migration
- [ ] Generate Alembic migration 036
- [ ] Write backfill SQL for `upgrade()`
- [ ] Run test: `pytest tests/integration/test_036_backfill_migration.py`
- [ ] ✅ Test passes (green phase)

### Audit & Security
- [ ] Ensure 403s trigger `_write_denial_audit`
- [ ] Implement structured logging for role evaluations
- [ ] Run test: `pytest tests/api/test_proposals_audit_trail_preserved.py`
- [ ] ✅ Test passes (green phase)

---

## Running Tests

```bash
# Run all activated unit & API tests
pytest tests/unit tests/api

# Run integration tests (requires testcontainers)
pytest tests/integration/test_036_backfill_migration.py -m integration
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

- ✅ All tests written as red-phase scaffolds with `pytest.mark.skip()`
- ✅ Implementation checklist created

### GREEN Phase (DEV Team - Next Steps)

1. Remove `pytest.mark.skip()` for one test
2. Confirm it fails
3. Implement code to make it pass
4. Move to next test
