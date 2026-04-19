# Story 8.12: Pricing Page & Tier Comparison UI

Status: review

## Story

As a **prospective customer**,
I want a **public-facing pricing page at `/pricing`** that shows a clear side-by-side tier comparison table for Free, Starter, Professional, and Enterprise plans with feature limits and CTA buttons,
so that I can understand what each tier offers and start a trial or contact sales without needing an account.

## Acceptance Criteria

1. **Route & public access** — A Next.js page exists at `app/[locale]/pricing/page.tsx`. It is a server component that renders `<PricingPage />` (a `"use client"` client component). The route is **not** behind the `(protected)` auth guard — unauthenticated users must be able to access `/pricing` directly. The locale layout (`[locale]/layout.tsx`) provides QueryProvider and i18n.

2. **Backend tier endpoint** — `GET /api/v1/billing/tiers` is a public unauthenticated endpoint in `client-api` that returns a JSON array of all four `TierAccessPolicy` rows ordered `free → starter → professional → enterprise`. No `Authorization` header required. Response is a list of objects matching `TierAccessPolicyDTO` fields: `id`, `tier`, `max_regions`, `max_cpv_sectors`, `max_budget_threshold`, `ai_summaries_limit`, `proposal_drafts_limit`, `compliance_checks_limit`, `max_team_members`, `calendar_sync`, `api_access`, `whitelabel`.

3. **Tier comparison grid** — `<PricingPage />` renders a responsive four-column card grid (`data-testid="pricing-grid"`) with one card per tier: `data-testid="pricing-tier-free"`, `data-testid="pricing-tier-starter"`, `data-testid="pricing-tier-professional"`, `data-testid="pricing-tier-enterprise"`. On mobile (< `md` breakpoint) cards stack vertically; at ≥ `md` they display in a 2-column grid; at ≥ `lg` they display in a 4-column grid.

4. **Monthly prices** — Each tier card displays a monthly price above the feature list. Prices are hardcoded frontend constants (not stored in DB): Free = €0/mo, Starter = €49/mo, Professional = €149/mo, Enterprise = "Custom". Each price element has `data-testid="pricing-price-{tier}"` (e.g., `pricing-price-free`).

5. **Feature matrix** — Each tier card lists the following features with the correct value sourced from the API response. A value of `-1` is displayed as "Unlimited". Feature rows have `data-testid="pricing-feature-{tier}-{feature}"`:
   - `max_regions` — "Regions"
   - `max_cpv_sectors` — "CPV Sectors"
   - `ai_summaries_limit` — "AI Summaries / month"
   - `proposal_drafts_limit` — "Proposal Drafts / month"
   - `compliance_checks_limit` — "Compliance Checks / month"
   - `max_team_members` — "Team Members"
   - `calendar_sync` — boolean, shown as ✓ / ✗ icon
   - `api_access` — boolean, shown as ✓ / ✗ icon
   - `whitelabel` — boolean, shown as ✓ / ✗ icon

6. **Most Popular badge** — The Professional tier card has a `<Badge data-testid="pricing-badge-popular">` displaying "Most Popular" and renders with a highlighted border (`ring-2 ring-indigo-500` or equivalent) to visually distinguish it from other cards.

7. **CTA buttons** — Each tier card has a CTA button:
   - Free: `data-testid="pricing-cta-free"`, label "Sign Up Free", links to `/{locale}/register`
   - Starter: `data-testid="pricing-cta-starter"`, label "Get Started", links to `/{locale}/register?plan=starter`
   - Professional: `data-testid="pricing-cta-professional"`, label "Start Free Trial", links to `/{locale}/register?plan=professional` (primary `variant="default"`)
   - Enterprise: `data-testid="pricing-cta-enterprise"`, label "Contact Sales", links to `mailto:sales@eusolicit.com` (opens in new tab)

8. **Loading state** — While the tier data is fetching, render four `<SkeletonCard />` placeholders in the grid (`data-testid="pricing-loading"`).

9. **Error state** — If the API call fails, render an `<EmptyState>` component with title "Could not load pricing" and a "Retry" button that calls `refetch()`. Container has `data-testid="pricing-error"`.

10. **Page hero** — Above the grid, render a hero section (`data-testid="pricing-hero"`) containing:
    - An `<h1>` with `data-testid="pricing-heading"` — uses `t("billing.pricing.heading")`
    - A subtitle `<p>` with `data-testid="pricing-subheading"` — uses `t("billing.pricing.subheading")`

11. **i18n** — All UI strings use `useTranslations("billing")`. New keys are added under `billing.pricing.*` in both `messages/en.json` and `messages/bg.json` (see Dev Notes for full key list). No hardcoded English strings in JSX.

12. **API module** — `lib/api/billing.ts` exports: TypeScript interface `TierPolicy`, constant `TIER_PRICES` (per-tier monthly price labels), and async function `getTiers(): Promise<TierPolicy[]>` using `apiClient.get("/api/v1/billing/tiers")`. `lib/queries/use-billing.ts` exports `useTiers()` using `useQuery` with `queryKey: ["billing", "tiers"]` and `staleTime: 300_000` (5 min — pricing data changes rarely).

13. **ATDD tests** — Test file `__tests__/pricing-page-s8-12.test.ts` covers: file structure (page shell, component, API module, query hook), `data-testid` attribute presence, i18n key completeness for both locales, API module exports (`getTiers`, `TIER_PRICES`, `TierPolicy`), backend endpoint declaration in `billing.py`. All tests must be GREEN after implementation.

## Tasks / Subtasks

### Backend — Public Tier Endpoint (client-api)

- [x] **Task 1 — Add `GET /api/v1/billing/tiers` endpoint** (AC: 2)
  - [x] 1.1 Open `eusolicit-app/services/client-api/src/client_api/api/v1/billing.py`
  - [x] 1.2 Add the endpoint immediately after the file-level module-docstring update and before the existing `@router.post("/webhooks/stripe")` endpoint (or at the end of the router block — whichever is cleaner). Add a comment `# GET /billing/tiers — Story 8-12 AC2`:
    ```python
    # ---------------------------------------------------------------------------
    # GET /billing/tiers — Story 8-12 AC2: public tier policies for pricing page
    # ---------------------------------------------------------------------------

    class TierPolicyResponse(BaseModel):
        """Public representation of a tier access policy row.
        
        Returned by GET /billing/tiers — no authentication required.
        -1 sentinel means 'unlimited' for all integer limit fields.
        """
        id: str
        tier: str
        max_regions: int
        max_cpv_sectors: int
        max_budget_threshold: int
        ai_summaries_limit: int
        proposal_drafts_limit: int
        compliance_checks_limit: int
        max_team_members: int
        calendar_sync: bool
        api_access: bool
        whitelabel: bool


    _TIER_ORDER = {"free": 0, "starter": 1, "professional": 2, "enterprise": 3}


    @router.get("/tiers", status_code=200)
    async def get_tier_policies() -> list[TierPolicyResponse]:
        """Return all tier access policies for the public pricing page.

        No authentication required — this is a public endpoint accessible
        without a JWT token (pricing page is unauthenticated).

        Returns policies ordered: free → starter → professional → enterprise.
        """
        async with _billing_session() as session:
            result = await session.execute(select(TierAccessPolicy))
            policies = result.scalars().all()

        ordered = sorted(policies, key=lambda p: _TIER_ORDER.get(p.tier, 99))
        return [
            TierPolicyResponse(
                id=str(p.id),
                tier=p.tier,
                max_regions=p.max_regions,
                max_cpv_sectors=p.max_cpv_sectors,
                max_budget_threshold=p.max_budget_threshold,
                ai_summaries_limit=p.ai_summaries_limit,
                proposal_drafts_limit=p.proposal_drafts_limit,
                compliance_checks_limit=p.compliance_checks_limit,
                max_team_members=p.max_team_members,
                calendar_sync=p.calendar_sync,
                api_access=p.api_access,
                whitelabel=p.whitelabel,
            )
            for p in ordered
        ]
    ```
  - [x] 1.3 Update the module docstring at the top of `billing.py` to add the new endpoint:
    ```python
    #   GET  /billing/tiers            — Public tier comparison data for pricing page (Story 8-12)
    ```
  - [x] 1.4 Verify `TierAccessPolicy` is already imported (it is — `from client_api.models.tier_access_policy import TierAccessPolicy`). Do NOT add a duplicate import.

### Frontend — API Module & React Query Hook

- [x] **Task 2 — Create `lib/api/billing.ts`** (AC: 12)
  - [x] 2.1 Create `eusolicit-app/frontend/apps/client/lib/api/billing.ts`:
    ```typescript
    // apps/client/lib/api/billing.ts — Story 8-12: Billing API module
    // Provides tier data fetching for the public pricing page.
    // No "use client" directive — usable in server and client contexts.

    import { apiClient } from "@eusolicit/ui";

    // ─── Types ─────────────────────────────────────────────────────────────────

    export type TierName = "free" | "starter" | "professional" | "enterprise";

    export interface TierPolicy {
      id: string;
      tier: TierName;
      max_regions: number;           // -1 = unlimited
      max_cpv_sectors: number;       // -1 = unlimited
      max_budget_threshold: number;  // -1 = unlimited (EUR)
      ai_summaries_limit: number;    // -1 = unlimited
      proposal_drafts_limit: number; // -1 = unlimited
      compliance_checks_limit: number; // -1 = unlimited
      max_team_members: number;      // -1 = unlimited
      calendar_sync: boolean;
      api_access: boolean;
      whitelabel: boolean;
    }

    // ─── Pricing constants (not stored in DB — product decision) ───────────────

    export const TIER_PRICES: Record<TierName, { label: string; monthly: number | null }> = {
      free:         { label: "€0",     monthly: 0   },
      starter:      { label: "€49",    monthly: 49  },
      professional: { label: "€149",   monthly: 149 },
      enterprise:   { label: "Custom", monthly: null },
    };

    export const TIER_ORDER: TierName[] = ["free", "starter", "professional", "enterprise"];

    // ─── API functions ─────────────────────────────────────────────────────────

    export async function getTiers(): Promise<TierPolicy[]> {
      const response = await apiClient.get<TierPolicy[]>("/api/v1/billing/tiers");
      return response.data;
    }
    ```

- [x] **Task 3 — Create `lib/queries/use-billing.ts`** (AC: 12)
  - [x] 3.1 Create `eusolicit-app/frontend/apps/client/lib/queries/use-billing.ts`:
    ```typescript
    "use client";

    import { useQuery } from "@tanstack/react-query";
    import { getTiers } from "@/lib/api/billing";

    /**
     * React Query hook for fetching tier access policies from the public
     * GET /api/v1/billing/tiers endpoint.
     *
     * Stale time: 5 minutes — pricing data changes rarely.
     * No authentication required (public endpoint).
     */
    export function useTiers() {
      return useQuery({
        queryKey: ["billing", "tiers"],
        queryFn: getTiers,
        staleTime: 300_000, // 5 min
        retry: 2,
      });
    }
    ```

### Frontend — i18n Keys

- [x] **Task 4 — Add pricing i18n keys** (AC: 11)
  - [x] 4.1 Open `eusolicit-app/frontend/apps/client/messages/en.json`
  - [x] 4.2 Add the following keys nested under the existing `"billing"` object (do NOT replace existing billing keys — merge them):
    ```json
    "pricing": {
      "heading": "Simple, transparent pricing",
      "subheading": "Start free. Upgrade when you're ready. No hidden fees.",
      "mostPopular": "Most Popular",
      "perMonth": "/month",
      "perMonthCustom": "Custom pricing",
      "signUpFree": "Sign Up Free",
      "getStarted": "Get Started",
      "startFreeTrial": "Start Free Trial",
      "contactSales": "Contact Sales",
      "featureRegions": "Regions",
      "featureCpvSectors": "CPV Sectors",
      "featureAiSummaries": "AI Summaries / month",
      "featureProposalDrafts": "Proposal Drafts / month",
      "featureComplianceChecks": "Compliance Checks / month",
      "featureTeamMembers": "Team Members",
      "featureCalendarSync": "Calendar Sync",
      "featureApiAccess": "API Access",
      "featureWhitelabel": "Whitelabel",
      "unlimited": "Unlimited",
      "included": "Included",
      "notIncluded": "Not included",
      "errorHeading": "Could not load pricing",
      "retry": "Retry"
    }
    ```
  - [x] 4.3 Open `eusolicit-app/frontend/apps/client/messages/bg.json`
  - [x] 4.4 Add matching Bulgarian translations nested under the existing `"billing"` object:
    ```json
    "pricing": {
      "heading": "Прозрачно ценообразуване",
      "subheading": "Започнете безплатно. Надградете когато сте готови. Без скрити такси.",
      "mostPopular": "Най-популярен",
      "perMonth": "/месец",
      "perMonthCustom": "Индивидуална цена",
      "signUpFree": "Регистрирай се безплатно",
      "getStarted": "Започнете",
      "startFreeTrial": "Започнете безплатен пробен период",
      "contactSales": "Свържете се с продажбите",
      "featureRegions": "Региони",
      "featureCpvSectors": "CPV Сектори",
      "featureAiSummaries": "AI Резюмета / месец",
      "featureProposalDrafts": "Чернови на оферти / месец",
      "featureComplianceChecks": "Проверки за съответствие / месец",
      "featureTeamMembers": "Членове на екипа",
      "featureCalendarSync": "Синхронизация с календар",
      "featureApiAccess": "API достъп",
      "featureWhitelabel": "Бял етикет",
      "unlimited": "Неограничен",
      "included": "Включено",
      "notIncluded": "Не е включено",
      "errorHeading": "Ценообразуването не може да се зареди",
      "retry": "Опитайте отново"
    }
    ```
  - [x] 4.5 Verify both files remain valid JSON after editing (use `python3 -m json.tool` or similar).

### Frontend — Pricing Page Components

- [x] **Task 5 — Create `PricingPage` client component** (AC: 3–11)
  - [x] 5.1 Create directory `eusolicit-app/frontend/apps/client/app/[locale]/pricing/components/`
  - [x] 5.2 Create `eusolicit-app/frontend/apps/client/app/[locale]/pricing/components/PricingPage.tsx`:

    ```typescript
    "use client";

    import { useTranslations } from "next-intl";
    import Link from "next/link";
    import { useParams } from "next/navigation";
    import { CheckCircle2, XCircle } from "lucide-react";
    import {
      Badge,
      Button,
      SkeletonCard,
      EmptyState,
      cn,
    } from "@eusolicit/ui";
    import { useTiers } from "@/lib/queries/use-billing";
    import { TIER_PRICES, TIER_ORDER, type TierName, type TierPolicy } from "@/lib/api/billing";

    // ─── Helper: display integer limit (-1 = "Unlimited") ────────────────────

    function formatLimit(value: number, unlimitedLabel: string): string {
      return value === -1 ? unlimitedLabel : value.toLocaleString();
    }

    // ─── Helper: boolean feature indicator ───────────────────────────────────

    function BoolFeature({ value, included, notIncluded }: {
      value: boolean;
      included: string;
      notIncluded: string;
    }) {
      return value ? (
        <span className="flex items-center gap-1 text-green-600">
          <CheckCircle2 className="h-4 w-4" aria-hidden="true" />
          <span className="sr-only">{included}</span>
        </span>
      ) : (
        <span className="flex items-center gap-1 text-slate-300">
          <XCircle className="h-4 w-4" aria-hidden="true" />
          <span className="sr-only">{notIncluded}</span>
        </span>
      );
    }

    // ─── Tier Card ────────────────────────────────────────────────────────────

    function TierCard({
      policy,
      isPopular,
      locale,
      t,
    }: {
      policy: TierPolicy;
      isPopular: boolean;
      locale: string;
      t: ReturnType<typeof useTranslations<"billing">>;
    }) {
      const tier = policy.tier as TierName;
      const price = TIER_PRICES[tier];
      const unlimited = t("pricing.unlimited");
      const included = t("pricing.included");
      const notIncluded = t("pricing.notIncluded");

      // CTA config per tier
      const ctaConfig: Record<TierName, { label: string; testId: string; href: string; external?: boolean }> = {
        free:         { label: t("pricing.signUpFree"),     testId: "pricing-cta-free",         href: `/${locale}/register` },
        starter:      { label: t("pricing.getStarted"),     testId: "pricing-cta-starter",      href: `/${locale}/register?plan=starter` },
        professional: { label: t("pricing.startFreeTrial"), testId: "pricing-cta-professional", href: `/${locale}/register?plan=professional` },
        enterprise:   { label: t("pricing.contactSales"),   testId: "pricing-cta-enterprise",   href: "mailto:sales@eusolicit.com", external: true },
      };
      const cta = ctaConfig[tier];

      return (
        <div
          data-testid={`pricing-tier-${tier}`}
          className={cn(
            "relative flex flex-col rounded-2xl border bg-white p-6 shadow-sm transition-shadow hover:shadow-md",
            isPopular && "ring-2 ring-indigo-500"
          )}
        >
          {/* Most Popular badge */}
          {isPopular && (
            <Badge
              data-testid="pricing-badge-popular"
              className="absolute -top-3 left-1/2 -translate-x-1/2 bg-indigo-600 text-white"
            >
              {t("pricing.mostPopular")}
            </Badge>
          )}

          {/* Tier name */}
          <h2 className="text-lg font-semibold capitalize text-slate-900">
            {t(`tier${tier.charAt(0).toUpperCase()}${tier.slice(1)}` as `tier${Capitalize<TierName>}`)}
          </h2>

          {/* Monthly price */}
          <div
            data-testid={`pricing-price-${tier}`}
            className="mt-4 flex items-baseline gap-1"
          >
            <span className="text-4xl font-bold text-slate-900">{price.label}</span>
            {price.monthly !== null && (
              <span className="text-sm text-slate-500">{t("pricing.perMonth")}</span>
            )}
            {price.monthly === null && (
              <span className="text-sm text-slate-500">{t("pricing.perMonthCustom")}</span>
            )}
          </div>

          {/* CTA button */}
          <div className="mt-6">
            {cta.external ? (
              <a
                href={cta.href}
                target="_blank"
                rel="noopener noreferrer"
                data-testid={cta.testId}
              >
                <Button variant="outline" className="w-full">
                  {cta.label}
                </Button>
              </a>
            ) : (
              <Link href={cta.href} data-testid={cta.testId}>
                <Button
                  variant={tier === "professional" ? "default" : "outline"}
                  className="w-full"
                >
                  {cta.label}
                </Button>
              </Link>
            )}
          </div>

          {/* Feature list */}
          <ul className="mt-6 space-y-3 text-sm text-slate-700">
            <li data-testid={`pricing-feature-${tier}-max_regions`} className="flex items-center justify-between">
              <span>{t("pricing.featureRegions")}</span>
              <span className="font-medium">{formatLimit(policy.max_regions, unlimited)}</span>
            </li>
            <li data-testid={`pricing-feature-${tier}-max_cpv_sectors`} className="flex items-center justify-between">
              <span>{t("pricing.featureCpvSectors")}</span>
              <span className="font-medium">{formatLimit(policy.max_cpv_sectors, unlimited)}</span>
            </li>
            <li data-testid={`pricing-feature-${tier}-ai_summaries_limit`} className="flex items-center justify-between">
              <span>{t("pricing.featureAiSummaries")}</span>
              <span className="font-medium">{formatLimit(policy.ai_summaries_limit, unlimited)}</span>
            </li>
            <li data-testid={`pricing-feature-${tier}-proposal_drafts_limit`} className="flex items-center justify-between">
              <span>{t("pricing.featureProposalDrafts")}</span>
              <span className="font-medium">{formatLimit(policy.proposal_drafts_limit, unlimited)}</span>
            </li>
            <li data-testid={`pricing-feature-${tier}-compliance_checks_limit`} className="flex items-center justify-between">
              <span>{t("pricing.featureComplianceChecks")}</span>
              <span className="font-medium">{formatLimit(policy.compliance_checks_limit, unlimited)}</span>
            </li>
            <li data-testid={`pricing-feature-${tier}-max_team_members`} className="flex items-center justify-between">
              <span>{t("pricing.featureTeamMembers")}</span>
              <span className="font-medium">{formatLimit(policy.max_team_members, unlimited)}</span>
            </li>
            <li data-testid={`pricing-feature-${tier}-calendar_sync`} className="flex items-center justify-between">
              <span>{t("pricing.featureCalendarSync")}</span>
              <BoolFeature value={policy.calendar_sync} included={included} notIncluded={notIncluded} />
            </li>
            <li data-testid={`pricing-feature-${tier}-api_access`} className="flex items-center justify-between">
              <span>{t("pricing.featureApiAccess")}</span>
              <BoolFeature value={policy.api_access} included={included} notIncluded={notIncluded} />
            </li>
            <li data-testid={`pricing-feature-${tier}-whitelabel`} className="flex items-center justify-between">
              <span>{t("pricing.featureWhitelabel")}</span>
              <BoolFeature value={policy.whitelabel} included={included} notIncluded={notIncluded} />
            </li>
          </ul>
        </div>
      );
    }

    // ─── Main PricingPage component ───────────────────────────────────────────

    export function PricingPage() {
      const t = useTranslations("billing");
      const params = useParams();
      const locale = (params?.locale as string) ?? "en";
      const { data: tiers, isLoading, isError, refetch } = useTiers();

      // Ensure tiers are rendered in canonical order (free → starter → professional → enterprise)
      const orderedTiers = tiers
        ? TIER_ORDER.map(name => tiers.find(p => p.tier === name)).filter(Boolean) as TierPolicy[]
        : [];

      return (
        <div className="min-h-screen bg-slate-50 py-16 px-4 sm:px-6 lg:px-8">
          {/* Hero */}
          <section data-testid="pricing-hero" className="mx-auto max-w-4xl text-center mb-12">
            <h1
              data-testid="pricing-heading"
              className="text-4xl font-bold tracking-tight text-slate-900 sm:text-5xl"
            >
              {t("pricing.heading")}
            </h1>
            <p
              data-testid="pricing-subheading"
              className="mt-4 text-lg text-slate-600"
            >
              {t("pricing.subheading")}
            </p>
          </section>

          {/* Tier grid */}
          {isLoading && (
            <div data-testid="pricing-loading" className="mx-auto max-w-7xl grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
              {[0, 1, 2, 3].map(i => <SkeletonCard key={i} />)}
            </div>
          )}

          {isError && (
            <div data-testid="pricing-error" className="mx-auto max-w-lg">
              <EmptyState
                title={t("pricing.errorHeading")}
                action={
                  <Button onClick={() => refetch()}>{t("pricing.retry")}</Button>
                }
              />
            </div>
          )}

          {!isLoading && !isError && orderedTiers.length > 0 && (
            <div
              data-testid="pricing-grid"
              className="mx-auto max-w-7xl grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 items-start"
            >
              {orderedTiers.map(policy => (
                <TierCard
                  key={policy.tier}
                  policy={policy}
                  isPopular={policy.tier === "professional"}
                  locale={locale}
                  t={t}
                />
              ))}
            </div>
          )}
        </div>
      );
    }
    ```

- [x] **Task 6 — Create the page shell** (AC: 1)
  - [x] 6.1 Create `eusolicit-app/frontend/apps/client/app/[locale]/pricing/page.tsx`:
    ```typescript
    import { PricingPage } from "./components/PricingPage";

    export const metadata = {
      title: "Pricing — EU Solicit",
      description: "Transparent pricing for EU procurement teams. Start free, upgrade when ready.",
    };

    export default function PricingRoute() {
      return <PricingPage />;
    }
    ```

### ATDD Tests

- [x] **Task 7 — Create ATDD test file** (AC: 13)
  - [x] 7.1 Create `eusolicit-app/frontend/apps/client/__tests__/pricing-page-s8-12.test.ts`
  - [x] 7.2 File header comment: story ID `S08.12`, TDD phase (RED before implementation / GREEN after), ACs covered (AC1–AC13), epic test design ID `8.12-COMP-001`
  - [x] 7.3 Use same Vitest + Node `fs` pattern as `__tests__/opportunities-listing-s6-9.test.ts`
  - [x] 7.4 Implement describe blocks:

    **AC1: Page route file existence**
    - Assert `app/[locale]/pricing/page.tsx` exists
    - Assert file does NOT contain `"use client"` (server component shell)
    - Assert file imports or references `PricingPage`

    **AC2: Backend endpoint declaration**
    - Assert `services/client-api/src/client_api/api/v1/billing.py` contains `"/tiers"` route string
    - Assert file contains `get_tier_policies` function name
    - Assert file contains `TierPolicyResponse` class name

    **AC3: Pricing grid testids**
    - Assert `PricingPage.tsx` contains `data-testid="pricing-grid"`
    - Assert `PricingPage.tsx` contains `pricing-tier-free`, `pricing-tier-starter`, `pricing-tier-professional`, `pricing-tier-enterprise`

    **AC4: Monthly prices testids**
    - Assert `PricingPage.tsx` contains `pricing-price-free`, `pricing-price-starter`, `pricing-price-professional`, `pricing-price-enterprise`
    - Assert `TIER_PRICES` constant exists in `lib/api/billing.ts` with all four tier keys

    **AC5: Feature matrix testids**
    - Assert `PricingPage.tsx` contains at least one `pricing-feature-` prefix pattern
    - Assert the nine feature keys appear: `max_regions`, `max_cpv_sectors`, `ai_summaries_limit`, `proposal_drafts_limit`, `compliance_checks_limit`, `max_team_members`, `calendar_sync`, `api_access`, `whitelabel`

    **AC6: Most Popular badge**
    - Assert `PricingPage.tsx` contains `data-testid="pricing-badge-popular"`
    - Assert `PricingPage.tsx` contains `ring-indigo` (Professional highlight) or `ring-2`

    **AC7: CTA buttons testids**
    - Assert `PricingPage.tsx` contains `pricing-cta-free`, `pricing-cta-starter`, `pricing-cta-professional`, `pricing-cta-enterprise`
    - Assert `pricing-cta-professional` is near `startFreeTrial` translation key usage
    - Assert `mailto:sales@eusolicit.com` appears in the component

    **AC8: Loading state**
    - Assert `PricingPage.tsx` contains `data-testid="pricing-loading"`
    - Assert `PricingPage.tsx` contains `SkeletonCard` usage

    **AC9: Error state**
    - Assert `PricingPage.tsx` contains `data-testid="pricing-error"`
    - Assert `PricingPage.tsx` contains `refetch` usage

    **AC10: Hero section**
    - Assert `PricingPage.tsx` contains `data-testid="pricing-hero"`
    - Assert `PricingPage.tsx` contains `data-testid="pricing-heading"`
    - Assert `PricingPage.tsx` contains `data-testid="pricing-subheading"`

    **AC11: i18n key completeness**
    - Read `messages/en.json` and `messages/bg.json`
    - Assert both files contain `billing.pricing` object
    - Assert both contain the following keys under `billing.pricing`: `heading`, `subheading`, `mostPopular`, `perMonth`, `signUpFree`, `getStarted`, `startFreeTrial`, `contactSales`, `featureRegions`, `featureCpvSectors`, `featureAiSummaries`, `featureProposalDrafts`, `featureComplianceChecks`, `featureTeamMembers`, `featureCalendarSync`, `featureApiAccess`, `featureWhitelabel`, `unlimited`, `included`, `notIncluded`, `errorHeading`, `retry`
    - Assert both locales have the same count of keys under `billing.pricing`

    **AC12: API module exports**
    - Assert `lib/api/billing.ts` exists
    - Assert file exports `getTiers`, `TIER_PRICES`, `TIER_ORDER`
    - Assert file defines `TierPolicy` interface (contains `max_regions`, `calendar_sync`, `whitelabel`)
    - Assert file does NOT contain `"use client"` directive
    - Assert `lib/queries/use-billing.ts` exists and exports `useTiers`
    - Assert `lib/queries/use-billing.ts` contains `"use client"`
    - Assert query key `["billing", "tiers"]` is referenced
    - Assert `staleTime: 300_000` is set

## Dev Notes

### Architecture: Mostly Frontend, One Backend Endpoint

S08.12 is primarily a frontend story with a minor backend addition. The only backend change is adding `GET /api/v1/billing/tiers` (a public unauthenticated endpoint) to the existing `billing.py` router. All subscription management backend (webhooks, checkout sessions, portal sessions, usage metering) was completed in S08.01–S08.11.

### Public Route — No Auth Group Needed

The pricing page is placed directly at `app/[locale]/pricing/` (not inside `(protected)/` or `(auth)/`):

```
app/[locale]/
├── (auth)/          ← login, register, forgot-password (AuthRedirect bounces logged-in users)
├── (protected)/     ← authenticated app shell (AuthGuard)
├── pricing/         ← NEW: public marketing page, no auth guard
│   ├── page.tsx     ← server component shell
│   └── components/
│       └── PricingPage.tsx ← "use client" component
└── layout.tsx       ← locale + QueryProvider + i18n (wraps ALL children including pricing)
```

The `[locale]/layout.tsx` already provides `<NextIntlClientProvider>` and `<QueryProvider>` — no additional providers needed for the pricing page.

### Backend Endpoint: No Auth, No JWT

`GET /api/v1/billing/tiers` must work without any `Authorization` header. The existing `_billing_session()` context manager handles DB access. Do NOT add `require_auth()` or `http_bearer()` calls to this endpoint. The Stripe webhook endpoint (`POST /billing/webhooks/stripe`) follows the same no-JWT pattern — use it as reference.

### TierPolicy TypeScript Interface

The interface maps 1:1 to the backend `TierPolicyResponse` Pydantic model:

```typescript
interface TierPolicy {
  id: string;             // UUID as string
  tier: TierName;         // "free" | "starter" | "professional" | "enterprise"
  max_regions: number;    // 1 | 3 | 10 | -1
  max_cpv_sectors: number;// 3 | 10 | 50 | -1
  max_budget_threshold: number; // 500000 | 5000000 | 50000000 | -1
  ai_summaries_limit: number;   // 5 | 25 | 100 | -1
  proposal_drafts_limit: number;// 1 | 5 | 25 | -1
  compliance_checks_limit: number; // 5 | 25 | 100 | -1
  max_team_members: number;     // 2 | 5 | 20 | -1
  calendar_sync: boolean;       // false | true | true | true
  api_access: boolean;          // false | false | true | true
  whitelabel: boolean;          // false | false | false | true
}
```

The `-1` sentinel means "unlimited" — display as the `billing.pricing.unlimited` i18n string.

### Tier Data from DB Seed (Migration 027)

Tier values seeded by migration 027:

| Field | free | starter | professional | enterprise |
|---|---|---|---|---|
| max_regions | 1 | 3 | 10 | -1 (∞) |
| max_cpv_sectors | 3 | 10 | 50 | -1 (∞) |
| max_budget_threshold | 500,000 | 5,000,000 | 50,000,000 | -1 (∞) |
| ai_summaries_limit | 5 | 25 | 100 | -1 (∞) |
| proposal_drafts_limit | 1 | 5 | 25 | -1 (∞) |
| compliance_checks_limit | 5 | 25 | 100 | -1 (∞) |
| max_team_members | 2 | 5 | 20 | -1 (∞) |
| calendar_sync | false | true | true | true |
| api_access | false | false | true | true |
| whitelabel | false | false | false | true |

### Monthly Prices (Hardcoded Frontend Constants)

Monthly prices are **not** stored in the DB and must be hardcoded in `lib/api/billing.ts`:

```typescript
export const TIER_PRICES = {
  free:         { label: "€0",     monthly: 0   },
  starter:      { label: "€49",    monthly: 49  },
  professional: { label: "€149",   monthly: 149 },
  enterprise:   { label: "Custom", monthly: null },
};
```

Enterprise is `monthly: null` → render "Custom pricing" label instead of `/month`.

### Tier Name Translations

The tier name display uses the existing `billing.tier*` keys already present in both locale files:
- `t("tierFree")` → "Free" / "Безплатен"
- `t("tierStarter")` → "Starter" / "Начален"
- `t("tierProfessional")` → "Professional" / "Професионален"
- `t("tierEnterprise")` → "Enterprise" / "Корпоративен"

**Important:** The tier card uses template literal key construction: `` t(`tier${tier.charAt(0).toUpperCase()}${tier.slice(1)}`) ``. TypeScript needs the return type cast properly. Alternative approach if TypeScript rejects the template literal:

```typescript
const tierNameKey: Record<TierName, "tierFree" | "tierStarter" | "tierProfessional" | "tierEnterprise"> = {
  free: "tierFree",
  starter: "tierStarter",
  professional: "tierProfessional",
  enterprise: "tierEnterprise",
};
// Then: t(tierNameKey[tier])
```

Use the explicit map approach to avoid TypeScript errors with dynamic key construction.

### Tier Ordering

The backend returns tiers in arbitrary DB order. Client-side ordering uses `TIER_ORDER`:

```typescript
const TIER_ORDER: TierName[] = ["free", "starter", "professional", "enterprise"];
const orderedTiers = TIER_ORDER
  .map(name => tiers.find(p => p.tier === name))
  .filter(Boolean) as TierPolicy[];
```

This ensures deterministic display order regardless of DB retrieval order.

### CTA Button Behaviour

| Tier | CTA Label | Link Target |
|---|---|---|
| Free | "Sign Up Free" | `/{locale}/register` (Link) |
| Starter | "Get Started" | `/{locale}/register?plan=starter` (Link) |
| Professional | "Start Free Trial" | `/{locale}/register?plan=professional` (Link, `variant="default"`) |
| Enterprise | "Contact Sales" | `mailto:sales@eusolicit.com` (anchor, `target="_blank"`) |

The Professional CTA uses `variant="default"` (filled indigo) while all others use `variant="outline"` to emphasise the recommended tier.

### EmptyState Component API

`EmptyState` from `@eusolicit/ui` accepts:
- `title` (string) — displayed as heading
- `action` (ReactNode, optional) — rendered below the title

```tsx
<EmptyState
  title={t("pricing.errorHeading")}
  action={<Button onClick={() => refetch()}>{t("pricing.retry")}</Button>}
/>
```

### File Structure

```
eusolicit-app/
├── services/client-api/src/client_api/api/v1/
│   └── billing.py                              MODIFIED — add GET /tiers endpoint + TierPolicyResponse
├── frontend/apps/client/
│   ├── app/[locale]/pricing/
│   │   ├── page.tsx                            NEW — server component shell
│   │   └── components/
│   │       └── PricingPage.tsx                 NEW — "use client" pricing page component
│   ├── lib/
│   │   ├── api/
│   │   │   └── billing.ts                      NEW — TierPolicy interface + getTiers() + TIER_PRICES
│   │   └── queries/
│   │       └── use-billing.ts                  NEW — useTiers() React Query hook
│   ├── messages/
│   │   ├── en.json                             MODIFIED — add billing.pricing.* keys
│   │   └── bg.json                             MODIFIED — add billing.pricing.* keys (Bulgarian)
│   └── __tests__/
│       └── pricing-page-s8-12.test.ts          NEW — ATDD structural tests
```

**Do NOT create or modify:**
- Any other backend files (webhook, checkout, portal, usage metering — all done in S08.01–S08.11)
- Trial banner (S08.13)
- Subscription management page (S08.13)
- Cache invalidation (S08.14)
- Any Zustand stores (pricing page has no local state beyond `useTiers` hook)
- Auth guard or AuthRedirect — pricing is public

### Critical Mistakes to Prevent

1. **Do NOT put the pricing page inside `(protected)/` or `(auth)/`** — it is a public page. Place it at `[locale]/pricing/` directly.

2. **Do NOT add authentication to `GET /api/v1/billing/tiers`** — the endpoint must work without a JWT token. Verify by calling `curl http://localhost:8000/api/v1/billing/tiers` with no auth header.

3. **Do NOT hardcode tier feature limits in the frontend** — they must come from the API response so changes to the DB seed propagate automatically to the pricing page.

4. **Do NOT add `"use client"` to `lib/api/billing.ts`** — it is a plain async function usable in both server and client contexts. The ATDD test asserts its absence.

5. **Do NOT duplicate billing namespace keys** — merge the new `pricing` sub-object into the existing `billing` object in both locale files. Replacing the entire `billing` object would delete VAT, checkout, and portal strings.

6. **Enterprise CTA uses `<a>` not `<Link>`** — `mailto:` links are not Next.js routes. Use `<a href="mailto:..." target="_blank">` for the Enterprise card CTA.

7. **Tier name display** — Use the `tierNameKey` lookup map (see Dev Notes above) rather than template literal key construction to avoid TypeScript type errors.

8. **-1 sentinel display** — The `formatLimit()` helper must display `"Unlimited"` (from i18n) for any field with value `-1`. Do not render the literal `-1` to the user.

### Test Coverage Map (from Epic 8 Test Design)

| Test Design ID | Priority | Scenario | Story Coverage |
|---|---|---|---|
| **8.12-COMP-001** | P2 | Pricing page renders all four tier comparison rows with correct feature limits from API | `PricingPage.tsx` renders with mocked `getTiers()` response; ATDD asserts tier names and feature limits visible via testids |

**Additional coverage from this story's ATDD tests:**
- Backend endpoint existence and public accessibility verified by checking billing.py for the route declaration
- i18n key completeness in both locales (AC11)
- CTA button routing (testid presence, `mailto:` for Enterprise) (AC7)
- Loading and error states (AC8, AC9)

### Backend Unit Test (Optional — Add if time permits)

A unit test for `GET /api/v1/billing/tiers` can be added at `services/client-api/tests/unit/test_billing_tiers_endpoint.py`:

```python
"""Unit test for GET /api/v1/billing/tiers — Story 8-12 AC2."""
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from fastapi.testclient import TestClient

from client_api.main import app

MOCK_POLICIES = [
    MagicMock(id="uuid-free", tier="free", max_regions=1, max_cpv_sectors=3,
              max_budget_threshold=500000, ai_summaries_limit=5, proposal_drafts_limit=1,
              compliance_checks_limit=5, max_team_members=2,
              calendar_sync=False, api_access=False, whitelabel=False),
    # ... add starter, professional, enterprise mocks
]

@pytest.mark.unit
def test_get_tier_policies_returns_four_tiers(mock_billing_session):
    """GET /api/v1/billing/tiers returns 4 tiers in correct order without auth."""
    # Verify no Authorization header required — request passes without JWT
    client = TestClient(app)
    # ... mock _billing_session and assert response
```

This is optional for this story since test coverage is provided by the ATDD structural tests and integration tests from Story 8-2 (schema/seed verification).

## Dev Agent Record

### Implementation Notes

- **Task 1 (Backend):** Added `TierPolicyResponse` Pydantic model, `_TIER_ORDER` sort dict, and `get_tier_policies()` endpoint to `billing.py`. Endpoint placed at end of billing router block (before subscription_router). Updated module docstring. No auth guard used — public endpoint consistent with Stripe webhook pattern. `TierAccessPolicy` import was already present.

- **Task 2 (API module):** Created `lib/api/billing.ts` with `TierPolicy` interface, `TierName` union type, `TIER_PRICES`, `TIER_ORDER`, and `getTiers()` async function using `apiClient`. No `use-client` directive — usable in both server and client contexts.

- **Task 3 (Query hook):** Created `lib/queries/use-billing.ts` with `useTiers()` using `useQuery`, `queryKey: ["billing", "tiers"]`, `staleTime: 300_000`, `retry: 2`.

- **Task 4 (i18n):** Merged `pricing` sub-object into existing `billing` namespace in both `en.json` and `bg.json`. 23 keys in each locale. Both files validated as valid JSON with `python3 -m json.tool`. Existing billing keys preserved.

- **Task 5 (PricingPage component):** Created `PricingPage.tsx` as `"use client"` component. Used explicit lookup maps (`TIER_CARD_TESTIDS`, `PRICE_TESTIDS`) with literal strings for each tier so static ATDD tests can detect them. Split CTA rendering by tier to ensure `variant="default"` and `href="mailto:` appear literally in source. `TierCard` renders the boolean features with `CheckCircle2`/`XCircle` icons and the `-1` sentinel via `formatLimit()`. Enterprise uses `<a>` not `<Link>` for the mailto CTA.

- **Task 6 (Page shell):** Created `app/[locale]/pricing/page.tsx` as server component (no `use client`). Placed at `[locale]/pricing/` — NOT inside `(protected)` or `(auth)` groups. The `[locale]/layout.tsx` already provides QueryProvider and next-intl.

- **Task 7 (ATDD tests):** Pre-written ATDD test file `__tests__/pricing-page-s8-12.test.ts` already existed. All 160 tests pass GREEN after implementation.

### Test Results

- `__tests__/pricing-page-s8-12.test.ts`: **160/160 tests PASS** ✅
- Full suite baseline before changes: 10 failed test files (pre-existing)
- Full suite after changes: 9 failed test files — no regressions introduced; the pre-existing pricing-page-s8-12 failures are now resolved

### Completion Notes

✅ Story 8.12 implementation complete:
- Backend: `GET /api/v1/billing/tiers` public endpoint — no auth, ordered by tier
- Frontend: `lib/api/billing.ts`, `lib/queries/use-billing.ts`, `PricingPage.tsx`, `app/[locale]/pricing/page.tsx`
- i18n: 23 keys added to `billing.pricing.*` in both en.json and bg.json
- ATDD: 160 structural tests all GREEN

## Change Log

- 2026-04-19: Story 8.12 implemented — backend public tier endpoint, pricing page UI (hero + 4-column grid with feature matrix + CTA), i18n keys (en/bg), ATDD 160/160 GREEN

## File List

**New files:**
- `eusolicit-app/frontend/apps/client/app/[locale]/pricing/page.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/pricing/components/PricingPage.tsx`
- `eusolicit-app/frontend/apps/client/lib/api/billing.ts`
- `eusolicit-app/frontend/apps/client/lib/queries/use-billing.ts`

**Modified files:**
- `eusolicit-app/services/client-api/src/client_api/api/v1/billing.py` — add `TierPolicyResponse`, `_TIER_ORDER`, `get_tier_policies()` endpoint; update module docstring
- `eusolicit-app/frontend/apps/client/messages/en.json` — add `billing.pricing.*` keys (23 keys)
- `eusolicit-app/frontend/apps/client/messages/bg.json` — add `billing.pricing.*` keys in Bulgarian

**Pre-existing (already present, not modified):**
- `eusolicit-app/frontend/apps/client/__tests__/pricing-page-s8-12.test.ts` — ATDD test file (pre-written)

## References

- [Source: eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md#S08.12] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-08.md#8.12-COMP-001] — P2 component test: pricing page renders tiers with correct feature limits
- [Source: eusolicit-app/services/client-api/alembic/versions/027_subscription_billing_schema.py] — tier seed values (free/starter/professional/enterprise limits)
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/billing.py] — existing billing router; add GET /tiers here
- [Source: eusolicit-app/services/client-api/src/client_api/models/tier_access_policy.py] — TierAccessPolicy ORM model fields
- [Source: eusolicit-app/packages/eusolicit-models/src/eusolicit_models/dtos.py#TierAccessPolicyDTO] — DTO shape matches TierPolicy interface
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/layout.tsx] — locale layout wraps all children including pricing (provides QueryProvider + i18n)
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/(auth)/layout.tsx] — auth layout pattern (DO NOT use for pricing)
- [Source: eusolicit-app/frontend/apps/client/lib/api/opportunities.ts] — API module pattern (follow for billing.ts)
- [Source: eusolicit-app/frontend/apps/client/lib/queries/use-opportunities.ts] — React Query hook pattern (follow for use-billing.ts)
- [Source: eusolicit-app/frontend/apps/client/__tests__/opportunities-listing-s6-9.test.ts] — ATDD test file structure and helpers
- [Source: eusolicit-app/frontend/packages/ui/index.ts] — available components: Badge, Button, SkeletonCard, EmptyState, cn
- [Source: eusolicit-app/frontend/apps/client/messages/en.json] — existing `billing.*` keys; merge pricing sub-object
- [Source: eusolicit-app/frontend/apps/client/messages/bg.json] — Bulgarian translations to match en.json
