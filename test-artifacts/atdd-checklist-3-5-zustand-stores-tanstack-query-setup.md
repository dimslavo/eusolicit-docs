---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-08'
workflowType: atdd
storyId: 3-5-zustand-stores-tanstack-query-setup
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/3-5-zustand-stores-tanstack-query-setup.md
  - eusolicit-docs/test-artifacts/test-design-epic-03.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/frontend/apps/client/lib/stores/ui-store.ts
  - eusolicit-docs/test-artifacts/atdd-checklist-3-4-responsive-layout-strategy.md
---

# ATDD Checklist: Story 3.5 — Zustand Stores & TanStack Query Setup

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated; feature not yet implemented
**Story:** `eusolicit-docs/implementation-artifacts/3-5-zustand-stores-tanstack-query-setup.md`
**Epic:** E03 — Frontend Shell & Design System

---

## Step 1: Preflight & Context

| Item | Status | Notes |
|------|--------|-------|
| Story has clear acceptance criteria | ✅ | 7 ACs; full implementation specs in Dev Notes |
| Playwright config found | ✅ | `eusolicit-app/playwright.config.ts` |
| Detected stack | ✅ | `frontend` — Next.js 14 App Router, Playwright E2E |
| Test artifacts directory | ✅ | `eusolicit-docs/test-artifacts/` |
| Epic test design loaded | ✅ | `test-design-epic-03.md` — E03-P0-007, P0-008, P1-001, P1-002, P1-012, P2-008 |
| Prior story ATDD loaded | ✅ | S3.4 checklist reviewed for patterns; E2E test structure consistent |
| Unit test framework | ⚠️ | Vitest required for unit tests on stores/hooks/apiClient — **not yet configured** (see prerequisites) |

### Unit Test Prerequisites (Before Unskipping Vitest Tests)

The following devDependencies must be added and vitest.config.ts files created before the unit tests can run:

**For `packages/ui` unit tests (auth-store, api-client, useSSE):**
```bash
# From eusolicit-app/frontend/
pnpm add -D vitest @vitest/globals @testing-library/react jsdom \
  @vitejs/plugin-react axios-mock-adapter \
  --filter @eusolicit/ui
```

**For `apps/client` unit tests (ui-store):**
```bash
pnpm add -D vitest @vitest/globals --filter client
```

**Create `eusolicit-app/frontend/packages/ui/vitest.config.ts`:**
```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
export default defineConfig({
  plugins: [react()],
  test: {
    globals:     true,
    environment: 'node', // default; jsdom overridden per-file for hook tests
    environmentMatchGlobs: [
      ['**/__tests__/lib/hooks/**', 'jsdom'],
    ],
  },
});
```

**Create `eusolicit-app/frontend/apps/client/vitest.config.ts`:**
```typescript
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: { globals: true, environment: 'node' },
});
```

---

## Step 2: Generation Mode

**Mode:** AI Generation (no browser recording — all target modules non-existent)
**Reason:** All files being tested (`auth-store.ts`, `api-client.ts`, `useSSE.ts`, `ui-store.ts` expansions,
`/dev/api-test` page) do not exist yet. Browser recording cannot capture selectors from pages
that return 404. All selectors and assertions derived from story AC specifications and Dev Notes.

---

## Step 3: Test Strategy

### AC → Test Mapping

| AC | Requirement Summary | Test Level | Priority | Epic Test ID | File |
|----|---------------------|-----------|----------|--------------|------|
| AC1 | authStore: fields, methods, localStorage persistence | Unit (Vitest) | P1 | E03-P1-001 | `auth-store.test.ts` |
| AC1 | authStore: rehydration restores isAuthenticated | Unit (Vitest) | P1 | E03-P1-001 | `auth-store.test.ts` |
| AC2 | uiStore: existing sidebar fields preserved exactly | Unit (Vitest) | P1 | E03-P1-002 | `ui-store.test.ts` |
| AC2 | uiStore: new theme/locale/toasts fields and actions | Unit (Vitest) | P1 | — | `ui-store.test.ts` |
| AC2 | uiStore: partialize — only sidebarCollapsed+locale persisted | Unit (Vitest) | P1 | E03-P1-002 | `ui-store.test.ts` |
| AC3 | QueryProvider available (validates integration with AC6) | E2E | P1 | E03-P1-012 | `api-test-page.spec.ts` |
| AC4 | apiClient: JWT Bearer header attached | Unit (Vitest) | P1 | — | `api-client.test.ts` |
| AC4 | apiClient: 401 dedup — 3 concurrent → 1 refresh | Unit (Vitest) | P0 | E03-P0-007 | `api-client.test.ts` |
| AC4 | apiClient: refresh failure → logout(); no loop | Unit (Vitest) | P0 | E03-P0-008 | `api-client.test.ts` |
| AC4 | apiClient: _retry flag prevents infinite loop | Unit (Vitest) | P0 | E03-P0-008 | `api-client.test.ts` |
| AC4 | apiClient: non-401 errors pass through | Unit (Vitest) | P1 | — | `api-client.test.ts` |
| AC5 | useSSE: EventSource.close() called on unmount | Unit (Vitest) | P2 | E03-P2-008 | `useSSE.test.ts` |
| AC5 | useSSE: status transitions (connecting → open) | Unit (Vitest) | P2 | — | `useSSE.test.ts` |
| AC5 | useSSE: data updates on message; JSON parsed | Unit (Vitest) | P2 | — | `useSSE.test.ts` |
| AC5 | useSSE: error event sets status="error" | Unit (Vitest) | P2 | — | `useSSE.test.ts` |
| AC5 | useSSE: null url → no EventSource created | Unit (Vitest) | P2 | — | `useSSE.test.ts` |
| AC6 | /dev/api-test page heading visible | E2E | P1 | E03-P1-012 | `api-test-page.spec.ts` |
| AC6 | /dev/api-test shows loading state | E2E | P1 | E03-P1-012 | `api-test-page.spec.ts` |
| AC6 | /dev/api-test shows success state (GET /health 200) | E2E | P1 | E03-P1-012 | `api-test-page.spec.ts` |
| AC6 | /dev/api-test shows error state (GET /health 500) | E2E | P1 | E03-P1-012 | `api-test-page.spec.ts` |
| AC7 | `pnpm build` exits 0 both apps | CI gate | P0 | E03-P0-001 | _(existing CI gate — no new test)_ |
| AC7 | `pnpm type-check` exits 0 all packages | CI gate | P0 | E03-P0-001 | _(existing CI gate — no new test)_ |

**Note on AC3 (QueryProvider config):** The QueryClient defaults (staleTime: 30k, gcTime: 300k, retry: 1,
refetchOnWindowFocus: true) are validated indirectly through E2E test T02 (loading state visible) — if
QueryProvider is missing from the root layout, the `useHealthCheck` hook has no QueryClient context
and the page throws, making T02 impossible to pass. Direct unit testing of QueryClient defaults is
deferred to the `automate` phase when the provider is wired in.

**Note on AC7 (exports + build gate):** `E03-P0-001` is an active CI gate established in Story 3.1.
All new exports from `packages/ui/index.ts` are validated by the TypeScript compiler via `pnpm type-check`.
No separate new test file is generated for AC7.

---

## Step 4: Generated Test Files (TDD RED PHASE)

### 🔴 File 1: `eusolicit-app/frontend/packages/ui/src/__tests__/stores/auth-store.test.ts`

**Framework:** Vitest
**Covers:** AC1 / E03-P1-001

| Test ID | it.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|----------------------|----------|-----|---------|----------------|
| T01 | initial state: all null / isAuthenticated=false | P1 | AC1 | — | `auth-store.ts` doesn't exist → import error |
| T02 | login() sets user, token, refreshToken, isAuthenticated=true | P1 | AC1 | — | `auth-store.ts` doesn't exist → import error |
| T03 | logout() clears all state | P1 | AC1 | E03-P1-001 | `auth-store.ts` doesn't exist → import error |
| T04 | setUser() updates user without changing tokens | P1 | AC1 | — | `auth-store.ts` doesn't exist → import error |
| T05 | setTokens() updates tokens and sets isAuthenticated=true | P1 | AC1 | — | `auth-store.ts` doesn't exist → import error |
| T06 | E03-P1-001: login() persists to localStorage key "eusolicit-auth-store" | P1 | AC1 | E03-P1-001 | `auth-store.ts` doesn't exist; persist key wrong |
| T07 | E03-P1-001: isAuthenticated=true restored after rehydration | P1 | AC1 | E03-P1-001 | `auth-store.ts` doesn't exist; `persist.rehydrate()` missing |

**Total:** 7 tests (all `it.skip()`)

---

### 🔴 File 2: `eusolicit-app/frontend/apps/client/lib/stores/__tests__/ui-store.test.ts`

**Framework:** Vitest
**Covers:** AC2 / E03-P1-002

| Test ID | it.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|----------------------|----------|-----|---------|----------------|
| T01 | existing: sidebarCollapsed defaults to false | P1 | AC2 | E03-P1-002 | `theme/locale/toasts` do not exist on UIState (TS error) |
| T02 | existing: toggleSidebar() inverts sidebarCollapsed | P1 | AC2 | E03-P1-002 | TS error on UIState due to missing new fields |
| T03 | existing: setSidebarCollapsed(true/false) sets explicitly | P1 | AC2 | E03-P1-002 | TS error on UIState |
| T04 | new: theme defaults to "system" | P1 | AC2 | — | `theme` property missing from UIState |
| T05 | new: locale defaults to "bg" | P1 | AC2 | — | `locale` property missing from UIState |
| T06 | new: toasts defaults to [] | P1 | AC2 | — | `toasts` property missing from UIState |
| T07 | new: addToast() appends with auto-generated id | P1 | AC2 | — | `addToast` method missing |
| T08 | new: removeToast() removes targeted toast by id | P1 | AC2 | — | `removeToast` method missing |
| T09 | E03-P1-002: only sidebarCollapsed+locale persisted; theme+toasts excluded | P1 | AC2 | E03-P1-002 | `partialize` not applied → all state including toasts persisted |
| T10 | E03-P1-002: sidebarCollapsed+locale rehydrate from localStorage | P1 | AC2 | E03-P1-002 | `persist.rehydrate()` missing; `devtools` not applied |

**Total:** 10 tests (all `it.skip()`)

---

### 🔴 File 3: `eusolicit-app/frontend/packages/ui/src/__tests__/lib/api-client.test.ts`

**Framework:** Vitest + axios-mock-adapter
**Covers:** AC4 / E03-P0-007 / E03-P0-008

| Test ID | it.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|----------------------|----------|-----|---------|----------------|
| T01 | request interceptor sets Authorization: Bearer <token> | P1 | AC4 | — | `api-client.ts` doesn't exist → import error |
| T02 | request interceptor omits header when token=null | P1 | AC4 | — | `api-client.ts` doesn't exist → import error |
| T03 | E03-P0-007: 3 concurrent 401s → POST /auth/refresh called ONCE → all 3 succeed | P0 | AC4 | E03-P0-007 | `api-client.ts` doesn't exist; `refreshPromise` singleton not implemented |
| T04 | E03-P0-008: POST /auth/refresh 401 → authStore.logout() called; no retry loop | P0 | AC4 | E03-P0-008 | `api-client.ts` doesn't exist; `catch { logout(); throw }` not implemented |
| T05 | _retry=true prevents interceptor re-entry (infinite loop guard) | P0 | AC4 | E03-P0-008 | `api-client.ts` doesn't exist; `_retry` check missing |
| T06 | non-401 errors pass through without refresh attempt | P1 | AC4 | — | `api-client.ts` doesn't exist → import error |

**Total:** 6 tests (all `it.skip()`)

---

### 🔴 File 4: `eusolicit-app/frontend/packages/ui/src/__tests__/lib/hooks/useSSE.test.ts`

**Framework:** Vitest + @testing-library/react (jsdom environment)
**Covers:** AC5 / E03-P2-008

| Test ID | it.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|----------------------|----------|-----|---------|----------------|
| T01 | E03-P2-008: EventSource.close() called on component unmount | P2 | AC5 | E03-P2-008 | `useSSE.ts` doesn't exist; no useEffect cleanup |
| T02 | status: "connecting" on mount → "open" after onopen fires | P2 | AC5 | — | `useSSE.ts` doesn't exist; status not tracked |
| T03 | data updates on message; JSON auto-parsed | P2 | AC5 | — | `useSSE.ts` doesn't exist; no onmessage handler |
| T04 | status becomes "error" on onerror event | P2 | AC5 | — | `useSSE.ts` doesn't exist; no onerror handler |
| T05 | null url → no EventSource created; status stays "closed" | P2 | AC5 | — | `useSSE.ts` doesn't exist; null guard not implemented |

**Total:** 5 tests (all `it.skip()`)

---

### 🔴 File 5: `eusolicit-app/e2e/specs/smoke/api-test-page.spec.ts`

**Framework:** Playwright
**Playwright projects:** `client-chromium`, `client-firefox`, `client-webkit`
**baseURL:** `http://localhost:3000`
**Covers:** AC6 / E03-P1-012

| Test ID | test.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|------------------------|----------|-----|---------|----------------|
| T01 | page heading visible after navigation to /dev/api-test | P1 | AC6 | E03-P1-012 | `apps/client/app/dev/api-test/page.tsx` missing → 404 |
| T02 | loading state "Checking /health…" visible while GET /health is pending | P1 | AC6 | E03-P1-012 | Page missing; QueryProvider not wired → query has no context |
| T03 | success state — GET /health 200 → JSON rendered in green pre block | P1 | AC6 | E03-P1-012 | Page missing; `useHealthCheck` query not implemented |
| T04 | error state — GET /health 500 → error message rendered in red pre block | P1 | AC6 | E03-P1-012 | Page missing; error branch never rendered |

**Total:** 4 tests (all `test.skip()`)

---

## Step 4C: Aggregate Summary

| Metric | Value |
|--------|-------|
| TDD Phase | 🔴 RED |
| Total tests generated | 32 |
| Vitest unit tests | 28 (Files 1–4) |
| Playwright E2E tests | 4 (File 5) |
| All tests use skip() | ✅ `it.skip()` (Vitest) / `test.skip()` (Playwright) |
| All tests assert expected behavior | ✅ |
| Placeholder assertions (`expect(true).toBe(true)`) | 0 |
| Execution mode | SEQUENTIAL |

### Priority Distribution

| Priority | Unit | E2E | Total |
|----------|------|-----|-------|
| P0 | 3 | 0 | 3 |
| P1 | 21 | 4 | 25 |
| P2 | 5 | 0 | 5 |
| P3 | 0 | 0 | 0 |
| **Total** | **28** | **4** | **32** |

**Note on P0:** 3 unit tests (T03, T04, T05 in api-client.test.ts) map directly to E03-P0-007 and
E03-P0-008 — the highest-priority risks in Epic 3 (E03-R-003: 401 refresh deduplication, score 6).
The AC7 build and type-check gate (E03-P0-001) is covered by the existing CI gate.

### AC Coverage

| AC | Tests Covering | Files |
|----|----------------|-------|
| AC1 | T01–T07 | auth-store.test.ts |
| AC2 | T01–T10 | ui-store.test.ts |
| AC3 | T02 (indirect: QueryProvider wired) | api-test-page.spec.ts |
| AC4 | T01–T06 | api-client.test.ts |
| AC5 | T01–T05 | useSSE.test.ts |
| AC6 | T01–T04 | api-test-page.spec.ts |
| AC7 | _(existing E03-P0-001 CI gate)_ | CI |

**All 7 ACs have test coverage. AC7 covered by existing CI gate.**

---

## Step 5: Validation

### Validation Checklist

- [x] Story has approved acceptance criteria (7 ACs, all clearly specified with implementation notes)
- [x] All 7 ACs have test coverage (AC7 via existing CI build gate; AC1–AC6 via new tests)
- [x] All Vitest tests use `it.skip()` (TDD red phase)
- [x] All Playwright tests use `test.skip()` (TDD red phase)
- [x] No placeholder assertions (`expect(true).toBe(true)`) — zero instances
- [x] Tests assert EXPECTED behavior (what will work once implemented)
- [x] No orphaned browser sessions (no Playwright CLI recording needed)
- [x] Unit test files saved to `__tests__/` subdirectories adjacent to source files
- [x] E2E test file saved to `eusolicit-app/e2e/specs/smoke/` (consistent with existing smoke tests)
- [x] Checklist saved to `eusolicit-docs/test-artifacts/`
- [x] E2E spec uses `.spec.ts` suffix (not `.admin.spec.ts`) → runs in client-chromium, Firefox, WebKit
- [x] Failure reasons are accurate (modules/pages do not exist yet)
- [x] Epic test IDs cross-referenced correctly (E03-P0-007, E03-P0-008, E03-P1-001, E03-P1-002, E03-P1-012, E03-P2-008)
- [x] localStorage mock applied in all Vitest store tests (`Object.defineProperty(globalThis, 'localStorage', ...)`)
- [x] `vi.resetModules()` in `beforeEach` prevents Zustand store state from leaking between tests
- [x] EventSource mock injected via `vi.stubGlobal('EventSource', MockEventSource)` in useSSE tests
- [x] `page.route()` used in Playwright E2E to mock GET /health responses (no real backend needed)

### Selector Strategy (E2E Tests)

E2E tests use minimal selectors derivable from the story implementation notes:

| Selector | Component | Source |
|----------|-----------|--------|
| `getByRole('heading', { name: /API Test/i })` | `<h1>` in `api-test/page.tsx` | AC6 Dev Notes |
| `getByText(/Checking \/health/i)` | `<p>Checking /health…</p>` | AC6 Dev Notes `{isLoading && ...}` |
| `locator('pre.bg-green-50')` | success `<pre>` block | AC6 Dev Notes class name |
| `locator('pre.bg-red-50')` | error `<pre>` block | AC6 Dev Notes class name |

### Risk Coverage

| Risk ID | Score | How Mitigated in Tests |
|---------|-------|------------------------|
| E03-R-003 (401 refresh deduplication) | 6 | T03 (api-client): 3 concurrent 401s → 1 refresh; T04: refresh 401 → logout |
| E03-R-004 (Zustand localStorage hydration) | 4 | T06–T07 (auth-store): persist key + rehydration; T09–T10 (ui-store): partialize + rehydration |
| E03-R-009 (SSE EventSource memory leak) | 2 | T01 (useSSE): close() called on unmount |

---

## Next Steps (TDD Green Phase)

After implementing Story 3.5 (all 9 tasks):

### 1. Configure Vitest (if not already done)

```bash
cd eusolicit-app/frontend
pnpm add -D vitest @vitest/globals @testing-library/react jsdom \
  @vitejs/plugin-react axios-mock-adapter --filter @eusolicit/ui
pnpm add -D vitest @vitest/globals --filter client
```

Create `packages/ui/vitest.config.ts` and `apps/client/vitest.config.ts` as described in Step 1 prerequisites.

### 2. Remove `it.skip()` / `test.skip()` from each test file

```
eusolicit-app/frontend/packages/ui/src/__tests__/stores/auth-store.test.ts      — 7 tests
eusolicit-app/frontend/apps/client/lib/stores/__tests__/ui-store.test.ts        — 10 tests
eusolicit-app/frontend/packages/ui/src/__tests__/lib/api-client.test.ts         — 6 tests
eusolicit-app/frontend/packages/ui/src/__tests__/lib/hooks/useSSE.test.ts       — 5 tests
eusolicit-app/e2e/specs/smoke/api-test-page.spec.ts                             — 4 tests
```

### 3. Start dev servers (for E2E tests)

```bash
cd eusolicit-app/frontend
pnpm dev --filter client    # http://localhost:3000
pnpm dev --filter admin     # http://localhost:3001 (needed for pnpm build check)
```

### 4. Run Vitest unit tests

```bash
# From eusolicit-app/frontend/
pnpm vitest run --filter @eusolicit/ui
pnpm vitest run --filter client
```

### 5. Run Playwright E2E smoke tests

```bash
# From eusolicit-app/
npx playwright test e2e/specs/smoke/api-test-page.spec.ts --project=client-chromium
```

### 6. Verify ALL 32 tests pass

If any tests fail after implementation:

- **Unit test fails** — Check the implementation matches the AC spec (e.g. wrong persist key, missing method)
- **E2E test fails** — Check that QueryProvider is wired in root layout; check selector names in page.tsx

### 7. Common GREEN-phase adjustments to expect

- **T03 (api-client P0-007):** The bare `axios.post` spy works only if the interceptor uses `axios.post` (not `apiClient.post`). If the implementation uses a different pattern, update the spy target.
- **T06–T07 (auth-store rehydration):** `persist.rehydrate()` is async. If still failing, add `await new Promise(r => setTimeout(r, 100))` after the call.
- **T09 (ui-store partialize):** If `theme` appears in persisted state, the `partialize` option is missing from the `persist()` call. Ensure `partialize: (state) => ({ sidebarCollapsed: state.sidebarCollapsed, locale: state.locale })`.
- **T02–T04 (api-test-page E2E):** If health check URL is `http://localhost:8000/health` (not relative), update `page.route('**/health', ...)` to `page.route('http://localhost:8000/health', ...)`.

### 8. Commit passing tests

Commit all 5 test files together with the Story 3.5 implementation.

---

## Implementation Guidance

### Files to Create (RED → GREEN triggers)

```
eusolicit-app/frontend/
├── packages/ui/src/lib/
│   ├── stores/
│   │   └── auth-store.ts          ← T01–T07 in auth-store.test.ts
│   ├── api-client.ts              ← T01–T06 in api-client.test.ts
│   ├── hooks/
│   │   └── useSSE.ts              ← T01–T05 in useSSE.test.ts
│   └── providers/
│       └── QueryProvider.tsx      ← T02–T04 in api-test-page.spec.ts (indirectly)
├── apps/client/
│   ├── lib/
│   │   ├── stores/
│   │   │   └── ui-store.ts        ← EXPAND: T01–T10 in ui-store.test.ts
│   │   └── queries/
│   │       └── use-health-check.ts ← T02–T04 in api-test-page.spec.ts
│   └── app/
│       ├── layout.tsx             ← ADD QueryProvider wrapper → T02–T04
│       └── dev/api-test/
│           └── page.tsx           ← T01–T04 in api-test-page.spec.ts
└── apps/admin/
    ├── lib/stores/ui-store.ts     ← EXPAND (same as client; separate store instance)
    └── app/layout.tsx             ← ADD QueryProvider wrapper
```

### Critical Implementation Requirements

1. **`auth-store.ts` persist key must be exactly `"eusolicit-auth-store"`** — T06 asserts this.
2. **Zustand middleware order `devtools(persist(...))`** — NOT `persist(devtools(...))` — required by TypeScript (Dev Notes §Critical Learnings point 9). Inverting causes TS errors which break T06/T07.
3. **`refreshPromise` singleton in apiClient** — `let refreshPromise: Promise<string> | null = null;` declared in module scope. Without this, T03 (P0-007) fails with `expect(1).toBe(1)` → `expect(3).toBe(1)`.
4. **Refresh uses bare `axios.post` (NOT `apiClient.post`)** — if `apiClient.post` is used, the refresh call re-enters the 401 interceptor, causing infinite recursion. T03's spy targets `axios.default.post`.
5. **`ui-store.ts` partialize** — must use `partialize: (state) => ({ sidebarCollapsed: state.sidebarCollapsed, locale: state.locale })`. Without it, T09 fails because `theme` and `toasts` appear in persisted state.
6. **`useSSE.ts` null guard** — `if (!url) return;` inside `useEffect` ensures T05 passes (no EventSource when url=null).
7. **`useSSE.ts` cleanup** — `return () => { es.close(); esRef.current = null; setStatus('closed'); }` inside `useEffect`. T01 (P2-008) asserts `mockClose` called exactly once.

---

## Notes & Assumptions

1. **Story 3.4 is fully implemented.** The `responsive-layout.spec.ts` is in GREEN phase. Story 3.5 adds infrastructure (stores, apiClient, hooks, QueryProvider) without touching layout components.

2. **No authentication required for E2E smoke test.** The `/dev/api-test` page is a developer utility under the `/dev/` route. Tests navigate directly without seeding auth state.

3. **`page.route('**/health', ...)` pattern.** Playwright's glob `**/health` matches `http://localhost:8000/health` (the apiClient's baseURL). If the actual backend URL differs from `http://localhost:8000`, update the route pattern.

4. **axios-mock-adapter version compatibility.** Install `axios-mock-adapter@^1.22.0` (supports Axios 1.x). The `new MockAdapter(apiClient)` wraps the axios instance at the adapter level, allowing per-test HTTP stubs without modifying the production interceptors.

5. **vitest `vi.resetModules()` in `beforeEach`.** This is essential for Zustand store tests because Zustand creates a singleton store per module. Without `resetModules()`, `login()` state from T02 leaks into T03. Each test imports a fresh store instance.

6. **`crypto.randomUUID()` availability.** Vitest runs in Node.js 20 LTS where `crypto.randomUUID()` is available globally. T07 and T08 (ui-store) assert toast ids are non-empty strings — this will pass as long as the implementation uses `crypto.randomUUID()` or any UUID strategy.

7. **Admin uiStore tested indirectly.** The admin app's `ui-store.ts` gets the same expansion as the client's. The unit tests target the client store. The admin store is validated by the existing CI build gate (TypeScript strict mode will flag any shape mismatch) and `pnpm build` exit code.

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Story:** 3.5 — Zustand Stores & TanStack Query Setup
**Version:** BMad v6 (sequential execution mode)
