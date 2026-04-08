# Story 3.8: Authentication Pages

Status: done

## Story

As a **frontend developer on the EU Solicit team**,
I want **four authentication pages (Login, Register, Forgot Password, OAuth Callback) built in the `(auth)` route group with a centred card layout, Zod validation, loading states, `next-intl` translations, and stubbed E02 API calls**,
so that **users can authenticate through the EU Solicit client app, the `authStore` is populated on successful login, and auth flows integrate cleanly with the route guard infrastructure being prepared for Story 3.12**.

## Acceptance Criteria

1. Auth layout `app/[locale]/(auth)/layout.tsx` renders: no sidebar, no top bar; full-page centred flex container (slate-50 background); EU Solicit logo text (`text-2xl font-bold text-indigo-600`) above a `max-w-md` white card (`rounded-xl shadow-md p-8`); children rendered inside the card
2. `/login` page: email + password fields; "Remember me" checkbox; "Forgot password?" link → `/${locale}/forgot-password`; "Sign in" primary button; divider + Google OAuth outline button; "Don't have an account?" link → `/${locale}/register`; submits to stubbed `POST /auth/login`; on success calls `authStore.login()` and redirects to `/${locale}/dashboard`
3. `/register` page: company name, EIK (validates 9 or 13-digit numeric via regex), first name, last name, email, password, confirm password, terms checkbox; "Already have an account?" link → `/${locale}/login`; submits to stubbed `POST /auth/register`; on success calls `authStore.login()` and redirects to `/${locale}/dashboard`
4. `/forgot-password` page: email field; "Send reset link" button; submits to stubbed `POST /auth/forgot-password`; on success the form is hidden and a "Check your email" success state with description text and "Back to sign in" link is shown
5. `/auth/callback` page (OAuth callback): `"use client"` component inside a `<Suspense>` boundary; reads `code` and `state` from `useSearchParams()`; renders `data-testid="auth-callback-loading"` processing state while exchanging; stubs token exchange via `exchangeOAuthCode()`; on success calls `authStore.login()` and redirects to `/${locale}/dashboard`; shows error state if `code` is absent or exchange throws
6. All forms validate inline on blur and on submit: `useZodForm(schema, { mode: "onBlur" })` from `packages/ui` (S03.06) + `<FormField>` component; Zod error messages appear below each field in red-500 text via `<FormField>`'s built-in error rendering
7. Loading states: the submit button renders `<Loader2 className="animate-spin mr-2 h-4 w-4" />` and is `disabled` during submission; all `<FormField>` inputs pass `disabled={form.formState.isSubmitting}`
8. Authenticated redirect: auth pages render `null` before Zustand hydration (`!mounted`); after mount, if `authStore.isAuthenticated` is `true`, `router.replace(`/${locale}/dashboard`)` fires — no flash of auth page content for already-logged-in users
9. Seven new `auth` namespace keys added to both `messages/en.json` and `messages/bg.json` in `apps/client`: `googleSignIn`, `noAccount`, `haveAccount`, `orContinueWith`, `processingAuth`, `sendResetLink`, `backToSignIn`; all visible auth page text (headings, labels, links, buttons, success messages) uses `useTranslations("auth")` or `useTranslations("errors")`; no hardcoded English strings
10. `pnpm build` exits 0 for both apps; `pnpm type-check` exits 0 across all packages; `pnpm check:i18n --filter client` exits 0 (key parity verified)

## Tasks / Subtasks

- [x] Task 1: Create auth route group layout (AC: 1)
  - [x] 1.1 Create `apps/client/app/[locale]/(auth)/layout.tsx` — server component; centred card layout with EU Solicit logo text; no sidebar or top bar; see Dev Notes for full implementation

- [x] Task 2: Create Zod validation schemas (AC: 2, 3, 4, 6)
  - [x] 2.1 Create `apps/client/lib/schemas/auth.ts` — `loginSchema`, `registerSchema`, `forgotPasswordSchema` with all field validations; see Dev Notes for full implementation

- [x] Task 3: Create stubbed auth API functions (AC: 2, 3, 4, 5)
  - [x] 3.1 Create `apps/client/lib/api/auth.ts` — `loginUser()`, `registerUser()`, `requestPasswordReset()`, `exchangeOAuthCode()` stub functions using `setTimeout` delay (800ms); see Dev Notes for full implementation

- [x] Task 4: Create Login page (AC: 2, 6, 7, 8, 9)
  - [x] 4.1 Create `apps/client/app/[locale]/(auth)/login/page.tsx` — `"use client"` page with login form, Google OAuth button, authenticated redirect; see Dev Notes for full implementation

- [x] Task 5: Create Register page (AC: 3, 6, 7, 8, 9)
  - [x] 5.1 Create `apps/client/app/[locale]/(auth)/register/page.tsx` — `"use client"` page with full registration form; see Dev Notes for full implementation

- [x] Task 6: Create Forgot Password page (AC: 4, 6, 7, 9)
  - [x] 6.1 Create `apps/client/app/[locale]/(auth)/forgot-password/page.tsx` — `"use client"` page with email field and conditional success state; see Dev Notes for full implementation

- [x] Task 7: Create OAuth Callback page (AC: 5, 9)
  - [x] 7.1 Create `apps/client/app/[locale]/(auth)/callback/page.tsx` — exports a server-compatible default with a `<Suspense>` wrapper around the inner `"use client"` component that reads `useSearchParams()`; see Dev Notes for full implementation

- [x] Task 8: Add missing i18n keys (AC: 9)
  - [x] 8.1 Add 7 new `auth` namespace keys to `apps/client/messages/en.json`: `googleSignIn`, `noAccount`, `haveAccount`, `orContinueWith`, `processingAuth`, `sendResetLink`, `backToSignIn`; see Dev Notes for exact values
  - [x] 8.2 Add corresponding BG translations to `apps/client/messages/bg.json`; see Dev Notes for exact values
  - [x] 8.3 Run `pnpm check:i18n --filter client` from `eusolicit-app/frontend/` — must exit 0

- [x] Task 9: Build and type-check verification (AC: 10)
  - [x] 9.1 Run `pnpm build` from `eusolicit-app/frontend/` — both `client` and `admin` apps must exit 0
  - [x] 9.2 Run `pnpm type-check` from `eusolicit-app/frontend/` — zero TypeScript errors across all packages

## Dev Notes

### Working Directory

All frontend code lives under: `eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run pnpm commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

### Critical Learnings from Stories 3.1–3.7 (MUST APPLY)

1. **`@/*` maps to `./` not `./src/`** — Inside `apps/client`, `@/lib/schemas/auth` resolves to `apps/client/lib/schemas/auth.ts`. Inside `packages/ui`, always use **relative paths** — the `@/` alias does NOT work inside `packages/ui`.
2. **`next.config.mjs` not `.ts`** — Both apps use `.mjs`. Do NOT create `.ts` next configs. The `withNextIntl` plugin is already in place from S3.7 — no config change needed for this story.
3. **Route group `(auth)` is a sibling of `(protected)` inside `app/[locale]/`** — The parentheses mean the folder is a route group and has no URL segment. Auth pages live at `app/[locale]/(auth)/login/page.tsx`, which maps to the URL `/{locale}/login`.
4. **`useAuthStore` is imported from `@eusolicit/ui`** — The auth store lives in `packages/ui/src/lib/stores/auth-store.ts` and is exported from `packages/ui`. Do NOT create a local `auth-store.ts` in `apps/client`. Import: `import { useAuthStore } from "@eusolicit/ui";`
5. **`useZodForm` and `FormField` imported from `@eusolicit/ui`** — Both established in S03.06 and exported from `packages/ui`. Import: `import { useZodForm, FormField } from "@eusolicit/ui";`
6. **Locale prefix in all in-app links** — Every `<Link href>` and `router.replace()` call must use `/${locale}/` prefix (e.g., `/${locale}/login`, `/${locale}/dashboard`). Never use bare paths like `/login` — they would bypass the `[locale]` segment.
7. **`useSearchParams()` must be inside a `<Suspense>` boundary** — Next.js App Router throws a build error if `useSearchParams()` is called in a component that is not wrapped in `<Suspense>`. The callback page uses an inner `CallbackPageContent` client component wrapped in `<Suspense>` from the page default export.
8. **`suppressHydrationWarning` stays on `<html>` only** — Already in root `app/layout.tsx`. Do NOT add it to auth layout or page components.
9. **Auth layout is a server component** — No `"use client"` directive on `(auth)/layout.tsx`. The layout itself is static (logo + card wrapper). The page children are client components.
10. **Zod schemas live in `apps/client/lib/schemas/`** — This is the established pattern from S03.06 (`lib/schemas/`). Do NOT put schemas in `packages/ui`. Export types alongside schemas (`LoginInput`, `RegisterInput`, `ForgotPasswordInput`).
11. **`useZodForm` mode for blur validation** — Pass `{ mode: "onBlur" }` as the second argument to `useZodForm(schema, { mode: "onBlur" })` so errors appear on blur AND on submit, satisfying AC6.
12. **`mounted` guard prevents hydration flash** — The `isAuthenticated` check for redirect runs after Zustand rehydrates from localStorage. The `mounted` state pattern (set in `useEffect`) ensures we don't redirect or render protected state prematurely.

### Architecture: File Structure Added in S03.08

```
apps/client/
  app/
    [locale]/
      (auth)/                        ← NEW: route group (no URL segment)
        layout.tsx                   ← NEW: centred card layout, no sidebar
        login/
          page.tsx                   ← NEW: login form
        register/
          page.tsx                   ← NEW: registration form
        forgot-password/
          page.tsx                   ← NEW: forgot password + success state
        callback/
          page.tsx                   ← NEW: OAuth callback handler
  lib/
    schemas/
      auth.ts                        ← NEW: loginSchema, registerSchema, forgotPasswordSchema
    api/
      auth.ts                        ← NEW: stub API functions
  messages/
    en.json                          ← MODIFIED: add 7 new auth namespace keys
    bg.json                          ← MODIFIED: add 7 new auth namespace keys (BG)
```

### Auth URL Routes (with locale prefix)

| Page | URL (BG default) |
|------|-----------------|
| Login | `/bg/login` or `/en/login` |
| Register | `/bg/register` or `/en/register` |
| Forgot Password | `/bg/forgot-password` or `/en/forgot-password` |
| OAuth Callback | `/bg/callback?code=...&state=...` |
| Dashboard (redirect target) | `/bg/dashboard` or `/en/dashboard` |

### `useAuthStore` Interface (from S03.05 — `packages/ui`)

```typescript
// Already available — import { useAuthStore, type User } from "@eusolicit/ui"
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
```

### Implementation Code: `app/[locale]/(auth)/layout.tsx`

```tsx
// apps/client/app/[locale]/(auth)/layout.tsx
// Server component — no "use client"
import type { ReactNode } from "react";

export default function AuthLayout({ children }: { children: ReactNode }) {
  return (
    <div className="min-h-screen bg-slate-50 flex flex-col items-center justify-center p-4">
      {/* EU Solicit logo */}
      <div className="mb-8 text-center">
        <span className="text-2xl font-bold text-indigo-600">EU Solicit</span>
      </div>
      {/* Card */}
      <div className="w-full max-w-md bg-white rounded-xl shadow-md p-8">
        {children}
      </div>
    </div>
  );
}
```

### Implementation Code: `lib/schemas/auth.ts`

```typescript
// apps/client/lib/schemas/auth.ts
import { z } from "zod";

export const loginSchema = z.object({
  email: z
    .string()
    .min(1, "This field is required")
    .email("Please enter a valid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
  rememberMe: z.boolean().optional().default(false),
});

export const registerSchema = z
  .object({
    companyName: z.string().min(2, "This field is required"),
    eik: z
      .string()
      .regex(/^\d{9}(\d{4})?$/, "Please enter a valid 9 or 13 digit EIK"),
    firstName: z.string().min(1, "This field is required"),
    lastName: z.string().min(1, "This field is required"),
    email: z
      .string()
      .min(1, "This field is required")
      .email("Please enter a valid email address"),
    password: z.string().min(8, "Password must be at least 8 characters"),
    confirmPassword: z.string().min(1, "This field is required"),
    agreeToTerms: z.literal(true, {
      errorMap: () => ({ message: "This field is required" }),
    }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"],
  });

export const forgotPasswordSchema = z.object({
  email: z
    .string()
    .min(1, "This field is required")
    .email("Please enter a valid email address"),
});

export type LoginInput = z.infer<typeof loginSchema>;
export type RegisterInput = z.infer<typeof registerSchema>;
export type ForgotPasswordInput = z.infer<typeof forgotPasswordSchema>;
```

**Note on i18n of Zod messages**: Zod schema error strings are in English here. Full locale-aware Zod validation (BG error messages) requires either passing translated messages at runtime or creating schemas inside the component. For MVP, English-only Zod errors are acceptable — the forms namespace in `messages/*.json` already has matching translations for when forms are later integrated with a translated schema factory. This is a known deferred item.

### Implementation Code: `lib/api/auth.ts` (stubs)

```typescript
// apps/client/lib/api/auth.ts
// Stub implementations — replace with real apiClient calls when E02 is deployed.
// apiClient is available from "@eusolicit/ui" (established in S03.05).

export interface LoginRequest {
  email: string;
  password: string;
  rememberMe?: boolean;
}

export interface AuthResponse {
  user: {
    id: string;
    email: string;
    name: string;
    companyId: string;
    role: string;
  };
  token: string;
  refreshToken: string;
}

export interface RegisterRequest {
  companyName: string;
  eik: string;
  firstName: string;
  lastName: string;
  email: string;
  password: string;
}

const STUB_DELAY_MS = 800;

function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

export async function loginUser(data: LoginRequest): Promise<AuthResponse> {
  await delay(STUB_DELAY_MS);
  // TODO: Replace with apiClient.post<AuthResponse>('/auth/login', data) when E02 is deployed
  return {
    user: {
      id: "stub-user-id",
      email: data.email,
      name: "Demo User",
      companyId: "stub-company-id",
      role: "owner",
    },
    token: "stub-jwt-token",
    refreshToken: "stub-refresh-token",
  };
}

export async function registerUser(data: RegisterRequest): Promise<AuthResponse> {
  await delay(STUB_DELAY_MS);
  // TODO: Replace with apiClient.post<AuthResponse>('/auth/register', data) when E02 is deployed
  return {
    user: {
      id: "stub-user-id",
      email: data.email,
      name: `${data.firstName} ${data.lastName}`,
      companyId: "stub-company-id",
      role: "owner",
    },
    token: "stub-jwt-token",
    refreshToken: "stub-refresh-token",
  };
}

export async function requestPasswordReset(email: string): Promise<void> {
  await delay(STUB_DELAY_MS);
  // TODO: Replace with apiClient.post('/auth/forgot-password', { email }) when E02 is deployed
  void email; // suppress unused variable warning in strict mode
}

export async function exchangeOAuthCode(
  code: string,
  state: string
): Promise<AuthResponse> {
  await delay(STUB_DELAY_MS);
  // TODO: Replace with apiClient.post('/auth/oauth/callback', { code, state }) when E02 is deployed
  void state;
  return {
    user: {
      id: `stub-oauth-${code.slice(0, 6)}`,
      email: "oauth@example.com",
      name: "OAuth User",
      companyId: "stub-company-id",
      role: "owner",
    },
    token: "stub-oauth-jwt-token",
    refreshToken: "stub-oauth-refresh-token",
  };
}
```

### Implementation Code: `app/[locale]/(auth)/login/page.tsx`

```tsx
"use client";

import { useState, useEffect } from "react";
import { useRouter } from "next/navigation";
import Link from "next/link";
import { useTranslations } from "next-intl";
import { Loader2 } from "lucide-react";
import { Button, useAuthStore, FormField, useZodForm } from "@eusolicit/ui";
import { loginSchema, type LoginInput } from "@/lib/schemas/auth";
import { loginUser } from "@/lib/api/auth";

export default function LoginPage({
  params: { locale },
}: {
  params: { locale: string };
}) {
  const t = useTranslations("auth");
  const tErrors = useTranslations("errors");
  const router = useRouter();
  const { login, isAuthenticated } = useAuthStore();
  const [mounted, setMounted] = useState(false);
  const [apiError, setApiError] = useState<string | null>(null);

  const form = useZodForm(loginSchema, { mode: "onBlur" });

  useEffect(() => {
    setMounted(true);
  }, []);

  // Redirect already-authenticated users away from login
  useEffect(() => {
    if (mounted && isAuthenticated) {
      router.replace(`/${locale}/dashboard`);
    }
  }, [mounted, isAuthenticated, locale, router]);

  // Prevent rendering auth page during hydration check
  if (!mounted) return null;

  async function onSubmit(data: LoginInput) {
    try {
      setApiError(null);
      const response = await loginUser(data);
      login(response.user, response.token, response.refreshToken);
      router.replace(`/${locale}/dashboard`);
    } catch {
      setApiError(tErrors("general"));
    }
  }

  return (
    <div>
      <h1 className="text-2xl font-semibold text-slate-900 mb-2">{t("signIn")}</h1>
      <p className="text-sm text-slate-500 mb-6">
        {t("noAccount")}{" "}
        <Link
          href={`/${locale}/register`}
          className="text-indigo-600 hover:underline font-medium"
        >
          {t("register")}
        </Link>
      </p>

      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4" noValidate>
        <FormField
          control={form.control}
          name="email"
          label={t("email")}
          type="email"
          disabled={form.formState.isSubmitting}
        />
        <FormField
          control={form.control}
          name="password"
          label={t("password")}
          type="password"
          disabled={form.formState.isSubmitting}
        />

        <div className="flex items-center justify-between">
          <FormField
            control={form.control}
            name="rememberMe"
            label={t("rememberMe")}
            type="checkbox"
            disabled={form.formState.isSubmitting}
          />
          <Link
            href={`/${locale}/forgot-password`}
            className="text-sm text-indigo-600 hover:underline"
          >
            {t("forgotPassword")}
          </Link>
        </div>

        {apiError && (
          <p className="text-sm text-red-500" role="alert">
            {apiError}
          </p>
        )}

        <Button
          type="submit"
          className="w-full"
          disabled={form.formState.isSubmitting}
        >
          {form.formState.isSubmitting && (
            <Loader2 className="animate-spin mr-2 h-4 w-4" />
          )}
          {t("signIn")}
        </Button>
      </form>

      {/* Divider */}
      <div className="relative my-6">
        <div className="absolute inset-0 flex items-center">
          <span className="w-full border-t border-slate-200" />
        </div>
        <div className="relative flex justify-center text-xs uppercase">
          <span className="bg-white px-2 text-slate-400">{t("orContinueWith")}</span>
        </div>
      </div>

      {/* Google OAuth button */}
      <Button
        variant="outline"
        className="w-full"
        type="button"
        disabled={form.formState.isSubmitting}
        onClick={() => {
          // TODO: Replace with real Google OAuth consent URL from E02 config when deployed
          window.location.href = `/api/auth/google?locale=${locale}`;
        }}
      >
        {/* Google G icon (inline SVG — no external dependency) */}
        <svg
          className="mr-2 h-4 w-4"
          viewBox="0 0 24 24"
          aria-hidden="true"
          focusable="false"
        >
          <path
            d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"
            fill="#4285F4"
          />
          <path
            d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"
            fill="#34A853"
          />
          <path
            d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"
            fill="#FBBC05"
          />
          <path
            d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"
            fill="#EA4335"
          />
        </svg>
        {t("googleSignIn")}
      </Button>
    </div>
  );
}
```

**Note on `rememberMe` with `FormField` checkbox**: If the `FormField type="checkbox"` variant does not align with the layout needed (checkbox inline with label beside "Forgot password?" link), use the `<Checkbox>` component directly from `@eusolicit/ui` with `{...form.register("rememberMe")}`. Adjust as needed to match the flex layout in the "remember me" row.

### Implementation Code: `app/[locale]/(auth)/register/page.tsx`

```tsx
"use client";

import { useState, useEffect } from "react";
import { useRouter } from "next/navigation";
import Link from "next/link";
import { useTranslations } from "next-intl";
import { Loader2 } from "lucide-react";
import { Button, useAuthStore, FormField, useZodForm } from "@eusolicit/ui";
import { registerSchema, type RegisterInput } from "@/lib/schemas/auth";
import { registerUser } from "@/lib/api/auth";

export default function RegisterPage({
  params: { locale },
}: {
  params: { locale: string };
}) {
  const t = useTranslations("auth");
  const tErrors = useTranslations("errors");
  const router = useRouter();
  const { login, isAuthenticated } = useAuthStore();
  const [mounted, setMounted] = useState(false);
  const [apiError, setApiError] = useState<string | null>(null);

  const form = useZodForm(registerSchema, { mode: "onBlur" });

  useEffect(() => {
    setMounted(true);
  }, []);

  useEffect(() => {
    if (mounted && isAuthenticated) {
      router.replace(`/${locale}/dashboard`);
    }
  }, [mounted, isAuthenticated, locale, router]);

  if (!mounted) return null;

  async function onSubmit(data: RegisterInput) {
    try {
      setApiError(null);
      const response = await registerUser({
        companyName: data.companyName,
        eik: data.eik,
        firstName: data.firstName,
        lastName: data.lastName,
        email: data.email,
        password: data.password,
      });
      login(response.user, response.token, response.refreshToken);
      router.replace(`/${locale}/dashboard`);
    } catch {
      setApiError(tErrors("general"));
    }
  }

  return (
    <div>
      <h1 className="text-2xl font-semibold text-slate-900 mb-2">{t("register")}</h1>
      <p className="text-sm text-slate-500 mb-6">
        {t("haveAccount")}{" "}
        <Link
          href={`/${locale}/login`}
          className="text-indigo-600 hover:underline font-medium"
        >
          {t("signIn")}
        </Link>
      </p>

      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4" noValidate>
        <div className="grid grid-cols-2 gap-4">
          <FormField
            control={form.control}
            name="firstName"
            label={t("firstName")}
            type="text"
            disabled={form.formState.isSubmitting}
          />
          <FormField
            control={form.control}
            name="lastName"
            label={t("lastName")}
            type="text"
            disabled={form.formState.isSubmitting}
          />
        </div>

        <FormField
          control={form.control}
          name="email"
          label={t("email")}
          type="email"
          disabled={form.formState.isSubmitting}
        />
        <FormField
          control={form.control}
          name="companyName"
          label={t("companyName")}
          type="text"
          disabled={form.formState.isSubmitting}
        />
        <FormField
          control={form.control}
          name="eik"
          label={t("eik")}
          type="text"
          disabled={form.formState.isSubmitting}
          description="9 or 13 digit company registration number"
        />
        <FormField
          control={form.control}
          name="password"
          label={t("password")}
          type="password"
          disabled={form.formState.isSubmitting}
        />
        <FormField
          control={form.control}
          name="confirmPassword"
          label={t("confirmPassword")}
          type="password"
          disabled={form.formState.isSubmitting}
        />
        <FormField
          control={form.control}
          name="agreeToTerms"
          label={t("terms")}
          type="checkbox"
          disabled={form.formState.isSubmitting}
        />

        {apiError && (
          <p className="text-sm text-red-500" role="alert">
            {apiError}
          </p>
        )}

        <Button
          type="submit"
          className="w-full"
          disabled={form.formState.isSubmitting}
        >
          {form.formState.isSubmitting && (
            <Loader2 className="animate-spin mr-2 h-4 w-4" />
          )}
          {t("register")}
        </Button>
      </form>
    </div>
  );
}
```

### Implementation Code: `app/[locale]/(auth)/forgot-password/page.tsx`

```tsx
"use client";

import { useState } from "react";
import Link from "next/link";
import { useTranslations } from "next-intl";
import { Loader2, Mail } from "lucide-react";
import { Button, FormField, useZodForm } from "@eusolicit/ui";
import { forgotPasswordSchema, type ForgotPasswordInput } from "@/lib/schemas/auth";
import { requestPasswordReset } from "@/lib/api/auth";

export default function ForgotPasswordPage({
  params: { locale },
}: {
  params: { locale: string };
}) {
  const t = useTranslations("auth");
  const tErrors = useTranslations("errors");
  const [submitted, setSubmitted] = useState(false);
  const [apiError, setApiError] = useState<string | null>(null);

  const form = useZodForm(forgotPasswordSchema, { mode: "onBlur" });

  async function onSubmit(data: ForgotPasswordInput) {
    try {
      setApiError(null);
      await requestPasswordReset(data.email);
      setSubmitted(true);
    } catch {
      setApiError(tErrors("general"));
    }
  }

  // Success state: form hidden, confirmation message shown
  if (submitted) {
    return (
      <div className="text-center space-y-4 py-4">
        <div className="flex justify-center">
          <Mail className="h-12 w-12 text-indigo-600" aria-hidden="true" />
        </div>
        <h1 className="text-2xl font-semibold text-slate-900">{t("checkEmail")}</h1>
        <p className="text-sm text-slate-500">{t("checkEmailDescription")}</p>
        <Link
          href={`/${locale}/login`}
          className="block text-sm text-indigo-600 hover:underline mt-2"
        >
          {t("backToSignIn")}
        </Link>
      </div>
    );
  }

  return (
    <div>
      <h1 className="text-2xl font-semibold text-slate-900 mb-2">
        {t("forgotPassword")}
      </h1>
      <p className="text-sm text-slate-500 mb-6">
        {t("forgotPasswordDescription")}
      </p>

      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4" noValidate>
        <FormField
          control={form.control}
          name="email"
          label={t("email")}
          type="email"
          disabled={form.formState.isSubmitting}
        />

        {apiError && (
          <p className="text-sm text-red-500" role="alert">
            {apiError}
          </p>
        )}

        <Button
          type="submit"
          className="w-full"
          disabled={form.formState.isSubmitting}
        >
          {form.formState.isSubmitting && (
            <Loader2 className="animate-spin mr-2 h-4 w-4" />
          )}
          {t("sendResetLink")}
        </Button>
      </form>

      <div className="mt-4 text-center">
        <Link
          href={`/${locale}/login`}
          className="text-sm text-indigo-600 hover:underline"
        >
          {t("backToSignIn")}
        </Link>
      </div>
    </div>
  );
}
```

**Note**: `t("forgotPasswordDescription")` uses a new key to be added in Task 8. Alternatively, the description can be added as a static string or use a key from the `errors` namespace — add `forgotPasswordDescription` to the 7 new keys in Task 8.1/8.2.

### Implementation Code: `app/[locale]/(auth)/callback/page.tsx`

```tsx
// apps/client/app/[locale]/(auth)/callback/page.tsx
// Inner component must use "use client" for useSearchParams().
// The outer page export is a compatible server-renderable wrapper with Suspense.

"use client";

import { Suspense, useEffect, useState } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import { useTranslations } from "next-intl";
import { Loader2, AlertCircle } from "lucide-react";
import { useAuthStore } from "@eusolicit/ui";
import { exchangeOAuthCode } from "@/lib/api/auth";

function CallbackPageContent({ locale }: { locale: string }) {
  const t = useTranslations("auth");
  const tErrors = useTranslations("errors");
  const router = useRouter();
  const searchParams = useSearchParams();
  const { login } = useAuthStore();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const code = searchParams.get("code");
    const state = searchParams.get("state") ?? "";

    if (!code) {
      setError("Missing authorization code.");
      return;
    }

    exchangeOAuthCode(code, state)
      .then((response) => {
        login(response.user, response.token, response.refreshToken);
        router.replace(`/${locale}/dashboard`);
      })
      .catch(() => {
        setError(tErrors("general"));
      });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  if (error) {
    return (
      <div
        className="text-center space-y-4 py-4"
        data-testid="auth-callback-error"
      >
        <AlertCircle
          className="h-12 w-12 text-red-500 mx-auto"
          aria-hidden="true"
        />
        <p className="text-slate-700">{error}</p>
        <a
          href={`/${locale}/login`}
          className="text-sm text-indigo-600 hover:underline"
        >
          {t("backToSignIn")}
        </a>
      </div>
    );
  }

  return (
    <div
      className="text-center space-y-4 py-4"
      data-testid="auth-callback-loading"
    >
      <Loader2
        className="h-12 w-12 text-indigo-600 animate-spin mx-auto"
        aria-hidden="true"
      />
      <p className="text-slate-500">{t("processingAuth")}</p>
    </div>
  );
}

export default function CallbackPage({
  params: { locale },
}: {
  params: { locale: string };
}) {
  return (
    <Suspense
      fallback={
        <div
          className="text-center py-4"
          data-testid="auth-callback-loading"
        >
          <Loader2 className="h-12 w-12 text-indigo-600 animate-spin mx-auto" />
        </div>
      }
    >
      <CallbackPageContent locale={locale} />
    </Suspense>
  );
}
```

### New i18n Keys to Add (Task 8)

Add these keys to the `"auth"` namespace in **both** message files. Insert after the existing `"checkEmailDescription"` key:

**`apps/client/messages/en.json` — additions to `"auth"` namespace:**
```json
"googleSignIn": "Continue with Google",
"noAccount": "Don't have an account?",
"haveAccount": "Already have an account?",
"orContinueWith": "Or continue with",
"processingAuth": "Processing authentication...",
"sendResetLink": "Send reset link",
"backToSignIn": "Back to sign in",
"forgotPasswordDescription": "Enter your email and we'll send you a reset link."
```

**`apps/client/messages/bg.json` — additions to `"auth"` namespace:**
```json
"googleSignIn": "Продължи с Google",
"noAccount": "Нямате акаунт?",
"haveAccount": "Вече имате акаунт?",
"orContinueWith": "Или продължете с",
"processingAuth": "Обработка на удостоверяването...",
"sendResetLink": "Изпрати линк за нулиране",
"backToSignIn": "Обратно към вход",
"forgotPasswordDescription": "Въведете имейла си и ще ви изпратим линк за нулиране на паролата."
```

**Note**: 8 new keys (not 7 — `forgotPasswordDescription` was added for the forgot-password page subtitle). Update AC9 accordingly — the test suite counts keys, not the story narrative.

After adding keys to both files, run:
```bash
pnpm check:i18n --filter client
```
This must exit 0, confirming BG and EN key sets are identical.

### FormField Component API Reference (from S03.06)

```tsx
// Correct usage — control + name are required:
<FormField
  control={form.control}     // required — RHF control object
  name="fieldName"           // required — must match Zod schema key
  label="Label Text"         // optional — rendered as <FormLabel>
  type="text|email|password|textarea|select|checkbox|radio|date|file"  // default: "text"
  disabled={boolean}         // passed to underlying input
  description="Helper text"  // optional — renders below field
/>
```

For `agreeToTerms` (`type="checkbox"`): the `label` prop text becomes the checkbox label. The checkbox variant renders a `<Checkbox>` from shadcn/ui wrapped by `<FormField>`'s `<FormControl>`. If the checkbox alignment needs adjustment for the terms field, consider wrapping it in a custom `<FormItem>` + `<Checkbox>` using `form.register("agreeToTerms")` directly.

### Test Design Reference (from `test-design-epic-03.md`)

Tests that this story enables (to be implemented by the TEA agent in the ATDD phase):

| Test ID | Description | Enabled By |
|---------|-------------|------------|
| **E03-P1-003** | Login form: invalid email + short password show inline Zod errors on blur/submit; valid creds show loading spinner + inputs disabled | Task 4 |
| **E03-P1-004** | Register form: invalid EIK format, password confirm mismatch, unchecked terms all show inline errors | Task 5 |
| **E03-P1-005** | Forgot password: valid email submission shows "Check your email" success state; form hidden after submit | Task 6 |
| **E03-P2-012** | OAuth callback page extracts `code` and `state` query params; shows `data-testid="auth-callback-loading"` | Task 7 |
| **E03-P0-004** (partial) | Authenticated user visiting `/login` or `/register` → redirected to `/dashboard` | AC8 / Tasks 4, 5 |
| **E03-P2-019** (prerequisite) | No redirect loop when auth state is inconsistent — `!mounted` guard prevents premature redirect | Auth pattern in all pages |

**Risk mitigations delivered in this story:**
- **E03-R-002** (route guard flash of protected content): Auth pages use `!mounted → return null` guard — no auth page content rendered before Zustand hydration. Full `<AuthGuard>` for protected routes is delivered in S03.12.
- **E03-R-001** (locale routing): All in-app `<Link href>` and `router.replace()` use `/${locale}/` prefix — no bare `/login` paths that would bypass `[locale]` segment routing.

### Stub Replacement Guide (Post-E02 Integration)

When E02 auth API is deployed, replace stubs in `apps/client/lib/api/auth.ts`:

```typescript
// Replace each stub:
import { apiClient } from "@eusolicit/ui";

export async function loginUser(data: LoginRequest): Promise<AuthResponse> {
  return apiClient.post<AuthResponse>("/auth/login", data);
}

export async function registerUser(data: RegisterRequest): Promise<AuthResponse> {
  return apiClient.post<AuthResponse>("/auth/register", data);
}

export async function requestPasswordReset(email: string): Promise<void> {
  await apiClient.post("/auth/forgot-password", { email });
}

export async function exchangeOAuthCode(
  code: string,
  state: string
): Promise<AuthResponse> {
  return apiClient.post<AuthResponse>("/auth/oauth/callback", { code, state });
}
```

`apiClient` handles JWT attachment and 401 refresh automatically (established in S03.05).

## Senior Developer Review

### Re-review — 2026-04-08

**Date:** 2026-04-08
**Verdict:** Approved
**Review Layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor (full mode — spec-backed)

**Summary:** All 5 prior patch items verified as correctly resolved. All 10 acceptance criteria pass. 0 new patch items. 3 new deferred enhancements added. 17 dismissed (noise, out-of-scope, or false positives).

**AC Verification:** AC1 ✅ | AC2 ✅ | AC3 ✅ | AC4 ✅ | AC5 ✅ | AC6 ✅ | AC7 ✅ | AC8 ✅ | AC9 ✅ | AC10 ✅

#### New Deferred Items (non-blocking)

- [x] [Review][Defer] **OAuth provider error params not surfaced** [`callback/page.tsx`] — deferred to E02 integration. When an OAuth provider returns `?error=access_denied`, the callback shows "Missing authorization code" instead of the provider's reason. Enhancement for real OAuth integration.
- [x] [Review][Defer] **Callback page has no authenticated-user redirect guard** [`callback/page.tsx`] — deferred to E02 integration. Unlike login/register/forgot-password, the callback page does not check `isAuthenticated` before exchanging the code. In theory an already-authenticated user could silently switch identity. OAuth `state` CSRF validation (E02) will address this.
- [x] [Review][Defer] **EIK field does not trim whitespace before regex validation** [`lib/schemas/auth.ts`] — deferred, UX enhancement. A copy-pasted EIK with leading/trailing spaces fails validation. Add `.trim()` before the regex in a future schema refinement pass.

### Initial Review — 2026-04-08

**Date:** 2026-04-08
**Verdict:** Changes Requested
**Review Layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor (full mode — spec-backed)

**Summary:** 5 patch, 2 defer, 1 dismissed. Two direct AC violations found (AC8 + AC9). All patch items have unambiguous fixes.

### Review Findings

- [x] [Review][Patch] **AC9 VIOLATION — Callback hardcoded English error string** [`app/[locale]/(auth)/callback/page.tsx`] — `setError("Missing authorization code.")` is visible user-facing text using a hardcoded English string. Must use `t("missingAuthCode")` or equivalent i18n key. Add `missingAuthCode` key to both `en.json` and `bg.json` auth namespace.
- [x] [Review][Patch] **AC9 VIOLATION — Register EIK description hardcoded English** [`app/[locale]/(auth)/register/page.tsx`] — `description="9 or 13 digit company registration number"` is visible helper text not using i18n. Must use `t("eikDescription")` or equivalent. Add `eikDescription` key to both `en.json` and `bg.json` auth namespace.
- [x] [Review][Patch] **AC8 VIOLATION — Forgot-password page missing `mounted` guard and authenticated redirect** [`app/[locale]/(auth)/forgot-password/page.tsx`] — AC8 requires all auth pages to render `null` before Zustand hydration (`!mounted`) and redirect to `/${locale}/dashboard` if `isAuthenticated` is true. Login and Register pages implement this correctly; forgot-password omits it entirely. Add `useState(false)` mounted guard, `useAuthStore` isAuthenticated check, and redirect `useEffect` matching the pattern in login/register pages.
- [x] [Review][Patch] **Stub role "owner" not in RBAC model** [`lib/api/auth.ts`] — All four stub functions return `role: "owner"` but the project's RBAC system defines roles as `admin | bid_manager | contributor | reviewer | read_only`. Use `role: "admin"` to match the backend model and prevent masking integration bugs when stubs are replaced.
- [x] [Review][Patch] **Callback error uses `<a>` instead of `<Link>`** [`app/[locale]/(auth)/callback/page.tsx`] — Error state navigation to login uses `<a href={...}>` causing full-page reload. All other navigation in auth pages uses Next.js `<Link>`. Replace with `<Link href={...}>` and add `import Link from "next/link"`.
- [x] [Review][Defer] **No AbortController in callback useEffect** [`app/[locale]/(auth)/callback/page.tsx`] — deferred, pre-existing pattern. The mount-only `useEffect` calling `exchangeOAuthCode` has no cleanup. If the component unmounts during the API call, `.then()` fires on stale state. Address when stubs are replaced with real API calls (E02 integration).
- [x] [Review][Defer] **OAuth state parameter not validated for CSRF** [`app/[locale]/(auth)/callback/page.tsx`] — deferred, stub behavior. Missing `state` defaults to empty string instead of treating it as a CSRF failure. Real OAuth CSRF protection will be implemented at E02 integration.

## Dev Agent Record

### Implementation Plan

All 7 auth page files were already implemented in a prior session. This session (2026-04-08) focused on resolving 5 code-review patch items identified in the Senior Developer Review:

1. **AC9 Patch — Callback i18n**: Replaced hardcoded `"Missing authorization code."` with `t("missingAuthCode")` in callback/page.tsx. Added `missingAuthCode` key to both en.json and bg.json.
2. **AC9 Patch — Register EIK i18n**: Replaced hardcoded `description="9 or 13 digit company registration number"` with `description={t("eikDescription")}` in register/page.tsx. Added `eikDescription` key to both locale files.
3. **AC8 Patch — Forgot-password mounted guard**: Added `useState(false)` mounted flag, `useAuthStore` isAuthenticated check, and redirect `useEffect` to forgot-password/page.tsx — matching the pattern already used in login and register pages.
4. **RBAC Patch — Stub roles**: Changed all three `role: "owner"` values in lib/api/auth.ts to `role: "admin"` to match the project's RBAC model (`admin | bid_manager | contributor | reviewer | read_only`).
5. **Navigation Patch — Callback `<a>` → `<Link>`**: Added `import Link from "next/link"` and replaced `<a href={...}>` with `<Link href={...}>` in the callback error state.

### Completion Notes

✅ Resolved review finding [Patch]: AC9 VIOLATION — Callback hardcoded English error string
✅ Resolved review finding [Patch]: AC9 VIOLATION — Register EIK description hardcoded English
✅ Resolved review finding [Patch]: AC8 VIOLATION — Forgot-password page missing mounted guard
✅ Resolved review finding [Patch]: Stub role "owner" not in RBAC model
✅ Resolved review finding [Patch]: Callback error uses `<a>` instead of `<Link>`

**All validations passed:**
- `node scripts/check-i18n-keys.mjs` → ✅ 77 keys in both bg.json and en.json
- `pnpm type-check` → ✅ 0 TypeScript errors (4 packages)
- `pnpm build` → ✅ Both client and admin apps exit 0
- `pnpm --filter client test` → ✅ 81 tests pass (5 test files, including 41 ATDD auth tests)
- `pnpm --filter "@eusolicit/ui" test` → ✅ 60 tests pass (8 test files)

## File List

**Modified (review patches):**
- `apps/client/app/[locale]/(auth)/callback/page.tsx` — Added Link import, i18n for missing auth code error, `<a>` → `<Link>`
- `apps/client/app/[locale]/(auth)/register/page.tsx` — EIK description uses `t("eikDescription")`
- `apps/client/app/[locale]/(auth)/forgot-password/page.tsx` — Added mounted guard + authenticated redirect
- `apps/client/lib/api/auth.ts` — Changed stub `role: "owner"` → `role: "admin"` (3 occurrences)
- `apps/client/messages/en.json` — Added `missingAuthCode`, `eikDescription` keys to auth namespace
- `apps/client/messages/bg.json` — Added `missingAuthCode`, `eikDescription` keys to auth namespace

**Pre-existing implementation files (Tasks 1–7, prior session):**
- `apps/client/app/[locale]/(auth)/layout.tsx`
- `apps/client/app/[locale]/(auth)/login/page.tsx`
- `apps/client/lib/schemas/auth.ts`

## Change Log

- 2026-04-08: Addressed code review findings — 5 patch items resolved (AC9 i18n violations ×2, AC8 mounted guard, RBAC role, Link navigation fix). i18n parity: 77 keys. Build, type-check, and all 81+ tests pass.
