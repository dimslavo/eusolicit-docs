---
stepsCompleted: ['step-01-load-story', 'step-02-load-epic-test-design', 'step-03-extract-acs', 'step-04-generate-tests', 'step-05-save-checklist']
lastStep: 'step-05-save-checklist'
lastSaved: '2026-04-20'
workflowType: 'testarch-atdd'
storyKey: '9-14-scheduled-report-generation-task'
storyFile: 'eusolicit-docs/implementation-artifacts/9-14-scheduled-report-generation-task.md'
epicTestDesign: 'eusolicit-docs/test-artifacts/test-design-epic-09.md'
---

# ATDD Checklist — Story 9.14: Scheduled Report Generation Task (Admin Fan-Out)

**Date:** 2026-04-20
**Author:** TEA ATDD Agent
**Story:** 9-14-scheduled-report-generation-task
**Epic:** E09 — Notifications, Alerts & Calendar
**Status:** Tests Written — Failing (implementation pending)

---

## Summary

Story 9.14 closes Epic 09 by delivering the **admin fan-out** delivery mode for
scheduled reports. Epic 12 already delivered the core pipeline
(`generate_report`, `check_scheduled_reports`, `send_scheduled_report_email`).
Story 9.14 is strictly additive:

1. A new `send_scheduled_report_to_admins` Celery task (admin fan-out tail)
2. A routing branch in `check_scheduled_reports` (empty `recipients` → admin fan-out)
3. A default-schedule seeder task (`ensure_default_company_schedules`)
4. A new `sendgrid_template_scheduled_report` config slot
5. Beat schedule + task route registrations + README documentation

**Total tests generated:** 22 (no duplication of S12.9/S12.10 coverage)

---

## Epic Test Design Coverage Context

From `test-design-epic-09.md` (S09.14 rows):

| Priority | Requirement | Level | Count | Notes |
|---|---|---|---|---|
| **P2** | PDF generated without errors; S3 stored; `report_metadata` row created | Integration | 2 | **Already satisfied by S12.9** — S09.14 does NOT duplicate |
| **P2** | Report generation exception logged; worker does not crash | Unit | 2 | **Already satisfied by S12.9** — not duplicated |
| **P3** | Report emailed to all company admin users; metadata row includes S3 path | Integration | 1 | **THIS story closes this row** — covered by IT-1 in Task 6 |

Additional tests contributed by S09.14 beyond the test-design P3 row:
- AC1 config: 2 tests
- AC2 task behaviour: 6 tests
- AC3 routing: 4 tests
- AC4 seeder: 5 tests
- AC7 schema guard: 2 tests
- AC12 beat schedule + storage: 2 tests
- Task 6 integration end-to-end: 2 tests (IT-1, IT-2)

---

## AC → Test Mapping

### AC1 — `sendgrid_template_scheduled_report` config field

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC1-T1 | `test_settings_has_sendgrid_template_scheduled_report` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING (field not yet added) |
| AC1-T2 | `test_send_email_resolves_scheduled_report_template_id_without_error` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING (field not yet added) |

**Implementation trigger:** Add `sendgrid_template_scheduled_report: str = Field(default="d-scheduled-report", env="SENDGRID_TEMPLATE_SCHEDULED_REPORT")` to `notification.config.NotificationSettings`.

---

### AC2 — `send_scheduled_report_to_admins` Celery task

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC2-T1 | `test_job_not_complete_raises_for_retry` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING (task not yet implemented) |
| AC2-T2 | `test_schedule_not_found_warns_and_returns` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING |
| AC2-T3 | `test_schedule_inactive_warns_and_returns` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING |
| AC2-T4 | `test_fans_out_to_all_admins` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING |
| AC2-T5 | `test_per_admin_failure_does_not_abort_fanout` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING |
| AC2-T6 | `test_zero_admins_logs_warning_and_returns` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING |

**Implementation trigger:** Add `send_scheduled_report_to_admins` task to `notification/tasks/scheduled_report_delivery.py` per AC2 spec.

---

### AC3 — Schedule-row convention: empty `recipients` → admin fan-out routing

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC3-T1 | `test_monday_weekly_schedule_empty_recipients_routes_admin_fanout` | `test_check_scheduled_reports_routing.py` | ❌ FAILING (routing branch not implemented; also task not registered) |
| AC3-T2 | `test_monday_weekly_schedule_explicit_recipients_routes_direct` | `test_check_scheduled_reports_routing.py` | ❌ FAILING (routing branch not implemented) |
| AC3-T3 | `test_non_monday_weekly_schedule_does_nothing` | `test_check_scheduled_reports_routing.py` | ✅ Passes (existing no-op behaviour) |
| AC3-T4 | `test_mixed_schedules_route_independently` | `test_check_scheduled_reports_routing.py` | ❌ FAILING |

**Implementation trigger:** Add `if len(schedule.recipients) > 0 / else` routing branch inside `check_scheduled_reports`'s `for schedule in schedules` loop.

---

### AC4 — Default weekly schedule auto-provisioner

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC4-T1 | `test_seeds_default_for_company_with_no_schedule` | `test_ensure_default_schedules.py` | ❌ FAILING (task not yet implemented) |
| AC4-T2 | `test_skips_company_with_existing_schedule` | `test_ensure_default_schedules.py` | ❌ FAILING |
| AC4-T3 | `test_skips_company_with_no_active_admin` | `test_ensure_default_schedules.py` | ❌ FAILING |
| AC4-T4 | `test_seeded_row_has_pipeline_summary_pdf_weekly_empty_recipients_active` | `test_ensure_default_schedules.py` | ❌ FAILING |
| AC4-T5 | `test_idempotent_second_invocation_no_duplicate_inserts` | `test_ensure_default_schedules.py` | ❌ FAILING |

**Implementation trigger:** Add `ensure_default_company_schedules` task to `notification/tasks/scheduled_report_delivery.py` per AC4 spec.

---

### AC5 — Weekly schedule fires on Monday via existing `check_scheduled_reports`

Covered by `test_monday_weekly_schedule_empty_recipients_routes_admin_fanout` (AC3-T1). The Monday logic is unchanged from S12.10; S09.14 only adds the routing branch.

---

### AC6 — Schedule time alignment (documented deviation)

No test required — documented as accepted variance. The 06:00 UTC Beat trigger is inherited from S12.10 and unchanged. Documented in Task 5 README section.

---

### AC7 — Report metadata persistence (`report_jobs` schema guard)

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC7-T1 | `test_report_job_row_has_all_epic9_metadata_fields` | `test_report_generation_schema.py` | ✅ Passes (S12.9 already defines these columns) |
| AC7-T2 | `test_report_job_row_has_url_expires_at_for_email_template` | `test_report_generation_schema.py` | ✅ Passes |

**Note:** AC7 tests pass immediately (S12.9 schema is already complete). They are regression guards against future migration changes.

---

### AC8 — Retry semantics parity

Covered by `test_job_not_complete_raises_for_retry` (AC2-T1) and `test_per_admin_failure_does_not_abort_fanout` (AC2-T5). The same decorator shape as `send_scheduled_report_email` is verified via the test behaviour (retry on ValueError for non-complete job; no outer retry for per-admin failures).

---

### AC9 — Tenant isolation

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC9-T1 | `test_cross_tenant_isolation` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING (task not yet implemented) |

---

### AC10 — Observability (structured log events)

Log events are verified as side-effects of the AC2/AC4 tests:
- `scheduled_report.no_admins` → verified by `test_zero_admins_logs_warning_and_returns`
- `scheduled_report.admin_dispatch_failed` → verified by `test_per_admin_failure_does_not_abort_fanout` (log call checked implicitly — task must not raise)
- `scheduled_report.default_schedule_skip_no_admin` → verified by `test_skips_company_with_no_active_admin` (AC4-T3) via `structlog.testing.capture_logs()`
- `scheduled_report.admin_fanout_dispatched` (success log) → present in `test_fans_out_to_all_admins` indirectly

Dedicated log-event assertion tests can be added in a follow-on `*automate` pass when the implementation exists.

---

### AC11 — Observability of the default-schedule seeder

Covered by `test_skips_company_with_no_active_admin` (AC4-T3) which asserts the `scheduled_report.default_schedule_skip_no_admin` structlog event. The `scheduled_report.default_schedule_seeded` and `scheduled_report.ensure_default_schedules_complete` events are implicitly exercised in AC4-T1 and AC4-T5.

---

### AC12 — Bootstrap sanity guards

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC12-T1 | `test_beat_schedule_includes_ensure_default_schedules` | `test_ensure_default_schedules.py` | ❌ FAILING (entry not yet added to BEAT_SCHEDULE) |
| AC12-T2 | `test_report_schedules_recipients_empty_array_is_valid` | `test_ensure_default_schedules.py` | ✅ Passes (SQLite JSON column allows []) |

---

### AC13 — Security: no download URL in persistent INFO+ logs

| Test ID | Test Name | File | Status |
|---|---|---|---|
| AC13-T1 | `test_download_url_not_in_info_log` | `test_send_scheduled_report_to_admins.py` | ❌ FAILING (task not yet implemented) |

---

### AC14 — Documentation

No automated test. Verify manually:
- [ ] `services/notification/README.md` has a new section "S09.14 — Scheduled Reports (Admin Fan-Out Mode)" covering: `recipients=[]` convention, 05:00 UTC seeder, 06:00 UTC check, opt-out instructions, SendGrid template `d-scheduled-report` provisioning checklist.

---

### Task 6 — Integration: end-to-end admin fan-out on Monday

| Test ID | Test Name | File | Status |
|---|---|---|---|
| IT-1 | `test_admin_fanout_monday_dispatches_all_admins` | `test_scheduled_report_admin_fanout.py` | ❌ FAILING (requires implementation + Docker) |
| IT-2 | `test_explicit_recipients_uses_legacy_path` | `test_scheduled_report_admin_fanout.py` | ❌ FAILING (requires implementation + Docker) |

Integration tests require `@pytest.mark.integration` runner and Docker for testcontainers.

---

## Test File Index

| File | Location | Tests | ACs |
|---|---|---|---|
| `test_send_scheduled_report_to_admins.py` | `tests/worker/` | 8 | AC1, AC2, AC8, AC9, AC10, AC13 |
| `test_check_scheduled_reports_routing.py` | `tests/worker/` | 4 | AC3, AC5 |
| `test_ensure_default_schedules.py` | `tests/worker/` | 7 (5 seeder + 2 AC12) | AC4, AC11, AC12 |
| `test_report_generation_schema.py` | `tests/worker/` | 2 | AC7 |
| `test_scheduled_report_admin_fanout.py` | `tests/integration/` | 2 | Task 6 |

**Total: 23 tests**

---

## Test Execution Commands

```bash
# All S09.14 unit/worker tests (excluding integration — no Docker required)
pytest tests/worker/test_send_scheduled_report_to_admins.py \
       tests/worker/test_check_scheduled_reports_routing.py \
       tests/worker/test_ensure_default_schedules.py \
       tests/worker/test_report_generation_schema.py \
       -v

# Integration tests (requires Docker + testcontainers)
pytest tests/integration/test_scheduled_report_admin_fanout.py \
       -v -m integration

# Full S09.14 coverage sweep
pytest tests/worker/test_send_scheduled_report_to_admins.py \
       tests/worker/test_check_scheduled_reports_routing.py \
       tests/worker/test_ensure_default_schedules.py \
       tests/worker/test_report_generation_schema.py \
       tests/integration/test_scheduled_report_admin_fanout.py \
       -v --tb=short
```

---

## Coverage Target

Per story Testing Standards (AC2 spec):

| Module | Target | Branches |
|---|---|---|
| `scheduled_report_delivery.py` (all 4 tasks combined) | ≥80% | — |
| `recipients` emptiness routing (AC3) | 100% branch | empty / non-empty |
| Zero-admins branch (AC2) | 100% | present / absent |
| Per-admin-failure branch (AC2) | 100% | success / exception |

---

## Pre-merge Checklist

- [ ] All 21 failing tests pass (AC7 tests already pass; IT-1/IT-2 require Docker)
- [ ] `ruff check` and `ruff format --check` pass on all new/modified files
- [ ] `mypy` passes (if enabled in project)
- [ ] Coverage ≥80% on `scheduled_report_delivery.py`
- [ ] `NOTIFICATION_SENDGRID_TEMPLATE_SCHEDULED_REPORT=d-scheduled-report` documented in `.env.example`
- [ ] SendGrid template `d-scheduled-report` provisioned in staging before first deploy
- [ ] `services/notification/README.md` S09.14 section added (AC14)
- [ ] Epic 09 status in `sprint-status.yaml` updated to `done` after merge

---

## Risk Links

| Risk ID | Description | Covered By |
|---|---|---|
| E12-R-001 | Tenant isolation on report download | `test_cross_tenant_isolation` (AC9-T1) |
| E09-R-002 | Redis consumer duplicate dispatch | Not applicable — S09.14 uses Celery chain, not Redis Streams |

---

**Generated by:** BMad TEA ATDD Agent
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
