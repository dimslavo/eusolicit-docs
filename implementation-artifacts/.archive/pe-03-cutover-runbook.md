# PE.03 Redis HA Migration — Cutover Runbook

> **⚠ SUPERSEDED 2026-05-11 — historical reference only.**
> EU Solicit pivoted away from AWS ElastiCache to single-host on-premise Docker on www1.endigitalx.com.
> This runbook documents the planned AWS migration that was never executed.
> See: `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md` for the pivot decision
> and `eusolicit-docs/planning-artifacts/architecture.md` §ADR-010 (rewritten 2026-05-11) for the new direction.
> The active replacement story is `onprem-02-redis-persistence-and-recovery` (sprint-status `development_status`).
> The `redis-py` resilience hardening described here was retained verbatim in the code base — only the ElastiCache provisioning and failover-drill sections are obsolete.

**Story:** 21-3-redis-ha-migration-sentinel-or-managed-cluster
**Epic:** E21 Platform Reliability for 99.9% SLA
**Author:** Story 21-3 Dev Agent (2026-05-04)
**Status:** Ready for operator-gated staging rehearsal (D-2: documented dry-run; D-1: failover drill + production cutover deferred to operator)

> **Purpose:** This runbook is the canonical cutover guide for migrating
> the EU Solicit Redis instance from docker-compose / single-node ElastiCache
> to a managed Multi-AZ AWS ElastiCache Replication Group (Redis 7,
> cluster-mode-disabled, 1 primary + 2 replicas, eu-central-1).
>
> **D-1 Deviation:** Production cutover and failover drill require a live AWS
> Multi-AZ ElastiCache Replication Group. These are deferred to operator-action
> follow-up — `§Failover Drill Steps`, `§Failover Drill Results`, and
> `§Staging Rehearsal Timing` are pre-populated with the operator's procedure
> and placeholder evidence rows to be completed at execution time.
>
> **D-2 Deviation:** Staging rehearsal is documented as a dry-run (no live
> staging AWS cluster available at dev-agent implementation time). The
> `§Staging Rehearsal Timing` section documents the expected step-by-step
> timing for Method B and should be updated with real measurements when the
> operator executes against a live staging cluster.
>
> The staging rehearsal MUST be completed and signed-off before the production
> cutover is attempted. Never skip the rehearsal. (Anti-pattern guard #3.)

---

## §Pre-flight Checklist

Complete **all** items before beginning any cutover step:

- [ ] Terraform plan reviewed and approved (`terraform plan -var-file=environments/staging/terraform.tfvars` or prod)
- [ ] **LIVE Terraform plan executed against the target AWS account** (NOT the §Terraform Plan Evidence dry-run output — that is the pre-execution preview). Confirm: (i) one `aws_elasticache_replication_group.main` create; (ii) one `aws_elasticache_subnet_group.main` create; (iii) one `aws_elasticache_parameter_group.main` create; (iv) one `aws_security_group.redis` create; (v) six `aws_secretsmanager_secret.redis_service[*]` + `_version` creates; (vi) one `random_password.redis_auth` + `aws_secretsmanager_secret.redis_auth` + `_version`; (vii) two `aws_cloudwatch_log_group` creates. **No** destroys of stable infra.
- [ ] **Existing Redis state captured** via `redis-cli --rdb /tmp/pre-pe03-snapshot.rdb` (the snapshot is the rollback artefact for consumer-group offset recovery)
- [ ] **Helm `externalsecret.yaml` smoke-test rendered** for each service:
  ```bash
  for svc in client-api admin-api data-pipeline ai-gateway notification integrations-api; do
    helm template eusolicit-service -f infra/helm/values/${svc}.yaml \
      --set externalSecret.redis.enabled=true \
      --show-only templates/externalsecret.yaml | grep "REDIS_URL\|CELERY_BROKER_URL"
  done
  ```
  Each service must emit `<UPPER_SERVICE>_REDIS_URL`; data-pipeline + notification must ALSO emit `CELERY_BROKER_URL` + `CELERY_RESULT_BACKEND`.
- [ ] Staging dry-run signed-off (§Staging Rehearsal Timing populated)
- [ ] Change ticket filed in incident management system (24h advance notice minimum)
- [ ] On-call engineer paged and available for full maintenance window + 60 min post-cutover
- [ ] CTO notified (executive sponsor of the 99.9% SLA)
- [ ] ESO ClusterSecretStore "aws-secrets-manager" verified reachable:
  ```bash
  kubectl get clustersecretstore aws-secrets-manager -n external-secrets
  ```
- [ ] All 6 service `/health` endpoints returning 200 on existing Redis
- [ ] Rollback decision tree reviewed by on-call engineer (§Rollback Plan)
- [ ] New ElastiCache Replication Group provisioned and healthy (`aws elasticache describe-replication-groups --replication-group-id eusolicit-prod-redis`)
- [ ] ESO ExternalSecret CRDs applied per-service; K8s Secrets synced from Secrets Manager (`kubectl get externalsecret -n eusolicit`)

---

## §Cutover Method

**Selected method: Method B — Stop-the-World Maintenance Window** (5 minutes)

**Rationale (per epic line 87):** Redis state is ephemeral except event-stream offsets, which
replicate via the ElastiCache Replication Group's RDB snapshot import at seed time. A
5-minute planned maintenance window is fully acceptable for the EU Solicit workload profile
(B2B SaaS, EU business hours, no real-time critical path beyond alert delivery which is
queued via Celery). Method A (dual-write) would require touching every Redis write site
(not just `from_url` — every `xadd`, `set`, `incr`, Lua-script call) which is out of scope
per AC-5.1 anti-pattern guard #3.

**Method B is the recommended path per epic line 87.** Dual-write (Method A) remains
documented in AC-5.1 for future reference if zero-downtime is required post-launch.

---

## §Method B Steps (Stop-the-World, ≤5-minute Maintenance Window)

### Phase 1 — Pre-maintenance preparation (can run while old Redis is live)

1. **Provision new ElastiCache Replication Group** via Terraform:
   ```bash
   cd eusolicit-app/infra/terraform
   terraform plan -var-file=environments/prod/terraform.tfvars   # review
   terraform apply -var-file=environments/prod/terraform.tfvars   # execute
   ```
   Record the primary endpoint: `<new-redis-primary-endpoint>`.

2. **Seed consumer-group offsets** (preserves Redis Streams consumer-group state):
   ```bash
   # On old Redis: export RDB snapshot
   redis-cli --rdb /tmp/cutover-pe03.rdb
   # Upload to S3 for ElastiCache snapshot import
   aws s3 cp /tmp/cutover-pe03.rdb s3://eusolicit-infra-backups/pe03-cutover/cutover-pe03.rdb
   # Import into new ElastiCache via snapshot-from-S3 (ElastiCache console or CLI)
   aws elasticache create-snapshot \
     --replication-group-id eusolicit-prod-redis \
     --cache-cluster-id eusolicit-prod-redis-0001-001 \
     --snapshot-name pe03-pre-cutover-$(date +%Y%m%d)
   ```
   _Alternative for speed_: skip RDB import and accept consumer-group offset loss if
   all 6 consumer groups are catch-up tolerant (notification groups have XACK-on-process
   semantics; replay from beginning of stream is safe within the 7-day RDB retention window).

3. **Update Secrets Manager** entries for each service with the new primary endpoint:
   ```bash
   # The Terraform apply in step 1 creates these automatically via secrets.tf.
   # Verify they reference the correct primary endpoint:
   aws secretsmanager get-secret-value \
     --secret-id eusolicit/prod/cache/client-api \
     --query SecretString --output text | python3 -m json.tool
   # Expected: {"host": "<new-primary>", "port": 6379, "auth_token": "...", "url": "rediss://..."}
   ```

### Phase 2 — Maintenance window (≤5 minutes)

4. **Enable maintenance page** (if applicable):
   ```bash
   kubectl patch ingress eusolicit-ingress -n eusolicit \
     --patch '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/default-backend":"maintenance-svc"}}}'
   ```

5. **Drain in-flight requests** (wait 30 seconds):
   ```bash
   sleep 30
   ```

6. **Restart all 6 service deployments** (ESO will refresh the Redis secret on pod start):
   ```bash
   kubectl rollout restart deploy -l tier=backend -n eusolicit
   kubectl rollout restart deploy -l tier=worker -n eusolicit
   kubectl rollout status deploy -l tier=backend -n eusolicit --timeout=120s
   kubectl rollout status deploy -l tier=worker -n eusolicit --timeout=120s
   ```

7. **Verify Celery workers reconnect** to new broker:
   ```bash
   kubectl exec -n eusolicit deploy/data-pipeline-worker -- \
     celery -A data_pipeline.workers.celery_app inspect ping -t 10
   kubectl exec -n eusolicit deploy/notification-worker -- \
     celery -A notification.workers.celery_app inspect ping -t 10
   ```
   Expected: both workers respond with `pong`.

8. **Lift maintenance page**:
   ```bash
   kubectl patch ingress eusolicit-ingress -n eusolicit \
     --patch '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/default-backend":null}}}'
   ```

### Phase 3 — Verification (run immediately after maintenance window)

9. **Verify all 6 service `/health` endpoints return 200**:
   ```bash
   for port in 8001 8002 8003 8004 8005 8007; do
     curl -s -o /dev/null -w "port ${port}: %{http_code}\n" http://localhost:${port}/health
   done
   ```

10. **Run integration smoke-test** (Redis-touching endpoints per service):
    ```bash
    # client-api rate-limit (Redis-backed LoginRateLimiter)
    curl -s -w "%{http_code}" -X POST http://localhost:8001/api/v1/auth/login -d '{"email":"smoke@test.com","password":"x"}'
    # ai-gateway circuit-breaker state (Redis-backed)
    curl -s -w "%{http_code}" http://localhost:8004/health
    # integrations-api alert stream consumer-group resume
    curl -s -w "%{http_code}" http://localhost:8007/health
    ```

---

## §Validation Steps

After the maintenance window, execute these validation steps in order:

1. **All 6 service `/health` endpoints return 200** — see Phase 3 step 9 above.
2. **Redis ping from a debug pod**:
   ```bash
   kubectl run redis-test --rm -it --image=redis:7 -n eusolicit -- \
     redis-cli -u "rediss://default:<auth_token>@<primary-endpoint>:6379/0" PING
   # Expected: PONG
   ```
3. **Consumer-group re-attachment check**:
   ```bash
   kubectl exec -n eusolicit deploy/notification-worker -- \
     redis-cli -u "$NOTIFICATION_REDIS_URL" XINFO GROUPS eu-solicit:notifications
   # Verify: all consumer groups present with last-delivered-id ≥ pre-cutover snapshot
   ```
4. **Celery task smoke-test** — enqueue a health-check task and verify it completes:
   ```bash
   kubectl exec -n eusolicit deploy/data-pipeline-worker -- \
     celery -A data_pipeline.workers.celery_app call pipeline.health
   ```
5. **k6 Redis INCR regression smoke-test** (1K iterations, post-cutover baseline):
   ```bash
   k6 run tests/load/k6-redis-incr-10k.js \
     --env BASE_URL=http://localhost:8001 \
     --env AUTH_TOKEN=<staging-jwt> \
     --vus 10 --iterations 1000
   # Expected: final_count == 1000, incr_errors.rate = 0%
   ```
6. **Monitor Sentry + structured logs** for 15 minutes post-cutover; zero Redis connection errors expected.

---

## §Rollback Plan

**Decision tree** — execute the matching branch as soon as the failure mode is identified:

### Branch A — Failure BEFORE ESO secret swap (old Redis still the target)

**Symptom:** Terraform apply fails; or new ElastiCache unhealthy before maintenance window starts.

**Action:**
1. Abort cutover — old Redis instance remains authoritative; NO service impact.
2. Debug new ElastiCache instance separately (check subnet group, SG ingress, auth_token mismatch).
3. Re-schedule cutover for next maintenance window.
4. **Time budget:** no urgency; old Redis is still serving traffic.

### Branch B — Failure AFTER ESO secret swap, services not connecting

**Symptom:** Services restart but report Redis connection errors; `/health` returns 503; Celery workers not responding to ping.

**Action:**
1. Immediately revert ESO secrets to old primary endpoint:
   ```bash
   # Update each secret back to old endpoint
   for svc in client-api admin-api data-pipeline ai-gateway notification integrations-api; do
     aws secretsmanager put-secret-value \
       --secret-id eusolicit/prod/cache/${svc} \
       --secret-string '{"host":"<OLD-ENDPOINT>","port":6379,"auth_token":"<OLD-TOKEN>","url":"redis://default:<OLD-TOKEN>@<OLD-ENDPOINT>:6379/0"}'
   done
   ```
2. Restart all deployments to pick up reverted secrets:
   ```bash
   kubectl rollout restart deploy -l tier=backend -n eusolicit
   kubectl rollout restart deploy -l tier=worker -n eusolicit
   ```
3. Verify services reconnect to old Redis; lift maintenance page.
4. **Time budget:** ≤10 min to detect (monitoring alert), ≤20 min to execute rollback.

### Branch C — Services healthy, but consumer-group offset corruption detected

**Symptom:** `XINFO GROUPS` shows `last-delivered-id` regressed; duplicate notifications observed; billing sync emits duplicate Stripe usage records.

**Action:**
1. Execute Branch B rollback (revert to old primary).
2. Import pre-cutover RDB snapshot (`/tmp/pre-pe03-snapshot.rdb`) into old instance to restore offsets:
   ```bash
   redis-cli --pipe < /tmp/pre-pe03-snapshot.rdb
   ```
3. Declare incident; page CTO; file post-mortem.
4. Audit stream consumer groups for duplicate delivery; idempotency keys in billing sync prevent duplicate Stripe charges (action='set' is idempotent per billing_usage_sync.py).
5. **Time budget:** ≤20 min to execute rollback; post-mortem within 48h.

---

## §Failover Drill Steps

> **D-1 Deviation:** Requires live AWS Multi-AZ ElastiCache Replication Group.
> Deferred to operator-action follow-up post-cutover.

**Trigger (execute ≥24h after stable production cutover):**

```bash
aws elasticache test-failover \
  --replication-group-id eusolicit-prod-redis \
  --node-group-id 0001
# Record T0 = timestamp when API returns
```

**Per-service measurement protocol:**

```bash
# Start 1-second /health polling 30s before T0 for each service:
for port in 8001 8002 8003 8004 8005 8007; do
  while true; do
    echo "$(date +%T) port=${port} status=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:${port}/health)"
    sleep 1
  done &
done
# Record T0, then let polling run for 5 minutes
```

**Consumer-group offset verification (before and after):**

```bash
# BEFORE T0: capture last-delivered-id for all consumer groups
redis-cli -u "$CLIENT_API_REDIS_URL" XINFO GROUPS eu-solicit:notifications
redis-cli -u "$CLIENT_API_REDIS_URL" XINFO GROUPS eu-solicit:alerts
# ... for each stream

# AFTER failover stabilizes (≥30s post-T0):
redis-cli -u "$CLIENT_API_REDIS_URL" XINFO GROUPS eu-solicit:notifications
# Verify: last-delivered-id ≥ pre-failover value (monotonic; preserved across failover)
```

**Lua-script atomicity re-run (1K iterations):**

```bash
k6 run tests/load/k6-redis-incr-10k.js \
  --env BASE_URL=http://localhost:8001 \
  --env AUTH_TOKEN=<staging-jwt> \
  --vus 50 --iterations 1000
# Expected: final_count == 1000, incr_errors.rate = 0%
```

**Anti-pattern guards:**
- NEVER consider the drill passed without re-running the k6 INCR scenario (AC-6, AP-GUARD-6).
- NEVER run the production drill before the staging drill is signed-off (AC-6, AP-GUARD-7).
- NEVER use `aws elasticache reboot-cache-cluster` — use `test-failover` (primary-replica swap, not reboot).

---

## §Connection Audit

**Grep gate output** (executed 2026-05-04, pre-edit baseline):

```
grep -rn "aioredis\.from_url\|redis\.asyncio\.from_url\|aioredis\.Redis\.from_url\|redis\.Redis\.from_url\|redis_sync\.Redis\.from_url" services/*/src --include="*.py"
```

**All from_url connection sites after PE.03 hardening (AC-4 kwargs applied to all):**

| Service | File | Call Pattern | `health_check_interval` | `socket_keepalive` | `retry` | `retry_on_error` | Notes |
|---------|------|-------------|------------------------|-------------------|---------|-----------------|-------|
| integrations-api | `integrations_api/core/redis.py` | `aioredis.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Singleton get_redis_client |
| ai-gateway | `ai_gateway/services/redis_client.py` | `aioredis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | init_redis (startup lifecycle) |
| ai-gateway | `ai_gateway/routers/health.py` | `aioredis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Short-lived health check client |
| client-api | `client_api/dependencies.py` | `aioredis.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Singleton get_redis_client; backing _USAGE_LUA |
| notification | `notification/dependencies.py` | `aioredis.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Singleton get_redis_client |
| notification | `notification/workers/tasks/billing_usage_sync.py` | `redis_sync.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Sync redis-py; same kwargs apply |
| notification | `notification/workers/tasks/csm_stall_alerts.py` | `redis_sync.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Fire-and-forget stream publish |
| notification | `notification/workers/tasks/outcome_brief_generation.py` | `_redis.Redis.from_url` (lazy `import redis as _redis`) | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Best-effort outcome brief publish |
| notification | `notification/workers/celery_app.py` | `redis_client.from_url` (dead-letter handler, lazy `import redis as redis_client`) | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Fire-and-forget dead-letter publish |
| data-pipeline | `data_pipeline/workers/tasks/cleanup_expired.py` | `redis.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Sync Celery task |
| data-pipeline | `data_pipeline/workers/tasks/process_enrichment_queue.py` | `redis.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Sync Celery task |
| data-pipeline | `data_pipeline/workers/tasks/publish_event.py` | `redis.Redis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Event publish after crawl |
| integrations-api | `integrations_api/consumer.py` | `redis.asyncio.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Consumer group cg:integrations-api:alerts |
| admin-api | `admin_api/dependencies.py` | `aioredis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Singleton get_redis_client |
| admin-api | `admin_api/api/v1/load_test_helpers.py` | `aioredis.from_url` | ✅ 30 | ✅ True | ✅ Retry(ExponentialBackoff(cap=10, base=1), 3) | ✅ [ConnErr, TimeoutErr] | Celery broker health check |

**Celery broker resilience (AC-4.3):**

| Service | File | `broker_connection_retry_on_startup` | `broker_transport_options` keys | `result_backend_transport_options` keys |
|---------|------|--------------------------------------|--------------------------------|----------------------------------------|
| data-pipeline | `data_pipeline/workers/celery_app.py` | ✅ True | retry_on_timeout, max_retries, socket_keepalive, socket_connect_timeout, socket_timeout | retry_on_timeout, socket_keepalive, socket_connect_timeout, socket_timeout |
| notification | `notification/workers/celery_app.py` | ✅ True | retry_on_timeout, max_retries, socket_keepalive, socket_connect_timeout, socket_timeout | retry_on_timeout, socket_keepalive, socket_connect_timeout, socket_timeout |

**Test fixture decision (AC-4.5):**
`tests/conftest.py` redis_client fixture (`aioredis.from_url(redis_url, decode_responses=True)`) is intentionally left without the PE.03 resilience kwargs. Rationale: test environment uses local docker-compose Redis (no Multi-AZ failover concerns); resilience kwargs are safe to add but the blast-radius of test-side changes is zero benefit. Decision: leave unchanged to keep test-side parity with the pre-PE.03 baseline.

---

## §ESO Wiring Decision

**Chosen shape: Shape A** — extending the existing `externalsecret.yaml` Helm template.

**Rationale:**
- The existing `externalsecret.yaml` at `infra/helm/eusolicit-service/templates/externalsecret.yaml` already uses the `ClusterSecretStore "aws-secrets-manager"` + env-prefix-derivation patterns that PE.03 needs.
- Adding a SECOND `ExternalSecret` resource in the same file (activated via `.Values.externalSecret.redis.enabled`) keeps per-service values overhead minimal (one boolean flag per service) and avoids introducing a new template file.
- The resulting K8s Secret (`<service>-redis-secret`) has all Redis env-var keys and is mounted via `envFrom` alongside the DB secret (`<service>-db-secret`).
- Shape B (new `externalsecret-redis.yaml`) was considered but rejected: it would add 6 per-service values entries AND a new template file with negligible added separation benefit at this scope.

**Per-service env-var key mapping:**

| Service | K8s Secret Key | Env Var (envFrom mount) | Who Reads It |
|---------|---------------|------------------------|--------------|
| client-api | `CLIENT_API_REDIS_URL` | `CLIENT_API_REDIS_URL` | `BaseServiceSettings.redis_url` via env_prefix=CLIENT_API_ |
| admin-api | `ADMIN_API_REDIS_URL` | `ADMIN_API_REDIS_URL` | `BaseServiceSettings.redis_url` via env_prefix=ADMIN_API_ |
| data-pipeline | `DATA_PIPELINE_REDIS_URL` | `DATA_PIPELINE_REDIS_URL` | `BaseServiceSettings.redis_url` via env_prefix=DATA_PIPELINE_ |
| data-pipeline | `CELERY_BROKER_URL` | `CELERY_BROKER_URL` | `celery_app.py` line 26: `os.environ.get("CELERY_BROKER_URL", ...)` (MANDATORY — AP-GUARD-6) |
| data-pipeline | `CELERY_RESULT_BACKEND` | `CELERY_RESULT_BACKEND` | `celery_app.py` line 27: `os.environ.get("CELERY_RESULT_BACKEND", ...)` (MANDATORY — AP-GUARD-6) |
| ai-gateway | `AI_GATEWAY_REDIS_URL` | `AI_GATEWAY_REDIS_URL` | `BaseServiceSettings.redis_url` via env_prefix=AI_GATEWAY_ |
| notification | `NOTIFICATION_REDIS_URL` | `NOTIFICATION_REDIS_URL` | `BaseServiceSettings.redis_url` via env_prefix=NOTIFICATION_ |
| notification | `CELERY_BROKER_URL` | `CELERY_BROKER_URL` | `celery_app.py` + `billing_usage_sync.py` + `outcome_brief_generation.py` (MANDATORY — AP-GUARD-6) |
| notification | `CELERY_RESULT_BACKEND` | `CELERY_RESULT_BACKEND` | `celery_app.py` result backend (MANDATORY — AP-GUARD-6) |
| integrations-api | `INTEGRATIONS_API_REDIS_URL` | `INTEGRATIONS_API_REDIS_URL` | `BaseServiceSettings.redis_url` via env_prefix=INTEGRATIONS_API_ |

**DB-index decision (AC-3.6):**
ElastiCache Replication Group with cluster-mode-disabled DOES support `SELECT` to switch logical DB indices. DB 0 = application cache + streams; DB 1 = Celery result backend (same convention as docker-compose). This is NOT the cluster-mode-enabled restriction (which disallows `SELECT`). No application code changes needed.

**Per-service role provisioning:**
NOT required (unlike PE.02 which bootstrapped PostgreSQL roles). Redis uses a single shared `default` user with the auth_token via Redis AUTH; per-service ACL is out-of-scope for PE.03. No `null_resource` bootstrap step is analogous to PE.02 AC-3.2.

---

## §Terraform Plan Evidence

> **D-2 Note:** This section contains the expected plan output from a local `terraform validate`
> run (no live AWS credentials at dev-agent implementation time). The operator MUST execute
> a live `terraform plan` against the target AWS account before cutover and capture the real
> plan summary here.

**Local `terraform validate` result (2026-05-04):**

```
infra/terraform/modules/redis/main.tf: validated OK
infra/terraform/modules/redis/variables.tf: validated OK
infra/terraform/modules/redis/outputs.tf: validated OK
infra/terraform/modules/redis/secrets.tf: validated OK

terraform validate → Success! The configuration is valid.
terraform fmt -check -recursive → No formatting issues.
```

**Expected staging plan summary** (operator to populate with live output):

```
# Operator: run `terraform plan -var-file=environments/staging/terraform.tfvars | tail -40`
# and paste the resource summary here before cutover.
#
# Expected resources to create (staging):
#   + aws_elasticache_replication_group.main (eusolicit-staging-redis)
#   + aws_elasticache_subnet_group.main
#   + aws_elasticache_parameter_group.main (eusolicit-staging-redis-params)
#   + aws_security_group.redis
#   + random_password.redis_auth
#   + aws_secretsmanager_secret.redis_auth
#   + aws_secretsmanager_secret_version.redis_auth
#   + aws_cloudwatch_log_group.redis_slow_log
#   + aws_cloudwatch_log_group.redis_engine_log
#   + aws_secretsmanager_secret.redis_service["client-api"] (× 6 services)
#   + aws_secretsmanager_secret_version.redis_service["client-api"] (× 6 services)
#
# Plan: ~22 to add, 0 to change, 0 to destroy.
<PASTE LIVE PLAN SUMMARY HERE>
```

---

## §Staging Rehearsal Timing

> **D-2 Note:** This section is pre-populated with expected timing for Method B.
> Operator MUST update with real measurements during staging rehearsal.

| Step | Expected Duration | Actual (staging) | Notes |
|------|------------------|-----------------|-------|
| Phase 1.1 — terraform apply (staging) | ~10 min | TBD | ElastiCache creation takes ~5-8 min |
| Phase 1.2 — RDB snapshot + S3 upload | ~2 min | TBD | Depends on Redis data size |
| Phase 1.3 — Secrets Manager verification | ~1 min | TBD | 6 service secrets × JSON blob check |
| Phase 2.4 — Maintenance page up | ~10 sec | TBD | kubectl patch ingress |
| Phase 2.5 — Drain in-flight requests | 30 sec | TBD | Fixed wait |
| Phase 2.6 — Rollout restart all 6 services | ~2-3 min | TBD | Depends on image pull + ESO secret refresh |
| Phase 2.7 — Verify Celery workers | ~30 sec | TBD | celery inspect ping -t 10 |
| Phase 2.8 — Lift maintenance page | ~10 sec | TBD | kubectl patch ingress |
| Phase 3 — Validation (6×/health + smoke) | ~5 min | TBD | Including k6 1K-iter regression |
| **Total maintenance window** | **~5 min** | **TBD** | Steps 2.4 through 2.8 |
| **Total end-to-end** | **~25 min** | **TBD** | Including pre-maintenance prep |

**Staging rehearsal sign-off:**
- [ ] All 6 service `/health` endpoints returned 200 post-cutover
- [ ] Celery workers responded to `inspect ping` within 10s
- [ ] Consumer-group `XINFO GROUPS` showed `last-delivered-id` ≥ pre-snapshot value
- [ ] k6 1K-iter INCR smoke-test: `final_count == 1000`, `incr_errors.rate = 0%`
- [ ] Staged rollback verified (ESO secret reverted to old endpoint; services reconnected within 30s)
- [ ] **Signed off by:** ____________ **Date:** ____________

---

## §Failover Drill Results

> **D-1 Deviation:** Populated by the operator after ≥24h stable production cutover.
> The following table structure is pre-defined per AC-6.5.

### Per-Service Reconnect Timing

| Service | first_5xx_at (T0+Xs) | first_2xx_at (T0+Ys) | error_window_seconds | consumer_groups_resumed | lua_atomicity_preserved |
|---------|---------------------|---------------------|---------------------|------------------------|------------------------|
| client-api | TBD | TBD | TBD | N/A | TBD |
| admin-api | TBD | TBD | TBD | N/A | N/A |
| ai-gateway | TBD | TBD | TBD | N/A | N/A |
| data-pipeline | TBD | TBD | TBD | N/A | N/A |
| notification | TBD | TBD | TBD | TBD | N/A |
| integrations-api | TBD | TBD | TBD | TBD | N/A |

**Target:** all error_window_seconds < 10 (per AC-6 + epic line 93).

### Consumer Group Offsets

| Consumer Group | Stream | last_delivered_id_pre | last_delivered_id_post | Preserved (≥) |
|---------------|--------|----------------------|----------------------|---------------|
| cg:approval | eu-solicit:notifications | TBD | TBD | TBD |
| cg:opportunity | eu-solicit:notifications | TBD | TBD | TBD |
| cg:subprocessor | eu-solicit:notifications | TBD | TBD | TBD |
| cg:subscription | eu-solicit:notifications | TBD | TBD | TBD |
| cg:task | eu-solicit:notifications | TBD | TBD | TBD |
| cg:integrations-api:alerts | eu-solicit:alerts | TBD | TBD | TBD |

### Lua-Script Re-Run (post-failover)

| Scenario | VUs | Iterations | final_count | expected | incr_errors.rate | PASSED |
|----------|-----|-----------|-------------|----------|-----------------|--------|
| k6-redis-incr-10k.js (1K-iter subset) | 50 | 1000 | TBD | 1000 | TBD | TBD |

> **Anti-pattern guard (AC-6.6):** Failover drill is NOT considered passed until the k6
> Lua-script re-run shows `final_count == expected`. A healthy `/health` 2xx alone
> does not prove Lua atomicity survived failover.
