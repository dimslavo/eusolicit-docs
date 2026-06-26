# Runbook: N8N Equivalence Harness Investigation

**Last updated**: 2026-05-14
**Owner**: backend
**Related stories**: S05.22 (harness), S05.20 (N8N templates), S05.21 (consumer), S05.23 (cutover)

---

## Purpose

This runbook explains how to read and act on the daily equivalence diff results in
`pipeline.equivalence_runs` produced by the S05.22 Phase-1 harness.

The harness runs daily at **03:00 UTC** and compares the legacy Celery crawl output
(`source_type ∈ {aop, ted, eu_grants}`) against the N8N/AgenticSAI shadow output
(`source_type ∈ {aop_n8n, ted_n8n, eu_grants_n8n}`) for the rolling 7-day window.

The Phase-2 kill-switch in S05.23 can only be flipped when the rolling gate is **green**
(`rolling_delta_pct < 0.1%`, `days_observed >= 7`, `total_baseline >= 350 rows`).

---

## a. Reading `equivalence_runs`

### Latest result per source

```sql
SELECT
    run_date,
    source_type,
    total_celery,
    total_shadow,
    matched,
    drift_count,
    missing_in_shadow,
    missing_in_celery,
    delta_pct,
    gate_status,
    created_at
FROM pipeline.equivalence_runs
ORDER BY run_date DESC, source_type
LIMIT 9;  -- 3 sources × 3 days
```

### Rolling 7-day aggregate per source

```sql
SELECT
    source_type,
    COUNT(*)                                                AS days_observed,
    SUM(total_celery)                                       AS total_baseline,
    SUM(drift_count + missing_in_shadow + missing_in_celery) AS total_delta,
    ROUND(
        SUM(drift_count + missing_in_shadow + missing_in_celery)::numeric
        / GREATEST(SUM(total_celery), 1) * 100, 4
    )                                                       AS rolling_delta_pct
FROM pipeline.equivalence_runs
WHERE run_date > CURRENT_DATE - 7   -- strict `>`: matches exactly 7 distinct
                                    -- dates (today + 6 prior).  This matches
                                    -- the production helper
                                    -- ``get_rolling_window_gate_status`` so
                                    -- Grafana panels and runbook queries agree
                                    -- on ``total_baseline`` (review cycle-2 R9).
GROUP BY source_type
ORDER BY source_type;
```

### Drill-down: inspect delta samples for a specific date

```sql
SELECT
    run_date,
    source_type,
    delta_pct,
    gate_status,
    jsonb_pretty(delta_samples) AS samples
FROM pipeline.equivalence_runs
WHERE run_date = '2026-05-14'
  AND source_type = 'aop';
```

---

## b. Interpreting `gate_status`

| Value | Meaning | Action |
|---|---|---|
| `green` | `delta_pct < 0.1%` AND `total_celery >= 50` | Accumulate 7 consecutive days; then see S05.23 cutover runbook |
| `red` | `delta_pct >= 0.1%` AND enough data | Investigate immediately — see §c–e below |
| `insufficient_data` | `total_celery < 50` (not enough Celery rows in window) | Check that Celery crawlers are running: `pipeline.crawler_runs` |

The **Phase-2 gate** requires all three sources to show `green` on the **rolling** 7-day
query (§a) — not just a single day.

---

## c. Drilling into a drift sample

Each `delta_samples` entry has the shape:

```json
{
  "source_id": "AOP-12345",
  "drift_kind": "content_drift",
  "fields_changed": ["budget_min", "deadline"]
}
```

To compare the two rows side-by-side:

```sql
SELECT
    source_type,
    title,
    description,
    deadline,
    budget_min,
    budget_max,
    cpv_codes,
    evaluation_criteria,
    mandatory_documents,
    published_at,
    status
FROM pipeline.opportunities
WHERE source_id = 'AOP-12345'
  AND source_type IN ('aop', 'aop_n8n')
ORDER BY source_type;
```

Compare the values for `fields_changed` entries to understand whether the drift
is a normalisation difference (e.g. one path strips trailing whitespace) or a
genuine semantic discrepancy.

---

## d. Investigating `missing_in_shadow`

`missing_in_shadow` means the opportunity exists in the Celery path but has no
matching `source_id` in the N8N shadow path.

**Step 1 — Did the N8N workflow run for this window?**

```sql
SELECT
    w.eusolicit_run_id,
    w.status,
    w.created_at,
    w.completed_at
FROM gateway.workflow_runs w
WHERE w.created_at BETWEEN '<window_start>' AND '<window_end>'
  AND w.metadata->>'source_type' = 'aop'  -- or ted / eu_grants
ORDER BY w.created_at DESC
LIMIT 20;
```

**Step 2 — Was the webhook DLQ'd?**

```bash
# Check the DLQ stream in Redis (DB 0)
redis-cli XLEN agenticsai.workflow.completed.dlq
redis-cli XRANGE agenticsai.workflow.completed.dlq - + COUNT 5
```

Or query the DLQ table if a persistent DLQ was configured:

```sql
SELECT * FROM gateway.webhook_dlq
WHERE created_at BETWEEN '<window_start>' AND '<window_end>'
ORDER BY created_at DESC
LIMIT 20;
```

**Step 3 — Is the N8N template running on schedule?**

In the AgenticSAI console (`https://agenticsai.endigitalx.com/`), check the workflow run
history for the relevant template (AOP / TED / EU-Grants). Compare timestamps against
`pipeline.crawler_runs` for the same window.

---

## e. Investigating `missing_in_celery`

`missing_in_celery` means the opportunity exists in the N8N shadow path but has no
matching `source_id` in the Celery path. This is often a **positive signal** (N8N
found something Celery missed) — but it still counts against the delta.

**Step 1 — Did the Celery crawler run for this window?**

```sql
SELECT
    crawler_type,
    status,
    started_at,
    ended_at,
    found,
    new,
    updated,
    errors
FROM pipeline.crawler_runs
WHERE crawler_type = 'aop'  -- or ted / eu_grants
  AND started_at >= '<window_start>'
ORDER BY started_at DESC
LIMIT 10;
```

**Step 2 — Is the source_id available in the upstream portal?**

Look up the `source_id` directly on the source portal (AOP / TED / EU Grants portal).
If the portal record exists and the Celery crawler should have found it, file a bug
against the Celery normaliser. If the record was published after the Celery window but
before the N8N window, this is a timing artefact — not a normaliser bug.

---

## f. Documented exclusions (known false-positive drifts)

*(No exclusions as of 2026-05-14 — the harness is fresh.)*

**Protocol for adding an exclusion:**

If a field is consistently appearing in `fields_changed` due to a non-deterministic
normalisation difference (e.g. one normaliser trims trailing whitespace from `description`
and the other does not), the process is:

1. Open an architecture amendment citing the specific field and root cause.
2. Get architect sign-off (Slack `#architecture` channel, tag Winston).
3. Add the field to `KNOWN_DRIFT_EXCLUSIONS` in `equivalence_checksum.py` in a new story.
4. Ensure the exclusion is documented here with the date and justification.

**Do NOT** silently extend the exclusion list without the above process — undocumented
exclusions inflate the green signal and can mask real quality issues.

---

## g. Escalation

| Condition | Action |
|---|---|
| `delta_pct > 1%` sustained for > 24 h on any source | Page **Deb** (`@dkslavo`) via Slack `#on-call` |
| Rolling gate still `red` after 3 days of investigation | Escalate to **architect** (`#architecture`); may need S05.23 delay |
| `gate_status = 'insufficient_data'` for > 48 h | Check Celery Beat is running: `celery -A data_pipeline.workers.celery_app inspect scheduled` |
| `equivalence.source_failed` in Celery logs | Check `pipeline_equivalence_gate_status` Grafana panel; restart the Beat worker if DB connectivity is the cause |
| Rolling gate `green` for ≥ 7 days on a source | Ready to cut over — see `e05-phase-2-cutover.md` |
| Cutover regression detected (post-flip) | Rollback per `e05-phase-2-rollback.md` (≤15min SLA) |

**Grafana panels** (alert threshold: `pipeline_equivalence_gate_status < 1`):

- `pipeline_equivalence_delta_pct{source_type="aop"}` — daily delta percentage
- `pipeline_equivalence_gate_status{source_type="aop"}` — 1=green, 0=red, -1=insufficient_data
- `pipeline_equivalence_drift_total{source_type="aop", drift_kind="content_drift"}` — cumulative drift counter

Replace `aop` with `ted` or `eu_grants` for the other sources.
