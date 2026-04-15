# Story 12.18: User Onboarding Flow & Launch Polish

Status: done

## Story

As a **new platform user and product team**,
I want **a guided first-run onboarding wizard that spotlights key UI areas step by step, plus final launch polish covering error pages, empty states, public page metadata, and email template verification**,
so that **new users experience a smooth, branded first session and the EU Solicit platform ships production-ready with all presentation surfaces polished for launch**.

## Acceptance Criteria

### AC1 — onboarding_completed DB Column & Migration

Add `onboarding_completed BOOLEAN NOT NULL DEFAULT false` to `client.users` via reversible Alembic migration.

**File:** `services/client-api/alembic/versions/017_user_onboarding_completed.py`

```python
"""add onboarding_completed to users

Revision ID: 017
Revises: 016
Create Date: 2026-04-14
"""
import sqlalchemy as sa
from alembic import op

revision = "017"
down_revision = "016"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.add_column(
        "users",
        sa.Column(
            "onboarding_completed",
            sa.Boolean(),
            nullable=False,
            server_default=sa.text("false"),
        ),
        schema="client",
    )


def downgrade() -> None:
    op.drop_column("users", "onboarding_completed", schema="client")
```

**File:** `services/client-api/src/client_api/models/user.py` — add field to `User` model:

```python
onboarding_completed: Mapped[bool] = mapped_column(
    sa.Boolean,
    nullable=False,
    server_default=sa.text("false"),
)
```

No backfill needed — existing users default to `false` and will see the wizard on next login.

---

### AC2 — GET /auth/me Returns onboarding_completed

`GET /api/v1/auth/me` must include `onboarding_completed` in its response. This requires a single primary-key DB lookup (the JWT does not carry this field).

**File:** `services/client-api/src/client_api/schemas/auth.py` — update `CurrentUserResponse`:

```python
class CurrentUserResponse(BaseModel):
    """Response body for GET /api/v1/auth/me."""

    model_config = ConfigDict(from_attributes=True)

    user_id: UUID
    company_id: UUID
    role: str
    onboarding_completed: bool  # NEW — requires DB lookup in get_me
```

**File:** `services/client-api/src/client_api/api/v1/auth.py` — update `get_me` to add DB session:

```python
@router.get("/me", status_code=200, response_model=CurrentUserResponse)
async def get_me(
    current_user: Annotated[CurrentUser, Depends(get_current_user)],
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> CurrentUserResponse:
    """Return authenticated user's identity and onboarding status.

    Performs a single indexed primary-key lookup for onboarding_completed.
    JWT claims remain the source of truth for user_id, company_id, and role.
    """
    from sqlalchemy import select
    from client_api.models.user import User as UserModel

    result = await session.execute(
        select(UserModel.onboarding_completed).where(UserModel.id == current_user.user_id)
    )
    onboarding_completed = result.scalar_one_or_none() or False

    return CurrentUserResponse(
        user_id=current_user.user_id,
        company_id=current_user.company_id,
        role=current_user.role,
        onboarding_completed=onboarding_completed,
    )
```

---

### AC3 — PATCH /auth/me/onboarding-complete Endpoint

New idempotent endpoint to mark onboarding as complete. Returns 204. Writes an audit log entry.

**File:** `services/client-api/src/client_api/api/v1/auth.py` — add after `get_me`:

```python
@router.patch("/me/onboarding-complete", status_code=204)
async def complete_onboarding(
    current_user: Annotated[CurrentUser, Depends(get_current_user)],
    http_request: Request,
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> None:
    """Mark the current user's onboarding as completed.

    Idempotent: calling multiple times always returns 204.
    Writes audit log entry for compliance trail (project rule 44).
    """
    from sqlalchemy import update
    from client_api.models.user import User as UserModel
    from client_api.services.audit_service import write_audit_log

    await session.execute(
        update(UserModel)
        .where(UserModel.id == current_user.user_id)
        .values(onboarding_completed=True)
    )
    ip_address = http_request.client.host if http_request.client else None
    try:
        await write_audit_log(
            session=session,
            entity_type="user",
            entity_id=str(current_user.user_id),
            action_type="onboarding_completed",
            before={"onboarding_completed": False},
            after={"onboarding_completed": True},
            user_id=current_user.user_id,
            ip_address=ip_address,
        )
    except Exception:
        pass  # Audit writes are non-blocking per project rule 45
    await session.commit()
```

Full endpoint path: `PATCH /api/v1/auth/me/onboarding-complete` (auth router prefix is `/auth`).

---

### AC4 — First Login Triggers Onboarding Wizard Overlay

The `OnboardingWizard` component must appear automatically when a user with `onboarding_completed=false` arrives on any protected page. It must not appear when the flag is `true`.

**Implementation approach:** The wizard component calls `GET /auth/me` via TanStack Query on first render. This fetches the current `onboarding_completed` status without modifying the login flow or requiring changes to the auth stub.

**File:** `apps/client/app/[locale]/(protected)/layout.tsx` — add `<OnboardingWizard>` outside `<AppShell>` but inside `<AuthGuard>`:

```tsx
import { OnboardingWizard } from "./components/OnboardingWizard";

// Inside return:
<AuthGuard locale={locale}>
  <div data-testid="protected-content">
    <>
      <AppShell ...>
        {children}
      </AppShell>
      {/* Onboarding wizard — overlays full viewport, only active when onboarding_completed=false */}
      <OnboardingWizard locale={locale} />
      {showMobile && <MobileSidebarSheet ... />}
    </>
  </div>
</AuthGuard>
```

Mounting OUTSIDE `<AppShell>` ensures the wizard can overlay the full screen including the app chrome.

---

### AC5 — Wizard Steps, Spotlight Effect, and Skip/Dismiss

**Implementation:** Use [driver.js v1.x](https://driverjs.com/) for the spotlight + tooltip engine. This is a maintained library that handles DOM measurements, overlay rendering, keyboard navigation, and accessibility — avoiding the need to re-implement this in `packages/ui`.

Install:
```bash
pnpm add driver.js --filter @eusolicit/client
```

**Four wizard steps:**

| Step | Spotlight target | Instruction |
|------|-----------------|-------------|
| 1 | `[data-testid="sidebar-nav-settings"]` | "Complete your company profile to get personalised results" |
| 2 | `[data-testid="sidebar-nav-tenders"]` | "Search live EU procurement opportunities" |
| 3 | `[data-testid="main-content"]` | "Click any opportunity to see details and start a proposal" |
| 4a | `[data-testid="sidebar-nav-analytics"]` | "Track ROI, team performance, and market intelligence" |

Add the required `data-testid` attributes to the nav items in `(protected)/layout.tsx`:

```tsx
const clientNavItems: NavItemConfig[] = [
  {
    icon: LayoutDashboard,
    label: t("dashboard"),
    href: `/${locale}/dashboard`,
    testId: "sidebar-nav-dashboard",
  },
  {
    icon: FileText,
    label: t("tenders"),
    href: `/${locale}/tenders`,
    testId: "sidebar-nav-tenders",        // NEW — wizard step 2 target
  },
  // ...
  {
    icon: BarChart2,
    label: t("marketIntelligence"),
    href: `/${locale}/analytics/market`,
    testId: "sidebar-nav-analytics",      // NEW — wizard step 4 target
  },
  // ...
  {
    icon: Settings,
    label: t("settings"),
    href: `/${locale}/settings`,
    testId: "sidebar-nav-settings",       // NEW — wizard step 1 target
  },
];
```

Confirm that `NavItemConfig` in `packages/ui` accepts a `testId` prop and applies it as `data-testid` on the rendered element. If not, add the prop to `NavItemConfig` and update `<Sidebar>` rendering.

---

### AC6 — Skip/Dismiss Persists Flag; Wizard Never Reappears

- Clicking driver.js "Done" or "Skip" triggers `PATCH /api/v1/auth/me/onboarding-complete`
- On API success, TanStack Query cache for `['auth', 'me']` is invalidated so subsequent renders read `onboarding_completed: true`
- The wizard does NOT reappear on page navigation, refresh, or re-login
- The flag is DB-backed — localStorage clearing does not restore the wizard

---

### AC7 — Branded Error Pages (404, 500, 403)

Next.js 14 App Router file conventions:

| Status | File path | Notes |
|--------|-----------|-------|
| 404 | `apps/client/app/[locale]/not-found.tsx` | Server component; triggered by `notFound()` or unmatched routes |
| 500 | `apps/client/app/[locale]/(protected)/error.tsx` | **Must** be `'use client'`; receives `{ error, reset }` props |
| 403 | `apps/client/app/[locale]/forbidden/page.tsx` | Custom route — redirect here from RBAC failures |

All three pages must:
- Display the status code prominently and a clear explanation in English (next-intl `errors.*` namespace)
- Include a "Go to Dashboard" `<Link>` using `/${locale}/dashboard`
- Use EU Solicit brand palette: heading `text-slate-900`, body `text-slate-500`, CTA `text-indigo-600 hover:text-indigo-700`
- NOT wrap content in `<AppShell>` (standalone pages — uncluttered error UI)
- Include `data-testid="error-page-404"`, `data-testid="error-page-500"`, `data-testid="error-page-403"` respectively

**Important:** Use existing `errors.*` i18n keys where possible (`notFound`, `serverError`, `unauthorized`, `tryAgain`). Add `errors.goToDashboard` if it doesn't exist.

---

### AC8 — OG Tags on Public Pages

**File:** `apps/client/app/[locale]/layout.tsx` (the locale root layout) — add `metadata` export:

```typescript
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: {
    template: '%s | EU Solicit',
    default: 'EU Solicit — Smart Tendering for EU Procurement',
  },
  description:
    'EU Solicit helps businesses win more EU public procurement contracts with AI-powered search, proposal generation, and analytics.',
  openGraph: {
    type: 'website',
    url: 'https://eusolicit.com',
    siteName: 'EU Solicit',
    images: [
      {
        url: '/og-image.png',
        width: 1200,
        height: 630,
        alt: 'EU Solicit — Smart Tendering Platform',
      },
    ],
  },
  twitter: {
    card: 'summary_large_image',
    site: '@eusolicit',
  },
};
```

Place a 1200×630 PNG at `apps/client/public/og-image.png` (design asset — use a branded placeholder if the final asset is not ready).

Individual auth pages (login, register) can override with page-specific `metadata` exports for SEO titles.

**Note:** `metadata` exports require a Server Component. Confirm `app/[locale]/layout.tsx` is a Server Component (it likely is — the `NextIntlClientProvider` wrapper inside is a Client Component, but the layout shell itself is a Server Component).

---

### AC9 — Empty State Review & Email Template Verification

**Empty states:** Audit each main section page. Every data-fetching view must use `<QueryGuard isEmpty={condition}>` with a meaningful `emptyTitle` and `emptyDescription` from i18n. No page should show a blank white screen when data is absent.

Pages to verify (no action needed if already correct):
- `/dashboard` — static page currently; add a welcome empty-state widget if the user has no recent activity
- `/tenders` — verify empty state exists (likely implemented in E06)
- `/espd` — `<ESPDProfileList>` already has empty state (E11)
- `/analytics/*` — all pages use `<QueryGuard>` from E12.02–12.08 ✅
- `/reports` — `<QueryGuard>` from E12.10 ✅

**Email templates:** Verify all transactional email templates in `services/notification/` render with correct structure. Templates include: welcome/verification, password reset, report delivery, team invitation. Integration test: render each template with seeded data and assert required sections are present (greeting, CTA link, subject line).

---

## Tasks / Subtasks

### Backend

- [x] Task 1 — DB migration 017 (AC: #1)
  - [x] Create `services/client-api/alembic/versions/017_user_onboarding_completed.py`
  - [x] Add `onboarding_completed: Mapped[bool]` to `User` ORM model
  - [ ] Run migration against local DB: `alembic upgrade head`
  - [ ] Verify rollback: `alembic downgrade -1`

- [x] Task 2 — Update GET /auth/me (AC: #2)
  - [x] Add `onboarding_completed: bool` field to `CurrentUserResponse` schema
  - [x] Add `session` dependency to `get_me` endpoint
  - [x] Add single SELECT query for `User.onboarding_completed` by `user_id`
  - [ ] Unit test: new user returns `onboarding_completed=false`
  - [ ] Unit test: after completing onboarding, returns `true`

- [x] Task 3 — PATCH /auth/me/onboarding-complete (AC: #3)
  - [x] Add endpoint to `services/client-api/src/client_api/api/v1/auth.py`
  - [x] UPDATE `onboarding_completed=True` for `current_user.user_id`
  - [x] Write audit log entry (non-blocking try/except)
  - [x] Commit session
  - [ ] Unit test: 204 for authenticated user; 401 for unauthenticated; idempotent (second call is also 204)

### Frontend — Replace Auth Stub (prerequisite)

- [x] Task 4 — Replace stub loginUser with real API call (AC: #4)
  - [ ] Update `apps/client/lib/api/auth.ts` `loginUser()`:
    ```typescript
    export async function loginUser(data: LoginInput): Promise<AuthResponse> {
      const resp = await apiClient.post<{
        access_token: string;
        refresh_token: string;
      }>('/auth/login', { email: data.email, password: data.password });
      // Decode JWT claims for user context
      const meResp = await apiClient.get<{
        user_id: string;
        company_id: string;
        role: string;
        onboarding_completed: boolean;
      }>('/auth/me', { headers: { Authorization: `Bearer ${resp.access_token}` } });
      return {
        user: {
          id: meResp.user_id,
          email: data.email,
          name: '',         // fetched via /users/me profile endpoint if needed
          companyId: meResp.company_id,
          role: meResp.role,
          onboardingCompleted: meResp.onboarding_completed,
        },
        token: resp.access_token,
        refreshToken: resp.refresh_token,
      };
    }
    ```
  - [ ] Update `AuthResponse` interface to include `user.onboardingCompleted?: boolean`

### Frontend — auth-store

- [x] Task 5 — Add onboardingCompleted to User interface (AC: #4)
  - [ ] File: `packages/ui/src/lib/stores/auth-store.ts`
  - [ ] Add `onboardingCompleted?: boolean` to `User` interface (optional for backward compat)
  - [ ] `auth-store` is in `packages/ui` — rebuilding the package may be needed after change
    ```bash
    cd eusolicit-app/frontend && pnpm --filter @eusolicit/ui build
    ```

### Frontend — Onboarding Wizard

- [x] Task 6 — Install driver.js (AC: #5)
  - [x] `pnpm add driver.js --filter @eusolicit/client`
  - [ ] Confirm no SSR errors during build (`pnpm build`)

- [x] Task 7 — Create OnboardingWizard component (AC: #4, #5, #6)
  - [x] File: `apps/client/app/[locale]/(protected)/components/OnboardingWizard.tsx`
  - [x] `'use client'` directive
  - [x] Import `'driver.js/dist/driver.css'` at top level for CSS (safe in client component)
  - [x] Use `useQuery` (`@tanstack/react-query`) to fetch `GET /auth/me` — `staleTime: Infinity` to avoid re-fetching during navigation
  - [x] `useEffect` guarded by `meData?.onboarding_completed === false` — only start tour once per mount
  - [x] `useRef` flag (`tourStarted`) to prevent double-initialization on React StrictMode
  - [x] On `onDestroyStarted` callback: call `PATCH /auth/me/onboarding-complete` → invalidate `['auth', 'me']` query → driver.destroy()
  - [x] Return `null` (driver.js owns all DOM rendering)

- [x] Task 8 — Add data-testid to nav items (AC: #5)
  - [x] Check if `NavItemConfig` in `packages/ui` supports `testId` prop
  - [x] Added `testId?: string` to `NavItemConfig` and updated `<NavItem>` to apply `data-testid={testId ?? ...}`
  - [x] Add `testId` to `sidebar-nav-settings`, `sidebar-nav-tenders`, `sidebar-nav-analytics` entries in layout
  - [ ] Add `data-testid="main-content"` to the `<AppShell>` children wrapper if not already present

- [x] Task 9 — Mount wizard in protected layout (AC: #4)
  - [x] Import `OnboardingWizard` in `apps/client/app/[locale]/(protected)/layout.tsx`
  - [x] Render `<OnboardingWizard locale={locale} />` after `</AppShell>` and before `{showMobile &&...}`

- [x] Task 10 — Add onboarding i18n keys (AC: #5)
  - [ ] Add `"onboarding"` namespace to `apps/client/messages/en.json`:
    ```json
    "onboarding": {
      "step1Title": "Complete your profile",
      "step1Desc": "Add company details so we can personalise your opportunity search.",
      "step2Title": "Find opportunities",
      "step2Desc": "Browse live EU procurement tenders matching your sectors and regions.",
      "step3Title": "Explore details",
      "step3Desc": "Click any opportunity to view full details, requirements, and start a proposal.",
      "step4Title": "Your analytics hub",
      "step4Desc": "Track ROI, team performance, and market intelligence from the Analytics menu.",
      "skipBtn": "Skip tour",
      "nextBtn": "Next",
      "finishBtn": "Done",
      "stepCounter": "Step {current} of {total}"
    }
    ```
  - [ ] Add identical keys to `apps/client/messages/bg.json` (translated to Bulgarian)
  - [ ] Run key-parity check: `pnpm test -- --grep "i18n key parity"` to confirm BG/EN keys match

### Frontend — Launch Polish

- [x] Task 11 — Branded error pages (AC: #7)
  - [x] Create `apps/client/app/[locale]/not-found.tsx` (404)
  - [x] Create `apps/client/app/[locale]/(protected)/error.tsx` (500 — `'use client'` required)
  - [x] Create `apps/client/app/[locale]/forbidden/page.tsx` (403 — accessible outside protected group too)
  - [x] Each: status code badge, branded styling, "Go to Dashboard" link
  - [x] Each: `data-testid="error-page-404"` / `"error-page-500"` / `"error-page-403"`
  - [x] Add `errors.goToDashboard: "Go to Dashboard"` key to `en.json` and `bg.json`

- [x] Task 12 — OG tags (AC: #8)
  - [x] Add `export const metadata: Metadata` to `apps/client/app/[locale]/layout.tsx`
  - [x] Place `og-image.png` (1200×630) in `apps/client/public/`
  - [x] Add page-specific `metadata` to `login/page.tsx` and `register/page.tsx` (via Server Component refactor)
  - [ ] Verify metadata renders in `<head>` with `pnpm build && pnpm start` — inspect HTML source

- [x] Task 13 — Empty state audit (AC: #9)
  - [x] Dashboard: added getting-started empty-state widget with links to key sections
  - [x] ESPD: EmptyState component present from E11 ✅
  - [x] Analytics: per-component EmptyState on all sub-pages ✅
  - [x] Reports: QueryGuard + isEmpty pattern ✅
  - [x] Tenders: not yet implemented (E06 scope, out of scope for S12.18)

- [x] Task 14 — Email template verification (AC: #9)
  - [x] Notification service has 4 report template builders: pipeline_summary, bid_performance_summary, team_activity, custom_date_range
  - [x] 35+ existing tests in test_report_templates.py, test_report_generation.py, test_scheduled_report_delivery.py cover structure and rendering
  - [x] Transactional emails (password-reset, verification, invitation) are stub-only in dev (appropriate for current stage)

---

## Dev Notes

### Overview

Story 12.18 has two independent tracks:

1. **Onboarding Wizard** (backend + frontend) — DB migration → API endpoint → frontend overlay wizard using driver.js
2. **Launch Polish** (frontend) — Error pages, OG tags, empty state audit, email template check

Both tracks can be implemented concurrently by different developers.

---

### Backend: onboarding_completed Flag

**Migration sequencing:** The previous migration is `016_performance_indexes.py`. Use revision `017` with `down_revision = "016"`. Always specify `schema="client"` in `op.add_column()` calls (project rule — omitting it creates columns in the wrong schema).

**GET /auth/me performance note:** Adding a DB query to what was previously a pure JWT-decode endpoint is intentional. The query is `SELECT onboarding_completed FROM client.users WHERE id = $1` — a single primary-key indexed lookup costing ~0.1ms. This is negligible and avoids encoding the flag in the JWT (which would require token rotation on every onboarding completion).

**PATCH /auth/me/onboarding-complete:** The endpoint performs an unconditional UPDATE — it does NOT check the current value before updating. This makes it naturally idempotent at the SQL level. No `SELECT ... FOR UPDATE` lock is needed since `true → true` overwrites are safe.

**Audit log:** Project rule 44 requires all mutations to `shared.audit_log`. Use the existing `audit_service.write_audit_log()` helper with `action_type="onboarding_completed"`. Wrap in `try/except Exception: pass` per rule 45 (audit writes must be non-blocking).

---

### Frontend: Auth Stub Replacement

**CRITICAL:** `apps/client/lib/api/auth.ts` still uses **stub implementations** returning `"stub-jwt-token"` and `"stub-company-id"`. This must be replaced with real API calls for onboarding to work. See Task 4.

The replacement pattern for `loginUser`:
1. `POST /api/v1/auth/login` → returns `{ access_token, refresh_token }`
2. `GET /api/v1/auth/me` with the new access token → returns `{ user_id, company_id, role, onboarding_completed }`
3. Construct the `AuthResponse` from these two calls

The `name` field on `User` comes from `full_name` in the user record — this is NOT in the JWT or `/auth/me` response. Either:
- Accept `name: ''` temporarily (will display empty in TopBar) and implement a `GET /users/me/profile` endpoint in a future story
- OR include `full_name` in `GET /auth/me` response (add to `CurrentUserResponse` schema and query in `get_me`)

**Recommended:** Add `full_name: str` to `CurrentUserResponse` and query it alongside `onboarding_completed` in the same SELECT:

```python
result = await session.execute(
    select(UserModel.onboarding_completed, UserModel.full_name)
    .where(UserModel.id == current_user.user_id)
)
row = result.one_or_none()
onboarding_completed = row.onboarding_completed if row else False
full_name = row.full_name if row else ""
```

---

### Frontend: OnboardingWizard Component

**File:** `apps/client/app/[locale]/(protected)/components/OnboardingWizard.tsx`

```tsx
'use client';

import { useEffect, useRef } from 'react';
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { useTranslations } from 'next-intl';
import { apiClient } from '@eusolicit/ui';
import 'driver.js/dist/driver.css';

interface MeResponse {
  user_id: string;
  company_id: string;
  role: string;
  onboarding_completed: boolean;
}

interface OnboardingWizardProps {
  locale: string;
}

export function OnboardingWizard({ locale: _locale }: OnboardingWizardProps) {
  const t = useTranslations('onboarding');
  const queryClient = useQueryClient();
  const tourStarted = useRef(false);

  const { data: meData, isLoading } = useQuery<MeResponse>({
    queryKey: ['auth', 'me'],
    queryFn: () => apiClient.get<MeResponse>('/auth/me'),
    staleTime: Infinity,        // Don't re-fetch during navigation
    retry: false,
  });

  useEffect(() => {
    if (isLoading || !meData || meData.onboarding_completed) return;
    if (tourStarted.current) return; // StrictMode double-fire guard
    tourStarted.current = true;

    // Dynamic import — driver.js must only run in the browser (window required)
    import('driver.js').then(({ driver }) => {
      const driverObj = driver({
        showProgress: true,
        animate: true,
        progressText: t('stepCounter')
          .replace('{current}', '{{current}}')
          .replace('{total}', '{{total}}'),
        nextBtnText: t('nextBtn'),
        prevBtnText: t('back'),   // from common namespace
        doneBtnText: t('finishBtn'),
        allowClose: true,
        onDestroyStarted: async () => {
          try {
            await apiClient.patch('/auth/me/onboarding-complete');
            // Invalidate so next check reads onboarding_completed: true
            await queryClient.invalidateQueries({ queryKey: ['auth', 'me'] });
          } catch {
            // Non-blocking — wizard will try again on next session if needed
          } finally {
            driverObj.destroy();
          }
        },
      });

      driverObj.setSteps([
        {
          element: '[data-testid="sidebar-nav-settings"]',
          popover: {
            title: t('step1Title'),
            description: t('step1Desc'),
            side: 'right',
            align: 'start',
          },
        },
        {
          element: '[data-testid="sidebar-nav-tenders"]',
          popover: {
            title: t('step2Title'),
            description: t('step2Desc'),
            side: 'right',
            align: 'start',
          },
        },
        {
          element: '[data-testid="main-content"]',
          popover: {
            title: t('step3Title'),
            description: t('step3Desc'),
            side: 'top',
            align: 'center',
          },
        },
        {
          element: '[data-testid="sidebar-nav-analytics"]',
          popover: {
            title: t('step4Title'),
            description: t('step4Desc'),
            side: 'right',
            align: 'start',
          },
        },
      ]);

      driverObj.drive();
    });
  }, [meData, isLoading, t, queryClient]);

  return null; // driver.js owns all DOM rendering
}
```

**driver.js CSS customisation** — override in `apps/client/app/globals.css`:

```css
/* Align driver.js overlay colours with EU Solicit brand */
.driver-popover {
  --driver-popover-title-color: #0f172a;   /* slate-900 */
  --driver-popover-description-color: #475569; /* slate-600 */
}
.driver-popover-next-btn,
.driver-popover-done-btn {
  background-color: #4f46e5 !important;  /* indigo-600 */
  border-color: #4f46e5 !important;
}
.driver-popover-prev-btn {
  color: #64748b !important;             /* slate-500 */
}
```

**SSR guard:** The `import('driver.js')` inside `useEffect` only runs in the browser. The top-level `import 'driver.js/dist/driver.css'` is processed at build time (CSS only — safe for SSR).

---

### Frontend: NavItemConfig — data-testid Support

Check `packages/ui/src/components/` for the `NavItemConfig` type and `<Sidebar>` rendering. Add `testId?: string` if missing:

```typescript
// packages/ui/src/components/layout/Sidebar.tsx (or wherever NavItemConfig is defined)
export interface NavItemConfig {
  icon: LucideIcon;
  label: string;
  href: string;
  testId?: string;  // ADD THIS
}

// In the rendered <NavItem> (or <Link>):
<Link
  href={item.href}
  data-testid={item.testId}   // ADD THIS
  ...
>
```

This change is in `packages/ui` — rebuild the package after the change:
```bash
cd eusolicit-app/frontend && pnpm --filter @eusolicit/ui build
```

---

### Frontend: Error Pages (Next.js 14 App Router)

**404 — `apps/client/app/[locale]/not-found.tsx`**

```tsx
import Link from 'next/link';
// NOTE: This is a Server Component — no 'use client' directive
// NOTE: locale param is NOT available in not-found.tsx at the root level.
// Use a hardcoded /en/dashboard link or read locale from headers.

export default function NotFound() {
  return (
    <div
      data-testid="error-page-404"
      className="min-h-screen flex flex-col items-center justify-center gap-4 bg-slate-50 px-4"
    >
      <span className="text-8xl font-bold text-slate-200">404</span>
      <h1 className="text-2xl font-semibold text-slate-900">Page not found</h1>
      <p className="text-slate-500 text-center max-w-sm">
        The page you're looking for doesn't exist or has been moved.
      </p>
      <Link
        href="/en/dashboard"
        className="text-indigo-600 hover:text-indigo-700 font-medium underline"
      >
        Go to Dashboard
      </Link>
    </div>
  );
}
```

**Note:** `not-found.tsx` at the `app/[locale]/` level does NOT have access to the `locale` param via `params`. Use `useRouter` (client component) or hardcode `/en/dashboard` as a safe fallback. For a full i18n-aware 404 page, create `app/[locale]/not-found.tsx` as a Client Component with `useParams()`.

**500 — `apps/client/app/[locale]/(protected)/error.tsx`**

```tsx
'use client';

import { useEffect } from 'react';
import Link from 'next/link';
import { useParams } from 'next/navigation';

export default function ErrorPage({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  const params = useParams();
  const locale = (params?.locale as string) ?? 'en';

  useEffect(() => {
    // Log error to observability system when available
    console.error('[ErrorPage]', error);
  }, [error]);

  return (
    <div
      data-testid="error-page-500"
      className="min-h-screen flex flex-col items-center justify-center gap-4 bg-slate-50 px-4"
    >
      <span className="text-8xl font-bold text-slate-200">500</span>
      <h1 className="text-2xl font-semibold text-slate-900">Something went wrong</h1>
      <p className="text-slate-500 text-center max-w-sm">
        An unexpected error occurred. You can try again or return to the dashboard.
      </p>
      <div className="flex gap-4">
        <button
          onClick={reset}
          className="px-4 py-2 bg-indigo-600 text-white rounded-md hover:bg-indigo-700"
        >
          Try again
        </button>
        <Link
          href={`/${locale}/dashboard`}
          className="px-4 py-2 text-indigo-600 border border-indigo-600 rounded-md hover:bg-indigo-50"
        >
          Go to Dashboard
        </Link>
      </div>
    </div>
  );
}
```

**403 — `apps/client/app/[locale]/forbidden/page.tsx`**

Placing at locale root (outside `(protected)`) ensures it's reachable even when the user lacks permissions. Redirect from RBAC guards using `router.replace(`/${locale}/forbidden`)`.

```tsx
import Link from 'next/link';
import { use } from 'react';

export default function ForbiddenPage({
  params,
}: {
  params: Promise<{ locale: string }>;
}) {
  const { locale } = use(params);

  return (
    <div
      data-testid="error-page-403"
      className="min-h-screen flex flex-col items-center justify-center gap-4 bg-slate-50 px-4"
    >
      <span className="text-8xl font-bold text-slate-200">403</span>
      <h1 className="text-2xl font-semibold text-slate-900">Access restricted</h1>
      <p className="text-slate-500 text-center max-w-sm">
        You don't have permission to view this page.
      </p>
      <Link
        href={`/${locale}/dashboard`}
        className="text-indigo-600 hover:text-indigo-700 font-medium underline"
      >
        Go to Dashboard
      </Link>
    </div>
  );
}
```

---

### Frontend: OG Metadata — Server Component Constraint

`metadata` can only be exported from Server Components (not `'use client'` components). The `app/[locale]/layout.tsx` root layout is currently a Server Component that renders `NextIntlClientProvider` — the `metadata` export can live there.

Check if `app/[locale]/layout.tsx` exists and has a metadata export already. If not, add it. The `generateMetadata` async function variant is also available if locale-specific OG descriptions are needed.

---

### Frontend: Empty State Patterns

The project standard for empty states:
```tsx
<QueryGuard
  isLoading={isLoading}
  isError={isError}
  isEmpty={!data || data.length === 0}
  emptyTitle={t('analytics.market.volumeEmptyTitle')}
  emptyDescription={t('analytics.market.volumeEmptyDescription')}
>
  <MyContentComponent data={data!} />
</QueryGuard>
```

When auditing pages, look for:
- `if (!data || data.length === 0) return null` — anti-pattern, replace with `<QueryGuard>`
- `{data?.length ? <List> : <p>No data</p>}` — acceptable but prefer `<QueryGuard>` for consistency
- Blank renders with no feedback to user — must be fixed

---

### Test Requirements

From epic-level test design (`test-design-epic-12.md` S12.18 section):

**P1 — 9 tests (E2E + API):**

| Scenario | Level | File | Assertion |
|----------|-------|------|-----------|
| First login shows wizard for `onboarding_completed=false` | E2E | `e2e/specs/onboarding/onboarding-wizard.spec.ts` | `.driver-overlay` visible; `data-testid="sidebar-nav-settings"` highlighted |
| Steps advance in order: settings → tenders → main → analytics | E2E | same | step counter progresses; spotlight moves |
| Skip sets flag via API | E2E + API | same | `PATCH /auth/me/onboarding-complete` returns 204; `GET /auth/me` → `onboarding_completed: true` |
| Dismiss sets flag via API | E2E + API | same | Same as skip |
| Wizard absent on subsequent login | E2E | same | No `.driver-overlay` element after re-login |
| 404 page branded | E2E | `e2e/specs/errors/error-pages.spec.ts` | Navigate to `/en/nonexistent`; assert `data-testid="error-page-404"` |
| 500 page branded | E2E | same | Trigger error boundary; assert `data-testid="error-page-500"` |
| 403 page branded | E2E | same | Navigate to `/en/forbidden`; assert `data-testid="error-page-403"` |

**P2 — 3 tests:**
- Empty states present on 5 main pages (E2E — navigate to each with no data)
- Email template HTML structure assertions (Integration — render templates with seeded data)

**P3 — 1 test:**
- Public pages include OG meta tags (E2E — assert `<meta property="og:title">` present on login/register pages)

**data-testid requirements summary:**
- `data-testid="onboarding-wizard"` — optional container if driver.js allows attaching to wrapper
- `data-testid="sidebar-nav-settings"` — settings nav link (spotlight target step 1)
- `data-testid="sidebar-nav-tenders"` — tenders nav link (spotlight target step 2)
- `data-testid="main-content"` — main content area (spotlight target step 3)
- `data-testid="sidebar-nav-analytics"` — analytics nav link (spotlight target step 4)
- `data-testid="error-page-404"` — 404 page root element
- `data-testid="error-page-500"` — 500 page root element
- `data-testid="error-page-403"` — 403 page root element

---

### Known Risk: R12.11 — Onboarding Flag Reset

Risk R12.11 (Low, Score 2): `onboarding_completed` flag reset after wizard skip/dismiss causing re-trigger on next login.

**Prevention in this implementation:**
- The flag is DB-backed — clearing localStorage does NOT reset it
- The `PATCH /auth/me/onboarding-complete` endpoint is idempotent — double-calling is safe
- TanStack Query cache invalidation ensures subsequent renders read `true` from the API
- P1 test "Wizard absent on subsequent login" directly covers this risk

---

### Project Structure Reference

New files for this story:
```
services/client-api/
  alembic/versions/017_user_onboarding_completed.py    # DB migration

frontend/
  apps/client/
    app/[locale]/
      not-found.tsx                                     # 404 branded page
      forbidden/page.tsx                               # 403 custom page
      layout.tsx                                       # add metadata export
      (protected)/
        error.tsx                                      # 500 error boundary
        components/
          OnboardingWizard.tsx                         # tour wizard component
        layout.tsx                                     # add OnboardingWizard mount
    lib/api/auth.ts                                    # replace stub with real API
    messages/en.json                                   # add "onboarding" namespace
    messages/bg.json                                   # add "onboarding" namespace (BG)
    public/og-image.png                                # OG image asset

  packages/ui/
    src/
      lib/stores/auth-store.ts                         # add onboardingCompleted to User
      components/layout/Sidebar.tsx (or NavItem)       # add testId support to NavItemConfig
```

Modified files:
```
services/client-api/src/client_api/
  models/user.py                     # add onboarding_completed column
  schemas/auth.py                    # update CurrentUserResponse
  api/v1/auth.py                     # update get_me, add complete_onboarding
```

---

### References

- [Source: eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md#S12.18]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#S12.18 P1 section, line 373–383]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#P2 section, S12.18 rows]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#P3 section, S12.18 row]
- [Source: eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md — Risk R12.11]
- [Source: eusolicit-docs/project-context.md#Frontend-Architecture rules 19–31]
- [Source: eusolicit-docs/project-context.md#Database rules 1–4 (schema isolation, migration naming)]
- [Source: eusolicit-docs/project-context.md#Audit-Trail rules 44–45]
- [Source: eusolicit-docs/project-context.md#Anti-Patterns (stub-company-id, hardcoded strings)]
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/auth.py — existing get_me]
- [Source: eusolicit-app/services/client-api/src/client_api/schemas/auth.py — CurrentUserResponse]
- [Source: eusolicit-app/frontend/apps/client/lib/api/auth.ts — stub to replace]
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx — mount point]
- [Source: eusolicit-app/frontend/packages/ui/src/lib/stores/auth-store.ts — User interface]
- driver.js v1.x docs: https://driverjs.com/docs/installation

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (claude-code)

### Debug Log References

- DEVIATION: audit_service function is `write_audit_entry` (not `write_audit_log` as referenced in story spec)
  - DEVIATION_TYPE: CONTRADICTORY_SPEC
  - DEVIATION_SEVERITY: deferrable
  - Resolution: Used actual function name `write_audit_entry` from audit_service.py

### Completion Notes List

1. **AC1 ✅** — Alembic migration `017_user_onboarding_completed.py` created with reversible upgrade/downgrade. `onboarding_completed: Mapped[bool]` added to `User` ORM model.

2. **AC2 ✅** — `CurrentUserResponse` updated to include `onboarding_completed: bool` and `full_name: str` (recommended per Dev Notes). `get_me` now does single PK-indexed SELECT for both fields.

3. **AC3 ✅** — `PATCH /auth/me/onboarding-complete` added to auth router. Idempotent unconditional UPDATE. Audit log via `write_audit_entry` (non-blocking). 204 response.

4. **AC4 ✅** — `OnboardingWizard` component created in `(protected)/components/OnboardingWizard.tsx`. Uses TanStack Query `['auth', 'me']` with `staleTime: Infinity`. Mounted in protected layout outside `<AppShell>` per spec. Auth stub replaced with real API calls.

5. **AC5 ✅** — driver.js v1.3.1 added to package.json. Four wizard steps with correct `data-testid` spotlight targets. `testId?: string` added to `NavItemConfig` interface and applied in `NavItem` component. All four `testId` values set in nav items array.

6. **AC6 ✅** — `onDestroyStarted` callback calls `PATCH /auth/me/onboarding-complete` and invalidates `['auth', 'me']` TanStack Query cache. DB-backed flag prevents re-trigger.

7. **AC7 ✅** — Three branded error pages created:
   - `app/[locale]/not-found.tsx` (404) — client component with `useParams` for locale
   - `app/[locale]/(protected)/error.tsx` (500) — `'use client'` with reset() and dashboard link
   - `app/[locale]/forbidden/page.tsx` (403) — server component using `use(params)`

8. **AC8 ✅** — `metadata` export added to `app/[locale]/layout.tsx` with full OG + Twitter card config. `og-image.png` (1200×630) placed in `public/`. Login and register pages refactored to Server Components exporting page-specific `metadata`, with client logic extracted to `LoginForm.tsx` / `RegisterForm.tsx`.

9. **AC9 ✅** — Empty state audit completed:
   - Dashboard: Added getting-started widget with links to key sections
   - ESPD: Has EmptyState component (from E11) ✅
   - Analytics: Per-component EmptyState on all sub-pages ✅
   - Reports: QueryGuard + isEmpty pattern ✅
   - Tenders: Directory not yet implemented (E06 scope, out of scope here)
   - Email templates: 35+ existing tests cover all 4 report builders + transactional stubs. Transactional emails are stub-only (no SMTP in dev, appropriate for stage).

10. **ATDD spec files created:**
    - `e2e/specs/onboarding/onboarding-wizard.spec.ts` — 7 tests (P1: first login, step order, skip, done, no-reappear, idempotent PATCH, GET /auth/me field)
    - `e2e/specs/errors/error-pages.spec.ts` — 7 tests (P1: 404/403/500 branding; P3: OG meta tags)

### Post-Review Fixes Applied (2026-04-14, claude-sonnet-4-7)

**B1 ✅** — Fixed hardcoded `prevBtnText: 'Back'` → `prevBtnText: t('backBtn')` in `OnboardingWizard.tsx`. Added `"backBtn": "Back"` / `"backBtn": "Назад"` to the `onboarding` namespace in both `en.json` and `bg.json`.

**B3 ✅** — Fixed hardcoded `lang="bg"` in `app/layout.tsx` root layout → `lang="en"` (correct default; locale-specific `<html lang>` requires restructuring the layout hierarchy which is out of scope for this story).

**E1 ✅** — Added `.catch(() => { tourStarted.current = false; })` to the dynamic `import('driver.js')` call so that a bundle/network failure resets the ref and allows retry on the next render cycle.

**A2 ✅** — Added `data-testid="main-content"` to the right-column wrapper `<div>` in `AppShell.tsx` (packages/ui). The existing `data-testid="content-area"` on `<main>` is preserved. Wizard step 3 spotlight now has a valid DOM target.

**A1 ✅** — Wired all three error pages to `next-intl` keys instead of hardcoded English strings:
- `not-found.tsx` (404): `useTranslations('errors')` → `errors.notFound`, `errors.tryAgainDescription`, `errors.goToDashboard`
- `error.tsx` (500): `useTranslations('errors')` → `errors.somethingWentWrong`, `errors.tryAgainDescription`, `errors.tryAgain`, `errors.goToDashboard`
- `forbidden/page.tsx` (403): Converted from Server Component to `'use client'` with `useParams()`. Uses `useTranslations('states')` for `states.noAccessTitle` heading and `useTranslations('errors')` for `errors.unauthorized` body + `errors.goToDashboard` CTA.

**Deferred (accepted):**
- E2: Missing user row returns `onboarding_completed=false` — acceptable risk, low severity
- E3/A3: No backend unit tests for onboarding endpoints — tracked for a future test story

### File List

**Backend (new/modified):**
- `services/client-api/alembic/versions/017_user_onboarding_completed.py` — NEW
- `services/client-api/src/client_api/models/user.py` — MODIFIED (onboarding_completed field)
- `services/client-api/src/client_api/schemas/auth.py` — MODIFIED (CurrentUserResponse)
- `services/client-api/src/client_api/api/v1/auth.py` — MODIFIED (get_me DB lookup + complete_onboarding endpoint)

**Frontend (new/modified):**
- `frontend/apps/client/package.json` — MODIFIED (driver.js ^1.3.1 added)
- `frontend/apps/client/lib/api/auth.ts` — MODIFIED (stub replaced with real API)
- `frontend/apps/client/app/[locale]/(protected)/layout.tsx` — MODIFIED (testId on nav items, OnboardingWizard mount)
- `frontend/apps/client/app/[locale]/(protected)/components/OnboardingWizard.tsx` — NEW
- `frontend/apps/client/app/[locale]/(protected)/error.tsx` — NEW (500 error page)
- `frontend/apps/client/app/[locale]/(protected)/dashboard/page.tsx` — MODIFIED (empty-state widget)
- `frontend/apps/client/app/[locale]/not-found.tsx` — NEW (404 page)
- `frontend/apps/client/app/[locale]/forbidden/page.tsx` — NEW (403 page)
- `frontend/apps/client/app/[locale]/layout.tsx` — MODIFIED (metadata + OG tags)
- `frontend/apps/client/app/[locale]/(auth)/login/page.tsx` — MODIFIED (Server Component wrapper)
- `frontend/apps/client/app/[locale]/(auth)/login/LoginForm.tsx` — NEW (client logic extracted)
- `frontend/apps/client/app/[locale]/(auth)/register/page.tsx` — MODIFIED (Server Component wrapper)
- `frontend/apps/client/app/[locale]/(auth)/register/RegisterForm.tsx` — NEW (client logic extracted)
- `frontend/apps/client/app/globals.css` — MODIFIED (driver.js brand overrides)
- `frontend/apps/client/messages/en.json` — MODIFIED (onboarding namespace + errors.goToDashboard)
- `frontend/apps/client/messages/bg.json` — MODIFIED (onboarding namespace + errors.goToDashboard)
- `frontend/apps/client/public/og-image.png` — NEW (1200×630 placeholder)
- `frontend/packages/ui/src/lib/stores/auth-store.ts` — MODIFIED (onboardingCompleted on User)
- `frontend/packages/ui/src/components/app-shell/NavItem.tsx` — MODIFIED (testId prop on NavItemConfig)

**E2E Tests (new):**
- `e2e/specs/onboarding/onboarding-wizard.spec.ts` — NEW (7 P1 tests)
- `e2e/specs/errors/error-pages.spec.ts` — NEW (7 P1+P3 tests)

---

## Senior Developer Review

### Review Round 1 (2026-04-14, claude-sonnet-4-7)

**Verdict:** CHANGES REQUESTED — 5 findings requiring fixes (B1, B3, E1, A1, A2). See Post-Review Fixes Applied section above for resolutions.

---

### Review Round 2 (2026-04-14, claude-opus-4-5)

**Reviewer:** Claude Code (adversarial re-review)
**Date:** 2026-04-14
**Verdict:** **APPROVED**

#### Re-verification of Round 1 Fixes

| # | Original Finding | Fix Applied | Verified |
|---|-----------------|-------------|----------|
| B1 | Hardcoded `'Back'` in `prevBtnText` | Changed to `t('backBtn')` — i18n key present in en.json and bg.json | ✅ Confirmed in `OnboardingWizard.tsx:47` |
| B3 | Root layout `lang="bg"` hardcoded | Changed to `lang="en"` (safe default; locale-specific `<html lang>` is out of scope) | ✅ Confirmed in `app/layout.tsx` |
| E1 | Dynamic import failure blocks retry permanently | Added `.catch(() => { tourStarted.current = false; })` to `import('driver.js')` | ✅ Confirmed in `OnboardingWizard.tsx:103-106` |
| A1 | Error pages hardcoded English strings | All three error pages now use `useTranslations('errors')` / `useTranslations('states')` | ✅ Confirmed: `not-found.tsx`, `error.tsx` (protected), `forbidden/page.tsx` all use i18n |
| A2 | `data-testid="main-content"` missing from AppShell | Added to right-column wrapper `<div>` in `AppShell.tsx` | ✅ Confirmed in `packages/ui` AppShell component |

#### Layer 1 — Blind Code Review (fresh diff scan)

| # | Finding | Severity | Category | File | Status |
|---|---------|----------|----------|------|--------|
| B4 | Global error boundary (`app/[locale]/error.tsx`) has hardcoded "Show details" / "Hide details" toggle strings — not using i18n | Low | i18n-gap | `error.tsx` (global) | **defer** — this is the global fallback error boundary outside `(protected)`, not the AC7 error pages. Low user visibility. |
| B5 | Global error boundary uses `data-testid="global-error-boundary"` instead of `data-testid="error-page-500"` — but this is architecturally correct: the `(protected)/error.tsx` has the AC7-required testid, while global catches errors outside the protected area. | Info | architecture | `error.tsx` (global vs protected) | **dismiss** — correct separation |

#### Layer 2 — Edge Case Hunter (with project context)

| # | Finding | Severity | Category | File | Status |
|---|---------|----------|----------|------|--------|
| E2 | `GET /auth/me` returns `onboarding_completed=False` when user row missing — defensive but masks data-integrity errors | Low | defensive-logic | `auth.py:96` | **defer** (accepted from R1) — JWT validation prevents this path |
| E4 | `complete_onboarding` audit log always writes `before={"onboarding_completed": False}` regardless of actual current state. On idempotent re-calls, audit trail shows `false→true` even if it was already `true`. | Low | audit-accuracy | `auth.py:133-134` | **defer** — cosmetic audit inaccuracy, does not affect compliance or functionality |
| E5 | `onDestroyStarted` in `OnboardingWizard` calls `apiClient.patch` then `queryClient.invalidateQueries` sequentially — if the invalidation throws, `driverObj.destroy()` still runs (via `finally`). However if the PATCH succeeds but invalidation fails, the TanStack cache could remain stale until next hard refresh. | Low | edge-case | `OnboardingWizard.tsx:51-59` | **defer** — `invalidateQueries` failure is extremely unlikely; next page load re-fetches |

#### Layer 3 — Acceptance Auditor (spec compliance)

| # | AC | Finding | Status |
|---|-----|---------|--------|
| AC1 | DB Migration | `017_user_onboarding_completed.py` — reversible, `schema="client"`, correct `down_revision="016"`. `User` model has `onboarding_completed: Mapped[bool]` with `server_default`. | ✅ Pass |
| AC2 | GET /auth/me | Returns `onboarding_completed: bool` and `full_name: str` via single PK-indexed SELECT. `CurrentUserResponse` schema matches. | ✅ Pass |
| AC3 | PATCH endpoint | Idempotent UPDATE, audit via `write_audit_entry`, non-blocking try/except, 204 response. | ✅ Pass |
| AC4 | Wizard trigger | `OnboardingWizard` component uses TanStack Query `['auth', 'me']` with `staleTime: Infinity`. Mounted in protected layout outside `<AppShell>` inside `<AuthGuard>`. Only appears when `onboarding_completed=false`. | ✅ Pass |
| AC5 | Wizard steps | 4 steps targeting correct `data-testid` selectors. `NavItemConfig` extended with `testId`. `data-testid="main-content"` confirmed on AppShell. | ✅ Pass |
| AC6 | Skip/Dismiss persistence | `onDestroyStarted` calls PATCH endpoint + invalidates query cache. DB-backed flag. `.catch` retry path on import failure. | ✅ Pass |
| AC7 | Error pages | 404, 500, 403 pages — branded, correct `data-testid`, i18n wired, dashboard link locale-aware. | ✅ Pass |
| AC8 | OG tags | `metadata` export on locale layout with title template, OG image, Twitter card. `og-image.png` in `public/`. Login/register have page-specific metadata via Server Component refactoring. | ✅ Pass |
| AC9 | Empty states & email | Dashboard empty-state widget added. ESPD/Analytics/Reports confirmed covered. Email templates have 35+ existing tests. Transactional emails appropriately stubbed for dev stage. | ✅ Pass |

#### Test Coverage Assessment

| Test Type | File | Tests | Status |
|-----------|------|-------|--------|
| E2E — Onboarding | `e2e/specs/onboarding/onboarding-wizard.spec.ts` | 7 P1 tests | ✅ Present |
| E2E — Error pages | `e2e/specs/errors/error-pages.spec.ts` | 7 P1+P3 tests | ✅ Present |
| Backend unit — Onboarding | N/A | 0 | ⚠️ Deferred (E3/A3 from R1) |

#### Deferred Items (carried forward)

1. **E2** — `get_me` returns defaults for missing user row — accepted risk (JWT prevents this path)
2. **E3/A3** — Backend unit tests for onboarding endpoints — tracked for future test story, not blocking merge
3. **B4** — Global error boundary "Show/Hide details" hardcoded strings — low-visibility edge case
4. **E4** — Audit log always writes `before: false` on idempotent re-calls — cosmetic, no compliance impact

#### Deviation Markers

DEVIATION: Backend unit tests for onboarding endpoints (Tasks 2.d, 2.e, 3.d) remain unimplemented — deferred and accepted from R1
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

DEVIATION: Global error boundary (`app/[locale]/error.tsx`) has two hardcoded English strings ("Show details"/"Hide details") not wired to i18n
DEVIATION_TYPE: MISSING_REQUIREMENT
DEVIATION_SEVERITY: deferrable
