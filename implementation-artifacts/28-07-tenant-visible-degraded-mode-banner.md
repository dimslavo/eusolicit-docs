# Story 28.07: Tenant-Visible Degraded-Mode Banner

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 2
**Type:** backend + frontend
**Dependencies:** S28.05 (reconciler observability for the signal source), UX amendment §5.13
**Blocks:** Slice 5 DoD; pe-04 AC4 (chaos network-partition drill verification)
**Created:** 2026-05-15
**Source:** E28 epic §S28.07

## Story

As **Elena trying to run a qualification while SirmaAI is degraded**,
I want **a yellow banner at the top of the app explaining that AI analysis is temporarily unavailable**,
so that **I don't sit confused at a "Run analysis" button that silently does nothing**.

## Acceptance Criteria

1. **Backend signal**: on circuit-breaker open > 5 minutes against SirmaAI (existing circuit-breaker state gauge per ADR-004), publish `platform.degraded_mode` event to Redis Stream `notification.degraded_mode_events`.
2. **Endpoint** `GET /api/v1/system/status` returns `{degraded_mode: bool, since: TIMESTAMPTZ|null, feature: "sirmaai_ai_analysis"|null}`.
3. **Frontend**: top-of-page banner in `frontend/apps/client/app/(protected)/layout.tsx` polls `/api/v1/system/status` every 30s. When `degraded_mode=true`:
   - Banner appears with copy: "AI analysis temporarily unavailable. Existing analyses are unaffected; new analyses will resume automatically."
   - Color: warning-yellow background.
   - `role="status"` + `aria-live="polite"`.
   - Dismissible per-tab (state in sessionStorage, resets on next page load).
4. **Recovery**: on circuit-breaker close, after 1-min hysteresis, `degraded_mode=false`. Banner clears.
5. **UX rules per amendment §5.13**:
   - Banner does NOT block UI actions.
   - User-facing language only (no "circuit-breaker" jargon).
   - Persisted across pages while degraded.
   - Admin app does NOT show this banner (admins see SirmaAI Health screen instead).
6. **Forced-degraded test on staging**: force circuit-open → banner appears within 1min → circuit-close → banner clears within 2min.
7. **WCAG 2.1 AA**: banner contrast OK; keyboard-dismissible (Esc); doesn't trap focus.

## Dev Notes

### Pattern reuse
- Existing banner pattern (e.g. trial-banner on dashboard).
- TanStack Query polling.
- Redis Stream consumer pattern.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/services/circuit_breaker_state_watcher.py` (new — publishes degraded_mode event)
- `services/client-api/src/client_api/routers/system.py` (new — `/api/v1/system/status`)
- `frontend/apps/client/app/(protected)/layout.tsx` (extend with banner)
- `frontend/packages/ui/src/components/DegradedModeBanner.tsx` (new)
- `frontend/apps/client/messages/en.json` + `bg.json` (i18n keys for `degradedMode.*`)
- `services/client-api/tests/integration/test_system_status.py`
- `tests/e2e/degraded-mode-banner.spec.ts`

### Out of scope
- Status-page link in banner (UX amendment §8 open question — confirm domain).
- Per-feature degradation (only SirmaAI for v1; other future features could share the pattern).

## Risks

- **R1**: 30s polling overhead — minimal; alternative SSE not worth complexity for once-per-30s state.
- **R2**: Banner copy localisation — must add i18n for both en + bg.

## Testing

- E2E Playwright: forced degraded state → banner shows; recovery → banner clears.
- Accessibility: axe-core.

## See also

- Epic file §S28.07
- UX amendment §5.13
- PRD amendment NFR-26
- pe-04 chaos drill (AC4 verifies this banner)
