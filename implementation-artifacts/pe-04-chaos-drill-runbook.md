# PE.04 Chaos Drill Runbook — PodDisruptionBudgets + Min-Replica Enforcement

**Story:** 21-4-poddisruptionbudgets-min-replica-enforcement-across-all-services  
**Date Authored:** 2026-05-05  
**Epic:** E21 — Platform Reliability for 99.9% SLA  
**Status:** DEFERRED (chaos drill pending live EKS staging cluster — D-1 pre-recorded deviation)  
**Verdict:** DEFERRED — See §Sign-off

---

## §Pre-flight

### Cluster + Namespace Listing

```bash
# Verify cluster context is staging (NEVER run against production without operator sign-off)
kubectl config current-context
kubectl get nodes -o wide

# Verify all 6 service namespaces / pods are running
kubectl get pods -A -l "app.kubernetes.io/part-of=eusolicit" -o wide

# Verify the 6 PDB resources are in place
kubectl get pdb -A
# Expected output (6 PDBs — one per service):
#   NAMESPACE   NAME                     MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
#   eu-solicit  eusolicit-client-api     1               N/A               2
#   eu-solicit  eusolicit-admin-api      1               N/A               1
#   eu-solicit  eusolicit-ai-gateway     1               N/A               1
#   eu-solicit  eusolicit-data-pipeline  1               N/A               1
#   eu-solicit  eusolicit-notification   1               N/A               1
#   eu-solicit  eusolicit-integrations-api 1             N/A               1
```

### nginx-ingress PDB Verification (AC-5)

```bash
# Verify nginx-ingress namespace exists
kubectl get ns ingress-nginx

# Verify replica count >= 2
kubectl get deploy -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.replicas}'

# Verify PDB exists with minAvailable >= 1
kubectl get pdb -n ingress-nginx -o yaml

# Capture upstream values overrides applied at deploy time
helm get values ingress-nginx -n ingress-nginx
```

**Assessment (D-6 — no live staging cluster at time of authoring):**  
The upstream `ingress-nginx/ingress-nginx` chart ships `controller.minAvailable: 1` + `controller.replicaCount: 2` as defaults. If the staging cluster was deployed with default chart values (no override), the PDB is already in place with `minAvailable: 1` and replica count = 2.

**Action required:** Operator must run the verification commands above against the live staging cluster and record the output below.

```
# OPERATOR CAPTURE — fill in after live verification
nginx-ingress PDB status:      [ ] present (minAvailable: 1) / [ ] missing
nginx-ingress replica count:   ____
Override patch required:       [ ] YES — apply infra/helm/values/ingress-nginx-overrides.yaml
                               [ ] NO — upstream defaults sufficient
```

**If PDB missing or replicas < 2:** Apply the override patch:
```bash
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -f infra/helm/values/ingress-nginx-overrides.yaml \
  -n ingress-nginx
```

The override file is at `eusolicit-app/infra/helm/values/ingress-nginx-overrides.yaml`:
```yaml
controller:
  replicaCount: 2
  minAvailable: 1
  # Per ingress-nginx upstream chart: controller.minAvailable maps to a
  # PodDisruptionBudget. Set replicaCount: 2 so minAvailable: 1 has slack.
```

---

## §HPA Sizing Review (AC-4)

Review of `autoscaling.maxReplicas` against Story 21-1 `load-test-results.md` §Sizing Recommendations for PE.04 (lines 808–827). **No maxReplicas changes made in this story** — all deferred to staging measurement (D-3).

| Service | minReplicas (post-PE.04) | maxReplicas | Story 21-1 Evidence Ref | Status |
|---------|--------------------------|-------------|-------------------------|--------|
| client-api | 3 | 20 | §Opportunities FTS — p95 14.4ms at 50 VUs; FTS migrated to GIN index (PE.02); 20 is a conservative ceiling | Deferred to staging measurement |
| admin-api | 2 | 3 | No k6 baseline (IP-allowlisted; not in load-test scope) | Deferred to staging measurement |
| ai-gateway | 2 | 10 | §SSE Concurrency Cap — TTFB p95 4.4ms at 12 VUs; cap-validation rejection rate not measured locally (401 returned before semaphore); line 817 confirms "default PDB minAvailable: 1 and HPA min-replicas: 2 stand; revisit after staging measurement" | Deferred to staging measurement per line 817 |
| data-pipeline | 2 | 8 | §data-pipeline-worker — queue depth not measured locally (Celery scoring task requires KraftData fixtures); line 826: "queue-depth-driven HPA scale decision deferred to staging measurement" | Deferred per D-3 |
| notification | 2 | 4 | No direct k6 baseline (event-driven Celery worker not in HTTP load-test scope) | Deferred to staging measurement |
| integrations-api | 2 | 4 | NEW service — no baseline today (post-Story 21-1) | Deferred to staging measurement |
| enterprise-api | 2 | 10 | No k6 baseline (Story 12.15 set maxReplicas: 10 unmeasured — canonical example of unmeasured value; visible in §HPA Sizing Review per AC-4.5 anti-pattern #9) | Deferred to staging measurement |

**Conclusion:** All `maxReplicas` values unchanged per D-3. Future operator measurement against staging/production load data triggers any change (per epic line 112 anti-pattern #9: NEVER raise maxReplicas without measurement evidence).

---

## §Drain Procedure (AC-8)

The canonical command sequence for draining a node during the PE.04 chaos drill. Execute **sequentially per service** — never drain more than one node simultaneously (AP-GUARD-13).

### Pre-Drill Checklist

```bash
# 1. Confirm all pods are Running + Ready
kubectl get pods -n eu-solicit -l "app.kubernetes.io/name=<service>" -o wide

# 2. Confirm PDB is in place and ALLOWED DISRUPTIONS >= 1
kubectl describe pdb eusolicit-<service> -n eu-solicit

# 3. Start traffic generator (parallel — runs throughout the drill)
#    client-api + ai-gateway: k6 run tests/load/k6-perf-core-flows.js
#    data-pipeline + notification + integrations-api: curl polling loop (no HTTP surface)
k6 run tests/load/k6-perf-core-flows.js &
# OR for services without HTTP surface:
# while true; do curl --max-time 3 -s -o /dev/null -w "%{http_code}" http://<service>:<port>/healthz; echo; sleep 1; done
```

### Per-Node Drain Steps

```bash
# Step 1 — Identify node hosting the service replica
kubectl get pod -l app.kubernetes.io/name=<service> -o wide -n eu-solicit
# Capture NODE column output: e.g. NODE=ip-10-0-1-45.eu-central-1.compute.internal

# Step 2 — Cordon the node (prevent new pods from scheduling onto it)
kubectl cordon <node-name>

# Step 3 — Drain the node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# Expected output for PDB-protected service (minReplicas=2, minAvailable=1):
#   evicting pod eu-solicit/eusolicit-<service>-<hash>
#   pod/eusolicit-<service>-<hash> evicted
# OR (if second replica is not yet Ready):
#   Cannot evict pod as it would violate the pod's disruption budget.
#   Retrying after 5s...  <- PDB enforcing; this is CORRECT behaviour

# Step 4 — Validate continuity (observe 0 5xx during drain window)
# k6 output should show zero http_req_failed entries for the service under test.
# For services using curl: no non-2xx responses during the drain window.

# Step 5 — Confirm reschedule
kubectl get pod -l app.kubernetes.io/name=<service> -n eu-solicit
# All replicas must be Running+Ready on different nodes than <node-name>

# Step 6 — Uncordon
kubectl uncordon <node-name>
```

---

## §Per-Service Drill (AC-8.1)

One subsection per service. Operator fills in evidence captures during live drill execution.

### client-api

| Step | Command | Expected | Operator Capture |
|------|---------|----------|-----------------|
| 1 | `kubectl get pod -l app.kubernetes.io/name=client-api -o wide -n eu-solicit` | 3 replicas on 3 different nodes | PENDING |
| 2 | Start k6: `k6 run tests/load/k6-perf-core-flows.js` | k6 running with 0% failure rate | PENDING |
| 3 | `kubectl cordon <node>` | node cordoned | PENDING |
| 4 | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` | Pod evicted; remaining 2 replicas serving | PENDING |
| 5 | Observe drain output | PDB enforcement message OR clean eviction | PENDING |
| 6 | k6 error count | 5xx count == 0 | PENDING |
| 7 | `kubectl get pod -l app.kubernetes.io/name=client-api -n eu-solicit` | 3 replicas Running+Ready on different nodes | PENDING |
| 8 | `kubectl uncordon <node>` | node returned to pool | PENDING |

**Verdict:** PENDING (D-1 — deferred to operator-on-call execution)

---

### admin-api

| Step | Command | Expected | Operator Capture |
|------|---------|----------|-----------------|
| 1 | `kubectl get pod -l app.kubernetes.io/name=admin-api -o wide -n eu-solicit` | 2 replicas on 2 different nodes | PENDING |
| 2 | `curl` polling: `while true; do curl --max-time 3 -s http://admin.eusolicit.com/healthz; sleep 1; done` | HTTP 200 responses throughout | PENDING |
| 3-8 | As per §Drain Procedure | 5xx count == 0; replicas reschedule | PENDING |

**Verdict:** PENDING (D-1)

---

### ai-gateway

| Step | Command | Expected | Operator Capture |
|------|---------|----------|-----------------|
| 1 | `kubectl get pod -l app.kubernetes.io/name=ai-gateway -o wide -n eu-solicit` | 2 replicas on 2 different nodes | PENDING |
| 2 | k6: `k6 run tests/load/k6-agent-endpoints.js` (if exists) OR `curl` polling against port 8004 | 0 errors | PENDING |
| 3-8 | As per §Drain Procedure | 5xx count == 0 | PENDING |

**Verdict:** PENDING (D-1)

---

### data-pipeline

| Step | Command | Expected | Operator Capture |
|------|---------|----------|-----------------|
| 1 | `kubectl get pod -l app.kubernetes.io/name=data-pipeline -o wide -n eu-solicit` | 2 replicas on 2 different nodes | PENDING |
| 2 | `curl` polling: `while true; do curl --max-time 3 -s http://localhost:8003/healthz; sleep 1; done` | HTTP 200 responses | PENDING |
| 3-8 | As per §Drain Procedure | 5xx count == 0 | PENDING |
| Lua-check | `k6 run tests/load/k6-redis-incr-10k.js --iterations 1000` (1K-iter subset per D-2) | `final_count == 1000` (Lua atomicity preserved) | PENDING |

**Verdict:** PENDING (D-1)

---

### notification

| Step | Command | Expected | Operator Capture |
|------|---------|----------|-----------------|
| 1 | `kubectl get pod -l app.kubernetes.io/name=notification -o wide -n eu-solicit` | 2 replicas on 2 different nodes | PENDING |
| 2 | `curl` polling against port 8005 | HTTP 200 from /healthz | PENDING |
| 3-8 | As per §Drain Procedure | 5xx count == 0; Celery tasks resume after reschedule | PENDING |

**Verdict:** PENDING (D-1)

---

### integrations-api

| Step | Command | Expected | Operator Capture |
|------|---------|----------|-----------------|
| 1 | `kubectl get pod -l app.kubernetes.io/name=integrations-api -o wide -n eu-solicit` | 2 replicas on 2 different nodes | PENDING |
| 2 | `curl` polling against port 8007 | HTTP 200 from /health | PENDING |
| 3-8 | As per §Drain Procedure | 5xx count == 0; Redis-Streams consumer group resumes | PENDING |

**Verdict:** PENDING (D-1)

---

## §PDB-Behaviour Evidence (AC-8.3)

Expected `kubectl drain` output excerpts demonstrating PDB enforcement. Operator captures verbatim output here.

**For a service with `minAvailable: 1` and `minReplicas: 2` (exactly 2 replicas running):**

```
# Case A — clean eviction (second replica already Ready on another node):
Evicting pod eu-solicit/eusolicit-admin-api-<hash>
pod/eusolicit-admin-api-<hash> evicted

# Case B — PDB enforcement (second replica not yet Ready):
Cannot evict pod as it would violate the pod's disruption budget.
Waiting for at least 1 Ready replicas for PodDisruptionBudget "eusolicit-admin-api"...
  [retrying after 5s]
pod/eusolicit-admin-api-<hash> evicted  # eventually evicted after new replica becomes Ready
```

Both Case A and Case B are correct. Case B proves the PDB is actively enforcing.

**Operator capture slots (fill during live drill):**

```
client-api drain output:        [PENDING]
admin-api drain output:         [PENDING]
ai-gateway drain output:        [PENDING]
data-pipeline drain output:     [PENDING]
notification drain output:      [PENDING]
integrations-api drain output:  [PENDING]
```

---

## §Cluster-Autoscaler Behaviour (AC-7)

> Per ADR-010 (architecture.md lines 757–764), the EKS cluster is SRE-provisioned at the AWS account level. These values are documentation captures, not Terraform in this repo (AP-GUARD-12).

### Expected Cluster-Autoscaler Behaviour During Drain

When a node is drained:
1. Pods on the drained node become Pending (no schedulable capacity on that node).
2. Kubernetes scheduler attempts to place Pending pods on remaining nodes.
3. If cluster has capacity (other nodes have headroom), reschedule completes immediately.
4. If cluster is at capacity, Cluster Autoscaler detects Pending pods within `scan-interval` (default: 10s) and requests a new node from the Auto Scaling Group.
5. New node becomes Ready (typically 2–4 minutes for EKS managed nodes).
6. Pending pods reschedule onto the new node.
7. After scale-down delay (default: 10 minutes post-scale-up), Cluster Autoscaler may reclaim the drained+uncordoned node if it becomes underutilized.

### SRE-Captured Values (Operator Must Fill)

```
Managed node group name:      ___________________________
Instance type(s):             ___________________________
min_size:                     ___  (AWS Auto Scaling Group min)
desired_capacity:             ___  (current desired)
max_size:                     ___  (AWS Auto Scaling Group max)
Cluster Autoscaler scan-interval:  10s (default; override if changed)
Scale-down delay:             10m (default; override if changed)
Scale-down unneeded time:     10m (default; override if changed)
```

### max_size Sanity Check Formula

```
max_size_floor = ceil(sum(maxReplicas) × avg_pod_cpu_request / node_cpu_capacity)
             = ceil((20 + 3 + 10 + 8 + 4 + 4 + 10) × 200m / node_cpu_capacity)
             = ceil(59 × 0.2 / node_cpu_cores)
             = ceil(11.8 / node_cpu_cores)
```

For m5.xlarge (4 vCPU): ceil(11.8 / 4) = 3 nodes minimum. With safety buffer 2×: 6 nodes.
Operator must confirm `max_size` ≥ this floor.

```
Computed max_size_floor:      ___  nodes
Actual max_size:              ___  nodes (must be >= max_size_floor)
Sufficient capacity:          [ ] YES  [ ] NO — if NO, request SRE to raise max_size before drill
```

---

## §Network-Policy Verification (AC-6)

Verification scan of all 7 values files' `networkPolicy.ingress` and `networkPolicy.egress` rules for HA-safe selector patterns. Results as of Story 21-4 implementation (2026-05-05).

| Service | Rule Direction | Rule Type | Selector Used | Verdict |
|---------|---------------|-----------|---------------|---------|
| client-api | ingress | from | `namespaceSelector: kubernetes.io/metadata.name: ingress-nginx` + `podSelector: app.kubernetes.io/name: ingress-nginx` | PASS — label-selector ✓ |
| client-api | egress DNS | to | `namespaceSelector: kubernetes.io/metadata.name: kube-system` | PASS — label-selector ✓ |
| client-api | egress PG | to | `[]` (allow-all namespaces on port 5432) | PASS — acceptable for external DB ✓ |
| client-api | egress Redis | to | `[]` (allow-all namespaces on port 6379) | PASS — acceptable for external Redis ✓ |
| client-api | egress ai-gateway | to | `podSelector: app.kubernetes.io/name: ai-gateway` | PASS — label-selector ✓ |
| client-api | egress HTTPS | to | `[]` (allow-all on port 443) | PASS — external HTTPS ✓ |
| admin-api | ingress | from | `ipBlock: cidr: 10.0.0.0/8`, `ipBlock: cidr: 172.16.0.0/12` | PASS — CIDR ipBlock acceptable for VPN/office IP allowlist (AC-6.1) ✓ |
| admin-api | egress | — | `[]` (no egress rules defined) | PASS — empty egress = allow-all ✓ |
| ai-gateway | ingress | from | `podSelector: app.kubernetes.io/name: client-api`, `podSelector: app.kubernetes.io/name: admin-api`, `podSelector: app.kubernetes.io/name: data-pipeline` | PASS — label-selectors ✓ |
| ai-gateway | egress DNS | to | `namespaceSelector: kubernetes.io/metadata.name: kube-system` | PASS — label-selector ✓ |
| ai-gateway | egress HTTPS | to | `[]` (allow-all on port 443) | PASS — external HTTPS ✓ |
| data-pipeline | ingress | — | `[]` (empty — no inbound) | PASS — no inbound traffic ✓ |
| data-pipeline | egress DNS | to | `namespaceSelector: kubernetes.io/metadata.name: kube-system` | PASS — label-selector ✓ |
| data-pipeline | egress outbound | to | `[]` (allow-all on ports 443, 5432, 6379) | PASS — external services ✓ |
| notification | ingress | — | `[]` (empty — no inbound) | PASS — no inbound traffic ✓ |
| notification | egress DNS | to | `namespaceSelector: kubernetes.io/metadata.name: kube-system` | PASS — label-selector ✓ |
| notification | egress PG/Redis/HTTPS | to | `[]` (allow-all on ports 5432, 6379, 443) | PASS — external services ✓ |
| integrations-api | ingress | — | `[]` (empty — consumer-side only) | PASS — no inbound traffic ✓ |
| integrations-api | egress DNS | to | `namespaceSelector: kubernetes.io/metadata.name: kube-system` | PASS — label-selector ✓ |
| integrations-api | egress PG/Redis/HTTPS | to | `[]` (allow-all on ports 5432, 6379, 443) | PASS — external services ✓ |
| enterprise-api | ingress | from | `namespaceSelector: kubernetes.io/metadata.name: ingress-nginx` + `podSelector: app.kubernetes.io/name: ingress-nginx` | PASS — label-selector ✓ |
| enterprise-api | egress client-api | to | `podSelector: app.kubernetes.io/name: client-api` | PASS — label-selector ✓ |
| enterprise-api | egress PG/Redis/DNS | to | `namespaceSelector` + `[]` patterns | PASS ✓ |

**Aggregate verdict: ALL PASS** — no literal pod IP addresses found. All cross-service rules use `podSelector` or `namespaceSelector` label-matching. HA failover paths are label-safe: when a replica is replaced after a drain, the new pod has a different IP but the same labels — all existing egress rules continue to allow the connection.

**Anti-pattern guard #11 verification:** `grep -r "to: { ip:" infra/helm/values/*.yaml` → **no results** (confirmed at authoring time, 2026-05-05).

---

## §Chaos-Drill Results (AC-8.2)

Aggregate result table. Operator fills in after live drill execution.

| Service | Drain Start | Drain End | Drain Duration | 5xx Count | New Replica Ready Time | Verdict |
|---------|------------|-----------|---------------|-----------|----------------------|---------|
| client-api | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| admin-api | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| ai-gateway | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| data-pipeline | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| notification | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| integrations-api | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |

**Aggregate Verdict:** DEFERRED  
**Reason:** Autopilot does not execute `kubectl drain` against a live AWS-account staging cluster. The chaos-drill is structurally complete in this runbook with operator-output capture placeholders. Production chaos-drill lands as a separate operator-on-call ticket post-merge (D-1 pre-recorded deviation: DEVIATION_TYPE: ACCEPTANCE_GAP, DEVIATION_SEVERITY: deferrable).

**Required to promote verdict from DEFERRED to PASSED:**
1. All 6 service rows show `5xx Count == 0`
2. All 6 service rows show `New Replica Ready Time <= 60s`
3. Lua-script-survives-drain check: `final_count == 1000` for data-pipeline 1K-iter run
4. Operator signs off in §Sign-off below

---

## §Sign-off (AC-10, AC-8.3)

| Field | Value |
|-------|-------|
| Author | bmad-dev-story autopilot (Claude Sonnet 4.6) |
| Date | 2026-05-05 |
| Story | 21-4-poddisruptionbudgets-min-replica-enforcement-across-all-services |
| Implementation Verdict | COMPLETED (code + config + CI gate + tests) |
| Chaos-Drill Verdict | **DEFERRED** — live staging drain pending operator-on-call execution (D-1) |
| PE.04 Gate Status | Code/config gate PASSED; operational drill DEFERRED |

**Operator Sign-off (to be completed after live drill):**

```
Operator name:          ___________________________
Drill execution date:   ___________________________
Drill environment:      [ ] staging / [ ] production
Aggregate verdict:      [ ] PASSED (all 6 services, 5xx Count == 0)
                        [ ] FAILED — root-cause: ___________________________
Signature:              ___________________________
```

> **Note for PE.06 runbook authors:** This file is the canonical procedure for safely removing an EKS node during maintenance. The §Drain Procedure is the reference for the on-call runbook entry "Node Drain During Maintenance." If the §Chaos-Drill Results table shows DEFERRED rather than PASSED, the procedure has been reviewed and documented but not yet executed against a live cluster — factor this into SLA confidence assessment.

---

## Known Deviations

| # | Deviation | Type | Severity |
|---|-----------|------|---------|
| D-1 | Production chaos drill deferred to operator-action follow-up. Autopilot cannot execute kubectl drain against live AWS staging cluster. | ACCEPTANCE_GAP | deferrable |
| D-2 | Lua-script-survives-drain check (AC-8.4) may execute as 1K-iter subset if full 10K isn't feasible in the chaos drill window. Atomicity verifiable at 1K iter. | ACCEPTANCE_GAP | cosmetic |
| D-3 | maxReplicas adjustments deferred to staging measurement. Story 21-1 line 826: "queue-depth-driven HPA scale decision deferred to staging measurement." §HPA Sizing Review captures current values + deferred verdict. | ACCEPTANCE_GAP | deferrable |
| D-5 | Cluster Autoscaler config documented (§Cluster-Autoscaler Behaviour) but not modified. SRE-managed at AWS account level per ADR-010. | ARCHITECTURAL_GAP | cosmetic |
| D-6 | nginx-ingress override patch conditional on upstream-defaults check. If staging already has replicaCount: 2 + minAvailable: 1 from upstream chart defaults, no override file required. File `infra/helm/values/ingress-nginx-overrides.yaml` provided for the case where PDB is missing. | SCOPE_GAP | cosmetic |
