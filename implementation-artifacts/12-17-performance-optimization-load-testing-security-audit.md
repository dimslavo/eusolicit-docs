# Story 12.17: Performance Optimization, Load Testing & Security Audit

Status: done

## Story

As a **platform operator and security lead**,
I want **all FastAPI services hardened with response compression and CORS controls, source-table query indexes added to accelerate materialized-view refreshes, Kubernetes network policies enabled for every service, a k6 load test suite passing p95 < 500 ms on staging, and a completed OWASP Top 10 checklist signed off before release**,
so that **EU Solicit ships as a performant, secure, production-ready platform with documented evidence of pre-launch technical hardening**.

## Acceptance Criteria

---

### AC1 — GZipMiddleware on All FastAPI Services

**Files:**
- `services/client-api/src/client_api/main.py`
- `services/enterprise-api/src/enterprise_api/main.py`
- `services/admin-api/src/admin_api/main.py`

Add `GZipMiddleware` **as the outermost middleware** on each app so that responses ≥ 1 000 bytes are compressed before egress. Registration must come **before** any auth or IP-allowlist middleware to avoid compressing responses before authentication short-circuits them.

**client-api/main.py** — add at the top of the middleware block, before SessionMiddleware:

```python
from starlette.middleware.gzip import GZipMiddleware

# Response compression — must be the outermost middleware (first to run on egress).
# minimum_size=1000 skips tiny JSON error responses and health-check payloads.
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

**enterprise-api/main.py** — add after `register_exception_handlers(app)` and before `_build_middlewares()`:

```python
from starlette.middleware.gzip import GZipMiddleware

# Compress responses ≥ 1 000 bytes (analytics JSON, OpenAPI spec).
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

**admin-api/main.py** — add after `register_exception_handlers(app)` and before `app.add_middleware(IPAllowlistMiddleware)`:

```python
from starlette.middleware.gzip import GZipMiddleware

# Compress responses ≥ 1 000 bytes (audit log exports, platform analytics JSON).
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

> **Dev note (middleware order in Starlette):** Starlette runs middleware in reverse-registration order (last registered = first to process an inbound request, but first to process an outbound response). `GZipMiddleware` registered first means it is the **outermost** wrapper — it sees the final response body and compresses it. Auth middleware registered after it processes the request first (correct) and passes an uncompressed response up through GZip (correct).

---

### AC2 — CORS Configuration on client-api

The client-api currently has no `CORSMiddleware`, leaving browser-initiated requests from the Next.js frontend and any third-party integrations without proper CORS headers. Add configurable CORS support.

**File: `services/client-api/src/client_api/config.py`**

Add the following field to `ClientApiSettings`:

```python
# CORS configuration (Story 12.17)
# Comma-separated list of allowed origins, e.g.:
#   https://eusolicit.com,https://www.eusolicit.com
# Set via CLIENT_API_CORS_ALLOWED_ORIGINS env var.
# An empty string or unset value defaults to the frontend_url only.
cors_allowed_origins: str = Field(
    default="",
    env="CORS_ALLOWED_ORIGINS",
    description=(
        "Comma-separated list of allowed CORS origins. "
        "Defaults to frontend_url when empty."
    ),
)

@property
def cors_origins_list(self) -> list[str]:
    """Return parsed list of allowed origins.

    Falls back to [frontend_url] when cors_allowed_origins is blank.
    """
    raw = self.cors_allowed_origins.strip()
    if not raw:
        return [self.frontend_url]
    return [o.strip() for o in raw.split(",") if o.strip()]
```

**File: `services/client-api/src/client_api/main.py`**

Add CORSMiddleware after GZipMiddleware registration and before SessionMiddleware:

```python
from starlette.middleware.cors import CORSMiddleware

# CORS — restrict browser cross-origin requests to the known frontend origin(s).
# Credentials (Authorization header) are allowed so the JWT Bearer flow works.
# Max-age of 600 s (10 min) reduces OPTIONS preflight traffic on analytics pages.
_settings = get_settings()
app.add_middleware(
    CORSMiddleware,
    allow_origins=_settings.cors_origins_list,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    max_age=600,
)
```

> **Dev note:** The enterprise-api is a programmatic API consumed via `X-API-Key` (not browser JWT). CORS is not required there. The admin-api is VPN-only with no browser clients from external origins; it does not need CORSMiddleware.

> **Environment variable:** In staging/production set `CLIENT_API_CORS_ALLOWED_ORIGINS=https://eusolicit.com,https://www.eusolicit.com`. In local dev the default `frontend_url=http://localhost:3000` is used automatically.

---

### AC3 — Ingress Rate Limiting Enhancements

The client-api nginx ingress currently has only `rate-limit-connections: "100"` (concurrent connection cap). Add an explicit **requests-per-second** rate limit per source IP so burst attacks are throttled at the ingress layer before reaching the application.

**File: `infra/helm/values/client-api.yaml`**

Replace the `ingress.annotations` block:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # Per-IP connection cap — limits concurrent connections from a single IP
    nginx.ingress.kubernetes.io/limit-connections: "50"
    # Per-IP request rate — limits new requests per second (burst of 5 × limit)
    nginx.ingress.kubernetes.io/limit-rps: "20"
    # Return 429 (not 503) when rate limit is exceeded
    nginx.ingress.kubernetes.io/limit-req-status-code: "429"
```

> **Dev note:** `limit-rps: "20"` translates to 20 req/s per IP with a burst of 5× (100 req/s short burst). Authenticated analytics clients hitting the API at normal usage rates (~1–2 req/s) are unaffected. Adjust `limit-rps` based on load test results if needed.

> **enterprise-api** already enforces per-API-key rate limiting via the Redis token bucket middleware (S12.15). Its ingress annotations do not require changes.

---

### AC4 — Kubernetes Network Policies: client-api, enterprise-api, notification

Three services currently have `networkPolicy.enabled: false`. Enable and configure ingress/egress rules for service-to-service isolation.

#### 4a — client-api (`infra/helm/values/client-api.yaml`)

Replace `networkPolicy` block:

```yaml
networkPolicy:
  enabled: true
  ingress:
    # Accept traffic from nginx-ingress-controller pods only
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8001
  egress:
    # DNS resolution (kube-system CoreDNS)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # PostgreSQL (primary + replica)
    - to: []
      ports:
        - protocol: TCP
          port: 5432
    # Redis
    - to: []
      ports:
        - protocol: TCP
          port: 6379
    # ai-gateway (agent calls: eligibility, budget-builder, etc.)
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: ai-gateway
      ports:
        - protocol: TCP
          port: 8004
    # External HTTPS (S3 signed URLs, SendGrid webhook, Google OAuth)
    - to: []
      ports:
        - protocol: TCP
          port: 443
```

#### 4b — enterprise-api (`infra/helm/values/enterprise-api.yaml`)

Replace `networkPolicy` block:

```yaml
networkPolicy:
  enabled: true
  ingress:
    # Accept traffic from nginx-ingress-controller pods only
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8003
  egress:
    # DNS resolution
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # PostgreSQL — API key lookup and rate limit key storage
    - to: []
      ports:
        - protocol: TCP
          port: 5432
    # Redis — token bucket rate limiting
    - to: []
      ports:
        - protocol: TCP
          port: 6379
    # Proxy to client-api
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: client-api
      ports:
        - protocol: TCP
          port: 8001
```

#### 4c — notification (`infra/helm/values/notification.yaml`)

Replace `networkPolicy` block:

```yaml
networkPolicy:
  enabled: true
  ingress: []   # notification is a Celery worker — no inbound traffic
  egress:
    # DNS resolution
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # PostgreSQL — materialized view refresh + report job writes
    - to: []
      ports:
        - protocol: TCP
          port: 5432
    # Redis — Celery broker + beat schedule
    - to: []
      ports:
        - protocol: TCP
          port: 6379
    # External HTTPS — S3 (report uploads), SendGrid (email delivery)
    - to: []
      ports:
        - protocol: TCP
          port: 443
```

> **Dev note:** The `to: []` egress blocks permit traffic to all IP ranges on those ports. This is intentional for managed services (AWS RDS, ElastiCache, S3) whose Cluster-external IPs are not pod-selectable. For further hardening post-MVP, scope these to specific CIDR blocks matching the AWS VPC NAT gateway.

---

### AC5 — Performance Index Migration (client-api migration 016)

Source tables created in migration 011 (`bid_decisions`, `bid_outcomes`, `bid_preparation_logs`, `competitor_records`, `shared.usage_meters`) have no indexes beyond their primary keys. MV refresh queries perform sequential scans against these tables on every Celery Beat tick. Add covering indexes on the columns used in each MV's SELECT query.

**New file: `services/client-api/alembic/versions/016_performance_indexes.py`**

```python
"""Add performance indexes on analytics source tables (Story 12.17).

These covering indexes reduce sequential scans during materialized view refresh
(REFRESH MATERIALIZED VIEW CONCURRENTLY) and during direct queries from the
pipeline service.

Tables targeted:
  client.bid_decisions          — feeds mv_market_intelligence
  client.bid_outcomes           — feeds mv_roi_tracker + mv_team_performance
  client.bid_preparation_logs   — feeds mv_roi_tracker + mv_team_performance
  client.competitor_records     — feeds mv_competitor_intelligence
  shared.usage_meters           — feeds mv_usage_consumption

Revision ID: 016
Revises:     015
Create Date: 2026-04-14
"""
from __future__ import annotations

from alembic import op

revision = "016"
down_revision = "015"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # -------------------------------------------------------------------------
    # client.bid_decisions — mv_market_intelligence refresh query filters by
    # company_id and groups by created_at month.
    # -------------------------------------------------------------------------
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_bid_decisions_company_created "
        "ON client.bid_decisions (company_id, created_at DESC)"
    )
    # opportunity_id FK lookup used in MV JOIN with pipeline.opportunities
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_bid_decisions_opportunity_id "
        "ON client.bid_decisions (opportunity_id)"
    )

    # -------------------------------------------------------------------------
    # client.bid_outcomes — mv_roi_tracker groups by company_id + won_at month;
    # mv_team_performance groups by company_id + status.
    # -------------------------------------------------------------------------
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_bid_outcomes_company_status "
        "ON client.bid_outcomes (company_id, status)"
    )
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_bid_outcomes_company_won_at "
        "ON client.bid_outcomes (company_id, won_at DESC NULLS LAST)"
    )

    # -------------------------------------------------------------------------
    # client.bid_preparation_logs — mv_roi_tracker sums cost_eur per proposal;
    # mv_team_performance aggregates hours per user_id per company.
    # proposal_id FK and user_id FK used in JOIN conditions.
    # -------------------------------------------------------------------------
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_bid_prep_logs_user_logged "
        "ON client.bid_preparation_logs (user_id, logged_at DESC)"
    )
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_bid_prep_logs_proposal_id "
        "ON client.bid_preparation_logs (proposal_id)"
    )

    # -------------------------------------------------------------------------
    # client.competitor_records — mv_competitor_intelligence groups by
    # (company_id, competitor_name, sector).
    # -------------------------------------------------------------------------
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_competitor_records_company_sector "
        "ON client.competitor_records (company_id, sector)"
    )

    # -------------------------------------------------------------------------
    # shared.usage_meters — mv_usage_consumption groups by
    # (company_id, meter_type, period_start).
    # -------------------------------------------------------------------------
    op.execute(
        "CREATE INDEX IF NOT EXISTS ix_usage_meters_company_type_period "
        "ON shared.usage_meters (company_id, meter_type, period_start DESC)"
    )


def downgrade() -> None:
    op.execute("DROP INDEX IF EXISTS client.ix_bid_decisions_company_created")
    op.execute("DROP INDEX IF EXISTS client.ix_bid_decisions_opportunity_id")
    op.execute("DROP INDEX IF EXISTS client.ix_bid_outcomes_company_status")
    op.execute("DROP INDEX IF EXISTS client.ix_bid_outcomes_company_won_at")
    op.execute("DROP INDEX IF EXISTS client.ix_bid_prep_logs_user_logged")
    op.execute("DROP INDEX IF EXISTS client.ix_bid_prep_logs_proposal_id")
    op.execute("DROP INDEX IF EXISTS client.ix_competitor_records_company_sector")
    op.execute("DROP INDEX IF EXISTS shared.ix_usage_meters_company_type_period")
```

> **Dev note:** All `CREATE INDEX` statements use `IF NOT EXISTS` so re-applying the migration is safe (e.g. in staging after a DB restore). The MV unique indexes (e.g. `uq_mv_market_intelligence`) were already created in migration 011 and are not repeated here.

> **EXPLAIN ANALYZE verification:** After applying this migration, run `EXPLAIN ANALYZE` on the MV refresh queries in staging with ≥ 100k rows. Confirm `Index Scan` (not `Seq Scan`) on all 7 new indexes. Document results in `eusolicit-docs/implementation-artifacts/load-test-results.md`.

---

### AC6 — k6 Load Test Script: Core User Flows

**New file: `tests/load/k6-perf-core-flows.js`**

```javascript
/**
 * k6 Load Test — Core User Flows (Story 12.17)
 *
 * Covers 4 key user flows against staging:
 *   1. Auth         — POST /auth/login  (rate-limited; 1 VU/scenario)
 *   2. Analytics    — GET  /analytics/market/volume, /roi/summary, /team/leaderboard, /usage
 *   3. Opportunity  — GET  /analytics/pipeline/forecast (Professional+ tier)
 *   4. Report       — POST /reports/generate (async job dispatch; no polling in load test)
 *
 * Pass criteria (Story 12.17 AC):
 *   - p(95) < 500 ms for all read endpoints
 *   - p(95) < 2 000 ms for write/dispatch endpoints (login, report generate)
 *   - Overall error rate < 1%
 *
 * Usage:
 *   k6 run tests/load/k6-perf-core-flows.js \
 *     --env BASE_URL=https://staging.eusolicit.com \
 *     --env AUTH_TOKEN=<seeded-jwt-starter> \
 *     --env PRO_TOKEN=<seeded-jwt-professional-plus>
 *
 * Results are printed to stdout and optionally exported to InfluxDB/Grafana
 * if K6_OUT env var is set.
 */
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// ---------------------------------------------------------------------------
// Thresholds & scenarios
// ---------------------------------------------------------------------------

export const options = {
  scenarios: {
    analytics_read: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 20 },   // ramp up
        { duration: '2m',  target: 50 },   // sustained load
        { duration: '30s', target: 0 },    // ramp down
      ],
      exec: 'analyticsFlow',
    },
    report_dispatch: {
      executor: 'constant-vus',
      vus: 5,
      duration: '2m',
      exec: 'reportDispatchFlow',
      startTime: '30s',  // start after analytics ramp completes
    },
  },
  thresholds: {
    // --- Read endpoints: p95 < 500 ms ---
    'http_req_duration{flow:analytics}':         ['p(95)<500'],
    'http_req_duration{flow:pipeline_forecast}': ['p(95)<500'],
    // --- Write/dispatch endpoints: p95 < 2 000 ms ---
    'http_req_duration{flow:report_dispatch}':   ['p(95)<2000'],
    // --- Overall ---
    'http_req_failed': ['rate<0.01'],
  },
};

// ---------------------------------------------------------------------------
// Environment variables
// ---------------------------------------------------------------------------

const BASE_URL   = __ENV.BASE_URL   || 'http://localhost:8001';
const TOKEN      = __ENV.AUTH_TOKEN  || '';
const PRO_TOKEN  = __ENV.PRO_TOKEN   || TOKEN;

const authHeaders = (tok) => ({
  headers: {
    Authorization: `Bearer ${tok}`,
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
});

// ---------------------------------------------------------------------------
// Scenario: analytics read flow
// ---------------------------------------------------------------------------

export function analyticsFlow() {
  group('market intelligence', () => {
    const r = http.get(
      `${BASE_URL}/api/v1/analytics/market/volume?page=1&page_size=20`,
      { ...authHeaders(TOKEN), tags: { flow: 'analytics' } },
    );
    check(r, {
      'market/volume 200': (res) => res.status === 200,
      'market/volume has items': (res) => {
        try { return JSON.parse(res.body).items !== undefined; } catch { return false; }
      },
    });
  });

  group('ROI summary', () => {
    const r = http.get(
      `${BASE_URL}/api/v1/analytics/roi/summary`,
      { ...authHeaders(TOKEN), tags: { flow: 'analytics' } },
    );
    check(r, { 'roi/summary 200': (res) => res.status === 200 });
  });

  group('team leaderboard', () => {
    const r = http.get(
      `${BASE_URL}/api/v1/analytics/team/leaderboard?page=1&page_size=10`,
      { ...authHeaders(TOKEN), tags: { flow: 'analytics' } },
    );
    check(r, { 'team/leaderboard 200': (res) => res.status === 200 });
  });

  group('usage dashboard', () => {
    const r = http.get(
      `${BASE_URL}/api/v1/analytics/usage`,
      { ...authHeaders(TOKEN), tags: { flow: 'analytics' } },
    );
    check(r, { 'analytics/usage 200': (res) => res.status === 200 });
  });

  group('pipeline forecast (Professional+)', () => {
    const r = http.get(
      `${BASE_URL}/api/v1/analytics/pipeline/forecast?page=1&page_size=20`,
      { ...authHeaders(PRO_TOKEN), tags: { flow: 'pipeline_forecast' } },
    );
    // 200 for Professional+ token, 403 for lower tiers — both are expected
    check(r, { 'pipeline/forecast non-500': (res) => res.status < 500 });
  });

  sleep(0.5);
}

// ---------------------------------------------------------------------------
// Scenario: report dispatch flow (async — does not wait for completion)
// ---------------------------------------------------------------------------

export function reportDispatchFlow() {
  group('on-demand report dispatch', () => {
    const payload = JSON.stringify({
      report_type: 'bid_performance_summary',
      format: 'pdf',
    });
    const r = http.post(
      `${BASE_URL}/api/v1/reports/generate`,
      payload,
      { ...authHeaders(TOKEN), tags: { flow: 'report_dispatch' } },
    );
    check(r, {
      'report/generate 202': (res) => res.status === 202,
      'report/generate has job_id': (res) => {
        try { return !!JSON.parse(res.body).job_id; } catch { return false; }
      },
    });
  });

  sleep(2);
}
```

> **Dev note — running the load test against staging:**
> ```bash
> k6 run tests/load/k6-perf-core-flows.js \
>   --env BASE_URL=https://staging.eusolicit.com \
>   --env AUTH_TOKEN="$(cat /tmp/staging-jwt-starter.txt)" \
>   --env PRO_TOKEN="$(cat /tmp/staging-jwt-professional-plus.txt)" \
>   --out influxdb=http://influxdb:8086/k6
> ```
> Generate seeded JWT tokens via the management command or by calling `POST /auth/login` with staging test credentials before running. Document all results (p50/p95/p99, RPS, error rate) in `eusolicit-docs/implementation-artifacts/load-test-results.md`.

---

### AC7 — Load Test Results Documentation Template

**New file: `eusolicit-docs/implementation-artifacts/load-test-results.md`**

```markdown
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
```

---

### AC8 — OWASP Top 10 Checklist & Secret Rotation Runbook

**New file: `eusolicit-docs/implementation-artifacts/security-audit-checklist.md`**

```markdown
# Security Audit Checklist — EU Solicit Pre-Launch (Story 12.17)

**Date:** _YYYY-MM-DD_
**Auditor:** _Name/Role_
**Status:** ☐ In Progress  ☐ Complete  ☐ Signed Off

---

## OWASP Top 10 (2021)

### A01 — Broken Access Control

| Check | Evidence | Status |
|:---|:---|:---:|
| All admin endpoints return 403 for non-VPN IPs (IPAllowlistMiddleware verified in staging) | `tests/e2e/admin/` P0 test run | ☐ |
| Standard user JWT cannot access other users' analytics data (`company_id` scoping on all MV queries) | Cross-tenant P0 API tests | ☐ |
| Tier gates return 403 + upgrade message on Professional+ gated endpoints | `tests/e2e/analytics/` P0 tier tests | ☐ |
| Enterprise API key revocation takes effect immediately (revoked key → 401) | `tests/e2e/enterprise/` P0 lifecycle test | ☐ |
| Company admin route guard blocks non-admin role from admin frontend pages | `tests/e2e/admin/` E2E P0 route guard test | ☐ |

### A02 — Cryptographic Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| All client-to-service traffic uses TLS 1.2+ (nginx `ssl_protocols TLSv1.2 TLSv1.3`) | `infra/nginx/eusolicit.com` reviewed | ☐ |
| JWT uses RS256 (not HS256); private key stored in Kubernetes Secret, not ConfigMap | `services/client-api/src/client_api/core/security.py` reviewed | ☐ |
| Enterprise API keys stored as SHA-256 hashes in DB; plaintext never persisted | `services/client-api/alembic/versions/015_enterprise_api_keys.py` + key creation code reviewed | ☐ |
| DB passwords use strong random strings (≥ 32 chars); no default passwords in production | Secret manager entry verified (see §Secret Rotation) | ☐ |
| S3 report bucket not publicly accessible; only signed URLs used | AWS S3 bucket policy verified | ☐ |

### A03 — Injection

| Check | Evidence | Status |
|:---|:---|:---:|
| All analytics queries use SQLAlchemy Core parameterised statements (no raw f-string SQL) | Code review: `services/client-api/src/client_api/services/analytics_*_service.py` | ☐ |
| Admin API queries use parameterised ORM (no raw SQL concatenation) | Code review: `services/admin-api/src/admin_api/services/` | ☐ |
| All Pydantic input models validate types and lengths (no unvalidated user strings in SQL filters) | Spot-check of request schemas in `schemas/` directories | ☐ |

### A04 — Insecure Design

| Check | Evidence | Status |
|:---|:---|:---:|
| Rate limiting applied at ingress layer (nginx `limit-rps: "20"`) and application layer (Redis token bucket for enterprise-api) | `infra/helm/values/client-api.yaml` + `enterprise-api/middleware/rate_limit.py` reviewed | ☐ |
| Login rate limiting (5 failures / 15 min) prevents brute force on auth endpoint | `services/client-api/src/client_api/core/rate_limit.py` reviewed | ☐ |
| Multi-tenant data isolation enforced at SQL layer (`WHERE company_id = :cid`) in all analytics services | All `analytics_*_service.py` files reviewed | ☐ |

### A05 — Security Misconfiguration

| Check | Evidence | Status |
|:---|:---|:---:|
| CORS origins restricted to known frontend origins (`cors_origins_list` from env var, not `*`) | `services/client-api/src/client_api/main.py` CORSMiddleware config | ☐ |
| Kubernetes network policies enabled for all 6 services (admin-api, ai-gateway, client-api, enterprise-api, notification, data-pipeline) | Helm values reviewed; `kubectl get networkpolicy -A` output attached | ☐ |
| No debug endpoints (`/debug`, `/admin/shell`) exposed in production | `main.py` files reviewed; ingress rules confirmed | ☐ |
| Swagger UI (`/v1/docs`) and Redoc (`/v1/redoc`) are enterprise-api only — not exposed on client-api or admin-api | `client_api/main.py` and `admin_api/main.py` — no `docs_url` set | ☐ |

### A06 — Vulnerable and Outdated Components

| Check | Evidence | Status |
|:---|:---|:---:|
| `pip-audit` run against all service `pyproject.toml`; no HIGH/CRITICAL CVEs | `pip-audit` output attached | ☐ |
| `npm audit` run against `frontend/package.json`; no HIGH/CRITICAL CVEs | `npm audit --json` output attached | ☐ |
| Base Docker images pinned to specific digest tags (not `:latest`) in production Dockerfiles | `Dockerfile` files reviewed | ☐ |

### A07 — Identification and Authentication Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| JWT expiry set to 15–60 minutes (access token); refresh token rotation enforced | `services/client-api/src/client_api/core/security.py` reviewed | ☐ |
| Refresh token revocation (logout) removes token from Redis blacklist | `services/client-api/src/client_api/api/v1/auth.py` logout endpoint reviewed | ☐ |
| OAuth CSRF state parameter validated in callback (Google OAuth flow) | `services/client-api/src/client_api/api/v1/auth.py` OAuth callback reviewed | ☐ |
| Admin JWT requires `role: admin` claim (not just a valid JWT) | `services/admin-api/src/admin_api/core/` admin auth dependency reviewed | ☐ |

### A08 — Software and Data Integrity Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| GitHub Actions CI pipeline enforces signed commits or branch protection rules | `.github/workflows/` CI config reviewed | ☐ |
| Celery task payloads are typed Pydantic models (no raw dict deserialization from untrusted sources) | `services/notification/src/notification/tasks/` reviewed | ☐ |

### A09 — Security Logging and Monitoring Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| Audit log middleware records all write operations (POST/PUT/PATCH/DELETE) with user_id, action, entity | `services/shared/audit_middleware.py` reviewed (Story 2.11) | ☐ |
| Admin tier-override actions create audit log entries with reason field | `services/admin-api/src/admin_api/api/v1/tenants.py` reviewed | ☐ |
| Failed auth attempts logged (login failure, invalid API key) | `services/client-api/src/client_api/api/v1/auth.py` + enterprise-api middleware reviewed | ☐ |

### A10 — Server-Side Request Forgery (SSRF)

| Check | Evidence | Status |
|:---|:---|:---:|
| No user-supplied URLs fetched server-side (report URL, webhook, redirect) without validation | All API endpoints reviewed; no user-controlled URL fetch patterns found | ☐ |
| AI gateway outbound requests are to fixed LLM provider endpoints (not user-controlled) | `services/ai-gateway/` config reviewed | ☐ |

---

## Network Policy Verification

Run the following in the staging cluster after deploying network policy changes (AC4):

```bash
# Verify all 6 services have network policies
kubectl get networkpolicy -n eusolicit

# Confirm client-api cannot reach admin-api directly (should timeout/refuse)
kubectl exec -n eusolicit deployment/client-api -- \
  curl -m 5 http://admin-api:8002/healthz

# Confirm enterprise-api can reach client-api proxy
kubectl exec -n eusolicit deployment/enterprise-api -- \
  curl -m 5 http://client-api:8001/healthz

# Confirm notification cannot be reached from outside the cluster
# (no ingress rules — all inbound traffic should be blocked)
```

| Test | Expected | Actual | Pass? |
|:---|:---|:---|:---:|
| `client-api → admin-api` direct call | Connection refused / timeout | | ☐ |
| `enterprise-api → client-api` proxy | 200 OK | | ☐ |
| `notification` inbound (no ingress) | Connection refused | | ☐ |

---

## Secret Rotation Runbook

All production secrets must be rotated before the release gate. Document the completion of each rotation below.

| Secret | Location | Rotated By | Date | New Value Stored In | Verified? |
|:---|:---|:---|:---|:---|:---:|
| `CLIENT_API_RSA_PRIVATE_KEY` (JWT signing key) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `CLIENT_API_RSA_PUBLIC_KEY` (JWT verification key) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `CLIENT_API_DATABASE_URL` (PostgreSQL password) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `ADMIN_API_DATABASE_URL` (PostgreSQL password) | K8s Secret `admin-api-secrets` | | | AWS Secrets Manager | ☐ |
| `ENTERPRISE_API_DATABASE_URL` (PostgreSQL password) | K8s Secret `enterprise-api-secrets` | | | AWS Secrets Manager | ☐ |
| `NOTIFICATION_DATABASE_URL` (PostgreSQL password) | K8s Secret `notification-secrets` | | | AWS Secrets Manager | ☐ |
| `INTERNAL_SERVICE_SECRET` (service-to-service auth) | K8s Secret (all services) | | | AWS Secrets Manager | ☐ |
| `CLIENT_API_OAUTH_SECRET_KEY` (session cookie signing) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `SENDGRID_API_KEY` | K8s Secret `notification-secrets` | | | AWS Secrets Manager | ☐ |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (S3) | K8s Secret (multiple services) | | | AWS IAM (role rotation) | ☐ |

> **Rotation procedure:**
> 1. Generate new secret value (use `openssl rand -base64 48` for symmetric keys; `openssl genrsa -out key.pem 4096` for RSA pairs).
> 2. Store in AWS Secrets Manager under the path `eusolicit/production/<secret-name>`.
> 3. Update the Kubernetes Secret via External Secrets Operator (or `kubectl create secret generic … --dry-run=client -o yaml | kubectl apply -f -`).
> 4. Perform a rolling restart: `kubectl rollout restart deployment/<service-name> -n eusolicit`.
> 5. Verify service health: `kubectl rollout status deployment/<service-name> -n eusolicit`.
> 6. Mark the row above as verified.

---

## Sign-Off

| Role | Name | Signature | Date |
|:---|:---|:---|:---|
| Security Lead | | | |
| Tech Lead | | | |
| Product Owner | | | |
```

---

## Dev Notes

### Risk Coverage

This story directly addresses the following Epic 12 risks from `test-design-epic-12.md`:

| Risk ID | Category | Description | AC Addressing It |
|:---|:---|:---|:---|
| **R12.3** | PERF | Analytics query latency > 3s p95 | AC5 (performance indexes), AC6 (load test) |
| **R12.9** | PERF | Load test p95 > 500ms at target concurrency | AC6 (k6 script), AC7 (results doc) |
| **R12.10** | SEC | OWASP failures, network policy gaps, unrotated secrets | AC2 (CORS), AC3 (rate limiting), AC4 (network policies), AC8 (checklist) |

### Test Expectations (from `test-design-epic-12.md`)

**P0 (must all pass before release):**
- Load test: p95 read latency < 500ms under target concurrency (50 VUs) on staging — AC6 k6 script enforces this as a threshold
- OWASP Top 10 checklist all items addressed or risk-accepted by Security Lead — AC8 checklist must be fully signed off

**P1 scenarios (S12.17 section):**
- `EXPLAIN ANALYZE` on all analytics queries shows no sequential scans on large datasets — verify via AC5 migration + AC7 results doc
- N+1 queries resolved — analytics endpoints use direct MV SELECT (1 query per request); verify via `EXPLAIN` output
- CORS configuration: allowed origins restricted; preflight returns correct headers — AC2; test via Playwright API test against non-whitelisted origin
- All public endpoints have rate limiting configured — AC3 nginx ingress + existing login rate limiter
- All production secrets documented as rotated in secret manager — AC8 rotation runbook
- Kubernetes network policies verify service-to-service isolation — AC4 + AC8 network verification section

**P2 scenario:**
- Response compression enabled: `Content-Encoding` header present on large responses — AC1 GZipMiddleware; verify with `curl -H "Accept-Encoding: gzip" /api/v1/analytics/market/volume -I` asserting `Content-Encoding: gzip`

### Affected Files Summary

| File | Change |
|:---|:---|
| `services/client-api/src/client_api/main.py` | Add GZipMiddleware + CORSMiddleware |
| `services/client-api/src/client_api/config.py` | Add `cors_allowed_origins` + `cors_origins_list` |
| `services/enterprise-api/src/enterprise_api/main.py` | Add GZipMiddleware |
| `services/admin-api/src/admin_api/main.py` | Add GZipMiddleware |
| `services/client-api/alembic/versions/016_performance_indexes.py` | New migration (source-table indexes) |
| `infra/helm/values/client-api.yaml` | Replace ingress annotations + enable networkPolicy |
| `infra/helm/values/enterprise-api.yaml` | Enable networkPolicy with ingress/egress rules |
| `infra/helm/values/notification.yaml` | Enable networkPolicy with egress-only rules |
| `tests/load/k6-perf-core-flows.js` | New k6 load test script |
| `eusolicit-docs/implementation-artifacts/load-test-results.md` | New results template |
| `eusolicit-docs/implementation-artifacts/security-audit-checklist.md` | New OWASP/secret rotation checklist |

### N+1 Query Note

The five analytics services (`analytics_market_service.py`, `analytics_roi_service.py`, `analytics_team_service.py`, `analytics_competitor_service.py`, `analytics_usage_service.py`) all use single SQLAlchemy Core SELECT queries directly against materialized views. There are no ORM relationship lazy-loads and no loops over DB results that issue additional queries. Each analytics endpoint makes **at most 2 DB round-trips**: one for the subscription/tier check (`require_paid_tier` dependency) and one for the MV SELECT. No N+1 remediation is required in the analytics layer.

If the QA team's query-count assertions detect unexpected query counts during E2E test runs against the full staging stack, investigate the `get_current_user` dependency (which issues 1 query to fetch the user + company) and the `require_paid_tier` dependency (1 subscription query). Both are intentional and necessary; they can be collapsed into a single query via a JOIN in a future optimization if latency targets are not met.

### Deferred Items (Not In Scope for S12.17)

- Full External Secrets Operator setup (ESO CRDs, SecretStore configuration) — the secret rotation runbook (AC8) documents the procedure and target state; ESO automation is a post-launch ops task
- `pip-audit` and `npm audit` CI gate integration — checklist items reference manual runs; automated gates are tracked in the CI pipeline story
- Chaos engineering (analytics service restart during MV refresh) — tracked in test-design-epic-12.md P3 scenarios; manual ops-assisted after launch
