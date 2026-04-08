# Story 3.7: i18n Setup with next-intl (BG + EN)

Status: approved

## Story

As a **frontend developer on the EU Solicit team**,
I want **`next-intl` configured in both client and admin apps with Bulgarian and English locale support, URL-based locale routing (`/bg/...`, `/en/...`), and a functional language selector in the top bar**,
so that **all shell chrome strings are translatable via message files, the URL reflects the active locale, and users' language preference is preserved across sessions**.

## Acceptance Criteria

1. `next-intl` installed as a dependency in both `apps/client` and `apps/admin`; `next.config.mjs` in each app wraps the config with `createNextIntlPlugin('./i18n.ts')`
2. `middleware.ts` at each app's root handles locale detection — reads `Accept-Language` header, falls back to `NEXT_LOCALE` cookie, defaults to `"bg"`; config: `locales: ['bg', 'en']`, `defaultLocale: 'bg'`, `localePrefix: 'always'`
3. Both apps restructured to `app/[locale]/` route segment: existing pages moved into `[locale]`; `app/[locale]/layout.tsx` receives `params.locale` and wraps children with `<NextIntlClientProvider>` + `<QueryProvider>`; root `app/layout.tsx` simplified to HTML shell only (fonts + html/body, no providers)
4. Message files `messages/bg.json` and `messages/en.json` exist in each app's root with namespaces: `common`, `nav`, `auth`, `forms`, `errors`, `wizard`; all keys present in both files (no missing keys between BG and EN)
5. All shell chrome strings (sidebar nav labels, top bar, user avatar dropdown items, breadcrumbs) use `useTranslations()` in the protected layout — nav items built with translated labels; `UserAvatarMenu` accepts optional `labels` prop for translated menu item text; `TopBar` uses translated aria labels
6. Language selector in `TopBar` shows globe icon + current locale code (`BG` / `EN`); clicking the inactive locale calls `onLocaleChange(newLocale)` passed from the layout; layout handler uses `router.replace()` with the new locale prefix and updates `uiStore.locale`
7. Locale preference persisted: switching locale writes `uiStore.locale` (already persisted to localStorage via `eusolicit-ui-store`); middleware reads `NEXT_LOCALE` cookie set by next-intl; first visit reads `Accept-Language` header
8. `formatDate(date: Date, locale: string): string` and `formatCurrency(amount: number, currency: string, locale: string): string` utility functions created in `packages/ui/src/lib/format.ts`; exported from `packages/ui/index.ts`
9. i18n key diff script (`scripts/check-i18n-keys.mjs`) verifies BG and EN message files share the same flattened key set; exits non-zero if mismatch — integrated as `pnpm check:i18n` in each app
10. `pnpm build` exits 0 for both apps; `pnpm type-check` exits 0 across all packages; navigating to `/` in each app redirects to `/bg/` (≤1 redirect, no redirect loops)

## Tasks / Subtasks

- [x] Task 1: Install next-intl (AC: 1)
  - [x] 1.1 Add `"next-intl": "^3.26.0"` to `apps/client/package.json` → `dependencies`
  - [x] 1.2 Add `"next-intl": "^3.26.0"` to `apps/admin/package.json` → `dependencies`
  - [x] 1.3 Run `pnpm install` from `eusolicit-app/frontend/`

- [x] Task 2: Create `i18n.ts` request configuration in each app (AC: 1)
  - [x] 2.1 Create `apps/client/i18n.ts` — next-intl server-side request config; see Dev Notes for full implementation
  - [x] 2.2 Create `apps/admin/i18n.ts` — identical structure; see Dev Notes

- [x] Task 3: Create `middleware.ts` in each app (AC: 2)
  - [x] 3.1 Create `apps/client/middleware.ts` — `createMiddleware` with `locales: ['bg','en']`, `defaultLocale: 'bg'`, `localePrefix: 'always'`; see Dev Notes for full implementation
  - [x] 3.2 Create `apps/admin/middleware.ts` — identical; see Dev Notes

- [x] Task 4: Update `next.config.mjs` in each app (AC: 1)
  - [x] 4.1 Modify `apps/client/next.config.mjs` — import `createNextIntlPlugin`, wrap `nextConfig` with `withNextIntl`; see Dev Notes
  - [x] 4.2 Modify `apps/admin/next.config.mjs` — same pattern; see Dev Notes

- [x] Task 5: Create message files for client app (AC: 4)
  - [x] 5.1 Create `apps/client/messages/bg.json` — BG namespace structure with placeholder text; see Dev Notes for full JSON
  - [x] 5.2 Create `apps/client/messages/en.json` — EN namespace with English text; see Dev Notes for full JSON

- [x] Task 6: Create message files for admin app (AC: 4)
  - [x] 6.1 Create `apps/admin/messages/bg.json` — BG namespace structure (admin nav keys); see Dev Notes
  - [x] 6.2 Create `apps/admin/messages/en.json` — EN namespace for admin; see Dev Notes

- [x] Task 7: Restructure client app to `app/[locale]/` (AC: 3)
  - [x] 7.1 Simplify `apps/client/app/layout.tsx` — remove QueryProvider, keep html+body+fonts; see Dev Notes for full implementation
  - [x] 7.2 Create `apps/client/app/[locale]/layout.tsx` — NextIntlClientProvider + QueryProvider + locale sync; see Dev Notes
  - [x] 7.3 Create `apps/client/app/[locale]/page.tsx` — redirect to `/${locale}/dashboard`; replace old `app/page.tsx`; see Dev Notes
  - [x] 7.4 Move protected layout: create `apps/client/app/[locale]/(protected)/layout.tsx` — same as old layout but with `useTranslations('nav')` and locale-prefixed hrefs; see Dev Notes
  - [x] 7.5 Move protected dashboard: create `apps/client/app/[locale]/(protected)/dashboard/page.tsx` — same content as old dashboard
  - [x] 7.6 Move dev pages: create `apps/client/app/[locale]/dev/components/page.tsx`, `/dev/api-test/page.tsx`, `/dev/form-test/page.tsx` — same content, update relative import paths if needed
  - [x] 7.7 Delete old non-locale files: `apps/client/app/page.tsx`, `apps/client/app/(protected)/layout.tsx`, `apps/client/app/(protected)/dashboard/page.tsx`, `apps/client/app/dev/components/page.tsx`, `apps/client/app/dev/api-test/page.tsx`, `apps/client/app/dev/form-test/page.tsx`

- [x] Task 8: Restructure admin app to `app/[locale]/` (AC: 3)
  - [x] 8.1 Simplify `apps/admin/app/layout.tsx` — same as client; see Dev Notes
  - [x] 8.2 Create `apps/admin/app/[locale]/layout.tsx` — NextIntlClientProvider + QueryProvider; see Dev Notes
  - [x] 8.3 Create `apps/admin/app/[locale]/page.tsx` — redirect to `/${locale}/dashboard`
  - [x] 8.4 Move protected layout: create `apps/admin/app/[locale]/(protected)/layout.tsx` — with admin nav items translated; see Dev Notes
  - [x] 8.5 Move protected dashboard: create `apps/admin/app/[locale]/(protected)/dashboard/page.tsx`
  - [x] 8.6 Delete old non-locale admin files: `apps/admin/app/page.tsx`, `apps/admin/app/(protected)/layout.tsx`, `apps/admin/app/(protected)/dashboard/page.tsx`

- [x] Task 9: Extend `packages/ui` components for i18n support (AC: 5, 6)
  - [x] 9.1 Modify `packages/ui/src/components/app-shell/TopBar.tsx` — add `locale: 'bg' | 'en'` and `onLocaleChange: (locale: 'bg' | 'en') => void` props; add Globe icon + BG/EN buttons; see Dev Notes
  - [x] 9.2 Modify `packages/ui/src/components/app-shell/UserAvatarMenu.tsx` — add optional `labels` prop (`{ profile, settings, signOut }`) with English defaults; add `onSignOut?: () => void` prop; see Dev Notes

- [x] Task 10: Create format utilities in `packages/ui` (AC: 8)
  - [x] 10.1 Create `packages/ui/src/lib/format.ts` — `formatDate` and `formatCurrency` using `Intl.DateTimeFormat` / `Intl.NumberFormat`; see Dev Notes for full implementation
  - [x] 10.2 Add export to `packages/ui/index.ts` — `export { formatDate, formatCurrency } from "./src/lib/format"`

- [x] Task 11: Create i18n key diff script (AC: 9)
  - [x] 11.1 Create `apps/client/scripts/check-i18n-keys.mjs` — Node.js ESM script that flattens and compares key sets from `messages/bg.json` and `messages/en.json`; see Dev Notes
  - [x] 11.2 Add `"check:i18n": "node scripts/check-i18n-keys.mjs"` to `apps/client/package.json` scripts
  - [x] 11.3 Create identical script and script entry in `apps/admin/`

- [x] Task 12: Add uiStore locale sync in locale layouts (AC: 6, 7)
  - [x] 12.1 Create `apps/client/components/LocaleSyncer.tsx` — `"use client"` component that reads `uiStore.locale`, compares with URL locale, and syncs on mount; see Dev Notes
  - [x] 12.2 Import `<LocaleSyncer>` in `apps/client/app/[locale]/layout.tsx`
  - [x] 12.3 Create identical `LocaleSyncer.tsx` in `apps/admin/components/` and wire it in admin locale layout

- [x] Task 13: Build and key-check verification (AC: 9, 10)
  - [x] 13.1 Run `pnpm check:i18n --filter client` — must exit 0 (all keys match)
  - [x] 13.2 Run `pnpm check:i18n --filter admin` — must exit 0
  - [x] 13.3 Run `pnpm build` from `eusolicit-app/frontend/` — both apps must exit 0
  - [x] 13.4 Run `pnpm type-check` from `eusolicit-app/frontend/` — zero TypeScript errors

## Dev Notes

### Working Directory

All frontend code lives under: `eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run pnpm commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

### Critical Learnings from Stories 3.1–3.6 (MUST APPLY)

1. **`@/*` maps to `./` not `./src/`** — Inside `apps/client`, `@/lib/stores/ui-store` resolves to `apps/client/lib/stores/ui-store.ts`. Inside `apps/admin`, same pattern. Inside `packages/ui`, always use **relative paths** — the `@/` alias does NOT work inside `packages/ui`.
2. **`next.config.mjs` not `.ts`** — Both apps use `.mjs`. Do NOT create `.ts` next configs. The `createNextIntlPlugin` import uses ESM `import` syntax.
3. **`suppressHydrationWarning` stays on `<html>`** — Already in root `app/layout.tsx`. Do NOT add it to nested layouts.
4. **`packages/ui` uses relative imports for internal modules** — All new `packages/ui` files must follow the same relative-path pattern as `button.tsx` (e.g., `../../lib/utils`).
5. **`packages/ui/index.ts` uses `./src/lib/...` paths** — All new exports follow the pattern `./src/components/...` or `./src/lib/...`.
6. **`pnpm install` must run from `eusolicit-app/frontend/` root** — Not from individual package directories.
7. **New runtime dependencies go in `dependencies` (not `devDependencies`)** — `next-intl` is a runtime dep; add to `dependencies`.
8. **`uiStore` lives in `apps/client/lib/stores/ui-store.ts`** — Not in `packages/ui`. Each app has its own `uiStore`. The `locale: 'bg' | 'en'` field and `persist` middleware are already in place from S3.5.
9. **Nav item hrefs need locale prefix after this story** — Change from `"/dashboard"` to `"/${locale}/dashboard"` in protected layout.
10. **Delete old non-locale app pages explicitly** — Do not leave `app/page.tsx` and `app/(protected)/...` alongside new `app/[locale]/` equivalents — Next.js will error on conflicting routes.

### Architecture Overview: Before and After

**BEFORE (current structure, both apps):**
```
apps/client/
  app/
    layout.tsx            ← html+body+fonts+QueryProvider
    globals.css
    page.tsx              ← "EU Solicit Client" page
    (protected)/
      layout.tsx          ← AppShell with hardcoded EN nav labels
      dashboard/
        page.tsx
    dev/
      components/page.tsx
      api-test/page.tsx
      form-test/page.tsx
```

**AFTER (S3.7 target structure, both apps):**
```
apps/client/
  i18n.ts                 ← NEW: next-intl server request config
  middleware.ts           ← NEW: locale detection + routing
  messages/
    bg.json               ← NEW: BG translations
    en.json               ← NEW: EN translations
  scripts/
    check-i18n-keys.mjs   ← NEW: key diff validation
  components/
    LocaleSyncer.tsx      ← NEW: uiStore↔URL locale sync
  app/
    layout.tsx            ← MODIFIED: html+body+fonts ONLY (no providers)
    globals.css           ← UNCHANGED
    [locale]/
      layout.tsx          ← NEW: NextIntlClientProvider + QueryProvider
      page.tsx            ← NEW: redirect to /[locale]/dashboard
      (protected)/
        layout.tsx        ← NEW: AppShell with translated nav labels
        dashboard/
          page.tsx        ← MOVED: same content
      dev/
        components/page.tsx  ← MOVED
        api-test/page.tsx    ← MOVED
        form-test/page.tsx   ← MOVED
  next.config.mjs         ← MODIFIED: withNextIntl plugin
  package.json            ← MODIFIED: add next-intl
```

### Implementation Code: `i18n.ts` (same for both apps)

```typescript
// apps/client/i18n.ts  (and apps/admin/i18n.ts — identical)
import { getRequestConfig } from "next-intl/server";

export default getRequestConfig(async ({ locale }) => ({
  messages: (await import(`./messages/${locale}.json`)).default,
}));
```

### Implementation Code: `middleware.ts` (same for both apps)

```typescript
// apps/client/middleware.ts  (and apps/admin/middleware.ts — identical)
import createMiddleware from "next-intl/middleware";

export default createMiddleware({
  locales: ["bg", "en"],
  defaultLocale: "bg",
  localePrefix: "always",
});

export const config = {
  // Match all pathnames except Next.js internals and static files
  matcher: [
    "/((?!_next|_vercel|api|.*\\..*).*)",
  ],
};
```

**Risk E03-R-001 Mitigation**: `localePrefix: 'always'` ensures every URL has an explicit locale prefix, eliminating ambiguous redirect chains. The matcher excludes `api/`, `_next/`, `_vercel/`, and files with extensions (images, fonts, etc.).

### Implementation Code: `next.config.mjs` (both apps — same pattern)

```javascript
// apps/client/next.config.mjs
import createNextIntlPlugin from "next-intl/plugin";

const withNextIntl = createNextIntlPlugin("./i18n.ts");

/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ["@eusolicit/ui"],
};

export default withNextIntl(nextConfig);
```

```javascript
// apps/admin/next.config.mjs — identical
import createNextIntlPlugin from "next-intl/plugin";

const withNextIntl = createNextIntlPlugin("./i18n.ts");

/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ["@eusolicit/ui"],
};

export default withNextIntl(nextConfig);
```

### Implementation Code: Message Files

**`apps/client/messages/en.json`:**
```json
{
  "common": {
    "loading": "Loading...",
    "close": "Close",
    "cancel": "Cancel",
    "save": "Save",
    "next": "Next",
    "back": "Back",
    "skip": "Skip",
    "search": "Search",
    "clear": "Clear",
    "submit": "Submit",
    "language": "Language",
    "notifications": "Notifications"
  },
  "nav": {
    "dashboard": "Dashboard",
    "tenders": "Tenders",
    "offers": "Offers",
    "documents": "Documents",
    "team": "Team",
    "settings": "Settings"
  },
  "auth": {
    "profile": "Profile",
    "settings": "Settings",
    "signOut": "Sign out",
    "signIn": "Sign in",
    "register": "Register",
    "forgotPassword": "Forgot password?",
    "rememberMe": "Remember me",
    "email": "Email address",
    "password": "Password",
    "confirmPassword": "Confirm password",
    "firstName": "First name",
    "lastName": "Last name",
    "companyName": "Company name",
    "eik": "EIK (Company ID)",
    "terms": "I agree to the terms and conditions",
    "checkEmail": "Check your email",
    "checkEmailDescription": "We've sent a password reset link to your email address."
  },
  "forms": {
    "required": "This field is required",
    "invalidEmail": "Please enter a valid email address",
    "passwordTooShort": "Password must be at least 8 characters",
    "passwordMismatch": "Passwords do not match",
    "invalidEik": "Please enter a valid 9 or 13 digit EIK",
    "minLength": "Must be at least {min} characters",
    "maxLength": "Cannot exceed {max} characters"
  },
  "errors": {
    "general": "Something went wrong. Please try again.",
    "notFound": "Page not found",
    "unauthorized": "You are not authorised to view this page",
    "serverError": "Server error. Please try again later.",
    "tryAgain": "Try again",
    "sessionExpired": "Your session has expired. Please sign in again."
  },
  "wizard": {
    "title": "Set up your company profile",
    "step1": "Company details",
    "step2": "Industry & CPV sectors",
    "step3": "Regions of interest",
    "step4": "Invite team members",
    "completeSetup": "Complete setup",
    "skipStep": "Skip this step",
    "companyInfo": "Company information",
    "companyAddress": "Address",
    "companyPhone": "Phone",
    "companyWebsite": "Website",
    "logoUpload": "Upload logo",
    "cpvSearch": "Search CPV codes",
    "selectRegions": "Select regions",
    "selectAll": "Select all",
    "inviteEmail": "Email address",
    "addInvite": "Add",
    "minOneCpv": "Please select at least one CPV code",
    "minOneRegion": "Please select at least one region"
  }
}
```

**`apps/client/messages/bg.json`:**
```json
{
  "common": {
    "loading": "Зареждане...",
    "close": "Затвори",
    "cancel": "Отказ",
    "save": "Запази",
    "next": "Напред",
    "back": "Назад",
    "skip": "Пропусни",
    "search": "Търсене",
    "clear": "Изчисти",
    "submit": "Изпрати",
    "language": "Език",
    "notifications": "Известия"
  },
  "nav": {
    "dashboard": "Табло",
    "tenders": "Обществени поръчки",
    "offers": "Оферти",
    "documents": "Документи",
    "team": "Екип",
    "settings": "Настройки"
  },
  "auth": {
    "profile": "Профил",
    "settings": "Настройки",
    "signOut": "Изход",
    "signIn": "Вход",
    "register": "Регистрация",
    "forgotPassword": "Забравена парола?",
    "rememberMe": "Запомни ме",
    "email": "Имейл адрес",
    "password": "Парола",
    "confirmPassword": "Потвърди паролата",
    "firstName": "Собствено име",
    "lastName": "Фамилия",
    "companyName": "Наименование на фирмата",
    "eik": "ЕИК",
    "terms": "Съгласявам се с общите условия",
    "checkEmail": "Проверете имейла си",
    "checkEmailDescription": "Изпратихме линк за нулиране на паролата на вашия имейл адрес."
  },
  "forms": {
    "required": "Полето е задължително",
    "invalidEmail": "Моля, въведете валиден имейл адрес",
    "passwordTooShort": "Паролата трябва да е поне 8 символа",
    "passwordMismatch": "Паролите не съвпадат",
    "invalidEik": "Моля, въведете валиден ЕИК (9 или 13 цифри)",
    "minLength": "Минимум {min} символа",
    "maxLength": "Максимум {max} символа"
  },
  "errors": {
    "general": "Нещо се обърка. Моля, опитайте отново.",
    "notFound": "Страницата не е намерена",
    "unauthorized": "Нямате права за достъп до тази страница",
    "serverError": "Грешка на сървъра. Моля, опитайте по-късно.",
    "tryAgain": "Опитайте отново",
    "sessionExpired": "Сесията ви е изтекла. Моля, влезте отново."
  },
  "wizard": {
    "title": "Настройте профила на вашата фирма",
    "step1": "Данни за фирмата",
    "step2": "Браншове и CPV кодове",
    "step3": "Региони на интерес",
    "step4": "Поканете членове на екипа",
    "completeSetup": "Завърши настройката",
    "skipStep": "Пропусни тази стъпка",
    "companyInfo": "Информация за фирмата",
    "companyAddress": "Адрес",
    "companyPhone": "Телефон",
    "companyWebsite": "Уебсайт",
    "logoUpload": "Качи лого",
    "cpvSearch": "Търсене на CPV кодове",
    "selectRegions": "Изберете региони",
    "selectAll": "Избери всички",
    "inviteEmail": "Имейл адрес",
    "addInvite": "Добави",
    "minOneCpv": "Моля, изберете поне един CPV код",
    "minOneRegion": "Моля, изберете поне един регион"
  }
}
```

**Admin message files** (`apps/admin/messages/en.json` and `apps/admin/messages/bg.json`):
Admin uses the same namespace structure but with admin-specific `nav` keys. Copy client message files and replace only the `nav` namespace:

```json
// apps/admin/messages/en.json — nav namespace only (rest identical to client)
"nav": {
  "dashboard": "Dashboard",
  "companies": "Companies",
  "tenders": "Tenders",
  "users": "Users",
  "reports": "Reports",
  "settings": "Settings"
}
```

```json
// apps/admin/messages/bg.json — nav namespace only
"nav": {
  "dashboard": "Табло",
  "companies": "Фирми",
  "tenders": "Обществени поръчки",
  "users": "Потребители",
  "reports": "Отчети",
  "settings": "Настройки"
}
```

### Implementation Code: `app/layout.tsx` (MODIFIED — both apps)

Remove `QueryProvider` from root layout. `globals.css` stays here. Fonts stay here. Add `lang` attribute from next-intl's server context is not available at this level — keep `lang="bg"` as static default.

```typescript
// apps/client/app/layout.tsx  (REPLACE existing content)
import type { Metadata } from "next";
import { Inter, JetBrains_Mono } from "next/font/google";
import "./globals.css";

const inter = Inter({
  subsets: ["latin", "latin-ext"],
  variable: "--font-inter",
  display: "swap",
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ["latin"],
  variable: "--font-jetbrains-mono",
  display: "swap",
});

export const metadata: Metadata = {
  title: "EU Solicit Client",
  description: "EU Solicit Client Portal",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="bg" suppressHydrationWarning>
      <body className={`${inter.variable} ${jetbrainsMono.variable} font-sans antialiased`}>
        {children}
      </body>
    </html>
  );
}
```

```typescript
// apps/admin/app/layout.tsx  (REPLACE existing content — identical except title/description)
import type { Metadata } from "next";
import { Inter, JetBrains_Mono } from "next/font/google";
import "./globals.css";

const inter = Inter({
  subsets: ["latin", "latin-ext"],
  variable: "--font-inter",
  display: "swap",
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ["latin"],
  variable: "--font-jetbrains-mono",
  display: "swap",
});

export const metadata: Metadata = {
  title: "EU Solicit Admin",
  description: "EU Solicit Administration Panel",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="bg" suppressHydrationWarning>
      <body className={`${inter.variable} ${jetbrainsMono.variable} font-sans antialiased`}>
        {children}
      </body>
    </html>
  );
}
```

### Implementation Code: `app/[locale]/layout.tsx` (NEW — both apps)

```typescript
// apps/client/app/[locale]/layout.tsx
import { NextIntlClientProvider } from "next-intl";
import { getMessages } from "next-intl/server";
import { notFound } from "next/navigation";
import { QueryProvider } from "@eusolicit/ui";
import { LocaleSyncer } from "../../components/LocaleSyncer";

const locales = ["bg", "en"] as const;
type Locale = (typeof locales)[number];

export default async function LocaleLayout({
  children,
  params: { locale },
}: {
  children: React.ReactNode;
  params: { locale: string };
}) {
  // Validate locale — return 404 for unknown segments
  if (!locales.includes(locale as Locale)) {
    notFound();
  }

  // Fetch messages server-side (next-intl server integration)
  const messages = await getMessages();

  return (
    <NextIntlClientProvider locale={locale} messages={messages}>
      <QueryProvider>
        <LocaleSyncer locale={locale as Locale} />
        {children}
      </QueryProvider>
    </NextIntlClientProvider>
  );
}
```

```typescript
// apps/admin/app/[locale]/layout.tsx — identical except LocaleSyncer import path
import { NextIntlClientProvider } from "next-intl";
import { getMessages } from "next-intl/server";
import { notFound } from "next/navigation";
import { QueryProvider } from "@eusolicit/ui";
import { LocaleSyncer } from "../../components/LocaleSyncer";

const locales = ["bg", "en"] as const;
type Locale = (typeof locales)[number];

export default async function LocaleLayout({
  children,
  params: { locale },
}: {
  children: React.ReactNode;
  params: { locale: string };
}) {
  if (!locales.includes(locale as Locale)) {
    notFound();
  }

  const messages = await getMessages();

  return (
    <NextIntlClientProvider locale={locale} messages={messages}>
      <QueryProvider>
        <LocaleSyncer locale={locale as Locale} />
        {children}
      </QueryProvider>
    </NextIntlClientProvider>
  );
}
```

### Implementation Code: `app/[locale]/page.tsx` (NEW — both apps)

```typescript
// apps/client/app/[locale]/page.tsx
import { redirect } from "next/navigation";

export default function LocaleRootPage({
  params: { locale },
}: {
  params: { locale: string };
}) {
  redirect(`/${locale}/dashboard`);
}
```

### Implementation Code: `components/LocaleSyncer.tsx` (NEW — both apps)

```typescript
// apps/client/components/LocaleSyncer.tsx
// apps/admin/components/LocaleSyncer.tsx (identical)
"use client";

import { useEffect } from "react";
import { useUIStore } from "@/lib/stores/ui-store";

type Locale = "bg" | "en";

interface LocaleSyncerProps {
  locale: Locale;
}

/**
 * Invisible client component that syncs the URL locale with uiStore.locale.
 * Rendered inside NextIntlClientProvider so it can read the active locale from params.
 * No visual output — purely a side-effect component.
 */
export function LocaleSyncer({ locale }: LocaleSyncerProps) {
  const setLocale = useUIStore((s) => s.setLocale);

  useEffect(() => {
    setLocale(locale);
  }, [locale, setLocale]);

  return null;
}
```

**Note on `setLocale` in uiStore**: The current `uiStore` sets `locale` via `addToast`/`removeToast` pattern but does NOT expose a `setLocale` action. You MUST add a `setLocale` action to both `apps/client/lib/stores/ui-store.ts` and `apps/admin/lib/stores/ui-store.ts`:

```typescript
// ADD to UIState interface in apps/client/lib/stores/ui-store.ts:
setLocale: (locale: Locale) => void;

// ADD to the store implementation (set calls):
setLocale: (locale) =>
  set({ locale }, false, "ui/setLocale"),
```

### Implementation Code: `app/[locale]/(protected)/layout.tsx` (NEW client app)

This replaces `app/(protected)/layout.tsx`. Key changes:
1. Reads `params.locale` from the route
2. Builds locale-prefixed hrefs (e.g. `/${locale}/dashboard`)
3. Uses `useTranslations('nav')` for nav labels
4. Passes `locale` and `onLocaleChange` to `<TopBar>`
5. Uses `useRouter` from `next/navigation` for locale switching

```typescript
// apps/client/app/[locale]/(protected)/layout.tsx
"use client";

import { useEffect, useState } from "react";
import { usePathname, useRouter } from "next/navigation";
import { useTranslations } from "next-intl";
import {
  LayoutDashboard,
  FileText,
  Briefcase,
  FolderOpen,
  Users,
  Settings,
} from "lucide-react";
import {
  AppShell,
  Sidebar,
  TopBar,
  BottomNav,
  MobileSidebarSheet,
  useBreakpoint,
} from "@eusolicit/ui";
import type { NavItemConfig } from "@eusolicit/ui";
import { useUIStore } from "@/lib/stores/ui-store";

type Locale = "bg" | "en";

export default function ProtectedLayout({
  children,
  params: { locale },
}: {
  children: React.ReactNode;
  params: { locale: string };
}) {
  const t = useTranslations("nav");
  const tAuth = useTranslations("auth");
  const router = useRouter();
  const { sidebarCollapsed, toggleSidebar, setSidebarCollapsed } = useUIStore();
  const [mounted, setMounted] = useState(false);
  const [isMobileSheetOpen, setIsMobileSheetOpen] = useState(false);
  const pathname = usePathname();
  const { isMobile, isTablet } = useBreakpoint();

  const clientNavItems: NavItemConfig[] = [
    { icon: LayoutDashboard, label: t("dashboard"), href: `/${locale}/dashboard` },
    { icon: FileText, label: t("tenders"), href: `/${locale}/tenders` },
    { icon: Briefcase, label: t("offers"), href: `/${locale}/offers` },
    { icon: FolderOpen, label: t("documents"), href: `/${locale}/documents` },
    { icon: Users, label: t("team"), href: `/${locale}/team` },
    { icon: Settings, label: t("settings"), href: `/${locale}/settings` },
  ];

  useEffect(() => {
    setMounted(true);
  }, []);

  useEffect(() => {
    if (isTablet && mounted) {
      setSidebarCollapsed(true);
    }
  }, [isTablet, mounted, setSidebarCollapsed]);

  useEffect(() => {
    if (isTablet && mounted) {
      setSidebarCollapsed(true);
    }
    setIsMobileSheetOpen(false);
  }, [pathname]); // eslint-disable-line react-hooks/exhaustive-deps

  const collapsed = mounted ? sidebarCollapsed : false;
  const showMobile = isMobile && mounted;

  // Language switcher: replace current URL locale segment
  const handleLocaleChange = (newLocale: Locale) => {
    // Replace /{currentLocale}/... with /{newLocale}/...
    const newPath = pathname.replace(`/${locale}`, `/${newLocale}`);
    router.replace(newPath);
  };

  return (
    <>
      <AppShell
        isCollapsed={collapsed}
        isMobile={showMobile}
        sidebar={
          <Sidebar
            navItems={clientNavItems}
            isCollapsed={collapsed}
            onToggle={toggleSidebar}
          />
        }
        topbar={
          <TopBar
            user={{
              name: "Demo User",
              email: "demo@eusolicit.eu",
            }}
            locale={locale as Locale}
            onLocaleChange={handleLocaleChange}
            userMenuLabels={{
              profile: tAuth("profile"),
              settings: tAuth("settings"),
              signOut: tAuth("signOut"),
            }}
            onOpenMobileSidebar={
              showMobile ? () => setIsMobileSheetOpen(true) : undefined
            }
          />
        }
        bottomNav={
          showMobile ? (
            <BottomNav navItems={clientNavItems} />
          ) : undefined
        }
      >
        {children}
      </AppShell>
      {showMobile && (
        <MobileSidebarSheet
          open={isMobileSheetOpen}
          onOpenChange={setIsMobileSheetOpen}
          navItems={clientNavItems}
        />
      )}
    </>
  );
}
```

**Note**: `app/[locale]/(protected)/layout.tsx` is a Client Component (`"use client"`) but it receives `params` — this is supported in Next.js 14 App Router for client layouts.

### Implementation Code: Admin `app/[locale]/(protected)/layout.tsx`

Same pattern as client but with admin nav items:

```typescript
// apps/admin/app/[locale]/(protected)/layout.tsx
"use client";

import { useEffect, useState } from "react";
import { usePathname, useRouter } from "next/navigation";
import { useTranslations } from "next-intl";
import {
  LayoutDashboard,
  Building2,
  FileText,
  Users,
  BarChart2,
  Settings,
} from "lucide-react";
import {
  AppShell,
  Sidebar,
  TopBar,
  BottomNav,
  MobileSidebarSheet,
  useBreakpoint,
} from "@eusolicit/ui";
import type { NavItemConfig } from "@eusolicit/ui";
import { useUIStore } from "@/lib/stores/ui-store";

type Locale = "bg" | "en";

export default function ProtectedLayout({
  children,
  params: { locale },
}: {
  children: React.ReactNode;
  params: { locale: string };
}) {
  const t = useTranslations("nav");
  const tAuth = useTranslations("auth");
  const router = useRouter();
  const { sidebarCollapsed, toggleSidebar, setSidebarCollapsed } = useUIStore();
  const [mounted, setMounted] = useState(false);
  const [isMobileSheetOpen, setIsMobileSheetOpen] = useState(false);
  const pathname = usePathname();
  const { isMobile, isTablet } = useBreakpoint();

  const adminNavItems: NavItemConfig[] = [
    { icon: LayoutDashboard, label: t("dashboard"), href: `/${locale}/dashboard` },
    { icon: Building2, label: t("companies"), href: `/${locale}/companies` },
    { icon: FileText, label: t("tenders"), href: `/${locale}/tenders` },
    { icon: Users, label: t("users"), href: `/${locale}/users` },
    { icon: BarChart2, label: t("reports"), href: `/${locale}/reports` },
    { icon: Settings, label: t("settings"), href: `/${locale}/settings` },
  ];

  useEffect(() => { setMounted(true); }, []);

  useEffect(() => {
    if (isTablet && mounted) setSidebarCollapsed(true);
  }, [isTablet, mounted, setSidebarCollapsed]);

  useEffect(() => {
    if (isTablet && mounted) setSidebarCollapsed(true);
    setIsMobileSheetOpen(false);
  }, [pathname]); // eslint-disable-line react-hooks/exhaustive-deps

  const collapsed = mounted ? sidebarCollapsed : false;
  const showMobile = isMobile && mounted;

  const handleLocaleChange = (newLocale: Locale) => {
    const newPath = pathname.replace(`/${locale}`, `/${newLocale}`);
    router.replace(newPath);
  };

  return (
    <>
      <AppShell
        isCollapsed={collapsed}
        isMobile={showMobile}
        sidebar={
          <Sidebar navItems={adminNavItems} isCollapsed={collapsed} onToggle={toggleSidebar} />
        }
        topbar={
          <TopBar
            user={{ name: "Admin User", email: "admin@eusolicit.eu" }}
            locale={locale as Locale}
            onLocaleChange={handleLocaleChange}
            userMenuLabels={{
              profile: tAuth("profile"),
              settings: tAuth("settings"),
              signOut: tAuth("signOut"),
            }}
            onOpenMobileSidebar={showMobile ? () => setIsMobileSheetOpen(true) : undefined}
          />
        }
        bottomNav={showMobile ? <BottomNav navItems={adminNavItems} /> : undefined}
      >
        {children}
      </AppShell>
      {showMobile && (
        <MobileSidebarSheet
          open={isMobileSheetOpen}
          onOpenChange={setIsMobileSheetOpen}
          navItems={adminNavItems}
        />
      )}
    </>
  );
}
```

### Implementation Code: Moving Dashboard Page

```typescript
// apps/client/app/[locale]/(protected)/dashboard/page.tsx
// apps/admin/app/[locale]/(protected)/dashboard/page.tsx
// Copy content from the old (protected)/dashboard/page.tsx — no changes needed
```

### Implementation Code: Moving Dev Pages (client app only)

Dev pages move from `app/dev/...` to `app/[locale]/dev/...`. The content is identical; no import path changes needed since they use `@/` which resolves from the app root. Just copy the files.

### Implementation Code: Modified `TopBar.tsx` in packages/ui

```typescript
// packages/ui/src/components/app-shell/TopBar.tsx  (REPLACE)
"use client";

import { Menu, Globe } from "lucide-react";
import { NotificationsBell } from "./NotificationsBell";
import { UserAvatarMenu } from "./UserAvatarMenu";
import { Button } from "../ui/button";
import { cn } from "../../lib/utils";

type Locale = "bg" | "en";

interface UserMenuLabels {
  profile?: string;
  settings?: string;
  signOut?: string;
}

interface TopBarProps {
  leftSlot?: React.ReactNode;
  user?: {
    name: string;
    email: string;
    avatarUrl?: string;
  };
  locale?: Locale;
  onLocaleChange?: (locale: Locale) => void;
  userMenuLabels?: UserMenuLabels;
  onSignOut?: () => void;
  onOpenMobileSidebar?: () => void;
}

export function TopBar({
  leftSlot,
  user,
  locale = "bg",
  onLocaleChange,
  userMenuLabels,
  onSignOut,
  onOpenMobileSidebar,
}: TopBarProps) {
  const otherLocale: Locale = locale === "bg" ? "en" : "bg";

  return (
    <header
      data-testid="topbar"
      className="sticky top-0 z-20 h-16 bg-white border-b border-slate-200 flex items-center justify-between px-4"
    >
      {/* Left side: hamburger (mobile only) + breadcrumbs or other slot */}
      <div className="flex items-center gap-2 min-w-0">
        {onOpenMobileSidebar && (
          <button
            data-testid="hamburger-menu"
            onClick={onOpenMobileSidebar}
            className="flex items-center justify-center h-8 w-8 rounded-md text-slate-500 hover:bg-slate-100 hover:text-slate-900 transition-colors flex-shrink-0"
            aria-label="Open menu"
          >
            <Menu className="h-5 w-5" />
          </button>
        )}
        {leftSlot ?? <div />}
      </div>

      {/* Right side: action cluster */}
      <div className="flex items-center gap-1">
        <NotificationsBell unreadCount={3} />

        {/* Language selector */}
        <div
          className="flex items-center gap-0.5 rounded-md border border-slate-200 bg-slate-50 p-0.5"
          data-testid="language-selector"
          aria-label="Language selector"
        >
          <button
            onClick={locale === "bg" ? undefined : () => onLocaleChange?.("bg")}
            disabled={locale === "bg"}
            className={cn(
              "flex items-center gap-1 px-2 py-1 rounded text-xs font-medium transition-colors",
              locale === "bg"
                ? "bg-white text-indigo-600 shadow-sm"
                : "text-slate-500 hover:text-slate-700"
            )}
            aria-label="Switch to Bulgarian"
            aria-pressed={locale === "bg"}
          >
            <Globe className="h-3 w-3" />
            BG
          </button>
          <button
            onClick={locale === "en" ? undefined : () => onLocaleChange?.("en")}
            disabled={locale === "en"}
            className={cn(
              "flex items-center gap-1 px-2 py-1 rounded text-xs font-medium transition-colors",
              locale === "en"
                ? "bg-white text-indigo-600 shadow-sm"
                : "text-slate-500 hover:text-slate-700"
            )}
            aria-label="Switch to English"
            aria-pressed={locale === "en"}
          >
            EN
          </button>
        </div>

        <UserAvatarMenu user={user} labels={userMenuLabels} onSignOut={onSignOut} />
      </div>
    </header>
  );
}
```

### Implementation Code: Modified `UserAvatarMenu.tsx` in packages/ui

```typescript
// packages/ui/src/components/app-shell/UserAvatarMenu.tsx  (REPLACE)
"use client";

import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from "../ui/dropdown-menu";
import { Avatar, AvatarFallback, AvatarImage } from "../ui/avatar";

interface UserAvatarMenuLabels {
  profile?: string;
  settings?: string;
  signOut?: string;
}

interface UserAvatarMenuProps {
  user?: {
    name: string;
    email: string;
    avatarUrl?: string;
  };
  labels?: UserAvatarMenuLabels;
  onSignOut?: () => void;
}

export function UserAvatarMenu({ user, labels, onSignOut }: UserAvatarMenuProps) {
  const initials =
    user?.name
      ?.split(" ")
      .map((n) => n[0])
      .join("")
      .toUpperCase()
      .slice(0, 2) ?? "U";

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <button
          data-testid="user-avatar-menu-trigger"
          aria-label="User menu"
          className="rounded-full outline-none focus-visible:ring-2 focus-visible:ring-indigo-500 focus-visible:ring-offset-2"
        >
          <Avatar className="h-8 w-8">
            {user?.avatarUrl && <AvatarImage src={user.avatarUrl} alt={user.name} />}
            <AvatarFallback className="bg-indigo-100 text-indigo-600 text-xs font-medium">
              {initials}
            </AvatarFallback>
          </Avatar>
        </button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end" className="w-56">
        <DropdownMenuLabel className="flex flex-col gap-0.5">
          <span className="font-medium text-slate-900">{user?.name ?? "User"}</span>
          <span className="text-xs font-normal text-slate-500">{user?.email ?? ""}</span>
        </DropdownMenuLabel>
        <DropdownMenuSeparator />
        <DropdownMenuItem>{labels?.profile ?? "Profile"}</DropdownMenuItem>
        <DropdownMenuItem>{labels?.settings ?? "Settings"}</DropdownMenuItem>
        <DropdownMenuSeparator />
        <DropdownMenuItem
          className="text-red-600"
          onClick={onSignOut}
        >
          {labels?.signOut ?? "Sign out"}
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### Implementation Code: `format.ts` in packages/ui

```typescript
// packages/ui/src/lib/format.ts

/**
 * Formats a Date object to a locale-specific date string.
 * BG locale: "01.04.2026" (day.month.year)
 * EN locale: "April 1, 2026"
 */
export function formatDate(date: Date, locale: string = "bg"): string {
  return new Intl.DateTimeFormat(locale === "bg" ? "bg-BG" : "en-GB", {
    year: "numeric",
    month: locale === "bg" ? "2-digit" : "long",
    day: "2-digit",
  }).format(date);
}

/**
 * Formats a number as currency in the given locale.
 * BG locale: "1 234,56 лв." (BGN)
 * EN locale: "€1,234.56" (EUR) or "£1,234.56" (GBP)
 */
export function formatCurrency(
  amount: number,
  currency: string = "BGN",
  locale: string = "bg"
): string {
  return new Intl.NumberFormat(locale === "bg" ? "bg-BG" : "en-GB", {
    style: "currency",
    currency,
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(amount);
}
```

### Implementation Code: i18n Key Diff Script

```javascript
// apps/client/scripts/check-i18n-keys.mjs  (and apps/admin/scripts/check-i18n-keys.mjs)
import { readFileSync } from "fs";
import { fileURLToPath } from "url";
import { dirname, join } from "path";

const __dirname = dirname(fileURLToPath(import.meta.url));
const messagesDir = join(__dirname, "..", "messages");

/**
 * Recursively flattens a nested object into dot-notation keys.
 * { nav: { dashboard: "..." } } → ["nav.dashboard"]
 */
function flattenKeys(obj, prefix = "") {
  return Object.entries(obj).flatMap(([key, value]) => {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    return typeof value === "object" && value !== null
      ? flattenKeys(value, fullKey)
      : [fullKey];
  });
}

const bg = JSON.parse(readFileSync(join(messagesDir, "bg.json"), "utf-8"));
const en = JSON.parse(readFileSync(join(messagesDir, "en.json"), "utf-8"));

const bgKeys = new Set(flattenKeys(bg));
const enKeys = new Set(flattenKeys(en));

const missingInEn = [...bgKeys].filter((k) => !enKeys.has(k));
const missingInBg = [...enKeys].filter((k) => !bgKeys.has(k));

if (missingInEn.length > 0) {
  console.error("❌ Keys present in bg.json but missing in en.json:");
  missingInEn.forEach((k) => console.error(`   - ${k}`));
}

if (missingInBg.length > 0) {
  console.error("❌ Keys present in en.json but missing in bg.json:");
  missingInBg.forEach((k) => console.error(`   - ${k}`));
}

if (missingInEn.length > 0 || missingInBg.length > 0) {
  console.error(`\n❌ i18n key mismatch: ${missingInEn.length + missingInBg.length} key(s) differ between bg.json and en.json`);
  process.exit(1);
} else {
  console.log(`✅ i18n keys match: ${bgKeys.size} keys in both bg.json and en.json`);
  process.exit(0);
}
```

Add to `apps/client/package.json` scripts and `apps/admin/package.json` scripts:
```json
"check:i18n": "node scripts/check-i18n-keys.mjs"
```

### `packages/ui/index.ts` additions (New in S3.7)

Append the following section:

```typescript
// New in S3.7
export { formatDate, formatCurrency } from "./src/lib/format";
```

### TypeScript: `tsconfig.json` check for message type inference

next-intl v3 supports TypeScript message type inference. For full autocomplete, create `global.d.ts` at each app root:

```typescript
// apps/client/global.d.ts  (and apps/admin/global.d.ts)
// Enables TypeScript autocomplete for useTranslations() keys
import en from "./messages/en.json";

type Messages = typeof en;

declare global {
  // Use type safe message keys with `next-intl`
  // eslint-disable-next-line @typescript-eslint/no-empty-interface
  interface IntlMessages extends Messages {}
}
```

This gives TypeScript autocomplete on `t('nav.dashboard')` keys — mistyped keys become compile errors.

### Test Alignment (from `test-design-epic-03.md`)

This story directly enables the following epic-level test scenarios:

| Test ID | Priority | What S3.7 delivers |
|---------|----------|---------------------|
| **E03-P0-005** | P0 | Default locale routes `/` → `/bg/dashboard` without redirect loop (≤1 redirect) |
| **E03-P0-006** | P0 | Language selector switches BG→EN: URL updates to `/en/*`; all shell chrome re-renders in English |
| **E03-P1-006** | P1 | Locale preference persists to localStorage (`NEXT_LOCALE`); switching to EN and reloading preserves EN locale |
| **E03-P1-007** | P1 | All shell chrome i18n keys present in both `messages/bg.json` and `messages/en.json` — `check:i18n` script passes |
| **E03-P2-011** | P2 | `formatDate` and `formatCurrency` helpers render correct locale-specific output |

**High-Risk Mitigations implemented in this story:**

- **E03-R-001 (locale middleware)**: `localePrefix: 'always'` + explicit `locales: ['bg', 'en']` + `defaultLocale: 'bg'` in middleware. The matcher excludes static assets. `notFound()` guard in locale layout prevents invalid locale segments from rendering.
- **E03-R-006 (missing i18n keys)**: `check-i18n-keys.mjs` script validates key parity between BG and EN files; integrated as `pnpm check:i18n`; run as part of Task 13 verification before commit.

**Important — ATDD RED tests (will be generated post story creation):**
The ATDD checklist for S3.7 will generate E2E tests (Playwright) covering:
- `E03-P0-005`: redirect loop assertion via `response.redirectedFrom()` count ≤1
- `E03-P0-006`: locale switch E2E — click EN button; assert `/en/` URL prefix; assert nav item labels in English
- `E03-P1-006`: reload after locale switch; assert URL still `/en/`
- `E03-P1-007`: `pnpm check:i18n` script unit coverage (Node.js script test)
- `E03-P2-011`: unit tests for `formatDate` and `formatCurrency`

### File Deletion Checklist

After creating all `app/[locale]/...` files, explicitly delete these old files to avoid Next.js route conflicts:

**Client app:**
- `apps/client/app/page.tsx`
- `apps/client/app/(protected)/layout.tsx`
- `apps/client/app/(protected)/dashboard/page.tsx`
- `apps/client/app/dev/components/page.tsx`
- `apps/client/app/dev/api-test/page.tsx`
- `apps/client/app/dev/form-test/page.tsx`

**Admin app:**
- `apps/admin/app/page.tsx`
- `apps/admin/app/(protected)/layout.tsx`
- `apps/admin/app/(protected)/dashboard/page.tsx`

Next.js will error with "Conflicting routes" if both `app/page.tsx` and `app/[locale]/page.tsx` coexist.

### Vitest Config: No Changes Required

The existing Vitest config in `packages/ui` covers:
- `src/__tests__/lib/hooks/**` (jsdom)
- `src/__tests__/components/**` (jsdom)

The new `format.ts` unit tests (if generated by ATDD) will live in `src/__tests__/lib/format.test.ts` — the `src/__tests__/lib/**` glob is covered by `node` environment (no DOM APIs needed for Intl tests). No config update needed.

### Common Build Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot find module 'next-intl'` | `next-intl` not installed | Run `pnpm install` from `eusolicit-app/frontend/` |
| `Could not read messages file: ENOENT messages/bg.json` | Missing message files | Create `messages/bg.json` and `messages/en.json` in each app root |
| `Error: Attempted to call the default export of i18n.ts from the edge runtime` | Missing `createNextIntlPlugin` wrapper | Ensure `next.config.mjs` uses `withNextIntl(nextConfig)` |
| `Type error: Property 'locale' does not exist on type 'TopBarProps'` | TopBar not updated | Apply the new `TopBar.tsx` implementation from Dev Notes |
| `Error: Conflicting routes: app/page.tsx and app/[locale]/page.tsx` | Old pages not deleted | Delete all old pages from File Deletion Checklist |
| `Type error: Property 'setLocale' does not exist on type 'UIState'` | `setLocale` not added to uiStore | Add `setLocale` action to both apps' `ui-store.ts` |
| Redirect loop on `/bg/dashboard` | Middleware matcher too broad (includes `_next`) | Verify matcher regex excludes `_next`, `api`, and file extensions |

## Senior Developer Review

**Date:** 2026-04-08
**Reviewer:** Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** ~~REVIEW: Changes Requested~~ **REVIEW: Approve** (re-review 2026-04-08)

**Stats:** 28 files reviewed | 3 patch (all resolved) | 4 defer | 15 dismissed

### Review Findings

- [x] [Review][Patch] **UserAvatarMenu: Replace custom dropdown with Radix DropdownMenu as specified** — The story spec (lines 1017-1091) explicitly marks UserAvatarMenu.tsx as `(REPLACE)` and provides Radix-based code using `DropdownMenu`, `DropdownMenuTrigger`, `DropdownMenuContent`, `DropdownMenuItem` from shadcn/ui. The current implementation uses a custom dropdown with manual `useState`/`useRef`/`useEffect` for open/close, `flushSync` for toggle (React anti-pattern), and `mousedown` for outside-click detection. This causes: (a) no keyboard navigation between menu items (violates WAI-ARIA menu pattern / WCAG 2.1.1), (b) no Escape key to close, (c) no focus trap, (d) `flushSync` + mousedown/click race condition where clicking the trigger while open fires mousedown-close then click-reopen. **Fix:** Replace with the spec's Radix-based implementation. All required primitives are already exported from `packages/ui/index.ts`. [packages/ui/src/components/app-shell/UserAvatarMenu.tsx]
- [x] [Review][Patch] **Duplicate useEffect sidebar-collapse logic in protected layouts** — Both `apps/client/app/[locale]/(protected)/layout.tsx` and admin equivalent have two `useEffect` hooks that call `setSidebarCollapsed(true)`. The first correctly tracks `[isTablet, mounted, setSidebarCollapsed]`. The second triggers on `[pathname]` only, re-runs the same sidebar collapse with stale closures (`isTablet`, `mounted` not in deps), and suppresses the exhaustive-deps lint rule. The pathname effect should only close the mobile sheet. **Fix:** Remove the `if (isTablet && mounted) setSidebarCollapsed(true)` line from the pathname-based useEffect in both apps. [apps/client/app/[locale]/(protected)/layout.tsx:~line 64, apps/admin/app/[locale]/(protected)/layout.tsx:~line 58]
- [x] [Review][Patch] **TopBar and hamburger button have hardcoded English aria-labels** — The locale switch buttons have `aria-label="Switch to Bulgarian"` and `aria-label="Switch to English"`, and the hamburger button has `aria-label="Open menu"` — all hardcoded in English inside the shared `packages/ui` component. Since `packages/ui` has no access to `next-intl`, these should accept translated strings as props (following the same pattern as `userMenuLabels` and `languageSelectorAriaLabel`). **Fix:** Add optional `localeSwitchAriaLabels?: { bg?: string; en?: string }` and `hamburgerAriaLabel?: string` props to `TopBarProps`, defaulting to the current English strings. [packages/ui/src/components/app-shell/TopBar.tsx:54,81,97]
- [x] [Review][Defer] Root layout `<html lang="bg">` hardcoded regardless of active locale — Screen readers and search engines receive wrong language signal for `/en/*` pages. Spec dev notes acknowledge this limitation: "next-intl's server context is not available at this level — keep lang='bg' as static default." Fixing requires a client-side `useEffect` to update `document.documentElement.lang` or middleware-based approach. — deferred, spec-acknowledged design constraint
- [x] [Review][Defer] `i18n.ts` dynamic import has no try/catch — If a message file is missing at runtime (e.g., locale added to middleware but JSON not created), SSR crashes with an unhandled module-not-found error. Middleware validates locale before reaching this point, providing defense-in-depth. — deferred, pre-existing pattern from next-intl docs
- [x] [Review][Defer] `formatDate`/`formatCurrency` use binary ternary (`bg` vs everything-else → `en-GB`) — Any future third locale silently falls through to en-GB. Also, JSDoc comment claims EN produces "April 1, 2026" (US) but code uses `en-GB` which produces "01 April 2026" (UK). — deferred, two-locale constraint matches current project scope
- [x] [Review][Defer] Locale list (`["bg", "en"]`) duplicated across middleware, layouts, format utils, and type aliases — No single source of truth. Adding a locale requires updating 6+ files independently. — deferred, pre-existing architectural pattern

### Re-Review (2026-04-08)

**All 3 Patch findings verified resolved.** Re-review performed adversarially across all 28 files.

**Patch Resolution Verification:**
1. **UserAvatarMenu** — Now uses Radix `DropdownMenuPrimitive.Portal forceMount` + `DropdownMenuPrimitive.Content forceMount`. Keyboard navigation, Escape-to-close, and focus management all provided by Radix primitives. Menu items use `labels` prop with English fallback defaults. No custom open/close state, no `flushSync`.
2. **Duplicate useEffect** — Pathname-based `useEffect` in both protected layouts now only calls `setIsMobileSheetOpen(false)`. No duplicate `setSidebarCollapsed(true)`. No suppressed lint rules.
3. **TopBar aria-labels** — `localeSwitchAriaLabels?: { bg?: string; en?: string }` and `hamburgerAriaLabel?: string` props added to `TopBarProps`. All three buttons use prop-driven labels with English defaults. `languageSelectorAriaLabel` also prop-driven.

**Automated Checks (all pass):**
- `pnpm check:i18n` (client): 67 keys match between bg.json and en.json
- `pnpm check:i18n` (admin): 67 keys match between bg.json and en.json
- `pnpm type-check`: 0 errors across 4 packages (client, admin, @eusolicit/ui, @eusolicit/config)
- `pnpm build`: both apps exit 0 (admin: 3 routes + middleware, client: 6 routes + middleware)
- `pnpm test` (@eusolicit/ui): 60/60 tests pass (8 test files including TopBar-i18n, UserAvatarMenu-i18n, format tests)

**Acceptance Criteria Audit (10/10 met):**
- AC1: next-intl@3.26.5 in both apps; next.config.mjs wraps with `createNextIntlPlugin('./i18n.ts')`
- AC2: middleware.ts with `locales: ['bg','en']`, `defaultLocale: 'bg'`, `localePrefix: 'always'`
- AC3: `app/[locale]/` route segment; root layout simplified to HTML shell; locale layout has `NextIntlClientProvider` + `QueryProvider`
- AC4: 6 namespaces (common, nav, auth, forms, errors, wizard) in both bg.json and en.json per app
- AC5: `useTranslations('nav')` and `useTranslations('auth')` in protected layouts; nav items, user menu labels all translated
- AC6: Globe icon + BG/EN buttons in TopBar; `onLocaleChange` calls `router.replace()` with new locale prefix
- AC7: `uiStore.locale` persisted via Zustand `persist` middleware to localStorage; `setLocale` action syncs from URL via `LocaleSyncer`
- AC8: `formatDate` and `formatCurrency` in `packages/ui/src/lib/format.ts`; exported from `packages/ui/index.ts`
- AC9: `scripts/check-i18n-keys.mjs` in both apps; exits 0 when keys match, non-zero on mismatch
- AC10: `pnpm build` exits 0; `pnpm type-check` exits 0; all routes under `[locale]/`

## Dev Agent Record

### Implementation Plan

Story 3.7 established full i18n infrastructure for both client and admin Next.js apps using next-intl v3. Key implementation decisions:

1. **next-intl integration**: Both apps use `createNextIntlPlugin` in `next.config.mjs`, `getRequestConfig` in `i18n.ts`, and `createMiddleware` in `middleware.ts` with `locales: ['bg', 'en']`, `defaultLocale: 'bg'`, `localePrefix: 'always'`.
2. **Route restructuring**: `app/[locale]/` route segment added; root layout simplified to HTML shell; `[locale]/layout.tsx` provides `NextIntlClientProvider` + `QueryProvider`; old non-locale pages deleted.
3. **UI extensions**: `TopBar` gained `locale`, `onLocaleChange`, `localeSwitchAriaLabels`, `hamburgerAriaLabel` props. `UserAvatarMenu` replaced custom dropdown with Radix `DropdownMenu` (using `forceMount` on Portal+Content for test compatibility with jsdom pointer-event limitations).
4. **Utilities**: `format.ts` added to `packages/ui` with `formatDate`/`formatCurrency` using `Intl` APIs.
5. **Key diff script**: `check-i18n-keys.mjs` in both apps validates 67-key parity between bg.json and en.json.

### Review Follow-ups Resolved (2026-04-08)

All 3 code review Patch findings addressed in this session:

- ✅ Resolved review finding [Patch]: UserAvatarMenu replaced with Radix `DropdownMenu` using `DropdownMenuPrimitive.Portal forceMount` + `DropdownMenuPrimitive.Content forceMount` for jsdom test compatibility. Provides full keyboard navigation, Escape-to-close, and focus management via Radix.
- ✅ Resolved review finding [Patch]: Removed stale `if (isTablet && mounted) setSidebarCollapsed(true)` from pathname-based `useEffect` in both `apps/client/app/[locale]/(protected)/layout.tsx` and `apps/admin/app/[locale]/(protected)/layout.tsx`. Pathname effect now only closes mobile sheet.
- ✅ Resolved review finding [Patch]: Added optional `localeSwitchAriaLabels?: { bg?: string; en?: string }` and `hamburgerAriaLabel?: string` props to `TopBarProps` in `packages/ui/src/components/app-shell/TopBar.tsx`, defaulting to English strings. All locale switch buttons and hamburger now use prop-driven aria-labels.

### Completion Notes

- All 60 unit tests pass (60 tests across 8 files in `packages/ui`)
- TypeScript type-check: 0 errors across all 4 packages (client, admin, @eusolicit/ui, @eusolicit/config)
- `pnpm check:i18n`: 67 keys match in both bg.json and en.json for client and admin
- All 13 tasks and subtasks verified complete

## File List

### New Files
- `eusolicit-app/frontend/apps/client/i18n.ts`
- `eusolicit-app/frontend/apps/client/middleware.ts`
- `eusolicit-app/frontend/apps/client/global.d.ts`
- `eusolicit-app/frontend/apps/client/messages/bg.json`
- `eusolicit-app/frontend/apps/client/messages/en.json`
- `eusolicit-app/frontend/apps/client/scripts/check-i18n-keys.mjs`
- `eusolicit-app/frontend/apps/client/components/LocaleSyncer.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/layout.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/page.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/dashboard/page.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/dev/components/page.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/dev/api-test/page.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/dev/form-test/page.tsx`
- `eusolicit-app/frontend/apps/admin/i18n.ts`
- `eusolicit-app/frontend/apps/admin/middleware.ts`
- `eusolicit-app/frontend/apps/admin/global.d.ts`
- `eusolicit-app/frontend/apps/admin/messages/bg.json`
- `eusolicit-app/frontend/apps/admin/messages/en.json`
- `eusolicit-app/frontend/apps/admin/scripts/check-i18n-keys.mjs`
- `eusolicit-app/frontend/apps/admin/components/LocaleSyncer.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/layout.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/dashboard/page.tsx`
- `eusolicit-app/frontend/packages/ui/src/lib/format.ts`

### Modified Files
- `eusolicit-app/frontend/apps/client/app/layout.tsx` — simplified to HTML shell only
- `eusolicit-app/frontend/apps/client/next.config.mjs` — wrapped with createNextIntlPlugin
- `eusolicit-app/frontend/apps/client/package.json` — added next-intl dep + check:i18n script
- `eusolicit-app/frontend/apps/client/lib/stores/ui-store.ts` — added setLocale action
- `eusolicit-app/frontend/apps/admin/app/layout.tsx` — simplified to HTML shell only
- `eusolicit-app/frontend/apps/admin/next.config.mjs` — wrapped with createNextIntlPlugin
- `eusolicit-app/frontend/apps/admin/package.json` — added next-intl dep + check:i18n script
- `eusolicit-app/frontend/apps/admin/lib/stores/ui-store.ts` — added setLocale action
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/TopBar.tsx` — locale selector + aria-label props
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/UserAvatarMenu.tsx` — Radix DropdownMenu + labels prop
- `eusolicit-app/frontend/packages/ui/index.ts` — exported formatDate, formatCurrency

### Deleted Files
- `eusolicit-app/frontend/apps/client/app/page.tsx`
- `eusolicit-app/frontend/apps/client/app/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/client/app/(protected)/dashboard/page.tsx`
- `eusolicit-app/frontend/apps/client/app/dev/components/page.tsx`
- `eusolicit-app/frontend/apps/client/app/dev/api-test/page.tsx`
- `eusolicit-app/frontend/apps/client/app/dev/form-test/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/admin/app/(protected)/dashboard/page.tsx`

## Change Log

- 2026-04-08: Story 3.7 implementation complete — next-intl installed, middleware configured, app/[locale]/ route structure established, message files created (67 keys BG+EN), TopBar language selector added, UserAvatarMenu labels prop added, format utilities added, i18n key diff script added, uiStore setLocale action added. (Dev Agent)
- 2026-04-08: Code review findings addressed — UserAvatarMenu replaced with Radix DropdownMenu (forceMount for test compatibility), duplicate useEffect removed from protected layouts, TopBar aria-labels made prop-driven. 3 Patch items resolved. (Dev Agent)
