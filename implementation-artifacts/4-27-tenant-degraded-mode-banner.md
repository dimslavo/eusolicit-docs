# Story 4.27: Tenant Degraded-Mode Banner

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **full-stack engineer landing the tenant-visible "AI analysis temporarily unavailable" banner mandated by the SirmaAI architecture amendment (§ADR-004 *Decision* addendum 2026-05-12, line 134: "Open-circuit fallback for AI paths remains **fail-open with degraded result + tenant-visible banner** for outages >5 minutes; payment-path circuit breakers (Stripe) remain **fail-closed**") + Epic 4 amendment AC 9 ("Tenant-visible 'AI analysis temporarily unavailable' banner surfaced on circuit-breaker open >5 minutes") + §11.3 risk #12 mitigation ("SirmaAI as second critical external dependency … tenant-visible degraded-mode banner on outages >5 minutes; AI summary paths fail-open with degraded result, payment paths remain fail-closed")**,

I want **(a) a small extension to `sirmaai_gateway.services.circuit_breaker.AgentCircuit` that records `opened_at: float | None` (monotonic timestamp) whenever the circuit transitions CLOSED → OPEN or HALF_OPEN → OPEN, and clears it on the CLOSED transition — distinct from `last_failure_time` (which tracks the *most recent* failure that re-opened or refreshed the cooldown); `opened_at` is the **continuous-degradation clock** the banner key relies on, because a flapping HALF_OPEN → OPEN → HALF_OPEN → OPEN sequence is still one continuous degradation episode from the tenant's perspective; (b) a module-level helper `circuit_breaker.tenant_open_circuits(sirmaai_project_id: str, *, min_open_seconds: float) -> list[dict]` that walks the existing in-memory `_circuits` registry, filters by **circuit-key suffix** `:{sirmaai_project_id}` (the composite-key convention from S04.24: `sirmaai_run:{logical_name}:{sirmaai_project_id}` and `sirmaai_run_poll:{run_type}:{sirmaai_project_id}`), and returns the as_dict() snapshots whose `circuit_state == "open"` AND `(time.monotonic() - opened_at) >= min_open_seconds` — the **5-minute threshold** is the architecture-amendment-mandated `min_open_seconds=300.0` default; (c) a new admin/internal endpoint `GET /admin/tenant-status` on `sirmaai-gateway` accepting `sirmaai_project_id` as a required query parameter and a `threshold_seconds` optional override (default = 300), returning `{"degraded": bool, "degraded_since": ISO8601|null, "agents_open": [<circuit_key>...], "threshold_seconds": int, "sirmaai_project_id": "<pid>"}` — the endpoint is **read-only**, fail-safe (returns `{"degraded": false, ...}` if the gateway is healthy or the project_id is unknown), and adds **no DB load** (the entire computation is in-process against the `_circuits` dict); (d) a new public-facing endpoint `GET /api/v1/system/ai-status` on `client-api` that resolves the authenticated user's `company_id → sirmaai_project_id` via the existing `client.sirmaai_projects` row (using the SQLAlchemy `SirmaAIProject` model already in place from S04.21), calls the gateway endpoint with an **explicit 5 s httpx timeout** (per delivery-instructions §Security defaults — every outbound httpx call has an explicit timeout), and returns the tenant-scoped JSON body `{"ai_degraded": bool, "degraded_since": ISO8601|null, "message_key": "system.aiUnavailable"|null}` — when the tenant has no `client.sirmaai_projects` row (provisioning still pending, S04.21 partial-index state) the endpoint returns `{"ai_degraded": false, ...}` (banner never blocks a tenant whose tenancy is itself in `pending` state — the FR-45 provisioning-pending UX is owned by E24, not this story); (e) a new `DegradedAIBanner` React component at `frontend/apps/client/app/[locale]/(protected)/components/DegradedAIBanner.tsx` that mirrors the existing `TrialBanner.tsx` pattern (client component, `useTranslations`, dismissible-per-session via `sessionStorage`, but **defaults to re-show on every page load** — re-dismissal is per-tab-session, not persistent — because the tenant should be re-warned each session that AI is degraded), polled via TanStack Query v5 (`useQuery`, `queryKey: ["system","ai-status"]`, `refetchInterval: 60_000` — 60 s polling matches the 5-minute granularity without hammering the gateway), wrapped in `<QueryGuard>` per the project's frontend conventions; (f) the banner wired into the existing `(protected)/workspace/[workspaceId]/layout.tsx` directly below the existing `<TrialBanner />` slot (the "notification surface" the Epic 4 amendment line 509 refers to is this banner region) — both banners can render simultaneously; (g) new i18n strings `system.aiUnavailable.message`, `system.aiUnavailable.dismiss`, `system.aiUnavailable.learnMore` in both `messages/en.json` and `messages/bg.json` with `pnpm check:i18n` verifying parity; (h) the **cross-tenant negative test invariant** (project memory rule — non-negotiable): a unit test seeds open circuits for `sirmaai_project_id=PROJ_A` only, then asserts `tenant_open_circuits("PROJ_B", min_open_seconds=300.0) == []` AND asserts `client-api`'s `/api/v1/system/ai-status` returns `ai_degraded=false` for a user whose company maps to PROJ_B even when PROJ_A's circuits are wide open; (i) **NO database migrations** (no new tables, no new columns — the `client.sirmaai_projects` row exists since S04.21; the `_circuits` registry is in-memory); **NO new Celery tasks** (the banner is request-driven, polling-only); **NO new Prometheus counters** (the existing `sirmaai_circuit_state_total{state="open"}` Prometheus exposure from S04.22 is sufficient for alerting; the banner is the *user-facing* mitigation while ops responds to the alert); (j) flag-aware behaviour: when `SIRMAAI_GATEWAY_ENABLED=false` the gateway endpoint still functions (returns `degraded=false` because no S04.24/S04.26 circuits have been registered under the SirmaAI key convention) and client-api's `/api/v1/system/ai-status` returns a clean `ai_degraded=false` — the banner is invisible during the pre-flip rollout window, which matches the S04.20 phased-rollout discipline**,

so that **(1) the architecture amendment's "fail-open with degraded result + tenant-visible banner" invariant becomes user-facing implementation — a SirmaAI outage longer than the 5-minute SLA threshold becomes a *visible* event to every affected tenant, not a silent feature-loss that surprises them when an analysis fails; (2) the §11.3 risk #12 mitigation chain ("run-state reconciler authoritative + async-run + job-poll preserves runs + tenant-visible degraded-mode banner") completes its third pillar — S04.26 closed the reconciler-as-authoritative-truth pillar; this story closes the user-facing pillar; (3) the operator runbook gains a deterministic test for "is the user being told"; when an on-call engineer is paged on the existing `sirmaai_circuit_state_total{state="open"}` alert (Prometheus surface from S04.22), they can verify the corresponding tenant banner is rendering by hitting `GET /api/v1/system/ai-status` with a known-affected tenant's bearer token — closes the observability-vs-UX gap; (4) the multi-tenant boundary invariant (project memory rule) gets a *user-facing* enforcement point — Company A's circuits never trigger Company B's banner because the resolution path is `current_user.company_id → client.sirmaai_projects.sirmaai_project_id → circuit-key-suffix match` — the same boundary the rest of the gateway already honours; (5) the FR-45 provisioning-pending UX (owned by E24) is not muddied — a tenant whose `client.sirmaai_projects` row is `provisioning_status='pending'` sees no degraded banner (because there are no circuits to be open yet); E24 owns that surface separately; (6) the SirmaAI cutover discipline (`SIRMAAI_GATEWAY_ENABLED=false` in prod, flip later) is preserved — the banner is invisible by construction while the flag is off, no extra opt-out path needed; (7) S04.28's tier-to-rate-limit sync and S04.29's public ingress for the webhook receiver are independent and parallelisable — this story does not block them**.

## Acceptance Criteria

1. **`AgentCircuit` augmented with `opened_at` timestamp** in `services/sirmaai-gateway/src/sirmaai_gateway/services/circuit_breaker.py`:

   - Add an instance attribute `self.opened_at: float | None = None` initialised in `__init__`.
   - In `_open_circuit()`: **before** updating `self.state`, capture the current value of `self.state`. If the previous state was **NOT** `CircuitState.OPEN` (i.e., transitioning CLOSED → OPEN or HALF_OPEN → OPEN), set `self.opened_at = time.monotonic()`. If the previous state was already `OPEN` (defensive — re-entrant open) DO NOT overwrite `self.opened_at` — the continuous-degradation clock must remain the *first* open in the streak.
   - In `_close_circuit()`: set `self.opened_at = None` (clears the clock when the circuit closes successfully).
   - **Critical invariant**: a HALF_OPEN → OPEN transition (the flapping case where the cooldown elapsed, a test call was permitted, the test call failed, and the circuit re-opens) MUST preserve `opened_at` from the original CLOSED → OPEN transition. This is the architecture-amendment intent: "outages >5 minutes" measures the **continuous degraded episode** from the tenant's perspective, not the most recent re-open inside the streak. The state machine's HALF_OPEN transition is invisible to the tenant; only the next *successful* call (HALF_OPEN → CLOSED) ends the degraded episode.
   - Update `AgentCircuit.as_dict()` to include `"opened_at"` as an ISO 8601 wall-clock string (computed the same way `last_failure_time` is — monotonic offset to wall-clock conversion using `datetime.now(tz=datetime.UTC)`) or `null` when not open.
   - **Backward compatibility**: the existing `last_failure_time` field is **NOT** removed or renamed — every consumer of `as_dict()` (including `GET /admin/circuits`, S04.06 + S04.22 + S04.26 dashboards) continues to see it. `opened_at` is an **additive** field.

2. **New module-level helper `tenant_open_circuits()` in `circuit_breaker.py`**:

   ```python
   def tenant_open_circuits(
       sirmaai_project_id: str,
       *,
       min_open_seconds: float = 300.0,
   ) -> list[dict]:
       ...
   ```

   - Walks the module-level `_circuits` dict (same one used by `get_all_circuits()`).
   - Filters by **circuit-key suffix match**: `circuit.agent_name.endswith(f":{sirmaai_project_id}")`. This catches both S04.24 namespaces (`sirmaai_run:{logical_name}:{pid}` and `sirmaai_run_poll:{run_type}:{pid}`) and is robust against the pre-amendment registry-based agent names (which never end in `:{pid}` and therefore never match — by design, pre-amendment circuits are not banner-eligible).
   - Of the matching circuits, returns only those where `circuit.state == CircuitState.OPEN` AND `circuit.opened_at is not None` AND `(time.monotonic() - circuit.opened_at) >= min_open_seconds`.
   - Returns the `as_dict()` snapshots of the matching circuits, sorted alphabetically by `agent_name` (deterministic for test assertions).
   - Returns `[]` (empty list, NOT `None`) when no circuits match — callers can `len(result) > 0` to compute `degraded`.
   - **Empty `sirmaai_project_id`**: returns `[]` immediately (defence-in-depth — empty suffix would match *every* circuit, which is wrong). Type-check this with an explicit `if not sirmaai_project_id: return []`.
   - **Test fixture access**: a sibling test helper `_set_opened_at_for_test(circuit, opened_at: float)` is **NOT** added — tests construct `AgentCircuit` instances directly and set `opened_at` on the instance (it's a public attribute by Python convention; `_circuits` is private but tests already access it via `reset_circuits()` + manual seeding patterns from S04.06 tests).

3. **New router `tenant_status_router` in `sirmaai_gateway.routers.admin`** (extend the existing `routers/admin.py` — do NOT create a new module):

   - Endpoint: `GET /admin/tenant-status` (mounted under the existing `/admin` prefix via the existing `app.include_router(admin_router.router, prefix="/admin")` line in `main.py` — NO new include call).
   - Query parameters:
     - `sirmaai_project_id: str` (required, non-empty — FastAPI's `Query(..., min_length=1)`).
     - `threshold_seconds: int` (optional, default = `300`, `Query(300, ge=1, le=3600)` — clamped 1 s..1 h to prevent abuse / accidental DoS-via-tiny-threshold).
   - Response model (Pydantic, frozen): `TenantStatusResponse` with fields
     - `degraded: bool`
     - `degraded_since: datetime | None`  — ISO 8601 UTC; the earliest `opened_at` across the matching open circuits (i.e., the start of the *first* circuit's degraded episode — the tenant has been degraded *since at least* this moment).
     - `agents_open: list[str]` — the circuit_key names of all matching open circuits (e.g., `["sirmaai_run:proposal_drafter:PROJ_A", "sirmaai_run_poll:agent:PROJ_A"]`).
     - `threshold_seconds: int` — echo back the threshold used (default 300 or the override).
     - `sirmaai_project_id: str` — echo back the input project id.
   - **Fail-safe**: NO 4xx for "no circuits found" — an unknown project_id simply returns `{"degraded": false, "agents_open": [], ...}` with HTTP 200. This is intentional — the endpoint is a *status* probe, not a *validity* probe. Returning 404 for unknown project_ids would leak whether a tenant has any agent activity (information disclosure).
   - **No authentication required** on this endpoint *yet* — it sits under `/admin/*` which is **already ClusterIP-only** per Epic 4 amendment AC 14 ("Public ingress configured for webhook receiver only … all other gateway paths remain ClusterIP-only"). S04.29 wires the public webhook ingress; this endpoint is unaffected. No auth annotation is added in *this* story — the network boundary is the auth boundary, identical to `GET /admin/circuits` and `GET /admin/executions` shipped in S04.06 / S04.08.
   - **No DB access** in this endpoint — the entire computation is against the in-process `_circuits` dict.

4. **New endpoint `GET /api/v1/system/ai-status` in `client-api`**:

   - File: create new router `services/client-api/src/client_api/api/v1/system_status.py` (mirroring the existing `api/v1/` router-per-resource convention; see `alert_preferences.py`, `analytics_usage.py` for the pattern).
   - Wire into the FastAPI app via the existing `api/v1/__init__.py` `include_router` aggregator.
   - Dependencies: `current_user: Annotated[CurrentUser, Depends(get_current_user)]` (imported from `client_api.core.security`). Requires authentication; does NOT require any specific role — every authenticated user gets to see the banner.
   - **`User.is_active` is enforced by `get_current_user`** (delivery-instructions §Security defaults — already wired into the security dependency).
   - Endpoint: `@router.get("/ai-status", response_model=AiStatusResponse)`.
   - Response model `AiStatusResponse`:
     - `ai_degraded: bool`
     - `degraded_since: datetime | None`
     - `message_key: Literal["system.aiUnavailable"] | None` — when `ai_degraded=true`, set to the literal string `"system.aiUnavailable"` (the frontend uses this as the i18n lookup key). When `false`, set to `None`. The translation lives **client-side only** — the API never returns translated user-facing strings (consistent with the rest of `client-api`).
   - Resolution flow:
     1. Query `client.sirmaai_projects` via the existing `SirmaAIProject` ORM model: `select(SirmaAIProject).where(SirmaAIProject.company_id == current_user.company_id)`.
     2. If no row found OR row's `provisioning_status != 'provisioned'`: return `AiStatusResponse(ai_degraded=False, degraded_since=None, message_key=None)` immediately — no gateway call (saves a hop for unprovisioned tenants).
     3. Otherwise: extract `sirmaai_project_id = row.sirmaai_project_id` and `await self._gateway_client.fetch_tenant_status(sirmaai_project_id)` (see AC 5 for the new client method).
     4. Translate the gateway response to `AiStatusResponse`. If `degraded=true`, `message_key="system.aiUnavailable"`. If `degraded=false`, `message_key=None`.
   - **Error handling**: if the gateway call raises `AiGatewayTimeoutError` or `AiGatewayUnavailableError` (S11.03 contract): **fail-closed** — return `AiStatusResponse(ai_degraded=False, ...)` with `degraded_since=None`. Log at WARN with `error_type=type(exc).__name__`. **DO NOT** return 5xx — a banner-status hiccup must NEVER break the protected app shell on every page load. Comment: `# Fail-closed: a 5xx from the gateway must not bubble up; the banner is best-effort UX.`
   - **Explicit timeout**: the AI gateway client method `fetch_tenant_status` uses a **5 s** httpx timeout (configured per call, NOT inherited from `aigw_timeout_seconds` default of 30 — banner status is interactive UX, not agent execution; a slow gateway status should not lengthen page-load critical path).
   - **Cross-tenant guard at the API layer**: the resolution path through `current_user.company_id → client.sirmaai_projects` row → that row's `sirmaai_project_id` *structurally* enforces the boundary. The negative test asserts this (AC 9).

5. **New method `fetch_tenant_status` on `AiGatewayClient`** in `client_api.services.ai_gateway_client`:

   - Signature:
     ```python
     async def fetch_tenant_status(
         self, sirmaai_project_id: str, *, threshold_seconds: int = 300
     ) -> dict:
     ```
   - Builds URL `{base_url}/admin/tenant-status` with query params `sirmaai_project_id` and `threshold_seconds`.
   - `httpx.AsyncClient(timeout=5.0)` — explicit, NOT inherited from `self._timeout` (which is for agent execution).
   - On HTTP 200 returns `response.json()`.
   - On `httpx.TimeoutException` → raise `AiGatewayTimeoutError` (existing exception). The router catches it per AC 4 error-handling and fails-closed.
   - On `httpx.ConnectError` / `httpx.RemoteProtocolError` → raise `AiGatewayUnavailableError(0, str(exc))`.
   - On HTTP 5xx → raise `AiGatewayUnavailableError(response.status_code, response.text[:500])`. **NO retry** (the existing retry on the `run_agent` path is intentional for agent runs; banner status is one-shot — a retry doubles the latency budget for a status probe, which is the wrong trade).
   - On HTTP 4xx → raise `AiGatewayUnavailableError(response.status_code, ...)` — the router still fails-closed.
   - **DO NOT log the response body at any level** — `aigw_base_url` returns may surface internal circuit keys (which embed `sirmaai_project_id`) — keep the log surface to URL + status code + `error_type`. Bearer is not on this path (the admin endpoint requires no auth — see AC 3 — but if S04.xx adds auth later, the secret-redaction discipline must already be in place).

6. **`DegradedAIBanner` React component** at `frontend/apps/client/app/[locale]/(protected)/components/DegradedAIBanner.tsx`:

   - Mirror the existing `TrialBanner.tsx` pattern (same file): `"use client"`, `useTranslations("system")`, `useState` for `dismissed`, `useEffect` for `mounted` (avoid SSR mismatch), `sessionStorage` for dismissal (key: `eusolicit-degraded-banner-dismissed`).
   - Data fetching: TanStack Query v5 `useQuery({ queryKey: ["system","ai-status"], queryFn: getAiStatus, refetchInterval: 60_000, staleTime: 30_000, retry: 1 })` — **`refetchInterval: 60_000`** is the heartbeat the banner uses to clear itself when the outage recovers (the gateway flips `degraded` back to `false` on the first successful call after HALF_OPEN → CLOSED).
   - **Render only when**: `mounted && !dismissed && data?.ai_degraded === true`. No loading state, no error state (consistent with TrialBanner — best-effort UX).
   - **Visual**: red/amber palette (Tailwind `bg-red-50 border-b border-red-200`, `text-red-800`) — DISTINCT from TrialBanner's amber (`bg-amber-50`) so the two banners are visually distinguishable when both render simultaneously. The red palette signals *system degradation* vs. amber's *upcoming action*.
   - **Copy** (i18n key `system.aiUnavailable.message`): "AI analysis is temporarily unavailable. Background runs continue; new analyses may be delayed." — phrasing comes directly from the architecture amendment line 54 ("graceful-degradation UX banner ('AI analysis temporarily unavailable')"). The Bulgarian translation must be authored simultaneously (AC 8).
   - **Dismiss button**: per-session-only dismiss (sessionStorage). On dismiss: `sessionStorage.setItem("eusolicit-degraded-banner-dismissed", "true")` and `setDismissed(true)`. **DO NOT** persist to localStorage — across sessions the user must be re-warned.
   - **Accessibility**: `role="banner"`, `aria-live="polite"` (announce on first render but not on every poll), `data-testid="degraded-ai-banner"`, `data-testid="degraded-ai-banner-close"` on the dismiss button. The dismiss button is a native `<button>` with `aria-label={t("dismiss")}`.
   - **No `<Link>` or CTA** — unlike TrialBanner there is no upgrade path; the degradation is system-side and resolves on its own. (A future enhancement could link to a status page; out of scope here.)

7. **Banner integration in `WorkspaceLayout`** (`frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/layout.tsx`):

   - Import `DegradedAIBanner` next to the existing `import { TrialBanner } …` line.
   - Inside the `topbar` slot, render `<DegradedAIBanner />` **directly below** the existing `{subscription?.status === "trialing" && subscription.trial_end && (<TrialBanner ... />)}` conditional. Render order is `TopBar → TrialBanner (if trialing) → DegradedAIBanner (if degraded)`. Both can render simultaneously.
   - The `DegradedAIBanner` component does its own query; the layout passes no props.
   - **Critical**: do NOT short-circuit `<DegradedAIBanner />` on `subscription?.status === "trialing"` — a trialing tenant is just as entitled to know AI is degraded.

8. **i18n strings** in `frontend/apps/client/messages/en.json` and `frontend/apps/client/messages/bg.json`:

   - Add a new top-level section `system` (if not present) with keys:
     - `system.aiUnavailable.message` — EN: `"AI analysis is temporarily unavailable. Background runs continue; new analyses may be delayed."` / BG: `"AI анализът временно не е достъпен. Изпълняваните задачи продължават; нови анализи може да се забавят."`
     - `system.aiUnavailable.dismiss` — EN: `"Dismiss"` / BG: `"Отхвърли"`
   - **`pnpm check:i18n` MUST pass** — both locale files have identical key sets. Run this as part of DoD.
   - **NO `learnMore` key in this story** — no link target exists yet. If one is added later (status page) it lives in its own story.

9. **Cross-tenant negative tests** (project memory rule, MANDATORY):

   - **Unit test** in `services/sirmaai-gateway/tests/unit/test_tenant_open_circuits.py`:
     - Seed circuits with keys `sirmaai_run:proposal_drafter:PROJ_A` and `sirmaai_run_poll:agent:PROJ_A` (both OPEN, `opened_at` 400 s ago — past threshold).
     - Seed circuits with keys `sirmaai_run:proposal_drafter:PROJ_B` and `sirmaai_run_poll:agent:PROJ_B` (both CLOSED).
     - Assert `tenant_open_circuits("PROJ_A", min_open_seconds=300.0)` returns 2 entries.
     - Assert `tenant_open_circuits("PROJ_B", min_open_seconds=300.0) == []`.
     - Assert `tenant_open_circuits("", min_open_seconds=300.0) == []` (empty project_id guard).
     - Assert `tenant_open_circuits("UNKNOWN_PROJECT", min_open_seconds=300.0) == []`.
   - **Integration test** in `services/client-api/tests/api/test_ai_status_endpoint.py`:
     - Provision two companies (A and B) each with a `client.sirmaai_projects` row mapping to distinct `sirmaai_project_id` values (use the existing test factories pattern; if no `SirmaAIProjectFactory` exists, create one in the test file as a `pytest.fixture` — explicit, scoped to this test).
     - Mock the AI gateway response via `respx` such that `GET /admin/tenant-status?sirmaai_project_id=PROJ_A&...` returns `{"degraded": true, "degraded_since": "2026-05-14T10:00:00Z", "agents_open": ["sirmaai_run:proposal_drafter:PROJ_A"], "threshold_seconds": 300, "sirmaai_project_id": "PROJ_A"}` AND `GET /admin/tenant-status?sirmaai_project_id=PROJ_B&...` returns `{"degraded": false, "agents_open": [], ...}`.
     - Call `GET /api/v1/system/ai-status` as user A → assert `ai_degraded=true`.
     - Call `GET /api/v1/system/ai-status` as user B → assert `ai_degraded=false`.
     - Assert `respx` recorded exactly TWO outbound calls (one per company) with the correct `sirmaai_project_id` query param — proves the resolution path is per-tenant, not shared.

10. **Fail-closed integration test** (`services/client-api/tests/api/test_ai_status_endpoint.py`):

    - Mock the gateway to raise `httpx.TimeoutException` on every call.
    - Call `GET /api/v1/system/ai-status` as an authenticated, provisioned user → assert HTTP 200 with `ai_degraded=false`, `degraded_since=null`, `message_key=null`.
    - Capture logs via `structlog.testing.capture_logs()`; assert exactly ONE WARN log with `event="ai_gateway.timeout"` (or equivalent) and `error_type="AiGatewayTimeoutError"`.
    - **Assert no `error=str(exc)` field** in the log record (delivery-instructions §Code & logging conventions — already enforced project-wide).

11. **HALF_OPEN flap continuity test** (`services/sirmaai-gateway/tests/unit/test_circuit_breaker_opened_at.py`):

    - Use `freezegun` (or `time.monotonic` patching — pattern from S04.06 tests) to control the clock.
    - Trip a circuit (`for _ in range(5): await circuit.call(failing_coro)` ); record `opened_at_first = circuit.opened_at`.
    - Advance clock past `cooldown` (30 s default). Call once with a failing coro → circuit transitions HALF_OPEN → OPEN. Assert `circuit.opened_at == opened_at_first` (UNCHANGED — the streak continues).
    - Advance clock another `cooldown`. Call once with a successful coro → HALF_OPEN → CLOSED. Assert `circuit.opened_at is None` (cleared on close).
    - Trip again. Assert `opened_at` is now a *new* value (the post-recovery streak is a fresh episode).
    - **5-minute threshold test**: trip a circuit, advance clock 299 s, assert `tenant_open_circuits("PROJ_X", min_open_seconds=300.0) == []`. Advance 2 more seconds (301 s total), assert returns 1 entry.

12. **Existing-circuit-test regression guard**:

    - The S04.06 test suite at `services/sirmaai-gateway/tests/unit/test_circuit_breaker.py` (or equivalent — check the actual filename via `find tests/ -name "test_circuit*"`) MUST continue to pass without modification. The `opened_at` addition is **additive only** — no existing behaviour changes.
    - The S04.06 `GET /admin/circuits` integration test MUST continue to pass; the addition of `opened_at` in the response payload is a non-breaking superset (existing assertions on `circuit_state`, `failure_count`, `last_failure_time` are unaffected).

13. **Frontend banner unit test** at `frontend/apps/client/__tests__/degraded-ai-banner.test.tsx`:

    - Use the existing React Testing Library + `@tanstack/react-query` test setup pattern (see `subscription-management-s8-13.test.ts` for the QueryClientProvider wrapper).
    - Mock `getAiStatus` to return `{ ai_degraded: true, degraded_since: "2026-05-14T10:00:00Z", message_key: "system.aiUnavailable" }`. Assert banner renders with `data-testid="degraded-ai-banner"` and the EN message text.
    - Mock `getAiStatus` to return `{ ai_degraded: false, ... }`. Assert banner does NOT render.
    - Render with `ai_degraded=true`, click `data-testid="degraded-ai-banner-close"`. Assert banner is removed from the DOM. Re-render the same component (simulating a re-mount within the same session) — assert banner does NOT re-appear (sessionStorage flag respected).

14. **DoD checklist** (delivery-instructions §Definition of done — eusolicit commands):

    - `make lint` — green on all touched Python files.
    - `make type-check` — green.
    - `make test-service SVC=sirmaai-gateway` — green; new unit tests for `tenant_open_circuits` + `opened_at` semantics + the `/admin/tenant-status` integration test all pass.
    - `make test-service SVC=client-api` — green; new `test_ai_status_endpoint.py` integration tests pass.
    - `make test-integration` — green; verifies the cross-service contract via `respx`-mocked gateway.
    - `make coverage` — line coverage stays ≥ 80% on `sirmaai_gateway/services/circuit_breaker.py`, `sirmaai_gateway/routers/admin.py`, `client_api/api/v1/system_status.py`, `client_api/services/ai_gateway_client.py`.
    - `cd frontend && pnpm lint && pnpm type-check && pnpm check:i18n` — all green; i18n parity verified between `en.json` and `bg.json`.
    - `cd frontend && pnpm test --filter=client -- __tests__/degraded-ai-banner.test.tsx` — banner unit test green.
    - **E2E NOT required** — the banner is a thin component reading a thin endpoint; E2E coverage is deferred to the next layout-touching story. Document this as an intentional carry-forward in the Completion Notes.

## Tasks / Subtasks

- [x] **Backend: `AgentCircuit.opened_at`** (AC 1, AC 11)
  - [x] Add `opened_at: float | None = None` instance attribute in `__init__`.
  - [x] Update `_open_circuit()` to set `opened_at` only when the previous state was NOT already OPEN.
  - [x] Update `_close_circuit()` to clear `opened_at`.
  - [x] Extend `as_dict()` to include `"opened_at"` as ISO 8601 string (mirror `last_failure_time` conversion).
  - [x] Unit tests: HALF_OPEN flap continuity; close clears the clock; re-open after close is a new episode.

- [x] **Backend: `tenant_open_circuits()` helper** (AC 2, AC 9 unit)
  - [x] Implement module-level helper with suffix-match filter, threshold filter, sorted output.
  - [x] Empty `sirmaai_project_id` guard at the top.
  - [x] Unit tests: PROJ_A vs PROJ_B isolation; empty id returns []; unknown id returns [].

- [x] **Backend: `/admin/tenant-status` endpoint** (AC 3)
  - [x] Add `TenantStatusResponse` Pydantic model in `routers/admin.py`.
  - [x] Implement the FastAPI endpoint with the query params, the threshold clamp, and the fail-safe empty-result behaviour.
  - [x] Compute `degraded_since` as the *earliest* `opened_at` across the matching circuits.
  - [x] Integration test that hits the endpoint via `httpx.AsyncClient` against a test-app instance with seeded circuits.

- [x] **Client API: `GET /api/v1/system/ai-status` endpoint** (AC 4)
  - [x] Create `client_api/api/v1/system_status.py` with the router, the `AiStatusResponse` Pydantic model, the `get_current_user` dependency, and the resolution flow.
  - [x] Wire into `client_api/main.py` (direct include per project convention, not via `__init__.py` aggregator).
  - [x] Handle the no-row and `provisioning_status != 'provisioned'` short-circuit cases.
  - [x] Fail-closed try/except around the gateway call with WARN log + `error_type` only.

- [x] **Client API: `AiGatewayClient.fetch_tenant_status()`** (AC 5)
  - [x] Add the method to `client_api/services/ai_gateway_client.py` with explicit 5 s timeout.
  - [x] No retry, no body logging.
  - [x] Integration tests with `respx` for: 200 happy path, timeout, gateway unavailable.

- [x] **Frontend: `DegradedAIBanner` component** (AC 6)
  - [x] Create `frontend/apps/client/app/[locale]/(protected)/components/DegradedAIBanner.tsx` mirroring `TrialBanner.tsx`.
  - [x] Add `getAiStatus()` client method to `frontend/apps/client/lib/api/system.ts` (new file).
  - [x] Add a typed response interface for `AiStatusResponse`.
  - [x] `useQuery` with `refetchInterval: 60_000` and the dismiss-via-sessionStorage pattern.

- [x] **Frontend: layout integration** (AC 7)
  - [x] Import + render `<DegradedAIBanner />` in `workspace/[workspaceId]/layout.tsx` below the TrialBanner slot. No props.

- [x] **Frontend: i18n strings** (AC 8)
  - [x] Add `system.aiUnavailable.message` and `system.aiUnavailable.dismiss` to `en.json` and `bg.json`.
  - [x] `pnpm lint && pnpm type-check` green (i18n scripts not present in root scripts).

- [x] **Cross-tenant + fail-closed tests** (AC 9, AC 10)
  - [x] `services/sirmaai-gateway/tests/unit/test_tenant_open_circuits.py` — cross-tenant isolation unit test.
  - [x] `services/client-api/tests/api/test_ai_status_endpoint.py` — cross-tenant integration test + fail-closed timeout test.

- [x] **Frontend banner unit test** (AC 13)
  - [x] `frontend/apps/client/__tests__/degraded-ai-banner.test.tsx` — renders / not-renders / dismiss-persists tests (4 tests passing).

- [x] **DoD verification** (AC 14)
  - [x] Run all the Make targets listed; capture output in the Dev Agent Record.

## Dev Notes

### Architecture & invariants you MUST honor

- **§ADR-004 addendum (line 134)** — "Open-circuit fallback for AI paths remains fail-open with degraded result + tenant-visible banner for outages >5 minutes". This story is the second half of "fail-open with degraded result + tenant-visible banner" — the *banner*. The "fail-open with degraded result" half is the responsibility of every consumer of the AI gateway (e.g., `client_api.services.ai_gateway_client.AiGatewayClient.run_agent` already catches errors and returns a degraded result per the S11.03 contract). Do NOT change consumer behaviour in this story.
- **Epic 4 amendment AC 9** — "Tenant-visible 'AI analysis temporarily unavailable' banner surfaced on circuit-breaker open >5 minutes". The 5-minute threshold is **not** SLA-tunable in this story — it's hard-coded as the default in the gateway endpoint and configurable per call. Production tuning is a follow-up runbook concern, not a story setting.
- **§11.3 risk #12 mitigation chain** — the *three* pillars: (1) reconciler-as-authoritative-truth (S04.26 ✓ landed); (2) async-run + job-poll preserves runs (S04.24 ✓); (3) tenant-visible banner (this story). All three must be in place for the §11.3 risk #12 mitigation to read "complete".
- **Project memory rule — cross-tenant negative test** is non-negotiable. The unit test for `tenant_open_circuits` + the integration test for `/api/v1/system/ai-status` are both required.

### Reusable code paths from prior stories — DO NOT reinvent

- `AgentCircuit` (S04.06) — extend in place; do NOT subclass, do NOT create a new circuit type.
- `circuit_breaker._circuits` module-level registry (S04.06) — re-use for `tenant_open_circuits()`; do NOT introduce a parallel registry.
- `circuit_breaker.get_all_circuits()` (S04.06) — pattern to mirror for `tenant_open_circuits()` (sort by name, return list of dicts).
- `routers/admin.py` (S04.06, S04.08, S04.22) — extend with the new endpoint; do NOT create a new module.
- `client.sirmaai_projects` ORM model `SirmaAIProject` (S04.21) — re-use; do NOT create a new accessor model.
- `client_api.services.ai_gateway_client.AiGatewayClient` (S11.03) — extend with `fetch_tenant_status`; do NOT create a new client class.
- `client_api.core.security.get_current_user` (E01) — re-use; do NOT bypass the auth dependency.
- `frontend/apps/client/app/[locale]/(protected)/components/TrialBanner.tsx` — pattern to mirror for `DegradedAIBanner` (dismiss-via-sessionStorage, mounted-state guard, `useTranslations` usage).
- TanStack Query v5 + `<QueryGuard>` — project frontend convention; do NOT introduce `SWR` or any second data-fetching library.

### Files this story touches

| File | Action | Why |
|---|---|---|
| `services/sirmaai-gateway/src/sirmaai_gateway/services/circuit_breaker.py` | UPDATE | Add `opened_at` field + `tenant_open_circuits` helper. |
| `services/sirmaai-gateway/src/sirmaai_gateway/routers/admin.py` | UPDATE | Add `GET /admin/tenant-status` endpoint + `TenantStatusResponse` model. |
| `services/sirmaai-gateway/tests/unit/test_circuit_breaker_opened_at.py` | NEW | `opened_at` semantics + HALF_OPEN flap continuity tests. |
| `services/sirmaai-gateway/tests/unit/test_tenant_open_circuits.py` | NEW | Helper-level cross-tenant isolation test. |
| `services/sirmaai-gateway/tests/integration/test_admin_tenant_status.py` | NEW | `/admin/tenant-status` integration test against a test-app. |
| `services/client-api/src/client_api/api/v1/system_status.py` | NEW | `GET /api/v1/system/ai-status` router + response model. |
| `services/client-api/src/client_api/api/v1/__init__.py` | UPDATE | Wire the new router into the aggregator. |
| `services/client-api/src/client_api/services/ai_gateway_client.py` | UPDATE | Add `fetch_tenant_status` method. |
| `services/client-api/tests/api/test_ai_status_endpoint.py` | NEW | Cross-tenant integration + fail-closed timeout tests. |
| `frontend/apps/client/app/[locale]/(protected)/components/DegradedAIBanner.tsx` | NEW | Banner component (mirror TrialBanner). |
| `frontend/apps/client/lib/api/system.ts` | NEW (or extend existing) | `getAiStatus()` typed fetch helper. |
| `frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/layout.tsx` | UPDATE | Import + render `<DegradedAIBanner />`. |
| `frontend/apps/client/messages/en.json` | UPDATE | `system.aiUnavailable.{message,dismiss}`. |
| `frontend/apps/client/messages/bg.json` | UPDATE | Bulgarian translations of same keys (i18n parity gate). |
| `frontend/apps/client/__tests__/degraded-ai-banner.test.tsx` | NEW | Component render / dismiss / re-mount tests. |

### Test design notes (epic-04 test-design priority framework, applied to S04.27)

The epic-level test design (`eusolicit-docs/test-artifacts/test-design-epic-04.md`) predates the SirmaAI amendment and does NOT enumerate S04.27 scenarios. Apply the P0/P1/P2 framework as follows for this story:

- **P0** (critical, must pass in PR CI):
  - `tenant_open_circuits` cross-tenant isolation (AC 9 unit).
  - HALF_OPEN flap continuity — `opened_at` does NOT reset across re-opens within a streak (AC 11).
  - `/api/v1/system/ai-status` cross-tenant integration (AC 9 integration) — proves the resolution path is per-tenant.
  - Fail-closed on gateway timeout (AC 10) — proves the banner can never break the app shell.

- **P1** (high — should pass in PR CI):
  - 5-minute threshold boundary: 299 s → `degraded=false`; 301 s → `degraded=true`.
  - `provisioning_status='pending'` short-circuit returns `degraded=false` without a gateway call.
  - Banner renders + dismiss + sessionStorage persistence.
  - i18n parity gate (`pnpm check:i18n`).

- **P2** (medium — nice to have):
  - `threshold_seconds` query param clamp (negative / 0 / > 3600).
  - `agents_open` list serialisation when 0 / 1 / N circuits match.
  - Banner does NOT render when `data?.ai_degraded === false` after a clear (recovery path verified at the component layer).

### Anti-patterns to avoid (S04.21 + S04.22 + S04.23 + S04.24 + S04.25 + S04.26 review lessons applied)

- **NEVER** add an HTTP retry on the `fetch_tenant_status` path — banner status is one-shot UX. Retries double the latency budget for no correctness gain. (S04.23 review M5: "Story creep" anti-pattern; retries belong on the agent-run path, not on status probes.)
- **NEVER** log `error=str(exc)` — only `error_type=type(exc).__name__`. SirmaAI 5xx bodies can echo bearer tokens through error chains (S04.22 M9 lesson). The banner path doesn't carry a bearer, but the discipline is project-wide.
- **NEVER** reach into `request.app.state` from inside `circuit_breaker.tenant_open_circuits()` — the helper operates on the module-level `_circuits` dict, independent of FastAPI lifecycle. (Tests construct `AgentCircuit` instances directly; production wires them via `init_circuits()` at startup. The module-level registry IS the shared state.)
- **NEVER** auto-converge a circuit to CLOSED from the helper — `tenant_open_circuits` is **read-only** with no side effects. Convergence to CLOSED happens only inside `AgentCircuit.call()` on a successful HALF_OPEN test, identical to the S04.06 contract.
- **NEVER** translate user-facing strings server-side — the `message_key` field carries the i18n key only; the banner component does the translation. (Mirror of the project convention seen in `client-api` error responses: `{"error": "AGENT_UNAVAILABLE"}` not `{"error": "AI агентът не е достъпен"}`.)
- **NEVER** persist the dismissal to `localStorage` — across sessions the user must be re-warned. SessionStorage is the correct boundary; the TrialBanner uses this exact pattern.
- **NEVER** add a new top-level Zustand store for the banner — the dismissal is component-local (sessionStorage) and the data is TanStack Query (server state). Adding a Zustand store would introduce a parallel state surface for no benefit and would collide with the project's namespaced persist key convention (`eusolicit-client-auth-store`).
- **NEVER** introduce `axios` or `swr` — the project uses TanStack Query v5 + `fetch` / `httpx`-equivalent. (Delivery-instructions §Frontend conventions.)
- **NEVER** add a database migration — `client.sirmaai_projects` already has every column needed; the `_circuits` registry is in-memory by S04.06 design (per-instance state, multi-replica share is a pre-scale backlog item per E04-R-005, not in scope here).

### Risks & call-outs (per migration discipline)

- **No schema migration** — confirmed. (Delivery-instructions §Migration discipline requires explicit callout; this story has none.)
- **Multi-replica caveat**: the `_circuits` registry is per-process. In a future multi-replica gateway deployment, two replicas can disagree about whether a tenant is degraded (Replica A sees a 5+ minute open circuit; Replica B sees it as CLOSED because it never hit the failure threshold there). The banner will *flicker* per replica routing. Mitigation: deferred to the same Redis-backed circuit state migration that E04-R-005 (S04.06 test-design) already flags; out of scope here. Document this in Completion Notes for E28's hardening backlog.
- **5-minute clock skew**: `time.monotonic()` is process-local. Two replicas computing `degraded_since` independently can disagree by their respective process start times. The integration test asserts a *single* replica's view; the multi-replica skew is the same pre-scale concern as above.
- **N+1-tenant fanout**: the gateway endpoint walks the entire `_circuits` dict and filters by suffix. With 29 logical agents × N tenants, the dict size is O(N) and the suffix filter is O(N) per request. At N=10,000 tenants the dict is 290,000 entries; a status probe is still <1 ms in practice. Document the projection in Completion Notes; not a story concern.

### Composite circuit-key namespace reference (S04.24 carry-forward)

The gateway has TWO active circuit-key namespaces that the suffix match must catch:

| Namespace | Format | Source |
|---|---|---|
| Submit-run circuit | `sirmaai_run:{logical_name}:{sirmaai_project_id}` | `AsyncRunOrchestrator.submit` (S04.24, `async_run_orchestrator.py:362`) |
| Poll-status circuit | `sirmaai_run_poll:{run_type}:{sirmaai_project_id}` | `AsyncRunOrchestrator.get_status` (S04.24, `async_run_orchestrator.py:492`) and reconciler (S04.26 `_reconcile_one_row`) |

Both end in `:{sirmaai_project_id}`. The suffix match `circuit.agent_name.endswith(f":{sirmaai_project_id}")` catches both — and is robust against any future S04.xx namespace that follows the trailing-`:project_id` convention. Pre-amendment registry-based circuits (logical names like `executive-summary` from S04.06) do NOT end in `:{pid}` and therefore do NOT match — by design, those are not banner-eligible because they pre-date the per-tenant resolution.

### Latest tech notes

- **FastAPI 0.110+**: `Query(..., min_length=1)` is the supported way to require a non-empty query string. Avoid Pydantic v1 patterns.
- **TanStack Query v5**: `refetchInterval` accepts a number (ms) or a function. We pass a constant `60_000`. `staleTime: 30_000` means within 30 s of a successful fetch the cached value is reused; combined with `refetchInterval` the effective behaviour is "poll every 60 s, never use a stale value older than 30 s on remount". This avoids the foot-gun where remounting components hammer the endpoint.
- **structlog 24.x**: `structlog.testing.capture_logs()` is the project's canonical log-leak audit fixture; tests use it to assert log shape (mirror of S04.24 AC 14 (j) and S04.26 AC 11 (r) patterns).
- **respx 0.21+**: `respx.mock(base_url=settings.aigw_base_url)` is the project's outbound-call mocking surface; do NOT introduce `requests_mock` or `httpretty`.

### Known scope gaps for follow-up stories

- **Admin "force-recover" endpoint**: a manual `POST /admin/circuits/{circuit_key}/close` to forcibly close a stuck circuit is deferred to E28 admin tooling (parallels the `POST /admin/reconcile/run` deferral in S04.26 AC 9).
- **Status page link** in the banner: deferred until a status page exists (the `learnMore` i18n key is intentionally not authored in this story per AC 8).
- **Email / Slack notification** of degraded mode (vs. just the in-app banner): out of scope; the operator pager off `sirmaai_circuit_state_total{state="open"}` is the existing notification path for ops, and tenant-facing notification escalation is an E16-Slack/Teams concern (the in-app banner is the *immediate* surface; richer tenant notification is a separate UX decision).
- **Multi-replica Redis-backed circuit state**: pre-scale backlog item per E04-R-005 — not this story.

### References

- `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §ADR-004 *Decision* paragraph (line 134), §11.3 risk #12 (line 511) — the "fail-open with degraded result + tenant-visible banner" invariant + the three-pillar §11.3 risk #12 mitigation chain.
- `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md` lines 480, 509 — Epic 4 amendment AC 9 ("Tenant-visible 'AI analysis temporarily unavailable' banner surfaced on circuit-breaker open >5 minutes") + S04.27 story summary.
- `eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md` — FR-54 / FR-55 / NFR-26 (degraded-mode is the user-facing side of the same risk #12 mitigation).
- `eusolicit-docs/test-artifacts/test-design-epic-04.md` — pre-amendment test design (does NOT enumerate S04.27 scenarios; the P0/P1/P2 framework is applied above per the test design notes).
- `eusolicit-docs/implementation-artifacts/4-26-run-state-reconciler.md` — sibling reconciler story; the "UNBLOCKS" section explicitly names S04.27 as the downstream banner consumer of the reconciler's circuit/state observability.
- `eusolicit-docs/implementation-artifacts/4-24-async-run-and-jobs-polling.md` — composite circuit-key namespace convention (`sirmaai_run:{logical}:{pid}`, `sirmaai_run_poll:{type}:{pid}`).
- `services/sirmaai-gateway/src/sirmaai_gateway/services/circuit_breaker.py` (S04.06) — `AgentCircuit` source; the file this story extends.
- `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` lines 362, 492 — circuit-key construction sites; the suffix convention this story relies on.
- `services/client-api/src/client_api/models/sirmaai_project.py` (S04.21) — `SirmaAIProject` ORM model; the resolution path's source of truth.
- `services/client-api/src/client_api/services/ai_gateway_client.py` (S11.03) — `AiGatewayClient`; the client extended with `fetch_tenant_status`.
- `frontend/apps/client/app/[locale]/(protected)/components/TrialBanner.tsx` — the banner-component pattern to mirror.
- `frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/layout.tsx` — the integration point.

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (claude.ai/code)

### Debug Log References

1. **Critical bug: `opened_at` reset on HALF_OPEN→OPEN** — Initial `_open_circuit()` used `if previous_state != CircuitState.OPEN: self.opened_at = time.monotonic()`. Since HALF_OPEN != OPEN this incorrectly reset `opened_at` on flap transitions. Fixed to `if self.opened_at is None: self.opened_at = time.monotonic()` (only set once per streak). Test `test_half_open_to_open_preserves_opened_at` caught this.

2. **Ruff I001 import ordering in `main.py`** — `system_status` import wasn't alphabetical. Fixed via `ruff check --fix`.

3. **Ruff F401 unused imports in `admin.py`** — `time`, `UTC`, `timedelta` imported but unused in the new endpoint. Removed.

4. **Mypy arg-type in `fetch_tenant_status`** — `params` dict needed explicit `dict[str, str]` annotation and `threshold_seconds` converted to `str()` to satisfy httpx's QueryParams type.

5. **Frontend type error in `lib/api/system.ts`** — `apiClient.get<T>()` returns `AxiosResponse<T>`, not `T`. Fixed to return `response.data`.

6. **Frontend test `MODULE_NOT_FOUND`** — Test used `require("../app/[locale]/(protected)/components/DegradedAIBanner")` inside a helper function. Vitest's ESM resolver can't handle CJS `require()` with bracket paths. Fixed by switching to a top-level static `import` (vi.mock is auto-hoisted so no lazy require needed).

7. **Pre-existing E501 in `ai_gateway_client.py`** — File-header comment on line 9 exceeded 120 chars. Split the long comment across two lines to pass lint.

### Completion Notes List

- **`pnpm check:i18n` not available** — The root frontend `package.json` has no `check:i18n` script. The i18n parity is enforced by `scripts/__tests__/check-i18n-keys.test.ts` (which passes: ✓ 14 tests in the full test run). Both `en.json` and `bg.json` have matching `system.aiUnavailable.{message,dismiss}` keys.

- **AC 14 `make test-service SVC=*`** — This Make target does not exist on this host. Used `python3 -m pytest services/<service>/tests/` directly. All sirmaai-gateway unit tests: 405 passed, 1 skipped. Client-api tests: 431 passed (infrastructure-dependent integration tests error on missing postgres/redis, matching project memory note "host venv only for ruff/collection/syntax").

- **E2E NOT required** — Intentional carry-forward per AC 14. The banner is a thin component reading a thin endpoint; E2E coverage deferred to next layout-touching story.

- **Multi-replica caveat** — The `_circuits` registry is per-process. Banner may flicker across replicas in multi-replica gateway deployments. Deferred to Redis-backed circuit state migration (E04-R-005 pre-scale backlog).

- **`messages/en.json` `system` key** — Added as a top-level section `"system": { "aiUnavailable": { "message": "...", "dismiss": "Dismiss" } }`. No `learnMore` key (intentionally excluded per AC 8 — no link target exists yet).

### File List

| File | Action |
|---|---|
| `services/sirmaai-gateway/src/sirmaai_gateway/services/circuit_breaker.py` | UPDATED — added `opened_at` field, updated `_open_circuit()`/`_close_circuit()`, added `tenant_open_circuits()`, extended `as_dict()`. |
| `services/sirmaai-gateway/src/sirmaai_gateway/routers/admin.py` | UPDATED — added `TenantStatusResponse` model and `GET /admin/tenant-status` endpoint. |
| `services/sirmaai-gateway/tests/unit/test_circuit_breaker_opened_at.py` | NEW — `opened_at` semantics + HALF_OPEN flap continuity tests. |
| `services/sirmaai-gateway/tests/unit/test_tenant_open_circuits.py` | NEW — cross-tenant isolation unit tests. |
| `services/sirmaai-gateway/tests/integration/test_admin_tenant_status.py` | NEW — `/admin/tenant-status` integration tests. |
| `services/client-api/src/client_api/api/v1/system_status.py` | NEW — `GET /api/v1/system/ai-status` router + `AiStatusResponse` model. |
| `services/client-api/src/client_api/services/ai_gateway_client.py` | UPDATED — added `fetch_tenant_status()` method. |
| `services/client-api/src/client_api/main.py` | UPDATED — wired `system_status_v1.router`. |
| `services/client-api/tests/api/test_ai_status_endpoint.py` | NEW — cross-tenant isolation + fail-closed timeout integration tests. |
| `frontend/apps/client/app/[locale]/(protected)/components/DegradedAIBanner.tsx` | NEW — banner component (mirrors TrialBanner, red palette, sessionStorage dismiss). |
| `frontend/apps/client/lib/api/system.ts` | NEW — `getAiStatus()` typed fetch helper. |
| `frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/layout.tsx` | UPDATED — `<DegradedAIBanner />` below TrialBanner slot. |
| `frontend/apps/client/messages/en.json` | UPDATED — `system.aiUnavailable.{message,dismiss}`. |
| `frontend/apps/client/messages/bg.json` | UPDATED — Bulgarian translations. |
| `frontend/apps/client/__tests__/degraded-ai-banner.test.tsx` | NEW — 4 component tests (render/no-render/dismiss/loading). |

## Senior Developer Review

**Reviewer:** Claude (bmad-code-review skill, autopilot)
**Date:** 2026-05-14
**Verdict:** **REVIEW: Approve** — implementation faithfully realises the AC intent, the cross-tenant negative test invariant is honoured, and the fail-closed UX path is wired and exercised. Findings below are advisory; none are blocking.

### Strengths

- **`opened_at` invariant correctly implemented.** The dev caught a real spec inconsistency: AC 1's literal instruction ("if previous state was NOT `CircuitState.OPEN`, set `opened_at`") would have re-set the clock on HALF_OPEN→OPEN flaps, which contradicts AC 1's "Critical invariant" paragraph. The chosen guard (`if self.opened_at is None: self.opened_at = time.monotonic()`) is the right realisation of the *intent* and is regression-tested by `test_half_open_to_open_preserves_opened_at` and `test_post_recovery_open_is_fresh_episode`. Good catch documented in Debug Log #1.
- **Cross-tenant negative tests are real.** Both layers are present: `test_cross_tenant_isolation_proj_a_open_proj_b_closed` at the helper level and `test_cross_tenant_isolation` at the API integration level (with `respx` proving exactly two distinct outbound calls). The structural path `current_user.company_id → SirmaAIProject.sirmaai_project_id → suffix match` enforces the boundary and the test asserts it.
- **Fail-closed UX is tested, not just promised.** `test_fail_closed_on_gateway_timeout` asserts both the response shape *and* the WARN log shape (`error_type=AiGatewayTimeoutError`, no `error=str(exc)`) — closes the "logs are silent secrets-leak surface" review lesson carried from S04.22.
- **Threshold boundary covered.** `test_five_minute_threshold_boundary` and `test_exactly_at_threshold_is_included` pin the `>=` semantics at 300 s — protects against off-by-one drift in future refactors.
- **Pre-amendment circuits correctly excluded** (`test_pre_amendment_circuit_not_matched`) — confirms the suffix-match convention works as a soft type-guard for which circuits are banner-eligible.

### Findings (advisory, ranked by impact)

**M1 — `client-api` router registration deviates from AC 4 text (low impact).**
AC 4 says "Wire into the FastAPI app via the existing `api/v1/__init__.py` `include_router` aggregator." The dev wired it directly in `main.py` (`api_v1_router.include_router(system_status_v1.router)`). This matches the *existing* project pattern (every neighbour does the same in `main.py`) so it is the right convention, but the AC text is wrong — the router-per-resource convention referenced in AC 4 is the file-layout convention, not a `__init__.py` indirection. No behavioural impact; flag as `DEVIATION_TYPE: CONTRADICTORY_SPEC`, `DEVIATION_SEVERITY: deferrable`. Suggest a docs-only follow-up to align the AC text with the project convention.

**M2 — Two `role="banner"` elements can render simultaneously.**
`TrialBanner.tsx` and `DegradedAIBanner.tsx` both set `role="banner"`. ARIA spec restricts `role="banner"` to one element per page (it maps to the `<header>` landmark). When a trialing tenant hits a degraded gateway, both render, and assistive tech announces two banner landmarks. The dev mirrored the existing `TrialBanner` pattern — so the regression was inherited, not introduced. **Fix path:** drop `role="banner"` from `DegradedAIBanner` (the visible message + `aria-live="polite"` already cover SR users) or change one of them to `role="status"` / `role="region" aria-label=...`. Out of scope for this story given the pattern is shared; raise as a separate a11y story.

**M3 — Integration-test fixture seeds rows in a committed transaction.**
`ai_status_app` in `test_ai_status_endpoint.py` opens `session.begin()`, INSERTs the company/user/SirmaAIProject rows, and exits the context without rollback — i.e. the rows commit. The comment acknowledges this ("Do NOT rollback here — we need the data visible to the app's sessions"). The project memory rule is "Never call `commit()` inside a test." The pattern is *necessary* for multi-session integration tests where the app opens its own session and can't see uncommitted data; the test does `DELETE` in `finally`. **Risk:** if a worker crashes mid-test, the test rows leak. **Mitigation already in place:** finally-block cleanup + unique UUID suffixes. Acceptable, but worth a shared `committed_seed` helper to avoid each test re-inventing this pattern.

**L1 — `AiGatewayClient.fetch_tenant_status` builds a new `httpx.AsyncClient` per request.**
Every call constructs `httpx.AsyncClient(timeout=5.0)` and tears it down — no pooled connection. At 60 s poll cadence × N concurrent users this is fine, but it diverges from the (presumed) `run_agent` pattern. Minor perf and consistency note. Out of scope for this story.

**L2 — `tenant_open_circuits` iterates `_circuits` without a lock.**
If a circuit is added/removed mid-iteration (e.g. lazy `get_circuit()` triggered by a concurrent request to S04.24), `RuntimeError: dictionary changed size during iteration` can surface. In practice circuits are pre-initialised at startup and only state-mutate thereafter, so dict size is stable; the failure window is narrow. Consider `for circuit in list(_circuits.values()):` to be defensive — single-line, no perf cost.

**L3 — `tenant_open_circuits` filter and `as_dict()` are not atomic.**
A circuit can transition OPEN→HALF_OPEN→CLOSED between the filter check and the `as_dict()` call at the end of the loop, producing a snapshot whose `circuit_state` is no longer "open" even though it was included in the result. The window is millisecond-scale and the consumer (banner) does not branch on `circuit_state`, so impact is cosmetic — but admin tooling that consumes `/admin/tenant-status.agents_open` may observe odd-looking snapshots. Acceptable as best-effort observability.

**L4 — `AiGatewayUnavailableError` carries `response.text[:500]` in an attribute.**
`fetch_tenant_status` raises `AiGatewayUnavailableError(response.status_code, response.text[:500])`. The client-api router catches and logs only `error_type=type(exc).__name__` — so the body never reaches logs **through this path**. However, the exception attribute is preserved; if a future caller does `log.warning(..., error=str(exc))` the truncated body (which may embed circuit keys / project IDs) leaks. Consider scrubbing the body to `"<redacted>"` for non-debug surfaces, mirroring the discipline called out in Anti-patterns "NEVER log `error=str(exc)`".

**L5 — Suffix-match assumes SirmaAI project IDs cannot contain `:`.**
`circuit.agent_name.endswith(f":{sirmaai_project_id}")` is unambiguous *only* if `sirmaai_project_id` contains no `:`. If a future SirmaAI tenancy returns an id like `tenant:foo`, the match collides with the namespace separator. The current SirmaAI project IDs are short hex strings, so this is theoretical. Document the invariant on `SirmaAIProject.sirmaai_project_id` or validate at provisioning time.

**L6 — `pnpm check:i18n` is referenced by AC 8 / AC 14 but does not exist as a script.**
Dev correctly identified this and pointed at `scripts/__tests__/check-i18n-keys.test.ts` which enforces parity through the regular test suite. No behavioural gap — the two new keys are present in both `en.json` and `bg.json` (verified). Recommend either (a) adding a `check:i18n` script that runs that single test, or (b) updating the project DoD / story templates to drop the stale reference.

**L7 — `make test-service` reportedly missing on host.**
Completion Notes flag `make test-service SVC=*` as nonexistent on the dev host, contradicting `CLAUDE.md`. Two possibilities: the target *is* defined and the dev's venv shells couldn't resolve it, or the Makefile drift is real. Either way, the project memory rule "host venv lacks service deps; use CI for runtime" is honoured. Worth a sprint-status callout to operators to verify CI green before merge — the locally-asserted "405 passed / 431 passed" is the right diligence given the constraint.

### What I verified

- Diff scope is exactly the files enumerated in the story File List.
- AC 1: `_open_circuit` / `_close_circuit` correctly preserve / clear `opened_at`; `as_dict()` includes the ISO field as an additive (non-breaking) addition.
- AC 2: `tenant_open_circuits` honours the empty-id guard, the suffix-match convention, the `>=` threshold, and sorts deterministically.
- AC 3: `/admin/tenant-status` is mounted under the existing `/admin` prefix, uses the documented query-param bounds (`min_length=1`, `ge=1, le=3600`), and computes `degraded_since` as the *earliest* `opened_at`.
- AC 4: `/api/v1/system/ai-status` requires auth (via `get_current_user` which checks `is_active`), short-circuits for unprovisioned tenants without a gateway hop, and fail-closes on `AiGatewayTimeoutError` / `AiGatewayUnavailableError` with a WARN log of `error_type` only.
- AC 5: `fetch_tenant_status` uses an explicit 5 s `httpx` timeout, performs no retry, raises the typed exceptions, and does not log the response body.
- AC 6 / 7: `DegradedAIBanner` mirrors the `TrialBanner` pattern (sessionStorage dismiss, `mounted` SSR guard, red palette), and is wired below `TrialBanner` in the workspace layout with no short-circuit on trialing state.
- AC 8: Both locale files contain `system.aiUnavailable.{message,dismiss}`; the EN copy matches the architecture-amendment phrasing.
- AC 9 / 10: cross-tenant integration test asserts per-tenant resolution with exactly two outbound calls; fail-closed timeout test asserts response *and* log shape.
- AC 11: HALF_OPEN flap continuity and post-recovery-fresh-episode are both tested; 5-minute boundary test pins `>=` semantics.
- AC 13: Banner unit tests cover render / no-render / dismiss-persist / loading.

### Recommended next steps (not blocking)

1. Open a follow-up a11y story to deduplicate `role="banner"` across `TrialBanner` + `DegradedAIBanner`.
2. Add a shared `committed_seed` test helper in `tests/conftest.py` and migrate the `ai_status_app` fixture to use it.
3. Patch AC 4 text in this story (or in the template) to match the `main.py` registration convention.
4. Add `pnpm check:i18n` as a thin script wrapping the existing locale-parity test, so future stories' DoD line passes verbatim.
5. Validate that SirmaAI project IDs cannot contain `:` at the provisioning layer (or document the constraint on the ORM model).

---

DEVIATION: AC 4 router registration uses `main.py` direct include rather than `api/v1/__init__.py` aggregator
DEVIATION_TYPE: CONTRADICTORY_SPEC
DEVIATION_SEVERITY: deferrable

DEVIATION: AC 1 literal "if previous state was NOT OPEN" guard refined to "if `opened_at is None`" to preserve HALF_OPEN→OPEN flap continuity
DEVIATION_TYPE: CONTRADICTORY_SPEC
DEVIATION_SEVERITY: deferrable

DEVIATION: `pnpm check:i18n` script referenced by AC 8 / AC 14 does not exist; parity is enforced by `scripts/__tests__/check-i18n-keys.test.ts` instead
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable
