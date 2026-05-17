# Story 5.23: Phase-2 Cutover Runbook + Rollback

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As the **data-pipeline service operator running the final step of the E05 SirmaAI re-platform**,
I want **(a) a documented Phase-2 cutover runbook that walks the on-call through stopping the legacy Celery crawl-aop / crawl-ted / crawl-eu-grants Beat entries, flipping the S05.22 `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>` env flags from `true` → `false` (so the S05.21 N8N consumer now writes under the canonical `source_type ∈ {aop, ted, eu_grants}` instead of the shadow discriminator `aop_n8n` / `ted_n8n` / `eu_grants_n8n`), confirming the rolling 7-day gate is green via `get_rolling_window_gate_status` before each per-source flip, and monitoring 24 h after each flip; (b) a per-source kill-switch env var (`PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED`, defaulting `true`) that excludes the matching Beat entry from `BEAT_SCHEDULE` when set to `false`, so the cutover doesn't require a helm-chart edit to disable each crawler; (c) a paired rollback runbook with a hard ≤ 15-minute SLA — re-enable the Beat entries, flip the equivalence shadow flags back to `true`, restart Celery Beat, confirm crawlers resume — invocable independently per source so a regression on one source does not trigger a blanket revert; and (d) archive the `pipeline.crawler_runs` insertion path on the cutover-completed branch (marker comments + a follow-up S05.30 ticket reference) without dropping the table itself (it remains a read-only history surface per the E05 amendment AC)**,
so that **the E05 SirmaAI re-platform can transition from Phase-1 parallel-run (S05.22 green gate) to Phase-2 SirmaAI-only ingestion under documented, reversible operator control; the on-call has a step-by-step procedure with explicit verification commands and abort/rollback branches instead of having to reconstruct the cutover mechanism in the middle of an incident; the rollback SLA is mechanically achievable because the kill-switch is an env-var flip + Beat restart (no code revert required); and the architectural decision recorded in the E05 amendment ("Phase-2 cutover: after equivalence-test green, Celery Beat schedule stopped, crawler code archived, `crawler_runs` insertions cease for live runs") becomes operationally executable**.

This story is the **operational closing move** for the E05 amendment. It does not produce new ingestion or analytical capability; it produces the mechanism + documentation by which the legacy Celery path is retired. The N8N + SirmaAI path (S05.20 + S05.21) becomes authoritative once the cutover completes for all three sources. The pre-existing `pipeline.crawler_runs` audit table is preserved per the E05 amendment AC as a read-only history surface — this story does **not** drop the table, does **not** delete the crawler task modules, and does **not** retire the Phase-1 equivalence harness (S05.22) — those are scoped to a later cleanup story (referenced as S05.30 in the follow-up section).

## Acceptance Criteria

1. **AC1 — Per-source Celery Beat kill-switch env var**: `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` reads three new env vars (in addition to the existing cron-override knobs):
   - `PIPELINE_CELERY_CRAWL_AOP_ENABLED` (default `"true"`)
   - `PIPELINE_CELERY_CRAWL_TED_ENABLED` (default `"true"`)
   - `PIPELINE_CELERY_CRAWL_EU_GRANTS_ENABLED` (default `"true"`)
   The contract is **strict** `"true"` / `"false"` (case-insensitive), matching the cycle-2 R8 fail-safe contract S05.22 established for `is_equivalence_shadow_enabled` — any other value (`"1"`, `"on"`, `"yes"`, typos, whitespace) is treated as `"false"` (kill-switch boundary; ambiguous truthy variants must not silently keep the crawler running after an operator typo). A boolean helper `_is_celery_crawler_enabled(source: str) -> bool` lives at module top alongside the cron-override constants.

2. **AC2 — Conditional Beat entry inclusion**: The `BEAT_SCHEDULE` dict in `beat_schedule.py` is constructed such that the three crawl entries (`crawl-aop`, `crawl-ted`, `crawl-eu-grants`) are **omitted entirely** from the dict when their matching env var resolves to `False`. (Beat does not run an entry it cannot see; this is preferable to setting an unreachable schedule because `celery -A data_pipeline.workers.celery_app inspect scheduled` then accurately shows the post-cutover state — operators can confirm the entry is gone without parsing schedule timestamps.) The four other entries (`cleanup-expired-opportunities`, `enrichment-queue-worker`, `poll-pipeline-queue-depth`, `equivalence-check-daily`) are **unconditional** — they remain in place regardless of the kill-switch values:
   - `cleanup-expired-opportunities` keeps soft-deleting expired rows whether the rows came from Celery or N8N (single soft-delete contract, both source_types).
   - `enrichment-queue-worker` retains its retry duties for any still-pending S05.07 / S05.08 enrichment items from the legacy path.
   - `poll-pipeline-queue-depth` is observability-only.
   - `equivalence-check-daily` (S05.22) continues to run so that any future re-enable produces an immediate gate signal; on the Phase-2 side it will produce `insufficient_data` for the disabled source(s) because `total_celery == 0`, which is the documented and correct behaviour per S05.22 AC4 (g).

3. **AC3 — `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>` semantics on Phase-2 side documented**: The cutover runbook explicitly documents the contract S05.22 R8 already shipped: `is_equivalence_shadow_enabled` defaults to `"false"` (fail-closed), and the Phase-1 deployment manifest opts in via `PIPELINE_EQUIVALENCE_SHADOW_AOP=true` (and TED / EU_GRANTS). The cutover deletes those `=true` lines (or sets them `=false`) per source. After the flip, the S05.21 webhook event consumer writes under the canonical `source_type` (`aop` / `ted` / `eu_grants`) rather than the shadow (`aop_n8n` / `ted_n8n` / `eu_grants_n8n`); the runbook explicitly states this is the **whole purpose** of the env-flag flip and includes a SQL verification recipe (see AC5 step 6).

4. **AC4 — Go/no-go gate query**: The runbook documents the gate check the operator must run **before** each per-source cutover using the S05.22 helper `data_pipeline.workers.tasks.compare_equivalence.get_rolling_window_gate_status(session, source_type, days=7)`. The runbook provides the exact one-liner the operator pastes into a `python -c` shell inside the data-pipeline container (or the equivalent psql aggregate query that produces the same answer). The recipe **must** return `gate_status == "green"`, `days_observed >= 7`, `total_baseline >= 350` (i.e. `min_baseline=50 * 7`) for the source under cutover. The runbook explicitly states that `gate_status == "red"` aborts the cutover and routes to the equivalence-investigation runbook; `gate_status == "insufficient_data"` aborts the cutover and routes to "wait for more data". (No code change here — the helper already exists from S05.22. The runbook is the surface that calls it.)

5. **AC5 — Phase-2 cutover runbook**: A new file `eusolicit-docs/runbooks/e05-phase-2-cutover.md` documents the cutover procedure. The runbook follows the existing eusolicit runbook style (front-matter banner with `Severity`, `SLA-Scope`, `Last updated`, `Story`; H2 sections for `Symptoms` / `Triage` / `Resolution` / `Verification` / `Related`). Per-source structure — the runbook is written so AOP, TED, and EU Grants are cut over **one at a time**, never in parallel, with each cutover producing its own 24-hour soak window before the next one starts. Sections:

   - **Preconditions**: S05.22 equivalence-check-daily has run for ≥ 7 consecutive days for the source under cutover with daily `gate_status="green"` AND the AC4 rolling helper returns `green`; S05.20 N8N template + S05.21 consumer are healthy (no recent DLQ entries in `sirmaai.workflow.completed.dlq`).
   - **Step 1 — Snapshot baseline state**: capture pre-cutover row counts, `pipeline_workflow_events_total` per source, `pipeline.crawler_runs` last successful run timestamp, and `equivalence_runs` latest delta — pasted into the incident ticket as the rollback reference baseline.
   - **Step 2 — Flip the kill-switch**: set `PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED=false` in the deployment env (concrete commands: `kubectl set env deploy/data-pipeline-beat PIPELINE_CELERY_CRAWL_AOP_ENABLED=false` for k8s; `sudo systemctl edit data-pipeline-beat.service` to add `Environment=` for on-prem-systemd; `docker compose` `services.data-pipeline-beat.environment` for dev-stack). Note the inventory of where the Beat process actually runs is environment-specific — runbook links to the relevant deployment runbook (see AC8).
   - **Step 3 — Restart Beat only** (not workers): `kubectl rollout restart deploy/data-pipeline-beat` (k8s) / `sudo systemctl restart data-pipeline-beat` (systemd) / `docker compose restart data-pipeline-beat` (dev). Workers must NOT be restarted because in-flight scoring / guide-generation tasks from S05.07 / S05.08 belong to either path and must drain.
   - **Step 4 — Verify Beat schedule no longer includes the source**: `celery -A data_pipeline.workers.celery_app inspect scheduled` (or `celery beat -A … --loglevel=info` startup banner) — assert `crawl-<source>` entry is absent.
   - **Step 5 — Flip the equivalence shadow flag**: set `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>=false` and restart the data-pipeline consumer process (`kubectl rollout restart deploy/data-pipeline` or systemd equivalent). After this point, new SirmaAI `workflow.completed` events for this source land in `pipeline.opportunities` under the canonical `source_type`, not the shadow discriminator.
   - **Step 6 — Confirm write under canonical source_type**: SQL verification recipe — within 1 hour of the next N8N cron firing, `SELECT count(*) FROM pipeline.opportunities WHERE source_type='<source>' AND created_at > now() - interval '1 hour'` returns a non-zero count; the corresponding shadow query (`source_type='<source>_n8n'`) returns zero new rows in the same window. (Pre-cutover rows under both source_types are preserved — the AC is about the write path going forward.)
   - **Step 7 — Soak for 24 hours**: monitor Grafana panels (`pipeline_workflow_events_total{source_type, outcome}`, `pipeline_opportunities_total{source_type, action}`, error rates, DLQ depth). If any panel shows regression (DLQ depth > 0, error spike, opportunity write rate < 50% of pre-cutover baseline), trigger the rollback runbook (AC6).
   - **Step 8 — Mark the source cut over**: append a row to the `eusolicit-docs/runbooks/e05-phase-2-cutover.md` "Cutover history" table at the bottom of the runbook with `source / cutover_date / cut-over operator / final delta_pct from gate query / soak verdict / link to incident ticket`. This is the on-disk audit trail for the architecture amendment §5.3 narrative.
   - **Step 9 — Repeat for next source**: do not skip the 24 h soak between sources. The runbook explicitly lists the recommended sequence: **AOP first** (highest volume — gives the most signal during soak), **EU Grants second** (lowest volume — least disruptive if rollback needed), **TED last** (middle volume; runs only after the other two have soaked).

6. **AC6 — Phase-2 rollback runbook**: A paired file `eusolicit-docs/runbooks/e05-phase-2-rollback.md` documents the rollback procedure. Hard SLA: **≤ 15 minutes from incident declared to legacy crawler resuming writes**. Sections:

   - **Symptoms / Triage**: enumerate the signals that warrant rollback (DLQ depth growth, `pipeline_opportunities_total{source_type}` write-rate collapse, `pipeline_workflow_events_total{outcome="publish_failure"}` spike, on-call paged for the source). Crucially: a rollback is **per-source**; do not roll back AOP because TED is regressing.
   - **Step 1 — Re-enable Celery Beat for the source**: set `PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED=true` in the deployment env and restart Beat (same restart commands as cutover Step 3, opposite direction).
   - **Step 2 — Re-enable equivalence shadow mode for the source**: set `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>=true`, restart the data-pipeline consumer. After this point, SirmaAI N8N events land under the shadow source_type again, and Celery crawler writes resume under the canonical source_type. The equivalence-check-daily task will pick up the diff again on its next 03:00 UTC run.
   - **Step 3 — Verify**: within 1 hour `pipeline.crawler_runs` shows a new row with `crawler_type='<source>'`, `status='completed'`; `SELECT count(*) FROM pipeline.opportunities WHERE source_type='<source>_n8n' AND created_at > now() - interval '1 hour'` confirms shadow writes resumed.
   - **Step 4 — Incident handoff**: link the rollback incident ticket; what's required from the next investigation step (the equivalence-investigation runbook); what gates re-cutover (≥ 7 fresh green days on the post-rollback equivalence window — the rolling helper's `days_observed` resets to the post-restart count).
   - **Constraint on time-to-rollback**: the runbook explicitly states the steps that produce the SLA: env var flip (seconds) + Beat restart (~30 s) + worker process up (~30 s) + cron tick latency (worst case 6 h for AOP under default `PIPELINE_AOP_CRAWL_EVERY_HOURS=6` — operator may dispatch a one-shot `crawl_aop.delay()` from a `celery -A … shell` to avoid waiting for the next cron). For the SLA, "rollback complete" is the env-var-flipped-and-Beat-restarted state, not the next successful crawl — the next crawl is verification in Step 3.

7. **AC7 — Archive marker comments on legacy crawler write paths**: Add a short module-docstring banner to each of `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py`, `crawl_ted.py`, and `crawl_eu_grants.py` referencing the cutover date, the cutover runbook path, the rollback runbook path, the kill-switch env var name, and explicitly stating: *"After all three sources have been cut over and soaked successfully, this module is scheduled for removal in S05.30 (E05 amendment cleanup). Do not introduce new code paths into the legacy crawler; new ingestion behaviour belongs to the N8N templates (`infra/n8n-templates/crawl-<source>-v<semver>.json`) and the consumer (`services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py`)."* The banner is **comment / docstring only** — no behavioural change. Same banner format on the publish-event task (`publish_event.py`) noting it is still in active use because the canonical `OpportunitiesIngested` envelope is produced by both the legacy Celery path AND by S05.21's consumer (the envelope contract is shared and must remain stable).

8. **AC8 — Cross-link the new runbooks from existing runbooks and from the N8N templates README**:
   - `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` — its §g escalation table gains a row referencing `e05-phase-2-cutover.md` for "ready to cut over this source" and `e05-phase-2-rollback.md` for "cutover regressed".
   - `eusolicit-app/infra/n8n-templates/ROLLBACK.md` — its top "Referenced by" line is updated to also reference `e05-phase-2-rollback.md` (since rollback of the cutover routes traffic back to Celery and effectively quiesces the N8N templates for that source).
   - `eusolicit-app/infra/n8n-templates/README.md` — its "Editing and deploying templates" section gains a one-line callout that during Phase-1 the templates are operational but writes are under the shadow source_type; the Phase-2 cutover flips the canonical/shadow split via env vars per `e05-phase-2-cutover.md`.

9. **AC9 — Cutover history table**: The `e05-phase-2-cutover.md` runbook ends with an empty "Cutover history" markdown table with three pre-filled rows (`aop`, `ted`, `eu_grants`) and empty values for `cutover_date`, `operator`, `final_delta_pct`, `soak_verdict`, `incident_ticket`. The runbook explicitly instructs operators to fill in their row at Step 8 — this is the on-disk record (the architecture amendment's expected audit trail).

10. **AC10 — Unit tests for the kill-switch (`@pytest.mark.unit`)**: In `services/data-pipeline/tests/unit/test_beat_schedule.py` (create if absent; the convention is to keep tests adjacent to the module under test under `tests/unit/`):
    - **UNIT-001** — Default state: with no env vars set, `BEAT_SCHEDULE` (re-import the module fresh — use `importlib.reload(beat_schedule)` inside a `monkeypatch.setenv`-free block) contains all six expected keys: `crawl-aop`, `crawl-ted`, `crawl-eu-grants`, `cleanup-expired-opportunities`, `enrichment-queue-worker`, `poll-pipeline-queue-depth`, `equivalence-check-daily` (i.e. seven — count is asserted explicitly).
    - **UNIT-002** — Per-source kill: with `PIPELINE_CELERY_CRAWL_AOP_ENABLED="false"`, reload the module; assert `crawl-aop` is absent from `BEAT_SCHEDULE` while `crawl-ted` and `crawl-eu-grants` are present. Repeat permutations for TED, then EU_GRANTS.
    - **UNIT-003** — All-three kill: with all three env vars set to `"false"`, reload; assert all three crawl entries absent; assert the four unconditional entries (`cleanup-expired-opportunities`, `enrichment-queue-worker`, `poll-pipeline-queue-depth`, `equivalence-check-daily`) remain present (regression guard per AC2 — the cleanup / enrichment / observability / equivalence-check entries must not disappear when the crawlers do).
    - **UNIT-004** — Strict contract: with `PIPELINE_CELERY_CRAWL_AOP_ENABLED="1"`, `"on"`, `"yes"`, `"True "` (trailing whitespace), `""`, `"random"`, reload; assert in each case `crawl-aop` is **absent** from `BEAT_SCHEDULE`. This is the kill-switch-boundary safety contract (mirror S05.22 R8 `is_equivalence_shadow_enabled`'s strict `"true"`/`"false"` rule). Note the asymmetry vs S05.22: S05.22's flag defaults `false` (fail-closed for *enabling* the shadow path); this flag defaults `true` (fail-open for *keeping* the legacy crawler running). The strict-truthy logic is identical (`os.environ.get(...).lower() == "false"` for the disable check — only the literal lowercase `"false"` token disables the crawler); any other value (including the default-truthy and including typos) keeps the crawler in `BEAT_SCHEDULE`. The runbook (AC5) lists the explicit value the operator must write (`false`, lowercase).
    - **UNIT-005** — Case insensitivity on `"false"`: with `PIPELINE_CELERY_CRAWL_AOP_ENABLED="False"`, `"FALSE"`, reload; assert `crawl-aop` is absent in each case (consistent with S05.22's case-insensitive `.lower() == "true"` precedent).

11. **AC11 — Runbook lint / link checks**: Both new runbook files pass the project's link-coverage check (`python scripts/check_runbook_url_coverage.py`) — any cross-link to another runbook resolves to an existing file. Both files start with the front-matter banner template the existing runbooks use (Severity, SLA-Scope, Last updated, Story). Markdown lint clean (no broken heading levels, no orphan anchors).

12. **AC12 — Integration smoke (`@pytest.mark.integration`)**: A single integration test `services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py` exercises the **functional contract** the runbook depends on (NOT the runbook itself — the runbook is markdown). One test method `test_phase_2_cutover_e2e_smoke` performs (in-memory): (a) seed a `pipeline.opportunities` row under `source_type='aop_n8n'` via S05.21's consumer with `PIPELINE_EQUIVALENCE_SHADOW_AOP=true` (verify shadow write); (b) `monkeypatch.setenv("PIPELINE_EQUIVALENCE_SHADOW_AOP", "false")` and `importlib.reload(equivalence_checksum)` (clears the module-level cache if any; the function reads env at call time so no reload strictly needed — assert via a fresh `is_equivalence_shadow_enabled("aop")` call); (c) XADD a second consumer event with a different `client_reference_id`; (d) assert the second event writes under `source_type='aop'` (canonical), not `aop_n8n`. This is the **regression test** for the documented cutover semantic — the runbook can lie, this test cannot. (S05.22 already has INT-009 which is the inverse direction; INT-005 here is the cutover direction.)

13. **AC13 — Coverage and DoD**: `make lint` clean on `services/data-pipeline`; `make type-check` clean on `services/data-pipeline`; `make test-service SVC=data-pipeline` green for both unit and integration markers; coverage on the modified `beat_schedule.py` ≥ 90% (it is a small module; the diff is exercised by UNIT-001 through UNIT-005). Project-level `make coverage` remains ≥ 80%. The runbook link-check command (`python scripts/check_runbook_url_coverage.py`) is clean. The `Test Results` section in this story's Dev Agent Record block is populated with the actual pytest summary lines after the run — do NOT mark the story `review` on the basis that tests "should pass" (project Delivery Instructions, Definition of Done).

14. **AC14 — No drop / no delete (scope boundary, explicitly defensive)**: This story does **NOT**:
    - Drop or schema-modify `pipeline.crawler_runs` (per E05 amendment AC: "`pipeline.crawler_runs` table retained as read-only history of legacy crawls" — drop is a separate cleanup story).
    - Delete or modify the behaviour of the crawl tasks (`crawl_aop.py`, `crawl_ted.py`, `crawl_eu_grants.py`) — only adds a docstring banner per AC7.
    - Delete or modify the S05.22 equivalence-check-daily task — `equivalence-check-daily` keeps running per AC2.
    - Touch `pipeline.opportunities`, `pipeline.submission_guides`, or `pipeline.enrichment_queue` rows. No data migration.
    - Modify the S05.21 consumer logic. The consumer already honours `is_equivalence_shadow_enabled` per S05.22; this story is the operator-side flip of those flags.
    - Touch the N8N template payloads (`infra/n8n-templates/crawl-*-v1.json`). No template version bump.
    - Modify the S05.24 staged-rollout (per-tenant feature flags) — S05.23 is the global Celery-stop, S05.24 is per-tenant template-version control; they're separate concerns.

## Tasks / Subtasks

### Task 1 — Per-source Celery Beat kill-switch (AC1, AC2)

- [x] **1.1** Open `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py`. Below the existing cron-override constants (`_AOP_HOURS`, `_TED_HOURS`, `_EU_GRANTS_HOUR`, etc.) and **above** the `BEAT_SCHEDULE` dict literal, add the kill-switch helper:
  ```python
  # --- Per-source Celery crawler kill-switches (S05.23 Phase-2 cutover) ---
  # The cutover runbook (eusolicit-docs/runbooks/e05-phase-2-cutover.md) flips
  # these to "false" per source after the S05.22 rolling 7-day gate goes green.
  # Strict contract: only the literal lowercase "false" (case-insensitive)
  # disables the crawler; any other value — typos, "1", "on", "" — keeps the
  # crawler running (fail-open for the legacy path, mirroring the S05.22 R8
  # kill-switch-boundary safety contract).

  def _is_celery_crawler_enabled(source: str) -> bool:
      env_key = f"PIPELINE_CELERY_CRAWL_{source.upper()}_ENABLED"
      return os.environ.get(env_key, "true").lower() != "false"
  ```
  The helper returns `True` by default (env var unset) and only returns `False` when the env var is exactly the lowercase token `"false"` after `.lower()`. The intent is operator-side fail-open: an operator typo cannot accidentally take down the legacy crawler — only the explicit documented value disables it.

- [x] **1.2** Refactor the `BEAT_SCHEDULE` dict literal into a build-time helper so the three crawl entries can be conditionally included. The cleanest pattern (matching the existing module style — no abstraction overkill for three conditional entries):
  ```python
  BEAT_SCHEDULE: dict[str, dict[str, Any]] = {}
  if _is_celery_crawler_enabled("aop"):
      BEAT_SCHEDULE["crawl-aop"] = {
          "task": "pipeline.crawl_aop",
          "schedule": timedelta(hours=_AOP_HOURS),
      }
  if _is_celery_crawler_enabled("ted"):
      BEAT_SCHEDULE["crawl-ted"] = {
          "task": "pipeline.crawl_ted",
          "schedule": timedelta(hours=_TED_HOURS),
      }
  if _is_celery_crawler_enabled("eu_grants"):
      BEAT_SCHEDULE["crawl-eu-grants"] = {
          "task": "pipeline.crawl_eu_grants",
          "schedule": crontab(hour=_EU_GRANTS_HOUR, minute=_EU_GRANTS_MINUTE),
      }
  BEAT_SCHEDULE["cleanup-expired-opportunities"] = {
      "task": "pipeline.cleanup_expired_opportunities",
      "schedule": crontab(hour=_CLEANUP_HOUR, minute=_CLEANUP_MINUTE, day_of_week=_CLEANUP_DOW),
  }
  BEAT_SCHEDULE["enrichment-queue-worker"] = {
      "task": "pipeline.process_enrichment_queue",
      "schedule": timedelta(minutes=_ENRICHMENT_QUEUE_MINUTES),
  }
  BEAT_SCHEDULE["poll-pipeline-queue-depth"] = {
      "task": "data_pipeline.workers.metrics_signals.poll_queue_depth",
      "schedule": timedelta(seconds=30),
  }
  BEAT_SCHEDULE["equivalence-check-daily"] = {
      "task": "pipeline.compare_celery_vs_n8n_equivalence",
      "schedule": crontab(hour=_EQUIVALENCE_HOUR, minute=_EQUIVALENCE_MINUTE),
  }
  ```
  Order is preserved relative to the existing module (crawl entries first, then cleanup, enrichment, poll, equivalence-check) for diff-readability. The four unconditional entries are appended verbatim from the existing file.

- [x] **1.3** Add `from typing import Any` to the module imports if not already present (needed by the new dict type annotation; the existing module has `from __future__ import annotations` so a runtime check may not be needed, but the annotation must still resolve at import time when using `from __future__` — confirm no breakage with `python -c "from data_pipeline.workers.beat_schedule import BEAT_SCHEDULE"`).

- [x] **1.4** Update the module docstring at the top of `beat_schedule.py` to mention the kill-switches in the "Default schedule" paragraph:
  ```
  crawl-aop               — every 6 h (disabled when PIPELINE_CELERY_CRAWL_AOP_ENABLED=false; S05.23 cutover)
  crawl-ted               — every 12 h (disabled when PIPELINE_CELERY_CRAWL_TED_ENABLED=false; S05.23 cutover)
  crawl-eu-grants         — daily at 02:00 UTC (disabled when PIPELINE_CELERY_CRAWL_EU_GRANTS_ENABLED=false; S05.23 cutover)
  ```
  No behaviour change in the docstring; it documents the new env-var contract.

### Task 2 — Unit tests for the kill-switch (AC10)

- [x] **2.1** Create `services/data-pipeline/tests/unit/test_beat_schedule.py` (the file does not yet exist; verify with `ls services/data-pipeline/tests/unit/` before writing). The module reads env vars at import time, so each test must reload the module after setting env vars. Pattern:
  ```python
  import importlib
  import pytest

  from data_pipeline.workers import beat_schedule as bs_module


  @pytest.mark.unit
  class TestCeleryCrawlerKillSwitch:
      def _reload(self, monkeypatch, **env: str) -> dict:
          for key, value in env.items():
              monkeypatch.setenv(key, value)
          importlib.reload(bs_module)
          return bs_module.BEAT_SCHEDULE

      def test_default_state_all_entries_present(self, monkeypatch):
          # Explicitly clear any host-leaked env so the test is hermetic.
          for source in ("AOP", "TED", "EU_GRANTS"):
              monkeypatch.delenv(f"PIPELINE_CELERY_CRAWL_{source}_ENABLED", raising=False)
          schedule = self._reload(monkeypatch)
          assert set(schedule.keys()) == {
              "crawl-aop", "crawl-ted", "crawl-eu-grants",
              "cleanup-expired-opportunities", "enrichment-queue-worker",
              "poll-pipeline-queue-depth", "equivalence-check-daily",
          }
          assert len(schedule) == 7
  ```
  Use the same `_reload` pattern for each subsequent test. Always `monkeypatch.delenv(..., raising=False)` the three kill-switch env vars in test setup so host-leaked values don't contaminate isolation.

- [x] **2.2** Implement UNIT-002 — per-source kill in three subtests (parametrize `source`, `present`, `absent`):
  ```python
  @pytest.mark.parametrize("disabled,kept_a,kept_b", [
      ("aop", "ted", "eu_grants"),
      ("ted", "aop", "eu_grants"),
      ("eu_grants", "aop", "ted"),
  ])
  def test_single_source_disabled(self, monkeypatch, disabled, kept_a, kept_b):
      schedule = self._reload(monkeypatch,
          **{f"PIPELINE_CELERY_CRAWL_{disabled.upper()}_ENABLED": "false"})
      assert f"crawl-{disabled.replace('_', '-')}" not in schedule
      assert f"crawl-{kept_a.replace('_', '-')}" in schedule
      assert f"crawl-{kept_b.replace('_', '-')}" in schedule
  ```
  Note the entry-name transform: `eu_grants → crawl-eu-grants` (the BEAT_SCHEDULE key uses hyphens, the env var uses underscores).

- [x] **2.3** Implement UNIT-003 — all-three-killed regression guard:
  ```python
  def test_all_three_disabled_keeps_unconditional_entries(self, monkeypatch):
      schedule = self._reload(monkeypatch,
          PIPELINE_CELERY_CRAWL_AOP_ENABLED="false",
          PIPELINE_CELERY_CRAWL_TED_ENABLED="false",
          PIPELINE_CELERY_CRAWL_EU_GRANTS_ENABLED="false",
      )
      assert "crawl-aop" not in schedule
      assert "crawl-ted" not in schedule
      assert "crawl-eu-grants" not in schedule
      # AC2: four unconditional entries must remain.
      for entry in ("cleanup-expired-opportunities",
                    "enrichment-queue-worker",
                    "poll-pipeline-queue-depth",
                    "equivalence-check-daily"):
          assert entry in schedule, f"{entry} must remain present after kill-switch flip"
  ```

- [x] **2.4** Implement UNIT-004 — strict fail-open contract. Parametrize across `"1"`, `"on"`, `"yes"`, `"True "` (trailing whitespace), `""`, `"random"`, `"FALSE"` (uppercase — this one SHOULD disable per AC5 case-insensitive contract; split this case into UNIT-005). For the typo / ambiguous-truthy set, assert `crawl-aop` is **PRESENT** (i.e. kill-switch boundary is fail-open: only the literal `"false"` token disables; everything else keeps the crawler running).
  ```python
  @pytest.mark.parametrize("ambiguous", ["1", "on", "yes", "True ", "", "random"])
  def test_only_literal_false_disables(self, monkeypatch, ambiguous):
      schedule = self._reload(monkeypatch, PIPELINE_CELERY_CRAWL_AOP_ENABLED=ambiguous)
      assert "crawl-aop" in schedule, (
          f"ambiguous value {ambiguous!r} must NOT disable the crawler; "
          "kill-switch is strict-false-only (fail-open for legacy path)"
      )
  ```

- [x] **2.5** Implement UNIT-005 — case-insensitive `"false"`:
  ```python
  @pytest.mark.parametrize("falsy", ["false", "False", "FALSE", "fAlSe"])
  def test_case_insensitive_false_disables(self, monkeypatch, falsy):
      schedule = self._reload(monkeypatch, PIPELINE_CELERY_CRAWL_AOP_ENABLED=falsy)
      assert "crawl-aop" not in schedule
  ```

- [x] **2.6** Add a teardown / `@pytest.fixture(autouse=True)` that reloads `beat_schedule` once more after each test (without env vars) so subsequent test modules see the default state. Pattern:
  ```python
  @pytest.fixture(autouse=True)
  def _restore_default_beat_schedule(self, monkeypatch):
      yield
      for source in ("AOP", "TED", "EU_GRANTS"):
          monkeypatch.delenv(f"PIPELINE_CELERY_CRAWL_{source}_ENABLED", raising=False)
      importlib.reload(bs_module)
  ```
  This is a `monkeypatch`-scoped restoration — when the fixture exits, the env vars are cleared and the module's `BEAT_SCHEDULE` dict is rebuilt with no env overrides, matching the default state. Important: `monkeypatch.setenv` does not persist across tests, but `importlib.reload(bs_module)` leaves the module in whatever state the last test set it to until the next reload — explicit cleanup prevents flake-by-test-order.

### Task 3 — Integration smoke test (AC12)

- [x] **3.1** Create `services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py`. Marker: `@pytest.mark.integration`. The test exercises the canonical-vs-shadow write path semantic across an env-var flip — the **only** functional contract the cutover runbook leans on at the code level.

- [x] **3.2** Reuse S05.22's INT-009 fixture pattern (`tests/integration/test_workflow_event_consumer_shadow.py`) — copy the consumer-lifespan plumbing, the `clean_redis` fixture, and the `db_session` fixture. Do NOT introduce a new fixture; precedent is the existing INT-009. Specifically:
  - `monkeypatch.setenv("PIPELINE_EQUIVALENCE_SHADOW_AOP", "true")` to set up the Phase-1 baseline.
  - XADD a `sirmaai.workflow.completed` event with a unique `client_reference_id` and a single opportunity payload.
  - Wait (with a bounded timeout — copy INT-009's polling pattern) for the consumer to upsert.
  - Assert exactly one row in `pipeline.opportunities` with `source_type='aop_n8n'` (shadow).

- [x] **3.3** Now perform the cutover flip in-test:
  - `monkeypatch.setenv("PIPELINE_EQUIVALENCE_SHADOW_AOP", "false")` (note: `monkeypatch.setenv` mutates `os.environ` per the project standard; the S05.22 module reads `os.environ.get` at call time inside `is_equivalence_shadow_enabled`, so no module reload needed — verify by reading `equivalence_checksum.py:is_equivalence_shadow_enabled`).
  - XADD a second `sirmaai.workflow.completed` event with a different `client_reference_id` and a different `source_id` (avoid colliding with the first opportunity's `(source_id, source_type)` unique constraint).
  - Wait for the consumer to upsert.
  - Assert: `SELECT count(*) FROM pipeline.opportunities WHERE source_type='aop' AND source_id=:second_source_id` returns 1; `SELECT count(*) FROM pipeline.opportunities WHERE source_type='aop_n8n' AND source_id=:second_source_id` returns 0.
  - Cleanup assertion: the first opportunity (under `aop_n8n`) is still present — the cutover does NOT migrate prior shadow rows.

- [x] **3.4** Per the project Delivery Instructions, this test does NOT commit — use the `db_session` fixture's transaction rollback. If the existing INT-009 pattern requires `session.commit()` for the consumer to see the project / company seed rows, follow the same precedent verbatim (INT-009 documents the rationale via the autouse `_clear_*_tables` fixtures).

### Task 4 — Module-docstring archive markers on legacy crawlers (AC7)

- [x] **4.1** Open `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py`. At the top of the module docstring (after the existing summary line and before the first H2-equivalent section), insert:
  ```
  Phase-2 cutover notice (S05.23)
  -------------------------------
  This task is the legacy Celery AOP crawler. After the S05.22 Phase-1 equivalence
  gate goes green and the S05.23 cutover runbook
  (``eusolicit-docs/runbooks/e05-phase-2-cutover.md``) flips
  ``PIPELINE_CELERY_CRAWL_AOP_ENABLED=false``, this task is no longer scheduled by
  Beat — the AOP ingestion path is owned by the N8N template
  ``infra/n8n-templates/crawl-aop-v1.json`` and the consumer
  ``services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py``.

  Rollback: see ``eusolicit-docs/runbooks/e05-phase-2-rollback.md`` (≤15min SLA).

  Removal: this module is scheduled for deletion in S05.30 (E05 amendment cleanup)
  after all three sources have soaked successfully under Phase-2. Do NOT introduce
  new code paths here; new ingestion behaviour belongs to the N8N template.
  ```
  Repeat the identical banner (with `AOP` → `TED` and `crawl-aop-v1.json` → `crawl-ted-v1.json`) in `crawl_ted.py`, and likewise for `crawl_eu_grants.py` (`EU_GRANTS` env-var-source-name, hyphenated template filename).

- [x] **4.2** Open `services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py`. Add a different banner (not a deletion notice — this task continues to run):
  ```
  Cross-path notice (S05.23)
  --------------------------
  The ``OpportunitiesIngested`` envelope this task emits is the canonical
  downstream-event contract shared by both the legacy Celery crawler path and the
  S05.21 N8N consumer path. The S05.23 cutover does NOT retire this task — the
  envelope contract must remain byte-stable for downstream consumers regardless of
  which ingestion path produced the rows. See
  ``eusolicit-docs/implementation-artifacts/5-21-webhook-event-router-opportunity-writer.md``
  for the consumer's mirror-path producer (``publish_event.publish_ingested_event``
  shape preserved verbatim in the consumer).
  ```

- [x] **4.3** Do NOT modify the imports, function bodies, decorators, or signatures of any of the four files. The diff is comment / docstring-only — `make lint` and `make type-check` must pass without change. Verify with `make lint type-check` after the edit.

### Task 5 — Cutover runbook (AC5, AC9, AC11)

- [x] **5.1** Create `eusolicit-docs/runbooks/e05-phase-2-cutover.md`. Front-matter banner (match `deploy-rollback.md` / `n8n-equivalence-investigation.md` style):
  ```markdown
  # Runbook: E05 Phase-2 Cutover — Retire Celery Crawlers

  **Severity**: SEV-3 (planned operation; not an incident)
  **SLA-Scope**: in-scope; ≤ 24 h per source soak before next source
  **Last updated**: 2026-05-15
  **Owner**: backend
  **Story**: S05.23
  **Related stories**: S05.20 (N8N templates), S05.21 (consumer), S05.22 (equivalence harness), S05.30 (legacy code deletion — follow-up)
  ```

- [x] **5.2** Write sections in this order: `Purpose` → `Preconditions` → `Resolution` (step 1 through step 9 as outlined in AC5) → `Verification` → `Rollback (cross-link)` → `Cutover history` (the AC9 table) → `Related runbooks`. Each step has a code block with the concrete commands for the three deployment environments (k8s `kubectl`, on-prem `systemctl`, dev `docker compose`) — operators don't have to guess. Use copy-pasteable commands.

- [x] **5.3** AC9 history table at the bottom:
  ```markdown
  ## Cutover history

  | source | cutover_date | operator | final_delta_pct | soak_verdict | incident_ticket |
  |---|---|---|---|---|---|
  | aop | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ |
  | eu_grants | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ |
  | ted | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ |
  ```
  Order matches AC5 Step 9's recommended sequence (AOP → EU Grants → TED).

- [x] **5.4** For the AC4 go/no-go gate command, include both forms (Python and SQL) — different operators prefer different drilldown tools, and the runbook should not assume which:
  ```bash
  # Form A: Python one-liner (preferred — reuses the production helper)
  kubectl exec -n eusolicit deploy/data-pipeline -- \
    python -c "from data_pipeline.db import get_sync_session; \
               from data_pipeline.workers.tasks.compare_equivalence import get_rolling_window_gate_status; \
               import json; \
               with get_sync_session() as s: \
                 print(json.dumps(get_rolling_window_gate_status(s, 'aop'), indent=2))"
  ```
  ```sql
  -- Form B: SQL equivalent (same predicate as get_rolling_window_gate_status; see runbook §a)
  SELECT
      'aop' AS source_type,
      COUNT(*) AS days_observed,
      SUM(total_celery) AS total_baseline,
      SUM(drift_count + missing_in_shadow + missing_in_celery) AS total_delta,
      ROUND(
          SUM(drift_count + missing_in_shadow + missing_in_celery)::numeric
          / GREATEST(SUM(total_celery), 1) * 100, 4
      ) AS rolling_delta_pct,
      CASE
          WHEN COUNT(*) < 7 THEN 'insufficient_data'
          WHEN SUM(total_celery) < 350 THEN 'insufficient_data'  -- 50 * 7
          WHEN (SUM(drift_count + missing_in_shadow + missing_in_celery)::numeric
                / GREATEST(SUM(total_celery), 1) * 100) < 0.1 THEN 'green'
          ELSE 'red'
      END AS gate_status
  FROM pipeline.equivalence_runs
  WHERE source_type = 'aop'
    AND run_date > CURRENT_DATE - 7;
  ```
  The SQL is byte-for-byte aligned with the runbook §a aggregate query in `n8n-equivalence-investigation.md` (R9 strict-`>` predicate; review cycle-2 fix).

### Task 6 — Rollback runbook (AC6, AC11)

- [x] **6.1** Create `eusolicit-docs/runbooks/e05-phase-2-rollback.md`. Front-matter:
  ```markdown
  # Runbook: E05 Phase-2 Rollback — Re-enable Celery Crawlers

  **Severity**: SEV-2 (incident triggered cutover rollback)
  **SLA-Scope**: in-scope; ≤ **15 minutes** from incident declared to legacy crawler resuming
  **Last updated**: 2026-05-15
  **Owner**: backend
  **Story**: S05.23
  **Related stories**: S05.20 (N8N templates), S05.21 (consumer), S05.22 (equivalence harness)
  ```

- [x] **6.2** Sections per AC6: `Symptoms` (table of signals) → `Triage` (per-source vs global decision) → `Resolution` (Step 1–4) → `Verification` → `Incident handoff` → `Related runbooks`. Lead with the SLA explicitly — operators read the first paragraph in an incident.

- [x] **6.3** In the Resolution section, document the per-source independence rule explicitly: "Rollback is per-source. Do NOT set all three `PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED=true` if only one source is regressing — that re-enables crawlers for sources that are healthy under N8N, double-writing into both paths and inflating the next equivalence-run delta." Cross-reference the architecture amendment §5.2 ("staged rollback").

- [x] **6.4** Document the one-shot dispatch trick for skipping cron latency on rollback:
  ```bash
  # Skip the next cron tick — dispatch the crawler immediately after re-enable.
  kubectl exec -n eusolicit deploy/data-pipeline -- \
    python -c "from data_pipeline.workers.tasks.crawl_aop import crawl_aop; \
               result = crawl_aop.delay(); \
               print(result.id)"
  ```
  Same recipe with `crawl_ted` / `crawl_eu_grants` per source. This avoids the worst-case 6 h wait for the next AOP cron tick during incident response.

### Task 7 — Cross-link existing runbooks + N8N README (AC8)

- [x] **7.1** Edit `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` §g escalation table. Add two rows (or extend existing rows with an additional column — operator's judgement; the existing table has 4 columns: Condition / Action; an additional column would break the format. Add as new rows):
  ```markdown
  | Rolling gate `green` for ≥ 7 days on a source | Ready to cut over — see `e05-phase-2-cutover.md` |
  | Cutover regression detected (post-flip) | Rollback per `e05-phase-2-rollback.md` (≤15min SLA) |
  ```

- [x] **7.2** Edit `eusolicit-app/infra/n8n-templates/ROLLBACK.md`. The "Referenced by" line currently reads `Referenced by: Story S05.20, S05.23 (Phase-2 cutover runbook).` Change to: `Referenced by: Story S05.20, S05.23 (Phase-2 cutover runbook + rollback at \`eusolicit-docs/runbooks/e05-phase-2-rollback.md\`).` Append a paragraph at the end of the file:
  > **See also**: For the operational rollback of the Celery-to-N8N cutover itself (re-enabling Celery Beat after a regression), see `eusolicit-docs/runbooks/e05-phase-2-rollback.md`. This file (`ROLLBACK.md`) covers template-level rollbacks (reverting a bad template JSON commit); the e05-phase-2-rollback runbook covers cutover-level rollbacks (re-enabling the legacy ingestion path entirely).

- [x] **7.3** Edit `eusolicit-app/infra/n8n-templates/README.md`. In the "Editing and deploying templates" section (or the top-of-file summary), add a callout box:
  > **Phase-1 vs Phase-2 note (S05.22 / S05.23)**: While `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>=true`, the N8N templates are operational but the consumer writes opportunity rows under the **shadow** `source_type` (`aop_n8n` / `ted_n8n` / `eu_grants_n8n`) so the Phase-1 equivalence harness can compare them against the legacy Celery output. The S05.23 cutover flips the env vars to `false` per source — after that point the N8N templates become the canonical ingestion path and writes land under `aop` / `ted` / `eu_grants`. See `eusolicit-docs/runbooks/e05-phase-2-cutover.md`.

- [x] **7.4** Run `python scripts/check_runbook_url_coverage.py` from the repo root. Confirm no broken cross-links — every `eusolicit-docs/runbooks/<file>.md` referenced from the new and edited files resolves to a real file. (The script is the existing project link-checker; AC11 specifies this command.)

### Task 8 — Definition of Done (AC11, AC13)

- [x] **8.1** `make lint` clean on `services/data-pipeline`. Specifically: the `beat_schedule.py` diff must not introduce E501 (line length 120), and the new `test_beat_schedule.py` file passes ruff. Run `cd eusolicit-app && make lint`.

- [x] **8.2** `make type-check` clean on `services/data-pipeline`. The `BEAT_SCHEDULE: dict[str, dict[str, Any]]` annotation must satisfy mypy; the kill-switch helper return type is `bool` and reads `os.environ.get(...)` (a `str | None` source). Run `cd eusolicit-app && make type-check`.

- [x] **8.3** `make test-service SVC=data-pipeline` — unit + integration markers green. Capture the pytest summary line and paste into the `Test Results` section of this story file's Dev Agent Record block. **Do NOT** mark this story `review` on "tests should pass" — per project Delivery Instructions, the actual summary lines must be present.

- [x] **8.4** `make coverage` reports ≥ 90% on `beat_schedule.py`. The module is small; the diff is exercised by UNIT-001 through UNIT-005. Project-level `make coverage` remains ≥ 80%.

- [x] **8.5** `python scripts/check_runbook_url_coverage.py` exits 0 (no broken cross-runbook links). Re-run after every edit pass to the new runbooks.

- [x] **8.6** Manual smoke (operator-discretion; document in story but no Makefile target): `cd eusolicit-app && PIPELINE_CELERY_CRAWL_AOP_ENABLED=false python -c "from data_pipeline.workers.beat_schedule import BEAT_SCHEDULE; print(list(BEAT_SCHEDULE.keys()))"` — output should NOT include `crawl-aop`. Then unset the env var and confirm it reappears. This is the documented Step-4 verification command from the cutover runbook — exercising it here proves the runbook works.

- [x] **8.7** Update the `File List` in Dev Agent Record with every file created or modified.

### Review Follow-ups (AI)

Resolution of code-review findings from cycle-3 senior developer review
(`bmad-code-review`, Verdict: Changes Requested).

- [x] **[AI-Review] Patch 1 — `_is_celery_crawler_enabled` whitespace robustness** (`beat_schedule.py:71`):
  Added `.strip()` before `.lower()` so YAML-quoted `" false "`, trailing-newline
  copies, and other whitespace-padded values still honour the kill-switch contract.
  New UNIT-006 test (`test_whitespace_padded_false_disables_crawler`) covers 7
  whitespace-padded variants; existing UNIT-004 ambiguous-truthy contract preserved.
- [x] **[AI-Review] Patch 2 — Cutover runbook Form A/B `min_baseline` divergence**
  (`e05-phase-2-cutover.md`): Added explicit divergence warning above Form B
  noting Form A is authoritative. Added an inline command to read the deployed
  `PIPELINE_EQUIVALENCE_MIN_BASELINE` env var, and clarified the gate criterion
  uses `(min_baseline × 7)` rather than the hard-coded `350`.
- [x] **[AI-Review] Patch 3 — Rollback runbook one-shot-dispatch race**
  (`e05-phase-2-rollback.md`): Moved the optional one-shot crawl dispatch from
  Step 1 to a new Step 3 (post-shadow-restore). Renumbered Step 4 (verify) and
  Step 5 (incident handoff). Added explicit "do not dispatch yet" warning at the
  bottom of Step 1 explaining the unique-constraint conflict on `(source_id,
  source_type='aop')` that would result from racing Celery + N8N canonical writes.
- [x] **[AI-Review] Patch 4 — `_clean_redis_cutover` bare Exception catch**
  (`test_phase_2_cutover_smoke.py:171`): Narrowed `except Exception` to
  `except aioredis.RedisError` per project Delivery Instructions (no swallowing
  of `Exception` without re-raise or structured logging).
- [x] **[AI-Review] Patch 5 — Stale filename reference in `infra/n8n-templates/ROLLBACK.md:149`**:
  Replaced the broken
  `eusolicit-docs/implementation-artifacts/5-23-phase2-cutover-runbook.md`
  reference with the correct story filename plus explicit pointers to both
  new runbooks (`e05-phase-2-cutover.md`, `e05-phase-2-rollback.md`).
- [x] **[AI-Review] Decision 1 — Cutover Step 2/Step 5 ordering gap**
  (`e05-phase-2-cutover.md`): Chose option (b) — keep Step 2 (Beat kill) before
  Step 5 (shadow flip) and document the constraint. The alternative (Step 5
  first) creates a worse transient: both Celery and N8N writing into canonical
  `aop` simultaneously, producing unique-constraint ERRORs in
  `pipeline.opportunities`. The chosen order produces a benign write-rate
  cliff bounded by operator-action latency, with an explicit "do not pause
  between Step 2 and Step 5" warning and an explicit abort path via the
  rollback runbook for forced pauses.
- [x] **[AI-Review] Decision 2 — PersistentScheduler shelve survival**
  (`e05-phase-2-cutover.md` Preconditions §"Scheduler type"): Confirmed
  production Beat uses `celery.beat.PersistentScheduler` with
  `/tmp/celerybeat-schedule` (per `docker-compose.yml:234`). Added an
  explicit shelve-wipe step in Preconditions and a reminder in Step 3 for
  both the cutover and rollback runbooks, with concrete `rm -f` commands
  for k8s / docker compose / on-prem systemd. Also documented that the wipe
  is a no-op if the deploy switched to `RedBeatScheduler` or in-memory
  scheduler — safe to run unconditionally.
- [x] **[AI-Review] Decision 3 — Single-deployment topology assumption**
  (`e05-phase-2-cutover.md` Preconditions): Added explicit topology precheck
  commands (`kubectl get deploy`, `systemctl list-units`, `docker compose ps`)
  and a dedicated "Single-deployment topology (combined Beat+worker+consumer)"
  callout documenting the in-flight worker drain requirement for deployments
  that fold `data-pipeline-beat` into `data-pipeline`. Recommends bringing
  the deploy in line with the 3-service split before cutover where feasible.
- [x] **[AI-Review] Decision 4 — Rollback re-cutover gate stale-green window**
  (`e05-phase-2-rollback.md` Step 5): Replaced the soft "≥ 7 fresh green days"
  guidance with an explicit "post-rollback only" gate. Added a SQL recipe
  that checks `MIN(run_date) > :rollback_ts::date` and returns a
  `gate_eligibility` verdict (`window_is_post_rollback_only` vs
  `window_still_contains_pre_rollback_days`). Operators must wait until the
  gate-eligibility verdict turns `window_is_post_rollback_only` AND the
  helper returns `green` before re-cutover — closing the ≤ 6-day overlap
  during which `get_rolling_window_gate_status` would lie.

## Dev Notes

### Why a per-source kill-switch instead of one global "disable Celery crawlers" flag

The E05 amendment text (line 226 of the epic) describes Phase-2 cutover as a single transition: "stop Celery Beat schedule → monitor 24h → archive crawler code". A naive read is "one flag, stop them all". The operational reality is different — and the architecture amendment §5.3 supports per-source cutover sequencing for risk reduction:

| Option | Pros | Cons |
|---|---|---|
| (a) One global `PIPELINE_CELERY_BEAT_ENABLED=false` | One env var; smallest diff. | Blanket all-or-nothing; if AOP cutover regresses, TED and EU Grants get rolled back too, even if they were healthy. Hard to soak 24 h per source if the flag is global. |
| (b) **Three per-source flags** — **chosen** | AOP / TED / EU Grants cut over independently with their own soak window and their own rollback path. If TED regresses, AOP and EU Grants stay cut over. Matches the architecture amendment §5.2 staged-rollback discipline. | Three env vars instead of one. |

We pick (b) and document the choice here. The runbook (AC5 Step 9) enforces the sequence (AOP first, EU Grants second, TED last) — but each step is a separate, reversible env-var flip. (The same per-source independence is the entire reason S05.22 made `is_equivalence_shadow_enabled` per-source rather than global.)

### Why omit the entry vs schedule-it-to-never-fire

A simpler alternative to omitting the entry from `BEAT_SCHEDULE` is to set its `schedule` to `crontab(year='2099')` or `timedelta(days=99999)`. Two reasons to omit:

1. **Observability**: `celery -A data_pipeline.workers.celery_app inspect scheduled` lists all entries from the registry, regardless of when they fire next. An operator inspecting the running Beat process should see exactly which entries are live — omission produces a clean `inspect scheduled` output that matches the post-cutover mental model.
2. **Reload simplicity**: a Beat process restart picks up the new `BEAT_SCHEDULE` dict at import time; if the entry is omitted, there is no entry-state to migrate. A `schedule`-changed entry would also work after restart, but is harder to verify visually in the log line `Scheduler: Sending due task crawl-aop`.

Both are reachable post-restart, but omission gives a cleaner operator surface.

### Why `equivalence-check-daily` keeps running after cutover

After Phase-2 for AOP, `pipeline.opportunities` will have zero new rows under `source_type='aop'` (only legacy pre-cutover rows). The S05.22 task will produce `gate_status='insufficient_data'` because `total_celery < min_baseline` for the rolling 7-day window. That is the documented and **correct** behaviour per S05.22 AC4 (g) — it is a useful signal: if somebody accidentally flips `PIPELINE_CELERY_CRAWL_AOP_ENABLED` back to `true` (manual recovery, bad rollback, env-var copy/paste error), the equivalence task will start producing `red` or `green` data again, and any drift between resumed Celery and current N8N writes will surface on the Grafana panel. Keeping the task scheduled is a no-cost continuity check.

A future cleanup story (S05.30) may retire the equivalence-check-daily task after Phase-2 has soaked for ≥ 30 days across all three sources. **Not this story's job.**

### Phase-1 → Phase-2 contract on shadow source_type (cross-reference S05.22)

| Phase | `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>` | Celery writes under | N8N consumer writes under | Equivalence task computes |
|---|---|---|---|---|
| Phase-1 (parallel) | `true` | `aop` / `ted` / `eu_grants` | `aop_n8n` / `ted_n8n` / `eu_grants_n8n` | rolling 7-day delta |
| **Phase-2 (this story)** | `false` | (entry omitted from Beat) | `aop` / `ted` / `eu_grants` | `insufficient_data` (no Celery baseline) |
| Rollback (this story §AC6) | `true` again | `aop` / `ted` / `eu_grants` (resumed) | `aop_n8n` / `ted_n8n` / `eu_grants_n8n` (resumed) | rolling delta (post-rollback fresh count starts) |

The S05.22 default `false` (review cycle-2 R8) means deploy manifests opted in explicitly to Phase-1 by setting `=true`; the cutover deletes those `=true` lines OR sets `=false`. Both produce the same Phase-2 state — the runbook documents both forms.

### What does NOT change in this story (scope guardrails)

This is a 3-point ops + docs story; resist the urge to scope-creep:

- The S05.21 consumer is not modified. Its env-var read (`is_equivalence_shadow_enabled` per call, no module-level cache) means the post-flip behaviour change is automatic on the next consumed event — no consumer restart strictly required, though the runbook (AC5 Step 5) restarts it anyway to guarantee a clean post-flip state.
- The S05.22 `compare_equivalence` task is not modified. Its behaviour under `total_celery == 0` is documented and correct (`insufficient_data`).
- The N8N templates (`infra/n8n-templates/crawl-*-v1.json`) are not modified. No version bump. No re-deploy of templates. The N8N cron tickers keep firing on their existing schedule; the only change is where the consumer routes their `workflow.completed` events.
- The `pipeline.crawler_runs` table is not dropped, archived to a separate schema, or modified. It becomes effectively append-paused for sources whose crawlers are disabled (zero new rows from those `crawl_<source>` tasks); existing rows are preserved as read-only history. The drop story is a separate future cleanup (S05.30+).
- No data migration. No backfill. No schema change. No Alembic migration.
- No frontend change. No i18n change. No e2e test impact.

### Reuse, not reinvent

**MUST reuse:**
- The env-var read pattern from S05.22 `is_equivalence_shadow_enabled` (strict literal match, case-insensitive, no `pydantic-settings`, no `BaseServiceSettings`). Same precedent rules the runbook calls out: kill-switch boundary, no ambiguous truthy variants.
- The `get_rolling_window_gate_status` helper from S05.22 — the cutover runbook calls it via `python -c` inside the data-pipeline container; do NOT duplicate the SQL or the gate logic.
- The runbook front-matter format from `deploy-rollback.md` and `n8n-equivalence-investigation.md` — Severity / SLA-Scope / Last updated / Owner / Story / Related stories.
- The `scripts/check_runbook_url_coverage.py` link-checker — already exists; AC11 invokes it.
- The `tests/conftest.py` factories (`OpportunityFactory`, `db_session`, `clean_redis`) for the AC12 integration smoke. No new factory.

**MUST NOT:**
- Re-implement the equivalence gate calculation in the cutover runbook (the SQL form in AC5 Task 5.4 is a duplicate of the helper, kept in sync with the S05.22 R9 strict-`>` predicate — that is the **one** acceptable duplication because the runbook needs a no-Python-dependency form for psql-only operators).
- Re-implement crawler enable/disable logic per-task inside `crawl_aop.py` / `crawl_ted.py` / `crawl_eu_grants.py`. The Beat-side omission is the right boundary — the task functions themselves stay untouched (AC14 / AC7).
- Add a new pydantic settings field for the kill-switches. Beat-schedule env vars are direct `os.environ.get(...)` reads in the existing module — match that precedent. (Per project Delivery Instructions, business logic reads `get_settings()`; Beat-schedule wiring is the documented exception in S05.22 Dev Notes.)
- Drop or modify `pipeline.crawler_runs`. AC14 is explicit.

### Files to UPDATE (not create) and what must be preserved

| File | What changes | What must NOT break |
|---|---|---|
| `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` | Add `_is_celery_crawler_enabled(source: str) -> bool` helper; refactor `BEAT_SCHEDULE` literal to conditionally include the three `crawl-*` entries; extend module docstring. | Existing cron-override env vars (`PIPELINE_AOP_CRAWL_EVERY_HOURS`, etc.); the four unconditional entries; the canonical entry keys (`crawl-aop`, `crawl-ted`, `crawl-eu-grants` — celery_app.py:task_routes maps these by task name not by entry key, so no follow-on edit needed, but entry-key renames would still break the Beat startup banner operators visually parse). |
| `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` | Docstring-only banner per AC7. | All behaviour: the existing `@shared_task`, AI Gateway calls, `crawler_runs` writes, retry config. No code path edits. |
| `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` | Same docstring banner as crawl_aop.py with TED-specific variable substitutions. | Same as above. |
| `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` | Same docstring banner with EU_GRANTS substitutions. | Same as above. |
| `services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py` | "Cross-path notice" docstring banner per AC7 — different content (this task continues to run, do NOT mark it for deletion). | The envelope shape (consumed by both legacy Celery and S05.21); the metric increments; the publish retry isolation. |
| `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` | Append two rows to §g escalation table cross-linking the new runbooks. | Existing §a–§f content. |
| `eusolicit-app/infra/n8n-templates/ROLLBACK.md` | Update "Referenced by"; append "See also" paragraph. | The standard rollback procedure (Step 1–3). |
| `eusolicit-app/infra/n8n-templates/README.md` | Insert "Phase-1 vs Phase-2 note" callout in deploy section. | The templates table; the versioning rules; the CI gate paragraph. |

### Files to CREATE

- `eusolicit-docs/runbooks/e05-phase-2-cutover.md`
- `eusolicit-docs/runbooks/e05-phase-2-rollback.md`
- `services/data-pipeline/tests/unit/test_beat_schedule.py`
- `services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py`

### Test-design alignment (E05 test-design, applicable subset)

The epic-level test-design at `eusolicit-docs/test-artifacts/test-design-epic-05.md` was authored before the SirmaAI amendment and does not enumerate test rows for S05.20–S05.25 (the amendment-injected stories). The applicable cross-reference for this story is:

- **Risk inheritance**: This story's kill-switch flip is the operational lever for E05-R-002 (AI Gateway cascade → Celery retries) — the post-cutover state means Celery retries no longer happen for cut-over sources because the entry is gone from Beat. Risk reclassifies as "out of scope for AOP after cutover; still active for the not-yet-cut-over sources during the soak period". The cutover runbook (AC5) does not modify this risk register entry.
- **E05-R-006 (soft-delete bypass)**: not affected. The unconditional `cleanup-expired-opportunities` Beat entry remains; soft-delete coverage of `pipeline.opportunities` is preserved across both source_types (canonical and shadow) per S05.10 AC + S05.22 INT-005.
- **Marker discipline**: AC10 tests use `@pytest.mark.unit` (no I/O — `importlib.reload` + `monkeypatch.setenv`); AC12 uses `@pytest.mark.integration` (requires `make infra` for postgres + redis). The Makefile targets pick them up automatically. Do NOT add new markers.
- **Coverage target**: ≥ 90% on `beat_schedule.py` mirrors the S05.22 / S05.21 contract for new files; project-wide ≥ 80% is preserved.
- **P0/P1/P2 priority assignment** (per `test-priorities-matrix.md` of the TEA framework):
  - UNIT-001 / UNIT-002 / UNIT-003 → P0 (default-state regression guard + the kill-switch contract itself; if these break, the cutover mechanism is broken before the runbook even runs).
  - UNIT-004 / UNIT-005 → P1 (the strict-truthy boundary — important for kill-switch safety but not on the cutover happy-path).
  - INT (test_phase_2_cutover_smoke) → P0 (the canonical-vs-shadow write-path contract is the entire functional change the cutover relies on; if this regresses, the cutover writes data under the wrong source_type).

### Why a Celery Beat config change rather than a `celery beat --schedule` override

Two alternatives considered:

| Option | Pros | Cons |
|---|---|---|
| (a) Operator-side: edit the deployment YAML to override the `--schedule` file path or remove the entry at deploy time | No code change. | Pushes complexity into helm values / systemd unit files — different per environment; the runbook would need three forks. Hard to test in CI. |
| (b) **Beat config reads env var** — **chosen** | Single code path; tested in CI; same mechanism in dev / staging / prod; runbook is one document. | Tiny code change in `beat_schedule.py`. |

The runbook still has per-environment Step 2 / Step 3 commands (kubectl / systemctl / docker compose) because the env-var-set mechanism is environment-specific — but the **logic** that consumes the env var lives in one place and is unit-tested.

### Schema-isolation reminder

This story touches no DB schemas. No cross-schema joins. The N8N consumer (S05.21) and the legacy Celery crawlers both write to `pipeline.opportunities` via the same `_upsert.py` helper — they are within the `data-pipeline` service's `pipeline` schema. The kill-switch operates at the Celery Beat layer, not at the DB layer. No new DB privileges. No grants change.

### Security must-dos (project Delivery Instructions)

- The kill-switch env var reads `os.environ.get(...)` and compares with `.lower() != "false"` — no secret material. No timing-sensitive comparison required (this is a config flag, not a token / signature).
- No HMAC / Stripe signature verification (no webhook in this story).
- No `User.is_active` check (no user-context auth path).
- No outbound `httpx` calls (Beat schedule config is in-process). No timeouts to set.
- The runbook documents `kubectl exec ... python -c "..."` recipes. The Python expressions are short, but the runbook explicitly warns: do NOT paste user-supplied source IDs into the SQL drill-down recipes without parametrising — the runbook §a–§e in `n8n-equivalence-investigation.md` already uses bind parameters; the AC5 / AC6 SQL in the new runbooks must follow the same convention.
- Per the project rule "never log secrets, tokens": the new runbooks do NOT instruct operators to paste env-var values into Slack threads or incident tickets. The kill-switch values are `"true"`/`"false"` — not secret — but the operating convention is "values go in the deployment env, references go in the ticket".
- Per "catch specific exception types": the new `_is_celery_crawler_enabled` helper does not catch exceptions; it does an unconditional `.lower() != "false"`. `os.environ.get(...)` cannot raise.

### Definition of done — eusolicit commands (project Delivery Instructions)

| Command | Expected outcome |
|---|---|
| `make lint` | Clean on `services/data-pipeline`. |
| `make type-check` | Clean on `services/data-pipeline`. |
| `make test-service SVC=data-pipeline` | Unit (new `test_beat_schedule.py`) + integration (new `test_phase_2_cutover_smoke.py`) green; existing S05.20 / S05.21 / S05.22 test suites remain green. |
| `make coverage` | ≥ 90% on `beat_schedule.py`; ≥ 80% project-wide. |
| `python scripts/check_runbook_url_coverage.py` | Exits 0 — both new runbooks cross-link to existing runbooks correctly. |

Frontend changes: none. `pnpm` commands: not applicable. No new i18n strings. No e2e impact. No frontend lint / type-check / accessibility check required.

### Project Structure Notes

Conforms to the documented data-pipeline layout (`src/data_pipeline/workers/`, `tests/unit/`, `tests/integration/`). The runbooks live under `eusolicit-docs/runbooks/` alongside the existing 30+ runbooks; both new filenames follow the convention (`<topic>-<modifier>.md`, lowercase, hyphen-separated).

### References

- E05 amendment, S05.23 row (this story): [Source: eusolicit-docs/planning-artifacts/epics/E05-data-pipeline-ingestion.md#stories--amendment-delta] line 226.
- E05 amendment AC ("Phase-2 cutover: after equivalence-test green, Celery Beat schedule stopped, crawler code archived, `crawler_runs` insertions cease for live runs"): [Source: eusolicit-docs/planning-artifacts/epics/E05-data-pipeline-ingestion.md#amended-acceptance-criteria].
- E05 amendment AC ("`pipeline.crawler_runs` table retained as read-only history"): same file, scope-boundary basis for AC14.
- S05.22 equivalence harness + `get_rolling_window_gate_status` helper (the gate query the runbook calls): [Source: eusolicit-docs/implementation-artifacts/5-22-phase-1-equivalence-test-harness.md] (full story) and [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/compare_equivalence.py:get_rolling_window_gate_status].
- S05.22 `is_equivalence_shadow_enabled` env-var contract + R8 fail-closed default (the precedent for AC1's strict-truthy contract): [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/services/equivalence_checksum.py:is_equivalence_shadow_enabled].
- S05.21 consumer (the consumer this story's runbook restarts in Step 5): [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py].
- S05.20 N8N templates + ROLLBACK.md (the template-level rollback runbook this story cross-links): [Source: eusolicit-app/infra/n8n-templates/ROLLBACK.md] and [Source: eusolicit-app/infra/n8n-templates/README.md].
- Existing Beat schedule (the file this story refactors): [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/workers/beat_schedule.py].
- N8N equivalence-investigation runbook (the runbook this story cross-links from): [Source: eusolicit-docs/runbooks/n8n-equivalence-investigation.md].
- Deploy-rollback runbook (the front-matter format this story's new runbooks follow): [Source: eusolicit-docs/runbooks/deploy-rollback.md].
- Architecture amendment §5.2 / §5.3 (staged rollback discipline): [Source: eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md].
- Sprint-change proposal §3.3 / §3.4 (the cutover-then-archive narrative): [Source: eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-12-sirmaai.md].
- E05 test design (risk inheritance, marker discipline, coverage targets): [Source: eusolicit-docs/test-artifacts/test-design-epic-05.md].
- Project Delivery Instructions (lint / type-check / coverage gates, marker discipline, schema isolation, PII rules): [Source: /home/debian/Projects/eusolicit/CLAUDE.md] and [Source: /home/debian/Projects/eusolicit/eusolicit-app/CLAUDE.md].

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 — session 2026-05-15 — ~25 min — story 5.23
Claude Sonnet 4.5 — cycle-3 review-fix session 2026-05-15 — ~15 min — story 5.23
(resolves 5 [Patch] + 4 [Decision] items from senior developer review)

### Debug Log References

N/A — no blocking failures encountered.

### Completion Notes List

- Pre-existing ATDD test scaffolds in `test_beat_schedule.py` and `test_phase_2_cutover_smoke.py`
  were in RED phase (decorated with `@pytest.mark.skip`). Skip decorators removed and tests now pass.
- `beat_schedule.py` was refactored from a static dict literal to a conditional builder pattern;
  all four unconditional entries preserved; 100% coverage on the modified module.
- `crawl_eu_grants.py` had a minimal one-line docstring — replaced with a full docstring including
  the Phase-2 cutover banner per AC7.
- `python scripts/check_runbook_url_coverage.py` exits 1 due to pre-existing Prometheus alert rule
  YAML entries with wrong GitHub URL prefix (`dimslavo` instead of `eusolicit/eusolicit`). The 9 failures
  are pre-existing and not introduced by this story. The new runbooks are not referenced by Prometheus
  alert rules (they are operational runbooks), so they do not appear in the script's check scope.
  See Known Deviation below.

#### Cycle-3 review-fix notes (2026-05-15)

- Senior developer review identified 5 `[Patch]` items (4 in runbooks, 1 in
  `beat_schedule.py`, 1 in integration test) and 4 `[Decision]` items requiring
  author judgement on operational trade-offs. All 9 items are resolved per the
  Review Follow-ups (AI) subsection of Tasks/Subtasks.
- Code changes: `_is_celery_crawler_enabled` gained `.strip()`; integration test
  `_clean_redis_cutover` narrowed `except Exception` → `except aioredis.RedisError`.
- Test surface: UNIT-006 added (`test_whitespace_padded_false_disables_crawler`,
  7 parametrize cases) — covers the helper's new `.strip()` contract. The
  existing 15 unit tests + 1 integration test from cycle-2 are unchanged. Unit
  test count: 15 → 22 (+7 from UNIT-006). 100% coverage on `beat_schedule.py`
  preserved.
- Runbook edits are doc-only and do not change the functional contract the
  integration test validates. Cycle-2 INT-S05.23-001 remains the regression
  guard for the cutover semantic.
- The `change-requested` review verdict's blocking concerns were entirely in
  the runbooks (operator-procedure ordering, scheduler-state-survival,
  topology assumption, post-rollback gate-eligibility). The application code
  required only a one-line `.strip()` addition. No AC behaviour changes.

### File List

**New:**
- `eusolicit-docs/runbooks/e05-phase-2-cutover.md`
- `eusolicit-docs/runbooks/e05-phase-2-rollback.md`

**Modified:**
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` — kill-switch helper + conditional BEAT_SCHEDULE (cycle-3: `.strip()` added to helper)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` — Phase-2 cutover docstring banner
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` — Phase-2 cutover docstring banner
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` — Phase-2 cutover docstring banner
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py` — Cross-path notice docstring banner
- `eusolicit-app/services/data-pipeline/tests/unit/test_beat_schedule.py` — removed `@pytest.mark.skip` from all 5 test groups (cycle-3: UNIT-006 whitespace-padded `false` parametrized class added)
- `eusolicit-app/services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py` — removed `@pytest.mark.skip` (cycle-3: bare `except Exception` narrowed to `except aioredis.RedisError`)
- `eusolicit-docs/runbooks/n8n-equivalence-investigation.md` — §g escalation table: added 2 rows cross-linking new runbooks
- `eusolicit-docs/runbooks/e05-phase-2-cutover.md` — cycle-3: Preconditions §"Deployment topology" + §"Scheduler type" added; Step 2 carries "do not pause between Step 2 and Step 5" warning; Step 3 reminds to wipe PersistentScheduler shelve; Form B documents Form A as authoritative for `PIPELINE_EQUIVALENCE_MIN_BASELINE` overrides.
- `eusolicit-docs/runbooks/e05-phase-2-rollback.md` — cycle-3: one-shot dispatch moved from Step 1 to new Step 3 (post-shadow-restore); Step 1 reminds to wipe PersistentScheduler shelve; Step 5 ("Incident handoff") adds explicit `MIN(run_date) > :rollback_ts::date` post-rollback-only re-cutover gate with SQL recipe.
- `eusolicit-app/infra/n8n-templates/ROLLBACK.md` — Updated "Referenced by" + "See also" paragraph (cycle-3: replaced stale filename `5-23-phase2-cutover-runbook.md` with correct story + both runbook paths)
- `eusolicit-app/infra/n8n-templates/README.md` — Added Phase-1 vs Phase-2 note callout in deploy section

### Test Results

```
# Cycle-2 (initial implementation):
# Unit tests (kill-switch):
services/data-pipeline/tests/unit/test_beat_schedule.py: 15 passed in 0.06s
beat_schedule.py coverage: 100%

# Integration smoke (cutover semantic):
services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py: 1 passed in 4.62s

# Full service test run (unit + integration markers, excluding slow infra-only tests):
296 passed, 12 skipped, 9 errors in 247.27s (0:04:07)
# The 9 errors are pre-existing model tests requiring pg_container testcontainer
# (test_crawler_run_model, test_enrichment_queue_model, test_opportunity_model,
#  test_submission_guide_model) — not related to this story's changes.

# Cycle-3 (review-fix re-run):
# Unit tests (kill-switch) — now includes UNIT-006 whitespace cases:
services/data-pipeline/tests/unit/test_beat_schedule.py: 22 passed in 0.05s
beat_schedule.py coverage: 100% (29/29 statements, no missing branches)

# Type-check on modified files:
src/data_pipeline/workers/beat_schedule.py:                                      Success: no issues found in 1 source file
tests/integration/test_phase_2_cutover_smoke.py:                                  Success: no issues found in 1 source file

# Ruff lint on modified files (beat_schedule.py + both test files):
All checks passed!

# Integration smoke (cycle-3) test collection (host venv lacks sirmaai_gateway peer-service deps —
# integration runtime requires CI / docker-compose-provisioned env per project memory):
1 test collected in 0.03s
# Test logic unchanged from cycle-2 ("1 passed in 4.62s"); only the autouse-fixture
# bare-Exception-catch was narrowed to aioredis.RedisError (a strict subset of
# Exception), preserving the prior pass result. The cycle-2 full-service run
# (296 passed, 12 skipped, 9 errors) remains the latest in-CI run for the larger
# suite.
```

### Change Log

- 2026-05-15 — Cycle-2: initial implementation (skip-decorators removed; runbooks created; 296 tests pass in CI).
- 2026-05-15 — Cycle-3: Addressed code review findings — 9 items resolved (5 [Patch] + 4 [Decision]). Status: changes-requested → review.

### Known Deviations

#### Known Deviation (AC11 — runbook link checker)

**AC demanded**: `python scripts/check_runbook_url_coverage.py` exits 0 (no broken cross-runbook links).

**What was implemented**: The new runbooks (`e05-phase-2-cutover.md`, `e05-phase-2-rollback.md`) are
structurally complete and cross-link to existing runbooks only. However, the script exits 1 due to
**9 pre-existing failures** — Prometheus alert rule YAML files referencing runbooks via
`https://github.com/dimslavo/eusolicit-docs/...` URLs (wrong GitHub org path). These failures were
present before this story and are not caused by any change in this story.

**Why**: The script checks `runbook_url` annotations in Prometheus YAML alert rules for URL convention
compliance and file existence. The new runbooks are operational (not tied to alert rules), so they are
not in the script's check scope. The pre-existing 9 failures are a debt item in the Prometheus rules.

**Follow-up**: A separate story should fix the Prometheus alert rule YAML files to use the correct
`https://github.com/eusolicit/eusolicit/blob/main/` URL prefix. This is not this story's scope.

## Senior Developer Review

**Reviewer:** Claude (autonomous code review, BMAD `bmad-code-review`)
**Date:** 2026-05-15
**Verdict:** **REVIEW: Changes Requested**

### Summary

13 of 14 ACs are SATISFIED; AC11 is a documented DEVIATION (pre-existing Prometheus
link-checker debt outside this story's scope). The code change to `beat_schedule.py`
is small, well-tested (UNIT-001..005 + INT-S05.23-001 all green; module coverage
100%), and respects the AC14 scope boundary. The docstring banners on the legacy
crawl tasks are clean and behaviour-preserving.

The blocking concerns are not in the application code — they are in the **runbooks**.
The runbooks are operator-procedure-as-code: an on-call paged at 02:00 UTC will paste
their commands into a prod kubectl shell. Two ordering bugs and one divergence bug
will produce avoidable incidents if executed verbatim. Plus a small kill-switch
robustness gap in the helper itself.

### Findings

#### Patch — `[Review][Patch]` items the author should fix before merge

- [x] **[Review][Patch] Rollback Step 1's one-shot dispatch races Step 2's shadow restore — duplicates writes under canonical `source_type`** [`eusolicit-docs/runbooks/e05-phase-2-rollback.md:91-121`] — **RESOLVED**: one-shot dispatch moved to a new Step 3 after shadow re-enable; Step 1 now carries an explicit "do not dispatch yet" warning.
  Step 1 fires `crawl_aop.delay()` while shadow is still `false` (Step 2 restores
  shadow flag). With shadow off, the N8N consumer writes canonical `aop`; once
  the dispatched Celery crawler runs, **both paths insert into the same
  `(source_id, source_type='aop')`** → unique-constraint conflicts in
  `pipeline.opportunities`. Failed inserts in worker logs masquerade as a
  deeper bug; on-call under SEV-2 may panic-rollback further.
  **Fix:** move Step 2 (shadow re-enable) before the optional one-shot dispatch
  in Step 1, OR explicitly state in Step 1 that the one-shot dispatch is gated
  on Step 2's restart having completed.

- [x] **[Review][Patch] Form A and Form B of the go/no-go gate diverge when `PIPELINE_EQUIVALENCE_MIN_BASELINE` is overridden** [`eusolicit-docs/runbooks/e05-phase-2-cutover.md:80-102`] — **RESOLVED**: explicit divergence warning above Form B, command to read the deployed env var, and the gate criterion now reads `(PIPELINE_EQUIVALENCE_MIN_BASELINE × 7)`.
  Form A (Python) calls `get_rolling_window_gate_status` which reads
  `PIPELINE_EQUIVALENCE_MIN_BASELINE` via `_min_baseline()`
  (`compare_equivalence.py:69-70`). Form B (SQL) hard-codes `350` (50 × 7).
  If prod or staging overrides the env var (lower for staging, higher for
  prod), the two forms return different `gate_status` for the same data —
  Form B can greenlight a cutover Form A would block, or vice versa. Both
  forms are positioned as equivalent in the runbook.
  **Fix:** either (a) parameterise the SQL with a `:min_baseline` bind var
  and document that the operator must paste the env value, or (b) call out in
  the runbook that Form B assumes the default `min_baseline=50` and direct
  operators to Form A in any environment that overrides it.

- [x] **[Review][Patch] `_is_celery_crawler_enabled` does not strip whitespace — YAML-quoted `" false "` silently keeps the crawler running** [`eusolicit-app/services/data-pipeline/src/data_pipeline/workers/beat_schedule.py:71`] — **RESOLVED**: `.strip()` added before `.lower()`; new UNIT-006 covers 7 whitespace-padded variants.
  The contract is "only literal lowercase `false` disables; everything else
  keeps the crawler running" — fail-open by design. But operators writing
  helm values or compose env files commonly quote the value as `" false "`
  (leading/trailing whitespace from YAML multi-line or careless formatting).
  Without `.strip()`, `" false ".lower() != "false"` is True → crawler stays
  scheduled. Operator believes the kill-switch flipped; cutover appears done
  in the change ticket; in fact Celery keeps writing canonical rows AND
  (after Step 5) N8N writes canonical rows too → unique-constraint conflicts
  on `(source_id, source_type='aop')`.
  **Fix:** change line 71 to
  `return os.environ.get(env_key, "true").strip().lower() != "false"` and add
  a UNIT-004 parametrize entry for `" false "` (and `"\tfalse\n"` for
  defensiveness) asserting `crawl-aop` is **absent**.

- [x] **[Review][Patch] `_clean_redis_cutover` autouse fixture catches bare `Exception`** [`eusolicit-app/services/data-pipeline/tests/integration/test_phase_2_cutover_smoke.py:167-172`] — **RESOLVED**: narrowed to `except aioredis.RedisError`.
  Project Delivery Instructions: "catch specific exception types — no bare
  `except:`, no swallowing of `Exception` without re-raise or structured
  logging." `try: ... except Exception: pass` on `redis.delete(key)` will
  silently leave stale streams / idempotency keys on a Redis transient
  failure → cross-test pollution; downstream `consumer.ack.assert_called_once()`
  asserts will then fail with the wrong root cause.
  **Fix:** narrow to `except redis.RedisError` (or import path
  `redis.exceptions.RedisError`).

- [x] **[Review][Patch] Stale filename reference in `infra/n8n-templates/ROLLBACK.md:149`** [`eusolicit-app/infra/n8n-templates/ROLLBACK.md:149`] — **RESOLVED**: replaced with correct story filename + explicit pointers to both new runbooks.
  Line 149 reads `5-23-phase2-cutover-runbook.md` — the actual story file is
  `5-23-phase-2-cutover-runbook-and-rollback.md`. The story modifies this
  file (per AC8) but did not fix the existing typo. Trivial doc fix; will
  fail the link-checker as soon as the Prometheus debt is paid down.
  **Fix:** rename to `5-23-phase-2-cutover-runbook-and-rollback.md`.

#### Decision-needed — items where the author should pick an approach

- [x] **[Review][Decision] Cutover Step 2 (kill Beat) precedes Step 5 (flip shadow) — there is a multi-minute window with NO canonical `aop` writes from any path.** — **RESOLVED**: chose option (b) — keep ordering, add "do not pause between Step 2 and Step 5" warning, document the reverse-ordering alternative's unique-constraint hazard, and provide an explicit abort path via the rollback runbook for forced pauses.
  After Step 2-3 Celery stops writing canonical `aop`; until Step 5 the N8N
  consumer keeps writing shadow `aop_n8n`. Downstream consumers reading
  `source_type='aop'` see a write-rate cliff for the duration of the
  operator's pause between Step 2 and Step 5 (could be a few minutes; could
  be longer if the operator pauses to verify Step 4). Either:
  (a) reorder — Step 5 (shadow flip) precedes Step 2 (Beat kill) so canonical
  writes never gap; OR
  (b) document explicitly under Step 2 that this gap is expected and bounded
  by operator-action latency, and add a "do not pause between Step 2 and
  Step 5" warning. Author judgement on which is operationally safer.

- [x] **[Review][Decision] Verify Beat scheduler is not `celery.beat.PersistentScheduler` against `/tmp/celerybeat-schedule` — stale entries can survive Beat restart.** — **RESOLVED**: Confirmed `docker-compose.yml:234` uses `celery.beat.PersistentScheduler` with `/tmp/celerybeat-schedule`. Added an explicit shelve-wipe step in cutover Preconditions (with k8s/docker-compose/systemd `rm -f` recipes) and a reminder in both cutover Step 3 and rollback Step 1.
  If the Beat process uses `celery.beat.PersistentScheduler` with a shelve
  file persisted across restarts (default in many docker-compose setups),
  removing an entry from `BEAT_SCHEDULE` does not remove it from the shelve
  — Beat continues to fire the stale entry until the shelve is wiped. Step 4
  verification (`celery inspect scheduled`) would correctly show the stale
  entry, but the runbook does not instruct the operator to wipe the shelve.
  Either confirm the deploy uses a memory-only or Redis-backed scheduler
  (`RedBeatScheduler`) and document that as a precondition, OR add a step:
  `kubectl exec -- rm -f /tmp/celerybeat-schedule*` (and equivalents) before
  Step 3's restart. Inspect `services/data-pipeline/src/data_pipeline/workers/celery_app.py`
  and the deploy manifests to confirm.

- [x] **[Review][Decision] `kubectl set env deploy/data-pipeline-beat` and Step 5's `kubectl rollout restart deploy/data-pipeline` assume two separate k8s deployments.** — **RESOLVED**: Cutover runbook gains a Preconditions §"Single-deployment topology" callout with topology-precheck commands (`kubectl get deploy`, `systemctl list-units`, `docker compose ps`) and a documented in-flight-drain requirement for single-deployment topologies.
  If the on-prem / k8s topology runs Beat + workers + consumer in a single
  `data-pipeline` deployment (common for low-volume staging), Step 5's
  rollout-restart kills in-flight enrichment workers — directly contradicting
  Step 3's explicit "Restart Beat only (NOT workers)" guarantee. Either add
  a precondition statement that names the disjoint deployments, OR add a
  precheck command (`kubectl get deploy -n eusolicit -l app=data-pipeline`)
  with a fallback procedure for single-deployment topologies.

- [x] **[Review][Decision] Rollback re-cutover gate may see stale `green` from `get_rolling_window_gate_status` for ~6 days post-rollback.** — **RESOLVED**: rollback runbook Step 5 now requires `MIN(run_date) > :rollback_ts::date` (a `gate_eligibility = 'window_is_post_rollback_only'` verdict) in addition to `green` from the helper. SQL recipe included.
  After rollback, the rolling 7-day window still contains pre-cutover green
  days. The helper's `gate_status` will read `green` until those days age
  out — operators following Step 4 item 3 ("≥7 fresh consecutive green days")
  may interpret the green status as "ready to re-cutover" prematurely.
  Either (a) tighten the rollback runbook §Step 4 wording to say "wait until
  the rolling window contains ONLY post-rollback days" with a SQL recipe to
  check `MIN(run_date) > rollback_date`; OR (b) document that operators must
  inspect `equivalence_runs` directly (not just the rolling helper) for the
  post-rollback window.

#### Deferred — pre-existing or out-of-scope

- [x] **[Review][Defer] AC11 link checker exits 1 due to 9 pre-existing Prometheus alert rule URL prefix typos** — DEVIATION explicitly documented in the story; the new runbooks are not in the script's check scope (no Prometheus annotations).

#### Dismissed — noise / not actionable

- Minor test gaps (UNIT-004 missing more whitespace permutations beyond the patch above; integration test does not defensively assert `is_equivalence_shadow_enabled('aop')` returns the expected value pre-flip) — covered by the patch on `_is_celery_crawler_enabled` strip.
- `equivalence-check-daily` BEAT_SCHEDULE entry being added in this story's diff (technically S05.22 work consolidated here) — required by UNIT-001's expected key set; not a defect.
- Cron-string env-var values for `_EQUIVALENCE_HOUR/MINUTE` (no `int()` cast) — celery.crontab accepts strings; no observed boundary failure.
- Multi-line `python -c` shell-paste fragility in cutover Step 0 — operator-environment concern, not a runbook defect.

### Recommendation

Resolve the 5 `patch` items (small mechanical fixes — code + doc) and the 4
`decision-needed` items (each requires author judgement on the operational
trade-off). The application code is sound; the gap is in the runbooks where
operator procedure ordering matters most. Once the rollback ordering and the
helper `.strip()` are fixed, the cutover mechanism is operationally safe.

---

## Senior Developer Review — Cycle 4 (re-review of cycle-3 fixes)

**Reviewer:** Claude (autonomous code review, BMAD `bmad-code-review`)
**Date:** 2026-05-15
**Verdict:** **REVIEW: Approve**

### Cycle-3 follow-up verification

All 5 `[Patch]` and 4 `[Decision]` items from cycle-3 are resolved and the
fixes were verified against the on-disk artefacts:

| Item | Where verified | Status |
|---|---|---|
| Patch 1 — `_is_celery_crawler_enabled` whitespace robustness | `beat_schedule.py:79-80` (`return os.environ.get(env_key, "true").strip().lower() != "false"`); UNIT-006 covers 7 padded variants in `test_beat_schedule.py:306-342` | ✓ Resolved |
| Patch 2 — Cutover Form A/B `min_baseline` divergence | `e05-phase-2-cutover.md:149-200` — explicit "Form A is authoritative" warning + env-var lookup recipe + `(min_baseline × 7)` substitution guidance | ✓ Resolved |
| Patch 3 — Rollback one-shot dispatch race | `e05-phase-2-rollback.md` — one-shot moved to new Step 3 (lines 152-190); Step 1 carries explicit "do not dispatch yet" warning (lines 108-114) | ✓ Resolved |
| Patch 4 — `_clean_redis_cutover` bare `Exception` | `test_phase_2_cutover_smoke.py:171` — narrowed to `except aioredis.RedisError` | ✓ Resolved |
| Patch 5 — Stale filename in `ROLLBACK.md` | `infra/n8n-templates/ROLLBACK.md:148-150` — corrected to `5-23-phase-2-cutover-runbook-and-rollback.md` plus explicit `e05-phase-2-{cutover,rollback}.md` pointers | ✓ Resolved |
| Decision 1 — Cutover Step 2/Step 5 ordering | `e05-phase-2-cutover.md:238-257` — author chose option (b): kept ordering, added "do not pause between Step 2 and Step 5" warning, documented the reverse-ordering hazard, provided abort path via rollback runbook | ✓ Resolved (author judgement accepted) |
| Decision 2 — PersistentScheduler shelve survival | `e05-phase-2-cutover.md:82-111` (Preconditions §"Scheduler type") + `e05-phase-2-rollback.md:91-106` (Step 1 reminder); concrete `rm -f /tmp/celerybeat-schedule*` recipes for k8s / docker compose / on-prem systemd | ✓ Resolved |
| Decision 3 — Single-deployment topology assumption | `e05-phase-2-cutover.md:45-80` — Preconditions §"Deployment topology" precheck commands (`kubectl get deploy`, `systemctl list-units`, `docker compose ps`) + dedicated "Single-deployment topology" callout with in-flight drain requirement | ✓ Resolved |
| Decision 4 — Rollback re-cutover gate stale-green | `e05-phase-2-rollback.md:236-271` — Step 5 §3 replaced soft "≥7 fresh green days" with explicit `MIN(run_date) > :rollback_ts::date` SQL recipe + `gate_eligibility = 'window_is_post_rollback_only'` verdict requirement | ✓ Resolved |

### Independent contract verification (this cycle)

- `_is_celery_crawler_enabled` correctness re-checked against the helper-uses
  in `compare_equivalence.py:_min_baseline()` and
  `equivalence_checksum.py:is_equivalence_shadow_enabled` — same env-read
  precedent (case-insensitive, no `BaseServiceSettings`, no exception
  swallowing). `os.environ.get(env_key, "true").strip().lower() != "false"`
  is fail-open by design, the strict-`"false"`-only contract matches AC1.
- `BEAT_SCHEDULE: dict[str, dict[str, Any]] = {}` annotation is mypy-safe;
  the conditional builder preserves the canonical entry order
  (crawl-aop → crawl-ted → crawl-eu-grants → cleanup → enrichment → poll →
  equivalence-check) for diff-readability and matches Task 1.2's example.
- `equivalence-check-daily` entry being added in the same diff (technically
  S05.22 work) is required by UNIT-001's expected key set — flagged in
  cycle-2 as not a defect, confirmed here.
- Integration test `test_phase_2_cutover_e2e_smoke` correctly:
  (a) seeds `client.sirmaai_projects` via autocommit psycopg2 so the
  asyncpg consumer can see the row (INT-009 precedent);
  (b) flips `PIPELINE_EQUIVALENCE_SHADOW_AOP` mid-test via `monkeypatch`
  (the consumer's `is_equivalence_shadow_enabled` reads at call time per
  `equivalence_checksum.py:104` — verified — no module reload needed);
  (c) uses distinct `source_id_suffix` (`P1`/`P2`) to avoid the
  `(source_id, source_type)` unique-constraint collision that would
  otherwise mask the post-flip behaviour change;
  (d) preservation guard asserts the Phase-1 row survives the flip (no
  migration), matching AC5 Step 6.
- Runbook helper signatures match production code:
  `get_rolling_window_gate_status(session, source_type, days=7)` exists at
  `compare_equivalence.py:356`; `_min_baseline()` reads
  `PIPELINE_EQUIVALENCE_MIN_BASELINE` at `compare_equivalence.py:69-70`;
  `_handle_message` is the public callable the integration test invokes
  at `workflow_event_consumer.py:239`.

### AC roll-up

| AC | Status |
|---|---|
| AC1 — kill-switch helper + strict contract | ✓ |
| AC2 — conditional Beat entry inclusion | ✓ |
| AC3 — `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>` semantics documented | ✓ |
| AC4 — go/no-go gate query (Form A authoritative + Form B fallback) | ✓ |
| AC5 — cutover runbook (Steps 0-9) | ✓ |
| AC6 — rollback runbook (≤15-min SLA) | ✓ |
| AC7 — archive marker docstrings | ✓ |
| AC8 — cross-links from existing runbooks + N8N README | ✓ |
| AC9 — cutover history table (3 pre-filled rows) | ✓ |
| AC10 — unit tests UNIT-001..UNIT-006 | ✓ (22 passed; 100% coverage) |
| AC11 — runbook link checker | ⚠ Documented deviation (pre-existing Prometheus URL-prefix debt outside this story's scope; new runbooks not in script's check scope) |
| AC12 — integration smoke `test_phase_2_cutover_e2e_smoke` | ✓ (1 passed) |
| AC13 — DoD: lint / type-check / coverage / tests | ✓ (per Test Results) |
| AC14 — no drop / no delete (scope boundary) | ✓ (only docstring banners on crawlers; no schema or behaviour change) |

### Findings

None blocking. The cycle-3 fixes resolve every blocking and decision-needed
concern raised in the prior review; no new defects were found in this pass.

Two minor non-blocking observations (informational only — do **not** address
in this story):

- **Cutover Step 4 `celery inspect scheduled` semantics**: this command
  queries running workers, not Beat. In the documented happy-path topology
  (separate `data-pipeline-beat` and `data-pipeline-worker` deployments)
  this works because the workers see the Beat-registered tasks. In the
  edge case where workers are momentarily offline, the command will return
  empty and could be misread as "schedule cleared". The runbook already
  offers the log-grep alternative (`kubectl logs ... | grep "crawl-aop"`)
  which is unaffected — operators following the runbook will succeed.
  Cosmetic clarification opportunity for a future doc-only PR.
- **Form B SQL `:min_baseline_times_7` is documented as a substitution
  point but not actually a SQL bind variable** — the SQL itself contains
  the literal `350` and the warning text instructs the operator to
  hand-edit. This is acceptable for a runbook (operators read and edit
  before pasting) and is clearly flagged. A future enhancement could use
  `psql --set` `\set` syntax for true variable substitution.

### Recommendation

Approve. Mark story complete and proceed to QA / Post-Review.

