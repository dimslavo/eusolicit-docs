# Story 3.9: Company Profile Setup Wizard

Status: done

## Story

As a **newly registered EU Solicit user**,
I want **to walk through a four-step company profile setup wizard that collects my company details, CPV sector interests, regions of interest, and team invite emails — with my progress automatically saved so I can refresh mid-wizard without losing data**,
so that **my company profile is fully configured and I land on the dashboard ready to use EU Solicit's tender matching features**.

## Acceptance Criteria

1. Wizard page exists at `app/[locale]/(protected)/setup/page.tsx` within the `(protected)` route group; the page renders a `<WizardStepper>` component above the active step content; the stepper displays four labelled steps (from `wizard` namespace translations) with a check icon for completed steps, an indigo-600 ring for the active step, and a slate-300 ring for future steps
2. Step 1 — Company Info: `companyName` and `eik` fields pre-filled from `authStore.user` (`user.name` → company name, looked up from `authStore`; in stub context, they are pre-populated from `wizardStore.step1` defaults seeded from user); address, phone, and website text inputs; logo upload via `<input type="file" accept="image/*">` with a 128×128 rounded preview (using `FileReader.readAsDataURL` to produce a persistent base64 `logoDataUrl`); "Next" validates Step 1 schema before advancing — address is required (min 2 chars), phone is optional, website is optional (if provided, must be a valid URL), logo upload is optional
3. Step 2 — Industry & CPV: a text search input filters `CPV_CODES` from `apps/client/lib/data/cpv-codes.ts` by code or description; matching items render in a scrollable list (max-h-48 `overflow-y-auto`); clicking a result adds it to `wizardStore.step2.cpvCodes` and renders it as a dismissible `<Badge>` below the search box; clicking the `×` on a badge removes it; "Next" validates at least one CPV code selected — shows `t("wizard.minOneCpv")` error message inline if zero selected
4. Step 3 — Regions: two labelled sections — "Bulgarian Planning Regions" (6 items from `BULGARIAN_REGIONS`) and "EU Member States" (27 items from `EU_MEMBER_STATES`); each section has a "Select all" toggle that checks/unchecks all items in that section; selections stored in `wizardStore.step3.regions` as region `id` strings; "Next" validates at least one region is selected — shows `t("wizard.minOneRegion")` error if none selected
5. Step 4 — Team Invites: an email text input and "Add" button; "Add" validates the email with a simple Zod email check — shows an inline error if invalid; valid emails appended to `wizardStore.step4.teamEmails`; each added email renders in a list with a remove `×` button; "Complete Setup" is always enabled on Step 4 regardless of how many (if any) emails are added
6. Navigation: "Back" button (Steps 2–4) navigates to the previous step without validation; "Next" button (Steps 1–3) validates the current step's Zod schema and only advances on success; "Complete Setup" button (Step 4) shows a `<Loader2 className="animate-spin" />` spinner during submission; submits to `updateCompanyProfile()` stub in `apps/client/lib/api/company.ts`; on success calls `wizardStore.reset()` and redirects to `/${locale}/dashboard`
7. Wizard state persisted in `useWizardStore` (Zustand `persist` middleware, name `"eusolicit-wizard-store"`); `partialize` excludes `logoFile` (non-serialisable `File` object); `logoDataUrl` (base64 string) IS persisted so the logo preview survives reload; all other step data (`cpvCodes`, `regions`, `teamEmails`, `address`, `phone`, `website`) is persisted; on reload with persisted state, Step 2–4 data is fully pre-populated
8. After "Complete Setup" succeeds, `wizardStore.reset()` clears the store and removes the `eusolicit-wizard-store` key from `localStorage` before `router.replace(`/${locale}/dashboard`)` is called
9. Register page (`app/[locale]/(auth)/register/page.tsx`) is updated: `router.replace(`/${locale}/dashboard`)` on successful registration changes to `router.replace(`/${locale}/setup`)` so newly registered users land directly on the wizard
10. All wizard UI strings use `useTranslations("wizard")` or `useTranslations("common")`; all required keys are already present in both `messages/en.json` and `messages/bg.json` (added in S03.07); no new i18n keys are required for this story
11. `pnpm build` exits 0 for both apps; `pnpm type-check` exits 0 across all packages; `pnpm check:i18n --filter client` exits 0

## Tasks / Subtasks

- [x] Task 1: Create `wizardStore` Zustand store (AC: 7, 8)
  - [x] 1.1 Create `apps/client/lib/stores/wizard-store.ts` — Zustand store with `persist` middleware; define `WizardStep1`–`WizardStep4` data interfaces and full `WizardState`; exclude `logoFile` from `partialize`; see Dev Notes for full implementation

- [x] Task 2: Create Zod validation schemas for each wizard step (AC: 2, 3, 4, 5, 6)
  - [x] 2.1 Create `apps/client/lib/schemas/wizard.ts` — `step1Schema`, `step2Schema`, `step3Schema`, `teamEmailSchema`; see Dev Notes for full implementation

- [x] Task 3: Create static CPV codes data file (AC: 3)
  - [x] 3.1 Create `apps/client/lib/data/cpv-codes.ts` — `CpvCode` interface + `CPV_CODES` array of ~30 top-level CPV divisions; see Dev Notes for full data

- [x] Task 4: Create static regions data file (AC: 4)
  - [x] 4.1 Create `apps/client/lib/data/regions.ts` — `Region` interface + `BULGARIAN_REGIONS` (6 items) + `EU_MEMBER_STATES` (27 items); see Dev Notes for full data

- [x] Task 5: Create company profile API stub (AC: 6)
  - [x] 5.1 Create `apps/client/lib/api/company.ts` — `updateCompanyProfile()` stub function with 800ms delay; see Dev Notes for full implementation

- [x] Task 6: Build `WizardStepper` component (AC: 1)
  - [x] 6.1 Create `apps/client/app/[locale]/(protected)/setup/components/WizardStepper.tsx` — horizontal stepper with step number, label, icon, and status styling; see Dev Notes for full implementation

- [x] Task 7: Build Step 1 — Company Info component (AC: 2)
  - [x] 7.1 Create `apps/client/app/[locale]/(protected)/setup/components/Step1CompanyInfo.tsx` — form fields for address, phone, website, logo upload with preview; reads/writes `wizardStore.step1`; see Dev Notes for full implementation

- [x] Task 8: Build Step 2 — Industry & CPV component (AC: 3)
  - [x] 8.1 Create `apps/client/app/[locale]/(protected)/setup/components/Step2CPVSectors.tsx` — searchable CPV list, badge selection UI; reads/writes `wizardStore.step2`; see Dev Notes for full implementation

- [x] Task 9: Build Step 3 — Regions component (AC: 4)
  - [x] 9.1 Create `apps/client/app/[locale]/(protected)/setup/components/Step3Regions.tsx` — checkable region sections with select-all toggles; reads/writes `wizardStore.step3`; see Dev Notes for full implementation

- [x] Task 10: Build Step 4 — Team Invites component (AC: 5)
  - [x] 10.1 Create `apps/client/app/[locale]/(protected)/setup/components/Step4TeamInvites.tsx` — email add/remove list; reads/writes `wizardStore.step4`; see Dev Notes for full implementation

- [x] Task 11: Build main wizard page (AC: 1, 6, 7, 8)
  - [x] 11.1 Create `apps/client/app/[locale]/(protected)/setup/page.tsx` — orchestrates stepper + active step component + Back/Next/Complete navigation; auth redirect guard; see Dev Notes for full implementation

- [x] Task 12: Update register page redirect to wizard (AC: 9)
  - [x] 12.1 Edit `apps/client/app/[locale]/(auth)/register/page.tsx` — change `router.replace(`/${locale}/dashboard`)` to `router.replace(`/${locale}/setup`)` in the `onSubmit` handler (line ~50); see Dev Notes

- [x] Task 13: Build and type-check verification (AC: 11)
  - [x] 13.1 Run `pnpm build` from `eusolicit-app/frontend/` — both `client` and `admin` apps must exit 0
  - [x] 13.2 Run `pnpm type-check` from `eusolicit-app/frontend/` — zero TypeScript errors across all packages
  - [x] 13.3 Run `pnpm check:i18n --filter client` from `eusolicit-app/frontend/` — must exit 0 (no new i18n keys were added, so this should pass without changes)

## Dev Notes

### Working Directory

All frontend code lives under: `eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run pnpm commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

### Critical Learnings from Stories 3.1–3.8 (MUST APPLY)

1. **`@/*` maps to `./` not `./src/`** — Inside `apps/client`, `@/lib/stores/wizard-store` resolves to `apps/client/lib/stores/wizard-store.ts`. Inside `packages/ui`, always use **relative paths** — the `@/` alias does NOT work inside `packages/ui`.
2. **`next.config.mjs` not `.ts`** — Both apps use `.mjs`. Do NOT create or modify next configs for this story.
3. **Route group `(protected)` is a sibling of `(auth)` inside `app/[locale]/`** — The setup wizard page at `app/[locale]/(protected)/setup/page.tsx` maps to the URL `/{locale}/setup`. No URL segment is added for the `(protected)` folder itself.
4. **`useAuthStore` is imported from `@eusolicit/ui`** — Import: `import { useAuthStore } from "@eusolicit/ui";`. Do NOT create a local auth store.
5. **`useZodForm` and `FormField` imported from `@eusolicit/ui`** — Step 1 text/url fields use these. Steps 2–4 use custom interaction patterns (custom lists, checkboxes not tied to a single RHF field) so they manage state directly in `wizardStore` rather than through `useZodForm`.
6. **Locale prefix in all in-app links** — Every `<Link href>` and `router.replace()` call must use `/${locale}/` prefix.
7. **`wizardStore` lives in `apps/client/lib/stores/wizard-store.ts`** — It is client-app-specific (wizard flow doesn't exist in admin app). Do NOT put it in `packages/ui`.
8. **`logoFile` cannot be persisted** — `File` objects are not JSON-serialisable. Use `FileReader.readAsDataURL()` to convert to base64 string (`logoDataUrl`) which IS serialisable. Store both in state but only persist `logoDataUrl` via `partialize`.
9. **Step components do NOT use `useZodForm` for their form state** — Steps 2–4 manage selections directly in the Zustand store. Step 1 uses `useZodForm` only for the text fields (address, phone, website). Validation of Steps 2–4 happens imperatively before advancing.
10. **`mounted` guard for auth redirect** — The setup page reads `authStore.isAuthenticated` after Zustand hydration. Use the same `mounted` + `useEffect` pattern from S03.08 auth pages to redirect unauthenticated users to `/login`.
11. **Colocated step components** — Step subcomponents live inside `app/[locale]/(protected)/setup/components/`. They are NOT in `packages/ui` (wizard-specific, not reusable across apps). The `components/` subfolder is not a Next.js route — files inside it are not page routes.
12. **No new i18n keys required** — All `wizard.*` keys are already present in both `messages/en.json` and `messages/bg.json` from S03.07. Use `useTranslations("wizard")` and `useTranslations("common")` for all strings.

### Architecture: File Structure Added in S03.09

```
apps/client/
  app/
    [locale]/
      (auth)/
        register/
          page.tsx              ← MODIFIED: redirect to /${locale}/setup after registration
      (protected)/
        setup/                  ← NEW: wizard route
          page.tsx              ← NEW: main wizard orchestrator
          components/           ← NEW: colocated step subcomponents (NOT page routes)
            WizardStepper.tsx   ← NEW: horizontal stepper UI
            Step1CompanyInfo.tsx
            Step2CPVSectors.tsx
            Step3Regions.tsx
            Step4TeamInvites.tsx
  lib/
    stores/
      wizard-store.ts           ← NEW: Zustand wizard state with persist
    schemas/
      wizard.ts                 ← NEW: step1Schema, step2Schema, step3Schema, teamEmailSchema
    data/
      cpv-codes.ts              ← NEW: CPV_CODES static data (~30 top-level categories)
      regions.ts                ← NEW: BULGARIAN_REGIONS (6) + EU_MEMBER_STATES (27)
    api/
      company.ts                ← NEW: updateCompanyProfile() stub
```

### Wizard URL Route

| Page | URL (BG default) |
|------|-----------------|
| Setup Wizard | `/bg/setup` or `/en/setup` |
| Dashboard (redirect after complete) | `/bg/dashboard` or `/en/dashboard` |

### Implementation Code: `lib/stores/wizard-store.ts`

```typescript
// apps/client/lib/stores/wizard-store.ts
"use client";

import { create } from "zustand";
import { persist, devtools } from "zustand/middleware";

export interface WizardStep1 {
  companyName: string;
  eik: string;
  address: string;
  phone: string;
  website: string;
  logoFile: File | null;      // NOT persisted (non-serialisable)
  logoDataUrl: string | null; // base64 from FileReader — persisted
}

export interface WizardStep2 {
  cpvCodes: string[]; // selected CPV code strings (e.g. "03000000")
}

export interface WizardStep3 {
  regions: string[]; // selected region id strings
}

export interface WizardStep4 {
  teamEmails: string[];
}

interface WizardState {
  currentStep: number; // 1–4
  step1: WizardStep1;
  step2: WizardStep2;
  step3: WizardStep3;
  step4: WizardStep4;
  setCurrentStep: (step: number) => void;
  setStep1: (data: Partial<WizardStep1>) => void;
  setStep2: (data: Partial<WizardStep2>) => void;
  setStep3: (data: Partial<WizardStep3>) => void;
  setStep4: (data: Partial<WizardStep4>) => void;
  reset: () => void;
}

const initialStep1: WizardStep1 = {
  companyName: "",
  eik: "",
  address: "",
  phone: "",
  website: "",
  logoFile: null,
  logoDataUrl: null,
};

const initialState = {
  currentStep: 1,
  step1: initialStep1,
  step2: { cpvCodes: [] },
  step3: { regions: [] },
  step4: { teamEmails: [] },
};

export const useWizardStore = create<WizardState>()(
  devtools(
    persist(
      (set) => ({
        ...initialState,
        setCurrentStep: (step) =>
          set({ currentStep: step }, false, "wizard/setCurrentStep"),
        setStep1: (data) =>
          set(
            (state) => ({ step1: { ...state.step1, ...data } }),
            false,
            "wizard/setStep1"
          ),
        setStep2: (data) =>
          set(
            (state) => ({ step2: { ...state.step2, ...data } }),
            false,
            "wizard/setStep2"
          ),
        setStep3: (data) =>
          set(
            (state) => ({ step3: { ...state.step3, ...data } }),
            false,
            "wizard/setStep3"
          ),
        setStep4: (data) =>
          set(
            (state) => ({ step4: { ...state.step4, ...data } }),
            false,
            "wizard/setStep4"
          ),
        reset: () => set(initialState, false, "wizard/reset"),
      }),
      {
        name: "eusolicit-wizard-store",
        partialize: (state) => ({
          currentStep: state.currentStep,
          step1: {
            companyName: state.step1.companyName,
            eik: state.step1.eik,
            address: state.step1.address,
            phone: state.step1.phone,
            website: state.step1.website,
            logoFile: null,          // EXCLUDED — non-serialisable File object
            logoDataUrl: state.step1.logoDataUrl, // INCLUDED — base64 string
          },
          step2: state.step2,
          step3: state.step3,
          step4: state.step4,
        }),
      }
    ),
    { name: "WizardStore" }
  )
);
```

### Implementation Code: `lib/schemas/wizard.ts`

```typescript
// apps/client/lib/schemas/wizard.ts
import { z } from "zod";

// Step 1: Company Info text fields
// Note: companyName and eik are pre-filled from authStore — validated to be non-empty
// but not re-entered by the user. Address is the key new required field.
export const step1Schema = z.object({
  companyName: z.string().min(2, "Company name is required"),
  eik: z.string().regex(/^\d{9}(\d{4})?$/, "Please enter a valid 9 or 13 digit EIK"),
  address: z.string().min(2, "Address is required"),
  phone: z.string().optional().or(z.literal("")),
  website: z
    .string()
    .optional()
    .refine(
      (val) => !val || val === "" || /^https?:\/\/.+/.test(val),
      "Please enter a valid URL (starting with http:// or https://)"
    ),
});

export type Step1Input = z.infer<typeof step1Schema>;

// Step 2: CPV selection — validated imperatively (not via RHF)
// Minimum 1 CPV code required; checked in wizard page before advancing
export const step2Schema = z.object({
  cpvCodes: z.array(z.string()).min(1, "Please select at least one CPV code"),
});

// Step 3: Region selection — validated imperatively
export const step3Schema = z.object({
  regions: z.array(z.string()).min(1, "Please select at least one region"),
});

// Step 4: Team invite email — used for individual email validation on "Add"
export const teamEmailSchema = z.object({
  email: z
    .string()
    .min(1, "Email is required")
    .email("Please enter a valid email address"),
});

export type TeamEmailInput = z.infer<typeof teamEmailSchema>;
```

### Implementation Code: `lib/data/cpv-codes.ts`

```typescript
// apps/client/lib/data/cpv-codes.ts
// Top-level CPV divisions (representative subset for MVP).
// Full CPV taxonomy has ~9,000 codes; this covers the major procurement categories.

export interface CpvCode {
  code: string;
  description: string;
  descriptionBg: string;
}

export const CPV_CODES: CpvCode[] = [
  { code: "03000000", description: "Agricultural, farming, fishing, forestry and related products", descriptionBg: "Земеделие, животновъдство, риболов, горско стопанство" },
  { code: "09000000", description: "Petroleum products, fuel, electricity and other energy sources", descriptionBg: "Нефтопродукти, горива, електричество" },
  { code: "14000000", description: "Mining, basic metals and related products", descriptionBg: "Добивна промишленост, основни метали" },
  { code: "15000000", description: "Food, beverages, tobacco and related products", descriptionBg: "Хранителни продукти, напитки, тютюн" },
  { code: "16000000", description: "Agricultural machinery", descriptionBg: "Земеделска техника" },
  { code: "18000000", description: "Clothing, footwear, luggage articles and accessories", descriptionBg: "Облекло, обувки, куфари и аксесоари" },
  { code: "19000000", description: "Leather and textile fabrics, plastic and rubber materials", descriptionBg: "Кожа, текстил, пластмаса, каучук" },
  { code: "22000000", description: "Printed matter and related products", descriptionBg: "Печатни материали и свързани продукти" },
  { code: "24000000", description: "Chemical products", descriptionBg: "Химически продукти" },
  { code: "30000000", description: "Office and computing machinery, equipment and supplies", descriptionBg: "Офис и компютърна техника" },
  { code: "31000000", description: "Electrical machinery, apparatus, equipment and consumables", descriptionBg: "Електрически машини и апарати" },
  { code: "32000000", description: "Radio, television, communication, telecommunication and related equipment", descriptionBg: "Радио, телевизия, телекомуникационно оборудване" },
  { code: "33000000", description: "Medical equipments, pharmaceuticals and personal care products", descriptionBg: "Медицинско оборудване и фармацевтични продукти" },
  { code: "34000000", description: "Transport equipment and auxiliary products to transportation", descriptionBg: "Транспортни средства и спомагателни продукти" },
  { code: "35000000", description: "Security, fire-fighting, police and defence equipment", descriptionBg: "Охрана, пожарна, полиция и отбрана" },
  { code: "37000000", description: "Musical instruments, sport goods, games, toys, handicraft and art materials", descriptionBg: "Спорт, игри, изкуство и занаяти" },
  { code: "38000000", description: "Laboratory, optical and precision equipments", descriptionBg: "Лабораторно и прецизно оборудване" },
  { code: "39000000", description: "Furniture, household equipment and cleaning products", descriptionBg: "Мебели, домакинско оборудване и почистване" },
  { code: "41000000", description: "Collected and purified water", descriptionBg: "Водоснабдяване и пречистване" },
  { code: "42000000", description: "Industrial machinery", descriptionBg: "Индустриални машини" },
  { code: "43000000", description: "Machinery for mining, quarrying, construction equipment", descriptionBg: "Строителни и минни машини" },
  { code: "44000000", description: "Construction structures and materials", descriptionBg: "Строителни конструкции и материали" },
  { code: "45000000", description: "Construction work", descriptionBg: "Строителство" },
  { code: "48000000", description: "Software package and information systems", descriptionBg: "Софтуер и информационни системи" },
  { code: "50000000", description: "Repair and maintenance services", descriptionBg: "Ремонт и поддръжка" },
  { code: "55000000", description: "Hotel, restaurant and retail trade services", descriptionBg: "Хотелиерство, ресторантьорство и търговия" },
  { code: "60000000", description: "Transport services (excl. refuse transport)", descriptionBg: "Транспортни услуги" },
  { code: "63000000", description: "Supporting and auxiliary transport services; travel agencies services", descriptionBg: "Спедиция, транспортна логистика, туризъм" },
  { code: "71000000", description: "Architectural, construction, engineering and inspection services", descriptionBg: "Архитектурни, строителни и инженерни услуги" },
  { code: "72000000", description: "IT services: consulting, software development, internet and support", descriptionBg: "ИТ услуги: консултиране, разработка, интернет" },
  { code: "73000000", description: "Research and development services and related consultancy services", descriptionBg: "Научноизследователска и развойна дейност" },
  { code: "75000000", description: "Administration, defence and social security services", descriptionBg: "Администрация, отбрана, социална сигурност" },
  { code: "79000000", description: "Business services: law, marketing, consulting, recruitment, printing and security", descriptionBg: "Бизнес услуги: право, маркетинг, консултации" },
  { code: "80000000", description: "Education and training services", descriptionBg: "Образование и обучение" },
  { code: "85000000", description: "Health and social work services", descriptionBg: "Здравеопазване и социални услуги" },
  { code: "90000000", description: "Sewage, refuse, cleaning and environmental services", descriptionBg: "Канализация, отпадъци, почистване и околна среда" },
  { code: "92000000", description: "Recreational, cultural and sporting services", descriptionBg: "Отдих, култура и спорт" },
  { code: "98000000", description: "Other community, social and personal services", descriptionBg: "Други обществени и лични услуги" },
];
```

### Implementation Code: `lib/data/regions.ts`

```typescript
// apps/client/lib/data/regions.ts

export interface Region {
  id: string;
  nameEn: string;
  nameBg: string;
}

export const BULGARIAN_REGIONS: Region[] = [
  { id: "bg-nw", nameEn: "Northwestern", nameBg: "Северозападен" },
  { id: "bg-nc", nameEn: "North Central", nameBg: "Северен централен" },
  { id: "bg-ne", nameEn: "Northeastern", nameBg: "Североизточен" },
  { id: "bg-sw", nameEn: "Southwestern", nameBg: "Югозападен" },
  { id: "bg-sc", nameEn: "South Central", nameBg: "Южен централен" },
  { id: "bg-se", nameEn: "Southeastern", nameBg: "Югоизточен" },
];

export const EU_MEMBER_STATES: Region[] = [
  { id: "eu-at", nameEn: "Austria", nameBg: "Австрия" },
  { id: "eu-be", nameEn: "Belgium", nameBg: "Белгия" },
  { id: "eu-bg", nameEn: "Bulgaria", nameBg: "България" },
  { id: "eu-hr", nameEn: "Croatia", nameBg: "Хърватия" },
  { id: "eu-cy", nameEn: "Cyprus", nameBg: "Кипър" },
  { id: "eu-cz", nameEn: "Czech Republic", nameBg: "Чехия" },
  { id: "eu-dk", nameEn: "Denmark", nameBg: "Дания" },
  { id: "eu-ee", nameEn: "Estonia", nameBg: "Естония" },
  { id: "eu-fi", nameEn: "Finland", nameBg: "Финландия" },
  { id: "eu-fr", nameEn: "France", nameBg: "Франция" },
  { id: "eu-de", nameEn: "Germany", nameBg: "Германия" },
  { id: "eu-gr", nameEn: "Greece", nameBg: "Гърция" },
  { id: "eu-hu", nameEn: "Hungary", nameBg: "Унгария" },
  { id: "eu-ie", nameEn: "Ireland", nameBg: "Ирландия" },
  { id: "eu-it", nameEn: "Italy", nameBg: "Италия" },
  { id: "eu-lv", nameEn: "Latvia", nameBg: "Латвия" },
  { id: "eu-lt", nameEn: "Lithuania", nameBg: "Литва" },
  { id: "eu-lu", nameEn: "Luxembourg", nameBg: "Люксембург" },
  { id: "eu-mt", nameEn: "Malta", nameBg: "Малта" },
  { id: "eu-nl", nameEn: "Netherlands", nameBg: "Нидерландия" },
  { id: "eu-pl", nameEn: "Poland", nameBg: "Полша" },
  { id: "eu-pt", nameEn: "Portugal", nameBg: "Португалия" },
  { id: "eu-ro", nameEn: "Romania", nameBg: "Румъния" },
  { id: "eu-sk", nameEn: "Slovakia", nameBg: "Словакия" },
  { id: "eu-si", nameEn: "Slovenia", nameBg: "Словения" },
  { id: "eu-es", nameEn: "Spain", nameBg: "Испания" },
  { id: "eu-se", nameEn: "Sweden", nameBg: "Швеция" },
];
```

### Implementation Code: `lib/api/company.ts`

```typescript
// apps/client/lib/api/company.ts
// Stub implementation — replace with apiClient calls when E02 company endpoints are deployed.
// The real endpoint is: PATCH /companies/:id/profile

export interface CompanyProfilePayload {
  companyId: string;
  address: string;
  phone?: string;
  website?: string;
  cpvCodes: string[];
  regions: string[];
  teamInvites: string[];
  logoDataUrl?: string | null;
}

const STUB_DELAY_MS = 800;

function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

export async function updateCompanyProfile(payload: CompanyProfilePayload): Promise<void> {
  await delay(STUB_DELAY_MS);
  // TODO: Replace with:
  // await apiClient.patch(`/companies/${payload.companyId}/profile`, payload);
  void payload; // suppress unused variable warning in strict mode
}
```

### Implementation Code: `setup/components/WizardStepper.tsx`

```tsx
// apps/client/app/[locale]/(protected)/setup/components/WizardStepper.tsx
import { Check } from "lucide-react";
import { cn } from "@eusolicit/ui";

interface Step {
  label: string;
  stepNumber: number;
}

interface WizardStepperProps {
  steps: Step[];
  currentStep: number; // 1-based
}

export function WizardStepper({ steps, currentStep }: WizardStepperProps) {
  return (
    <nav aria-label="Setup wizard progress" className="w-full">
      <ol className="flex items-center justify-between">
        {steps.map((step, index) => {
          const isCompleted = step.stepNumber < currentStep;
          const isActive = step.stepNumber === currentStep;
          const isFuture = step.stepNumber > currentStep;
          const isLast = index === steps.length - 1;

          return (
            <li key={step.stepNumber} className="flex items-center flex-1">
              {/* Step indicator */}
              <div className="flex flex-col items-center">
                <div
                  className={cn(
                    "flex h-9 w-9 items-center justify-center rounded-full border-2 text-sm font-semibold transition-colors",
                    isCompleted && "border-indigo-600 bg-indigo-600 text-white",
                    isActive && "border-indigo-600 bg-white text-indigo-600",
                    isFuture && "border-slate-300 bg-white text-slate-400"
                  )}
                  aria-current={isActive ? "step" : undefined}
                >
                  {isCompleted ? (
                    <Check className="h-5 w-5" />
                  ) : (
                    <span>{step.stepNumber}</span>
                  )}
                </div>
                <span
                  className={cn(
                    "mt-1 text-xs font-medium hidden sm:block",
                    isActive && "text-indigo-600",
                    isCompleted && "text-indigo-600",
                    isFuture && "text-slate-400"
                  )}
                >
                  {step.label}
                </span>
              </div>

              {/* Connector line between steps */}
              {!isLast && (
                <div
                  className={cn(
                    "mx-2 h-0.5 flex-1 transition-colors",
                    isCompleted ? "bg-indigo-600" : "bg-slate-200"
                  )}
                />
              )}
            </li>
          );
        })}
      </ol>
    </nav>
  );
}
```

### Implementation Code: `setup/components/Step1CompanyInfo.tsx`

```tsx
// apps/client/app/[locale]/(protected)/setup/components/Step1CompanyInfo.tsx
"use client";

import { useRef } from "react";
import { useTranslations } from "next-intl";
import { useAuthStore, FormField, useZodForm } from "@eusolicit/ui";
import { step1Schema, type Step1Input } from "@/lib/schemas/wizard";
import { useWizardStore } from "@/lib/stores/wizard-store";

interface Step1Props {
  onNext: (data: Step1Input) => void;
}

export function Step1CompanyInfo({ onNext }: Step1Props) {
  const t = useTranslations("wizard");
  const tCommon = useTranslations("common");
  const { user } = useAuthStore();
  const { step1, setStep1 } = useWizardStore();
  const fileInputRef = useRef<HTMLInputElement>(null);

  const form = useZodForm(step1Schema, {
    defaultValues: {
      companyName: step1.companyName || user?.name || "",
      eik: step1.eik || "",
      address: step1.address || "",
      phone: step1.phone || "",
      website: step1.website || "",
    },
  });

  function handleLogoChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;
    setStep1({ logoFile: file });
    const reader = new FileReader();
    reader.onload = (event) => {
      const dataUrl = event.target?.result as string;
      setStep1({ logoDataUrl: dataUrl });
    };
    reader.readAsDataURL(file);
  }

  function handleSubmit(data: Step1Input) {
    // Persist text field values to store before advancing
    setStep1({
      companyName: data.companyName,
      eik: data.eik,
      address: data.address,
      phone: data.phone ?? "",
      website: data.website ?? "",
    });
    onNext(data);
  }

  return (
    <form onSubmit={form.handleSubmit(handleSubmit)} className="space-y-4" noValidate>
      <h2 className="text-lg font-semibold text-slate-900">{t("companyInfo")}</h2>

      <div className="grid grid-cols-2 gap-4">
        <FormField
          control={form.control}
          name="companyName"
          label={t("companyInfo")}
          type="text"
        />
        <FormField
          control={form.control}
          name="eik"
          label="EIK"
          type="text"
        />
      </div>

      <FormField
        control={form.control}
        name="address"
        label={t("companyAddress")}
        type="text"
        placeholder="ул. Витоша 1, София 1000"
      />

      <FormField
        control={form.control}
        name="phone"
        label={t("companyPhone")}
        type="text"
        placeholder="+359 2 000 0000"
      />

      <FormField
        control={form.control}
        name="website"
        label={t("companyWebsite")}
        type="text"
        placeholder="https://example.com"
      />

      {/* Logo upload */}
      <div className="space-y-2">
        <label className="text-sm font-medium text-slate-700">{t("logoUpload")}</label>
        <div className="flex items-center gap-4">
          {step1.logoDataUrl && (
            <img
              src={step1.logoDataUrl}
              alt="Company logo preview"
              className="h-16 w-16 rounded-full object-cover border border-slate-200"
            />
          )}
          <input
            ref={fileInputRef}
            type="file"
            accept="image/*"
            onChange={handleLogoChange}
            className="text-sm text-slate-600 file:mr-3 file:py-1.5 file:px-3 file:rounded-md file:border-0 file:text-sm file:font-medium file:bg-indigo-50 file:text-indigo-600 hover:file:bg-indigo-100"
          />
        </div>
      </div>

      <div className="flex justify-end pt-2">
        <button
          type="submit"
          className="inline-flex items-center rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-indigo-600 focus:ring-offset-2 disabled:opacity-50"
        >
          {tCommon("next")}
        </button>
      </div>
    </form>
  );
}
```

### Implementation Code: `setup/components/Step2CPVSectors.tsx`

```tsx
// apps/client/app/[locale]/(protected)/setup/components/Step2CPVSectors.tsx
"use client";

import { useState } from "react";
import { useTranslations } from "next-intl";
import { X } from "lucide-react";
import { Badge, Input } from "@eusolicit/ui";
import { CPV_CODES } from "@/lib/data/cpv-codes";
import { useWizardStore } from "@/lib/stores/wizard-store";

interface Step2Props {
  onNext: () => void;
  onBack: () => void;
  validationError: string | null;
}

export function Step2CPVSectors({ onNext, onBack, validationError }: Step2Props) {
  const t = useTranslations("wizard");
  const tCommon = useTranslations("common");
  const { step2, setStep2 } = useWizardStore();
  const [search, setSearch] = useState("");

  const filtered = search.trim()
    ? CPV_CODES.filter(
        (c) =>
          c.code.includes(search.trim()) ||
          c.description.toLowerCase().includes(search.trim().toLowerCase()) ||
          c.descriptionBg.toLowerCase().includes(search.trim().toLowerCase())
      )
    : CPV_CODES;

  function addCode(code: string) {
    if (!step2.cpvCodes.includes(code)) {
      setStep2({ cpvCodes: [...step2.cpvCodes, code] });
    }
    setSearch("");
  }

  function removeCode(code: string) {
    setStep2({ cpvCodes: step2.cpvCodes.filter((c) => c !== code) });
  }

  const selectedCodes = CPV_CODES.filter((c) => step2.cpvCodes.includes(c.code));

  return (
    <div className="space-y-4">
      <h2 className="text-lg font-semibold text-slate-900">{t("step2")}</h2>

      {/* Search input */}
      <Input
        type="text"
        placeholder={t("cpvSearch")}
        value={search}
        onChange={(e) => setSearch(e.target.value)}
      />

      {/* Filtered results list */}
      {search.trim() && (
        <ul className="max-h-48 overflow-y-auto rounded-md border border-slate-200 divide-y divide-slate-100">
          {filtered.length === 0 ? (
            <li className="px-3 py-2 text-sm text-slate-400">No results</li>
          ) : (
            filtered.slice(0, 20).map((cpv) => (
              <li key={cpv.code}>
                <button
                  type="button"
                  onClick={() => addCode(cpv.code)}
                  disabled={step2.cpvCodes.includes(cpv.code)}
                  className="w-full px-3 py-2 text-left text-sm hover:bg-indigo-50 disabled:opacity-40 disabled:cursor-not-allowed"
                >
                  <span className="font-mono text-xs text-slate-500 mr-2">{cpv.code}</span>
                  {cpv.description}
                </button>
              </li>
            ))
          )}
        </ul>
      )}

      {/* Selected badges */}
      {selectedCodes.length > 0 && (
        <div className="flex flex-wrap gap-2">
          {selectedCodes.map((cpv) => (
            <Badge
              key={cpv.code}
              variant="secondary"
              className="flex items-center gap-1 pr-1"
            >
              <span className="font-mono text-xs">{cpv.code}</span>
              <button
                type="button"
                onClick={() => removeCode(cpv.code)}
                aria-label={`Remove ${cpv.code}`}
                className="ml-1 rounded hover:bg-slate-200 p-0.5"
              >
                <X className="h-3 w-3" />
              </button>
            </Badge>
          ))}
        </div>
      )}

      {/* Validation error */}
      {validationError && (
        <p className="text-sm text-red-500" role="alert">{validationError}</p>
      )}

      {/* Navigation */}
      <div className="flex justify-between pt-2">
        <button
          type="button"
          onClick={onBack}
          className="inline-flex items-center rounded-md border border-slate-300 bg-white px-4 py-2 text-sm font-medium text-slate-700 hover:bg-slate-50"
        >
          {tCommon("back")}
        </button>
        <button
          type="button"
          onClick={onNext}
          className="inline-flex items-center rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700"
        >
          {tCommon("next")}
        </button>
      </div>
    </div>
  );
}
```

### Implementation Code: `setup/components/Step3Regions.tsx`

```tsx
// apps/client/app/[locale]/(protected)/setup/components/Step3Regions.tsx
"use client";

import { useTranslations } from "next-intl";
import { Checkbox } from "@eusolicit/ui";
import { BULGARIAN_REGIONS, EU_MEMBER_STATES } from "@/lib/data/regions";
import { useWizardStore } from "@/lib/stores/wizard-store";

interface Step3Props {
  onNext: () => void;
  onBack: () => void;
  validationError: string | null;
  locale: string;
}

export function Step3Regions({ onNext, onBack, validationError, locale }: Step3Props) {
  const t = useTranslations("wizard");
  const tCommon = useTranslations("common");
  const { step3, setStep3 } = useWizardStore();

  function toggleRegion(id: string) {
    const current = step3.regions;
    const next = current.includes(id)
      ? current.filter((r) => r !== id)
      : [...current, id];
    setStep3({ regions: next });
  }

  function toggleAll(regionList: { id: string }[], checked: boolean) {
    const ids = regionList.map((r) => r.id);
    if (checked) {
      // Add all that aren't already selected
      const next = Array.from(new Set([...step3.regions, ...ids]));
      setStep3({ regions: next });
    } else {
      // Remove all from this group
      const next = step3.regions.filter((r) => !ids.includes(r));
      setStep3({ regions: next });
    }
  }

  const allBgSelected = BULGARIAN_REGIONS.every((r) => step3.regions.includes(r.id));
  const allEuSelected = EU_MEMBER_STATES.every((r) => step3.regions.includes(r.id));
  const isBg = locale === "bg";

  return (
    <div className="space-y-5">
      <h2 className="text-lg font-semibold text-slate-900">{t("step3")}</h2>

      {/* Bulgarian Planning Regions */}
      <div>
        <div className="flex items-center justify-between mb-2">
          <h3 className="text-sm font-semibold text-slate-700">
            {isBg ? "Планови региони на България" : "Bulgarian Planning Regions"}
          </h3>
          <label className="flex items-center gap-1.5 text-xs text-slate-500 cursor-pointer">
            <Checkbox
              checked={allBgSelected}
              onCheckedChange={(checked) => toggleAll(BULGARIAN_REGIONS, !!checked)}
            />
            {t("selectAll")}
          </label>
        </div>
        <div className="grid grid-cols-2 gap-2">
          {BULGARIAN_REGIONS.map((region) => (
            <label key={region.id} className="flex items-center gap-2 cursor-pointer">
              <Checkbox
                checked={step3.regions.includes(region.id)}
                onCheckedChange={() => toggleRegion(region.id)}
              />
              <span className="text-sm text-slate-700">
                {isBg ? region.nameBg : region.nameEn}
              </span>
            </label>
          ))}
        </div>
      </div>

      {/* EU Member States */}
      <div>
        <div className="flex items-center justify-between mb-2">
          <h3 className="text-sm font-semibold text-slate-700">
            {isBg ? "Държави — членки на ЕС" : "EU Member States"}
          </h3>
          <label className="flex items-center gap-1.5 text-xs text-slate-500 cursor-pointer">
            <Checkbox
              checked={allEuSelected}
              onCheckedChange={(checked) => toggleAll(EU_MEMBER_STATES, !!checked)}
            />
            {t("selectAll")}
          </label>
        </div>
        <div className="grid grid-cols-2 sm:grid-cols-3 gap-2 max-h-64 overflow-y-auto pr-1">
          {EU_MEMBER_STATES.map((region) => (
            <label key={region.id} className="flex items-center gap-2 cursor-pointer">
              <Checkbox
                checked={step3.regions.includes(region.id)}
                onCheckedChange={() => toggleRegion(region.id)}
              />
              <span className="text-sm text-slate-700">
                {isBg ? region.nameBg : region.nameEn}
              </span>
            </label>
          ))}
        </div>
      </div>

      {/* Validation error */}
      {validationError && (
        <p className="text-sm text-red-500" role="alert">{validationError}</p>
      )}

      {/* Navigation */}
      <div className="flex justify-between pt-2">
        <button
          type="button"
          onClick={onBack}
          className="inline-flex items-center rounded-md border border-slate-300 bg-white px-4 py-2 text-sm font-medium text-slate-700 hover:bg-slate-50"
        >
          {tCommon("back")}
        </button>
        <button
          type="button"
          onClick={onNext}
          className="inline-flex items-center rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700"
        >
          {tCommon("next")}
        </button>
      </div>
    </div>
  );
}
```

### Implementation Code: `setup/components/Step4TeamInvites.tsx`

```tsx
// apps/client/app/[locale]/(protected)/setup/components/Step4TeamInvites.tsx
"use client";

import { useState } from "react";
import { useTranslations } from "next-intl";
import { X, Loader2 } from "lucide-react";
import { Input } from "@eusolicit/ui";
import { teamEmailSchema } from "@/lib/schemas/wizard";
import { useWizardStore } from "@/lib/stores/wizard-store";

interface Step4Props {
  onComplete: () => void;
  onBack: () => void;
  isSubmitting: boolean;
}

export function Step4TeamInvites({ onComplete, onBack, isSubmitting }: Step4Props) {
  const t = useTranslations("wizard");
  const tCommon = useTranslations("common");
  const { step4, setStep4 } = useWizardStore();
  const [emailInput, setEmailInput] = useState("");
  const [emailError, setEmailError] = useState<string | null>(null);

  function addEmail() {
    const parsed = teamEmailSchema.safeParse({ email: emailInput.trim() });
    if (!parsed.success) {
      setEmailError(parsed.error.errors[0]?.message ?? "Invalid email");
      return;
    }
    const email = parsed.data.email;
    if (!step4.teamEmails.includes(email)) {
      setStep4({ teamEmails: [...step4.teamEmails, email] });
    }
    setEmailInput("");
    setEmailError(null);
  }

  function removeEmail(email: string) {
    setStep4({ teamEmails: step4.teamEmails.filter((e) => e !== email) });
  }

  function handleKeyDown(e: React.KeyboardEvent<HTMLInputElement>) {
    if (e.key === "Enter") {
      e.preventDefault();
      addEmail();
    }
  }

  return (
    <div className="space-y-4">
      <h2 className="text-lg font-semibold text-slate-900">{t("step4")}</h2>

      {/* Email input row */}
      <div className="flex gap-2">
        <Input
          type="email"
          placeholder={t("inviteEmail")}
          value={emailInput}
          onChange={(e) => {
            setEmailInput(e.target.value);
            setEmailError(null);
          }}
          onKeyDown={handleKeyDown}
          disabled={isSubmitting}
          className="flex-1"
        />
        <button
          type="button"
          onClick={addEmail}
          disabled={isSubmitting}
          className="inline-flex items-center rounded-md bg-indigo-600 px-3 py-2 text-sm font-medium text-white hover:bg-indigo-700 disabled:opacity-50"
        >
          {t("addInvite")}
        </button>
      </div>
      {emailError && (
        <p className="text-sm text-red-500" role="alert">{emailError}</p>
      )}

      {/* Added emails list */}
      {step4.teamEmails.length > 0 && (
        <ul className="space-y-1">
          {step4.teamEmails.map((email) => (
            <li
              key={email}
              className="flex items-center justify-between rounded-md border border-slate-200 px-3 py-2 text-sm"
            >
              <span className="text-slate-700">{email}</span>
              <button
                type="button"
                onClick={() => removeEmail(email)}
                disabled={isSubmitting}
                aria-label={`Remove ${email}`}
                className="text-slate-400 hover:text-red-500 disabled:opacity-50"
              >
                <X className="h-4 w-4" />
              </button>
            </li>
          ))}
        </ul>
      )}

      {/* Navigation */}
      <div className="flex justify-between pt-2">
        <button
          type="button"
          onClick={onBack}
          disabled={isSubmitting}
          className="inline-flex items-center rounded-md border border-slate-300 bg-white px-4 py-2 text-sm font-medium text-slate-700 hover:bg-slate-50 disabled:opacity-50"
        >
          {tCommon("back")}
        </button>
        <button
          type="button"
          onClick={onComplete}
          disabled={isSubmitting}
          className="inline-flex items-center rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700 disabled:opacity-50"
        >
          {isSubmitting && <Loader2 className="animate-spin mr-2 h-4 w-4" />}
          {t("completeSetup")}
        </button>
      </div>
    </div>
  );
}
```

### Implementation Code: `app/[locale]/(protected)/setup/page.tsx`

```tsx
// apps/client/app/[locale]/(protected)/setup/page.tsx
"use client";

import { useState, useEffect } from "react";
import { useRouter } from "next/navigation";
import { useTranslations } from "next-intl";
import { useAuthStore } from "@eusolicit/ui";
import { useWizardStore } from "@/lib/stores/wizard-store";
import { updateCompanyProfile } from "@/lib/api/company";
import { step2Schema, step3Schema } from "@/lib/schemas/wizard";
import type { Step1Input } from "@/lib/schemas/wizard";
import { WizardStepper } from "./components/WizardStepper";
import { Step1CompanyInfo } from "./components/Step1CompanyInfo";
import { Step2CPVSectors } from "./components/Step2CPVSectors";
import { Step3Regions } from "./components/Step3Regions";
import { Step4TeamInvites } from "./components/Step4TeamInvites";

export default function SetupPage({
  params: { locale },
}: {
  params: { locale: string };
}) {
  const t = useTranslations("wizard");
  const router = useRouter();
  const { isAuthenticated, user } = useAuthStore();
  const { currentStep, step1, step2, step3, step4, setCurrentStep, reset } =
    useWizardStore();

  const [mounted, setMounted] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [step2Error, setStep2Error] = useState<string | null>(null);
  const [step3Error, setStep3Error] = useState<string | null>(null);

  useEffect(() => {
    setMounted(true);
  }, []);

  // Auth redirect — if not authenticated, go to login
  useEffect(() => {
    if (mounted && !isAuthenticated) {
      router.replace(`/${locale}/login`);
    }
  }, [mounted, isAuthenticated, locale, router]);

  // Suppress render during Zustand hydration
  if (!mounted) return null;

  const steps = [
    { stepNumber: 1, label: t("step1") },
    { stepNumber: 2, label: t("step2") },
    { stepNumber: 3, label: t("step3") },
    { stepNumber: 4, label: t("step4") },
  ];

  // Step 1 → 2
  function handleStep1Next(_data: Step1Input) {
    setCurrentStep(2);
  }

  // Step 2 → 3 (validate CPV selection)
  function handleStep2Next() {
    const result = step2Schema.safeParse({ cpvCodes: step2.cpvCodes });
    if (!result.success) {
      setStep2Error(t("minOneCpv"));
      return;
    }
    setStep2Error(null);
    setCurrentStep(3);
  }

  // Step 3 → 4 (validate regions selection)
  function handleStep3Next() {
    const result = step3Schema.safeParse({ regions: step3.regions });
    if (!result.success) {
      setStep3Error(t("minOneRegion"));
      return;
    }
    setStep3Error(null);
    setCurrentStep(4);
  }

  // Step 4 → Complete Setup
  async function handleComplete() {
    setIsSubmitting(true);
    try {
      await updateCompanyProfile({
        companyId: user?.companyId ?? "stub-company-id",
        address: step1.address,
        phone: step1.phone || undefined,
        website: step1.website || undefined,
        cpvCodes: step2.cpvCodes,
        regions: step3.regions,
        teamInvites: step4.teamEmails,
        logoDataUrl: step1.logoDataUrl,
      });
      reset(); // Clear wizard state and remove from localStorage
      router.replace(`/${locale}/dashboard`);
    } catch {
      // In stub mode this won't fail; add toast error handling when real API is integrated
      setIsSubmitting(false);
    }
  }

  return (
    <div className="max-w-2xl mx-auto py-8 px-4">
      {/* Page title */}
      <h1 className="text-2xl font-bold text-slate-900 mb-2">{t("title")}</h1>

      {/* Stepper */}
      <div className="mb-8">
        <WizardStepper steps={steps} currentStep={currentStep} />
      </div>

      {/* Step content card */}
      <div className="rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
        {currentStep === 1 && (
          <Step1CompanyInfo onNext={handleStep1Next} />
        )}
        {currentStep === 2 && (
          <Step2CPVSectors
            onNext={handleStep2Next}
            onBack={() => setCurrentStep(1)}
            validationError={step2Error}
          />
        )}
        {currentStep === 3 && (
          <Step3Regions
            onNext={handleStep3Next}
            onBack={() => setCurrentStep(2)}
            validationError={step3Error}
            locale={locale}
          />
        )}
        {currentStep === 4 && (
          <Step4TeamInvites
            onComplete={handleComplete}
            onBack={() => setCurrentStep(3)}
            isSubmitting={isSubmitting}
          />
        )}
      </div>
    </div>
  );
}
```

### Modification: `register/page.tsx` — Change Redirect to `/setup`

In `apps/client/app/[locale]/(auth)/register/page.tsx`, locate the `onSubmit` handler and change the redirect:

```diff
- router.replace(`/${locale}/dashboard`);
+ router.replace(`/${locale}/setup`);
```

This is the only line that needs to change in the register page (approximately line 50 in the current file).

### Test Expectations (from Epic 3 Test Design)

The following test IDs from `test-design-epic-03.md` directly cover Story 3.9:

**P1 (High Priority) — Must Pass Before Demo:**
- **E03-P1-008**: Full wizard Step 1 → Step 4 completion — each "Next" validates current step; "Complete Setup" submits and redirects to `/dashboard`. Test: fill all 4 steps with valid data; assert stepper advances at each step; assert POST call is made (mocked); assert redirect to `/dashboard`.
- **E03-P1-009**: Wizard state survives page reload mid-wizard (after Step 2 filled): reload page; assert Step 2 CPV selections are pre-populated from persisted Zustand `wizardStore`. Test fixture: `wizardStateFactory` creates partial step 2 state; navigate to `/setup`; reload; assert CPV badges still present.

**P2 (Medium Priority) — Cover in Sprint:**
- **E03-P2-013**: Step 2 CPV search — entering a search term filters the list; clicking "Next" with zero CPV codes selected shows the `wizard.minOneCpv` validation error.
- **E03-P2-014**: Step 3 regions — "Select all" for Bulgarian regions checks all 6; "Select all" for EU member states checks all 27; clicking "Next" with zero regions selected shows the `wizard.minOneRegion` error.
- **E03-P2-015**: Step 4 team invites — add 2 emails, remove 1, assert 1 remains; click "Complete Setup" with zero emails — no validation error (step is optional).

**Related Risk Mitigation (E03-R-007):**
The wizard uses Zustand `persist` middleware with `partialize` to exclude the non-serialisable `logoFile`. If localStorage serialisation fails (corrupt data), the store falls back to `initialState`. The ATDD test for E03-P1-009 validates this persistence behaviour.

### Data-testid Attributes for E2E Tests

Add these `data-testid` attributes to enable Playwright selectors:

| Element | `data-testid` |
|---------|--------------|
| Wizard page root `<div>` | `"wizard-page"` |
| `WizardStepper` `<nav>` | `"wizard-stepper"` |
| Step 1 form | `"wizard-step-1"` |
| Step 2 container | `"wizard-step-2"` |
| Step 3 container | `"wizard-step-3"` |
| Step 4 container | `"wizard-step-4"` |
| CPV search input | `"cpv-search-input"` |
| CPV selected badges container | `"cpv-selected-badges"` |
| Regions validation error | `"regions-error"` |
| CPV validation error | `"cpv-error"` |
| "Complete Setup" button | `"complete-setup-button"` |
| Team email input | `"team-email-input"` |
| Team email list | `"team-email-list"` |

### Senior Developer Review

**Review Date:** 2026-04-08
**Verdict:** Approve
**Tests:** 48/48 pass | Type-check: pass | Build: pass (both apps)

#### Round 1 Findings (all resolved)

All 7 patch items from the initial review have been verified as applied in the codebase:

- [x] [Review][Patch][RESOLVED] **AC2 — Logo preview size 128x128** — `h-32 w-32` confirmed in Step1CompanyInfo.tsx
- [x] [Review][Patch][RESOLVED] **AC8 — localStorage key removal** — `localStorage.removeItem("eusolicit-wizard-store")` confirmed in handleComplete before reset()
- [x] [Review][Patch][RESOLVED] **Logo upload file size guard** — 2MB limit with `MAX_SIZE` check and `reader.onerror` handler confirmed
- [x] [Review][Patch][RESOLVED] **Error feedback on completion failure** — `submitError` state and rendered `<p role="alert">` confirmed in setup/page.tsx
- [x] [Review][Patch][RESOLVED] **Out-of-range currentStep** — `Math.max(1, Math.min(4, step))` in setCurrentStep + fallback render for currentStep < 1 or > 4 confirmed
- [x] [Review][Patch][RESOLVED] **Double-click guard** — `if (isSubmitting) return;` guard at top of handleComplete confirmed
- [x] [Review][Patch][RESOLVED] **isSubmitting reset in finally block** — `finally { setIsSubmitting(false); }` confirmed

#### Round 2 Adversarial Review (approval pass)

**Acceptance Criteria Audit — 11/11 pass:**
- AC1: Wizard page at correct route, WizardStepper renders 4 steps with check/indigo/slate styling
- AC2: Step 1 fields pre-filled from authStore, logo upload with 128x128 rounded preview via FileReader.readAsDataURL
- AC3: CPV search filters by code/description/descriptionBg, badge selection with dismiss, min-1 validation with t("minOneCpv")
- AC4: Two region sections (6 BG + 27 EU), select-all toggles, min-1 validation with t("minOneRegion")
- AC5: Email add/validate via Zod/remove, Complete Setup always enabled regardless of email count
- AC6: Back without validation, Next validates schema, Complete Setup shows Loader2 spinner, submits to updateCompanyProfile(), reset() + redirect to dashboard
- AC7: Zustand persist with name "eusolicit-wizard-store", partialize excludes logoFile, logoDataUrl and all step data persisted
- AC8: localStorage.removeItem + reset() + router.replace to dashboard
- AC9: Register page onSubmit redirects to /${locale}/setup
- AC10: All UI strings use useTranslations("wizard"/"common"), all 17 wizard + 2 common keys verified in en.json and bg.json
- AC11: pnpm type-check 4/4 tasks pass, pnpm build 2/2 tasks pass, /[locale]/setup route in build output

**Architecture Alignment:** Wizard-specific components correctly colocated in setup/components/, not in shared packages/ui. Zustand store in apps/client/lib/stores/ (client-only). Import paths use @/ alias correctly. All data-testid attributes present for E2E tests (13/13).

**Code Quality:** TypeScript types enforced throughout. Proper error boundaries. Accessibility: aria-label on stepper nav, aria-current on active step, role="alert" on all error messages, aria-label on remove buttons.

#### Deferred Items (carried forward, not blocking)

- [x] [Review][Defer] Auth hydration race — scoped to S03.12 AuthGuard
- [x] [Review][Defer] Multi-tab wizard sync — Zustand persist doesn't cross-tab sync
- [x] [Review][Defer] Stub companyId fallback — stub-only code
- [x] [Review][Defer] Case-sensitive email dedup — defer to production hardening
- [x] [Review][Defer] Next.js 15 async params — future migration concern

### Known Limitations and Future Work

1. **Logo upload**: `logoDataUrl` (base64) stored in `localStorage` can be large for high-resolution images. For production, replace with an S3/CDN upload that returns a URL — store only the URL in `wizardStore`.
2. **CPV taxonomy**: `cpv-codes.ts` contains ~38 top-level divisions. The full CPV taxonomy (~9,000 codes) will be served from the backend as a paginated search endpoint (E04+ scope). The search + select UI in Step 2 is designed to work with either a static list or an API-backed search — simply replace `CPV_CODES` import with a `useQuery` call.
3. **Team invites**: Step 4 collects emails but does not send invitations. Real email sending is in E07 (Notification Service). The stub records the intent in the company profile payload.
4. **EIK pre-fill**: The `user.name` field from `authStore` contains the display name (e.g. "Demo User"), not the company name. In production with E02 APIs, `authStore.user.companyId` allows fetching the company record to pre-fill both `companyName` and `eik`. For the stub, the user fills them in manually or they default to empty.
5. **`(protected)` AuthGuard**: Full route guard enforcement comes in S03.12. The setup page has its own `mounted + !isAuthenticated → redirect` guard as a stopgap.
