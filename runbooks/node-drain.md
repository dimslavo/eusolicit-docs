# Runbook: Kubernetes Node Drain

**Severity**: SEV-2 (if unplanned) / SEV-3 (if planned maintenance)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) — lifts operator-facing content from PE.04 `pe-04-chaos-drill-runbook.md` §Drain Procedure + §Per-Service Drill

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| Karpenter / Cluster-Autoscaler node-drain event in Kubernetes events | `kubectl get events -n kube-system` |
| AWS EC2 Spot interruption or scheduled maintenance | AWS Console / CloudWatch |
| Operator-initiated `kubectl drain` for maintenance | IC action log in `#platform-incidents` |
| Pod evictions across multiple services simultaneously | `kubectl get pods -n eusolicit` |
| PDB-protected pod stuck in `Pending` (PDB blocks eviction) | `kubectl describe pdb -n eusolicit` |
| Service health endpoint degraded during pod rescheduling | Per-service Grafana dashboard |

---

## Triage

1. **Identify the node being drained**:
   ```bash
   kubectl get nodes
   kubectl describe node <node-name> | grep -E "(Conditions|Taints|Draining)"
   ```

2. **Check PDB status** (PDB protects against unsafe evictions):
   ```bash
   kubectl get pdb -n eusolicit
   ```
   All 6 services have `minAvailable: 1` PDB per PE.04. Verify:
   - `ALLOWED DISRUPTIONS` should be ≥ 1 before drain proceeds.
   - If `ALLOWED DISRUPTIONS = 0` → PDB is blocking eviction → **correct response** (wait for rescheduling or scale up).

3. **Check replica counts** (all services must have `minReplicas ≥ 2` per PE.04 invariant):
   ```bash
   kubectl get deploy -n eusolicit -o custom-columns=NAME:.metadata.name,REPLICAS:.spec.replicas,READY:.status.readyReplicas
   ```
   If any service has `READY < 2` → it cannot tolerate further evictions safely.

4. **Identify which services have pods on the draining node**:
   ```bash
   kubectl get pods -n eusolicit -o wide | grep <node-name>
   ```

5. **Was this Karpenter/Autoscaler (automatic) or operator-initiated (planned)?**
   - Automatic: proceed with monitoring.
   - Planned: confirm IC approval in `#platform-incidents` Slack before proceeding.

---

## Resolution

### Branch A — Operator-initiated planned drain

1. **Pre-drain checklist** (per PE.04 §Pre-Drill Checklist):
   - Confirm all 6 services have ≥ 2 ready replicas.
   - Confirm all PDBs have `ALLOWED DISRUPTIONS ≥ 1`.
   - Confirm HPA is not at `minReplicas` (scale-out headroom available).
   - Post intent in Slack `#platform-incidents`.

2. **Cordon the node** (prevent new pod scheduling):
   ```bash
   kubectl cordon <node-name>
   ```

3. **Drain the node** with PDB enforcement and emptyDir cleanup:
   ```bash
   kubectl drain <node-name> \
     --ignore-daemonsets \
     --delete-emptydir-data \
     --grace-period=300 \
     --timeout=600s
   ```
   - `--ignore-daemonsets`: required (DaemonSet pods cannot be drained; they are rescheduled automatically).
   - `--delete-emptydir-data`: required for pods using emptyDir volumes.
   - `--grace-period=300`: 5-minute graceful shutdown window per service (matches Kubernetes termination-grace-period).
   - `--timeout=600s`: abort if drain takes >10 minutes (prevents indefinite blocking).

4. **Validate continuity during drain** (per PE.04 §Per-Service Drill):
   ```bash
   # Run in parallel in a separate terminal during drain
   while true; do
     for svc in client-api admin-api agenticsai-gateway data-pipeline notification integrations-api; do
       STATUS=$(kubectl get deploy/$svc -n eusolicit -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
       echo "$svc ready=$STATUS"
     done
     sleep 5
   done
   ```
   All services must maintain `readyReplicas ≥ 1` throughout drain. If any service drops to 0 → PDB enforcement failure → investigate immediately.

5. **Uncordon after maintenance is complete**:
   ```bash
   kubectl uncordon <node-name>
   ```
   Or, if the node is permanently removed (Spot interruption), allow Karpenter to provision a replacement.

### Branch B — Automated drain (Karpenter/Spot interruption)

1. Karpenter handles drain automatically when it terminates a node for consolidation or Spot interruption.
2. **Monitor pod rescheduling**:
   ```bash
   kubectl get events -n eusolicit --sort-by='.lastTimestamp' | tail -20
   ```
3. **Confirm all services return to `minReplicas`** after rescheduling (typically 60–120s).
4. If HPA cannot scale out (insufficient cluster capacity) → Karpenter provisions a new node. Wait for new node to join:
   ```bash
   kubectl get nodes -w
   ```

### Branch C — PDB blocking eviction (service stuck at `ALLOWED DISRUPTIONS = 0`)

This is **correct PDB behavior** — the runbook should NOT override PDB protection.

1. Confirm the blocking service's pod count: `kubectl get pods -n eusolicit | grep <service>`.
2. If the blocking pod is Pending (not running) → it is already being rescheduled; wait.
3. If the blocking pod is Running on the draining node → it cannot be evicted safely.
   - Scale up the service: `kubectl scale deploy/<service> -n eusolicit --replicas=<N+1>`.
   - Wait for new replica to become Ready.
   - Re-run drain: `kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --grace-period=300`.

---

## Verification

1. **All services maintain ≥ 1 ready replica throughout drain** (PDB invariant):
   ```bash
   kubectl get deploy -n eusolicit
   ```
   All `READY` counts must be ≥ 1 (never 0) during drain.

2. **All services return to `minReplicas`** after drain completes:
   - client-api: 2 replicas; admin-api: 2; agenticsai-gateway: 2; data-pipeline: 2; notification: 2; integrations-api: 2.

3. **HPA functions correctly**: if drain caused scale-out, HPA should scale back down after traffic normalises.

4. **Draining node is fully drained** (all pods evicted or rescheduled):
   ```bash
   kubectl get pods -n eusolicit -o wide | grep <node-name>
   ```
   Expected: no pods remaining on the draining node.

5. **No customer-visible impact** — error rate and latency remain within SLO bounds during drain.

---

## Rollback

1. **If drain causes service disruption** (a pod drops to 0 ready replicas):
   ```bash
   kubectl uncordon <node-name>  # Allow pod rescheduling back to this node
   ```
   Then investigate why PDB protection failed before re-draining.

2. **If new node provisioning is slow** (Karpenter not scaling up):
   ```bash
   kubectl logs -n karpenter deploy/karpenter --since=10m | grep -E "(provision|launch|error)"
   ```
   Check for capacity constraints in the region. Fall back to manual node scaling via the EKS managed node group.

3. **Post-drain pod restarts expected**: connection pools re-establish; this is normal. Monitor for > 2 restarts on the same pod within 5 minutes (CrashLoopBackOff pattern → separate investigation).

---

## Related

- `pg-failover.md` — if node drain triggers DB reconnect
- `redis-failover.md` — if node drain triggers Redis reconnect
- `ingress-controller-restart.md` — if the ingress controller pod is on the draining node
- `deploy-rollback.md` — if a deploy was in-flight during drain
- Story 21-4 `pe-04-chaos-drill-runbook.md` §Drain Procedure + §Per-Service Drill — PDB evidence from chaos drill (dev-side audit trail)
