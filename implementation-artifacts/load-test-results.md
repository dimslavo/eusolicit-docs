# Load Test Results — EU Solicit Staging (Story 12.17)

**Date:** _YYYY-MM-DD_
**k6 script:** `tests/load/k6-perf-core-flows.js`
**Staging URL:** `https://staging.eusolicit.com`
**Executor:** _Name/Role_

## Test Configuration

| Parameter | Value |
|:---|:---|
| Analytics scenario VUs | 0 → 50 (ramping) |
| Analytics scenario duration | 3 min total (30s ramp + 2m sustained + 30s ramp-down) |
| Report dispatch VUs | 5 (constant) |
| Report dispatch duration | 2 min |
| Staging DB seeded rows | ≥ 500k per analytics source table |

## Results Summary

### Analytics Read Flow

| Metric | p50 | p95 | p99 | Target p95 | Pass? |
|:---|:---:|:---:|:---:|:---:|:---:|
| `GET /analytics/market/volume` | — ms | — ms | — ms | < 500 ms | ☐ |
| `GET /analytics/roi/summary`   | — ms | — ms | — ms | < 500 ms | ☐ |
| `GET /analytics/team/leaderboard` | — ms | — ms | — ms | < 500 ms | ☐ |
| `GET /analytics/usage`         | — ms | — ms | — ms | < 500 ms | ☐ |
| `GET /analytics/pipeline/forecast` | — ms | — ms | — ms | < 500 ms | ☐ |

### Report Dispatch Flow

| Metric | p50 | p95 | p99 | Target p95 | Pass? |
|:---|:---:|:---:|:---:|:---:|:---:|
| `POST /reports/generate`       | — ms | — ms | — ms | < 2 000 ms | ☐ |

### Overall

| Metric | Value | Target | Pass? |
|:---|:---:|:---:|:---:|
| Total requests | — | — | — |
| Throughput (RPS) | — | — | — |
| Error rate | —% | < 1% | ☐ |
| HTTP 5xx count | — | 0 | ☐ |

## Bottlenecks Found & Resolutions

_Document any p95 > 500 ms queries here, with EXPLAIN ANALYZE output and the fix applied._

| Endpoint | Issue | Fix Applied | Result After Fix |
|:---|:---|:---|:---|
| | | | |

## EXPLAIN ANALYZE Results (MV Source Tables)

After applying migration 016, run the following on staging with ≥ 100k rows in each source table and paste the query plan output here:

```sql
-- bid_decisions covering index verification
EXPLAIN ANALYZE
SELECT company_id, DATE_TRUNC('month', created_at) AS month, COUNT(*)
FROM client.bid_decisions
WHERE company_id = '<test-uuid>'
GROUP BY company_id, month;
```

| Index | Query Plan (Seq Scan / Index Scan) | Execution Time | Pass? |
|:---|:---|:---:|:---:|
| `ix_bid_decisions_company_created` | — | — ms | ☐ |
| `ix_bid_outcomes_company_status` | — | — ms | ☐ |
| `ix_bid_prep_logs_user_logged` | — | — ms | ☐ |
| `ix_competitor_records_company_sector` | — | — ms | ☐ |
| `ix_usage_meters_company_type_period` | — | — ms | ☐ |
