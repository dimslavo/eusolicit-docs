---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-21'
workflowType: 'bmad-testarch-atdd'
mode: 'create'
storyKey: '10-9-approval-decision-engine'
detectedStack: 'backend'
generationMode: 'AI Generation (backend story — no browser recording)'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/10-9-approval-decision-engine.md'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-10-8-approval-workflow-stages-crud-api.md'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-9-10-task-approval-notification-consumers.md'
  - 'eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py'
  - 'eusolicit-app/services/client-api/tests/api/test_approval_workflows.py'
  - 'eusolicit-app/services/client-api/tests/unit/test_approval_workflow_service.py'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
  - 'eusolicit-app/services/client-api/tests/api/conftest.py'
testFramework: 'pytest + pytest-asyncio'
---

# ATDD Checklist: Story 10.9 — Approval Decision Engine

**Story:** `10-9-approval-decision-engine`
**Date:** 2026-04-21
**TDD Phase:** 🔴 RED (all tests `@pytest.mark.skip` — failing until implementation complete)
**Stack:** backend (Python / FastAPI / SQLAlchemy / Redis Streams)
**Test Framework:** pytest + pytest-asyncio (`asyncio_mode="auto"`)

> **Coverage note:** No epic-level test design doc exists for Epic 10 (see story Dev Notes).
> Coverage strategy follows the established patterns from Stories 10.1–10.8: `httpx.AsyncClient`
> + `async with AsyncSession` for API tests; `pytest-asyncio` with `asyncio_mode="auto"`;
> cross-tenant isolation returns 404 (NOT 403) for existence-leakage safety. Test structure
> follows the arrange/act/assert block format from `tests/api/test_approval_workflows.py`.

---

## TDD Red Phase Summary

✅ Failing tests generated — **2 test files**, **55 tests total**

All tests are marked `@pytest.mark.skip(reason="RED PHASE: ...")`.
They will run and **pass (GREEN)** when the corresponding implementation is complete.

---

## Generated Test Files

| File | Tests | Priority | ACs Covered |
|------|-------|----------|-------------|
| `services/client-api/tests/api/test_approvals.py` | 35 | P0/P1 | AC3, AC4, AC5, AC6, AC7, AC8, AC9, AC10 |
| `services/client-api/tests/unit/test_approval_decision_service.py` | 20 | P1/P2 | AC1, AC2, AC3, AC4, AC5, AC10 |

**Total: 55 tests** across 2 files.

---

## Acceptance Criteria Coverage

### ✅ AC1 — `client.approval_decisions` Alembic migration + `proposals.approved_at`

- [ ] **`test_migration_043_file_exists`** (unit) — `043_approval_decisions.py` exists in alembic versions dir.
- [ ] **`test_migration_043_revision_chain`** (unit) — `revision == "043"`, `down_revision == "042"`.
- [ ] **`test_migration_043_creates_approval_decisions_table`** — table exists with all required columns + CHECK constraint. *(Integration — requires live DB; to be written in `tests/integration/test_migration_043.py` during GREEN phase)*
- [ ] **`test_migration_043_adds_approved_at_to_proposals`** — column exists, nullable, indexed. *(Integration)*
- [ ] **`test_migration_043_stage_id_fk_restrict`** — `DELETE FROM client.approval_stages WHERE id = :sid_with_decision` raises `IntegrityError`. *(Integration)*
- [ ] **`test_migration_043_proposal_id_fk_cascade`** — deleting a proposal cascades to delete all its `approval_decisions` rows. *(Integration)*
- [ ] **`test_migration_043_decision_check_constraint`** — inserting `decision = 'maybe'` raises `IntegrityError`. *(Integration)*
- [ ] **`test_migration_043_downgrade_reverses`** — downgrade drops table + removes `approved_at` + its index. *(Integration)*

### ✅ AC2 — `ApprovalDecision` ORM model

- [ ] **`test_orm_approval_decision_importable`** (unit) — `from client_api.models import ApprovalDecision` succeeds.
- [ ] **`test_orm_approval_decision_schema_is_client`** (unit) — `__table__.schema == "client"`.
- [ ] **`test_orm_approval_decision_columns`** (unit) — all 7 required columns present.
- [ ] **`test_orm_approval_decision_check_constraint_name`** (unit) — `ck_approval_decisions_decision` in `__table_args__`.
- [ ] **`test_orm_approval_decision_composite_index_exists`** (unit) — `ix_approval_decisions_proposal_stage` composite index.
- [ ] **`test_orm_proposal_has_approved_at_column`** (unit) — `Proposal.approved_at` is nullable.
- [ ] **`test_orm_proposal_approved_at_index_exists`** (unit) — `ix_proposals_approved_at` index in `Proposal.__table__`.
- [ ] **`test_orm_approval_decision_in_models_init`** (unit) — `ApprovalDecision` exported from `client_api.models.__init__`.

### ✅ AC3 — Pydantic schemas (`schemas/approvals.py`)

- [ ] **`test_schema_approval_decision_type_enum`** (unit) — `ApprovalDecisionType` has `approved | rejected | returned_for_revision`.
- [ ] **`test_schema_decision_request_valid_approved`** (unit) — valid approved request; `comment` defaults to `None`.
- [ ] **`test_schema_decision_request_comment_max_5000_chars`** (unit) — >5000 chars raises `ValidationError`.
- [ ] **`test_schema_decision_request_rejected_requires_nonempty_comment`** (unit) — rejected + no comment → `ValidationError`.
- [ ] **`test_schema_decision_request_rejected_whitespace_only_comment_raises`** (unit) — whitespace-only comment rejected after `.strip()`.
- [ ] **`test_schema_decision_request_returned_for_revision_requires_comment`** (unit) — returned_for_revision + no comment → `ValidationError`.
- [ ] **`test_schema_decision_response_from_attributes`** (unit) — `model_config["from_attributes"] == True`.
- [ ] **`test_schema_next_stage_info_fields`** (unit) — has `id, name, order, required_role, auto_advance`; NOT `workflow_id`.
- [ ] **`test_schema_decide_response_fields`** (unit) — `decision, next_stage, proposal_approved`.
- [ ] **`test_schema_history_response_fields`** (unit) — `items, total`.
- [ ] **`test_schema_invalid_decision_enum_rejected`** (unit) — `decision="maybe"` → 422.
- [ ] **`test_decide_rejected_without_comment_returns_422`** (API) — via HTTP endpoint.
- [ ] **`test_decide_returned_for_revision_no_comment_returns_422`** (API) — via HTTP endpoint.
- [ ] **`test_decide_approved_without_comment_returns_201`** (API) — comment optional for approvals.
- [ ] **`test_decide_invalid_decision_value_returns_422`** (API) — `decision="maybe"` → 422.
- [ ] **`test_decide_invalid_stage_id_uuid_returns_422`** (API) — non-UUID `stage_id` → 422.

### ✅ AC4 — `POST /api/v1/proposals/{proposal_id}/approvals/decide`

**Happy paths:**
- [ ] **`test_decide_approved_first_stage_returns_201_with_next_stage`** (API) — 201; `decision.decision == "approved"`; `next_stage.id == stage2.id`; `proposal_approved is False`; one DB row.
- [ ] **`test_decide_approved_final_stage_sets_proposal_approved`** (API) — 201; `next_stage is None`; `proposal_approved is True`; `proposals.approved_at` non-null.
- [ ] **`test_decide_approved_auto_advance_false_next_stage_is_null`** (API) — `auto_advance=False` → `next_stage is None` even if next stage exists.
- [ ] **`test_decide_rejected_returns_201_no_next_stage`** (API) — 201; `next_stage is None`; DB row with `decision='rejected'`.
- [ ] **`test_decide_returned_for_revision_no_event_emitted`** (API) — 201; event NOT emitted; audit row written.

**Gate failures:**
- [ ] **`test_decide_non_collaborator_returns_404`** (API) — `require_proposal_role` dep → 404.
- [ ] **`test_decide_read_only_collaborator_returns_403`** (API) — allowed-roles check → 403.
- [ ] **`test_decide_soft_deleted_workflow_stage_returns_404`** (API) — stage-load JOIN filters `deleted_at IS NULL` → 404.
- [ ] **`test_decide_cross_tenant_stage_returns_404`** (API) — cross-tenant → 404 (existence-leakage safe).
- [ ] **`test_decide_nonexistent_stage_returns_404`** (API) — unknown stage_id → 404.
- [ ] **`test_decide_wrong_proposal_role_returns_403`** (API) — role mismatch → 403; `access_denied` audit row written.
- [ ] **`test_decide_admin_bypasses_all_role_checks`** (API) — company admin bypasses `require_proposal_role` AND stage-role gate → 201.
- [ ] **`test_decide_prior_stage_no_decision_returns_409`** (API) — no prior decision → 409 with detail naming stage.
- [ ] **`test_decide_prior_stage_latest_rejected_returns_409`** (API) — prior latest = rejected → 409.
- [ ] **`test_decide_prior_stage_returned_for_revision_returns_409`** (API) — prior latest = returned_for_revision → 409.
- [ ] **`test_decide_prior_stage_reject_then_approve_override_succeeds`** (API) — rejected then approved override → stage 2 succeeds.

**Service unit:**
- [ ] **`test_service_prior_stage_query_uses_distinct_on`** (unit) — `DISTINCT ON (stage_id)` pattern verified.
- [ ] **`test_service_workflow_linkage_guard_no_prior_decisions_returns_none`** (unit) — no decisions → `None` returned.
- [ ] **`test_service_final_stage_detection_single_stage_workflow`** (unit) — single-stage: `order(1) == max_order(1)` → `True`.
- [ ] **`test_service_final_stage_detection_middle_stage`** (unit) — stage 2 of 3 → `False`.
- [ ] **`test_service_next_stage_suppressed_when_auto_advance_false`** (unit) — `auto_advance=False` → `None`.
- [ ] **`test_service_next_stage_returned_when_auto_advance_true`** (unit) — `auto_advance=True` + approved → next stage returned.
- [ ] **`test_service_next_stage_none_on_rejected_decision`** (unit) — rejected → `None` regardless of `auto_advance`.

**Final-stage + `approved_at`:**
- [ ] **`test_decide_final_stage_approved_twice_does_not_reset_approved_at`** (API) — idempotency guard (`WHERE approved_at IS NULL`).
- [ ] **`test_decide_single_stage_workflow_immediately_approves_proposal`** (API) — single-stage workflow → `proposal_approved=True` immediately.

**Audit trail:**
- [ ] **`test_decide_approved_writes_audit_row`** (API) — `audit_log` row with `entity_type="approval_decision"`, `action_type="create"`, `entity_id=decision.id`; `after` contains `proposal_id`, `stage_id`, `decision`, `final_stage`, `proposal_approved`.

### ✅ AC5 — Event payload shape matches `eusolicit_models.events.ApprovalDecided`

**Unit (schema contract):**
- [ ] **`test_approval_decided_schema_approved_round_trip`** (unit) — construct `ApprovalDecided(decision="approved")` → round-trip via `TypeAdapter(ServiceEvent)`.
- [ ] **`test_approval_decided_schema_rejected_round_trip`** (unit) — same for `decision="rejected"`.
- [ ] **`test_approval_decided_schema_returned_for_revision_rejected`** (unit) — `returned_for_revision` not valid in `ApprovalDecided` (Literal guard).
- [ ] **`test_approval_decided_decision_note_is_optional`** (unit) — `decision_note=None` is valid.
- [ ] **`test_approval_decided_event_payload_fields_match_ac5_spec`** (unit) — all AC5 mandatory fields present in `model_dump()`.

**Integration (Redis Streams):**
- [ ] **`test_decide_approved_emits_approval_decided_event`** (API) — one message on `eu-solicit:approvals`; envelope parses via `TypeAdapter(ServiceEvent)`; correct `approval_id`, `proposal_id`, `decider_id`, `decision="approved"`, `proposal_owner_id == proposal.created_by`.
- [ ] **`test_decide_event_falls_back_to_bid_manager_when_created_by_null`** (API) — `created_by IS NULL` → emitted with bid_manager fallback as `proposal_owner_id`.
- [ ] **`test_decide_no_owner_no_event_emitted_decision_still_succeeds`** (API) — no `created_by` AND no bid_manager → event skipped; WARN logged; 201 returned.
- [ ] **`test_decide_redis_unavailable_decision_still_succeeds`** (API) — EventPublisher raises → 201; WARN logged; non-fatal.
- [ ] **`test_decide_returned_for_revision_no_event_on_stream`** (API) — `returned_for_revision` → no message on stream.

**Service unit (emission logic):**
- [ ] **`test_service_event_emission_skipped_for_returned_for_revision`** (unit) — publisher NOT called.
- [ ] **`test_service_event_emission_called_for_approved`** (unit) — publisher called once with `stream="eu-solicit:approvals"`.
- [ ] **`test_service_event_emission_called_for_rejected`** (unit) — publisher called (rejected IS in `ApprovalDecided` schema).
- [ ] **`test_service_event_emission_failure_does_not_raise`** (unit) — publisher raises → WARN logged; no re-raise.
- [ ] **`test_service_event_emission_skipped_when_owner_id_is_none`** (unit) — `proposal_owner_id=None` → publisher NOT called; WARN logged.

### ✅ AC6 — Workflow-linkage guard

- [ ] **`test_decide_mixed_workflow_returns_409`** (API) — existing decision on workflow A → stage of workflow B → 409 `"Proposal already in a different approval workflow"`.
- [ ] **`test_decide_no_prior_decisions_any_workflow_accepted`** (API) — no prior decisions → any workflow's stage accepted.
- [ ] **`test_decide_same_workflow_decision_accepted`** (API) — prior decisions on workflow A → new decision on workflow A accepted.
- [ ] **`test_decide_soft_deleted_workflow_blocks_new_decisions_on_its_stages`** (API) — soft-deleted workflow → stage-load JOIN filters it → 404.

### ✅ AC7 — Multi-decision-per-stage semantics

- [ ] **`test_decide_stage1_reject_then_approve_unlocks_stage2`** (API) — rejected then approved → stage 2 succeeds (latest = approved).
- [ ] **`test_decide_stage1_approved_then_rejected_blocks_stage2`** (API) — approved then rejected → stage 2 fails 409 (latest = rejected).
- [ ] **`test_decide_stage1_approved_then_returned_blocks_stage2`** (API) — approved then returned_for_revision → stage 2 fails 409.

### ✅ AC8 — `GET /api/v1/proposals/{proposal_id}/approvals/history`

- [ ] **`test_history_returns_all_decisions_ordered_by_created_at_asc`** (API) — 4 decisions seeded; response has `total=4`; `items` ordered by `created_at ASC`.
- [ ] **`test_history_empty_proposal_returns_zero_items`** (API) — no decisions → `items=[]`, `total=0`.
- [ ] **`test_history_read_only_collaborator_can_view`** (API) — `read_only` role → 200.
- [ ] **`test_history_cross_tenant_returns_404`** (API) — Company B user → 404.

### ✅ AC9 — Router wiring

- [ ] **`test_router_decide_path_exists`** (API) — `POST /api/v1/proposals/{proposal_id}/approvals/decide` returns non-404.
- [ ] **`test_router_history_path_exists`** (API) — `GET /api/v1/proposals/{proposal_id}/approvals/history` returns non-404.
- [ ] **`test_router_no_doubled_prefix`** (API) — inspects `app.routes` to confirm no `/proposals/proposals/` double-prefix.

---

## Not-yet-covered (defer to Implementation / bmad-testarch-automate)

| AC | Gap | Reason |
|----|-----|--------|
| AC1 | Live migration integration tests (table DDL, FK RESTRICT, CASCADE, downgrade) | Requires live DB; out of scope for ATDD checklist; to be added in `tests/integration/test_migration_043.py` during GREEN phase |
| AC5 | XREAD poll with exact `msg_id` from `>` consumer group | Requires consumer group bootstrap; acceptable to use `xrange` in tests |
| AC10 | `approved_at` stays set after re-reject of final stage (`WHERE approved_at IS NULL` guard) | Deferred — requires 3-step DB setup; verify at code-review layer |
| AC10 | `proposals.created_by` non-null → event emitted with that user_id directly | Covered by `test_decide_approved_emits_approval_decided_event`; no separate case needed |

---

## Risk Traceability

| Risk | Test | Status |
|------|------|--------|
| Existence-leakage via cross-tenant stage_id | `test_decide_cross_tenant_stage_returns_404` | 🔴 RED |
| Existence-leakage via soft-deleted workflow | `test_decide_soft_deleted_workflow_stage_returns_404` | 🔴 RED |
| Non-fatal Redis failure swallowed | `test_decide_redis_unavailable_decision_still_succeeds` | 🔴 RED |
| Mixed-workflow contamination | `test_decide_mixed_workflow_returns_409` | 🔴 RED |
| Re-vote override (rejected → approved unlocks) | `test_decide_stage1_reject_then_approve_unlocks_stage2` | 🔴 RED |
| Re-vote reversal (approved → rejected re-locks) | `test_decide_stage1_approved_then_rejected_blocks_stage2` | 🔴 RED |
| Single-stage workflow final detection | `test_decide_single_stage_workflow_immediately_approves_proposal` | 🔴 RED |
| Double prefix path mounting (AC9 rationale) | `test_router_no_doubled_prefix` | 🔴 RED |
| Event schema Literal guard (returned_for_revision) | `test_approval_decided_schema_returned_for_revision_rejected` | 🔴 RED |
| access_denied audit row on 403 | `test_decide_wrong_proposal_role_returns_403` | 🔴 RED |
| Story 9.10 consumer schema compatibility | `test_approval_decided_schema_approved_round_trip` + `test_decide_approved_emits_approval_decided_event` | 🔴 RED |

---

## TDD Red → Green Activation Sequence

Remove `@pytest.mark.skip` decorators in this order:

### Phase 1: ORM + migration (Tasks 1–2)
```bash
cd eusolicit-app
# Enable:
# tests/unit/test_approval_decision_service.py
#   - test_migration_043_* (pure Python)
#   - test_orm_approval_decision_*
#   - test_orm_proposal_has_approved_at_column
pytest services/client-api/tests/unit/test_approval_decision_service.py -k "orm or migration"
```

### Phase 2: Pydantic schemas (Task 3)
```bash
# Enable:
# tests/unit/test_approval_decision_service.py - test_schema_*
pytest services/client-api/tests/unit/test_approval_decision_service.py -k "schema"
```

### Phase 3: Event schema contract (AC5 unit)
```bash
# Enable:
# tests/unit/test_approval_decision_service.py - test_approval_decided_*
pytest services/client-api/tests/unit/test_approval_decision_service.py -k "approval_decided"
```

### Phase 4: Service helpers (AC4 unit)
```bash
# Enable all remaining unit tests
pytest services/client-api/tests/unit/test_approval_decision_service.py
```

### Phase 5: Router + gate failures (AC9, AC4 gates)
```bash
# Enable API tests: router, gate failures, validation 422s
pytest services/client-api/tests/api/test_approvals.py -k "router or 404 or 403 or 422"
```

### Phase 6: Happy paths + multi-decision (AC4, AC7)
```bash
pytest services/client-api/tests/api/test_approvals.py -k "approved or rejected or returned or multi"
```

### Phase 7: Workflow linkage + event emission (AC5, AC6)
```bash
pytest services/client-api/tests/api/test_approvals.py -k "workflow or event or emits"
```

### Phase 8: History endpoint (AC8)
```bash
pytest services/client-api/tests/api/test_approvals.py -k "history"
```

### Phase 9: Full suite
```bash
pytest services/client-api/tests/api/test_approvals.py
pytest services/client-api/tests/unit/test_approval_decision_service.py
```

---

## Quality Gate

- [ ] All 55 tests GREEN (skip removed)
- [ ] `ruff check services/client-api/tests/api/test_approvals.py services/client-api/tests/unit/test_approval_decision_service.py` — no lint errors
- [ ] Coverage ≥ 80% on `approval_decision_service.py`, `api/v1/approvals.py`
- [ ] Event schema contract tests pass (Story 9.10 compliance)
- [ ] `test_router_no_doubled_prefix` passes (path mounting verified)
- [ ] No regression on `test_approval_workflows.py` (Story 10.8 — all existing tests still green)

---

## Implementation Files Expected (New)

| File | Required By |
|------|-------------|
| `services/client-api/src/client_api/api/v1/approvals.py` | AC9 |
| `services/client-api/src/client_api/models/approval_decision.py` | AC2 |
| `services/client-api/src/client_api/schemas/approvals.py` | AC3 |
| `services/client-api/src/client_api/services/approval_decision_service.py` | AC4 |
| `services/client-api/alembic/versions/043_approval_decisions.py` | AC1 |

## Implementation Files Expected (Modified)

| File | Change |
|------|--------|
| `services/client-api/src/client_api/models/__init__.py` | Export `ApprovalDecision` |
| `services/client-api/src/client_api/models/proposal.py` | Add `approved_at: Mapped[datetime \| None]` + `ix_proposals_approved_at` index |
| `services/client-api/src/client_api/main.py` | Include `approvals_v1.router` alongside `proposal_comments_v1` and `proposal_section_locks_v1` |

---

## Setup & Fixtures Required

- [ ] `approval_ctx` fixture: registers Company A + admin, creates proposal, adds collaborators for all 5 roles (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only), creates 3-stage workflow + 1-stage workflow + workflow-B (for mixed-workflow tests), yields full context dict. *(Already implemented in test file)*
- [ ] `_decide_payload()` helper: builds a minimal `ApprovalDecisionRequest`-shaped dict. *(Already implemented)*
- [ ] Existing conftest fixtures reused: `client_api_session_factory`, `test_redis_client`, `rsa_test_key_pair`, `_make_rs256_token`.

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Story:** 10-9-approval-decision-engine
**Date:** 2026-04-21
