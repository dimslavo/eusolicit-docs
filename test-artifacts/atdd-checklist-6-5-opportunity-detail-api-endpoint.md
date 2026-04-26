---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-23'
storyId: '6.5'
storyKey: '6-5-opportunity-detail-api-endpoint'
storyFile: '/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/6-5-opportunity-detail-api-endpoint.md'
atddChecklistPath: '/home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-6-5-opportunity-detail-api-endpoint.md'
generatedTestFiles: 
  - /home/debian/Projects/eusolicit/eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py
inputDocuments:
  - /home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/6-5-opportunity-detail-api-endpoint.md
  - /home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design-epic-06.md
---

# ATDD Checklist: 6-5-opportunity-detail-api-endpoint

## Preflight & Context Loading
- Detected Stack: `backend`
- Story Loaded: `6-5-opportunity-detail-api-endpoint`
- Test Framework: `pytest`

## Generation Mode Selection
- Mode: AI Generation (backend stack detected)

## Test Strategy
**Test Level:** API/Integration
**Priorities:**
- **P0**: Tier enforcement (E06-P0-001, E06-P0-003) - critical revenue/security path.
- **P1**: Core data delivery (E06-P1-012, E06-P1-013, E06-P1-014, E06-P1-015) - primary functionality.
- **Unranked**: Authentication and null states.
**Red Phase Confirmation:** Tests will be written to fail against the non-existent or incomplete `get_opportunity_detail` endpoint before the implementation is done.

 0 tests (backend stack, not applicable)
- All tests assert EXPECTED behavior (no placeholders)

## Acceptance Criteria Coverage
- AC1: Implicit Unauthenticated test.
- AC2: test_detail_free_tier_blocked
- AC3: test_detail_starter_out_of_scope_blocked
- AC4: test_detail_not_found
- AC5, AC6: test_detail_paid_tier_full_response, test_detail_related_opportunities
- AC7: test_detail_submission_guide_present
- AC8: test_detail_ai_summary_returned_without_decrement

## Next Steps (Task-by-Task Activation)
During implementation of each task:
1. Remove `@pytest.mark.skip(reason="ATDD Red Phase scaffold")` from the current test scenario.
2. Run tests: `pytest eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py`
3. Verify the activated test fails first, then passes after implementation (green phase).
4. Commit passing tests.

