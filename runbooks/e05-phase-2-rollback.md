# Runbook: E05 Phase-2 Rollback — Re-enable Celery Crawlers

**Severity**: SEV-2 (incident triggered cutover rollback)
**SLA-Scope**: in-scope; ≤ **15 minutes** from incident declared to legacy crawler resuming
**Last updated**: 2026-05-15
**Owner**: backend
**Story**: S05.23
**Related stories**: S05.20 (N8N templates), S05.21 (consumer), S05.22 (equivalence harness)

---

> **⚠ SLA: 15 minutes.** The steps that produce the SLA are:
> env var flip (seconds) + Beat restart (~30 s) + worker process up (~30 s) +
> optional one-shot crawl dispatch (skip cron wait — see Step 1).
>
> "Rollback complete" is defined as: env var flipped AND Beat restarted (legacy
> crawler re-enters the Beat schedule). The next successful crawl is **verification**
> (Step 3), not the SLA boundary.

---

## Symptoms

| Signal | Severity | Action |
|---|---|---|
| DLQ depth growing: `redis-cli XLEN sirmaai.workflow.completed.dlq` > 0 | SEV-2 | Investigate; rollback if DLQ is growing for the cut-over source |
| `pipeline_opportunities_total{source_type='aop'}` write rate < 50% of pre-cutover baseline | SEV-2 | Rollback AOP only |
| `pipeline_workflow_events_total{outcome='publish_failure'}` spike for cut-over source | SEV-2 | Rollback the specific source |
| On-call paged for `pipeline_equivalence_gate_status < 1` immediately after cutover | SEV-3 | Check if `insufficient_data` (expected after cutover) vs `red` (needs rollback) |
| Downstream consumers report missing opportunities for a source type | SEV-2 | Rollback the specific source |

**Critical: Rollback is per-source.** Do NOT set all three
`PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED=true` if only one source is regressing.
Re-enabling crawlers for healthy sources double-writes into both paths, inflating
the next equivalence-run delta and undoing a successful cutover.
See architecture amendment §5.2 ("staged rollback").

---

## Triage

1. **Identify the regressing source** from the symptom signal (Grafana label or log entry).
2. **Confirm this source is currently cut over** (its kill-switch is `false`):
   ```bash
   kubectl exec -n eusolicit deploy/data-pipeline-beat -- \
     celery -A data_pipeline.workers.celery_app inspect scheduled
   # Confirm 'crawl-<source>' is absent from the output
   ```
3. **Scope the rollback** to the specific source. Other sources remain cut over.
4. **Declare the incident** and link the change ticket from the cutover (Step 8 in
   `e05-phase-2-cutover.md`) as the change that introduced the regression.

---

## Resolution

### Step 1 — Re-enable Celery Beat for the regressing source

Set `PIPELINE_CELERY_CRAWL_<SOURCE>_ENABLED=true` and restart Beat.

> **Use the explicit value `true`** (lowercase). The kill-switch is strict
> `"false"`-only for disabling; any non-`"false"` value keeps the crawler running.
> Setting back to `true` is the documented safe state.

**Kubernetes:**
```bash
# Replace AOP/aop with TED/ted or EU_GRANTS/eu_grants as needed.
kubectl set env deploy/data-pipeline-beat \
  -n eusolicit \
  PIPELINE_CELERY_CRAWL_AOP_ENABLED=true
kubectl rollout restart deploy/data-pipeline-beat -n eusolicit
kubectl rollout status deploy/data-pipeline-beat -n eusolicit
```

**On-prem systemd:**
```bash
sudo systemctl edit data-pipeline-beat.service
# Update the Environment= line to: PIPELINE_CELERY_CRAWL_AOP_ENABLED=true
# (or remove the line entirely — the default is true / fail-open)
sudo systemctl daemon-reload
sudo systemctl restart data-pipeline-beat
sudo systemctl status data-pipeline-beat
```

**Docker Compose:**
```bash
# Update the environment value in docker-compose.yml or .env, then:
docker compose restart data-pipeline-beat
```

> **Wipe the PersistentScheduler shelve before restart** (production Beat
> uses `celery.beat.PersistentScheduler` per `docker-compose.yml:234`). The
> shelve survives the restart and would re-fire the stale schedule entries
> that were active at cutover time, but it also will not lose the re-enabled
> entry on rollback — wiping ensures Beat rebuilds the schedule cleanly from
> the current `BEAT_SCHEDULE` (which now includes `crawl-<source>` again).
>
> ```bash
> kubectl exec -n eusolicit deploy/data-pipeline-beat -- \
>   rm -f /tmp/celerybeat-schedule /tmp/celerybeat-schedule.db /tmp/celerybeat-schedule.bak /tmp/celerybeat-schedule.dat /tmp/celerybeat-schedule.dir
> # docker compose:
> docker compose exec data-pipeline-beat \
>   rm -f /tmp/celerybeat-schedule /tmp/celerybeat-schedule.db /tmp/celerybeat-schedule.bak /tmp/celerybeat-schedule.dat /tmp/celerybeat-schedule.dir
> # On-prem:
> sudo rm -f /tmp/celerybeat-schedule*
> ```

> **Do NOT dispatch the one-shot crawl yet.** The optional one-shot dispatch
> recipe is **Step 3 — after Step 2 has re-enabled shadow mode**. Running it
> here, while shadow mode is still off (canonical N8N writes ongoing), would
> race the Celery crawler into the same `(source_id, source_type='aop')` rows
> as the N8N consumer, producing unique-constraint conflicts in
> `pipeline.opportunities` that masquerade as a deeper bug under SEV-2 and
> can trigger panic-escalation. Cycle-3 review fix.

### Step 2 — Re-enable equivalence shadow mode for the source

Set `PIPELINE_EQUIVALENCE_SHADOW_<SOURCE>=true` and restart the data-pipeline
consumer process.

After this point, SirmaAI N8N events for the source land under the **shadow**
`source_type` (`aop_n8n` / `ted_n8n` / `eu_grants_n8n`) again, and Celery
crawler writes resume under the canonical `source_type` (`aop` / `ted` / `eu_grants`).

The `equivalence-check-daily` task (S05.22) will pick up the diff again on its
next 03:00 UTC run and start producing fresh gate signals.

**Kubernetes:**
```bash
# Replace AOP/aop with TED/ted or EU_GRANTS/eu_grants as needed.
kubectl set env deploy/data-pipeline \
  -n eusolicit \
  PIPELINE_EQUIVALENCE_SHADOW_AOP=true
kubectl rollout restart deploy/data-pipeline -n eusolicit
kubectl rollout status deploy/data-pipeline -n eusolicit
```

**On-prem systemd:**
```bash
sudo systemctl edit data-pipeline.service
# Update: Environment=PIPELINE_EQUIVALENCE_SHADOW_AOP=true
sudo systemctl daemon-reload
sudo systemctl restart data-pipeline
```

**Docker Compose:**
```bash
# Update docker-compose.yml or .env, then:
docker compose restart data-pipeline
```

### Step 3 — (Optional) Skip cron latency with a one-shot dispatch

> **Only run this AFTER Step 2 has completed and the data-pipeline consumer has
> restarted with shadow mode re-enabled.** Verify via `kubectl get pods -n
> eusolicit -l app=data-pipeline` (or systemctl/`docker compose ps`) that the
> consumer pod is `Running`/`active` before continuing. Dispatching the
> one-shot crawl before Step 2 completes will race the Celery crawler into
> canonical `aop` rows while the N8N consumer is also still writing canonical
> `aop` (shadow off), producing unique-constraint conflicts. Cycle-3 review fix.

After Beat restarts, the next crawl fires at the next scheduled interval (worst
case: 6 h for AOP under the default `PIPELINE_AOP_CRAWL_EVERY_HOURS=6`). To
avoid waiting during incident response, dispatch a one-shot crawl immediately:

```bash
# AOP:
kubectl exec -n eusolicit deploy/data-pipeline -- \
  python -c "
from data_pipeline.workers.tasks.crawl_aop import crawl_aop
result = crawl_aop.delay()
print(result.id)
"

# TED:
kubectl exec -n eusolicit deploy/data-pipeline -- \
  python -c "
from data_pipeline.workers.tasks.crawl_ted import crawl_ted
result = crawl_ted.delay()
print(result.id)
"

# EU Grants:
kubectl exec -n eusolicit deploy/data-pipeline -- \
  python -c "
from data_pipeline.workers.tasks.crawl_eu_grants import crawl_eu_grants
result = crawl_eu_grants.delay()
print(result.id)
"
```

### Step 4 — Verify rollback is in effect

Within 1 hour of the one-shot dispatch (Step 3) or next cron tick:

```sql
-- Celery crawler resumed — new row in crawler_runs
SELECT crawler_type, status, started_at, ended_at, found, new, updated
FROM pipeline.crawler_runs
WHERE crawler_type = 'aop'   -- replace as needed
ORDER BY started_at DESC
LIMIT 3;
-- Expected: a new row with status='completed' and started_at > rollback time

-- Shadow writes resumed — new opportunity rows under shadow source_type
SELECT count(*) FROM pipeline.opportunities
WHERE source_type = 'aop_n8n'   -- replace as needed
  AND created_at > now() - interval '1 hour';
-- Expected: non-zero (N8N events land under shadow again)

-- Canonical writes NOT from N8N (only from Celery now)
SELECT count(*) FROM pipeline.opportunities
WHERE source_type = 'aop'   -- replace as needed
  AND created_at > now() - interval '1 hour';
-- Expected: grows only from resumed Celery crawler writes
```

Also confirm Beat schedule is restored:
```bash
kubectl exec -n eusolicit deploy/data-pipeline-beat -- \
  celery -A data_pipeline.workers.celery_app inspect scheduled
# Expected: 'crawl-<source>' IS listed again
```

### Step 5 — Incident handoff

1. **Link the rollback incident ticket** to the original cutover change ticket
   (Step 8 in `e05-phase-2-cutover.md`). Update `soak_verdict` in the cutover
   history table to `rolled-back` with a link to the rollback incident.

2. **Next investigation step**: use `n8n-equivalence-investigation.md` to
   determine the root cause of the regression — was it a consumer bug, an N8N
   template issue, a DLQ backlog, or data quality?

3. **Gate for re-cutover**: After rollback, the **rolling 7-day window will
   still contain pre-cutover green days** for ≤ 6 days. The
   `get_rolling_window_gate_status` helper will therefore continue to read
   `green` until those pre-cutover days age out. **Do NOT use the helper's
   `green` reading directly as the re-cutover gate during this 6-day overlap.**

   Cycle-3 review fix: explicit "post-rollback only" gate. The re-cutover gate
   requires ALL of:
   - ≥ 7 **fresh** consecutive green days (post-rollback window only — the
     rolling window must contain ZERO pre-cutover days)
   - `total_baseline >= 350` in that **post-rollback-only** window
   - Root cause of the regression identified and resolved

   Verify the rolling window contains only post-rollback days with this query
   (`:rollback_ts` is the timestamp when Step 2 completed):

   ```sql
   -- Check the earliest day in the current rolling window
   SELECT
       source_type,
       MIN(run_date) AS earliest_run_date_in_window,
       MAX(run_date) AS latest_run_date_in_window,
       COUNT(*) AS days_in_window,
       :rollback_ts::date AS rollback_date,
       CASE
           WHEN MIN(run_date) > :rollback_ts::date THEN 'window_is_post_rollback_only'
           ELSE 'window_still_contains_pre_rollback_days'
       END AS gate_eligibility
   FROM pipeline.equivalence_runs
   WHERE source_type = 'aop'   -- replace as needed
     AND run_date > CURRENT_DATE - 7;
   ```

   Only proceed to re-cutover when `gate_eligibility = 'window_is_post_rollback_only'`
   AND the helper returns `green`.

   Do NOT attempt re-cutover until all three conditions are met.

4. **Per-source scope reminder**: only the rolled-back source needs re-investigation.
   Sources that are still cut over and healthy remain in Phase-2.

---

## Verification

After rollback:

1. `crawl-<source>` appears in `celery inspect scheduled` output.
2. New `pipeline.crawler_runs` row with `status='completed'` within 1 h.
3. `pipeline.opportunities` shows new rows under `source_type='<source>_n8n'`
   (shadow restored) within 1 h.
4. DLQ depth returns to 0: `redis-cli XLEN sirmaai.workflow.completed.dlq`.
5. No continued error spikes in `data_pipeline` service logs.

---

## Related

- `e05-phase-2-cutover.md` — the cutover procedure this rollback reverses
- `n8n-equivalence-investigation.md` — root-cause investigation for equivalence failures
- `deploy-rollback.md` — general service deploy rollback
- S05.22 implementation: `eusolicit-docs/implementation-artifacts/5-22-phase-1-equivalence-test-harness.md`
- S05.23 implementation: `eusolicit-docs/implementation-artifacts/5-23-phase-2-cutover-runbook-and-rollback.md`
- Architecture amendment §5.2 (staged rollback): `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md`
