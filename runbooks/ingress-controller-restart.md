# Runbook: nginx-Ingress Controller Restart

**Severity**: SEV-2 (if unplanned crashloop) / SEV-3 (if planned maintenance restart)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6)

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| nginx-ingress controller pod in CrashLoopBackOff | `kubectl get pods -n ingress-nginx` |
| All service endpoints returning 502 Bad Gateway or connection timeout | Customer reports + Grafana error-rate spike |
| cert-manager certificate renewal failure (nginx-ingress interaction) | `kubectl get certificate -A` |
| `HighErrorBudgetBurnRate` alert co-firing (all traffic impacted) | `error-budget-burn.md` |
| Kubernetes events showing pod eviction or OOMKill on ingress pod | `kubectl get events -n ingress-nginx` |
| Rolling restart in progress (planned; connection drain expected) | IC action log |

---

## Triage

1. **Check ingress controller pod status**:
   ```bash
   kubectl get pods -n ingress-nginx
   kubectl describe pod -n ingress-nginx <ingress-pod-name>
   ```
   - `CrashLoopBackOff` → crashing on startup → check logs.
   - `Running` with high restart count → intermittent crashes → check logs.
   - `Pending` → scheduling issue → check node resources.

2. **Check ingress controller logs**:
   ```bash
   kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --previous --tail=50
   ```
   Common causes:
   - OOMKill (high traffic volume exhausting memory).
   - TLS certificate file not found (cert-manager hasn't provisioned yet).
   - Configuration error (invalid Ingress annotation introduced by a recent deploy).

3. **Check PDB status** (PDB `minAvailable: 1` per PE.04 ensures at least 1 replica survives):
   ```bash
   kubectl get pdb -n ingress-nginx
   ```
   If PDB `ALLOWED DISRUPTIONS = 0` → all replicas are down or being evicted simultaneously → **SEV-1**.

4. **Check cert-manager status**:
   ```bash
   kubectl get certificate -A
   kubectl get certificaterequest -A
   ```
   Certificates in `Not Ready` state may block ingress if TLS is mandatory.

5. **Check for a recently applied Ingress resource** that may have invalid syntax:
   ```bash
   kubectl get ingress -n eusolicit -o yaml | grep -A5 "annotations"
   ```

---

## Resolution

### Branch A — Unplanned crash (CrashLoopBackOff)

1. **Rolling restart** (nginx-ingress supports zero-downtime rolling restart; PDB ensures ≥1 replica running):
   ```bash
   kubectl rollout restart deploy/ingress-nginx-controller -n ingress-nginx
   ```
   Monitor restart progress:
   ```bash
   kubectl rollout status deploy/ingress-nginx-controller -n ingress-nginx --timeout=5m
   ```

2. **PDB behaviour during rolling restart**: existing connections are handled by surviving replicas.
   - PDB `minAvailable: 1` prevents simultaneous eviction of all replicas.
   - New connections route to surviving replicas during the restart.
   - Zero customer-visible 502s expected if PDB is functioning correctly.

3. **If crash is OOMKill**: increase memory limit in Helm values and re-deploy. Current limit review:
   ```bash
   kubectl describe deploy/ingress-nginx-controller -n ingress-nginx | grep -A5 "Limits"
   ```

4. **If crash is cert-manager related** (TLS secret not found):
   ```bash
   kubectl get secret -n ingress-nginx | grep tls
   kubectl describe certificate -n ingress-nginx <cert-name>
   ```
   If certificate is `Not Ready` → check Let's Encrypt ACME challenge status:
   ```bash
   kubectl get challenges -A
   ```
   Force renewal: `kubectl delete certificate <cert-name> -n ingress-nginx` (cert-manager will re-issue).

5. **If crash is caused by an invalid Ingress annotation** (recent deploy):
   - Identify the bad Ingress: `kubectl get events -n ingress-nginx | grep -i error`.
   - Remove or fix the Ingress annotation in the Helm values.
   - Rolling-restart the ingress controller.

### Branch B — Planned maintenance restart (cert renewal, Helm upgrade)

1. **Verify PDB allows disruption**:
   ```bash
   kubectl get pdb -n ingress-nginx
   ```
   `ALLOWED DISRUPTIONS` must be ≥ 1.

2. **Announce in `#platform-incidents`**:
   > "Planned nginx-ingress controller restart at <time>. Brief connection drain expected. PDB protects availability. Duration: ~2 minutes."

3. **Execute rolling restart**:
   ```bash
   kubectl rollout restart deploy/ingress-nginx-controller -n ingress-nginx
   ```

4. **Monitor** (as per Branch A step 2).

---

## Verification

1. **Ingress controller pods are `Running` and `Ready`**:
   ```bash
   kubectl get pods -n ingress-nginx
   ```
   All pods `1/1 Running`.

2. **Zero customer-visible 502s** — check error rate panel in Grafana for the restart window.

3. **All service Ingress routes functioning**:
   ```bash
   curl -sf https://api.eusolicit.eu/health && echo "client-api: OK"
   curl -sf https://admin.eusolicit.eu/health && echo "admin-api: OK"
   ```

4. **cert-manager certificates valid**:
   ```bash
   kubectl get certificate -A | grep -v Ready
   ```
   Expected: no certificates in `Not Ready` state.

5. **PDB intact post-restart**:
   ```bash
   kubectl get pdb -n ingress-nginx
   ```
   `ALLOWED DISRUPTIONS` back to ≥ 1.

---

## Rollback

If rolling restart causes unexpected prolonged outage:

1. **Check rollout history** and rollback:
   ```bash
   kubectl rollout history deploy/ingress-nginx-controller -n ingress-nginx
   kubectl rollout undo deploy/ingress-nginx-controller -n ingress-nginx
   ```

2. If Helm was used for the upgrade: `helm rollback ingress-nginx <previous-revision>`.

3. If cert-manager interaction is blocking: temporarily disable TLS redirect and serve HTTP while cert is re-issued.

---

## Related

- `node-drain.md` — if ingress controller pod is on a draining node
- `error-budget-burn.md` — burn-rate alert co-fires if ingress is fully down
- `deploy-rollback.md` — if a Helm upgrade introduced the crash
- Story 21-4 `pe-04-chaos-drill-runbook.md` — PDB `minAvailable: 1` invariant evidence
