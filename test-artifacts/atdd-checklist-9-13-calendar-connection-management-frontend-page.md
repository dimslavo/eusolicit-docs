---
stepsCompleted: ['step-01-load-story', 'step-02-load-epic-test-design', 'step-03-map-acs', 'step-04-generate-tests', 'step-05-write-output']
lastStep: 'step-05-write-output'
lastSaved: '2026-04-20'
workflowType: 'testarch-atdd'
mode: 'story-level'
storyKey: '9-13-calendar-connection-management-frontend-page'
epicNumber: 9
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/9-13-calendar-connection-management-frontend-page.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-09.md'
  - 'eusolicit-app/frontend/apps/client/__tests__/alert-preferences-s9-12.test.ts'
testFile: 'eusolicit-app/frontend/apps/client/__tests__/calendar-connections-s9-13.test.ts'
status: 'RED — test file written; all tests fail until story is implemented'
---

# ATDD Checklist — Story 9.13: Calendar Connection Management Frontend Page

**Story:** S09.13 — Calendar Connection Management Frontend Page
**Epic:** E09 — Notifications, Alerts & Calendar
**Story Points:** 3
**Type:** Frontend + Thin Backend Endpoint
**Date:** 2026-04-20
**Status:** 🔴 RED — failing acceptance tests written; implementation not yet started

---

## Coverage Strategy (from Epic Test Design)

Source: `eusolicit-docs/test-artifacts/test-design-epic-09.md` — rows scoped to S09.13.

| Priority | Epic Test Design Row | Level | Count | Owner | Notes |
|---|---|---|---|---|---|
| **P1** | Google Calendar connect button initiates OAuth; page shows connected state on return with `?connected=google` (sync timestamp + disconnect button visible) | Component/E2E | 3 | QA | Mock OAuth redirect; assert connected state — DEFERRED to TEA |
| **P1** | Disconnect shows confirmation dialog; confirm calls DELETE API and removes connection display | Component | 2 | Dev | Simulate disconnect click — DEFERRED to TEA |
| **P2** | iCal URL copy-to-clipboard calls `navigator.clipboard.writeText`; regenerate warning text present | Component | 2 | Dev | Assert clipboard API called — DEFERRED to TEA |
| **P2** | Sync status 30-second poll updates last-synced display without full page reload | Component | 2 | Dev | `jest.useFakeTimers()` — DEFERRED to TEA |
| **P3** | Microsoft Outlook error state (expired token) displays reconnect CTA | E2E | 1 | QA | Seed expired-token connection — DEFERRED to TEA |

**ATDD scope for this story (static, `environment: 'node'`):**
File existence, export surface, `data-testid` string presence in source, backend route + schema presence (Python source grep), i18n key parity across both locales.
The 10 component-render/E2E tests (P1/P2/P3) are deferred to the TEA `*automate` phase — consistent with Stories 6.x, 7.x, 8.x, 9.12, 11.x, 12.x precedent.

**Coverage gate from epic test design:**
Frontend components (calendar connection page): ≥ 70% statement coverage — to be verified during TEA.

---

## Acceptance Criteria → Test Mapping

### AC1 — Route + page scaffold

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 1.1 | `settings/calendar/page.tsx` exists | File existence | ✅ exists |
| 1.2 | `page.tsx` imports `CalendarConnectionsPage` | Source grep | contains `CalendarConnectionsPage` |
| 1.3 | `settings/calendar/components/CalendarConnectionsPage.tsx` exists | File existence | ✅ exists |
| 1.4 | `CalendarConnectionsPage.tsx` has `"use client"` directive | Source grep | starts with `"use client"` |
| 1.5 | `CalendarConnectionsPage.tsx` has `data-testid="calendar-connections-page"` | Source grep | string present |
| 1.6 | `CalendarConnectionsPage.tsx` has `data-testid="calendar-connections-page-title"` | Source grep | string present |

---

### AC2 — Backend endpoint + Pydantic schema file

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 2.1 | `services/client-api/src/client_api/api/v1/calendar.py` has `@router.get("/connections")` | Python source grep | string present |
| 2.2 | `services/client-api/src/client_api/schemas/calendar_status.py` exists | File existence | ✅ exists |
| 2.3 | `calendar_status.py` has `class SyncSummary` | Python source grep | class defined |
| 2.4 | `calendar_status.py` has `class ConnectionStatus` | Python source grep | class defined |
| 2.5 | `calendar_status.py` has `class ICalStatus` | Python source grep | class defined |
| 2.6 | `calendar_status.py` has `class CalendarConnectionsResponse` | Python source grep | class defined |
| 2.7 | `calendar_status.py` does NOT contain `encrypted_access_token` | Security check | field absent (E09-R-001) |
| 2.8 | `calendar_status.py` does NOT contain `encrypted_refresh_token` | Security check | field absent (E09-R-001) |

---

### AC3 — Loading / error states

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 3.1 | `CalendarConnectionsPage.tsx` has `data-testid="calendar-connections-loading"` | Source grep | string present |
| 3.2 | Loading state renders `<Skeleton>` placeholders | Source grep | contains `Skeleton` |
| 3.3 | `CalendarConnectionsPage.tsx` has `data-testid="calendar-connections-error"` | Source grep | string present |
| 3.4 | Error state uses `Alert` with `variant="destructive"` | Source grep | contains `variant="destructive"` |
| 3.5 | `CalendarConnectionsPage.tsx` has `data-testid="calendar-connections-stale-indicator"` for polling failure | Source grep | string present |
| 3.6 | Page renders all three section components (`GoogleCalendarSection`, `MicrosoftCalendarSection`, `ICalSection`) | Source grep | all three names present |

---

### AC4 + AC5 — Google section

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 4.1 | `settings/calendar/components/GoogleCalendarSection.tsx` exists | File existence | ✅ exists |
| 4.2 | `GoogleCalendarSection.tsx` has `data-testid="calendar-section-google"` | Source grep | string present |
| 4.3 | `GoogleCalendarSection.tsx` has `data-testid="calendar-google-connect-btn"` | Source grep | string present |
| 4.4 | Connect element is an `<a>` anchor with `href="/api/v1/calendar/google/connect"` (NOT a `<button>` or `fetch()`) | Source grep | href present in source; no `fetch` for this action |
| 4.5 | Connect anchor does NOT have `target="_blank"` | Source grep | `target="_blank"` absent near connect button |
| 4.6 | `GoogleCalendarSection.tsx` has `data-testid="calendar-google-connected-at"` | Source grep | string present |
| 4.7 | `GoogleCalendarSection.tsx` has `data-testid="calendar-google-last-synced"` | Source grep | string present |
| 4.8 | `GoogleCalendarSection.tsx` has event-count testid template for `created` | Source grep | `calendar-google-events-created` pattern present |
| 4.9 | `GoogleCalendarSection.tsx` has event-count testid template for `updated` | Source grep | `calendar-google-events-updated` pattern present |
| 4.10 | `GoogleCalendarSection.tsx` has event-count testid template for `deleted` | Source grep | `calendar-google-events-deleted` pattern present |
| 4.11 | `GoogleCalendarSection.tsx` has `data-testid="calendar-google-sync-error"` for last-sync error | Source grep | string present |
| 4.12 | `GoogleCalendarSection.tsx` has `data-testid="calendar-google-disconnect-btn"` | Source grep | string present |
| 4.13 | `GoogleCalendarSection.tsx` has `data-testid="calendar-google-disconnect-dialog"` | Source grep | string present |
| 4.14 | Disconnect uses `<AlertDialog>` from `@eusolicit/ui` | Source grep | contains `AlertDialog` |

---

### AC6 — Microsoft section

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 6.1 | `settings/calendar/components/MicrosoftCalendarSection.tsx` exists | File existence | ✅ exists |
| 6.2 | `MicrosoftCalendarSection.tsx` has `data-testid="calendar-section-microsoft"` | Source grep | string present |
| 6.3 | `MicrosoftCalendarSection.tsx` has `data-testid="calendar-microsoft-connect-btn"` | Source grep | string present |
| 6.4 | Connect element href is `/api/v1/calendar/microsoft/connect` | Source grep | href present in source |
| 6.5 | `MicrosoftCalendarSection.tsx` has `data-testid="calendar-microsoft-connected-at"` | Source grep | string present |
| 6.6 | `MicrosoftCalendarSection.tsx` has `data-testid="calendar-microsoft-last-synced"` | Source grep | string present |
| 6.7 | `MicrosoftCalendarSection.tsx` has event-count testid for `created` | Source grep | `calendar-microsoft-events-created` pattern present |
| 6.8 | `MicrosoftCalendarSection.tsx` has event-count testid for `updated` | Source grep | `calendar-microsoft-events-updated` pattern present |
| 6.9 | `MicrosoftCalendarSection.tsx` has event-count testid for `deleted` | Source grep | `calendar-microsoft-events-deleted` pattern present |
| 6.10 | `MicrosoftCalendarSection.tsx` has `data-testid="calendar-microsoft-disconnect-btn"` | Source grep | string present |
| 6.11 | `MicrosoftCalendarSection.tsx` has `data-testid="calendar-microsoft-disconnect-dialog"` | Source grep | string present |

---

### AC7 + AC8 — iCal section

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 7.1 | `settings/calendar/components/ICalSection.tsx` exists | File existence | ✅ exists |
| 7.2 | `ICalSection.tsx` has `data-testid="calendar-section-ical"` | Source grep | string present |
| 7.3 | `ICalSection.tsx` has `data-testid="calendar-ical-generate-btn"` | Source grep | string present |
| 7.4 | `ICalSection.tsx` has `data-testid="calendar-ical-url-input"` on a `readOnly` input | Source grep | string present |
| 7.5 | URL input uses `readOnly` attribute (NOT `disabled`) | Source grep | contains `readOnly` |
| 7.6 | `ICalSection.tsx` has `data-testid="calendar-ical-copy-btn"` | Source grep | string present |
| 7.7 | Copy button calls `navigator.clipboard.writeText` | Source grep | contains `navigator.clipboard` |
| 7.8 | Clipboard API call is wrapped in try/catch (does not crash on error) | Source grep | contains `try` near `clipboard` |
| 7.9 | `ICalSection.tsx` has `data-testid="calendar-ical-rotated-at"` | Source grep | string present |
| 7.10 | `ICalSection.tsx` has `data-testid="calendar-ical-instructions"` on a `<details>` element | Source grep | string present |
| 7.11 | Instructions block references all three calendar providers: Google Calendar, Apple Calendar, Outlook | Source grep | `howTo.google`, `howTo.apple`, `howTo.outlook` keys present |
| 7.12 | `ICalSection.tsx` has `data-testid="calendar-ical-regenerate-btn"` | Source grep | string present |
| 7.13 | `ICalSection.tsx` has `data-testid="calendar-ical-regenerate-dialog"` | Source grep | string present |
| 7.14 | Regenerate uses `<AlertDialog>` from `@eusolicit/ui` | Source grep | contains `AlertDialog` |

---

### AC9 — Polling

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 9.1 | `lib/queries/use-calendar.ts` exists | File existence | ✅ exists |
| 9.2 | `use-calendar.ts` has `"use client"` directive | Source grep | starts with `"use client"` |
| 9.3 | `useCalendarConnections` uses `refetchInterval: 30_000` | Source grep | `30_000` or `30000` present |
| 9.4 | `useCalendarConnections` sets `refetchIntervalInBackground: false` | Source grep | string present |
| 9.5 | Cache key is `["calendar", "connections"]` | Source grep | both `"calendar"` and `"connections"` present |

---

### AC10 — OAuth callback toast handling

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 10.1 | `CalendarConnectionsPage.tsx` uses `useSearchParams` | Source grep | contains `useSearchParams` |
| 10.2 | `CalendarConnectionsPage.tsx` uses `useEffect` with `searchParams` as dependency | Source grep | `searchParams` in deps array |
| 10.3 | `CalendarConnectionsPage.tsx` calls `router.replace` to scrub query params after reading them | Source grep | contains `router.replace` |
| 10.4 | Page handles `?connected=google` and `?connected=microsoft` callback params | Source grep | both `connected=google`-style strings or `connected` param check present |
| 10.5 | Page handles `?error=oauth_error` callback param | Source grep | `oauth_error` string present |

---

### AC11 — Sidebar navigation entry

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 11.1 | `app/[locale]/(protected)/layout.tsx` imports `CalendarDays` from `lucide-react` | Source grep | `CalendarDays` + `lucide-react` present |
| 11.2 | Layout has `testId: "sidebar-nav-calendar"` in nav items | Source grep | string present |
| 11.3 | Layout nav item href contains `settings/calendar` | Source grep | string present |
| 11.4 | Calendar entry appears BEFORE the Alerts (`sidebar-nav-alerts`) entry in source order | Source grep | `sidebar-nav-calendar` occurs before `sidebar-nav-alerts` |

---

### AC12 — i18n keys

All keys below must be present in both `messages/en.json` and `messages/bg.json` with non-empty values.

**Required keys (~50 leaf keys, excluding array-typed `howTo.*.steps`):**

```
calendar.connections.pageTitle
calendar.connections.pageDescription
calendar.connections.staleData
calendar.connections.genericError
calendar.connections.connectedAt
calendar.connections.lastSynced
calendar.connections.eventsCreated
calendar.connections.eventsUpdated
calendar.connections.eventsDeleted
calendar.connections.disconnectBtn
calendar.connections.cancelBtn
calendar.connections.retryBtn
calendar.connections.google.title
calendar.connections.google.disconnectedBody
calendar.connections.google.connectBtn
calendar.connections.google.connected
calendar.connections.google.disconnected
calendar.connections.google.oauthError
calendar.connections.google.disconnectConfirmTitle
calendar.connections.google.disconnectConfirmDescription
calendar.connections.google.disconnectConfirmBtn
calendar.connections.microsoft.title
calendar.connections.microsoft.disconnectedBody
calendar.connections.microsoft.connectBtn
calendar.connections.microsoft.connected
calendar.connections.microsoft.disconnected
calendar.connections.microsoft.oauthError
calendar.connections.microsoft.disconnectConfirmTitle
calendar.connections.microsoft.disconnectConfirmDescription
calendar.connections.microsoft.disconnectConfirmBtn
calendar.connections.ical.title
calendar.connections.ical.notGeneratedBody
calendar.connections.ical.generateBtn
calendar.connections.ical.regenerateBtn
calendar.connections.ical.regenerated
calendar.connections.ical.copied
calendar.connections.ical.copyFailed
calendar.connections.ical.urlLabel
calendar.connections.ical.regenerateWarning
calendar.connections.ical.regenerateConfirmTitle
calendar.connections.ical.regenerateConfirmDescription
calendar.connections.ical.regenerateConfirmBtn
calendar.connections.ical.rotatedAt
calendar.connections.ical.howTo.google.title
calendar.connections.ical.howTo.apple.title
calendar.connections.ical.howTo.outlook.title
nav.calendar
```

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 12.1 | `messages/en.json` exists | File existence | ✅ exists |
| 12.2 | `messages/bg.json` exists | File existence | ✅ exists |
| 12.3 | en.json has `calendar.connections` namespace | JSON key check | defined |
| 12.4 | bg.json has `calendar.connections` namespace | JSON key check | defined |
| 12.5 | en.json has all required leaf keys | JSON key check (it.each) | all defined and non-empty |
| 12.6 | bg.json has all required leaf keys | JSON key check (it.each) | all defined and non-empty |
| 12.7 | en.json and bg.json have the same top-level key count under `calendar.connections` | Structural parity | counts match |

---

### AC13 — ATDD coverage (static)

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 13.1 | `__tests__/calendar-connections-s9-13.test.ts` exists | File existence | ✅ exists |
| 13.2 | Test file has ≥ 20 `it(...)` blocks | Structural | count ≥ 20 |

---

## Additional Static Checks — API module + React Query hooks

### lib/api/calendar.ts

| # | Check | Type | Expected outcome |
|---|---|---|---|
| A.1 | `lib/api/calendar.ts` exists | File existence | ✅ exists |
| A.2 | Exports `CalendarConnectionsResponse` type | Source grep | string present |
| A.3 | Exports `ConnectionStatus` type | Source grep | string present |
| A.4 | Exports `SyncSummary` type | Source grep | string present |
| A.5 | Exports `ICalStatus` type | Source grep | string present |
| A.6 | Exports `getCalendarConnections` function | Source grep | string present |
| A.7 | Exports `generateIcalToken` function | Source grep | string present |
| A.8 | Exports `disconnectGoogle` function | Source grep | string present |
| A.9 | Exports `disconnectMicrosoft` function | Source grep | string present |
| A.10 | Uses `apiClient` (same pattern as `lib/api/alerts.ts`) | Source grep | `apiClient` present |

### lib/queries/use-calendar.ts (extended)

| # | Check | Type | Expected outcome |
|---|---|---|---|
| Q.1 | Exports `useGenerateIcalToken` mutation | Source grep | string present |
| Q.2 | Exports `useDisconnectGoogle` mutation | Source grep | string present |
| Q.3 | Exports `useDisconnectMicrosoft` mutation | Source grep | string present |
| Q.4 | All mutations call `invalidateQueries` on success | Source grep | contains `invalidateQueries` |

---

## Test File Location

```
eusolicit-app/frontend/apps/client/__tests__/calendar-connections-s9-13.test.ts
```

**Run command:**
```bash
cd eusolicit-app/frontend && pnpm test --filter client -- calendar-connections-s9-13
```

---

## Coverage Summary

| AC | Describe Block | Static Checks | Component (TEA) |
|---|---|---|---|
| AC1 | Route + page scaffold | 6 | — |
| AC2 | Backend endpoint + Pydantic schema | 8 | — |
| AC3 | Loading / error states | 6 | — |
| AC4+AC5 | Google section | 14 | 5 (P1) |
| AC6 | Microsoft section | 11 | — |
| AC7+AC8 | iCal section | 14 | 2 (P2) |
| AC9 | Polling | 5 | 2 (P2) |
| AC10 | OAuth callback toast handling | 5 | — |
| AC11 | Sidebar nav entry | 4 | — |
| AC12 | i18n parity | 7 + it.each | — |
| AC13 | Test file structure | 2 | — |
| lib/api + hooks | API module + React Query | 14 | — |
| **Total** | | **≥ 55 `it(...)` blocks** | **10 deferred to TEA** |

---

## Pass Criteria

- 🔴 **RED** (now): All static `it(...)` blocks fail because none of the target files exist.
- 🟢 **GREEN** (after implementation): All static `it(...)` blocks pass. Developer may not mark story `done` until this file is fully green.
- TEA gate: ≥ 70% component coverage achieved via the 10 deferred component/E2E tests in the TEA `*automate` phase.

---

## Related Artefacts

- Story file: `eusolicit-docs/implementation-artifacts/9-13-calendar-connection-management-frontend-page.md`
- Epic test design: `eusolicit-docs/test-artifacts/test-design-epic-09.md` (§S09.13 rows at lines 186–187, 215–216, 234)
- ATDD precedent (S9.12): `eusolicit-docs/test-artifacts/atdd-checklist-9-12-alert-preferences-frontend-page.md`
- Test file precedent: `eusolicit-app/frontend/apps/client/__tests__/alert-preferences-s9-12.test.ts`
- i18n parity test: `eusolicit-app/frontend/apps/client/__tests__/i18n-setup.test.ts`
- Backend file to check: `eusolicit-app/services/client-api/src/client_api/api/v1/calendar.py`
- Backend schema to check: `eusolicit-app/services/client-api/src/client_api/schemas/calendar_status.py`
