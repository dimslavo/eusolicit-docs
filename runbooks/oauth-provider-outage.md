# Runbook: OAuth Provider Outage (Google / Microsoft)

**Severity**: SEV-2 (new logins affected; active sessions unaffected during JWT grace window)
**SLA-Scope**: EXEMPT (vendor outage)
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6)

---

> **SLA-EXEMPT notice**: Google OAuth and Microsoft OAuth are upstream identity providers.
> OAuth provider outages are **explicitly excluded from the platform 99.9% SLA scope** per the
> Epic 4 AgenticSAI isolation precedent applied to OAuth providers (architecture.md line 762 pattern).
> The platform owns the fallback-to-email-password path; the OAuth provider's availability
> itself is not in the platform SLA.
> See `severity-definitions.md` SLA-scope table.

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| Google OAuth login failure rate elevated (`/auth/google/callback` returning 500/503) | client-api logs |
| Microsoft OAuth login failure rate elevated (`/auth/microsoft/callback` returning 500/503) | client-api + calendar-sync logs |
| Customer reports of "cannot log in with Google" or "cannot log in with Microsoft" | Support channel |
| Epic 9 calendar-sync degradation — Google Calendar sync failing | agenticsai-gateway / integrations-api logs |
| OAuth provider status page reporting an incident | Google: https://status.google.com / Microsoft: https://status.microsoft.com |
| JWT token refresh failures (sessions expiring during active use) | client-api logs → token refresh endpoint |

**Affected stories**: Story 2-6 (Google OAuth), Epic 9 (Microsoft OAuth + calendar-sync).

---

## Triage

1. **Check OAuth provider status pages**:
   - Google: https://status.google.com — look for "Google Authentication" or "Google Identity" incident.
   - Microsoft: https://status.azure.com — look for "Azure Active Directory" or "Microsoft 365" incident.
   - Confirmed provider incident → EXEMPT status confirmed.

2. **Identify the scope of impact**:
   - Are new logins (OAuth flow) failing, but active sessions still working?
   - Existing sessions: JWT tokens expire after 24h per architecture.md ADR-002. Active sessions within the 24h window are **unaffected** by OAuth provider outage (JWT is self-contained; no OAuth provider call needed for session refresh).
   - New logins: users who need to authenticate for the first time during the outage → must use email-password fallback.

3. **Check client-api auth-service logs**:
   ```bash
   kubectl logs -n eusolicit deploy/client-api --since=10m | grep -i "oauth\|callback\|google\|microsoft"
   ```
   - `ConnectionError` or `TimeoutError` from OAuth provider → confirmed upstream failure.
   - `InvalidToken` or `BadRequest` → may be a platform-side misconfiguration (not EXEMPT).

4. **Confirm email-password fallback is functional** (Story 2-6 fallback path):
   ```bash
   curl -X POST https://api.eusolicit.eu/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email": "<test-user@example.com>", "password": "<test-pass>"}'
   ```
   Expected: 200 OK with JWT token. If failing → separate SEV-1 (auth-service down); follow `error-budget-burn.md`.

5. **Check Epic 9 calendar-sync degradation**:
   ```bash
   kubectl logs -n eusolicit deploy/integrations-api --since=10m | grep -i "calendar\|oauth\|microsoft"
   ```
   Calendar-sync failures during Microsoft OAuth outage: expected behaviour (SLA-EXEMPT).

---

## Resolution

**Primary action: WAIT + COMMUNICATE.** OAuth provider outage is upstream; the platform cannot resolve it.

1. **Acknowledge the alert** with status note:
   > "Google/Microsoft OAuth outage confirmed at <time> per <status_url>. New OAuth logins are failing (SLA-EXEMPT per architecture.md line 762). Active sessions within 24h JWT window are UNAFFECTED. Email-password login is available as fallback. Monitoring for provider recovery."

2. **Communicate to affected users** via status page:
   - Use `status-page-comms-templates.md` SEV-2 template (do not use SEV-1 since active sessions are unaffected).
   - Message: "We are experiencing issues with Google/Microsoft OAuth login due to an upstream provider incident. Please use email-password login as an alternative. Your active sessions are unaffected."

3. **Post in Slack `#platform-incidents`**:
   > "OAuth provider outage active since <time>. Email-password login working normally. Existing sessions unaffected (JWT 24h expiry). Update expected within 1h or when provider recovers."

4. **Epic 9 calendar-sync degradation** — if Microsoft OAuth is down, calendar-sync will fail gracefully:
   - Calendar-sync is a best-effort background process; it retries automatically when OAuth recovers.
   - No manual intervention needed for calendar-sync.

5. **Once provider recovers**:
   - Test OAuth flow end-to-end: log in with Google/Microsoft.
   - Verify Epic 9 calendar-sync resumes (check integrations-api logs for successful token refresh).
   - Close the Slack incident thread.

---

## Verification

1. **OAuth provider status page shows "All Systems Operational"**.

2. **OAuth login flow functional** — test with a Google/Microsoft test account:
   ```bash
   # Initiate OAuth flow and verify redirect chain completes successfully
   curl -sf https://api.eusolicit.eu/auth/google/login
   ```

3. **client-api auth logs show no OAuth errors** for 5 minutes post-recovery:
   ```bash
   kubectl logs -n eusolicit deploy/client-api --since=5m | grep -i "oauth\|callback" | grep -v "200"
   ```

4. **Epic 9 calendar-sync resumed** — integrations-api logs show successful Microsoft token refresh.

5. **Email-password login continues to function** (should not be affected by OAuth outage, but verify):
   ```bash
   # Quick smoke test
   curl -X POST https://api.eusolicit.eu/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email": "<test-user>", "password": "<test-pass>"}'
   ```

---

## Rollback

Not applicable — OAuth provider outage recovery requires no platform-side rollback.

If any platform-side changes were made during the incident (e.g., OAuth config modified):
- Revert via `deploy-rollback.md`.
- Verify email-password login is unaffected.

---

## Related

- `severity-definitions.md` — SLA-scope table (OAuth providers listed as EXEMPT)
- `status-page-comms-templates.md` — SEV-2 status page template
- `error-budget-burn.md` — if auth failures somehow impact platform SLO (should not per carve-out)
- Story 2-6 — Google OAuth + email-password fallback implementation
- Epic 9 — Microsoft OAuth + calendar-sync
- architecture.md line 762 — vendor isolation precedent
- Google status: https://status.google.com
- Microsoft status: https://status.azure.com
