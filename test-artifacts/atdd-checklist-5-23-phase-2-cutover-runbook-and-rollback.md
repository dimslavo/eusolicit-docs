---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-05-15'
storyId: '5.23'
storyKey: 5-23-phase-2-cutover-runbook-and-rollback
storyFile: /home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/5-23-phase-2-cutover-runbook-and-rollback.md
atddChecklistPath: /home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/atdd-checklist-5-23-phase-2-cutover-runbook-and-rollback.md
generatedTestFiles:
  - eusolicit-app/services/data-pipeline/tests/unit/test_beat_schedule.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-23-phase-2-cutover-runbook-and-rollback.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/unit/test_celery_beat.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_workflow_event_consumer_shadow.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/beat_schedule.py
  - eusolicit-app/services/data-pipeline/tests/unit/conftest.py
  - eusolicit-app/services/data-pipeline/tests/integration/conftest.py
  - eusolicit-docs/project-context.md
  - eusolicit-app/_bmad/tea/config.yaml
---

# ATDD Checklist: Story 5.23 — Phase-2 Cutover Runbook + Rollback

**Date:** 2026-05-15
**Story:** S05.23 — Phase-2 Cutover Runbook + Rollback
**Stack:** Backend (data-pipeline service)
**Mode:** Sequential — AI Generation (no browser recording; backend-only story)
**TDD Phase:** 🔴 RED (all tests skipped; will fail once activated before implementation)

---

## Summary

Story 5.23 is the **operational closing move for the E05 SirmaAI re-platform**. It
produces no new ingestion capability — instead it delivers:

- A per-source Celery Beat kill-switch env var (`PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED`)
- A conditional `BEAT_SCHEDULE` construction that omits crawl entries when their
  kill-switch is `"false"`
- A Phase-2 cutover runbook (`eusolicit-docs/runbooks/e05-phase-2-cutover.md`)
- A paired rollback runbook (`eusolicit-docs/runbooks/e05-phase-2-rollback.md`) with
  ≤ 15-minute SLA
- Archive marker docstrings on the legacy crawler modules
- Cross-links from existing runbooks and N8N template README/ROLLBACK.md

The two test files below define the **code-level acceptance contracts** for this story.
The runbooks, docstrings, and cross-links are documentation work — no automated tests
exist for those (AC11 link-checker is a separate script, not a pytest test).

---

## TDD Red Phase Status

✅ Red-phase test scaffolds generated

| File | Tests | Status |
|------|-------|--------|
| `tests/unit/test_beat_schedule.py` | 5 unit tests (UNIT-001 → UNIT-005) | 🔴 All `@pytest.mark.skip` |
| `tests/integration/test_phase_2_cutover_smoke.py` | 1 integration test (INT-S05.23-001) | 🔴 `@pytest.mark.skip` |

**Total: 6 test methods** (7 parametrized cases in UNIT-002; 6 parametrized cases in
UNIT-004; 4 parametrized cases in UNIT-005 → 6 + 7 + 6 + 4 - 3 base = 20 individual
test invocations once all parametrize is expanded).

---

## Acceptance Criteria Coverage

| AC | Description | Test(s) | Priority | Status |
|----|-------------|---------|----------|--------|
| AC1 | `_is_celery_crawler_enabled(source)` helper — strict `.lower() != "false"` contract | UNIT-004, UNIT-005 | P1 | 🔴 RED |
| AC2 | Conditional `BEAT_SCHEDULE` — crawl entries omitted when kill-switch=false; four unconditional entries always present | UNIT-001, UNIT-002, UNIT-003 | P0 | 🔴 RED |
| AC3 | Shadow env-var semantics documented in runbook | *(Runbook — no pytest test; AC11 link-check covers file existence)* | — | Docs |
| AC4 | Go/no-go gate query (`get_rolling_window_gate_status`) in runbook | *(Runbook — no pytest test)* | — | Docs |
| AC5 | Phase-2 cutover runbook (`e05-phase-2-cutover.md`) | *(Runbook — no pytest test; AC11 link-check)* | — | Docs |
| AC6 | Phase-2 rollback runbook (`e05-phase-2-rollback.md`) with ≤15min SLA | *(Runbook — no pytest test; AC11 link-check)* | — | Docs |
| AC7 | Archive marker docstrings on crawler + publish_event modules | *(Docstring-only; ruff/mypy is the gate)* | — | Lint |
| AC8 | Cross-links from equivalence-investigation runbook, ROLLBACK.md, README.md | *(Runbook edits — no pytest test; AC11 link-check)* | — | Docs |
| AC9 | Cutover history table (empty, pre-filled source rows) | *(Runbook content — no pytest test)* | — | Docs |
| AC10 | Unit tests UNIT-001 → UNIT-005 for kill-switch | UNIT-001, UNIT-002, UNIT-003, UNIT-004, UNIT-005 | P0/P1 | 🔴 RED |
| AC11 | Runbook link-check (`python scripts/check_runbook_url_coverage.py`) | *(CI script — not a pytest test)* | — | Script |
| AC12 | Integration smoke `test_phase_2_cutover_e2e_smoke` (in-test env-flag flip) | INT-S05.23-001 | P0 | 🔴 RED |
| AC13 | `make lint`, `make type-check`, `make test-service SVC=data-pipeline` green; coverage ≥90% on `beat_schedule.py` | *(DoD gates — not a separate test)* | — | CI |
| AC14 | No drop/delete/modify on `pipeline.crawler_runs`, crawler task bodies, S05.22 task, pipeline tables | *(Scope boundary — no positive test; verified by absence of such changes)* | — | Review |

### Coverage notes

- **AC3/AC4/AC5/AC6/AC8/AC9**: Runbook-only deliverables. No pytest test can validate
  markdown prose. The AC11 link-checker (`check_runbook_url_coverage.py`) validates
  file existence and cross-link correctness.
- **AC7**: Docstring-only banner. `make lint` (ruff) and `make type-check` (mypy) gate
  that no behavioural change sneaked in. No separate test needed.
- **AC14**: Scope-boundary AC. No positive test — reviewers verify that the diff does
  not contain table drops, consumer changes, or template payload edits.
- **AC12 (INT-S05.23-001)** is the **sole functional regression test** for this story's
  code-level contract. The cutover runbook describes the operational procedure; this test
  asserts that the underlying mechanism (env-var flip → write-path change) actually works.

---

## Test Strategy

### Detected Stack

`fullstack` (project config); this story is **backend-only** → no E2E browser tests.
No frontend changes. No i18n changes. No e2e impact.

### Test Levels Selected

| Level | Rationale |
|-------|-----------|
| **Unit** (`@pytest.mark.unit`) | `beat_schedule.py` is pure module-level configuration with no I/O — `importlib.reload + monkeypatch.setenv` is the correct isolation pattern. No DB, no Redis, no external calls. |
| **Integration** (`@pytest.mark.integration`) | The Phase-2 cutover semantic requires a real PostgreSQL + Redis stack to exercise `_handle_message` with the actual `is_equivalence_shadow_enabled()` env-read path. |

### No API/E2E Tests

This story has no new HTTP endpoints, no new Redis Streams events, and no frontend
changes. API-level tests and E2E tests are not applicable.

### Priority Assignments

| Tests | Priority | Rationale |
|-------|----------|-----------|
| UNIT-001, UNIT-002, UNIT-003 | P0 | Default-state regression guard + the kill-switch contract itself. If these break, the cutover mechanism is broken before the runbook even runs. |
| UNIT-004, UNIT-005 | P1 | Strict-truthy boundary — important for kill-switch safety but not on the cutover happy-path. |
| INT-S05.23-001 | P0 | The canonical-vs-shadow write-path contract is the sole functional change the cutover relies on. If this regresses, the cutover writes data under the wrong `source_type`. |

### Risk Inheritance from E05 Test Design

| E05 Risk | Relevance to S05.23 | Handling |
|----------|---------------------|---------|
| **E05-R-002** (AI Gateway cascade → Celery retries) | Post-cutover: Celery retries no longer happen for cut-over sources (Beat entry gone). Risk reclassifies to "out of scope for AOP after cutover". | AC2 guard (UNIT-003): unconditional `equivalence-check-daily` entry remains; if a source is accidentally re-enabled, the gate signal resumes. |
| **E05-R-006** (soft-delete bypass) | Not affected: `cleanup-expired-opportunities` is unconditional per AC2. | UNIT-003 asserts this entry is always present. |

---

## Red-Phase Activation Guide

### Per-Task Activation Instructions

Remove `@pytest.mark.skip` from each test as you implement the corresponding task:

| Task | Activates | When to Remove Skip |
|------|-----------|---------------------|
| Task 1.1 — Add `_is_celery_crawler_enabled` helper | UNIT-004, UNIT-005 | After helper is added to `beat_schedule.py` |
| Task 1.2 — Conditional `BEAT_SCHEDULE` construction | UNIT-001, UNIT-002, UNIT-003 | After `BEAT_SCHEDULE` is refactored to use `if _is_celery_crawler_enabled(...)` blocks |
| Task 3 — Integration smoke | INT-S05.23-001 | After Tasks 1–2 are done and INT-009 is still green |

### Activation workflow

```bash
# 1. Implement the feature (Task 1.1 / 1.2 / 3)
# 2. Remove @pytest.mark.skip from the relevant test(s)
# 3. Run the test — it should FAIL (confirming the red phase is real)
# 4. Complete the implementation
# 5. Run again — it should PASS (green phase achieved)
# 6. Commit the green test + implementation together

# Run unit tests only (no infra needed):
cd eusolicit-app && make test-unit

# Run the specific new unit file:
cd eusolicit-app && pytest services/data-pipeline/tests/unit/test_beat_schedule.py -v --no-header

# Run integration smoke (requires postgres + redis):
cd eusolicit-app && make infra && make migrate-all
cd eusolicit-app && pytest services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py -v -s

# Full service test suite:
cd eusolicit-app && make test-service SVC=data-pipeline
```

### Manual smoke (AC13 §8.6)

```bash
# Verify kill-switch works at the Python import level (runbook Step 4 verification):
cd eusolicit-app
PIPELINE_CELERY_CRAWL_AOP_ENABLED=false python -c \
  "from data_pipeline.workers.beat_schedule import BEAT_SCHEDULE; \
   assert 'crawl-aop' not in BEAT_SCHEDULE, 'kill-switch did not disable crawl-aop'; \
   print('OK — crawl-aop absent:', sorted(BEAT_SCHEDULE.keys()))"
```

---

## Implementation Guidance for Developer

### Files to create

| File | Description |
|------|-------------|
| `eusolicit-docs/runbooks/e05-phase-2-cutover.md` | Phase-2 cutover runbook (AC5) |
| `eusolicit-docs/runbooks/e05-phase-2-rollback.md` | Rollback runbook ≤15min SLA (AC6) |

### Files to modify

| File | Change |
|------|--------|
| `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` | Add `_is_celery_crawler_enabled(source)` helper; refactor `BEAT_SCHEDULE` to conditional construction (AC1, AC2) |
| `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` | Docstring archive banner (AC7) |
| `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` | Docstring archive banner (AC7) |
| `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` | Docstring archive banner (AC7) |
| `services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py` | "Cross-path notice" docstring banner — NOT a deletion notice (AC7) |
| `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` | Append two rows to §g escalation table (AC8) |
| `eusolicit-app/infra/n8n-templates/ROLLBACK.md` | Update "Referenced by"; add "See also" paragraph (AC8) |
| `eusolicit-app/infra/n8n-templates/README.md` | Insert Phase-1 vs Phase-2 callout in deploy section (AC8) |

### Key implementation contracts

1. **`_is_celery_crawler_enabled` helper** (beat_schedule.py):
   ```python
   def _is_celery_crawler_enabled(source: str) -> bool:
       env_key = f"PIPELINE_CELERY_CRAWL_{source.upper()}_ENABLED"
       return os.environ.get(env_key, "true").lower() != "false"
   ```
   Only the lowercase token `"false"` (after `.lower()`) returns `False`.
   Any other value — including `"1"`, `"on"`, `"yes"`, `""`, typos — returns `True`
   (fail-open: the legacy crawler keeps running unless explicitly disabled).

2. **Conditional BEAT_SCHEDULE construction**:
   ```python
   BEAT_SCHEDULE: dict[str, dict[str, Any]] = {}
   if _is_celery_crawler_enabled("aop"):
       BEAT_SCHEDULE["crawl-aop"] = {...}
   if _is_celery_crawler_enabled("ted"):
       BEAT_SCHEDULE["crawl-ted"] = {...}
   if _is_celery_crawler_enabled("eu_grants"):
       BEAT_SCHEDULE["crawl-eu-grants"] = {...}
   # Always-on entries (unconditional):
   BEAT_SCHEDULE["cleanup-expired-opportunities"] = {...}
   BEAT_SCHEDULE["enrichment-queue-worker"] = {...}
   BEAT_SCHEDULE["poll-pipeline-queue-depth"] = {...}
   BEAT_SCHEDULE["equivalence-check-daily"] = {...}
   ```

3. **Integration smoke pattern** (mirrors INT-009):
   - Seed one `sirmaai_project` row (psycopg2 + autocommit).
   - Call `_handle_message` with `PIPELINE_EQUIVALENCE_SHADOW_AOP=true` → assert shadow row.
   - Flip env var via `monkeypatch.setenv("PIPELINE_EQUIVALENCE_SHADOW_AOP", "false")`.
   - Call `_handle_message` again with a **different** `source_id_suffix` → assert canonical row.
   - Assert Phase-1 shadow row is preserved (no migration).

4. **DoD gates** (must all pass before marking story `review`):
   - `make lint` clean on `services/data-pipeline`
   - `make type-check` clean on `services/data-pipeline`
   - `make test-service SVC=data-pipeline` green (unit + integration)
   - Coverage on `beat_schedule.py` ≥ 90%
   - `python scripts/check_runbook_url_coverage.py` exits 0

---

## ATDD Artifacts

- **Checklist**: `eusolicit-docs/test-artifacts/atdd-checklist-5-23-phase-2-cutover-runbook-and-rollback.md`
- **Unit tests**: `eusolicit-app/services/data-pipeline/tests/unit/test_beat_schedule.py`
- **Integration test**: `eusolicit-app/services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py`
- **Story file**: `eusolicit-docs/implementation-artifacts/5-23-phase-2-cutover-runbook-and-rollback.md`

---

## Next Recommended Workflow

1. **`dev-story`** — Implement Story 5.23 following the task list in the story file.
   Activate each `@pytest.mark.skip` as the corresponding task is completed.
2. After implementation: **`bmad-code-review`** to validate the runbook prose and
   the `beat_schedule.py` diff against the AC contracts.
3. After code review: **`bmad-testarch-automate`** (if additional E2E coverage for the
   cutover flow is desired — currently deferred; no E2E test surface in this story).
