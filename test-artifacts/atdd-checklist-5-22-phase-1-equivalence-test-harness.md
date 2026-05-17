---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
lastStep: step-04c-aggregate
lastSaved: '2026-05-14'
storyId: '5.22'
storyKey: 5-22-phase-1-equivalence-test-harness
storyFile: /home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/5-22-phase-1-equivalence-test-harness.md
atddChecklistPath: /home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/atdd-checklist-5-22-phase-1-equivalence-test-harness.md
generatedTestFiles:
  - eusolicit-app/services/data-pipeline/tests/unit/test_equivalence_checksum.py
  - eusolicit-app/services/data-pipeline/tests/unit/test_compare_equivalence.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_compare_equivalence.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_workflow_event_consumer_shadow.py
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-22-phase-1-equivalence-test-harness.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-docs/project-context.md
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_workflow_event_consumer.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/services/equivalence_checksum.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/compare_equivalence.py
---

# ATDD Checklist: Story 5.22 — Phase-1 Equivalence Test Harness

**Story:** S05.22 — Phase-1 Equivalence Test Harness (E05 SirmaAI cutover)  
**Date:** 2026-05-14  
**Author:** TEA Master Test Architect  
**TDD Phase:** RED (tests define expected behavior; must fail until implementation is complete)  
**Stack:** Backend Python — `@pytest.mark.unit` + `@pytest.mark.integration`

---

## Context & Coverage Strategy

This story is the **measurement instrument** for the E05 SirmaAI re-platform.
It instruments dual-pipeline coexistence (Celery + N8N shadow paths), computes
daily equivalence diffs, and emits the Phase-1 gate signal ("<0.1% delta over
rolling 7-day window") that unblocks the Phase-2 cutover in S05.23.

**E05 Test Design inheritance:**

| E05 Risk | Mitigation in this story |
|---|---|
| E05-R-001 (dedup race) | Shadow source_type uses same atomic `ON CONFLICT DO UPDATE`; INT-001/INT-006 cover idempotency |
| E05-R-006 (soft-delete bypass) | INT-005 explicitly tests `deleted_at IS NULL` filter on both sides of the diff |
| E05-P1-004 (soft-delete model filter) | Same principle applied to both `aop` + `aop_n8n` query paths |

---

## TDD Red Phase Status

🔴 **Red Phase: Test scaffolds generated and verified.**

- Unit tests: **28 test cases** across 2 files (UNIT-001 through UNIT-007 + AC13)
- Integration tests: **15 test cases** across 2 files (INT-001 through INT-010 + INT-009 ×3 sub-cases)
- Activated tests will **fail** until the implementation modules are complete and correct.
- **Note:** Auto-sync has shipped skeleton implementation files. Tests may partially pass
  during dev-story execution; the obligation is that all tests pass (green phase) at story close.

---

## Acceptance Criteria → Test Coverage Map

### AC1 — `pipeline.equivalence_runs` Alembic migration

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| E05-AC1-MIGRATE-UP | `alembic upgrade head` creates `equivalence_runs` with all columns + unique index | Integration (manual: `make migrate-service SVC=data-pipeline`) | Alembic migration | 🔴 Verify on infra |
| E05-AC1-MIGRATE-DOWN | `alembic downgrade -1` cleanly drops table + index | Integration (manual) | Alembic migration | 🔴 Verify on infra |

> **Notes:** Migration must be verified via `make reset-db && make infra && make migrate-service SVC=data-pipeline`.
> The ORM model `EquivalenceRun` is tested implicitly by all INT tests that call `_upsert_equivalence_run`.

---

### AC2 — Shadow `source_type` discriminator in workflow event consumer

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **UNIT-007a** | `shadow_source_type` applied when flag=true; upsert receives `'aop_n8n'` | Unit | `tests/unit/test_compare_equivalence.py::TestShadowConsumerPath::test_shadow_applied_when_flag_true` | 🔴 Red |
| **UNIT-007b** | Flag=false; upsert receives original `'aop'` (Phase-2 regression guard) | Unit | `tests/unit/test_compare_equivalence.py::TestShadowConsumerPath::test_shadow_NOT_applied_when_flag_false` | 🔴 Red |
| **UNIT-007c** | TED shadow applied when TED flag=true | Unit | `tests/unit/test_compare_equivalence.py::TestShadowConsumerPath::test_ted_shadow_applied_when_flag_true` | 🔴 Red |
| **UNIT-007d** | EU Grants shadow inactive when flag=false | Unit | `tests/unit/test_compare_equivalence.py::TestShadowConsumerPath::test_eu_grants_shadow_when_false` | 🔴 Red |
| **INT-009a** | XADD event (AOP, flag=true) → DB row has `source_type='aop_n8n'`; no `'aop'` row | Integration | `tests/integration/test_workflow_event_consumer_shadow.py::test_int_009_shadow_flag_on_writes_shadow_source_type` | 🔴 Red |
| **INT-009b** | XADD event (AOP, flag=false) → DB row has `source_type='aop'`; no `'aop_n8n'` row | Integration | `tests/integration/test_workflow_event_consumer_shadow.py::test_int_009_shadow_flag_off_writes_original_source_type` | 🔴 Red |
| **INT-009c** | `OpportunitiesIngested` envelope uses ORIGINAL `crawler_type='aop'` even when shadow=on | Integration | `tests/integration/test_workflow_event_consumer_shadow.py::test_int_009_original_event_fields_preserved_in_shadow_mode` | 🔴 Red |

---

### AC3 — Canonical opportunity checksum function

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **UNIT-001a** | Dict and ORM mock → same hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_dict_and_orm_same_hash` | 🔴 Red |
| **UNIT-001b** | Key order irrelevant | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_key_order_irrelevant` | 🔴 Red |
| **UNIT-001c** | None and absent key → same hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_none_and_missing_key_equivalent` | 🔴 Red |
| **UNIT-001d** | Different `id`/`created_at`/`raw_data`/`relevance_scores` → same hash (excluded) | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_different_id_same_hash` | 🔴 Red |
| **UNIT-001e** | Title change → different hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_title_change_different_hash` | 🔴 Red |
| **UNIT-001f** | Deadline change → different hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_deadline_change_different_hash` | 🔴 Red |
| **UNIT-001g** | `budget_min` change → different hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_budget_min_change_different_hash` | 🔴 Red |
| **UNIT-001h** | Currency change → different hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_currency_change_different_hash` | 🔴 Red |
| **UNIT-001i** | CPV codes sorted → order independent | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_cpv_codes_order_independent` | 🔴 Red |
| **UNIT-001j** | CPV code value change → different hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_cpv_codes_value_change_different_hash` | 🔴 Red |
| **UNIT-001k** | `evaluation_criteria` nested struct change → different hash | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_evaluation_criteria_nested_structure` | 🔴 Red |
| **UNIT-001l** | `evaluation_criteria` key order irrelevant | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_evaluation_criteria_key_order_irrelevant` | 🔴 Red |
| **UNIT-001m** | Output is lowercase hex SHA-256 (64 chars) | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_hash_is_lowercase_hex_sha256` | 🔴 Red |
| **UNIT-001n** | Byte-stable across repeated calls | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_hash_byte_stable_across_calls` | 🔴 Red |
| **UNIT-001o** | `source_type` change does NOT affect hash (excluded) | Unit | `tests/unit/test_equivalence_checksum.py::TestCanonicalOpportunityHash::test_source_type_excluded` | 🔴 Red |

---

### AC2 / AC3 — Shadow source_type mapper

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **UNIT-002a** | `aop → aop_n8n`, `ted → ted_n8n`, `eu_grants → eu_grants_n8n` | Unit | `tests/unit/test_equivalence_checksum.py::TestShadowSourceType::test_known_sources_mapped` | 🔴 Red |
| **UNIT-002b** | Unknown source returns input unchanged (defence-in-depth) | Unit | `tests/unit/test_equivalence_checksum.py::TestShadowSourceType::test_unknown_returns_input` | 🔴 Red |

---

### AC4 — Daily diff Celery task

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **UNIT-003a** | `delta_pct` exactly 0.1% → `gate_status='red'` (strict `<`, not `<=`) | Unit | `tests/unit/test_compare_equivalence.py::TestDeltaPctAndGateStatus::test_delta_pct_exact_boundary_is_red` | 🔴 Red |
| **UNIT-003b** | `delta_pct` < 0.1% → `gate_status='green'` | Unit | `tests/unit/test_compare_equivalence.py::TestDeltaPctAndGateStatus::test_delta_pct_below_boundary_is_green` | 🔴 Red |
| **UNIT-003c** | `2/2000 * 100 = 0.1000` rounding exactness | Unit | `tests/unit/test_compare_equivalence.py::TestDeltaPctAndGateStatus::test_delta_pct_rounding_two_of_2000` | 🔴 Red |
| **UNIT-004** | Gate status decision matrix (6 parametrized cases) | Unit | `tests/unit/test_compare_equivalence.py::TestDeltaPctAndGateStatus::test_gate_status_matrix` | 🔴 Red |
| **INT-001** | 100 matched rows → `gate_status='green'`, all counts correct | Integration | `tests/integration/test_compare_equivalence.py::test_int_001_happy_path_green` | 🔴 Red |
| **INT-002** | 3 drifts → `drift_count=3`, `delta_pct=3.0000`, `gate_status='red'`, samples populated | Integration | `tests/integration/test_compare_equivalence.py::test_int_002_drift_detection` | 🔴 Red |
| **INT-003** | 100 Celery / 95 shadow → `missing_in_shadow=5`, `delta_pct=5.0000`, `gate_status='red'` | Integration | `tests/integration/test_compare_equivalence.py::test_int_003_asymmetric_cardinality` | 🔴 Red |
| **INT-004** | Rows outside rolling window excluded from counts | Integration | `tests/integration/test_compare_equivalence.py::test_int_004_window_exclusion` | 🔴 Red |
| **INT-005** | Soft-deleted rows excluded from both sides of diff | Integration | `tests/integration/test_compare_equivalence.py::test_int_005_soft_delete_exclusion` | 🔴 Red |

---

### AC5 — Celery Beat schedule entry

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| Beat schedule entry | `equivalence-check-daily` entry in `BEAT_SCHEDULE` at 03:00 UTC, env-overridable | Unit (manual inspection + unit test in `test_celery_beat.py`) | `tests/unit/test_celery_beat.py` | 🟡 Check existing beat tests cover new entry |

---

### AC6 — Rolling 7-day rollup query helper

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **INT-010a** | 7 days: days 1-6 at 0.05%, day 7 at 0.50% → rolling `gate_status='red'` | Integration | `tests/integration/test_compare_equivalence.py::test_int_010_rolling_window_gate_status` | 🔴 Red |
| **INT-010b** | Only 5 days seeded → `days_observed=5`, `gate_status='insufficient_data'` | Integration | `tests/integration/test_compare_equivalence.py::test_int_010_rolling_insufficient_data` | 🔴 Red |
| **INT-010c** | 7 green days → `gate_status='green'`, `rolling_delta_pct≈0.0` | Integration | `tests/integration/test_compare_equivalence.py::test_int_010_green_rolling` | 🔴 Red |

---

### AC7 — Operator runbook

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| Runbook file existence | `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` exists with required sections | Manual review | `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` | 🔴 Create during dev-story |

---

### AC8 — Prometheus metrics

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| Metrics registered | `pipeline_equivalence_delta_pct`, `pipeline_equivalence_drift_total`, `pipeline_equivalence_gate_status` on `PIPELINE_METRICS_REGISTRY` | Unit (test_metrics.py) | `tests/unit/test_metrics.py` | 🟡 Extend existing metrics test |
| **INT-007** | `gate_status='insufficient_data'` → `EQUIVALENCE_GATE_STATUS.labels(source_type="aop") == -1` | Integration | `tests/integration/test_compare_equivalence.py::test_int_007_insufficient_data` | 🔴 Red |

---

### AC9 — Idempotency on task retry

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **INT-006** | Second run same day: no duplicate `equivalence_runs` row; counter not double-counted | Integration | `tests/integration/test_compare_equivalence.py::test_int_006_idempotent_retry` | 🔴 Red |

---

### AC10 — Unit tests

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **UNIT-005a** | `delta_samples` capped at 20 entries even with 5000 drifts | Unit | `tests/unit/test_equivalence_checksum.py::TestDeltaSamplesCap::test_samples_capped_at_20` | 🔴 Red |
| **UNIT-005b** | Each sample has exactly `{source_id, drift_kind, fields_changed}` | Unit | `tests/unit/test_equivalence_checksum.py::TestDeltaSamplesCap::test_sample_keys_strict` | 🔴 Red |
| **UNIT-006** | AOP diff raises → TED + EU Grants still run; `equivalence.source_failed` logged | Unit | `tests/unit/test_compare_equivalence.py::TestPerSourceFailureIsolation::test_aop_failure_ted_and_eu_grants_run` | 🔴 Red |
| **UNIT-007** | Shadow env flag regression: flag=true → shadow applied; flag=false → original used | Unit | `tests/unit/test_compare_equivalence.py::TestShadowConsumerPath` (4 test methods) | 🔴 Red |

---

### AC11 — Integration tests (subset: INT-008)

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **INT-008** | AOP raises (monkeypatched) → no AOP row; TED + EU Grants rows created; `equivalence.source_failed` logged | Integration | `tests/integration/test_compare_equivalence.py::test_int_008_per_source_isolation` | 🔴 Red |

---

### AC12 — Rolling 7-day helper (see AC6 above)

_INT-010a/b/c above cover AC12._

---

### AC13 — No cross-tenant leakage in `delta_samples`

| Test ID | Description | Level | File | Status |
|---------|-------------|-------|------|--------|
| **AC13-unit** | `delta_samples` entries have exactly `{source_id, drift_kind, fields_changed}`; no `company_id`, `relevance_scores`, `raw_data` | Unit | `tests/unit/test_compare_equivalence.py::TestDeltaSamplesNoPII::test_no_pii_in_delta_samples_from_drift` | 🔴 Red |

---

### AC14 — Coverage and DoD

| Gate | Target | Verification method |
|------|--------|-------------------|
| `make lint` | Clean on `services/data-pipeline` | `ruff check` — no errors |
| `make type-check` | Clean on new modules | `mypy` strict on `compare_equivalence.py`, `equivalence_checksum.py`, `models/equivalence_run.py` |
| `make test-service SVC=data-pipeline` | Unit + integration all green | Run after `make infra && make migrate-all` |
| Coverage on new files | ≥ 90% line coverage | `make coverage` reports on 4 new/modified files |
| Project-level coverage | ≥ 80% | `make coverage` |

---

## Test File Inventory

### Unit tests (no I/O — `@pytest.mark.unit`)

| File | Test IDs covered | AC |
|------|-----------------|-----|
| `tests/unit/test_equivalence_checksum.py` | UNIT-001 (×15 sub-cases), UNIT-002, UNIT-005, UNIT-007 (env flag) | AC3, AC2, AC10 |
| `tests/unit/test_compare_equivalence.py` | UNIT-003, UNIT-004, UNIT-006, UNIT-007 (consumer), AC13 | AC4, AC10, AC13 |

### Integration tests (`@pytest.mark.integration` — needs `make infra`)

| File | Test IDs covered | AC |
|------|-----------------|-----|
| `tests/integration/test_compare_equivalence.py` | INT-001, INT-002, INT-003, INT-004, INT-005, INT-006, INT-007, INT-008, INT-010 (×3) | AC4, AC6, AC9, AC11, AC12 |
| `tests/integration/test_workflow_event_consumer_shadow.py` | INT-009 (×3 sub-cases) | AC2, AC11 |

---

## Red Phase Activation Guide

During dev-story implementation, activate tests task-by-task:

1. **Task 1 (AC1):** Run `make migrate-service SVC=data-pipeline` — verify migration up/down.
2. **Task 2 (AC3):** Activate `test_equivalence_checksum.py` — all UNIT-001, UNIT-002, UNIT-005 tests.
3. **Task 3 (AC2):** Activate `TestShadowConsumerPath` in both unit files; run INT-009.
4. **Task 4 (AC4, AC8, AC9):** Activate `test_compare_equivalence.py` unit + integration tests.
5. **Task 5 (AC8):** Verify metrics in `test_metrics.py`; INT-007 gate metric value.
6. **Task 6 (AC7):** Review runbook file exists with required sections.
7. **Task 7 (AC10, AC11, AC12, AC13):** All remaining unit + integration tests.
8. **Task 8 (AC14):** `make lint && make type-check && make test-service SVC=data-pipeline && make coverage`.

> **CRITICAL:** Never claim a story complete on the basis that tests "should pass."
> Run `make test-service SVC=data-pipeline` and paste the pytest summary into the
> story file's `Dev Agent Record → Test Results` section.

---

## Implementation Guidance

### Modules to CREATE

| Module | Purpose |
|--------|---------|
| `services/data-pipeline/alembic/versions/004_equivalence_runs.py` | Alembic migration: `pipeline.equivalence_runs` table + unique index on `(run_date, source_type)` |
| `services/data-pipeline/src/data_pipeline/models/equivalence_run.py` | SQLAlchemy ORM model for `equivalence_runs` |
| `services/data-pipeline/src/data_pipeline/services/equivalence_checksum.py` | `STABLE_FIELDS`, `canonical_opportunity_hash`, `shadow_source_type`, `is_equivalence_shadow_enabled` |
| `services/data-pipeline/src/data_pipeline/workers/tasks/compare_equivalence.py` | Celery task `pipeline.compare_celery_vs_n8n_equivalence` + helpers |
| `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` | Operator runbook (AC7) |

### Modules to UPDATE

| Module | Change |
|--------|--------|
| `services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py` | Apply `shadow_source_type` + `is_equivalence_shadow_enabled`; only change the `effective_source_type` arg to `upsert_opportunities` |
| `services/data-pipeline/src/data_pipeline/workers/celery_app.py` | Add `"data_pipeline.workers.tasks.compare_equivalence"` to `include=[]` |
| `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` | Add `equivalence-check-daily` entry at 03:00 UTC |
| `services/data-pipeline/src/data_pipeline/metrics.py` | Add 3 new Prometheus metrics on `PIPELINE_METRICS_REGISTRY` |
| `services/data-pipeline/src/data_pipeline/models/__init__.py` | Export `EquivalenceRun` |

### Key design decisions (Dev Notes summary)

- **Shadow discriminator is in `source_type`**, NOT in `source_id` prefix — clean JOIN, preserves `source_id` for downstream consumers.
- **`STABLE_FIELDS`** excludes `id`, `source_type`, `relevance_scores`, `raw_data`, `created_at`, `updated_at`, `deleted_at`, `tsv` — see AC3 for rationale.
- **`delta_pct` gate**: strict `<0.1%` (not `<=`) per E05 amendment. Exactly 0.1% is `'red'`.
- **Idempotency**: `INSERT … ON CONFLICT DO UPDATE` + `xmax = 0` trick for `is_new` detection; counters only incremented on `is_new=True`.
- **`OpportunitiesIngested.crawler_type`**: always the ORIGINAL source_type, never the shadow discriminator.

---

## Coverage Summary

| Priority | Test Count | AC Coverage |
|----------|------------|------------|
| P0 (critical) | 8 | AC4 INT-001, INT-002, INT-003, AC9 INT-006, AC11 INT-008, AC2 INT-009 |
| P1 (high) | 18 | UNIT-001 (×15), UNIT-002, UNIT-003, UNIT-004, INT-004, INT-005, INT-010 |
| P2 (medium) | 6 | UNIT-005, UNIT-006, UNIT-007, AC13, INT-007 |
| P3 (low) | 3 | Beat schedule, metrics registration, runbook review |
| **Total** | **35** | **AC1–AC14** |

---

**Generated by:** BMad TEA Agent — Master Test Architect  
**Workflow:** `bmad-testarch-atdd`  
**Story:** S05.22 Phase-1 Equivalence Test Harness  
**Epic:** E05 — Data Pipeline & Opportunity Ingestion (SirmaAI re-platform)
