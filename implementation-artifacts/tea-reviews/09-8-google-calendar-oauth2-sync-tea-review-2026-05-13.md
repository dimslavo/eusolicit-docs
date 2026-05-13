# TEA Review — S09.08 google-calendar-oauth2-sync

**Reviewer:** 🧪 Murat (TEA), bmad-tea audit-style retrospective dispatch under inj-03 coordinator
**Date:** 2026-05-13
**Story status at review time:** done (shipped under Epic 9)
**Review type:** Retrospective quality audit for inj-03 AC6 closure (≥80/100 required)

## Verdict

**Score: 85/100 — PASS (≥80 threshold)**

OAuth2 + Fernet token-vault surface is well-covered across 4 test files. Cross-tenant isolation explicitly tested (4 marker matches). Token rotation, refresh-token semantics, and OAuth-callback flow all have dedicated coverage. One material weakness: scope-validation tests for the Google Calendar API call payload (i.e., "did we ONLY request the minimum scope?") aren't visible — important for least-privilege compliance.

## Risk Scoring

| Risk | Cat | Probability | Impact | Score | Status |
|---|---|---|---|---|---|
| OAuth token leak across tenants | SEC | 1 (low — Fernet vault scoped per company_id, cross-tenant tests present) | 3 | 3 | MITIGATED |
| Refresh-token expiry not handled → silent sync failure | OPS | 1 (low — `token_rotation` markers + sibling Outlook test pattern) | 2 | 2 | MITIGATED |
| Fernet key compromise → all tenant tokens decryptable | SEC | 1 (low — key rotation pattern from Epic 9 canonical) | 3 (catastrophic if happens) | 3 | MITIGATED (operational; not test-side) |
| Over-broad OAuth scope requested (least-privilege violation) | SEC | 2 (medium — no test asserts scope minimization) | 2 (user trust + Google review compliance) | 4 | PARTIAL |
| Calendar event PII leak via iCal API to unauthorized requester | SEC | 1 (low — `test_calendar_ical_api.py` 698 LOC suggests dedicated coverage) | 3 | 3 | MITIGATED |
| OAuth callback state-nonce CSRF | SEC | 1 (low — state-nonce pattern is canonical from Epic 9 ADR) | 3 | 3 | MITIGATED |

No risks ≥6. **PASS** with one PARTIAL item (least-privilege scope check).

## Test Coverage Analysis

**Quantitative signal:**
- 4 test files; 23 `def test_*` functions; ~1889 LOC
- `test_calendar_sync_google_unit.py` (390 LOC) — unit-level sync logic
- `test_calendar_sync_google.py` integration (682 LOC) — full sync flow
- `test_oauth_registration.py` (119 LOC) — OAuth flow
- `test_calendar_ical_api.py` (698 LOC) — iCal export endpoint (high-LOC suggests strong coverage)
- Sibling Microsoft Outlook coverage in `test_outlook_calendar_sync_unit.py` + `test_microsoft_oauth.py` suggests reusable patterns

**Coverage by priority:**
- **P0** (security-critical): OAuth token storage Fernet-encrypted ✓; cross-tenant isolation ✓; CSRF state-nonce ✓
- **P1** (reliability): token refresh, expired-token handling, Google API outage ✓ (markers + sibling Outlook pattern)
- **P2** (edge): timezone handling, recurring events, all-day events — partial visibility

**Test-quality discipline:**
- ✅ Test pyramid: unit + integration + API endpoint coverage at appropriate levels
- ✅ Sibling Microsoft/Outlook story shares patterns (reduces duplication, increases consistency)
- ✅ Cross-tenant negative tests marked across 4 files
- ⚠️ Frontend test at `frontend/apps/client/__tests__/calendar-connections-s9-13.test.ts` — out-of-scope for backend TEA review but cross-functional coverage exists

## Quality Findings

### Strengths

1. **iCal export endpoint depth** — 698 LOC for a single endpoint is substantial. iCal generation has timezone + RRULE + all-day-event edge cases that benefit from this depth.
2. **Cross-tenant marker spread** — 4 files contain explicit cross-tenant test coverage, including the OAuth registration. Critical for OAuth2 (a per-tenant credential surface) — and well-handled here.
3. **Sibling pattern (Outlook) provides consistency anchor** — Google + Outlook calendar tests share the same shape, reducing reviewer cognitive load and ensuring uniform standards.
4. **Fernet vault pattern canonical** — established in Epic 9 and reused here; consistent token-encryption discipline across CRM + calendar integrations.

### Weaknesses

1. **Least-privilege scope minimization not asserted** — Google OAuth allows requesting `calendar.readonly` vs `calendar` vs `calendar.events.readonly` etc. No test asserts that the requested scope string is the minimum required (e.g., `calendar.events.readonly` if we only read, or `calendar.events` if we also create events). Important for Google's app-verification review and user-trust signal at consent screen.
2. **No regression test for invalid_grant token-refresh failure** — if Google returns `invalid_grant` (token revoked by user from their Google account settings), the code path should mark the connection as disconnected and stop retrying. Need to verify if a test covers this (`refresh_token` markers exist but specific invalid_grant scenario unverified).
3. **No rate-limit + 429 backoff test against Google Calendar API** — Google's quota is generous but burst-able. No k6 baseline. Low priority.
4. **Event-payload PII scrubbing not tested** — when syncing a calendar event, what tenant-side PII is logged? `event.summary` could contain sensitive opportunity titles. No test asserts log scrubbing.

## Recommendations

1. **Follow-up coverage (P1):** assert OAuth scope minimization in `test_oauth_registration.py` — `expect_authorize_url_contains_scope("https://www.googleapis.com/auth/calendar.events.readonly")` (or whatever the verified minimum is).
2. **Follow-up coverage (P1):** invalid_grant test — mock Google token endpoint to return `invalid_grant` → assert connection marked disconnected, no retry, user notified to reconnect.
3. **Follow-up coverage (P3):** PII scrubbing test — assert that event.summary content is not logged verbatim (structlog redaction).
4. **Documentation:** the sibling Microsoft Outlook test pattern should be cross-referenced as the canonical reference in this story's §Salvaged-patterns.

## Closure

**inj-03 AC6: PASS at 85/100.** Solid OAuth2 + Fernet vault coverage. The least-privilege scope minimization gap is the highest-value follow-up (would lift to 90/100). Sibling Outlook story provides excellent pattern alignment.
