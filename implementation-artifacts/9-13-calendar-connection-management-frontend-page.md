# Story 9.13: Calendar Connection Management Frontend Page

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 09: Notifications, Alerts & Calendar

## Metadata
- **Story Key:** 9-13-calendar-connection-management-frontend-page
- **Points:** 3
- **Type:** frontend (with one thin backend GET endpoint to expose connection status)
- **Module:** `frontend/apps/client` (new `/settings/calendar` route + supporting `lib/api/calendar.ts`, `lib/queries/use-calendar.ts`, i18n keys, sidebar nav entry) + tiny additive backend endpoint in `services/client-api/src/client_api/api/v1/calendar.py`
- **Priority:** P1 (Beta milestone — front-door for the entire calendar pipeline shipped in Stories 9.7 / 9.8 / 9.9. Without this page, users have no way to initiate Google or Microsoft OAuth, copy their iCal subscription URL, view sync status, or disconnect — i.e. the calendar feature is functionally inaccessible. The OAuth callback URLs in the backend (`calendar.py:208–209, 384–386`) already redirect to `/{locale}/settings/calendar?connected=...`, so this route is the explicit landing target the backend assumes exists.)

## Story

As a **company user with subscription tier ≥ Starter who tracks opportunity deadlines**,
I want **a single settings page where I can connect my Google or Microsoft Outlook calendar via OAuth, copy a private iCal subscription URL, see when my connections last synced and how many events were created/updated/deleted, and disconnect any provider at will**,
so that **the deadlines for opportunities I track in EU Solicit appear automatically in the calendar app I already use, without me having to re-import .ics files or build any integration myself**.

## Description

This is the **frontend half** of the calendar-connection feature. Every server-side piece is already done and verified in production:

- **Story 9.7** delivered the iCal feed: `POST /api/v1/calendar/ical/generate-token` rotates (or first-creates) the per-user opaque UUID token, returns `{feed_url, token, rotated_at}`; `GET /api/v1/calendar/ical/{token}` is the public unauthenticated subscription endpoint that returns a `text/calendar` body. Both live in `services/client-api/src/client_api/api/v1/calendar.py`.
- **Story 9.8** delivered Google Calendar: `GET /api/v1/calendar/google/connect` (initiates OAuth, sets `oauth_user_id` in the session, redirects to Google), `GET /api/v1/calendar/google/callback` (exchanges code, encrypts tokens with Fernet, stores in `client.calendar_connections`, redirects to `/{locale}/settings/calendar?connected=google` on success or `?error=oauth_error&provider=google` on failure), `DELETE /api/v1/calendar/google/disconnect` (revokes at Google + deletes the connection row, returns 204 on success or 404 if no connection exists), plus a Celery periodic sync task in the Notification Service that writes to `notification.sync_log`.
- **Story 9.9** delivered the Microsoft Outlook mirror: `/api/v1/calendar/microsoft/connect|callback|disconnect` with identical semantics, Microsoft Graph instead of Google APIs.

**What this story renders:** a Next.js page at `app/[locale]/(protected)/settings/calendar/page.tsx` with three sections — Google, Microsoft, and iCal — each showing the right empty / connected / error state, with Connect / Disconnect / Generate-URL / Copy / Regenerate actions wired to the existing endpoints. The page ALSO polls a new `GET /api/v1/calendar/connections` endpoint every 30 s (per the epic AC and test-design row P2) to show the latest sync timestamp + events-synced counts from `notification.sync_log` joined with `client.calendar_connections`.

**Why one new backend endpoint is in scope:** there is no existing way to ask the backend "what calendars does this user have connected, when did each last sync, and how many events were touched". Trying every disconnect/connect endpoint to deduce state is brittle and abusive; a single read-only listing endpoint is the correct surface and is ~50 lines of Python. It is strictly additive (no existing endpoint changes) and lives in the same `calendar.py` router. Story 9.12 stayed strictly frontend because Story 9.3 had already delivered `GET /api/v1/alerts/preferences`; the calendar epic shipped only OAuth callbacks and the iCal endpoints, so the listing surface needs to be added here.

**The page is `"use client"`** — every interaction is either an OAuth redirect, a mutation, or a polled query. SSR offers nothing here.

**What ships:**

1. **Backend (additive):** `GET /api/v1/calendar/connections` in `services/client-api/src/client_api/api/v1/calendar.py`. Authenticated. Returns a fixed-shape JSON: `{google: ConnectionStatus, microsoft: ConnectionStatus, ical: ICalStatus}` where `ConnectionStatus = {connected: bool, connected_at: datetime | null, last_sync: SyncSummary | null}` and `SyncSummary = {started_at: datetime, completed_at: datetime | null, events_created: int, events_updated: int, events_deleted: int, error_message: str | null}`. `ICalStatus = {generated: bool, feed_url: str | null, rotated_at: datetime | null}`. Implementation queries `client.calendar_connections` for the three providers and (for google/microsoft) a single `LATERAL` or correlated subquery against `notification.sync_log` ORDER BY `started_at DESC LIMIT 1`. Cross-schema read is OK (the Notification Service already grants `notification.sync_log` SELECT to the `client_api` DB role per the test design). Pydantic schemas live in `services/client-api/src/client_api/schemas/calendar_status.py` (new file).
2. **API module** — `lib/api/calendar.ts` (new) exposing typed `getCalendarConnections()`, `generateIcalToken()`, `disconnectGoogle()`, `disconnectMicrosoft()` against the existing routes plus the new GET. Plus types: `CalendarConnectionsResponse`, `ConnectionStatus`, `SyncSummary`, `ICalStatus`. Uses `apiClient` from `@eusolicit/ui` (same pattern as `lib/api/alerts.ts` and `lib/api/billing.ts`).
3. **React Query hooks** — `lib/queries/use-calendar.ts` (new) exposing `useCalendarConnections()` (queryKey `["calendar", "connections"]`, `refetchInterval: 30_000` per AC9), `useGenerateIcalToken()`, `useDisconnectGoogle()`, `useDisconnectMicrosoft()`. All mutations invalidate `["calendar", "connections"]` on success.
4. **Page route** — `app/[locale]/(protected)/settings/calendar/page.tsx` is a 1-line server component that renders `<CalendarConnectionsPage />`. Mirrors the `settings/billing/page.tsx` recipe verbatim.
5. **`CalendarConnectionsPage` client component** — `app/[locale]/(protected)/settings/calendar/components/CalendarConnectionsPage.tsx`. Owns the polling query, renders three section components (Google, Microsoft, iCal), and handles the success/error toast on mount when `?connected=google|microsoft` or `?error=oauth_error&provider=...` is present in the URL (read via `useSearchParams`). After reading the param it must scrub it from the URL via `router.replace(window.location.pathname)` so a refresh doesn't re-fire the toast.
6. **`GoogleCalendarSection` component** — shows the right state (loading / disconnected / connected / error), Connect button (a plain `<a href="/api/v1/calendar/google/connect">` because OAuth is a full-page redirect — NOT `fetch()`), Disconnect button + confirmation dialog wired to the mutation, and a "Last synced" sub-row when `last_sync` is present.
7. **`MicrosoftCalendarSection` component** — identical UI to the Google section but pointing at Microsoft endpoints.
8. **`ICalSection` component** — Generate / Regenerate URL button (mutation), copyable read-only `<input>` for the URL with a `<Button>` that calls `navigator.clipboard.writeText(...)` (same pattern as `app/[locale]/(protected)/settings/api-keys/components/ApiKeyManagementPage.tsx:100`), instructions list with localized strings for Google Calendar / Apple Calendar / Outlook subscription, and a regenerate confirmation dialog warning that the existing URL stops working.
9. **Sidebar nav entry** — `app/[locale]/(protected)/layout.tsx` is updated to add a "Calendar" link under `/${locale}/settings/calendar` with the `CalendarDays` icon from `lucide-react`. Inserted BEFORE the existing "Alerts" entry (which lives BEFORE the "Settings" entry per the Story 9.12 work, so the final order reads: ... → Calendar → Alerts → Settings).
10. **i18n** — new `calendar.connections.*` namespace in both `messages/en.json` and `messages/bg.json`. Plus `nav.calendar` key. Total ~40 new leaf keys.
11. **ATDD static test file** — `__tests__/calendar-connections-s9-13.test.ts` (Vitest, `environment: 'node'`) verifies file existence, exports, testid presence in component source, i18n key parity in both locales, and the new backend endpoint signature (the Pydantic schema file exists with the expected fields). Pattern is identical to `__tests__/alert-preferences-s9-12.test.ts` (Story 9.12), `__tests__/grant-tools-s11-11.test.ts` (Story 11.11), and `__tests__/subscription-management-s8-13.test.ts` (Story 8.13).

**What this story does NOT change:**
- The OAuth flows themselves — Stories 9.8 and 9.9 are `done`. We DO NOT touch `oauth.google_calendar`/`oauth.microsoft_calendar` registration, the Fernet token-encryption helper, the session-cookie identity binding, or any sync task in the notification service.
- The iCal feed generation logic (Story 9.7 owns `render_ical_for_user`) — we only wrap the existing `POST /api/v1/calendar/ical/generate-token` mutation.
- The alert-preferences page (Story 9.12 — separate sibling story, also `done`) — both pages live under `/settings` but are independent.
- The hardcoded `bg` locale in the OAuth callback redirect URLs (`calendar.py:208`, `:209`, `:384`, `:386`). That is a known backend smell tracked separately; for this story the page must work whether the user is on `/bg/settings/calendar` (matches backend) or `/en/settings/calendar` (mismatch — the toast wiring still detects `?connected=...` regardless of locale). Do NOT "fix" the backend redirect in this story; flag in Open Questions.

## Acceptance Criteria

1. [ ] **AC1 — Route + page scaffold.** `app/[locale]/(protected)/settings/calendar/page.tsx` exists as a server component that renders `<CalendarConnectionsPage />`. The client component has `data-testid="calendar-connections-page"` on its outermost wrapper and `data-testid="calendar-connections-page-title"` on its `<h1>` reading `t("calendar.connections.pageTitle")`. Visiting `/{locale}/settings/calendar` while authenticated renders without runtime error; unauthenticated visits are intercepted by the existing `<AuthGuard>` in `(protected)/layout.tsx` and redirected to `/login` (no new auth wiring in this story). The page MUST be marked `"use client"` because it owns interactive state, polling, and `useSearchParams`.

2. [ ] **AC2 — Backend GET /api/v1/calendar/connections endpoint.** A new authenticated endpoint added to `services/client-api/src/client_api/api/v1/calendar.py`:
   - Path: `GET /api/v1/calendar/connections`
   - Auth: `Depends(get_current_user)` — same dependency used by every other authenticated route in the file. Unauthenticated → 401.
   - Response (200) — Pydantic model in `services/client-api/src/client_api/schemas/calendar_status.py` (new file):
     ```python
     class SyncSummary(BaseModel):
         started_at: datetime
         completed_at: datetime | None
         events_created: int
         events_updated: int
         events_deleted: int
         error_message: str | None
     class ConnectionStatus(BaseModel):
         connected: bool
         connected_at: datetime | None
         last_sync: SyncSummary | None
     class ICalStatus(BaseModel):
         generated: bool
         feed_url: str | None
         rotated_at: datetime | None
     class CalendarConnectionsResponse(BaseModel):
         google: ConnectionStatus
         microsoft: ConnectionStatus
         ical: ICalStatus
     ```
   - Implementation: a single round-trip query joining `client.calendar_connections` (filtered to `user_id = current_user.user_id`) with the latest `notification.sync_log` row per `(calendar_connection_id, provider)` (use `DISTINCT ON (calendar_connection_id) ... ORDER BY started_at DESC` or a correlated sub-select). For the `ical` provider entry the `feed_url` is rebuilt on the server as `f"{settings.api_public_base_url}/api/v1/calendar/ical/{token}"` (same shape as the existing `/ical/generate-token` response). When a row is missing for a provider, return `connected=False`/`generated=False` and `null`s for the dependent fields. Backend test (`services/client-api/tests/api/test_calendar_connections.py`, new) covers: (a) all-three-providers seeded → returns all three connected with sync data; (b) only google seeded → microsoft + ical return disconnected/empty; (c) unauthenticated → 401; (d) cross-user isolation (user A's call returns NO rows from user B). Tests use the existing async test client + per-test DB rollback fixture.
   - Router prefix is unchanged (`/calendar` already on `router = APIRouter(prefix="/calendar", tags=["Calendar"])` in `calendar.py:34`).

3. [ ] **AC3 — Loading + error + populated states for the page.**
   - Loading (initial fetch only): render a section `data-testid="calendar-connections-loading"` containing 3 `<Skeleton>` placeholders (one per provider section).
   - Error (fetch fails): render an inline `<Alert variant="destructive">` with `data-testid="calendar-connections-error"` and a Retry button calling `query.refetch()`. Subsequent successful refetch removes the alert.
   - Populated: render the three section components in this fixed visual order: **Google → Microsoft → iCal**.
   - Polling refetch failures DO NOT replace the populated state with the error state — keep last-known data and show a non-blocking inline `<Tooltip>` on the page header `data-testid="calendar-connections-stale-indicator"` saying `t("calendar.connections.staleData")`. Use `query.isError && query.data` as the condition.

4. [ ] **AC4 — Google section: disconnected state.** When `data.google.connected === false`:
   - Render `<GoogleCalendarSection>` with `data-testid="calendar-section-google"`.
   - Header: localized `t("calendar.connections.google.title")` ("Google Calendar") + Google logo / icon (use `lucide-react` `Calendar` icon — the brand assets are NOT imported in this story).
   - Body copy: `t("calendar.connections.google.disconnectedBody")` (~1 sentence) explaining that connecting will sync tracked opportunity deadlines.
   - Primary action: an anchor tag (NOT a button), `<a href="/api/v1/calendar/google/connect" data-testid="calendar-google-connect-btn">` reading `t("calendar.connections.google.connectBtn")` ("Connect Google Calendar"). Anchor MUST be `href` to a same-origin path so the browser performs a full navigation — `fetch()` cannot follow the OAuth redirect chain. No `target="_blank"` (OAuth state binding requires the same browser session — see calendar.py:180 docstring).

5. [ ] **AC5 — Google section: connected state.** When `data.google.connected === true`:
   - Replace the body with a connected card.
   - Show "Connected on" + `t("calendar.connections.connectedAt", { date })` formatted via the user's locale (`Intl.DateTimeFormat`), `data-testid="calendar-google-connected-at"`.
   - When `data.google.last_sync` is non-null, show:
     - "Last synced" + relative-time string for `last_sync.completed_at ?? last_sync.started_at`, `data-testid="calendar-google-last-synced"`.
     - Three numeric badges for `events_created`, `events_updated`, `events_deleted`, each `data-testid={`calendar-google-events-${kind}`}` where kind ∈ `created | updated | deleted`. Use `t("calendar.connections.eventsCreated", { n })` style interpolation.
     - When `last_sync.error_message` is non-null, render a subtle `<Alert variant="warning">` `data-testid="calendar-google-sync-error"` with `last_sync.error_message` (truncate to 200 chars; backend already log-redacts secrets per E09-R-001).
   - Disconnect button `data-testid="calendar-google-disconnect-btn"` opens an `<AlertDialog>` `data-testid="calendar-google-disconnect-dialog"`. Cancel closes without API call. Confirm calls `useDisconnectGoogle().mutate()`. On success: toast `t("calendar.connections.google.disconnected")` and the section flips back to disconnected on the next query refetch (mutation `onSuccess` invalidates the query). On 404 (already disconnected): treat as success (idempotent UX). On 5xx: toast `t("calendar.connections.genericError")`.

6. [ ] **AC6 — Microsoft section: identical contract to AC4 + AC5.** Replace every `google` testid with `microsoft` (e.g., `calendar-section-microsoft`, `calendar-microsoft-connect-btn`, `calendar-microsoft-disconnect-btn`, `calendar-microsoft-events-${kind}`). Connect anchor href: `/api/v1/calendar/microsoft/connect`. i18n keys mirror under `calendar.connections.microsoft.*` (`title` = "Microsoft Outlook", `connectBtn` = "Connect Outlook Calendar"). Sync-stats payload comes from `data.microsoft.last_sync` populated by the backend's read of `notification.sync_log WHERE provider = 'microsoft'`.

7. [ ] **AC7 — iCal section: not-yet-generated state.** When `data.ical.generated === false`:
   - Render `<ICalSection>` with `data-testid="calendar-section-ical"`.
   - Header: `t("calendar.connections.ical.title")` ("iCal Subscription URL").
   - Body: `t("calendar.connections.ical.notGeneratedBody")` (~2 sentences explaining iCal is the no-OAuth option supported by every calendar app).
   - Generate button `data-testid="calendar-ical-generate-btn"` calls `useGenerateIcalToken().mutate()`. On success → query is invalidated and the section re-renders into the generated state (AC8). On 5xx → toast `t("calendar.connections.genericError")`.

8. [ ] **AC8 — iCal section: generated state.** When `data.ical.generated === true`:
   - Render a read-only `<input data-testid="calendar-ical-url-input">` with `value={data.ical.feed_url}` and `readOnly` (NOT `disabled` — readers need to triple-click-select the value if clipboard fails).
   - Copy button `data-testid="calendar-ical-copy-btn"` calls `navigator.clipboard.writeText(data.ical.feed_url)` then shows a toast `t("calendar.connections.ical.copied")`. On clipboard API failure (e.g., insecure context, permission denied) — toast `t("calendar.connections.ical.copyFailed")` and DO NOT crash. Mirror the recipe in `app/[locale]/(protected)/settings/api-keys/components/ApiKeyManagementPage.tsx:100`.
   - "Generated on" sub-row `data-testid="calendar-ical-rotated-at"` showing `t("calendar.connections.ical.rotatedAt", { date })` for `data.ical.rotated_at`.
   - Subscription instructions: a `<details>` element `data-testid="calendar-ical-instructions"` with three `<summary>`/body pairs — Google Calendar, Apple Calendar, Outlook — each containing localized 2–3 step instructions referencing the URL above. Strings are i18n keys `calendar.connections.ical.howTo.google|apple|outlook` (each with title + steps array).
   - Regenerate button `data-testid="calendar-ical-regenerate-btn"` opens an `<AlertDialog>` `data-testid="calendar-ical-regenerate-dialog"` warning that "regenerating invalidates the previous URL — calendar apps subscribed to the old URL will stop receiving updates" (string `t("calendar.connections.ical.regenerateWarning")`). Cancel closes without API call. Confirm calls `useGenerateIcalToken().mutate()` (same mutation — backend `POST /ical/generate-token` is upsert, returns the new token, and the existing record's `token` field is replaced atomically per `calendar.py:73–86`). On success: toast `t("calendar.connections.ical.regenerated")` and the new URL appears in the input.

9. [ ] **AC9 — Polling.** `useCalendarConnections()` sets `refetchInterval: 30_000` (30 s) so the page picks up sync-log updates without a manual refresh. Polling MUST be paused when the document is hidden — pass `refetchIntervalInBackground: false` (default in TanStack Query v5 — explicitly assert it). Cleanup (interval cancellation) is automatic on component unmount. The 30 s cadence matches the test-design row P2 ("Sync status 30-second poll updates last-synced display without full page reload").

10. [ ] **AC10 — OAuth callback toast handling.** On mount, `<CalendarConnectionsPage>` reads `useSearchParams()`:
    - If `?connected=google` → fire success toast `t("calendar.connections.google.connected")` and scrub the param via `router.replace(pathname, { scroll: false })`.
    - If `?connected=microsoft` → fire success toast `t("calendar.connections.microsoft.connected")` and scrub.
    - If `?error=oauth_error&provider=google` → fire error toast `t("calendar.connections.google.oauthError")` and scrub.
    - If `?error=oauth_error&provider=microsoft` → fire error toast `t("calendar.connections.microsoft.oauthError")` and scrub.
    - All four cases run inside a `useEffect` with `[searchParams]` as dep so the toast fires exactly once per visit. The query `useCalendarConnections()` is invalidated AFTER firing the success toast so the section flips to the connected state on next refetch.

11. [ ] **AC11 — Sidebar navigation entry.** `app/[locale]/(protected)/layout.tsx` is updated:
    - Import `CalendarDays` from `lucide-react`.
    - Add `{ icon: CalendarDays, label: t("calendar"), href: \`/${locale}/settings/calendar\`, testId: "sidebar-nav-calendar" }` to `clientNavItems` IMMEDIATELY BEFORE the existing `{ icon: Bell, label: t("alerts"), ... }` entry (line ~74 — verify position before editing). Final visual order: ... → tenders → espd → offers → documents → team → **Calendar** → Alerts → Settings.
    - Add `nav.calendar` key to `messages/en.json` ("Calendar") and `messages/bg.json` ("Календар"). Do NOT remove or reorder any existing nav items; this is purely additive.

12. [ ] **AC12 — i18n keys.** New `calendar.connections.*` namespace (NOT replacing existing `calendar.*` keys, if any) in both `messages/en.json` and `messages/bg.json`. Required leaf keys:
    - **Page chrome:** `calendar.connections.pageTitle`, `pageDescription`, `staleData`, `genericError`
    - **Google section:** `calendar.connections.google.title`, `disconnectedBody`, `connectBtn`, `connected`, `disconnected`, `oauthError`, `disconnectConfirmTitle`, `disconnectConfirmDescription`, `disconnectConfirmBtn`
    - **Microsoft section (mirror of Google):** `calendar.connections.microsoft.title`, `disconnectedBody`, `connectBtn`, `connected`, `disconnected`, `oauthError`, `disconnectConfirmTitle`, `disconnectConfirmDescription`, `disconnectConfirmBtn`
    - **iCal section:** `calendar.connections.ical.title`, `notGeneratedBody`, `generateBtn`, `regenerateBtn`, `regenerated`, `copied`, `copyFailed`, `urlLabel`, `regenerateWarning`, `regenerateConfirmTitle`, `regenerateConfirmDescription`, `regenerateConfirmBtn`, `rotatedAt`, `howTo.google.title`, `howTo.google.steps`, `howTo.apple.title`, `howTo.apple.steps`, `howTo.outlook.title`, `howTo.outlook.steps`
    - **Shared:** `calendar.connections.connectedAt` (interpolates `{date}`), `lastSynced` (interpolates `{date}`), `eventsCreated|eventsUpdated|eventsDeleted` (each interpolates `{n}`), `disconnectBtn`, `cancelBtn`, `retryBtn`
    - **Sidebar:** `nav.calendar`
    Both locale files must contain the SAME set of leaf keys (the existing `__tests__/i18n-setup.test.ts` parity check enforces this — adding any key to one file without the other breaks CI). For text that is genuinely difficult to translate (e.g., calendar-app subscription instructions), use a best-effort Bulgarian translation — DO NOT leave any key missing, and flag any uncertain translations in the Open Questions.

13. [ ] **AC13 — ATDD coverage (static).** `__tests__/calendar-connections-s9-13.test.ts` is added with Vitest static checks. Required `describe` blocks (one per AC where there is a static-checkable artefact):
    - `AC1 — Route + page scaffold`
    - `AC2 — Backend endpoint + Pydantic schema file`
    - `AC3 — Loading / error states`
    - `AC4 + AC5 — Google section`
    - `AC6 — Microsoft section`
    - `AC7 + AC8 — iCal section`
    - `AC9 — Polling`
    - `AC10 — OAuth callback toast handling`
    - `AC11 — Sidebar nav entry`
    - `AC12 — i18n parity`
    - `AC13 — Test file structure`
    Each block performs: file-existence checks, grep-for-testid-string checks against the source files (not via DOM rendering — `environment: 'node'`), and i18n-key-presence checks against both locale JSON files. Test count target ≥ 20 individual `it(...)` blocks. The test file lives at `eusolicit-app/frontend/apps/client/__tests__/calendar-connections-s9-13.test.ts`. Pattern is identical to `__tests__/alert-preferences-s9-12.test.ts` (Story 9.12). For AC2 the test grep-checks the Python file at `eusolicit-app/services/client-api/src/client_api/api/v1/calendar.py` for the new route registration (`@router.get("/connections")`) AND the schema file at `eusolicit-app/services/client-api/src/client_api/schemas/calendar_status.py` for the four Pydantic class names — Vitest reading Python source is fine, it is a static string presence check.

14. [ ] **AC14 — Backend test.** `services/client-api/tests/api/test_calendar_connections.py` (new) covers four scenarios listed in AC2 (all-three-providers seeded; only google seeded; unauthenticated 401; cross-user isolation). Use the existing async pytest fixtures `client`, `db_session`, `auth_headers_for(user)` (search `services/client-api/tests/api/test_alert_preferences.py` and `tests/api/test_calendar.py` for the fixture names — match them verbatim). Tests must pass with `cd eusolicit-app/services/client-api && pytest tests/api/test_calendar_connections.py -v`. Do NOT mock SQL — use the real per-test DB transaction rollback fixture so the join against `notification.sync_log` is exercised end-to-end.

15. [ ] **AC15 — Lint, type-check, build.** All three pass for the `client` workspace:
    - `cd eusolicit-app/frontend && pnpm lint --filter client` — pass
    - `cd eusolicit-app/frontend && pnpm type-check --filter client` — pass
    - `cd eusolicit-app/frontend && pnpm build --filter client` — pass
    Plus the backend lint/type:
    - `cd eusolicit-app/services/client-api && ruff check src/client_api/api/v1/calendar.py src/client_api/schemas/calendar_status.py` — pass
    - `cd eusolicit-app/services/client-api && mypy src/client_api/api/v1/calendar.py src/client_api/schemas/calendar_status.py` (or whatever type-checker the service uses — verify by reading `services/client-api/pyproject.toml`) — pass
    New / modified files do NOT introduce `// @ts-ignore`, `// eslint-disable-next-line` blanket suppressions, or `any` types. All new components have explicit prop interfaces.

## Tasks / Subtasks

- [ ] **Task 1: Backend GET /api/v1/calendar/connections endpoint. (AC2, AC14)**
  - [ ] 1.1 Create `services/client-api/src/client_api/schemas/calendar_status.py` with the four Pydantic classes (`SyncSummary`, `ConnectionStatus`, `ICalStatus`, `CalendarConnectionsResponse`). Use `model_config = ConfigDict(from_attributes=True)` so SQLAlchemy row objects map cleanly. All `datetime` fields are timezone-aware (the backend stores UTC).
  - [ ] 1.2 Add `GET /connections` route to `services/client-api/src/client_api/api/v1/calendar.py`. Inject `Depends(get_current_user)` and `Depends(get_db_session)`. Query `client.calendar_connections WHERE user_id = current_user.user_id` for all three providers (`google`, `microsoft`, `ical`). For each of `google` and `microsoft`, do a single `SELECT * FROM notification.sync_log WHERE calendar_connection_id = :id ORDER BY started_at DESC LIMIT 1`. Build the response dict, fall back to `connected=False`/`generated=False` for missing providers. For ical: `feed_url = f"{settings.api_public_base_url}/api/v1/calendar/ical/{token}"` if a token exists.
  - [ ] 1.3 The cross-schema query against `notification.sync_log` MUST use a SQL string or `sa.text(...)` if the SQLAlchemy `SyncLog` model is not imported into the client_api package — verify whether `client_api` already imports any `notification.*` model (search `from notification` in `services/client-api/src/`). If not, use `sa.text("SELECT ... FROM notification.sync_log ...")` with bound params. Document the choice in the code comment.
  - [ ] 1.4 Create `services/client-api/tests/api/test_calendar_connections.py` with the four scenarios from AC14. Reuse the test fixtures from `services/client-api/tests/api/test_calendar.py` (it already exists for the iCal/OAuth routes — read it to match the fixture names and DB-rollback pattern). Run `cd eusolicit-app/services/client-api && pytest tests/api/test_calendar_connections.py -v` and confirm all pass.
  - [ ] 1.5 Run `ruff check` + the configured type-checker on the two changed files.

- [ ] **Task 2: Frontend API module. (AC2, AC4, AC5, AC6, AC7, AC8)**
  - [ ] 2.1 Create `eusolicit-app/frontend/apps/client/lib/api/calendar.ts` exporting:
    - Types: `SyncSummary`, `ConnectionStatus`, `ICalStatus`, `CalendarConnectionsResponse` — mirror the backend shape EXACTLY. All datetime fields typed as `string` (ISO 8601) — backend serializes `datetime` to ISO string by default in FastAPI.
    - `getCalendarConnections()` → `apiClient.get<CalendarConnectionsResponse>("/api/v1/calendar/connections")`
    - `generateIcalToken()` → `apiClient.post<{feed_url: string; token: string; rotated_at: string}>("/api/v1/calendar/ical/generate-token")` (return shape mirrors `ICalTokenResponse` Pydantic schema in `services/client-api/src/client_api/schemas/calendar_ical.py`)
    - `disconnectGoogle()` → `apiClient.delete("/api/v1/calendar/google/disconnect")` (returns 204; treat 404 as already-disconnected success)
    - `disconnectMicrosoft()` → mirror
  - [ ] 2.2 Use the existing `apiClient` from `@eusolicit/ui` (same import as `lib/api/billing.ts:5` and `lib/api/alerts.ts:1`).

- [ ] **Task 3: React Query hooks. (AC9)**
  - [ ] 3.1 Create `eusolicit-app/frontend/apps/client/lib/queries/use-calendar.ts` with `"use client"` header.
  - [ ] 3.2 Export `CALENDAR_QUERY_KEY = ["calendar", "connections"]`.
  - [ ] 3.3 `useCalendarConnections()` → `useQuery({queryKey: CALENDAR_QUERY_KEY, queryFn: getCalendarConnections, refetchInterval: 30_000, refetchIntervalInBackground: false, staleTime: 15_000})`.
  - [ ] 3.4 `useGenerateIcalToken()`, `useDisconnectGoogle()`, `useDisconnectMicrosoft()` are mutations that invalidate `CALENDAR_QUERY_KEY` on `onSuccess`. No optimistic updates on these — disconnect is destructive enough that a brief loading state is acceptable; iCal regenerate is rare enough that a full refetch is fine.

- [ ] **Task 4: Page route + page component. (AC1, AC3, AC10)**
  - [ ] 4.1 Create `app/[locale]/(protected)/settings/calendar/page.tsx` (server component, ~5 lines) — renders `<CalendarConnectionsPage />`. Add `metadata = { title, description }` block (mirror `settings/billing/page.tsx`).
  - [ ] 4.2 Create `app/[locale]/(protected)/settings/calendar/components/CalendarConnectionsPage.tsx` with `"use client"`.
  - [ ] 4.3 Use `useCalendarConnections()` for state. Render loading / error / populated per AC3. Stale-data tooltip when polling fails but cached data exists.
  - [ ] 4.4 Implement OAuth callback toast handling per AC10. Use `useSearchParams`, `useRouter`, `usePathname` from `next/navigation`. Wrap in a `useEffect` with `[searchParams]` dep. After firing the toast, call `router.replace(pathname, { scroll: false })` to scrub the query string. Use the toast helper from Story 3.11 (search `lib/` for `useToast` or similar).
  - [ ] 4.5 Render the three sections in order: `<GoogleCalendarSection data={data.google} />`, `<MicrosoftCalendarSection data={data.microsoft} />`, `<ICalSection data={data.ical} />`.

- [ ] **Task 5: GoogleCalendarSection + MicrosoftCalendarSection. (AC4, AC5, AC6)**
  - [ ] 5.1 Create `app/[locale]/(protected)/settings/calendar/components/GoogleCalendarSection.tsx`. Props: `{data: ConnectionStatus}`.
  - [ ] 5.2 Disconnected branch: header + body + `<a href="/api/v1/calendar/google/connect">` connect anchor (anchor, NOT button — see AC4 rationale).
  - [ ] 5.3 Connected branch: connected-at row, last-synced row (when `last_sync` non-null), three event-count badges, optional sync-error alert, disconnect button.
  - [ ] 5.4 Disconnect button opens `<AlertDialog>` (use `@eusolicit/ui` component). Confirm calls `useDisconnectGoogle().mutate()`. Treat 404 from the mutation as success-equivalent (use `onSettled` to invalidate regardless of mutation status).
  - [ ] 5.5 Create `MicrosoftCalendarSection.tsx` as a near-identical copy with the `microsoft` testids and i18n keys. Consider extracting a shared `<OAuthCalendarSection provider="google"|"microsoft">` if the duplication feels heavy — the trade-off is testid template-string complexity. Reviewer's call; either approach is fine. If extracted, the wrapper still passes through to the per-provider testids exactly as listed in AC4–AC6 so the static ATDD grep checks pass.

- [ ] **Task 6: ICalSection. (AC7, AC8)**
  - [ ] 6.1 Create `app/[locale]/(protected)/settings/calendar/components/ICalSection.tsx`. Props: `{data: ICalStatus}`.
  - [ ] 6.2 Not-generated branch: header + body + Generate button calling `useGenerateIcalToken().mutate()`.
  - [ ] 6.3 Generated branch: read-only URL `<input>` + Copy button (clipboard API with try/catch + toast) + rotated-at row + `<details>` instructions block + Regenerate button (opens confirmation dialog, calls same mutation).
  - [ ] 6.4 The instructions block uses `t("calendar.connections.ical.howTo.google.steps")` returning a string array; render as `<ol>`. If `next-intl` does not support array values directly, store as numbered keys (`step1`, `step2`, `step3`) and render explicitly.

- [ ] **Task 7: Sidebar nav entry. (AC11)**
  - [ ] 7.1 Edit `app/[locale]/(protected)/layout.tsx`: add `CalendarDays` to the `lucide-react` import block (alphabetically near `Bell`). Add the new nav item to `clientNavItems` immediately BEFORE the existing `Bell`/`alerts` entry. Use `testId: "sidebar-nav-calendar"`.
  - [ ] 7.2 Add `nav.calendar` key to both locale JSON files (en: "Calendar", bg: "Календар").

- [ ] **Task 8: i18n keys. (AC12)**
  - [ ] 8.1 Add the full `calendar.connections.*` sub-namespace described in AC12 to `messages/en.json`. Group keys logically (page chrome → google → microsoft → ical → shared).
  - [ ] 8.2 Mirror the same key set in `messages/bg.json` with Bulgarian translations. For `howTo.*` step arrays, translate best-effort and flag any uncertain rendering in Open Questions.
  - [ ] 8.3 Run `cd eusolicit-app/frontend && pnpm test --filter client -- i18n-setup` — must pass.

- [ ] **Task 9: ATDD static test file. (AC13)**
  - [ ] 9.1 Create `eusolicit-app/frontend/apps/client/__tests__/calendar-connections-s9-13.test.ts` modelled on `__tests__/alert-preferences-s9-12.test.ts` (read that file first to match the helpers, the `APP_ROOT` path constants, the `readFile` / `readJSON` pattern, and the `resolveNestedKey` helper — copy verbatim).
  - [ ] 9.2 Implement the describe blocks listed in AC13. Total `it(...)` count ≥ 20.
  - [ ] 9.3 For the AC2 backend checks: resolve the path `join(APP_ROOT, '..', '..', 'services', 'client-api', 'src', 'client_api', 'api', 'v1', 'calendar.py')` and grep for `@router.get("/connections")`. Resolve `services/client-api/src/client_api/schemas/calendar_status.py` and grep for `class SyncSummary`, `class ConnectionStatus`, `class ICalStatus`, `class CalendarConnectionsResponse`.
  - [ ] 9.4 Run `cd eusolicit-app/frontend && pnpm test --filter client -- calendar-connections-s9-13` — must pass.

- [ ] **Task 10: Quality gates. (AC15)**
  - [ ] 10.1 `cd eusolicit-app/frontend && pnpm lint --filter client` — pass.
  - [ ] 10.2 `cd eusolicit-app/frontend && pnpm type-check --filter client` — pass.
  - [ ] 10.3 `cd eusolicit-app/frontend && pnpm build --filter client` — pass.
  - [ ] 10.4 `cd eusolicit-app/services/client-api && ruff check src/client_api/api/v1/calendar.py src/client_api/schemas/calendar_status.py tests/api/test_calendar_connections.py` — pass.
  - [ ] 10.5 `cd eusolicit-app/services/client-api && pytest tests/api/test_calendar_connections.py -v` — all green.
  - [ ] 10.6 Local smoke (optional, non-blocking): `pnpm dev` and visit `http://localhost:3000/en/settings/calendar` while authenticated; verify three sections render, Connect anchors point at the right backend paths, sidebar shows the Calendar entry. Capture a screenshot in the dev log if convenient.

- [ ] **Task 11: Documentation polish. (non-blocking)**
  - [ ] 11.1 Add a short README at `app/[locale]/(protected)/settings/calendar/components/README.md` listing the four components (`CalendarConnectionsPage`, `GoogleCalendarSection`, `MicrosoftCalendarSection`, `ICalSection`), the testid contract, the i18n namespace, and the polling cadence. ~30 lines.

## Dev Notes

### Test Expectations (from Epic 09 Test Design)

Source: `eusolicit-docs/test-artifacts/test-design-epic-09.md` rows scoped to S09.13.

| Priority | Requirement | Level | Count | Owner | Test Harness |
|---|---|---|---|---|---|
| **P1** | Google Calendar connect button initiates OAuth; page shows connected state on return with `?connected=google` (sync timestamp + disconnect button visible) | Component / E2E | 3 | QA | Mock OAuth redirect; assert connected state with last-synced timestamp and disconnect button |
| **P1** | Disconnect shows confirmation dialog; confirm calls DELETE API and removes connection display | Component | 2 | Dev | Simulate disconnect click; assert dialog; confirm; assert DELETE API called |
| **P2** | iCal URL copy-to-clipboard calls `navigator.clipboard.writeText`; regenerate warning text present | Component | 2 | Dev | Assert clipboard API called on copy; assert warning text visible before regenerate |
| **P2** | Sync status 30-second poll updates last-synced display without full page reload | Component | 2 | Dev | `jest.useFakeTimers()`; advance 30s; assert API called; assert display updated |
| **P3** | Microsoft Outlook error state (expired token) displays reconnect CTA | E2E | 1 | QA | Seed expired-token connection; visit page; assert error state and reconnect button |

**Total Story 9.13 contribution from epic test design:** 10 tests (3 P1 + 2 P1 + 2 P2 + 2 P2 + 1 P3).

**This story's ATDD scope (Task 9):** the static (`environment: 'node'`) Vitest file alone — file existence, exports, testid contract, i18n parity, backend route + schema presence. The 10 component-render / E2E tests above are deferred to the TEA / `*automate` phase (consistent with how Stories 6.x, 7.x, 8.x, 9.12, 11.x, 12.x frontend stories handled component-tier coverage). The static ATDD file is a *non-negotiable gate* for the dev phase: it must be GREEN before this story moves out of `in-progress`.

**Coverage gates per the epic quality bar** (`test-design-epic-09.md` → "Coverage Targets" line 365):
- Frontend components (alert preferences page, calendar connection page): ≥ 70%. Story 9.12 owned the alert side; this story owns calendar.

**Risk links:**
- This story does NOT directly mitigate any high-priority epic-level risk (E09-R-001 through E09-R-004 are all backend / consumer / encryption / atomicity risks). It does indirectly **expose** the iCal token via the URL surfaced to the user (E09-R-012 — low priority): the token is already meant to be shared with calendar apps, but it should never appear in browser history search-bar-suggestions when the user is sharing their screen. **This story does not need to harden against E09-R-012** (it is documented as ops/log-scrubbing scope), but the Regenerate flow IS the user's mitigation if a token is ever leaked, so the warning copy on regenerate is intentionally explicit (AC8).
- E09-R-001 (Fernet key exposure) is out of scope here — this UI never sees plaintext tokens; the encrypted blobs live entirely in `client.calendar_connections`. The new `GET /connections` endpoint MUST NOT serialize `encrypted_access_token` or `encrypted_refresh_token` — the Pydantic response model in AC2 explicitly omits both fields, and the backend test in AC14 should assert the JSON response body does not contain the strings `"encrypted_access_token"` or `"encrypted_refresh_token"` for safety.

### Relevant Architecture Patterns & Constraints

1. **Next.js 14 App Router + `[locale]` segment** — the route lives under `app/[locale]/(protected)/settings/calendar/` and follows the same convention as every other protected page in the app: `page.tsx` (server) → `<FeaturePage>` (client). Do NOT introduce a new layout pattern; reuse the existing `(protected)/layout.tsx` chrome. This is identical to how Story 9.12 placed `/settings/alerts` and Story 8.13 placed `/settings/billing`.
2. **Auth handled by `<AuthGuard>` in `(protected)/layout.tsx`** — no new auth wiring needed in this story. If the user is not authenticated, the layout already redirects to `/login`.
3. **OAuth is a full-page navigation, not a `fetch`.** The Connect buttons MUST be `<a href="/api/v1/calendar/google/connect">` anchors — the backend sets a session cookie (`oauth_user_id`) and immediately 302s to Google. A `fetch()` would never complete the redirect chain because of CORS + the session-cookie binding requirement. See `calendar.py:180` docstring (Project Context Rule #39).
4. **TanStack Query is the canonical data layer.** Every read goes through a hook in `lib/queries/use-*.ts`. Cache key for this feature is `["calendar", "connections"]`. Mutations invalidate this exact tuple on success.
5. **`@eusolicit/ui` is the design system.** Pull `<Sheet>`, `<Card>`, `<Button>`, `<Skeleton>`, `<Alert>`, `<AlertDialog>`, `<Tooltip>`, `<Input>` from there. Do NOT install new component libraries.
6. **i18n via `next-intl`.** Every user-visible string goes through `t("calendar.connections.*")`. No hardcoded English in JSX. The `__tests__/i18n-setup.test.ts` parity check will fail CI if any key is missing from either locale file.
7. **`data-testid` is the QA contract.** Every interactive element exposes a stable `data-testid`. The static ATDD test file in Task 9 grep-checks for these strings in source.
8. **Polling pattern.** TanStack Query v5: `refetchInterval: 30_000` for the cadence, `refetchIntervalInBackground: false` to pause when tab is hidden. NO manual `setInterval`.
9. **Cross-schema query in the new backend endpoint.** The `notification.sync_log` table lives in the Notification Service's schema but is read-only-accessible from the `client_api` DB role per the test design (line 114 of `test-design-epic-09.md`). Use `sa.text(...)` if importing the SQLAlchemy model is awkward; comment the cross-schema read explicitly so future readers understand the boundary.
10. **The hardcoded `bg` locale in the OAuth callback redirect.** Backend at `calendar.py:208–209` and `:384–386` uses `f"{frontend_url}/bg/settings/calendar?..."` regardless of the user's actual locale. This is a pre-existing smell; do NOT fix it in this story (it would require adding locale propagation through the OAuth state param, which is out of scope and risks breaking the `oauth_user_id` session binding). The frontend must therefore work on `/bg/settings/calendar` (matches backend) AND `/en/settings/calendar` (mismatch). The `useSearchParams`-based toast handling in AC10 is locale-agnostic, so this Just Works™. Flag in Open Questions for product/PM follow-up.

### Source Tree Components to Touch

**New files:**
- `eusolicit-app/services/client-api/src/client_api/schemas/calendar_status.py` — Pydantic models (AC2)
- `eusolicit-app/services/client-api/tests/api/test_calendar_connections.py` — backend tests (AC14)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/page.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/CalendarConnectionsPage.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/GoogleCalendarSection.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/MicrosoftCalendarSection.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/ICalSection.tsx`
- `eusolicit-app/frontend/apps/client/lib/api/calendar.ts`
- `eusolicit-app/frontend/apps/client/lib/queries/use-calendar.ts`
- `eusolicit-app/frontend/apps/client/__tests__/calendar-connections-s9-13.test.ts`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/README.md` (Task 11.1, optional)

**Modified files:**
- `eusolicit-app/services/client-api/src/client_api/api/v1/calendar.py` — add `GET /connections` route only (do NOT touch existing routes)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx` — add `CalendarDays` import + nav entry (Task 7)
- `eusolicit-app/frontend/apps/client/messages/en.json` — add `calendar.connections.*` namespace + `nav.calendar` key (Task 8)
- `eusolicit-app/frontend/apps/client/messages/bg.json` — same key set, Bulgarian translations (Task 8)

**Files intentionally NOT modified:**
- `eusolicit-app/services/client-api/src/client_api/api/v1/calendar.py` lines 60–506 (the existing iCal + OAuth routes) — Stories 9.7/9.8/9.9 are `done`. Touch ONLY the new GET endpoint.
- `eusolicit-app/services/client-api/src/client_api/core/token_crypto.py` — Fernet helpers are correct; do NOT touch.
- `eusolicit-app/services/client-api/src/client_api/core/oauth.py` — OAuth client registration is correct; do NOT touch.
- `eusolicit-app/services/notification/**` — sync tasks own `notification.sync_log` writes; no consumer or task changes.
- `eusolicit-app/frontend/packages/ui/**` — no design system changes; reuse what is already exported.

### Testing Standards Summary

- **Vitest** for the static ATDD file (Task 9). `environment: 'node'` (no JSDOM needed for source-grep checks). Match the conventions in `__tests__/alert-preferences-s9-12.test.ts` (Story 9.12) — copy the helpers verbatim.
- **pytest** for the backend test (Task 1.4). Async fixtures from `services/client-api/tests/api/test_calendar.py`.
- **Component-tier tests** (the 10 P1/P2/P3 cases enumerated above) are deferred to the TEA `*automate` phase per the epic-09 plan. This story's static ATDD + the backend pytest are the gates.
- **i18n parity** is enforced by the existing `__tests__/i18n-setup.test.ts`. Run after Task 8.
- **Lint / type-check / build** must all pass before marking the story done (AC15). Per the Story 11.x / 12.x / 9.12 precedent.

### Previous Story Intelligence

**From Story 9.7 (iCal Feed Generation — backend, done):**
- The iCal feed endpoint is at `GET /api/v1/calendar/ical/{token}` (public, unauthenticated, returns `text/calendar`).
- Token rotation is a single endpoint: `POST /api/v1/calendar/ical/generate-token` (authenticated). It is upsert: if a row exists for `(user_id, provider='ical')`, the token is replaced atomically; otherwise inserted. Response: `{feed_url, token, rotated_at}` per `ICalTokenResponse` schema in `services/client-api/src/client_api/schemas/calendar_ical.py`.
- The `feed_url` is built server-side as `{api_public_base_url}/api/v1/calendar/ical/{token}`. The frontend MUST NOT reconstruct it from `token` alone — use the field as returned.
- Token is stored in `client.calendar_connections.token` (UUID column, unique-indexed). Old tokens after regeneration become orphaned (the column is overwritten); the OLD URL therefore returns 404 — that is the regenerate-warning UX in AC8.

**From Story 9.8 (Google Calendar OAuth2 + sync — backend, done):**
- OAuth flow: `GET /api/v1/calendar/google/connect` redirects to Google with `access_type=offline` (so refresh tokens are issued). `oauth_user_id` is bound to the Starlette session cookie BEFORE the redirect. Callback: `GET /api/v1/calendar/google/callback` reads `oauth_user_id` from the session, exchanges code, encrypts both access + refresh tokens with Fernet, stores in `client.calendar_connections` with `provider='google'`, then 302s to `{frontend_url}/bg/settings/calendar?connected=google` (success) or `?error=oauth_error&provider=google` (failure). **Note the hardcoded `/bg/`** — this is a known pre-existing smell, see Architecture Pattern #10 above.
- Disconnect: `DELETE /api/v1/calendar/google/disconnect` returns 204 on success, 404 if no connection exists. Treat 404 as success-equivalent in the UI per AC5.
- The Notification Service runs a Celery periodic task every 15 min that writes a `notification.sync_log` row per sync run with `events_created/updated/deleted` counts and `error_message`. The new `GET /connections` endpoint exposes this data via the `last_sync` field.
- Encryption key is `CALENDAR_ENCRYPTION_KEY`. The `connect` endpoint fails fast with 401 if the key is missing (`calendar.py:165–171`). The new `GET /connections` endpoint does NOT need to validate the key (it never decrypts anything — it just reports row presence).

**From Story 9.9 (Microsoft Outlook OAuth2 + sync — backend, done):**
- Mirror of the Google flow with `/microsoft/` paths. Identical contract. Microsoft revoke endpoint is NOT called on disconnect (per `calendar.py:489–491` "Microsoft does not expose a suitable revoke endpoint for MVP scopes") — disconnect just deletes the row. UI behavior is identical.

**From Story 9.12 (Alert Preferences Frontend Page — closest frontend precedent, done):**
- The page-component pattern, the `lib/api/*.ts` + `lib/queries/use-*.ts` split, the toast feedback recipe, the static ATDD test layout, the sidebar nav entry pattern, the i18n namespace structure — ALL of these are established there. Mirror them verbatim. The implementation report's File List for 9.12 is the recommended reference.
- The `__tests__/alert-preferences-s9-12.test.ts` file is the EXACT template for the Task 9 static ATDD file. Copy its helpers (APP_ROOT path constants, `readFile`, `readJSON`, `resolveNestedKey`) — do not re-derive them.
- The sidebar nav added the Bell/Alerts entry BEFORE Settings; this story adds Calendar BEFORE Alerts. Final order: ... → team → Calendar → Alerts → Settings.

**From Story 8.13 (Subscription Management Page — older frontend precedent, done):**
- Confirmation-dialog recipe (`<AlertDialog>`), card layout, react-query mutation feedback patterns. Useful secondary reference if 9.12's recipe is unclear on any point.

**From Story 3.11 (Toast notification system — done):**
- The toast helper is established and ready to use. Search `lib/` for the helper's import path; do not roll a new toast system.

**From the Project Context (`eusolicit-docs/project-context.md`):**
- Rule #39: OAuth callbacks bind identity via session cookie, never via JWT in URL. (Already enforced by the backend; UI just needs to use anchor navigation per AC4.)
- Rule #45: Audit-write failures are silently suppressed. (Not relevant for the UI, but be aware that the OAuth disconnect 404 path emits NO audit row — already handled by the backend.)
- Rule #180: Audit writes use a dedicated session in fire-and-forget mode. (Not relevant for the UI.)
- E09-R-001: Never log plaintext tokens. (Reinforces why the new `GET /connections` response excludes the encrypted-token byte fields.)

### Project Structure Notes

- The route `/{locale}/settings/calendar` slots in alongside the existing `/settings/alerts` (Story 9.12), `/settings/billing` (Story 8.13), `/settings/api-keys`, `/settings/company`, `/settings/reports`. Sibling pattern is uniform — a thin `page.tsx` server component delegating to a single `<FeaturePage>` client component.
- The new backend endpoint `GET /api/v1/calendar/connections` slots into the existing `Calendar` router prefix (`router = APIRouter(prefix="/calendar", tags=["Calendar"])`, `calendar.py:34`). No new router registration is needed in `main.py`.
- The Pydantic schema file `calendar_status.py` is new. It does NOT replace `calendar_ical.py` (Story 9.7) — both coexist. Keep them separate; do not consolidate.
- No new top-level dependencies in either the frontend `package.json` or the backend `pyproject.toml`. Every library used (TanStack Query, react-hook-form, zod, lucide-react, FastAPI, SQLAlchemy, Pydantic) is already present.

### References

- Epic file: `eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md` § S09.13 (lines 206–219)
- Test design: `eusolicit-docs/test-artifacts/test-design-epic-09.md` § P1/P2/P3 rows for S09.13 (lines 186–187, 215–216, 234)
- Backend reference: `eusolicit-app/services/client-api/src/client_api/api/v1/calendar.py` (full file — existing iCal + OAuth routes) [Source: services/client-api/src/client_api/api/v1/calendar.py]
- Backend schema reference: `eusolicit-app/services/client-api/src/client_api/schemas/calendar_ical.py` [Source: services/client-api/src/client_api/schemas/calendar_ical.py]
- Calendar connection model: `eusolicit-app/services/client-api/src/client_api/models/calendar_connection.py`
- Sync log model: `eusolicit-app/services/notification/src/notification/models/sync_log.py`
- Frontend precedent: `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/alerts/components/AlertPreferencesPage.tsx` (Story 9.12)
- Frontend API precedent: `eusolicit-app/frontend/apps/client/lib/api/alerts.ts` (Story 9.12)
- React Query precedent: `eusolicit-app/frontend/apps/client/lib/queries/use-alerts.ts` (Story 9.12)
- ATDD test precedent: `eusolicit-app/frontend/apps/client/__tests__/alert-preferences-s9-12.test.ts` (Story 9.12)
- Sidebar layout (will be modified): `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx`
- Clipboard recipe: `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/api-keys/components/ApiKeyManagementPage.tsx:100`
- Project Context Rules: `eusolicit-docs/project-context.md` (Rules #39, #45, #180)

## Open Questions

1. **Backend OAuth callback hardcodes `/bg/` locale.** `calendar.py:208–209` and `:384–386` redirect to `f"{frontend_url}/bg/settings/calendar?..."` regardless of which locale the user initiated `/connect` from. Result: an English-locale user clicking Connect lands back on the Bulgarian page. The frontend page in this story works in both locales (the toast handler in AC10 is locale-agnostic), so this is non-blocking — but it is a UX paper-cut that should be fixed in a follow-up by either (a) propagating the locale via the OAuth `state` param, or (b) reading the user's preferred locale from `users.locale` at callback time. **Decision needed from PM:** track as a separate enhancement story under Epic 9 or absorb into the next calendar-feature touch.
2. **iCal subscription-instructions translations.** The `howTo.google.steps`, `howTo.apple.steps`, `howTo.outlook.steps` arrays in Bulgarian require accurate UI labels matching what each app currently shows. Best-effort translations will ship; please verify with a native-speaker QA pass after the page is live.
3. **Microsoft "expired token" error state (test-design P3 row).** The current backend OAuth implementation does not surface a per-connection token-validity status — `connected=true` is purely "row exists in `client.calendar_connections`". A truly expired token is detected only at sync time and produces a `notification.sync_log.error_message`. This story's UI covers that path (AC5/AC6 sync-error alert) but does NOT add explicit "Reconnect" CTA wording for it (the existing Disconnect → Connect flow accomplishes the same thing). **Decision needed from PM:** is the sync-error alert sufficient, or do we want a dedicated "Token expired — reconnect" affordance? If the latter, scope a follow-up story.

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (dev-story phase — code review fixes pass)

### Debug Log References

### Completion Notes List

- Ultimate context engine analysis completed — comprehensive developer guide created.
- Code review fixes applied (2026-04-20): removed all `@ts-ignore` / `any` from TS files; localized hardcoded "How to subscribe" via `t("ical.howToHeader")`; dropped unused imports (`Calendar`, `error`, `dataUpdatedAt` from CalendarConnectionsPage; `CheckCircle2`, `AlertCircle` from ICalSection; `AlertDialogAction` from OAuthCalendarSection); rewrote backend test file with proper line lengths (ruff clean); added `queryClient.invalidateQueries` after AC10 success toasts; fixed `notSyncedYet` copy for connected-no-sync state; added `howToHeader` and `notSyncedYet` i18n keys to both locale files and ATDD test REQUIRED_I18N_KEYS. ATDD: 145/145 ✅, ruff: all checks passed ✅, next lint on calendar files: ✅.

### File List

**Modified (review fixes):**
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/ICalSection.tsx` — removed `@ts-ignore`, `any`-typed InfoIcon, unused imports; used `Info` from lucide-react; localized "How to subscribe"; typed `t.raw(...)` as `string[]`
- `eusolicit-app/frontend/apps/client/lib/api/calendar.ts` — `catch (error: any)` → `catch (error: unknown)` with safe type narrowing
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/CalendarConnectionsPage.tsx` — removed unused `Calendar` import + `error`/`dataUpdatedAt` destructures; added `queryClient.invalidateQueries` after AC10 success toasts; added `queryClient` to useEffect dep array
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/calendar/components/OAuthCalendarSection.tsx` — removed unused `AlertDialogAction`; fixed `staleData` → `notSyncedYet` for connected-no-sync state
- `eusolicit-app/services/client-api/tests/api/test_calendar_connections.py` — rewrote for ruff compliance (line length, imports, whitespace)
- `eusolicit-app/frontend/apps/client/messages/en.json` — added `howToHeader`, `notSyncedYet` keys
- `eusolicit-app/frontend/apps/client/messages/bg.json` — added `howToHeader`, `notSyncedYet` keys (Bulgarian)
- `eusolicit-app/frontend/apps/client/__tests__/calendar-connections-s9-13.test.ts` — added `howToHeader`, `notSyncedYet` to REQUIRED_I18N_KEYS (now 145/145)

## Senior Developer Review

**Outcome: REVIEW: Changes Requested**

**Reviewer:** Claude (autopilot code-review phase)
**Date:** 2026-04-20
**Scope:** AC1–AC15, full file walkthrough; ran ATDD test, ruff, lint, type-check, build.

### What works (no changes needed)

- **AC2 backend endpoint** is correct: `@router.get("/connections")` in `calendar.py` queries `client.calendar_connections`, then a single `DISTINCT ON` cross-schema query against `notification.sync_log`. Response model omits `encrypted_access_token`/`encrypted_refresh_token` per E09-R-001.
- **AC2/AC14 schema + tests:** `schemas/calendar_status.py` has all four classes; `tests/api/test_calendar_connections.py` covers the four required scenarios (all-three, only-google, 401, cross-user isolation).
- **AC3, AC9, AC10, AC11, AC12, AC13** are met — static ATDD passes (`__tests__/calendar-connections-s9-13.test.ts`: 141/141 ✅), i18n parity is clean (50 leaf keys per locale, `nav.calendar` present in both), polling cadence is 30 s with `refetchIntervalInBackground: false`, OAuth toast handler scrubs the URL, sidebar entry inserted before Alerts.
- **AC4–AC8** components render the right testids; AlertDialog-as-Dialog aliasing is documented as a deliberate workaround for missing `@eusolicit/ui` export.

### Blocking issues (AC15 explicit prohibitions and gate failures)

1. **`@ts-ignore` used three times** — `ICalSection.tsx:140, 155, 170` — AC15 explicitly forbids `@ts-ignore`. Replace with `@ts-expect-error` (and ideally type the array via `t.raw(...) as string[]`). ESLint flags these as errors.
2. **`any` types used** — AC15 forbids `any`:
   - `lib/api/calendar.ts:65, 80` — `catch (error: any)` in the disconnect helpers. Type as `unknown` and narrow via a type guard, or import an `AxiosError` type.
   - `ICalSection.tsx:192` — `function InfoIcon(props: any)`. Even better: drop the custom SVG — the parent already imports `Info` from `lucide-react`.
3. **Hardcoded English copy** — `ICalSection.tsx:130` — literal `"How to subscribe"` in JSX. Architecture Pattern #6 / AC12 require all visible strings go through `t(...)`. Add `calendar.connections.ical.howToHeader` to both locale files (and to the ATDD `REQUIRED_I18N_KEYS` list) and use `t("ical.howToHeader")`.
4. **Unused imports / variables — block `pnpm lint` and therefore `pnpm build` (AC15.3):**
   - `CalendarConnectionsPage.tsx`: `Calendar`, `error`, `dataUpdatedAt` (3 errors).
   - `ICalSection.tsx`: `CheckCircle2`, `AlertCircle` (and the unused-but-aliased `AlertDialog*` set if the alias chain is collapsed).
   - `OAuthCalendarSection.tsx`: `AlertDialogAction`.
5. **Backend ruff fails on the new test file** — AC15 / Task 10.4 lists `tests/api/test_calendar_connections.py` in the ruff target. 34 issues, mostly `W293` (blank line whitespace) plus a few `I001`/`F401` candidates. Run `ruff check --fix` then re-run.
6. **`pnpm build` exits non-zero** because of (4) — AC15.3 is therefore unmet. Build is gated on `next lint` clean.

### Non-blocking observations (please address but won't gate the story alone)

- **`OAuthCalendarSection.tsx:166`** — when `connected === true` but `last_sync` is null, the UI shows `t("staleData")` (intended for *failed-poll* state). This is misleading — a never-synced connection isn't stale data. Add a dedicated `calendar.connections.notSyncedYet` key or render an empty placeholder.
- **AC10 partial deviation** — story says "the query `useCalendarConnections()` is invalidated AFTER firing the success toast so the section flips to the connected state on next refetch." `CalendarConnectionsPage.tsx:42–61` only fires the toast and scrubs the URL; it relies on the natural 30 s poll. Add `queryClient.invalidateQueries({ queryKey: CALENDAR_QUERY_KEY })` inside the success branches so the connected card appears immediately after OAuth return.
- **OAuthCalendarSection re-uses the generic `Calendar` lucide icon for both Google and Microsoft.** Story permits this (AC4 explicitly allows `Calendar` icon as the brand-asset stand-in), but consider adding a `provider`-driven label-color tint so the two cards are visually distinct.
- **`ICalSection.handleGenerate`** uses `data.generated` to choose the toast (`regenerated` vs `generated`), but the toast fires *after* `mutateAsync()` resolves and the query is invalidated; there is no race that flips `data.generated`, but this depends on the parent re-render order. Capture `wasGenerated = data.generated` before `await` to avoid the subtle dependency.
- **AlertDialog-as-Dialog alias** is a defensible shortcut, but `disabled={isDisconnecting}` on the trigger means the dialog can't even open during the in-flight mutation — fine for the destructive case, but worth a code comment so future maintainers don't think it's a bug.

### Required actions before re-review

1. Remove all `@ts-ignore` and `any` from the four new TS files.
2. Localize "How to subscribe" header.
3. Drop unused imports/vars so `pnpm lint --filter client` is clean for the new files.
4. Run `ruff check --fix tests/api/test_calendar_connections.py` (and re-verify ruff passes for the two src files).
5. Confirm `pnpm build --filter client` passes.
6. (Optional but recommended) Invalidate `CALENDAR_QUERY_KEY` immediately after the AC10 success toast.
7. (Optional) Replace the misuse of `staleData` for the connected-no-sync state.

Once these are fixed, re-run the ATDD file (will still be green; add the new `howToHeader` key to `REQUIRED_I18N_KEYS`) and the backend pytest, and the story is good to merge.
