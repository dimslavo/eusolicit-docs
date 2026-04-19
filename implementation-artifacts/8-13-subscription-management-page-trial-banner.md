# Story 8.13: Subscription Management Page & Trial Banner

Status: review

## Story

As a **company user during or after a trial**,
I want a **subscription management dashboard at `/settings/billing` showing current tier, usage meters, billing history, and a persistent trial banner in the topbar during the trial period**,
so that I can **monitor usage, understand my plan limits, and upgrade proactively before the trial ends**.

## Acceptance Criteria

1. **AC1 — Refactored Subscription Dashboard** — The existing `app/[locale]/(protected)/settings/billing/page.tsx` is replaced by a refactored `SubscriptionManagementPage` client component that displays:
   - Current tier name + status badge (trialing / active / past_due / canceled) with `data-testid="subscription-tier"` and `data-testid="subscription-status-badge"`
   - Trial end date (if `is_trial: true`) formatted as a locale date string with `data-testid="subscription-trial-end"`
   - Billing period from `current_period_end` with `data-testid="subscription-billing-period"`
   - Usage meters section (three linear progress bars — AI summaries, proposal drafts, compliance checks) using data from `GET /api/v1/subscription/usage`
   - "Manage Subscription" portal button (preserve existing portal mutation from S08.07)
   - Billing history table from `GET /api/v1/billing/invoices`
   - Page title `data-testid="subscription-page-title"`

2. **AC2 — Usage Meters (Progress Bars)** — Three progress bar meters rendered in a section with `data-testid="subscription-usage-meters"`. Each meter:
   - Outer wrapper: `data-testid="subscription-usage-{metric}"` where `{metric}` ∈ `{"ai_summary", "proposal_draft", "compliance_check"}`
   - Shows label (from i18n), consumed/limit text `data-testid="subscription-usage-{metric}-text"` (e.g., "45 / 100"), and percentage `data-testid="subscription-usage-{metric}-pct"`
   - Linear bar with `data-testid="subscription-usage-{metric}-bar"` filled to `Math.min(consumed / limit * 100, 100)%`
   - Bar color: `bg-indigo-500` when < 80%, `bg-amber-500` when 80–99%, `bg-red-500` when ≥ 100%
   - When `limit == null` (Enterprise unlimited): render "Unlimited" label instead of meter; `data-testid="subscription-usage-{metric}-unlimited"`
   - Loading state: three `<Skeleton>` placeholders with `data-testid="subscription-usage-loading"`

3. **AC3 — Billing History Table** — A table section with `data-testid="subscription-billing-history"` renders invoices from `GET /api/v1/billing/invoices`. Columns: Date (`created_at`, formatted locale date), Status (open / paid), Payment Terms (NET 30 / NET 60), Amount (from `line_items` sum). Empty state: `<EmptyState data-testid="subscription-billing-empty" title={t("billing.subscription.noInvoices")} />`. Max 12 invoices shown.

4. **AC4 — Trial Banner Component** — New `TrialBanner` component at `app/[locale]/(protected)/components/TrialBanner.tsx` (`"use client"`):
   - Rendered only when `subscription.status === "trialing"` AND `subscription.trial_end` is set
   - Full-width amber strip with `data-testid="trial-banner"`, `role="banner"`
   - Days remaining computed from `trial_end` minus today (Math.ceil, min 0)
   - Text: `t("billing.trialBanner.daysRemaining", { days: N })` with `data-testid="trial-banner-days"`
   - "Upgrade Now" `<Link>` to `/{locale}/settings/billing` with `data-testid="trial-banner-upgrade-cta"`, `variant="default"`, size small
   - Close button (X) with `data-testid="trial-banner-close"` that dismisses banner for the session only (sessionStorage flag `eusolicit-trial-banner-dismissed`)
   - Renders nothing (null) if dismissed flag is set in sessionStorage
   - Zero-day state: renders "Your trial has ended — upgrade to continue" text instead of countdown

5. **AC5 — Trial Banner Integration in Protected Layout** — `app/[locale]/(protected)/layout.tsx` is updated to:
   - Import `TrialBanner` from `./components/TrialBanner`
   - Fetch subscription via `useQuery({ queryKey: ["billing", "subscription"], ... })` (same key as billing page — shares cache, no extra network request)
   - Pass the banner in the `topbar` slot as a React Fragment:
     ```tsx
     topbar={
       <>
         <TopBar ... />
         {subscription?.status === "trialing" && (
           <TrialBanner trialEnd={subscription.trial_end!} locale={locale} />
         )}
       </>
     }
     ```
   - `useQuery` for subscription must NOT block layout rendering — use `{ enabled: true, staleTime: 60_000 }` so it runs in background; banner simply does not render until data arrives

6. **AC6 — Backend Billing Invoices Endpoint** — `GET /api/v1/billing/invoices` added to `billing.py`:
   - Authenticated (any company user — same `require_auth` pattern as `/subscription`)
   - Queries `client.enterprise_invoices` WHERE `company_id = current_user.company_id` ORDER BY `created_at DESC` LIMIT 12
   - Returns list of `InvoiceDTO` objects: `id`, `stripe_invoice_id`, `payment_terms`, `line_items`, `status`, `created_at` (ISO string), `paid_at` (ISO string or null)
   - Returns empty list `[]` when no invoices exist (no 404)
   - Uses `_billing_session()` context manager (consistent with existing billing endpoints)
   - Module docstring in `billing.py` updated to add: `GET /billing/invoices — Company billing history (Story 8-13)`

7. **AC7 — API Module & React Query Hooks** — `lib/api/billing.ts` updated (NOT replaced) to add:
   - `SubscriptionStatus` TypeScript interface (consolidate the inline definition from `settings/billing/page.tsx`)
   - `InvoiceRecord` TypeScript interface: `{ id: string; stripe_invoice_id: string; payment_terms: string; line_items: unknown[]; status: string; created_at: string; paid_at: string | null; }`
   - `UsageResponse` TypeScript interface matching `/api/v1/subscription/usage` response shape
   - `getSubscription(): Promise<SubscriptionStatus>` — calls `GET /api/v1/billing/subscription`
   - `getUsage(): Promise<UsageResponse>` — calls `GET /api/v1/subscription/usage`
   - `getInvoices(): Promise<InvoiceRecord[]>` — calls `GET /api/v1/billing/invoices`

   `lib/queries/use-billing.ts` updated (NOT replaced) to add:
   - `useSubscription()` hook: `queryKey: ["billing", "subscription"]`, `staleTime: 60_000`, `refetchOnWindowFocus: true`
   - `useUsage()` hook: `queryKey: ["billing", "usage"]`, `staleTime: 120_000`
   - `useInvoices()` hook: `queryKey: ["billing", "invoices"]`, `staleTime: 300_000`

8. **AC8 — i18n** — New keys merged (NOT replacing existing keys) into `billing.*` namespace in both `en.json` and `bg.json`. New sub-namespaces:
   - `billing.trialBanner.daysRemaining` — "X days remaining in your Professional trial" (`{days}` interpolation)
   - `billing.trialBanner.zeroDay` — "Your trial has ended — upgrade to continue"
   - `billing.trialBanner.upgradeNow` — "Upgrade Now"
   - `billing.trialBanner.dismiss` — "Dismiss trial banner"
   - `billing.subscription.pageTitle` — "Subscription & Billing" (replaces inline use of existing `billing.pageTitle`)
   - `billing.subscription.usageMetersTitle` — "Usage This Billing Period"
   - `billing.subscription.aiSummaryLabel` — "AI Summaries"
   - `billing.subscription.proposalDraftLabel` — "Proposal Drafts"
   - `billing.subscription.complianceCheckLabel` — "Compliance Checks"
   - `billing.subscription.unlimited` — "Unlimited"
   - `billing.subscription.billingPeriodLabel` — "Current Billing Period"
   - `billing.subscription.billingHistoryTitle` — "Billing History"
   - `billing.subscription.noInvoices` — "No invoices found"
   - `billing.subscription.invoiceDate` — "Date"
   - `billing.subscription.invoiceStatus` — "Status"
   - `billing.subscription.invoiceTerms` — "Payment Terms"
   - `billing.subscription.statusBadge.trialing` — "Trial"
   - `billing.subscription.statusBadge.active` — "Active"
   - `billing.subscription.statusBadge.pastDue` — "Past Due"
   - `billing.subscription.statusBadge.canceled` — "Canceled"

9. **AC9 — ATDD Tests** — `__tests__/subscription-management-s8-13.test.ts` covers:
   - File structure: `settings/billing/page.tsx` exists, `TrialBanner.tsx` exists, `lib/api/billing.ts` exists (updated), `lib/queries/use-billing.ts` exists (updated)
   - Backend endpoint: `billing.py` contains `"/invoices"` route string and `get_invoices` or `InvoiceDTO` function name
   - Component testids: `subscription-page-title`, `subscription-tier`, `subscription-status-badge`, `subscription-billing-period`, `subscription-usage-meters`, `subscription-usage-ai_summary`, `subscription-usage-proposal_draft`, `subscription-usage-compliance_check`, `subscription-billing-history`
   - Trial banner testids: `trial-banner`, `trial-banner-days`, `trial-banner-upgrade-cta`, `trial-banner-close`
   - API module exports: `billing.ts` exports `getSubscription`, `getUsage`, `getInvoices`, `SubscriptionStatus`, `InvoiceRecord`, `UsageResponse`
   - Hook exports: `use-billing.ts` exports `useSubscription`, `useUsage`, `useInvoices`, `useTiers`
   - i18n completeness: both locales contain `billing.trialBanner.daysRemaining`, `billing.trialBanner.upgradeNow`, `billing.subscription.usageMetersTitle`, all metric labels, `billing.subscription.noInvoices`
   - Protected layout updated: `(protected)/layout.tsx` contains `TrialBanner` import reference and `"billing", "subscription"` query key

## Tasks / Subtasks

### Backend — Billing Invoices Endpoint (client-api)

- [x] **Task 1 — Add `GET /api/v1/billing/invoices` to `billing.py`** (AC: 6)
  - [x] 1.1 Open `eusolicit-app/services/client-api/src/client_api/api/v1/billing.py`
  - [x] 1.2 Add import for `EnterpriseInvoice` model at top of file (with other model imports):
    ```python
    from client_api.models.enterprise_invoice import EnterpriseInvoice
    ```
  - [x] 1.3 Add `InvoiceDTO` Pydantic response model (after `TierPolicyResponse`, before `_TIER_ORDER`):
    ```python
    # ---------------------------------------------------------------------------
    # GET /billing/invoices — Story 8-13 AC6: company billing history
    # ---------------------------------------------------------------------------

    class InvoiceDTO(BaseModel):
        """Public representation of an enterprise invoice row for the billing history table."""

        id: str
        stripe_invoice_id: str
        payment_terms: str
        line_items: list
        status: str
        created_at: str   # ISO datetime string
        paid_at: str | None
    ```
  - [x] 1.4 Add `get_invoices` endpoint (after `get_tier_policies`, before the Stripe webhook handler block or at end of `billing` router block):
    ```python
    @router.get("/invoices", status_code=200)
    async def get_invoices(
        request: Request,
    ) -> list[InvoiceDTO]:
        """Return billing invoice history for the authenticated company.

        Returns enterprise custom invoices ordered newest-first, up to 12 records.
        Returns empty list [] when no invoices exist — never 404.

        Auth: any authenticated company user (JWT required, any role).
        Story 8-13 AC6.
        """
        credentials = await http_bearer(request)
        current_user: CurrentUser = await require_auth(credentials)

        async with _billing_session() as session:
            result = await session.execute(
                select(EnterpriseInvoice)
                .where(EnterpriseInvoice.company_id == current_user.company_id)
                .order_by(EnterpriseInvoice.created_at.desc())
                .limit(12)
            )
            invoices = result.scalars().all()

        return [
            InvoiceDTO(
                id=str(inv.id),
                stripe_invoice_id=inv.stripe_invoice_id,
                payment_terms=inv.payment_terms,
                line_items=inv.line_items or [],
                status=inv.status,
                created_at=inv.created_at.isoformat(),
                paid_at=inv.paid_at.isoformat() if inv.paid_at else None,
            )
            for inv in invoices
        ]
    ```
  - [x] 1.5 Update the module docstring at the top of `billing.py` to include:
    ```python
    #   GET  /billing/invoices          — Company billing history (Story 8-13)
    ```

### Frontend — API Module & React Query Hooks

- [x] **Task 2 — Update `lib/api/billing.ts`** (AC: 7)
  - [x] 2.1 Open `eusolicit-app/frontend/apps/client/lib/api/billing.ts`
  - [x] 2.2 Add the following TypeScript interfaces (after `TierPolicy`, before `TIER_PRICES`):
    ```typescript
    // ─── Subscription types ─────────────────────────────────────────────────────

    export interface SubscriptionStatus {
      tier: string;
      status: string | null;
      is_trial: boolean;
      trial_end: string | null;
      current_period_end: string | null;
      stripe_customer_id: string | null;
    }

    export interface UsageMetric {
      consumed: number;
      limit: number | null;  // null = unlimited (Enterprise)
      remaining: number | null;
    }

    export interface UsageResponse {
      billing_period: string;
      usage: {
        ai_summary: UsageMetric;
        proposal_draft: UsageMetric;
        compliance_check: UsageMetric;
      };
    }

    export interface InvoiceRecord {
      id: string;
      stripe_invoice_id: string;
      payment_terms: string;
      line_items: unknown[];
      status: string;
      created_at: string;
      paid_at: string | null;
    }
    ```
  - [x] 2.3 Add the following API functions (after `getTiers()`):
    ```typescript
    export async function getSubscription(): Promise<SubscriptionStatus> {
      const response = await apiClient.get<SubscriptionStatus>("/api/v1/billing/subscription");
      return response.data;
    }

    export async function getUsage(): Promise<UsageResponse> {
      const response = await apiClient.get<UsageResponse>("/api/v1/subscription/usage");
      return response.data;
    }

    export async function getInvoices(): Promise<InvoiceRecord[]> {
      const response = await apiClient.get<InvoiceRecord[]>("/api/v1/billing/invoices");
      return response.data;
    }
    ```

- [x] **Task 3 — Update `lib/queries/use-billing.ts`** (AC: 7)
  - [x] 3.1 Open `eusolicit-app/frontend/apps/client/lib/queries/use-billing.ts`
  - [x] 3.2 Update imports to include `getSubscription`, `getUsage`, `getInvoices`:
    ```typescript
    import { getTiers, getSubscription, getUsage, getInvoices } from "@/lib/api/billing";
    ```
  - [x] 3.3 Add three new hooks (after `useTiers()`):
    ```typescript
    /**
     * Hook for current subscription state.
     * queryKey: ["billing", "subscription"] — shared with protected layout trial banner.
     * refetchOnWindowFocus: true — refreshes on portal return (AC5 of Story 8-7).
     */
    export function useSubscription() {
      return useQuery({
        queryKey: ["billing", "subscription"],
        queryFn: getSubscription,
        staleTime: 60_000,         // 1 min
        refetchOnWindowFocus: true, // re-fetch when user returns from Stripe portal tab
        retry: 1,
      });
    }

    /**
     * Hook for usage metering data (Redis counters vs. tier limits).
     * queryKey: ["billing", "usage"]
     */
    export function useUsage() {
      return useQuery({
        queryKey: ["billing", "usage"],
        queryFn: getUsage,
        staleTime: 120_000, // 2 min
        retry: 1,
      });
    }

    /**
     * Hook for billing invoice history.
     * queryKey: ["billing", "invoices"]
     */
    export function useInvoices() {
      return useQuery({
        queryKey: ["billing", "invoices"],
        queryFn: getInvoices,
        staleTime: 300_000, // 5 min
        retry: 1,
      });
    }
    ```

### Frontend — i18n Keys

- [x] **Task 4 — Add i18n keys to `en.json`** (AC: 8)
  - [x] 4.1 Open `eusolicit-app/frontend/apps/client/messages/en.json`
  - [x] 4.2 Merge the following into the existing `"billing"` object (do NOT overwrite existing keys):
    ```json
    "trialBanner": {
      "daysRemaining": "{days} days remaining in your Professional trial",
      "zeroDay": "Your trial has ended — upgrade to continue",
      "upgradeNow": "Upgrade Now",
      "dismiss": "Dismiss trial banner"
    },
    "subscription": {
      "pageTitle": "Subscription & Billing",
      "usageMetersTitle": "Usage This Billing Period",
      "aiSummaryLabel": "AI Summaries",
      "proposalDraftLabel": "Proposal Drafts",
      "complianceCheckLabel": "Compliance Checks",
      "unlimited": "Unlimited",
      "billingPeriodLabel": "Current Billing Period",
      "billingHistoryTitle": "Billing History",
      "noInvoices": "No invoices found",
      "invoiceDate": "Date",
      "invoiceStatus": "Status",
      "invoiceTerms": "Payment Terms",
      "statusBadge": {
        "trialing": "Trial",
        "active": "Active",
        "pastDue": "Past Due",
        "canceled": "Canceled"
      }
    }
    ```

- [x] **Task 5 — Add i18n keys to `bg.json`** (AC: 8)
  - [x] 5.1 Open `eusolicit-app/frontend/apps/client/messages/bg.json`
  - [x] 5.2 Merge matching Bulgarian translations into the existing `"billing"` object:
    ```json
    "trialBanner": {
      "daysRemaining": "{days} дни остават от вашия Professional пробен период",
      "zeroDay": "Вашият пробен период е изтекъл — надградете за да продължите",
      "upgradeNow": "Надградете сега",
      "dismiss": "Затвори банера за пробния период"
    },
    "subscription": {
      "pageTitle": "Абонамент и фактуриране",
      "usageMetersTitle": "Употреба за текущия период",
      "aiSummaryLabel": "AI Резюмета",
      "proposalDraftLabel": "Чернови на оферти",
      "complianceCheckLabel": "Проверки за съответствие",
      "unlimited": "Неограничен",
      "billingPeriodLabel": "Текущ период на фактуриране",
      "billingHistoryTitle": "История на фактурите",
      "noInvoices": "Няма намерени фактури",
      "invoiceDate": "Дата",
      "invoiceStatus": "Статус",
      "invoiceTerms": "Условия на плащане",
      "statusBadge": {
        "trialing": "Пробен",
        "active": "Активен",
        "pastDue": "Закъснял",
        "canceled": "Отменен"
      }
    }
    ```
  - [x] 5.3 Validate both JSON files remain valid: `python3 -m json.tool messages/en.json > /dev/null && python3 -m json.tool messages/bg.json > /dev/null`

### Frontend — Trial Banner Component

- [x] **Task 6 — Create `TrialBanner.tsx`** (AC: 4)
  - [x] 6.1 Create `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/components/TrialBanner.tsx`:
    ```typescript
    "use client";

    import { useState, useEffect } from "react";
    import Link from "next/link";
    import { useTranslations } from "next-intl";
    import { X } from "lucide-react";
    import { Button } from "@eusolicit/ui";

    const DISMISS_KEY = "eusolicit-trial-banner-dismissed";

    interface TrialBannerProps {
      trialEnd: string;   // ISO datetime string from subscription.trial_end
      locale: string;
    }

    export function TrialBanner({ trialEnd, locale }: TrialBannerProps) {
      const t = useTranslations("billing");
      const [dismissed, setDismissed] = useState(false);
      const [mounted, setMounted] = useState(false);

      useEffect(() => {
        setMounted(true);
        // Read sessionStorage only on client to avoid SSR mismatch
        setDismissed(sessionStorage.getItem(DISMISS_KEY) === "true");
      }, []);

      if (!mounted || dismissed) return null;

      const daysRemaining = Math.max(
        0,
        Math.ceil((new Date(trialEnd).getTime() - Date.now()) / (1000 * 60 * 60 * 24))
      );

      const handleDismiss = () => {
        sessionStorage.setItem(DISMISS_KEY, "true");
        setDismissed(true);
      };

      return (
        <div
          data-testid="trial-banner"
          role="banner"
          className="w-full bg-amber-50 border-b border-amber-200 px-4 py-2 flex items-center justify-between gap-4"
        >
          <span className="text-sm text-amber-800">
            {daysRemaining === 0
              ? t("trialBanner.zeroDay")
              : (
                <span data-testid="trial-banner-days">
                  {t("trialBanner.daysRemaining", { days: daysRemaining })}
                </span>
              )
            }
          </span>
          <div className="flex items-center gap-2 flex-shrink-0">
            <Link href={`/${locale}/settings/billing`} data-testid="trial-banner-upgrade-cta">
              <Button size="sm" variant="default">
                {t("trialBanner.upgradeNow")}
              </Button>
            </Link>
            <button
              data-testid="trial-banner-close"
              onClick={handleDismiss}
              aria-label={t("trialBanner.dismiss")}
              className="text-amber-600 hover:text-amber-900 p-1 rounded"
            >
              <X className="h-4 w-4" aria-hidden="true" />
            </button>
          </div>
        </div>
      );
    }
    ```

### Frontend — Protected Layout Update

- [x] **Task 7 — Update `(protected)/layout.tsx` for trial banner** (AC: 5)
  - [x] 7.1 Open `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx`
  - [x] 7.2 Add imports at top:
    ```typescript
    import { useQuery } from "@tanstack/react-query";
    import { getSubscription } from "@/lib/api/billing";
    import { TrialBanner } from "./components/TrialBanner";
    ```
  - [x] 7.3 Add subscription query inside the component body (after existing `useEffect` hooks, before `return`):
    ```typescript
    // Trial banner: fetch subscription state in background — does NOT block layout render
    const { data: subscription } = useQuery({
      queryKey: ["billing", "subscription"],
      queryFn: getSubscription,
      staleTime: 60_000,
      retry: 1,
    });
    ```
  - [x] 7.4 Update the `topbar` prop of `<AppShell>` to wrap `<TopBar>` and `<TrialBanner>` in a React Fragment:
    ```tsx
    topbar={
      <>
        <TopBar
          user={{ name: user?.name ?? "", email: user?.email ?? "" }}
          locale={locale as Locale}
          onLocaleChange={handleLocaleChange}
          onSignOut={handleSignOut}
          userMenuLabels={{ profile: tAuth("profile"), settings: tAuth("settings"), signOut: tAuth("signOut") }}
          onOpenMobileSidebar={showMobile ? () => setIsMobileSheetOpen(true) : undefined}
        />
        {subscription?.status === "trialing" && subscription.trial_end && (
          <TrialBanner trialEnd={subscription.trial_end} locale={locale} />
        )}
      </>
    }
    ```

### Frontend — Subscription Management Page (Refactor)

- [x] **Task 8 — Create `SubscriptionManagementPage` component** (AC: 1, 2, 3)
  - [x] 8.1 Create directory `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/billing/components/`
  - [x] 8.2 Create `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/billing/components/SubscriptionManagementPage.tsx`:
    ```typescript
    "use client";

    // SubscriptionManagementPage.tsx — Story 8-13
    // Full subscription management dashboard: tier + status, usage meters,
    // portal button, and billing history. Replaces the S08.06/S08.07 stub.

    import { useMutation } from "@tanstack/react-query";
    import { useTranslations } from "next-intl";
    import { useParams } from "next/navigation";
    import Link from "next/link";
    import {
      Badge,
      Button,
      Skeleton,
      EmptyState,
      Table,
      TableBody,
      TableCell,
      TableHead,
      TableHeader,
      TableRow,
      useToast,
    } from "@eusolicit/ui";
    import { apiClient } from "@eusolicit/ui";
    import { useSubscription, useUsage, useInvoices } from "@/lib/queries/use-billing";
    import type { UsageMetric } from "@/lib/api/billing";

    // ─── Usage meter helpers ─────────────────────────────────────────────────────

    const METRIC_KEYS = ["ai_summary", "proposal_draft", "compliance_check"] as const;
    type MetricKey = (typeof METRIC_KEYS)[number];

    function usagePct(metric: UsageMetric): number {
      if (!metric.limit) return 0;
      return Math.min(Math.round((metric.consumed / metric.limit) * 100), 100);
    }

    function barColor(pct: number): string {
      if (pct >= 100) return "bg-red-500";
      if (pct >= 80) return "bg-amber-500";
      return "bg-indigo-500";
    }

    // ─── Status badge helper ─────────────────────────────────────────────────────

    const STATUS_VARIANT: Record<string, "default" | "secondary" | "destructive" | "outline"> = {
      trialing: "secondary",
      active: "default",
      past_due: "destructive",
      canceled: "outline",
    };

    // ─── Main component ──────────────────────────────────────────────────────────

    export function SubscriptionManagementPage() {
      const t = useTranslations("billing");
      const toast = useToast();
      const params = useParams();
      const locale = (params?.locale as string) ?? "en";

      const { data: subscription, isLoading: subLoading } = useSubscription();
      const { data: usageData, isLoading: usageLoading } = useUsage();
      const { data: invoices, isLoading: invoicesLoading } = useInvoices();

      // Stripe Customer Portal (from S08.07 — preserved)
      const portalMutation = useMutation({
        mutationFn: async () => {
          const res = await apiClient.post<{ portal_url: string }>("/api/v1/billing/portal/session", {});
          return res.data;
        },
        onSuccess: (data) => {
          window.open(data.portal_url, "_blank");
        },
        onError: () => {
          toast.error(t("portalError"));
        },
      });

      const tierLabel = (tier: string) => ({
        free: t("tierFree"),
        starter: t("tierStarter"),
        professional: t("tierProfessional"),
        enterprise: t("tierEnterprise"),
      }[tier] ?? tier);

      const statusLabel = (status: string | null) => {
        if (!status) return "";
        return ({
          trialing: t("subscription.statusBadge.trialing"),
          active: t("subscription.statusBadge.active"),
          past_due: t("subscription.statusBadge.pastDue"),
          canceled: t("subscription.statusBadge.canceled"),
        }[status] ?? status);
      };

      const metricLabel = (key: MetricKey) => ({
        ai_summary: t("subscription.aiSummaryLabel"),
        proposal_draft: t("subscription.proposalDraftLabel"),
        compliance_check: t("subscription.complianceCheckLabel"),
      }[key]);

      return (
        <div className="mx-auto max-w-3xl p-6 space-y-8">
          <h1 data-testid="subscription-page-title" className="text-2xl font-bold">
            {t("subscription.pageTitle")}
          </h1>

          {/* ── Current Plan Card ─────────────────────────────────────────── */}
          <section className="rounded-lg border bg-card p-5 space-y-3">
            {subLoading ? (
              <Skeleton className="h-8 w-48" />
            ) : subscription && (
              <>
                <div className="flex items-center gap-3">
                  <span data-testid="subscription-tier" className="text-xl font-semibold">
                    {tierLabel(subscription.tier)}
                  </span>
                  {subscription.status && (
                    <Badge
                      data-testid="subscription-status-badge"
                      variant={STATUS_VARIANT[subscription.status] ?? "secondary"}
                    >
                      {statusLabel(subscription.status)}
                    </Badge>
                  )}
                </div>
                {subscription.is_trial && subscription.trial_end && (
                  <p data-testid="subscription-trial-end" className="text-sm text-amber-600">
                    {t("trialEndsOn", { date: new Date(subscription.trial_end).toLocaleDateString() })}
                  </p>
                )}
                {subscription.current_period_end && (
                  <p data-testid="subscription-billing-period" className="text-sm text-muted-foreground">
                    {t("subscription.billingPeriodLabel")}{": "}
                    {new Date(subscription.current_period_end).toLocaleDateString()}
                  </p>
                )}
                {/* Manage Subscription — Stripe Customer Portal */}
                <div className="pt-2">
                  <Button
                    onClick={() => portalMutation.mutate()}
                    disabled={portalMutation.isPending || !subscription.stripe_customer_id}
                    data-testid="manage-subscription-btn"
                    variant="outline"
                  >
                    {portalMutation.isPending ? t("portalOpening") : t("manageBilling")}
                  </Button>
                </div>
              </>
            )}
          </section>

          {/* ── Usage Meters ──────────────────────────────────────────────── */}
          <section>
            <h2 className="text-lg font-medium mb-4">{t("subscription.usageMetersTitle")}</h2>
            <div data-testid="subscription-usage-meters" className="space-y-4">
              {usageLoading ? (
                <div data-testid="subscription-usage-loading" className="space-y-4">
                  {[0, 1, 2].map((i) => <Skeleton key={i} className="h-10 w-full" />)}
                </div>
              ) : usageData && METRIC_KEYS.map((key) => {
                const metric = usageData.usage[key];
                const isUnlimited = metric?.limit === null || metric?.limit === -1;
                const pct = isUnlimited ? 0 : usagePct(metric);
                return (
                  <div key={key} data-testid={`subscription-usage-${key}`} className="space-y-1">
                    <div className="flex justify-between text-sm">
                      <span>{metricLabel(key)}</span>
                      {isUnlimited ? (
                        <span data-testid={`subscription-usage-${key}-unlimited`} className="text-muted-foreground">
                          {t("subscription.unlimited")}
                        </span>
                      ) : (
                        <>
                          <span data-testid={`subscription-usage-${key}-text`} className="font-medium">
                            {metric?.consumed ?? 0} / {metric?.limit ?? 0}
                          </span>
                          <span data-testid={`subscription-usage-${key}-pct`} className="text-muted-foreground ml-2">
                            {pct}%
                          </span>
                        </>
                      )}
                    </div>
                    {!isUnlimited && (
                      <div className="h-2 w-full rounded-full bg-slate-200">
                        <div
                          data-testid={`subscription-usage-${key}-bar`}
                          className={`h-full rounded-full transition-all ${barColor(pct)}`}
                          style={{ width: `${pct}%` }}
                        />
                      </div>
                    )}
                  </div>
                );
              })}
            </div>
          </section>

          {/* ── Billing History Table ─────────────────────────────────────── */}
          <section>
            <h2 className="text-lg font-medium mb-4">{t("subscription.billingHistoryTitle")}</h2>
            <div data-testid="subscription-billing-history">
              {invoicesLoading ? (
                <Skeleton className="h-24 w-full" />
              ) : !invoices?.length ? (
                <EmptyState
                  data-testid="subscription-billing-empty"
                  title={t("subscription.noInvoices")}
                />
              ) : (
                <Table>
                  <TableHeader>
                    <TableRow>
                      <TableHead>{t("subscription.invoiceDate")}</TableHead>
                      <TableHead>{t("subscription.invoiceStatus")}</TableHead>
                      <TableHead>{t("subscription.invoiceTerms")}</TableHead>
                    </TableRow>
                  </TableHeader>
                  <TableBody>
                    {invoices.map((inv) => (
                      <TableRow key={inv.id}>
                        <TableCell>
                          {new Date(inv.created_at).toLocaleDateString()}
                        </TableCell>
                        <TableCell>
                          <Badge variant={inv.status === "paid" ? "default" : "secondary"}>
                            {inv.status}
                          </Badge>
                        </TableCell>
                        <TableCell>{inv.payment_terms}</TableCell>
                      </TableRow>
                    ))}
                  </TableBody>
                </Table>
              )}
            </div>
          </section>
        </div>
      );
    }
    ```

- [x] **Task 9 — Replace `settings/billing/page.tsx` shell** (AC: 1)
  - [x] 9.1 Replace `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx` with:
    ```typescript
    import { SubscriptionManagementPage } from "./components/SubscriptionManagementPage";

    export const metadata = {
      title: "Subscription & Billing — EU Solicit",
      description: "Manage your subscription, monitor usage, and view billing history.",
    };

    export default function BillingSettingsRoute() {
      return <SubscriptionManagementPage />;
    }
    ```
    **Important**: Remove the `"use client"` directive from `page.tsx` — the shell is a server component that imports the client component.

### ATDD Tests

- [x] **Task 10 — Create `__tests__/subscription-management-s8-13.test.ts`** (AC: 9)
  - [x] 10.1 Create `eusolicit-app/frontend/apps/client/__tests__/subscription-management-s8-13.test.ts`
  - [x] 10.2 Follow the Vitest + Node `fs` pattern from `__tests__/pricing-page-s8-12.test.ts`
  - [x] 10.3 Implement describe blocks:

    **File structure**
    - Assert `app/[locale]/(protected)/settings/billing/page.tsx` exists and does NOT contain `"use client"` (server shell)
    - Assert `app/[locale]/(protected)/settings/billing/components/SubscriptionManagementPage.tsx` exists
    - Assert `app/[locale]/(protected)/components/TrialBanner.tsx` exists
    - Assert `lib/api/billing.ts` exists
    - Assert `lib/queries/use-billing.ts` exists

    **Backend endpoint**
    - Assert `services/client-api/src/client_api/api/v1/billing.py` contains `"/invoices"` string
    - Assert file contains `get_invoices` function name
    - Assert file contains `InvoiceDTO` class name
    - Assert file contains `EnterpriseInvoice` import

    **Subscription management component testids**
    - Assert `SubscriptionManagementPage.tsx` contains `data-testid="subscription-page-title"`
    - Assert file contains `data-testid="subscription-tier"`
    - Assert file contains `data-testid="subscription-status-badge"`
    - Assert file contains `data-testid="subscription-billing-period"`
    - Assert file contains `data-testid="subscription-usage-meters"`
    - Assert file contains `subscription-usage-ai_summary`, `subscription-usage-proposal_draft`, `subscription-usage-compliance_check`
    - Assert file contains `data-testid="subscription-billing-history"`
    - Assert file contains `data-testid="manage-subscription-btn"`

    **Trial banner testids**
    - Assert `TrialBanner.tsx` contains `data-testid="trial-banner"`
    - Assert file contains `data-testid="trial-banner-days"`
    - Assert file contains `data-testid="trial-banner-upgrade-cta"`
    - Assert file contains `data-testid="trial-banner-close"`
    - Assert file contains `DISMISS_KEY` or `eusolicit-trial-banner-dismissed`

    **API module exports**
    - Assert `lib/api/billing.ts` contains `getSubscription`
    - Assert file contains `getUsage`
    - Assert file contains `getInvoices`
    - Assert file contains `SubscriptionStatus`
    - Assert file contains `InvoiceRecord`
    - Assert file contains `UsageResponse`
    - Assert file does NOT contain `"use client"` directive

    **Hook exports**
    - Assert `lib/queries/use-billing.ts` contains `useSubscription`
    - Assert file contains `useUsage`
    - Assert file contains `useInvoices`
    - Assert file contains `useTiers` (existing — regression guard)
    - Assert file contains `refetchOnWindowFocus: true` (AC5 from Story 8-7 — portal return refresh)

    **Protected layout**
    - Assert `app/[locale]/(protected)/layout.tsx` contains `TrialBanner`
    - Assert file contains `"billing", "subscription"` query key string

    **i18n key completeness**
    - Assert both `messages/en.json` and `messages/bg.json` contain `billing.trialBanner.daysRemaining`
    - Assert both contain `billing.trialBanner.upgradeNow`
    - Assert both contain `billing.trialBanner.zeroDay`
    - Assert both contain `billing.subscription.usageMetersTitle`
    - Assert both contain `billing.subscription.aiSummaryLabel`
    - Assert both contain `billing.subscription.proposalDraftLabel`
    - Assert both contain `billing.subscription.complianceCheckLabel`
    - Assert both contain `billing.subscription.noInvoices`
    - Assert key counts match between en and bg under `billing.trialBanner` and `billing.subscription`

### Review Follow-ups (AI)

- [x] **[AI-Review][High]** Add Amount column (sum of `line_items`) to billing history table per AC3 (Finding #1 — SubscriptionManagementPage.tsx)
  - [x] Add `invoiceAmount` helper to sum `line_items[*].amount` with defensive handling for missing/empty arrays and non-numeric amount values
  - [x] Add fourth `<TableHead>{t("subscription.invoiceAmount")}</TableHead>` and corresponding `<TableCell>` rendering the summed amount as localized currency
  - [x] Add `billing.subscription.invoiceAmount` i18n key to `en.json` ("Amount") and `bg.json` ("Сума")
  - [x] Add ATDD assertions: `subscription.invoiceAmount` key present in component; `line_items` + `reduce|invoiceAmount(` pattern present; `billing.subscription.invoiceAmount` added to `REQUIRED_SUBSCRIPTION_KEYS` so both locales are checked

## Dev Notes

### Architecture: Mostly Frontend Enhancement + One Backend Endpoint

S08.13 is a frontend-heavy story with a single backend addition. The key implementation challenge is that `settings/billing/page.tsx` already exists from S08.06 and S08.07 — it must be **refactored**, not created from scratch. Do NOT accidentally drop the existing portal mutation logic (from S08.07).

### Existing `settings/billing/page.tsx` — Preserve These Features

The current page (from S08.06/S08.07) contains:
- `useQuery` for subscription status (inline `queryFn` — migrate this to `useSubscription()` from `use-billing.ts`)
- `useMutation` for Stripe Customer Portal session (`POST /api/v1/billing/portal/session`) — KEEP this
- `useMutation` for Stripe Checkout Session (`POST /api/v1/billing/checkout/session`) — **REMOVE**: the upgrade flow is now handled via the pricing page + `billing.trialBanner.upgradeNow` CTA (which links to `/settings/billing` without triggering a checkout directly)
- Toast error for portal error — KEEP

The refactored page should replace the tier selection grid (upgrade buttons) with usage meters and billing history. The "Manage Subscription" portal button stays.

**Why remove the checkout mutation**: The S08.12 pricing page and S08.13 trial banner CTA both route to `/settings/billing` for upgrades. The Checkout Session flow is triggered from the pricing page's CTA or through the Stripe Customer Portal link — not from the subscription management page directly.

### Trial Banner: No TopBar Modification Required

The `TopBar` component in `packages/ui` does NOT need to be modified. The banner is injected as a React Fragment sibling alongside `<TopBar>` inside the `topbar` slot prop of `<AppShell>`. Since `AppShell` renders `{topbar}` as-is:

```tsx
<div className="flex flex-col flex-1 min-w-0 ...">
  {topbar}     {/* <-- receives Fragment: <TopBar> + <TrialBanner> */}
  <main>
    {children}
  </main>
</div>
```

The Fragment naturally stacks `<TopBar>` (sticky h-16) and `<TrialBanner>` (full-width amber strip) vertically. No layout changes needed to `AppShell.tsx` or `TopBar.tsx`.

### Query Cache Sharing: No Double Fetch

The protected layout and `SubscriptionManagementPage` both use `queryKey: ["billing", "subscription"]`. TanStack Query deduplicates identical queries — only one network request is made per stale period. The layout's background query populates the cache; the billing page consumes the cached data immediately.

### Usage Data: `/api/v1/subscription/usage` Response Shape

From Story 8.8 AC5, the endpoint returns:
```json
{
  "billing_period": "2026-04",
  "usage": {
    "ai_summary":       {"consumed": 5,  "limit": 50, "remaining": 45},
    "proposal_draft":   {"consumed": 2,  "limit": 20, "remaining": 18},
    "compliance_check": {"consumed": 1,  "limit": 10, "remaining": 9}
  }
}
```
When `limit == -1` (Enterprise unlimited), the API returns `"limit": null` and `"remaining": null`. The `UsageMetric` interface uses `limit: number | null` to model this. Display "Unlimited" when `limit === null || limit === -1`.

### Backend: `EnterpriseInvoice` Model Already Exists

The `EnterpriseInvoice` ORM model is at `client_api.models.enterprise_invoice`. Fields relevant to `InvoiceDTO`: `id` (UUID), `stripe_invoice_id` (str), `payment_terms` ("NET 30" / "NET 60"), `line_items` (JSONB list), `status` ("open" / "paid"), `created_at` (datetime), `paid_at` (datetime | None).

Do NOT add the `EnterpriseInvoice` import to `billing.py` if it already exists — check first.

### Billing History: Enterprise-Only Data in Practice

Most companies will have an empty billing history table (only Enterprise-tier customers get custom invoices from Story 8.11). This is expected — show the `<EmptyState>` for most users. The empty state is not an error condition.

### File Structure

```
eusolicit-app/
├── services/client-api/src/client_api/api/v1/
│   └── billing.py                                           MODIFIED — add GET /invoices + InvoiceDTO
├── frontend/apps/client/
│   ├── app/[locale]/(protected)/
│   │   ├── layout.tsx                                       MODIFIED — add TrialBanner + subscription query
│   │   ├── components/
│   │   │   └── TrialBanner.tsx                              NEW — trial countdown banner component
│   │   └── settings/billing/
│   │       ├── page.tsx                                     MODIFIED — server shell → imports SubscriptionManagementPage
│   │       └── components/
│   │           └── SubscriptionManagementPage.tsx           NEW — full management dashboard
│   ├── lib/
│   │   ├── api/
│   │   │   └── billing.ts                                   MODIFIED — add interfaces + getSubscription/getUsage/getInvoices
│   │   └── queries/
│   │       └── use-billing.ts                               MODIFIED — add useSubscription/useUsage/useInvoices
│   ├── messages/
│   │   ├── en.json                                          MODIFIED — add billing.trialBanner.* + billing.subscription.*
│   │   └── bg.json                                          MODIFIED — add billing.trialBanner.* + billing.subscription.*
│   └── __tests__/
│       └── subscription-management-s8-13.test.ts            NEW — ATDD structural tests
```

**Do NOT create or modify:**
- `settings/billing/success/page.tsx` and `cancel/page.tsx` — Stripe Checkout return pages from S08.06 (untouched)
- `AppShell.tsx` or `TopBar.tsx` in `packages/ui` — no structural changes needed
- Any other service beyond `client-api` billing endpoint
- Story 8.14 work (Redis Streams `subscription.changed` consumer cache invalidation) — separate story

### Critical Mistakes to Prevent

1. **Do NOT drop the portal mutation** from the billing page. The `POST /api/v1/billing/portal/session` call and `window.open(portal_url, "_blank")` must be preserved. Losing it breaks Story 8.7.

2. **Do NOT add `"use client"` to `billing.ts` API module** — it is a pure async function module, usable server-side and client-side.

3. **Do NOT use `useQuery` inline in `layout.tsx` with a new `queryFn` definition** — import `getSubscription` from `lib/api/billing` and pass it as `queryFn`. This ensures the query key matches the billing page's query for cache sharing.

4. **Do NOT break existing `useTiers()` hook** when updating `use-billing.ts` — it is used by the public pricing page (`app/[locale]/pricing/`). Add new hooks below the existing ones.

5. **Do NOT block layout render on subscription fetch** — the layout's subscription query uses `{ retry: 1 }` without `suspense: true`. The trial banner simply does not render until data arrives (no loading spinner in topbar).

6. **Do NOT modify the `(protected)/layout.tsx` `topbar` slot naively** — wrap BOTH `<TopBar ...>` and the conditional `<TrialBanner ...>` in a React Fragment `<>...</>`. The Fragment is what gets passed as the `topbar` prop. Miss this and the layout breaks.

7. **Merging i18n keys**: The existing `billing` namespace already has `trialEndsOn`, `manageBilling`, `portalOpening`, `portalError`, `tierFree`, `tierStarter`, etc. Add `trialBanner` and `subscription` as nested objects. Do NOT replace or overwrite the existing `billing` object in either JSON file.

8. **`page.tsx` must be a server component**: The billing page shell (`page.tsx`) should NOT have `"use client"`. The current file has `"use client"` at line 1 (from S08.06) — this must be removed. The client-side logic moves into `SubscriptionManagementPage.tsx`.

### Test Coverage Map (from Epic 8 Test Design)

| Test Design ID | Priority | Scenario | Coverage in This Story |
|---|---|---|---|
| **AC-14** (trial banner) | P1 (bundled) | Trial banner shows days remaining + upgrade CTA during trialing status | `TrialBanner.tsx` component with `trial-banner-days`, `trial-banner-upgrade-cta` testids; ATDD structural assertions |
| **AC-15** (subscription management page) | P1 | Subscription management page: tier, usage meters, portal link, billing history | `SubscriptionManagementPage.tsx` with testids; `useSubscription`, `useUsage`, `useInvoices` hooks; E2E expansion left to ATDD phase |

Note from test design (AC-14 coverage): "8.13-E2E (covered in 8.6-E2E-001 setup)" — the epic test design bundles trial banner E2E testing with the checkout flow test. The ATDD structural tests added in Task 10 cover file-level and component-level validation.

### Previous Story Learnings (S08.12)

From Story 8.12 implementation notes:
- The ATDD tests use Vitest + Node `fs` pattern (no DOM rendering — purely structural file assertions)
- Both locale JSON files need to use `python3 -m json.tool` validation after editing
- The `lib/api/billing.ts` module must NOT contain `"use client"` — the ATDD test asserts its absence
- The explicit tier name lookup map approach (not template literals) avoids TypeScript type errors
- The `useTiers` hook uses `staleTime: 300_000` (5 min) — higher than subscription (1 min) since pricing data changes rarely

### References

- [Source: eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md#S08.13] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-08.md#AC-14, AC-15] — trial banner + subscription management page test coverage
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx] — protected layout (add TrialBanner here)
- [Source: eusolicit-app/frontend/packages/ui/src/components/app-shell/AppShell.tsx] — `topbar` slot is a React.ReactNode; Fragment injection pattern
- [Source: eusolicit-app/frontend/packages/ui/src/components/app-shell/TopBar.tsx] — no modifications required
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx] — existing file to refactor (S08.06/S08.07 implementation)
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/billing.py] — add GET /invoices here; existing pattern for `_billing_session()` and `require_auth`
- [Source: eusolicit-app/services/client-api/src/client_api/models/enterprise_invoice.py] — EnterpriseInvoice ORM model fields
- [Source: eusolicit-app/services/client-api/src/client_api/models/subscription.py] — Subscription model (tier, status, is_trial, trial_end, current_period_end, stripe_customer_id)
- [Source: eusolicit-docs/implementation-artifacts/8-7-stripe-customer-portal-integration.md] — portal mutation pattern to preserve
- [Source: eusolicit-docs/implementation-artifacts/8-8-usage-metering-with-redis-counters-stripe-sync.md#AC5] — /subscription/usage response shape
- [Source: eusolicit-docs/implementation-artifacts/8-12-pricing-page-tier-comparison-ui.md] — billing.ts + use-billing.ts creation pattern; ATDD structural test pattern
- [Source: eusolicit-app/frontend/apps/client/lib/api/billing.ts] — add new interfaces and functions here (file already exists from S08.12)
- [Source: eusolicit-app/frontend/apps/client/lib/queries/use-billing.ts] — add new hooks here (file already exists from S08.12)
- [Source: eusolicit-app/frontend/apps/client/messages/en.json#billing] — existing billing keys; merge trialBanner + subscription sub-objects
- [Source: eusolicit-app/frontend/apps/client/__tests__/pricing-page-s8-12.test.ts] — ATDD test pattern to follow
- [Source: eusolicit-app/frontend/packages/ui/index.ts] — available components: Badge, Button, Skeleton, EmptyState, Table/TableHeader/TableBody/TableCell/TableRow/TableHead, useToast

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

None — implementation completed without errors. All 182 ATDD tests pass on first run after static testid fix (METRIC_TESTIDS lookup table replaced template literal approach).

### Completion Notes List

- **Static testid requirement**: ATDD tests use Node `fs` string-contains checks, not DOM rendering. Template literals like `` data-testid={`subscription-usage-${key}`} `` do not appear as literal strings in source. Fixed by adding `METRIC_TESTIDS` lookup table with all 15 static testid strings (3 metrics × 5 types: wrapper, text, pct, bar, unlimited). Component uses `data-testid={ids.wrapper}` etc.
- **Portal mutation preserved**: S08.07 `POST /api/v1/billing/portal/session` + `window.open` logic retained verbatim in `SubscriptionManagementPage.tsx`. Checkout mutation removed as directed.
- **React Fragment in topbar slot**: `layout.tsx` topbar prop wraps `<TopBar>` and `<TrialBanner>` in `<>...</>` — no AppShell or TopBar modifications needed.
- **Query cache sharing**: Both `layout.tsx` and `SubscriptionManagementPage` use `queryKey: ["billing", "subscription"]` — TanStack Query deduplicates; single network request per stale period.
- **SSR safety**: `TrialBanner` uses `mounted` state + `useEffect` for sessionStorage access — no hydration mismatch.
- **page.tsx server component**: `"use client"` directive removed from billing `page.tsx`; client logic isolated in `SubscriptionManagementPage.tsx`.
- ✅ Resolved review finding [High]: AC3 gap — Amount column added to billing history table via `invoiceAmount()` helper that reduces over `inv.line_items` (defensive against missing/non-numeric amount fields); rendered as a fourth `<TableCell>` formatted via `toLocaleString(locale, { style: "currency", currency: "EUR" })`. New i18n key `billing.subscription.invoiceAmount` added to both locales ("Amount" / "Сума"). ATDD suite extended with two new assertions (`subscription.invoiceAmount` source contains + `line_items` reduce pattern) plus inclusion in `REQUIRED_SUBSCRIPTION_KEYS` — total test count 186 (was 182), all passing.

### File List

- `eusolicit-app/services/client-api/src/client_api/api/v1/billing.py` — MODIFIED: added `InvoiceDTO` Pydantic model, `get_invoices` endpoint, updated module docstring
- `eusolicit-app/frontend/apps/client/lib/api/billing.ts` — MODIFIED: added `SubscriptionStatus`, `UsageMetric`, `UsageResponse`, `InvoiceRecord` interfaces; added `getSubscription`, `getUsage`, `getInvoices` functions
- `eusolicit-app/frontend/apps/client/lib/queries/use-billing.ts` — MODIFIED: added `useSubscription`, `useUsage`, `useInvoices` hooks
- `eusolicit-app/frontend/apps/client/messages/en.json` — MODIFIED: added `billing.trialBanner.*` and `billing.subscription.*` keys
- `eusolicit-app/frontend/apps/client/messages/bg.json` — MODIFIED: added `billing.trialBanner.*` and `billing.subscription.*` keys (Bulgarian translations)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/components/TrialBanner.tsx` — NEW: trial countdown banner component with session dismiss
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx` — MODIFIED: added subscription query + TrialBanner integration in topbar React Fragment
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/billing/components/SubscriptionManagementPage.tsx` — NEW: full subscription management dashboard
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx` — MODIFIED: replaced with server component shell importing SubscriptionManagementPage
- `eusolicit-app/frontend/apps/client/__tests__/subscription-management-s8-13.test.ts` — NEW (pre-written): 182 ATDD structural tests, all passing

### Change Log

| Date | Version | Description | Author |
|---|---|---|---|
| 2026-04-19 | 1.0 | Story 8.13 implementation complete — all 182 ATDD tests pass | claude-sonnet-4-5 |
| 2026-04-19 | 1.1 | Addressed code review findings — 1 blocking item resolved (AC3 Amount column). ATDD count 182 → 186, all passing. | claude-sonnet-4-5 |

## Senior Developer Review

**Reviewer:** Automated adversarial review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-04-19
**Outcome:** Changes Requested (blocking finding resolved 2026-04-19 — see Review Follow-ups (AI))
**Scope reviewed:** `billing.py` (backend invoices endpoint), `TrialBanner.tsx`, `SubscriptionManagementPage.tsx`, `(protected)/layout.tsx`, `lib/api/billing.ts`, `lib/queries/use-billing.ts`, `messages/{en,bg}.json`, `subscription-management-s8-13.test.ts`.

### Findings — Blocking

1. ✅ **[RESOLVED 2026-04-19]** **AC3 gap — Amount column missing from billing history table** (`SubscriptionManagementPage.tsx` lines 242–265).
   AC3 explicitly lists **four** columns: `Date`, `Status`, `Payment Terms`, **`Amount` (from `line_items` sum)`**. The implementation renders only three columns (Date, Status, Payment Terms). `line_items` is exposed in `InvoiceRecord` and returned by the backend, but never summed or displayed. The ATDD test file also omits any assertion for an Amount column, so the test suite does not catch this regression.
   - **Remediation:** Add a fourth `TableHead`/`TableCell` that renders the sum of amounts from `inv.line_items`. Add an i18n key `billing.subscription.invoiceAmount` in both `en.json` and `bg.json`. Add an assertion in `subscription-management-s8-13.test.ts` that the component source contains `invoiceAmount` and a sum/reduce over `line_items`. The summing logic should tolerate `line_items` being an empty array or missing `amount` fields (defensive).
   - `DEVIATION: AC3 billing history table renders 3 of 4 required columns — Amount column is missing.`
   - `DEVIATION_TYPE: MISSING_REQUIREMENT`
   - `DEVIATION_SEVERITY: blocking`

### Findings — Non-blocking (recommend fixing before merge)

2. **Config duplication between `(protected)/layout.tsx` and `useSubscription()` hook.** The layout declares its own inline `useQuery({ queryKey: ["billing", "subscription"], queryFn: getSubscription, staleTime: 60_000, retry: 1 })` without `refetchOnWindowFocus: true`, whereas the `useSubscription()` hook in `use-billing.ts` does enable focus refetch. Because TanStack Query evaluates each hook's options independently, the two callsites can drift. When the user returns from the Stripe portal tab while viewing a non-billing page, the trial banner state will not refresh. Replace the inline `useQuery` in `layout.tsx` with a direct call to `useSubscription()` so that config lives in a single place (as the dev notes already recommend).

3. **Duplicate `role="banner"` landmark.** `TrialBanner.tsx` sets `role="banner"` and the existing `TopBar` (which is a header landmark) is rendered as a sibling. The WAI-ARIA authoring spec permits only one `banner` landmark per page; duplicating it reduces usefulness for assistive-tech users. The AC prescribes `role="banner"`, so this is a spec inherited-defect — flag to product/UX to consider `role="status"` or removing the role (the amber strip does not need a landmark role to be exposed).

4. **`usagePct` treats `limit === 0` as "no limit" (renders 0%).** `if (!metric.limit) return 0;` short-circuits when limit is `0`, `null`, or `undefined`. The unlimited check upstream only triggers on `null | -1`, so a literal `0` limit would silently render "0 / 0" with a 0% bar. This is an unlikely data shape, but worth guarding with `metric.limit == null || metric.limit < 0 || metric.limit === 0` handling (render "N/A" or treat as unlimited per product decision).

5. **Invalid `trial_end` → NaN days.** `TrialBanner.tsx` lines 29–32 trusts `new Date(trialEnd).getTime()` to produce a finite number. Malformed ISO strings yield NaN, and `Math.max(0, Math.ceil(NaN))` is `NaN`. The banner would render "NaN days remaining in your Professional trial". Add `Number.isFinite` guard and render the zero-day branch as a fallback.

6. **`subscription-trial-end` testid in paragraph but AC wording.** Minor: AC1 asks for `data-testid="subscription-trial-end"` rendering the formatted trial end date. Implementation renders it as `t("trialEndsOn", { date: ... })`. The i18n key `trialEndsOn` is from a prior story and its string template is not re-verified in the ATDD tests. Low risk.

7. **Portal popup may be blocked.** `window.open(data.portal_url, "_blank")` executes after an async network call, which many browsers treat as non-user-initiated and block. This is pre-existing from S08.07, but since `SubscriptionManagementPage.tsx` is new code that re-implements the pattern, it's worth considering `window.open` synchronously with a placeholder tab and assigning the URL once the mutation resolves, or rendering the `portal_url` into an `<a href>` the user can click.

### Findings — Tests

8. **ATDD suite is static-only.** The 182 assertions are all file-string checks (no DOM render tests). They verify presence of testid strings, interface names, and i18n keys but cannot detect:
   - Rendering logic errors (wrong branch in the loading/error/empty states).
   - The AC3 Amount column gap above.
   - TrialBanner countdown correctness (off-by-one in `Math.ceil`).
   Consider adding at least one `@testing-library/react` render test for `TrialBanner` (countdown math, zero-day branch, sessionStorage dismiss) and one for `SubscriptionManagementPage` (loading → data → empty transitions) before the next story in this epic.

9. **No backend test for `GET /api/v1/billing/invoices`.** The story notes that the endpoint is "untested at runtime"; the ATDD file verifies the route string exists but does not exercise the endpoint. Given the tracked changes already include `services/client-api/tests/integration/` fixtures, adding one happy-path + one empty-list integration test is cheap and would surface schema regressions.

### Positives

- SSR safety on `TrialBanner` (mounted flag + sessionStorage in `useEffect`) is correctly handled.
- Server/client split of `page.tsx` → `SubscriptionManagementPage.tsx` is clean; no `"use client"` leaked to the shell.
- Checkout mutation correctly removed per dev notes; portal mutation preserved.
- React Fragment injection into `AppShell`'s `topbar` slot avoids modifying shared UI packages.
- i18n parity verified: all required keys exist in both en and bg with matching leaf counts.
- `InvoiceDTO` uses `list[...]` return type so empty history returns `[]`, not 404 — aligns with AC6.

### Decision

**REVIEW: Changes Requested** — Finding #1 (missing Amount column) is a direct AC3 miss and must be implemented before this story can be closed. Findings #2–#9 are recommended improvements but are not individually blocking.

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T06:15:58Z (session 1284d3c6-c46a-40c7-adce-cf9d522bc7fb)

- AC3 billing history table renders 3 of 4 required columns — Amount column (sum of line_items) is missing from SubscriptionManagementPage.tsx and not asserted in ATDD suite.` _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- AC3 billing history table renders 3 of 4 required columns — Amount column (sum of line_items) is missing from SubscriptionManagementPage.tsx and not asserted in ATDD suite.` _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
