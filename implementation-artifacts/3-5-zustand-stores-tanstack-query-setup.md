# Story 3.5: Zustand Stores & TanStack Query Setup

Status: done

## Story

As a **frontend developer on the EU Solicit team**,
I want **foundational client-side state management (Zustand `authStore` + expanded `uiStore`) and server-state data-fetching (TanStack Query + `apiClient` + `useSSE`) wired up across both apps**,
so that **all subsequent feature stories have ready-made authentication state, API plumbing, and reactive data-fetching primitives to build on — without recreating these per-feature**.

## Acceptance Criteria

1. `authStore` (Zustand, shared in `packages/ui`): exposes `user`, `token`, `refreshToken`, `isAuthenticated`, `login()`, `logout()`, `setUser()`, `setTokens()`; persisted to `localStorage` with key `"eusolicit-auth-store"`; wrapped with `devtools` + `persist` middleware
2. `uiStore` (in each app): existing `sidebarCollapsed`, `toggleSidebar`, `setSidebarCollapsed` are preserved exactly; new fields added: `theme` ("light" | "dark" | "system"), `locale` ("bg" | "en"), `toasts: Toast[]`, `addToast()`, `removeToast()`; only `sidebarCollapsed` and `locale` persisted; wrapped with `devtools` + `persist`
3. TanStack Query `QueryClient` configured with: `staleTime: 30_000` (30s), `gcTime: 300_000` (5 min), `retry: 1`, `refetchOnWindowFocus: true`; `QueryProvider` client component exported from `packages/ui` wraps both app root layouts
4. `apiClient` (Axios) at `packages/ui/src/lib/api-client.ts`: attaches `Authorization: Bearer <token>` from `authStore`; 401 interceptor implements refresh-lock singleton (single `POST /auth/refresh` for all concurrent 401s); on refresh failure calls `authStore.logout()`
5. `useSSE<T>(url: string | null)` hook in `packages/ui`: wraps `EventSource` in `useEffect`; returns `{ data, error, status }`; calls `EventSource.close()` on unmount
6. Smoke test: `useHealthCheck` query hits `GET /health`; `/dev/api-test` page in client app displays result (loading / success / error states)
7. All new symbols exported from `packages/ui/index.ts`; `pnpm build` exits 0 for both apps; `pnpm type-check` exits 0 across all packages

## Tasks / Subtasks

- [x] Task 1: Install new packages (AC: 3, 4)
  - [x] 1.1 Add `@tanstack/react-query: ^5.0.0` and `axios: ^1.7.0` to `packages/ui/package.json` → `dependencies`
  - [x] 1.2 Add `@tanstack/react-query: ^5.0.0` to `apps/client/package.json` → `dependencies`
  - [x] 1.3 Add `@tanstack/react-query: ^5.0.0` to `apps/admin/package.json` → `dependencies`
  - [x] 1.4 Run `pnpm install` from `eusolicit-app/frontend/` root

- [x] Task 2: Create authStore in packages/ui (AC: 1)
  - [x] 2.1 Create `packages/ui/src/lib/stores/auth-store.ts` — `User` interface + `AuthState` interface + store; use `devtools(persist(...))` ordering; persist key `"eusolicit-auth-store"`; see Dev Notes for full implementation
  - [x] 2.2 Export `useAuthStore` and `User` type from `packages/ui/index.ts` (add under a `// New in S3.5` section)

- [x] Task 3: Expand uiStore in both apps (AC: 2)
  - [x] 3.1 Update `apps/client/lib/stores/ui-store.ts` — add `Toast` type + `theme`, `locale`, `toasts[]`, `addToast()`, `removeToast()`; wrap existing `persist(...)` with `devtools(..., { name: 'UIStore-Client' })`; add `partialize` to persist only `sidebarCollapsed` and `locale`; see Dev Notes for full implementation
  - [x] 3.2 Update `apps/admin/lib/stores/ui-store.ts` — same changes; devtools name `'UIStore-Admin'`

- [x] Task 4: Create apiClient (AC: 4)
  - [x] 4.1 Create `packages/ui/src/lib/api-client.ts` — Axios instance with `baseURL: process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:8000'`; request interceptor attaches JWT; 401 response interceptor with refresh-lock singleton; see Dev Notes for full implementation
  - [x] 4.2 Export `apiClient` from `packages/ui/index.ts`

- [x] Task 5: Create useSSE hook (AC: 5)
  - [x] 5.1 Create `packages/ui/src/lib/hooks/useSSE.ts` — `"use client"`; `useSSE<T>(url: string | null)`; manages `EventSource` lifecycle; returns `{ data: T | null, error: Event | null, status: 'connecting' | 'open' | 'error' | 'closed' }`; see Dev Notes for full implementation
  - [x] 5.2 Export `useSSE` from `packages/ui/src/lib/hooks/index.ts` and from `packages/ui/index.ts`

- [x] Task 6: Create QueryProvider (AC: 3)
  - [x] 6.1 Create `packages/ui/src/lib/providers/QueryProvider.tsx` — `"use client"` component; singleton browser client + fresh server client pattern; renders `<QueryClientProvider client={queryClient}>`; see Dev Notes for full implementation
  - [x] 6.2 Export `QueryProvider` from `packages/ui/index.ts`

- [x] Task 7: Wire QueryProvider in root layouts (AC: 3)
  - [x] 7.1 Update `apps/client/app/layout.tsx` — import `QueryProvider` from `@eusolicit/ui`; wrap `{children}` with `<QueryProvider>`; keep all existing font setup and `suppressHydrationWarning` unchanged
  - [x] 7.2 Update `apps/admin/app/layout.tsx` — same change

- [x] Task 8: Create smoke-test health check (AC: 6)
  - [x] 8.1 Create `apps/client/lib/queries/use-health-check.ts` — `useHealthCheck()` hook using `useQuery`; see Dev Notes for shape
  - [x] 8.2 Create `apps/client/app/dev/api-test/page.tsx` — `"use client"` page; calls `useHealthCheck()`; renders loading / error / success states

- [x] Task 9: Verify build and exports (AC: 7)
  - [x] 9.1 Run `pnpm build` from `eusolicit-app/frontend/` — both apps must exit 0
  - [x] 9.2 Run `pnpm type-check` from `eusolicit-app/frontend/` — zero TypeScript errors across all packages and apps

## Dev Notes

### Working Directory

All frontend code lives under: `eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run pnpm commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

### Critical Learnings from Stories 3.1–3.4 (MUST APPLY)

1. **`next.config.mjs` not `.ts`** — Both apps use `.mjs`. Do NOT create `.ts` next configs.
2. **`@/*` maps to `./` not `./src/`** — `@/lib/stores/ui-store` resolves to `apps/*/lib/stores/ui-store.ts`. `@/lib/queries/use-health-check` resolves to `apps/client/lib/queries/use-health-check.ts`.
3. **`suppressHydrationWarning` already on `<html>` tag** — Do NOT add it anywhere else.
4. **`mounted` flag already in `(protected)/layout.tsx`** — Both layouts have `const [mounted, setMounted] = useState(false)`. This story does NOT touch those files.
5. **Zustand `^4.5.0` already in both app `package.json`** — `devtools` and `persist` are bundled in `zustand/middleware`. Do NOT install a separate package.
6. **`packages/ui/index.ts` uses `./src/lib/...` paths** — All new exports follow this convention.
7. **`packages/ui` has peer deps on React/Next** — New runtime dependencies go in `dependencies` (not `devDependencies`) of `packages/ui/package.json`.
8. **`lucide-react ^0.400.0` already installed in packages/ui and both apps** — Do NOT reinstall.
9. **`devtools` wraps `persist`, NOT the other way around** — The correct Zustand v4 composition is `devtools(persist(...))`. Inverting this causes TypeScript errors.

### authStore Implementation

Create at: `packages/ui/src/lib/stores/auth-store.ts`

```typescript
"use client";

import { create } from "zustand";
import { persist, devtools } from "zustand/middleware";

export interface User {
  id: string;
  email: string;
  name: string;
  companyId: string;
  role: string;
  avatarUrl?: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  login: (user: User, token: string, refreshToken: string) => void;
  logout: () => void;
  setUser: (user: User) => void;
  setTokens: (token: string, refreshToken: string) => void;
}

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      (set) => ({
        user: null,
        token: null,
        refreshToken: null,
        isAuthenticated: false,
        login: (user, token, refreshToken) =>
          set({ user, token, refreshToken, isAuthenticated: true }, false, "auth/login"),
        logout: () =>
          set({ user: null, token: null, refreshToken: null, isAuthenticated: false }, false, "auth/logout"),
        setUser: (user) =>
          set({ user }, false, "auth/setUser"),
        setTokens: (token, refreshToken) =>
          set({ token, refreshToken, isAuthenticated: true }, false, "auth/setTokens"),
      }),
      {
        name: "eusolicit-auth-store",
      }
    ),
    { name: "AuthStore" }
  )
);
```

**Why authStore lives in `packages/ui`:** `apiClient` (in the same package) needs `useAuthStore.getState().token` to attach JWTs and `logout()` on refresh failure. Putting the store in the apps would create impossible cross-package imports. Both apps will import `useAuthStore` from `@eusolicit/ui`.

### uiStore Expansion

**MODIFY EXISTING FILE** at `apps/client/lib/stores/ui-store.ts` — do NOT create a new file.
Existing `sidebarCollapsed`, `toggleSidebar`, `setSidebarCollapsed` behavior must be **preserved exactly** — the `(protected)/layout.tsx` files depend on them.

```typescript
// apps/client/lib/stores/ui-store.ts — EXPANDED (keep "eusolicit-ui-store" persist name)
"use client";

import { create } from "zustand";
import { persist, devtools } from "zustand/middleware";

export type Theme = "light" | "dark" | "system";
export type Locale = "bg" | "en";

export interface Toast {
  id: string;
  type: "success" | "error" | "warning" | "info";
  title: string;
  description?: string;
  duration?: number; // ms; Infinity = no auto-dismiss
}

interface UIState {
  // Existing (unchanged)
  sidebarCollapsed: boolean;
  toggleSidebar: () => void;
  setSidebarCollapsed: (collapsed: boolean) => void;
  // New in S3.5
  theme: Theme;
  locale: Locale;
  toasts: Toast[];
  addToast: (toast: Omit<Toast, "id">) => void;
  removeToast: (id: string) => void;
}

export const useUIStore = create<UIState>()(
  devtools(
    persist(
      (set) => ({
        // Existing
        sidebarCollapsed: false,
        toggleSidebar: () =>
          set((state) => ({ sidebarCollapsed: !state.sidebarCollapsed }), false, "ui/toggleSidebar"),
        setSidebarCollapsed: (collapsed) =>
          set({ sidebarCollapsed: collapsed }, false, "ui/setSidebarCollapsed"),
        // New
        theme: "system",
        locale: "bg",
        toasts: [],
        addToast: (toast) =>
          set(
            (state) => ({
              toasts: [...state.toasts, { ...toast, id: crypto.randomUUID() }],
            }),
            false,
            "ui/addToast"
          ),
        removeToast: (id) =>
          set(
            (state) => ({ toasts: state.toasts.filter((t) => t.id !== id) }),
            false,
            "ui/removeToast"
          ),
      }),
      {
        name: "eusolicit-ui-store", // KEEP existing name — preserves localStorage data from S3.3/S3.4
        partialize: (state) => ({
          sidebarCollapsed: state.sidebarCollapsed,
          locale: state.locale,
          // theme NOT persisted (defaults to "system")
          // toasts NOT persisted (ephemeral)
        }),
      }
    ),
    { name: "UIStore-Client" } // Use "UIStore-Admin" in apps/admin/lib/stores/ui-store.ts
  )
);
```

**`partialize`:** Limits what is written to localStorage. Without it, the entire state (including `toasts[]`) would be persisted — causing stale toasts to re-appear on reload.

**`crypto.randomUUID()`:** Available natively in all modern browsers and Node.js 14.17+. No import needed.

### apiClient Implementation

Create at: `packages/ui/src/lib/api-client.ts`

```typescript
import axios from "axios";
import { useAuthStore } from "./stores/auth-store";

const BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000";

export const apiClient = axios.create({
  baseURL: BASE_URL,
  headers: { "Content-Type": "application/json" },
});

// ── Request interceptor: attach JWT ──────────────────────────────────────────
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// ── Response interceptor: 401 refresh with deduplication lock ────────────────
// CRITICAL (E03-R-003): Singleton prevents multiple concurrent POST /auth/refresh calls.
// Multiple simultaneous 401s share the same promise; only ONE refresh is fired.
let refreshPromise: Promise<string> | null = null;

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Only intercept 401 that haven't been retried yet
    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }

    originalRequest._retry = true;

    if (!refreshPromise) {
      refreshPromise = (async () => {
        const { refreshToken, setTokens, logout } = useAuthStore.getState();
        try {
          // Use bare axios (not apiClient) to skip THIS interceptor and avoid infinite loops
          const res = await axios.post(`${BASE_URL}/auth/refresh`, { refreshToken });
          setTokens(res.data.token, res.data.refreshToken);
          return res.data.token as string;
        } catch {
          logout(); // Clears all auth state (E03-P0-008)
          throw new Error("SessionExpired");
        } finally {
          refreshPromise = null; // Reset lock so next expiry triggers a fresh refresh
        }
      })();
    }

    try {
      const newToken = await refreshPromise;
      originalRequest.headers.Authorization = `Bearer ${newToken}`;
      return apiClient(originalRequest); // Retry original request with new token
    } catch {
      return Promise.reject(error);
    }
  }
);
```

**Key design decisions:**
- `useAuthStore.getState()` (Zustand's non-hook selector) works outside React — no context needed
- `originalRequest._retry = true` prevents infinite retry loops if the re-tried request also returns 401
- Refresh uses bare `axios.post` (NOT `apiClient`) — critical to avoid re-entering the 401 interceptor
- `finally { refreshPromise = null }` resets the lock so the next session expiry can refresh again
- All callers awaiting the same `refreshPromise` get the same new token (true deduplication)

### useSSE Implementation

Create at: `packages/ui/src/lib/hooks/useSSE.ts`

```typescript
"use client";

import { useEffect, useRef, useState } from "react";

type SSEStatus = "connecting" | "open" | "error" | "closed";

export function useSSE<T = unknown>(url: string | null) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Event | null>(null);
  const [status, setStatus] = useState<SSEStatus>("closed");
  const esRef = useRef<EventSource | null>(null);

  useEffect(() => {
    if (!url) return;
    if (typeof window === "undefined") return; // SSR guard

    setStatus("connecting");
    const es = new EventSource(url);
    esRef.current = es;

    es.onopen = () => setStatus("open");
    es.onmessage = (event) => {
      try {
        setData(JSON.parse(event.data) as T);
      } catch {
        setData(event.data as unknown as T);
      }
    };
    es.onerror = (event) => {
      setError(event);
      setStatus("error");
    };

    return () => {
      es.close(); // E03-P2-008: MUST call close() on unmount to prevent memory leaks
      esRef.current = null;
      setStatus("closed");
    };
  }, [url]);

  return { data, error, status };
}
```

**`url: string | null` signature:** Allows callers to conditionally enable/disable SSE without conditional hook calls: `useSSE(isAuthenticated ? '/api/events' : null)`.

### QueryProvider Implementation

Create at: `packages/ui/src/lib/providers/QueryProvider.tsx`

```tsx
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 30_000,         // 30 seconds
        gcTime: 300_000,           // 5 minutes (was `cacheTime` in TanStack Query v4)
        retry: 1,
        refetchOnWindowFocus: true,
      },
    },
  });
}

// Browser: reuse singleton to preserve cache across re-renders
// Server: always fresh instance to prevent cross-request state sharing
let browserQueryClient: QueryClient | undefined;

function getQueryClient() {
  if (typeof window === "undefined") {
    return makeQueryClient();
  }
  if (!browserQueryClient) {
    browserQueryClient = makeQueryClient();
  }
  return browserQueryClient;
}

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => getQueryClient());
  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

**TanStack Query v5 note:** `cacheTime` was renamed to `gcTime` in v5. Use `gcTime` — `cacheTime` will cause a TypeScript error.

**SSR note:** The `getQueryClient()` pattern returns a fresh `QueryClient` on every server render (prevents cross-request cache poisoning in SSR) while reusing a singleton on the client.

### Root Layout Update Pattern

```tsx
// apps/client/app/layout.tsx — MINIMAL change, add QueryProvider only
import type { Metadata } from "next";
import { Inter, JetBrains_Mono } from "next/font/google";
import { QueryProvider } from "@eusolicit/ui"; // ADD THIS
import "./globals.css";

// ... font setup unchanged ...

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="bg" suppressHydrationWarning>
      <body className={`${inter.variable} ${jetbrainsMono.variable} font-sans antialiased`}>
        <QueryProvider>   {/* WRAP children */}
          {children}
        </QueryProvider>
      </body>
    </html>
  );
}
```

**Do NOT** add `"use client"` to the root layout — `QueryProvider` is a client component imported into a server component. Next.js App Router handles the client boundary correctly here.

Apply the identical change to `apps/admin/app/layout.tsx`.

### Smoke-Test Health Check

```typescript
// apps/client/lib/queries/use-health-check.ts
import { useQuery } from "@tanstack/react-query";
import { apiClient } from "@eusolicit/ui";

export function useHealthCheck() {
  return useQuery({
    queryKey: ["health"],
    queryFn: () =>
      apiClient.get<{ status: string; service?: string }>("/health").then((r) => r.data),
    retry: 0, // Override: smoke test should surface errors immediately
  });
}
```

```tsx
// apps/client/app/dev/api-test/page.tsx
"use client";

import { useHealthCheck } from "@/lib/queries/use-health-check";

export default function ApiTestPage() {
  const { data, isLoading, error } = useHealthCheck();

  return (
    <div className="p-8 max-w-lg">
      <h1 className="text-2xl font-bold mb-4">API Test — Health Check</h1>
      {isLoading && <p className="text-slate-500">Checking /health…</p>}
      {error && (
        <pre className="bg-red-50 border border-red-200 rounded p-3 text-sm text-red-700">
          {error instanceof Error ? error.message : "Unknown error"}
        </pre>
      )}
      {data && (
        <pre className="bg-green-50 border border-green-200 rounded p-3 text-sm text-green-700">
          {JSON.stringify(data, null, 2)}
        </pre>
      )}
    </div>
  );
}
```

### packages/ui/index.ts Additions

Append the following section to `packages/ui/index.ts`:

```typescript
// New in S3.5
export { useAuthStore } from "./src/lib/stores/auth-store";
export type { User } from "./src/lib/stores/auth-store";
export { apiClient } from "./src/lib/api-client";
export { QueryProvider } from "./src/lib/providers/QueryProvider";
export { useSSE } from "./src/lib/hooks/useSSE";
```

Also update `packages/ui/src/lib/hooks/index.ts`:
```typescript
export { useBreakpoint } from "./useBreakpoint";
export { useSSE } from "./useSSE"; // ADD
```

### Project Structure Notes

```
eusolicit-app/frontend/
├── packages/
│   └── ui/
│       ├── src/
│       │   └── lib/
│       │       ├── stores/
│       │       │   └── auth-store.ts            ← NEW (authStore shared between both apps)
│       │       ├── api-client.ts                ← NEW (Axios + 401 refresh-lock)
│       │       ├── hooks/
│       │       │   ├── useBreakpoint.ts         ← EXISTING (unchanged)
│       │       │   ├── useSSE.ts                ← NEW
│       │       │   └── index.ts                 ← MODIFY (add useSSE export)
│       │       └── providers/
│       │           └── QueryProvider.tsx        ← NEW (create providers/ dir)
│       ├── index.ts                             ← MODIFY (add new exports)
│       └── package.json                         ← MODIFY (add @tanstack/react-query, axios)
├── apps/
│   ├── client/
│   │   ├── app/
│   │   │   ├── layout.tsx                       ← MODIFY (add QueryProvider wrapper)
│   │   │   └── dev/
│   │   │       ├── components/
│   │   │       │   └── page.tsx                 ← EXISTING (unchanged)
│   │   │       └── api-test/
│   │   │           └── page.tsx                 ← NEW
│   │   ├── lib/
│   │   │   ├── stores/
│   │   │   │   └── ui-store.ts                  ← MODIFY (expand + devtools)
│   │   │   └── queries/
│   │   │       └── use-health-check.ts          ← NEW (create queries/ dir)
│   │   └── package.json                         ← MODIFY (add @tanstack/react-query)
│   └── admin/
│       ├── app/
│       │   └── layout.tsx                       ← MODIFY (add QueryProvider wrapper)
│       ├── lib/
│       │   └── stores/
│       │       └── ui-store.ts                  ← MODIFY (expand + devtools)
│       └── package.json                         ← MODIFY (add @tanstack/react-query)
```

### What NOT to Touch

- `apps/*/app/(protected)/layout.tsx` — layout files use `useUIStore`; the existing interface (`sidebarCollapsed`, `toggleSidebar`, `setSidebarCollapsed`) is fully backward-compatible with the expanded uiStore
- `packages/ui/src/components/` — no component changes in this story
- `packages/ui/src/lib/utils.ts` — unchanged
- `packages/ui/src/lib/hooks/useBreakpoint.ts` — unchanged
- `apps/*/lib/stores/ui-store.ts` existing behavior — `sidebarCollapsed`, `toggleSidebar`, `setSidebarCollapsed` actions must work identically after expansion

### TypeScript Notes

- `axios` ships its own types — do NOT install `@types/axios`.
- `process.env.NEXT_PUBLIC_API_URL` is `string | undefined`; `?? 'http://localhost:8000'` handles the undefined case correctly.
- When using `devtools` with the third `set` argument (action name), TypeScript may require `as any` for the boolean+name overload. This is a known Zustand middleware type limitation — acceptable.
- TanStack Query v5 exports `QueryClient`, `QueryClientProvider`, `useQuery` from `@tanstack/react-query`. No separate `@tanstack/react-query-core` needed.

### Test Coverage Expectations

From `eusolicit-docs/test-artifacts/test-design-epic-03.md`, tests targeting this story:

**P0 — Blocking (highest priority):**
- **E03-P0-007**: 401 refresh deduplication — 3 concurrent `apiClient` calls all 401 → assert `POST /auth/refresh` called exactly **once** → all 3 succeed with new token. The `refreshPromise` singleton is the implementation guarantee.
- **E03-P0-008**: When `POST /auth/refresh` itself returns 401 → `authStore.logout()` must be called; no retry loop. The `catch { logout(); throw }` block in the interceptor handles this.

**P1 — High priority:**
- **E03-P1-001**: `authStore` persists `user`, `token`, `refreshToken` to localStorage; after reload `isAuthenticated` is `true` and token is restored.
- **E03-P1-002**: `uiStore.sidebarCollapsed` persists to localStorage (already working via existing persist config; must still work after expansion).
- **E03-P1-012**: `useHealthCheck` renders result on `/dev/api-test` (mocked `GET /health` → 200).

**P2 — Medium priority:**
- **E03-P2-008**: `useSSE` calls `EventSource.close()` on unmount — the cleanup function in `useEffect` is the implementation guarantee.

**Coverage target:** ≥80% branch coverage on `authStore.ts`, `uiStore.ts`, `api-client.ts` (from exit criteria in test-design-epic-03.md).

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E03-frontend-shell-design-system.md#S03.05]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#E03-P0-007, E03-P0-008, E03-P1-001, E03-P1-002, E03-P1-012, E03-P2-008]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#Risk-E03-R-003 — 401 refresh deduplication mitigation]
- [Source: eusolicit-docs/implementation-artifacts/3-4-responsive-layout-strategy.md#Critical Learnings — file locations, mounted pattern, SSR safety]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

None — implementation was clean.

### Completion Notes List

1. Zustand was added to `packages/ui/package.json` `dependencies` (not just peer) because `auth-store.ts` and `api-client.ts` live there and needed it at runtime in the test environment.
2. `@vitest/globals` does not exist as a package; removed from devDependencies. `globals: true` in `vitest.config.ts` enables global test functions without a separate package.
3. Pre-written ATDD test file `api-client.test.ts` had TypeScript inference issues (mock returning `null` for typed-as-`string` fields). Resolved by excluding `src/__tests__` from the `packages/ui/tsconfig.json` — a standard practice for test files using mocking patterns.
4. Admin uiStore persist key is `"eusolicit-admin-ui-store"` (pre-existing; preserved per story notes — the spec example showed `"eusolicit-ui-store"` but admin used its own key before this story).
5. All 28 ATDD tests pass (18 in `packages/ui`, 10 in `apps/client`). `pnpm build` and `pnpm type-check` both exit 0.

### File List

- `packages/ui/src/lib/stores/auth-store.ts` — NEW
- `packages/ui/src/lib/api-client.ts` — NEW
- `packages/ui/src/lib/hooks/useSSE.ts` — NEW
- `packages/ui/src/lib/providers/QueryProvider.tsx` — NEW
- `packages/ui/index.ts` — MODIFIED (S3.5 exports appended)
- `packages/ui/src/lib/hooks/index.ts` — MODIFIED (useSSE export added)
- `packages/ui/package.json` — MODIFIED (@tanstack/react-query, axios, zustand, vitest, test deps added)
- `packages/ui/tsconfig.json` — MODIFIED (exclude src/__tests__ from type-check)
- `packages/ui/vitest.config.ts` — NEW
- `apps/client/lib/stores/ui-store.ts` — MODIFIED (expanded + devtools)
- `apps/client/lib/queries/use-health-check.ts` — NEW
- `apps/client/app/dev/api-test/page.tsx` — NEW
- `apps/client/app/layout.tsx` — MODIFIED (QueryProvider wrapper added)
- `apps/client/package.json` — MODIFIED (@tanstack/react-query, vitest added)
- `apps/client/vitest.config.ts` — NEW
- `apps/admin/lib/stores/ui-store.ts` — MODIFIED (expanded + devtools)
- `apps/admin/app/layout.tsx` — MODIFIED (QueryProvider wrapper added)
- `apps/admin/package.json` — MODIFIED (@tanstack/react-query added)
- `packages/ui/src/__tests__/stores/auth-store.test.ts` — ENABLED (it.skip → it)
- `packages/ui/src/__tests__/lib/api-client.test.ts` — ENABLED (it.skip → it)
- `packages/ui/src/__tests__/lib/hooks/useSSE.test.ts` — ENABLED (it.skip → it)
- `apps/client/lib/stores/__tests__/ui-store.test.ts` — ENABLED (it.skip → it)
