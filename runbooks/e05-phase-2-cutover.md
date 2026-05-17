# Runbook: E05 Phase-2 Cutover — Retire Celery Crawlers

**Severity**: SEV-3 (planned operation; not an incident)
**SLA-Scope**: in-scope; ≤ 24 h per source soak before next source
**Last updated**: 2026-05-15
**Owner**: backend
**Story**: S05.23
**Related stories**: S05.20 (N8N templates), S05.21 (consumer), S05.22 (equivalence harness), S05.30 (legacy code deletion — follow-up)

---

## Symptoms

This is a **planned operational procedure**, not an incident response. The trigger
to execute this runbook is:

- The S05.22 `equivalence-check-daily` Beat task has run for **≥ 7 consecutive days**
  for a given source with daily `gate_status='green'`.
- The AC4 rolling helper (see §Resolution Step 0) confirms `gate_status='green'`,
  `days_observed >= 7`, `total_baseline >= 350`.
- S05.20 N8N templates and S05.21 consumer are healthy (no DLQ entries in
  `sirmaai.workflow.completed.dlq`, no recent errors in `data_pipeline` logs).

If none of these conditions are met, do NOT execute this runbook. See
`n8n-equivalence-investigation.md` for investigation guidance.

---

## Triage

**Recommended cutover sequence** (one source at a time, never in parallel):

1. **AOP first** — highest volume; most signal during the 24 h soak.
2. **EU Grants second** — lowest volume; least disruptive if rollback needed.
3. **TED last** — middle volume; runs only after both AOP and EU Grants have soaked.

Each source has its own 24 h soak window before starting the next source.

**Per-source independence**: a rollback on one source does NOT affect the others.
If AOP regresses after cutover, roll back only AOP. TED and EU Grants that are
already cut over remain cut over.

---

## Preconditions

Before starting, verify the deployment topology and scheduler type. The
runbook commands below assume the docker-compose / production layout where
`data-pipeline-beat`, `data-pipeline-worker`, and `data-pipeline` (consumer)
are **separate** deployments / services. If a deployment runs all three in a
single pod / service, Step 5's restart command will kill in-flight workers
and contradict Step 3's "restart Beat only" guarantee.

```bash
# Confirm separate Beat / consumer deployments exist (kubernetes)
kubectl get deploy -n eusolicit -l app=data-pipeline
# Expect to see at least: data-pipeline (consumer), data-pipeline-beat
# If only one deployment is returned, see the "Combined-deployment topology"
# note in §Single-deployment topology below before executing.

# On-prem / systemd
systemctl list-units --type=service | grep data-pipeline
# Expect at least: data-pipeline.service, data-pipeline-beat.service

# Docker compose
docker compose ps data-pipeline data-pipeline-beat
```

### Single-deployment topology (combined Beat+worker+consumer)

If the deploy runs everything in a single pod / process (occasionally seen
in low-volume staging or in dev-stack docker-compose if `data-pipeline-beat`
is folded into `data-pipeline`), Step 3 and Step 5 below collapse into a
single restart that also restarts in-flight workers. Add an explicit gate:
**drain in-flight enrichment / scoring tasks before Step 3**. The
recommended path is to bring this deploy in line with the documented
3-service split (see `docker-compose.yml` services `data-pipeline-beat`,
`data-pipeline-worker`, `data-pipeline`) before running the cutover. If
that is not possible, document the in-flight drain time in the change
ticket so operators know the cutover window includes worker drain.

### Scheduler type — PersistentScheduler shelve must NOT survive the restart

The production / docker-compose Beat process runs
`celery.beat.PersistentScheduler` with `/tmp/celerybeat-schedule` (confirmed
in `docker-compose.yml:234`). The shelve file persists schedule state
across restarts. **Removing an entry from `BEAT_SCHEDULE` does NOT
automatically remove it from the shelve** — Beat will continue to fire the
stale entry on its existing cadence until the shelve is wiped.

Before Step 3's Beat restart, wipe the shelve so Beat rebuilds from
`BEAT_SCHEDULE` rather than from the stale shelve:

```bash
# Kubernetes:
kubectl exec -n eusolicit deploy/data-pipeline-beat -- \
  rm -f /tmp/celerybeat-schedule /tmp/celerybeat-schedule.db /tmp/celerybeat-schedule.bak /tmp/celerybeat-schedule.dat /tmp/celerybeat-schedule.dir

# Docker Compose:
docker compose exec data-pipeline-beat \
  rm -f /tmp/celerybeat-schedule /tmp/celerybeat-schedule.db /tmp/celerybeat-schedule.bak /tmp/celerybeat-schedule.dat /tmp/celerybeat-schedule.dir

# On-prem (path may differ — check the service's --schedule flag):
sudo rm -f /tmp/celerybeat-schedule*
```

(If the deploy switched to `RedBeatScheduler` or in-memory scheduler, the
shelve wipe is a no-op and safe to skip — but harmless to run anyway.)

Step 3 below restarts Beat; the shelve is reconstructed from
`BEAT_SCHEDULE` on next startup with the cut-over entry omitted.

## Resolution

### Step 0 — Go/no-go gate query (run before each per-source cutover)

Confirm the rolling 7-day gate is green for the source under cutover. Two forms
(use whichever tooling is available):

**Form A: Python one-liner (preferred — uses the production helper)**

```bash
# Replace 'aop' with 'ted' or 'eu_grants' for other sources.
kubectl exec -n eusolicit deploy/data-pipeline -- \
  python -c "
from data_pipeline.db import get_sync_session
from data_pipeline.workers.tasks.compare_equivalence import get_rolling_window_gate_status
import json
with get_sync_session() as s:
    print(json.dumps(get_rolling_window_gate_status(s, 'aop'), indent=2))
"
```

Expected output (must show ALL three conditions):
```json
{
  "gate_status": "green",
  "days_observed": 7,
  "total_baseline": 420,
  "rolling_delta_pct": 0.0
}
```

Abort if `gate_status` is `"red"` → investigate with `n8n-equivalence-investigation.md`.
Abort if `gate_status` is `"insufficient_data"` → wait for more data.

**Form B: SQL equivalent (psql-only operators)**

> **⚠ Form A is authoritative.** Form A reads
> `PIPELINE_EQUIVALENCE_MIN_BASELINE` at runtime via `_min_baseline()` in
> `compare_equivalence.py:69-70`. Form B below hard-codes the default
> `min_baseline = 50` (× 7 days = 350) for `total_baseline` threshold. If
> the deploy overrides `PIPELINE_EQUIVALENCE_MIN_BASELINE` (e.g. lower for
> staging, higher for prod), Form B will diverge from Form A and may
> greenlight a cutover Form A would block (or vice versa).
>
> Cycle-3 review fix: Form B now requires the operator to substitute the
> effective `:min_baseline` value (a `:min_baseline_times_7` bind reference
> is included for transparency). To find the deployed value:
> ```bash
> kubectl exec -n eusolicit deploy/data-pipeline -- \
>   printenv PIPELINE_EQUIVALENCE_MIN_BASELINE 2>/dev/null || echo "default=50"
> ```
> Then paste `:min_baseline_times_7` = `(min_baseline × 7)` into the SQL
> below. If the env var is unset, the default `350` applies and Form B
> agrees with Form A.

```sql
-- Replace 'aop' with 'ted' or 'eu_grants' for other sources.
-- Replace 350 with (PIPELINE_EQUIVALENCE_MIN_BASELINE × 7) if overridden.
-- Authoritative source: Form A. This SQL is for psql-only operators who cannot
-- exec into the data-pipeline pod.
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
        WHEN SUM(total_celery) < 350 THEN 'insufficient_data'  -- PIPELINE_EQUIVALENCE_MIN_BASELINE × 7 (default 50 × 7)
        WHEN (SUM(drift_count + missing_in_shadow + missing_in_celery)::numeric
              / GREATEST(SUM(total_celery), 1) * 100) < 0.1 THEN 'green'
        ELSE 'red'
    END AS gate_status
FROM pipeline.equivalence_runs
WHERE source_type = 'aop'
  AND run_date > CURRENT_DATE - 7;  -- strict > matches exactly 7 dates
```

Gate criteria for proceed:
- `gate_status = 'green'`
- `days_observed >= 7`
- `total_baseline >= (PIPELINE_EQUIVALENCE_MIN_BASELINE × 7)` — default `350`
  (50 rows/day × 7 days). If the deploy overrides
  `PIPELINE_EQUIVALENCE_MIN_BASELINE`, prefer Form A or paste the effective
  value into Form B.

### Step 1 — Snapshot baseline state

Before any change, capture the pre-cutover state. Paste into the incident / change ticket
as the rollback reference baseline.

```bash
# Row counts by source_type (both canonical and shadow)
psql "$DATABASE_URL" -c "
SELECT source_type, COUNT(*) AS row_count
FROM pipeline.opportunities
WHERE source_type IN ('aop', 'aop_n8n', 'ted', 'ted_n8n', 'eu_grants', 'eu_grants_n8n')
GROUP BY source_type
ORDER BY source_type;
"

# Last successful crawler_runs entry for this source
psql "$DATABASE_URL" -c "
SELECT crawler_type, status, started_at, ended_at, found, new, updated
FROM pipeline.crawler_runs
WHERE crawler_type = 'aop'   -- replace as needed
ORDER BY started_at DESC
LIMIT 3;
"

# Latest equivalence_runs entry
psql "$DATABASE_URL" -c "
SELECT run_date, source_type, total_celery, total_shadow, delta_pct, gate_status
FROM pipeline.equivalence_runs
WHERE source_type = 'aop'   -- replace as needed
ORDER BY run_date DESC
LIMIT 3;
"
```

### Step 2 — Flip the kill-switch (disable Celery Beat for the source)

> **⚠ Do not pause between Step 2 and Step 5.** Between Step 2-3 (Beat kill)
> and Step 5 (shadow flip), the N8N consumer is still writing under the
> shadow `source_type` (`aop_n8n`) while Celery has stopped writing canonical
> `aop`. Downstream consumers reading `source_type='aop'` will see a write-rate
> cliff for the duration of the gap. Cycle-3 review fix: this gap is expected
> and acceptable for a few minutes, but operators must NOT pause for a meal
> break, escalation review, or paged-out triage between Step 2 and Step 5.
>
> If you must pause (incident escalation), abort the cutover via the rollback
> runbook (`e05-phase-2-rollback.md`) — re-enable the kill-switch, restart
> Beat, and resume from Step 0 later. The cutover is reversible specifically
> to allow this kind of decision.
>
> An alternative order (Step 5 before Step 2) creates a different transient:
> both Celery (canonical `aop`) and N8N (now also canonical `aop` after Step 5)
> would write into the same `(source_id, source_type='aop')`, producing
> unique-constraint conflicts in `pipeline.opportunities`. That transient is
> worse than the write-rate cliff because it surfaces as ERROR-logged
> failures during the operator's pause window. We therefore keep Step 2
> before Step 5 and rely on the "do not pause" discipline above.

Set `PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED=false` in the deployment environment.

> **Operator must write the explicit lowercase value `false`.** The kill-switch
> uses strict `"false"` matching (case-insensitive). Any other value keeps the crawler
> running (fail-open by design — S05.23 AC1).

**Kubernetes:**
```bash
# Replace AOP/aop with TED/ted or EU_GRANTS/eu_grants as needed.
kubectl set env deploy/data-pipeline-beat \
  -n eusolicit \
  PIPELINE_CELERY_CRAWL_AOP_ENABLED=false
```

**On-prem systemd:**
```bash
sudo systemctl edit data-pipeline-beat.service
# In the override editor, add under [Service]:
#   Environment=PIPELINE_CELERY_CRAWL_AOP_ENABLED=false
# Save and close.
```

**Docker Compose (dev/staging):**
Edit `docker-compose.yml` (or the `.env` file referenced by compose) and add:
```yaml
services:
  data-pipeline-beat:
    environment:
      PIPELINE_CELERY_CRAWL_AOP_ENABLED: "false"
```

Then reload the env (see Step 3).

### Step 3 — Restart Beat only (NOT workers)

> **IMPORTANT**: Restart only the Beat process. Do NOT restart the Celery workers.
> In-flight scoring / guide-generation tasks from S05.07 / S05.08 belong to either
> path and must drain normally. Only Beat picks up the new schedule at startup.

> **Wipe the PersistentScheduler shelve first** (see Preconditions §"Scheduler
> type"). If the shelve is not wiped, Beat will rebuild from the stale shelve
> and the cut-over entry will continue to fire on its existing cadence — the
> operator believes the kill-switch flipped but Celery keeps running.

**Kubernetes:**
```bash
kubectl rollout restart deploy/data-pipeline-beat -n eusolicit
kubectl rollout status deploy/data-pipeline-beat -n eusolicit
```

**On-prem systemd:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart data-pipeline-beat
sudo systemctl status data-pipeline-beat
```

**Docker Compose:**
```bash
docker compose restart data-pipeline-beat
docker compose logs --tail=20 data-pipeline-beat
```

### Step 4 — Verify Beat schedule no longer includes the source

After Beat restarts, confirm the entry is absent from the registered schedule.

```bash
# Kubernetes:
kubectl exec -n eusolicit deploy/data-pipeline-beat -- \
  celery -A data_pipeline.workers.celery_app inspect scheduled

# Or check the Beat startup banner in logs:
kubectl logs -n eusolicit deploy/data-pipeline-beat --tail=50 | grep "crawl-aop"
```

Expected: `crawl-aop` (or `crawl-ted` / `crawl-eu-grants`) is **NOT listed**.

If the entry is still present: the env var was not picked up. Re-check Step 2
and restart Beat again (Step 3).

**Local smoke verification:**
```bash
# From within the Beat container (or virtualenv with data_pipeline installed):
PIPELINE_CELERY_CRAWL_AOP_ENABLED=false python -c \
  "from data_pipeline.workers.beat_schedule import BEAT_SCHEDULE; print(list(BEAT_SCHEDULE.keys()))"
# Output must NOT include 'crawl-aop'
```

### Step 5 — Flip the equivalence shadow flag

Set `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>=false` and restart the data-pipeline
consumer process (not Beat — the consumer is the S05.21 workflow event handler).

After this point, new SirmaAI `workflow.completed` events for this source will
land in `pipeline.opportunities` under the **canonical** `source_type` (`aop`,
`ted`, or `eu_grants`) rather than the shadow discriminator (`aop_n8n`, `ted_n8n`,
`eu_grants_n8n`). This is the **whole purpose** of the env-flag flip.

> **Context**: During Phase-1, the deploy manifest opted in to shadow mode via
> `PIPELINE_EQUIVALENCE_SHADOW_AOP=true` (and TED / EU_GRANTS). The cutover
> **deletes** those `=true` lines (or sets them `=false`). Both produce the same
> Phase-2 state — the S05.22 `is_equivalence_shadow_enabled` helper defaults
> `false` (fail-closed for enabling shadow; S05.22 R8).

**Kubernetes:**
```bash
# Replace AOP/aop with TED/ted or EU_GRANTS/eu_grants as needed.
kubectl set env deploy/data-pipeline \
  -n eusolicit \
  PIPELINE_EQUIVALENCE_SHADOW_AOP=false
kubectl rollout restart deploy/data-pipeline -n eusolicit
kubectl rollout status deploy/data-pipeline -n eusolicit
```

**On-prem systemd:**
```bash
sudo systemctl edit data-pipeline.service
# Add: Environment=PIPELINE_EQUIVALENCE_SHADOW_AOP=false
sudo systemctl daemon-reload
sudo systemctl restart data-pipeline
```

**Docker Compose:**
```bash
# Update docker-compose.yml or .env, then:
docker compose restart data-pipeline
```

### Step 6 — Confirm write under canonical source_type

Within **1 hour** of the next N8N cron firing (see template cadence in
`infra/n8n-templates/README.md`), run:

```sql
-- Canonical write (new rows since cutover — must be non-zero within 1 hour of next cron)
SELECT count(*) FROM pipeline.opportunities
WHERE source_type = 'aop'    -- replace as needed
  AND created_at > now() - interval '1 hour';

-- Shadow write (must return zero new rows for this source going forward)
SELECT count(*) FROM pipeline.opportunities
WHERE source_type = 'aop_n8n'    -- replace as needed
  AND created_at > now() - interval '1 hour';
```

Expected:
- Canonical query (`'aop'`): **non-zero** (new rows being written under canonical source_type)
- Shadow query (`'aop_n8n'`): **zero** (no new rows; pre-cutover rows preserved but no new writes)

> **Note**: Pre-cutover rows under both `source_type` values are preserved.
> The acceptance criterion is about the **write path going forward**, not migration
> of historical rows.

### Step 7 — Soak for 24 hours

Monitor the following Grafana panels for 24 hours:

- `pipeline_workflow_events_total{source_type, outcome}` — event consumption rate
- `pipeline_opportunities_total{source_type, action}` — opportunity write rate
- DLQ depth: `redis-cli XLEN sirmaai.workflow.completed.dlq` (should be 0)
- Error rate in `data_pipeline` service logs

**Abort / rollback triggers** during soak:
- DLQ depth grows above 0 → trigger rollback runbook `e05-phase-2-rollback.md`
- `pipeline_opportunities_total{source_type='aop'}` write rate < 50% of
  pre-cutover baseline → investigate or trigger rollback
- Any `publish_failure` spike for this source → trigger rollback runbook

If any panel shows regression, trigger `e05-phase-2-rollback.md` immediately.

### Step 8 — Mark the source cut over

After the 24 h soak is clean, fill in the Cutover history table at the bottom
of this file. Record:

- `source`: e.g. `aop`
- `cutover_date`: ISO date (e.g. `2026-05-16`)
- `operator`: your name / handle
- `final_delta_pct`: from the go/no-go gate query (Step 0)
- `soak_verdict`: `clean` or `issues-resolved` (with a note)
- `incident_ticket`: link to the change/incident ticket

### Step 9 — Repeat for next source

Do NOT skip the 24 h soak between sources.

**Recommended sequence (AC5 Step 9):**
1. **AOP** → 24 h soak → mark complete
2. **EU Grants** → 24 h soak → mark complete
3. **TED** → 24 h soak → mark complete (final Phase-2 completion)

After all three sources are cut over and soaked, the E05 SirmaAI re-platform
Phase-2 is complete. Legacy crawler modules (`crawl_aop.py`, `crawl_ted.py`,
`crawl_eu_grants.py`) are scheduled for deletion in S05.30.

---

## Verification

After each source cutover and soak:

1. `celery -A data_pipeline.workers.celery_app inspect scheduled` — confirm
   the crawl entry for the cut-over source is absent.
2. SQL canonical write check (Step 6 above) returns non-zero canonical rows.
3. SQL shadow write check (Step 6 above) returns zero new shadow rows.
4. `equivalence-check-daily` (S05.22) produces `gate_status='insufficient_data'`
   for the cut-over source — this is correct and expected (no Celery baseline).
5. No DLQ growth, no error spikes over the 24 h soak.

---

## Rollback

If any verification step fails or the 24 h soak shows a regression, trigger
the per-source rollback immediately.

See `e05-phase-2-rollback.md` — **SLA ≤ 15 minutes** from incident declared to
legacy crawler resuming writes.

**Rollback is per-source**: do NOT roll back AOP because TED is regressing. Each
source has its own kill-switch and its own rollback path.

---

## Related

- `e05-phase-2-rollback.md` — rollback procedure (≤15min SLA)
- `n8n-equivalence-investigation.md` — how to read the equivalence gate and act on red/insufficient_data
- `deploy-rollback.md` — general service deploy rollback
- S05.22 implementation: `eusolicit-docs/implementation-artifacts/5-22-phase-1-equivalence-test-harness.md`
- S05.23 implementation: `eusolicit-docs/implementation-artifacts/5-23-phase-2-cutover-runbook-and-rollback.md`
- S05.30 (future): Legacy Celery crawler deletion story
- Architecture amendment: `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §5.3

---

## Cutover history

Fill in each row at Step 8 after the source's 24 h soak completes.

| source | cutover_date | operator | final_delta_pct | soak_verdict | incident_ticket |
|---|---|---|---|---|---|
| aop | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ |
| eu_grants | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ |
| ted | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ | _(pending)_ |
