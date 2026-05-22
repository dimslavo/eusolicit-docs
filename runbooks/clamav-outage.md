# Runbook: ClamAV Scanner Outage

**Severity**: SEV-2 (file-upload feature degraded)
**SLA-Scope**: EXEMPT (security scanner; degraded scan = degraded feature, not platform outage)
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6)

---

> **SLA-EXEMPT notice**: ClamAV is a security-scanner component. A ClamAV outage degrades
> the file-upload feature (proposal attachments, ESPD documents) but does NOT constitute
> a platform outage per the Epic 4 AgenticSAI isolation precedent applied to ClamAV.
> The platform owns ClamAV restart/recovery; ClamAV scan failures during the outage window
> are captured in the decision tree below (queue vs. reject).
> See `severity-definitions.md` SLA-scope table.

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| ClamAV `clamav:3310` health-probe failures in client-api logs | `kubectl logs -n eusolicit deploy/client-api` |
| File-upload endpoint returning 503 or "scan service unavailable" | client-api → `/documents/upload` response |
| `kubectl get pods -n eusolicit` shows `clamav` pod in CrashLoopBackOff | `kubectl get pods -n eusolicit` |
| Virus-definition update failure (ClamAV freshclam timeout) | ClamAV pod logs |
| Customer reports that document uploads are failing | Support channel |

---

## Triage

1. **Check ClamAV pod status**:
   ```bash
   kubectl get pods -n eusolicit | grep clamav
   kubectl describe pod -n eusolicit <clamav-pod-name>
   ```
   - `CrashLoopBackOff` → ClamAV is crashing repeatedly → proceed to Branch A.
   - `Pending` → scheduling issue (node resource constraint) → check `kubectl describe pod`.
   - `Running` but health-probe failing → ClamAV is up but not responding on port 3310 → Branch B.

2. **Check ClamAV logs for root cause**:
   ```bash
   kubectl logs -n eusolicit deploy/clamav --previous --tail=50
   ```
   Common causes:
   - OOM kill (virus definition database too large for memory limit).
   - Freshclam download failure (network connectivity to ClamAV mirror).
   - Corrupt virus database on restart.

3. **Check the client-api health-probe flow**:
   ```bash
   kubectl exec -n eusolicit deploy/client-api -- nc -zv clamav 3310
   ```
   - Connection refused → ClamAV pod is down.
   - Connection accepted but no clamd response → clamd initialising (wait 60s for DB load).

4. **Assess business impact** — is file upload completely blocked or just slow?
   - Check client-api configuration: `CLAMAV_SCAN_TIMEOUT` setting.
   - If timeout is short (< 30s) and ClamAV is slow-starting → may auto-recover.

---

## Resolution

### Branch A — ClamAV pod in CrashLoopBackOff

1. **Restart ClamAV deployment**:
   ```bash
   kubectl rollout restart deploy/clamav -n eusolicit
   ```
   Wait for pod to become `Ready` (clamd takes 30–90s to load the virus database on startup).

2. **Monitor startup**:
   ```bash
   kubectl logs -n eusolicit deploy/clamav -f
   ```
   Expected: `"LibClamAV info: Database correctly reloaded"` → clamd ready.

3. **If restart fails due to OOM** (pod killed with OOMKilled):
   ```bash
   kubectl describe pod -n eusolicit <clamav-pod-name> | grep OOMKilled
   ```
   - Increase memory limit in the ClamAV Helm values (virus database = ~350MB; pod needs ≥512MB).
   - Temporary workaround: use `queue` mode (see Branch C) while memory limit is increased.

4. **If freshclam network failure** (cannot download virus database update):
   - Disable freshclam auto-update temporarily by setting `FRESHCLAM_ENABLED=false` env var.
   - ClamAV will use the last successfully-downloaded database.

### Branch B — ClamAV running but not responding on port 3310

1. Clamd may be initialising (loading virus database). Wait 90s and re-test:
   ```bash
   sleep 90 && kubectl exec -n eusolicit deploy/client-api -- nc -zv clamav 3310
   ```

2. If still unresponsive after 120s → force restart (Branch A).

### Branch C — Queue-vs-reject decision (sustained outage > 15 min)

**Decision tree**:
- **< 15 min outage**: queue uploaded files for deferred scanning (do not reject). Users can upload; files are scanned when ClamAV recovers.
- **15–60 min outage**: continue queuing with a user-visible "uploads are being processed" message. Do NOT reject legitimate business documents.
- **> 60 min outage**: evaluate whether to reject uploads (safer for untrusted files) or continue queuing. Consult CTO for policy decision. Log decision in `#platform-incidents`.

Configure queue vs. reject behaviour via the `CLAMAV_FALLBACK_MODE` env var in client-api:
```bash
kubectl set env deploy/client-api -n eusolicit CLAMAV_FALLBACK_MODE=queue  # or 'reject'
```
After ClamAV recovers, reset to normal scanning mode:
```bash
kubectl set env deploy/client-api -n eusolicit CLAMAV_FALLBACK_MODE=scan
```

---

## Verification

1. **ClamAV pod is `Running` and `Ready`**:
   ```bash
   kubectl get pods -n eusolicit | grep clamav
   ```

2. **Port 3310 responding**:
   ```bash
   kubectl exec -n eusolicit deploy/client-api -- nc -zv clamav 3310 && echo "ClamAV: OK"
   ```

3. **File upload endpoint functional** — test with a benign file:
   ```bash
   curl -X POST https://api.eusolicit.eu/documents/upload \
     -H "Authorization: Bearer <test-token>" \
     -F "file=@/tmp/test.pdf"
   ```
   Expected: 200 OK with scan result.

4. **client-api health probe succeeds**:
   ```bash
   kubectl exec -n eusolicit deploy/client-api -- curl -sf http://localhost:8001/health
   ```

5. **Virus database is up-to-date** (freshclam):
   ```bash
   kubectl logs -n eusolicit deploy/clamav --tail=20 | grep "freshclam"
   ```

---

## Rollback

If ClamAV restart worsened the situation (e.g., pod in CrashLoopBackOff after restart):

1. Revert to the previous ClamAV image tag if an image update was the cause:
   ```bash
   kubectl rollout undo deploy/clamav -n eusolicit
   ```

2. Enable `CLAMAV_FALLBACK_MODE=queue` to maintain upload functionality while ClamAV is recovered.

3. Escalate to a ClamAV image rebuild if the virus database is corrupt:
   ```bash
   kubectl delete pvc clamav-db-pvc -n eusolicit  # Forces fresh DB download on next start
   kubectl rollout restart deploy/clamav -n eusolicit
   ```

---

## Related

- `severity-definitions.md` — SLA-scope table (ClamAV listed as EXEMPT)
- `deploy-rollback.md` — if ClamAV image rollback is needed
- `node-drain.md` — if ClamAV pod is affected by a node drain
- architecture.md line 762 — vendor isolation precedent (applied to ClamAV)
