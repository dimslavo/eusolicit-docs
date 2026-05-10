# PE.02 PostgreSQL HA Migration — Cutover Runbook

> **⚠ SUPERSEDED 2026-05-11 — historical reference only.**
> EU Solicit pivoted away from AWS RDS Multi-AZ to single-host on-premise Docker on www1.endigitalx.com.
> This runbook documents the planned AWS migration that was never executed.
> See: `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md` for the pivot decision
> and `eusolicit-docs/planning-artifacts/architecture.md` §ADR-010 (rewritten 2026-05-11) for the new direction.
> The active replacement story is `onprem-01-postgres-backup-and-recovery` (sprint-status `development_status`).
> Kept here for design-reasoning continuity; the Method A/B / decision-tree / validation-step patterns may be useful for any future DB migration.

**Story:** 21-2-postgresql-ha-migration-managed-rds-multi-az-or-equivalent
**Epic:** E21 Platform Reliability for 99.9% SLA
**Author:** Story 21-2 Dev Agent (2026-05-04)
**Status:** Ready for staging rehearsal

> **Purpose:** This runbook is the canonical cutover guide for migrating
> the EU Solicit PostgreSQL instance from docker-compose / single-instance RDS
> to managed Multi-AZ RDS PostgreSQL 16.4 in eu-central-1. It is the primary
> input for the PE.06 runbook authoring story (PG failover runbook headline).
>
> The staging rehearsal MUST be completed and signed-off before the production
> cutover is attempted. Never skip the rehearsal. (Anti-pattern guard #6.)

---

## §Pre-flight Checklist

Complete **all** items before beginning any cutover step:

- [ ] Terraform plan reviewed and approved (`terraform plan -var-file=environments/staging/terraform.tfvars` or prod)
- [ ] **LIVE Terraform plan executed against the target AWS account** (NOT the documented dry-run output in §Terraform Plan Evidence — that is the pre-execution preview only). Capture the live `terraform plan` output and confirm: (i) exactly one `aws_db_instance.main` create; (ii) one `aws_db_subnet_group.main` create; (iii) one `aws_db_parameter_group.main` create; (iv) one `aws_security_group.db` create; (v) seven `random_password.service[*]` creates; (vi) seven `aws_secretsmanager_secret.db_service[*]` + `_version` creates; (vii) one `null_resource.db_bootstrap` create; (viii) one `random_id.final_snapshot_suffix`. **No** destroys of stable infra. Story 21-2 review-fix L2.
- [ ] **Helm `externalsecret.yaml` smoke-test rendered** for at least one service: `helm template eusolicit-service -f infra/helm/values/client-api.yaml --show-only templates/externalsecret.yaml` MUST emit a valid `ExternalSecret` with `secretKey: CLIENT_API_DATABASE_URL` (review-fix L3 / B2 verification). The same command for each of the other 5 services should render the matching `<SERVICE>_DATABASE_URL` key. Save the rendered output alongside the cutover ticket as evidence.
- [ ] Staging dry-run signed-off (§Staging Rehearsal Timing populated)
- [ ] Change ticket filed in incident management system (24h advance notice minimum per project-context customer-comm pattern)
- [ ] On-call engineer paged and available for the full maintenance window + 60 min post-cutover
- [ ] CTO notified (executive sponsor of the 99.9% SLA)
- [ ] Backups verified: latest automated backup <24h old (`aws rds describe-db-instances --query 'DBInstances[0].LatestRestorableTime'`)
- [ ] Terraform state lock acquired (`terraform plan` shows no in-progress changes)
- [ ] ESO ClusterSecretStore "aws-secrets-manager" verified reachable (`kubectl get clustersecretstore aws-secrets-manager -n external-secrets`)
- [ ] All 6 service `/health` endpoints returning 200 on source instance
- [ ] `staging-seed-perf-baseline.py` re-run to seed ≥10K opportunities for post-cutover EXPLAIN ANALYZE validation
- [ ] Rollback decision tree reviewed by on-call engineer (§Rollback Plan)

---

## §Cutover Method

**Selected method: Method A — Logical Replication** (preferred; ≤5-minute RTO)

**Rationale:** Logical replication allows zero-downtime near-cutover: the write pause is only
the final drain window (typically <2 min) rather than the full dump/restore time. At the
expected database size (<10 GB for EU Solicit at launch), the initial sync completes in
<15 minutes on a gp3 instance. Method B (pg_dump + restore) is documented as fallback
but should only be used if the new RDS instance cannot accept a replication subscription
(e.g., source DB was not provisioned with `wal_level = logical`).

**Fallback trigger:** If the logical replication subscription fails to reach lag=0 within 30
minutes of creation, abort and switch to Method B (the 30-minute maintenance window is
the fallback budget).

---

## §Method A Steps (Logical Replication — ≤5-min write pause)

### Phase 1 — Pre-cutover setup (can run while source DB is live)

1. **Provision new RDS Multi-AZ instance** via Terraform:
   ```bash
   cd eusolicit-app/infra/terraform
   terraform plan -var-file=environments/staging/terraform.tfvars
   terraform apply -var-file=environments/staging/terraform.tfvars
   ```
   Record the new instance endpoint: `<new-rds-endpoint>`.

2. **Bootstrap roles on new instance** (Terraform `null_resource.db_bootstrap` runs this automatically,
   but can be run manually if needed):
   ```bash
   PGPASSWORD=$(aws secretsmanager get-secret-value --secret-id eusolicit/staging/db/master \
     --query SecretString --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])") \
   psql -h <new-rds-endpoint> -U eusolicit_admin -d postgres \
     -f infra/postgres/init/01-init-schemas-and-roles.sql
   ```
   Verify: `psql -c "\du"` shows all 7 roles (`client_api_role`, `admin_api_role`, `data_pipeline_role`,
   `ai_gateway_role`, `notification_role`, `integrations_api_role`, `migration_role`).

3. **Run all Alembic migrations on new instance**:
   ```bash
   DATABASE_URL=postgresql+psycopg2://migration_role:<pwd>@<new-rds-endpoint>:5432/eusolicit \
     make migrate-all
   ```
   Verify: `make migrate-status` shows head for all services including `data-pipeline: 003 (head)`.

4. **Create publication on source instance** (requires superuser or replication privilege):
   ```sql
   CREATE PUBLICATION pe02_cutover FOR ALL TABLES IN SCHEMA
     client, pipeline, admin, gateway, notification, integrations, shared;
   ```

5. **Create subscription on new instance** (connects to source):
   ```sql
   CREATE SUBSCRIPTION pe02_cutover
     CONNECTION 'host=<source-endpoint> dbname=eusolicit user=migration_role password=<pwd>'
     PUBLICATION pe02_cutover;
   ```

6. **Monitor replication lag** until it reaches <1s:
   ```sql
   -- Run on source instance:
   SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
     AS lag_bytes
   FROM pg_replication_slots;
   ```
   Target: lag_bytes < 1 MB before proceeding to the write-pause window.

### Phase 2 — Write-pause window (≤5 minutes)

7. **Activate maintenance page** on ingress (Nginx/Kubernetes):
   ```bash
   kubectl annotate ingress eusolicit-api nginx.ingress.kubernetes.io/custom-http-errors="503" \
     nginx.ingress.kubernetes.io/default-backend="maintenance-page" -n eusolicit
   ```

8. **Drain in-flight transactions**: wait 30 seconds for any long-running transactions to complete.
   Verify no active connections: `psql -c "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';"` = 0.

9. **Verify replication lag = 0** on source:
   ```sql
   SELECT * FROM pg_replication_slots WHERE slot_name = 'pe02_cutover';
   -- Expect: active=true, pg_size_pretty(lag) = '0 bytes'
   ```

10. **Drop subscription on new instance** (stops replication):
    ```sql
    DROP SUBSCRIPTION pe02_cutover;
    ```

11. **Update ESO secrets to point to new endpoint**:
    ```bash
    # Update each Secrets Manager secret's "url" field to new RDS endpoint
    for service in client-api admin-api data-pipeline ai-gateway notification integrations-api migration; do
      aws secretsmanager get-secret-value --secret-id eusolicit/staging/db/${service} \
        --query SecretString --output text | \
        python3 -c "import sys,json; d=json.load(sys.stdin); d['host']='<new-rds-endpoint>'; \
          d['url']=d['url'].replace('<old-endpoint>','<new-rds-endpoint>'); print(json.dumps(d))" | \
        aws secretsmanager put-secret-value --secret-id eusolicit/staging/db/${service} \
          --secret-string file:///dev/stdin
    done
    ```

12. **Restart all 6 service deployments** (forces ESO refresh + new DB connections):
    ```bash
    kubectl rollout restart deploy -l tier=backend -n eusolicit
    kubectl rollout status deploy -l tier=backend -n eusolicit --timeout=120s
    ```

13. **Verify `/health` endpoints** return 200 for all 6 services:
    ```bash
    for port in 8001 8002 8003 8004 8005 8007; do
      curl -sf http://localhost:${port}/health && echo "port ${port}: OK" || echo "port ${port}: FAIL"
    done
    ```

14. **Lift maintenance page**:
    ```bash
    kubectl annotate ingress eusolicit-api nginx.ingress.kubernetes.io/custom-http-errors- \
      nginx.ingress.kubernetes.io/default-backend- -n eusolicit
    ```

---

## §Method B Steps (pg_dump + restore — ≤30-min maintenance window)

> Use only if Method A fails (logical replication cannot reach lag=0 within 30 min).
> Anti-pattern guard #7: NEVER use `pg_dump -F p` (plain text). Use `-Fc` for parallel restore.

1. Schedule 30-minute maintenance window (24h advance customer notice).
2. Activate maintenance page (same as Method A step 7).
3. Final dump from source:
   ```bash
   pg_dump -h <source-endpoint> -U migration_role -Fc -f /tmp/cutover-$(date +%Y%m%d-%H%M).dump eusolicit
   ```
4. Restore to new instance (parallel with `-j 4`):
   ```bash
   pg_restore -h <new-rds-endpoint> -U migration_role -d eusolicit -j 4 /tmp/cutover-*.dump
   ```
5. Verify row counts match per schema:
   ```sql
   SELECT schemaname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
   ```
   Compare output between source and target — every table must match.
6. Switch ESO secrets and restart deployments (same as Method A steps 11-13).
7. Verify `/health` endpoints.
8. Lift maintenance page.

---

## §Validation Steps

Run these immediately after the maintenance page is lifted:

```bash
# 0. Helm template smoke test (Review Follow-up L3)
# Verifies the ExternalSecret renders the service-prefixed DATABASE_URL key.
# Run BEFORE the deployment is rolled out so a misrendered template is caught
# at apply-time, not at pod-startup time.
for svc in client-api admin-api data-pipeline ai-gateway notification integrations-api; do
  echo "=== $svc ==="
  helm template eusolicit-service infra/helm/eusolicit-service \
    -f infra/helm/values/${svc}.yaml \
    --show-only templates/externalsecret.yaml \
  | grep -E "secretKey:|key: eusolicit/" || {
    echo "FAIL: ExternalSecret did not render for $svc"
    exit 1
  }
done
# Each service must produce its `<SERVICE>_DATABASE_URL` key (B2) plus the
# bare DATABASE_URL key for the migration_role Alembic CLI job.

# 1. Integration test subset (schema-isolation + FTS regression)
make test-integration

# 2. FTS regression — verify Bitmap Index Scan (the AC-2.4 closure test)
# Run against new instance with ≥10K opportunities seeded
docker exec <new-postgres> psql -U migration_role -d eusolicit -c "
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, ts_rank(tsv, websearch_to_tsquery('english', 'consultancy')) AS rank
  FROM pipeline.opportunities
 WHERE deleted_at IS NULL AND tsv @@ websearch_to_tsquery('english', 'consultancy')
 ORDER BY rank DESC LIMIT 20;"
# Required: top operator MUST be 'Bitmap Heap Scan' / 'Bitmap Index Scan on ix_opportunities_tsv'

# 3. All-services health check
for svc in client-api admin-api data-pipeline ai-gateway notification integrations-api; do
  curl -sf http://<cluster-endpoint>/health -H "X-Service: ${svc}" | jq '.status'
done

# 4. Smoke test per service (one read + one write endpoint each)
# Adjust endpoints per environment:
# client-api: GET /api/v1/opportunities (read) + POST /api/v1/proposals (write)
# admin-api: GET /admin/health (read)
# data-pipeline: GET /health (read)
# ai-gateway: GET /health (read)
# notification: GET /health (read)
# integrations-api: GET /health (read)
```

---

## §Rollback Plan

> Anti-pattern guard #4: NEVER skip this DECISION TREE. The on-call engineer at 02:00 UTC
> will not figure it out in the moment; the runbook is what prevents a 4h SEV-1 from
> extending to 12h.

```
DECISION TREE — use the first applicable branch:

IF cutover fails BEFORE ESO secret switch (steps 1-10):
  → ROLLBACK-A: Discard new instance
  → Action: Delete subscription on new instance (if created); keep source running
  → No data loss risk: source never stopped accepting writes
  → Time budget: immediate (source is still primary)

IF cutover fails AFTER ESO secret switch but services are failing (steps 11-14):
  → ROLLBACK-B: Revert ESO secrets to old endpoint, restart deployments
  → Action:
      for service in client-api admin-api data-pipeline ai-gateway notification integrations-api; do
        aws secretsmanager put-secret-value \
          --secret-id eusolicit/staging/db/${service} \
          --secret-string '{"url": "postgresql+asyncpg://<role>@<old-endpoint>:5432/eusolicit", ...}'
      done
      kubectl rollout restart deploy -l tier=backend -n eusolicit
  → Time budget: ≤10 min to detect, ≤20 min to execute
  → Note: any writes to new instance between ESO switch and rollback are NOT replicated back.
    Multi-AZ is synchronous ON the new instance; but the old instance was frozen at drain time.
    Data written after drain to the new instance = LOST on rollback (within the drain-to-rollback window).
    Document and escalate any data gap to CTO.

IF services are healthy but data corruption discovered (read-only anomaly check):
  → ROLLBACK-C: PITR restore old instance to T-30min
  → Action:
      aws rds restore-db-instance-to-point-in-time \
        --source-db-instance-identifier <old-instance-id> \
        --target-db-instance-identifier <old-instance-id>-pitr \
        --restore-time $(date -u -d "30 minutes ago" +%Y-%m-%dT%H:%M:%SZ)
  → Declare incident, page CTO
  → Time budget: ≤10 min to detect, PITR restore typically 15-30 min
```

---

## §Failover Drill Steps

Execute ≥24 hours after the cutover stabilizes (do NOT drill immediately after cutover):

```bash
# 1. Write sentinel rows from each service (pre-failover data-loss check)
# client-api: write via API
curl -X POST https://<endpoint>/api/v1/proposals \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "pe02-failover-sentinel-'$(date +%Y%m%d%H%M%S)'", "company_id": "<cid>"}'

# 2. Start health polling (1-second interval, 5-minute window)
for svc in 8001 8002 8003 8004 8005 8007; do
  (while true; do
    code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${svc}/health)
    echo "$(date +%H:%M:%S) port=${svc} http=${code}"
    sleep 1
  done) &
done

# 3. Trigger Multi-AZ failover (record T0)
T0=$(date +%s)
echo "Failover trigger at T0=$(date -d @${T0})"
aws rds reboot-db-instance \
  --db-instance-identifier eusolicit-staging \
  --force-failover

# 4. Continue polling for 5 minutes, then kill background jobs
sleep 300
kill %1 %2 %3 %4 %5 %6

# 5. Verify sentinel rows are readable (data-loss check)
# All 6 sentinels MUST be present — Multi-AZ is synchronous, so data loss = bug

# 6. Re-run integration tests
make test-integration
```

---

## §Connection Pool Audit

Verified via `grep -n "create_async_engine\|create_engine" services/*/src/*/dependencies.py services/*/src/*/db.py` on 2026-05-04:

| Service | Engine Factory | pool_pre_ping | pool_size | max_overflow | pool_recycle | Notes |
|---------|---------------|--------------|-----------|-------------|-------------|-------|
| client-api | `dependencies.py:get_engine()` | ✅ True | default (5) | default (10) | not set | AC-3 AP-GUARD-2: pool_pre_ping present |
| client-api (pipeline read) | `dependencies.py:get_pipeline_engine()` | ✅ True | 5 | default | not set | Cross-schema read engine |
| admin-api | `dependencies.py:get_engine()` | ✅ True | default | default | not set | |
| data-pipeline | `db.py` (sync) | ✅ True | default | default | not set | Celery workers use sync engine |
| ai-gateway | `services/db.py` | ✅ True | default | default | not set | |
| notification | `dependencies.py:get_engine()` | ✅ True | default | default | not set | Multiple task engines also set pool_pre_ping=True |
| integrations-api | `db.py` | ✅ True | default | default | not set | |

**Recommendation for pool_recycle:** Add `pool_recycle=300` to all 6 services' engine factories for Multi-AZ best practices
(prevents connections from accumulating state across RDS's standard 60-min idle connection management timeout).
This is tracked as a follow-up item; the current `pool_pre_ping=True` already provides the critical Multi-AZ
reconnect behaviour (AP-GUARD-2: pool_pre_ping=True is present in all engines).

---

## §Terraform Plan Evidence

The staging terraform plan was validated locally (no AWS credentials in the dev environment per D-2 pre-recorded deviation).
The plan summary expected from `terraform plan -var-file=environments/staging/terraform.tfvars`:

```
Terraform will perform the following actions:

  # aws_db_instance.main will be created
  + resource "aws_db_instance" "main" {
      + engine                              = "postgres"
      + engine_version                      = "16.4"
      + instance_class                      = "db.t3.medium"
      + multi_az                            = true
      + backup_retention_period             = 35
      + storage_encrypted                   = true
      + manage_master_user_password         = true
      + performance_insights_enabled        = true
      ...
    }

  # aws_db_subnet_group.main will be created
  + resource "aws_db_subnet_group" "main" { ... }

  # aws_db_parameter_group.main will be created
  + resource "aws_db_parameter_group" "main" { ... }

  # aws_security_group.db will be created
  + resource "aws_security_group" "db" { ... }

  # aws_secretsmanager_secret.db_service["client-api"] will be created
  # aws_secretsmanager_secret.db_service["admin-api"] will be created
  # aws_secretsmanager_secret.db_service["data-pipeline"] will be created
  # aws_secretsmanager_secret.db_service["ai-gateway"] will be created
  # aws_secretsmanager_secret.db_service["notification"] will be created
  # aws_secretsmanager_secret.db_service["integrations-api"] will be created
  # aws_secretsmanager_secret.db_service["migration"] will be created
  + resource "aws_secretsmanager_secret" ...

Plan: 11 to add, 0 to change, 0 to destroy.
```

**Note (D-2):** A live `terraform plan` against the staging AWS account was not executed in this session
(dev environment lacks staging cluster credentials — pre-recorded deviation D-2 applies).
The plan summary above is the expected output based on the implemented Terraform resources.
The on-call engineer MUST run `terraform plan -var-file=environments/staging/terraform.tfvars` and review
the actual output before applying.

---

## §Staging Rehearsal Timing

> Anti-pattern guard #6: NEVER skip the staging rehearsal "because the path is well-trodden."
> Story 21-1 evidence: even well-trodden paths surface DNS caching, ESO refresh latency,
> and kubelet rollout sequencing surprises. Rehearsal is MANDATORY before production cutover.

| Step | Description | Expected Duration | Actual Duration | Notes |
|------|-------------|------------------|-----------------|-------|
| Pre-flight | Terraform plan review + change ticket | 30 min | — | Pending staging cluster access |
| Phase 1.1-1.3 | Provision RDS + bootstrap roles + migrations | 20 min | — | Pending staging cluster access |
| Phase 1.4-1.5 | Create publication + subscription | 5 min | — | |
| Phase 1.6 | Wait for lag <1s | 10-15 min | — | Initial table sync time |
| Phase 2 write pause | Drain + verify lag=0 + drop subscription | 3-5 min | — | |
| Phase 2 rollout | ESO switch + service restart + health check | 3-5 min | — | |
| Total write pause | | ≤5 min | — | |
| Post-cutover validation | Integration tests + EXPLAIN ANALYZE | 10 min | — | |

**Status:** Staging rehearsal pending (D-2 deviation: dev environment lacks staging cluster credentials).
The rehearsal steps are documented above and will be executed by the on-call engineer with staging access.
Production cutover is BLOCKED until the staging rehearsal timing table is populated.

---

## §Failover Drill Results

> This section is populated after the failover drill executes (≥24h after staging cutover stabilizes).

| Service | Port | first_5xx_at | first_2xx_at | error_window_seconds | sentinel_row_intact | tests_pass |
|---------|------|-------------|-------------|---------------------|--------------------|----|
| client-api | 8001 | — | — | — | — | — |
| admin-api | 8002 | — | — | — | — | — |
| data-pipeline | 8003 | — | — | — | — | — |
| ai-gateway | 8004 | — | — | — | — | — |
| notification | 8005 | — | — | — | — | — |
| integrations-api | 8007 | — | — | — | — | — |

**Requirement:** all `error_window_seconds` values MUST be ≤30s per AC-6 (epic line 74).
**Status:** Pending staging rehearsal completion (D-2) and subsequent failover drill (D-1 for production).

---

## §Failover Drill Results — Production

> Populated by the operator-on-call after ≥24h of production stability post-cutover.
> Per AP17-C1, this drill requires separate operator approval — it is NOT executed by bmad-dev-story autopilot.

| Service | Port | first_5xx_at | first_2xx_at | error_window_seconds | sentinel_row_intact | tests_pass |
|---------|------|-------------|-------------|---------------------|--------------------|----|
| client-api | 8001 | — | — | — | — | — |
| admin-api | 8002 | — | — | — | — | — |
| data-pipeline | 8003 | — | — | — | — | — |
| ai-gateway | 8004 | — | — | — | — | — |
| notification | 8005 | — | — | — | — | — |
| integrations-api | 8007 | — | — | — | — | — |

---

## References

- Architecture ADR-010: `eusolicit-docs/planning-artifacts/architecture.md` lines 757–762
- Architecture §6.2 Production Topology: lines 624–634
- Architecture §6.5 Backup/DR: lines 667–671
- Epic E21 PE.02 scope: `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` lines 44–74
- Story 21-2 EXPLAIN ANALYZE evidence: `eusolicit-docs/implementation-artifacts/load-test-results.md` §EXPLAIN ANALYZE Results — Post-PE.02 Migration
- Pre-PE.02 plan (regression baseline): `eusolicit-docs/implementation-artifacts/load-test-results.md` §EXPLAIN ANALYZE Results (lines 434–620)
- Init SQL (canonical role bootstrap): `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql`
- Terraform module: `eusolicit-app/infra/terraform/modules/database/`
- ESO ExternalSecret CRD: `eusolicit-app/infra/helm/eusolicit-service/templates/externalsecret.yaml`
