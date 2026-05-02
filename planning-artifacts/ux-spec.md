---
title: "EU Solicit — UX Specification"
project: eusolicit
author: Deb
date: 2026-04-28
status: Draft v2
stepsCompleted: [1, 2, 3, 4, 5, 6, 7]
lastStep: 7
sources:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/EU_Solicit_UX_Supplement_v1.md
  - eusolicit-docs/EU_Solicit_Requirements_Brief_v5.md
  - eusolicit-docs/EU_Solicit_Solution_Architecture_v5.md
  - eusolicit-docs/project-context.md
  - eusolicit-docs/planning-artifacts/project-context.md
inputDocuments:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/EU_Solicit_Requirements_Brief_v5.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/EU_Solicit_UX_Supplement_v1.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/project-context.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/project-context.md"
---

# EU Solicit — UX Specification

**Author:** Deb
**Date:** 2026-04-28
**Status:** Draft v2 (consolidated; supersedes the prior in-progress draft of 2026-04-27)

---

## 1. Purpose & Scope

This document defines the User Experience (UX) specification for **EU Solicit**, a multi-tenant SaaS platform that automates the lifecycle of EU public procurement and grant applications via a multi-agent AI pipeline. It is the primary UX contract for engineering, design, product, and QA across **Phase 1 (MVP)**, with structural decisions extending into **Phase 2 (Growth)** and **Phase 3 (Expansion)**.

It covers, in order:

1. Vision and design principles
2. **Personas** — primary, secondary, tertiary, and operational
3. **Information architecture** and the screen inventory
4. **Primary user journeys** — step-by-step, with screens, system actions, and states
5. **Key screens and their states** (empty, loading, streaming, error, success, paywalled, restricted)
6. **Cross-cutting UX patterns** — AI operations, notifications, search/filter, mobile, billing, i18n, time
7. **Accessibility** — WCAG 2.1 AA conformance plan and testing gates
8. **Visual foundations and component direction**
9. **Performance / UX budgets**
10. **Risks, open questions, and MVP acceptance checklist**

It complements `EU_Solicit_UX_Supplement_v1.md` (canonical for granular tokens, table specs, and component-level details) and supersedes any earlier `ux-spec.md` draft for MVP scope.

---

## 2. Vision & Design Principles

### 2.1 Product vision

EU Solicit transforms a chaotic, multi-week, paper-heavy bidding process into a streamlined, AI-assisted, collaborative workflow — making high-quality bid response accessible to organizations of every size, from solo consultants to enterprise firms.

### 2.2 Design principles

1. **Clarity over cleverness.** Procurement is high-stakes and time-pressured. Every screen reveals what the user must do next within ~3 seconds; no decorative chrome competes with content.
2. **AI as a co-pilot, never an autopilot.** Users see the AI's reasoning, sources, and confidence; nothing leaves the platform without an explicit user confirmation. Every AI output is reviewable and reversible.
3. **Progressive disclosure.** A 150-page tender condenses to a one-page summary, expands to a structured checklist, then drills to underlying clauses on demand. Complexity is absorbed, not flattened.
4. **Consistent, content-dense, professional.** A Linear / Vercel / Notion-class aesthetic: monochromatic slate base, single indigo accent, semantic colors reserved for status. One filter pattern, one table, one AI state machine, one empty-state template — applied everywhere.
5. **Trust through transparency.** Every AI conclusion is sourced. Every mutation lands in an immutable audit log. Status, deadlines, and quotas are always visible.
6. **Tone of voice.** Direct, expert, encouraging. Bulgarian users see Bulgarian by default; English fallback is always available.

### 2.3 Critical experience moments

| Moment | What good feels like |
|---|---|
| **Aha** | Within 90 seconds of uploading a tender, the user sees a one-page AI summary, a structured checklist, and flagged risks — work that previously took two days. |
| **Confidence** | Compliance is green and the score simulator is at 90+. The user knows *why* and *what to fix* if it isn't. |
| **Collaboration** | Two contributors edit different sections in parallel without overwriting each other. Comments and @mentions feel like Google Docs, not email. |
| **Calm** | Two days before deadline, the proposal is done; the dashboard shows three other bids on track; no fire-drill. |

### 2.4 Anti-patterns to avoid

- Hidden or ambiguous save actions in a collaborative editor.
- Deep, nested navigation menus.
- Overuse of modals for complex tasks (prefer dedicated views or side panels).
- Color as the sole signal for status.
- Empty states that are dead ends.
- Generic document-editor framing — every feature is tailored to proposal writing.

---

## 3. Personas

### 3.1 Primary — Elena, the Overwhelmed Bid Manager *(Professional tier)*

| Attribute | Detail |
|---|---|
| Role | Bid manager at a mid-size Bulgarian consulting firm; owns proposal lifecycle from go/no-go to submission |
| Team | 5 contributors, 10–15 concurrent bids |
| Tech comfort | High; Office 365, Google Workspace, Asana |
| Languages | Bulgarian (primary), English (working) |
| Devices | Desktop primary; mobile for notifications and approvals |
| Goals | Win more bids; cut prep time per bid by 60%; never miss a mandatory document or deadline |
| Pains | Unstructured tender PDFs; coordination via email/spreadsheets; opaque scoring rubrics; last-minute compliance failures |
| UX needs | Multi-proposal pipeline, parallel collaboration, deadline-first surfacing, automated compliance gates, score simulator, version history |
| Success signal | Submits a confident, compliance-validated bid days early; sees a quantified ROI on the dashboard |

### 3.2 Secondary — Maria, the Cautious Solopreneur *(Free → Starter)*

| Attribute | Detail |
|---|---|
| Role | Independent environmental consultant; deep expertise, solo operation |
| Tech comfort | Moderate |
| Languages | Bulgarian |
| Devices | Laptop primary, phone for browsing |
| Goals | Find one or two well-fitted opportunities per quarter; avoid paying for tools she won't use |
| Pains | Government portals are unsearchable; consultants are unaffordable; she can't tell which opportunities she could win |
| UX needs | Generous discovery on the free tier, transparent paywalls with previews, one-click upgrade, single-user editor flows, no team-only clutter |
| Success signal | Discovers a fit-for-purpose tender on the free tier; upgrades to Starter for a single month to bid on it |

### 3.3 Tertiary — Marko, the Enterprise Director *(Enterprise tier)*

| Attribute | Detail |
|---|---|
| Role | Partner / Practice Lead at a large CEE consulting firm; manages 3 sub-tenants and 30+ users |
| Goals | White-labeled platform for clients; centralized governance; CRM/BI integration; strong audit trail |
| Pains | Cross-client data leakage risk; manual roll-up reporting; integrating data into Salesforce |
| UX needs | Workspace switcher, sub-tenant management, white-label theming, RBAC matrix UI, API key management, audit log explorer |

### 3.4 Contributor / Subject-Matter Expert (SME)

| Attribute | Detail |
|---|---|
| Role | Drafts technical, methodology, financial, or capability sections of a proposal |
| Goals | Distraction-free editing; contextual access to RFP requirements while writing |
| Pains | Overwriting other contributors' work; lacking context on which requirement they are addressing |
| UX needs | Split-pane editor with requirement context, explicit per-section locking, inline comments, AI assistance scoped to the active section |

### 3.5 Reviewer / Executive

| Attribute | Detail |
|---|---|
| Role | Decides go/no-go and provides final sign-off |
| Goals | Rapidly synthesize large RFPs; understand risk, win probability, ROI |
| Pains | Lack of time to read full RFPs |
| UX needs | AI-generated one-page executive summaries, Bid/No-Bid decision matrix, high-level risk reports, one-click approval |

### 3.6 Operational — Internal Platform Admin

| Attribute | Detail |
|---|---|
| Role | EU Solicit operator |
| Devices | Desktop only; corporate VPN |
| Goals | Keep crawlers healthy; keep compliance frameworks current; manage tenants and incidents |
| Success signal | Crawler error rate < 2%; compliance frameworks updated within 7 days of legal change |
| UX needs | Separate `admin.eusolicit.com` portal, IP-restricted, MFA-enforced; CRUD for compliance frameworks, crawler dashboards, tenant lifecycle, audit log viewer |

### 3.7 Integration — API Consumer Developer

| Attribute | Detail |
|---|---|
| Role | Developer at an Enterprise customer |
| Goals | Pipe scored opportunities into Salesforce / BI |
| Success signal | First successful API call within 30 minutes of receiving the key |
| UX needs | Self-serve API keys, embedded OpenAPI explorer, copyable code samples, rate-limit feedback, signed webhooks |

---

## 4. Information Architecture

### 4.1 Top-level navigation — Client app (`eusolicit.com`)

```
EU Solicit
├── Dashboard                 (default landing after login)
├── Opportunities             (search, filter, save, score)
│   └── Opportunity Detail    (overview, AI analysis, requirements, risks, documents, activity)
├── Proposals                 (pipeline of in-progress + submitted)
│   └── Proposal Editor       (split-pane: editor | inspector)
├── Grants                    (Phase 2: eligibility matcher, budget builder)
├── Analytics                 (Phase 2: ROI, team performance, pipeline forecast)
├── Calendar                  (deadlines + external calendar sync)
├── Content Library           (reusable blocks, ESPD profile, past performance)
├── Lessons Learned           (Phase 2: post-bid retrospectives)
└── Settings                  (Workspace · Members · Billing · Integrations · Profile · Notifications)
```

### 4.2 Top-level navigation — Admin app (`admin.eusolicit.com`)

```
admin.eusolicit.com  (VPN-restricted, MFA-required)
├── Tenants                   (list, detail, suspend, impersonate-with-audit)
├── Subscriptions & Billing Ops (refunds, manual extensions, invoice ops)
├── Compliance Frameworks     (CRUD with versioning + effective dates)
├── Crawler Status            (per-source health, error rate, selector diagnostics)
├── AI Agent Registry         (per-agent health, version, rollback)
├── System Health             (latency, error budgets, dependencies)
└── Audit Log                 (immutable; search, filter, export CSV)
```

### 4.3 Layout regions (Client app)

- **Left sidebar** — primary navigation, collapsible 256→64px, persisted per-user.
- **Top bar** — breadcrumbs, `Cmd+K` global command menu, notifications bell, language switcher (BG/EN), workspace switcher, user/tier badge.
- **Main content** — fluid width, max-width 1440px on `2xl`.
- **Right inspector** — contextual detail (e.g., live requirement extraction while editing); 380px on desktop, full-screen sheet on mobile.

### 4.4 Responsive tiers

| Tier | Width | Treatment |
|---|---|---|
| Mobile | < 768px | Bottom tab bar (5 icons + "More"); single column; sheets replace side panels; touch targets ≥ 44×44px; body text ≥ 16px; heavy editing redirects gracefully to desktop |
| Tablet | 768–1023px | Collapsible sidebar; 1–2 column layouts; inspector as right-side sheet (60vw) |
| Desktop | ≥ 1024px | Persistent sidebar, optional pinned right inspector; full data tables with sortable headers |

---

## 5. Primary User Journeys

Each journey is decomposed into discrete steps with **screens involved**, **AI/system actions**, **state transitions**, and **decision points**. Steps tagged **(MVP)** are in scope for Phase 1; **(P2)** are post-MVP; **(P3)** are vision.

### Journey J1 — Elena: Tender → Submitted Proposal *(MVP, primary value path)*

**Trigger:** New tender PDF received (email / portal link / direct upload).
**Outcome:** Compliance-validated, score-simulated proposal submitted with time to spare.

| # | Step | Screen | System / AI action | Notable states |
|---|---|---|---|---|
| 1 | Sign up via Google or email/password | `Auth — Sign Up` | Sends verification email; creates Workspace skeleton; auto-enrolls 14-day Professional trial | Default → Verification-pending → Verified |
| 2 | Onboard: company profile + sectors (CPV codes) + key personnel | `Onboarding Wizard` (3 steps) | Profile drives downstream relevance scoring + ESPD pre-fill | Empty → Filled → Saved |
| 3 | Lands on Dashboard; sees "Add your first opportunity" prompt | `Dashboard` | Surfaces relevance-scored opportunities and trial banner | Empty → Populated |
| 4 | Uploads tender PDF | `Opportunity → Upload` modal | Document-parser agent extracts text; summarizer + checklist + risk agents enqueue | Idle → Uploading → Parsing → Streaming summary |
| 5 | Sees streaming AI executive summary | `Opportunity Detail → Analysis tab` | Tokens stream into summary card; checklist renders progressively; risk clauses flagged | Streaming → Complete |
| 6 | Reviews requirements checklist & flagged risks | `Opportunity Detail → Requirements + Risks tabs` | Each row links to the exact clause in the source PDF | Loaded |
| 7 | Runs Bid/No-Bid decision matrix | `Opportunity Detail → Decision panel` | AI returns a scored recommendation across strategic-fit, win-probability, capacity | Estimating → Result |
| 8 | Clicks "Generate Proposal Draft" | `Proposal — New` | Draft-generator agent creates structured draft from profile + requirements | Generating (skeleton) → Streaming → Complete |
| 9 | Invites team & assigns sections (P2 collaboration; MVP single-user with comment-based assignment) | `Proposal Editor → Members panel` | RBAC + per-section assignment | — |
| 10 | Edits in rich-text editor (Tiptap); accepts/rejects AI suggestions | `Proposal Editor` | Autosave every 5s; version snapshot every 60s | Saving → Saved → Conflict (P2) |
| 11 | Runs compliance check | `Proposal Editor → Compliance panel` | Compliance agent validates against extracted checklist + framework rules | Idle → Running → Pass / Fail with row-level diagnostics |
| 12 | Fixes flagged gaps (missing attachment, page-limit overrun, etc.) | Compliance panel inline actions | Re-runs check incrementally | — |
| 13 | Runs Score Simulator | `Proposal Editor → Score panel` | Score agent emits 0–100 score plus 2–4 prioritized improvement suggestions | Estimating → Result |
| 14 | Applies suggestions, re-runs simulator until satisfied | Score panel | — | — |
| 15 | Sends for reviewer approval (P2) | `Proposal → Approval` | Reviewer notified; one-click approve/reject with comment | Pending → Approved / Changes-requested |
| 16 | Exports final PDF/DOCX | `Proposal Editor → Export modal` | Server-side export; watermark on free/trial | Generating → Download ready |
| 17 | Marks "Submitted" + records outcome | `Proposal Header → Status menu` | Status change writes to audit log; Lessons-Learned wizard prompted post-decision (P2) | Draft → In Review → Submitted → Won / Lost |

**Critical UX guarantees in J1**

- A new user can reach **Step 5 (AI summary visible)** in **< 5 minutes** from sign-up.
- First-draft generation produces a **first token within 500 ms** and a **complete draft within 2 minutes p95**, with skeletons that mirror final layout (no reflow).
- Compliance results are **explainable**: every fail surfaces (a) the rule, (b) the offending location, (c) a one-click jump-to-section or apply-fix action.

---

### Journey J2 — Maria: Free discovery → Paid first bid *(MVP, conversion path)*

**Trigger:** Maria hears about EU Solicit from a peer.
**Outcome:** She converts to Starter for a single month and submits her bid.

| # | Step | Screen | System action | Notable states |
|---|---|---|---|---|
| 1 | Email-only sign-up | `Auth — Sign Up` | Free account; company-profile wizard offered later, never blocking | — |
| 2 | Lands on Opportunities list (free view) | `Opportunities` | Full list with name, deadline, contracting authority, country, CPV; paid fields blurred with lock icon | Free-tier blurred state |
| 3 | Filters: country = BG, CPV = environment | `Opportunities → Filter sheet` | Filters apply instantly; result count updates | — |
| 4 | Sorts by deadline ascending | Same | — | — |
| 5 | Opens a tender; sees free-view detail | `Opportunity Detail (Free)` | Budget, full documents, AI summary locked behind paywall card with preview shadow | Paywalled state with "Unlock — Starter from €29/mo" CTA |
| 6 | Clicks Unlock → upgrade flow | `Billing → Upgrade` modal → Stripe Checkout redirect | Stripe-hosted (PCI scope minimized) | Redirecting → Paid → Returning |
| 7 | Returns to opportunity, fully unlocked | `Opportunity Detail (Starter)` | All fields render; AI summary + checklist generate | Streaming → Complete |
| 8 | Drafts proposal solo | `Proposal Editor (single-user mode)` | Same editor as Elena, with team-invite hidden | — |
| 9 | Runs compliance + export | Compliance + Export | Same as J1 step 11/16 | — |
| 10 | Submits manually outside platform; marks status "Submitted" | Proposal header | Audit log entry | — |

**Critical UX guarantees in J2**

- Free-tier users **never see broken/empty paid features**; locks are intentional and educational, not error-shaped.
- Upgrade returns the user to **the exact opportunity** they were viewing — no detour through a generic billing page.
- Downgrade and cancellation are **as discoverable as upgrade** (Settings → Billing) — a regulatory and trust requirement.
- Paywall previews show **a redacted shadow of value** (blurred summary, hidden budget) — never a blank wall.

---

### Journey J3 — Marko: Multi-tenant administration & API integration *(Phase 2/3)*

**Trigger:** Firm signs Enterprise contract; partners want Salesforce integration.
**Outcome:** Three sub-tenant workspaces are configured; API delivers scored opportunities into Salesforce.

| # | Step | Screen | System action |
|---|---|---|---|
| 1 | Logs in; opens Workspace switcher | Top-bar avatar → menu | Lists parent + child workspaces |
| 2 | Creates a sub-tenant for Client A | `Settings → Sub-Tenants → New` | Provisions isolated schema scope |
| 3 | Invites users; assigns roles via the RBAC matrix | `Settings → Members` | Email invites; RBAC matrix UI (roles × resources × permissions) |
| 4 | Applies white-label theme: logo, primary color, custom domain | `Settings → Branding` | Live preview; theme tokens saved; SSL provisioning |
| 5 | Generates an API key | `Settings → API & Webhooks → New key` | Key shown once, copyable; scopes selectable |
| 6 | Reads OpenAPI docs in-app | `Settings → API → Reference` | Embedded Swagger UI with try-it-out using a sandbox key |
| 7 | Configures opportunity webhook | `Settings → Webhooks` | Event: `opportunity.scored ≥ 80`; HMAC-signed payloads; retry policy visible |
| 8 | Reviews audit log | `Settings → Audit Log` | Immutable, filterable, exportable CSV |

---

### Journey J4 — Internal Admin: Compliance update & crawler triage

| # | Step | Screen | Action |
|---|---|---|---|
| 1 | Logs into `admin.eusolicit.com` (VPN + MFA) | `Admin Login` | MFA enforced; IP allow-list checked |
| 2 | Edits the Horizon Europe framework: adds 3 new rules with severity | `Admin → Compliance Frameworks → Edit` | Versioned save; effective-date controls; impact preview ("affects N in-flight proposals") |
| 3 | Notices BG AOP crawler error rate above threshold | `Admin → Crawler Status` | Sparkline + 24h logs; selector-mismatch indicator |
| 4 | Files an internal incident | `Admin → Incidents → New` | Linked to crawler; pages on-call |

---

### Journey J5 — Onboarding & ESPD profile build *(cross-cutting)*

Shared by J1 and J2.

1. **Step 1 — Company basics.** Name, registration number (auto-validated against the BG commercial registry where possible), VAT, address.
2. **Step 2 — Capabilities.** CPV codes (multi-select with hierarchical picker + search), past performance highlights (free-text + optional doc upload).
3. **Step 3 — Personnel.** Key staff with roles and certifications. Each entry feeds the ESPD generator.
4. **Step 4 (optional, dismissible).** Connect Google Calendar / Outlook for deadline sync (Pro+).
5. **Step 5.** Profile completeness meter (0–100%). Below 60% the platform shows non-blocking nudges; above 80% it unlocks the "ESPD auto-generate" CTA.

---

### Journey J6 — Reviewer one-click approval *(P2)*

1. Reviewer receives email + in-app notification: "Elena requested your approval on PROP-2026-0042."
2. Opens the proposal in **Reviewer mode** — read-only, with a sticky header summarizing scope, score, compliance status, and budget.
3. Toggles "Show changes since last review" diff.
4. Adds optional inline comments.
5. Clicks **Approve** or **Request changes**. Approval is logged; on Request-changes, the proposal returns to the bid manager with the comment thread.

---

### Journey J7 — Deadline triage on mobile *(MVP companion experience)*

1. Push or email at T-3 days: "PROP-2026-0042 deadline approaches."
2. User taps the link; lands on the mobile **Proposal Status** view.
3. Sees compliance status, owner, last edit, and outstanding tasks.
4. Adds a comment, reassigns a section, or flags as blocked.
5. For deep editing the screen surfaces a "Open on desktop for full editor" CTA (no silent degradation).

---

## 6. Screen Inventory & Key States

The following screens are in **MVP scope** unless tagged **(P2)** or **(P3)**. For each, the listed states are required and must be designed.

### 6.1 Authentication & Onboarding

| Screen | Required states |
|---|---|
| Sign Up (email + Google) | Default · Validating · Error · Verification-sent · Already-exists |
| Email Verification | Pending · Verified · Expired-link · Resend-cooldown |
| Login | Default · Bad-credentials · Locked-out · MFA-challenge (admin) |
| Forgot / Reset Password | Request · Sent · Reset form · Success · Token-invalid |
| Onboarding Wizard (3 steps) | Step 1 · Step 2 · Step 3 · Skipped · Completed |

### 6.2 Dashboard

States: Empty (zero opportunities saved) · Trial banner · Populated · Loading skeleton · Error reload.

Modules: deadline ribbon (next 14 days), top-5 scored opportunities, in-progress proposals, usage meter (AI runs, seats, storage), trial countdown if applicable.

### 6.3 Opportunities List ("My Matches")

States: Loading skeleton · Empty (no matches) · Populated · Filter-active chips · Free-tier blurred fields · Saved/starred subset · Bulk-action mode · Error.

Components: faceted filter sheet (CPV, country, region, budget range, deadline, status, source), saved searches, `Cmd+F` quick filter, sort menu, relevance score chip with tooltip explaining the score.

### 6.4 Opportunity Detail

States: Loading · Streaming AI summary · Complete · No-source-document · Compliance-framework-mismatch warning · Free-tier paywalled · Restricted (RBAC).

Tabs: **Overview** · **Analysis (AI)** · **Requirements** · **Risks** · **Documents** · **Activity** · **Decision** (Bid/No-Bid).

### 6.5 Proposal Editor (most complex screen)

```
┌─────────── Top toolbar: title · status · members · run AI · export ───────────┐
│┌──────────────┬──────────────────────────────┬──────────────────────────────┐│
││ Outline      │ Editor (Tiptap)              │ Inspector (tabbed)           ││
││ (sections)   │                              │ - Requirements coverage      ││
││              │                              │ - Compliance findings        ││
││              │                              │ - Score simulator            ││
││              │                              │ - Comments (P2)              ││
││              │                              │ - Versions                   ││
│└──────────────┴──────────────────────────────┴──────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────┘
```

States: Loading · Empty (new draft) · Generating (streaming first draft) · Editing · Saving · Saved · Offline-queued · Conflict (P2) · Locked-section (P2: by self / by other) · Compliance-running · Compliance-pass · Compliance-fail with diagnostics · Score-running · Score-result · Exporting · Submitted (read-only).

**AI generation in-line.** Streamed AI output appears with a distinct pale-indigo background; an overlay offers **Accept** / **Reject** / **Regenerate** until the user acts. After acceptance the highlight fades over 600ms.

### 6.6 Compliance Panel (inside Proposal Editor)

States: Idle · Running · Pass · Fail (rule-by-rule list with severity, location, fix-action) · Stale (content changed since last run) · Framework-out-of-date warning.

### 6.7 Score Simulator Panel

States: Idle · Estimating · Result (score 0–100 + suggestions) · Confidence-low banner · Re-estimate-after-edit prompt.

### 6.8 Requirements Coverage Panel

A vertical list of every extracted requirement with: status (Met / Partially Met / Missing / N/A), linked editor anchor, and an AI-suggested snippet.

### 6.9 ESPD Builder

States: Profile-incomplete (with checklist) · Generating · Preview · Edit overrides · Export XML+PDF · Validation-error (per-field).

### 6.10 Calendar

States: Empty · Populated · External-calendar-disconnected · Sync-error · Two-way-sync (Pro+).

### 6.11 Analytics (P2 initial set)

ROI Tracker · Team Performance · Pipeline Forecast · Win/Loss Reasons · Usage. Each chart has: Loading · Populated · Empty · Insufficient-data (need ≥ N datapoints) · Export-CSV.

### 6.12 Settings

Workspace · Members & Roles (RBAC matrix) · Billing (current plan, invoices, usage, payment methods, manage via Stripe portal) · Integrations (Google, Outlook, Slack, Teams, Webhooks) · API Keys · Branding (Enterprise) · Profile · Notifications · Danger Zone (delete workspace).

### 6.13 Admin Portal screens

Tenants (list/detail, suspend, impersonate-with-audit) · Subscriptions Ops (refunds, manual extensions) · Compliance Frameworks (CRUD with versioning + effective dates) · Crawler Status (per-source health, last run, error rate, retry, selector diagnostics) · AI Agent Registry (per-agent health, version, rollback) · System Health · Audit Log (search, filter, export).

---

## 7. Cross-Cutting UX Patterns

### 7.1 AI operation state machine

A single state machine governs all AI features (summarize, checklist, risk, draft, compliance, score, ESPD, decision matrix).

```
idle → requesting → (streaming | processing) → complete
                                            ↘ error → retry?
                                            ↘ canceled
```

UI rules:

- **First-token TTFB < 500 ms** for streaming agents; otherwise show a meaningful progress indicator with current step ("Parsing document…", "Extracting requirements…").
- **Cancelable.** Every long-running AI op exposes Cancel that aborts cleanly and returns the user to a known state.
- **Sourced.** Generated content links to its source (clause, prior proposal, content block).
- **Confidence visibility.** Where the agent reports confidence, surface it as a chip (High / Medium / Low) with tooltip; never as a misleading exact percentage.
- **Re-run hygiene.** If the underlying input changes, mark prior outputs as **Stale** rather than silently invalidating.
- **Errors.** Distinguish `transient` (auto-retry once, then offer manual retry) from `quota` (link to billing) from `content-policy` (link to docs) from `system` (incident page link).

### 7.2 Empty states

One template across the app: **icon + headline + 1–2 sentence description + primary CTA + optional secondary link**. No dead ends.

### 7.3 Notifications

- **In-app bell.** Toast for transient (autosave, copy link); persistent center for actionable (compliance fail, mention, deadline approaching).
- **Email digests.** Daily/weekly cadence per user; immediate for assignments and approvals.
- **Slack/Teams (P2).** Workspace-level channel mapping; per-user mute.
- **Browser push.** Off by default; explicit opt-in; respects OS quiet-hours.

### 7.4 Search and filter

One pattern wherever a list exists: top search bar + filter chip rail + "More filters" sheet. Saved searches and shareable URLs.

### 7.5 Mobile

The full client app is responsive, but **mobile is a companion experience, not the primary surface**. Prioritize on mobile: dashboard, opportunities browse, deadline calendar, notifications, quick comments/approvals, read-only proposal review. Heavy editing redirects gracefully ("Open on desktop for the full editor") rather than degrading silently.

### 7.6 Trial, paywall, and downgrade

- Trial countdown is **always visible** in the top bar during the trial; not alarmist.
- Paywalls show **a preview of what will be unlocked** (blurred summary, locked budget) — never a blank wall.
- Downgrade triggers a **data-preservation explainer** ("Your 12 saved opportunities will remain visible. Pro-only features will be archived; you can restore by upgrading.").

### 7.7 Internationalization

- All UI strings live in `next-intl` resource files. **No hard-coded strings.**
- Default locale: `bg-BG`. Fallback: `en-GB`.
- Number, date, currency formatting via `Intl.*` APIs (`1 234,56 €`).
- Pluralization via ICU MessageFormat.
- `pnpm check:i18n` must pass before merge.

### 7.8 Time, deadlines, time zones

- All deadlines stored UTC, displayed in the user's locale, default `Europe/Sofia`.
- Deadline cells always display **absolute date + relative ("in 3 days")**.
- Past deadlines are visually distinct (muted + strikethrough) and never sortable above future deadlines unless the user opts in.

### 7.9 Audit and trust signals

- All mutations write an entry to the immutable audit log; the user can see "Last edited by X at Y."
- Outbound AI calls are logged with model, version, latency, token count — surfaced to admins.

### 7.10 Command palette (`Cmd+K` / `Ctrl+K`)

A keyboard-first jump-to surface. Scoped sections:

- **Navigate** (any sidebar item or recent screen)
- **Find** (opportunities and proposals by title/ID)
- **Do** (run AI summary, run compliance, lock section, export, invite member — context-aware)
- **Help** (open shortcut sheet, contact support)

Listbox semantics; results announced; `Esc` returns focus to the trigger.

### 7.11 Real-time collaboration patterns *(P2)*

- Avatar cursors with name labels (Google-Docs-style).
- Per-section locking with explicit lock/unlock buttons; locked sections are read-only with a banner stating "Locked by [User Name]."
- Comment threads anchored to text selections; resolvable; @mentions notify by email + in-app.
- Conflict resolution favors **operational transformation** within a section; cross-section edits never conflict because of the section-locking pattern.

---

## 8. Accessibility (WCAG 2.1 Level AA)

EU Solicit serves government-facing workflows; accessibility is a **hard requirement (NFR-18, NFR-19, NFR-20)**, not a stretch goal.

### 8.1 Conformance commitments

- **WCAG 2.1 AA** across both client and admin apps.
- **EN 301 549** — the EU public-sector accessibility directive's reference standard — explicitly committed; documented in a public Accessibility Statement.
- A **VPAT-equivalent** report is published before MVP launch and refreshed each major release.

### 8.2 Perceivable

- **Color contrast.** Text/background ≥ 4.5:1 (normal) and 3:1 (large/UI). The slate + indigo palette is contrast-tested in light and dark themes.
- **Color is never the sole channel.** Status uses color **and** an icon **and** a text label (e.g., compliance fail = red icon + cross + "Fail: missing attachment").
- **Text resize.** Layout remains usable at 200% zoom and at 320 CSS-pixel width (WCAG 1.4.10 reflow).
- **Alt text.** All informative images, icons, and chart data points carry meaningful `alt` or `aria-label`. Decorative SVGs use `aria-hidden="true"`.
- **Charts.** Every chart has a parallel data table accessible via "View as table" and exposes ARIA roles for series and points.

### 8.3 Operable

- **Full keyboard support.** Every interactive element is reachable in logical tab order; no keyboard traps. Custom widgets (combobox, table, editor, command menu) implement WAI-ARIA Authoring Practices patterns.
- **Visible focus.** A 2 px solid focus ring with 3:1 contrast against any background; never removed by `outline: none`.
- **Skip links.** "Skip to main content" and "Skip to navigation" on every page.
- **Shortcuts.** Listed in a help modal (`?` to open) and re-mappable where they conflict with assistive tech.
- **No motion-only triggers.** Animations are decorative and respect `prefers-reduced-motion`.
- **Timeouts.** Trial countdowns and session timeouts warn the user with at least 60 seconds to extend (WCAG 2.2.1).

### 8.4 Understandable

- **Language declared.** `<html lang>` set per active locale; segments in another language carry their own `lang`.
- **Form errors.** Inline, associated via `aria-describedby`, announced via live regions, never color-only. Error messages name the field and the corrective action.
- **Predictable.** Navigation and page structure are consistent across screens; opening a modal does not silently navigate.
- **Plain language** in microcopy; jargon (CPV, ESPD, ZOP) is glossed via tooltip or inline help.

### 8.5 Robust

- **Valid semantic HTML.** Headings form a single coherent outline.
- **Compatible with NVDA, JAWS, VoiceOver, TalkBack.** Each release runs a smoke test on at least one screen reader per OS.
- **ARIA used sparingly.** Native HTML controls preferred; ARIA fills only semantic gaps.
- **Live regions.** AI streaming output uses `aria-live="polite"`; compliance results use `aria-live="assertive"` only when an error blocks submission.

### 8.6 Component-specific accessibility notes

| Component | Accessibility note |
|---|---|
| Tiptap editor | ProseMirror accessibility extensions; toolbar is `role="toolbar"` with arrow-key navigation; formatting state announced via `aria-pressed` |
| Data tables | `role="table"` with `scope` on headers; column sort state on `aria-sort`; row selection announced |
| Combobox / CPV picker | WAI-ARIA 1.2 combobox pattern; full keyboard typeahead |
| Command menu (`Cmd+K`) | Listbox semantics; results announced; `Esc` restores focus to trigger |
| Charts | Parallel table view; series colors paired with patterns/markers for color-blind users |
| File upload | Labeled `<input type=file>`; drag-and-drop also exposes a click-to-browse fallback; progress announced via live region |
| Stripe Checkout | Stripe-hosted; we provide an accessible loading state and a confirm-callback page; we do not embed card fields |
| Modal dialogs | `role="dialog"` with `aria-modal="true"`, focus trap, ESC closes, return focus to trigger |
| Toast notifications | `role="status"` (transient) or `role="alert"` (action-required); auto-dismiss respects `prefers-reduced-motion` |

### 8.7 Testing & gating

- **Automated.** `axe-core` runs in CI on every PR via Playwright; zero **serious** or **critical** violations may merge.
- **Manual.** A keyboard-only and screen-reader walkthrough of journeys J1 and J2 is performed for every release-candidate.
- **External audit.** A third-party WCAG audit is commissioned prior to MVP launch and annually thereafter.

---

## 9. Visual Design Foundations (summary)

These are deliberately summarized; the canonical tokens live in the design-system supplement and the Tailwind config (`frontend/packages/config/tokens`).

| Token group | Value summary |
|---|---|
| Typography | Inter for UI; IBM Plex Sans Bulgarian fallback; sizes 12/14/16/18/24/32; line-height 1.5 base, 1.25 headings |
| Color (light) | Surface slate-50; ink slate-900; primary indigo-600; success emerald-600; warning amber-600; danger rose-600 |
| Color (dark) | Surface slate-950; ink slate-50; primary indigo-400 |
| Spacing | 4px base scale: 4, 8, 12, 16, 24, 32, 48, 64 |
| Radius | 6 (controls), 10 (cards), 16 (modals) |
| Elevation | Three levels via subtle shadow + 1px border |
| Motion | 150–250 ms ease-out for UI; reduced-motion respected |
| Iconography | `lucide-react` only |

---

## 10. Component Library Direction

- **Client + admin apps share `packages/ui`** (shadcn/ui-based primitives) inside the `frontend/` Turborepo. **No fork.**
- **Forms.** `useZodForm` + `<FormField>` wrappers (React Hook Form + Zod). Schemas co-located with the form and reused for server validation contracts.
- **Data fetching.** TanStack Query v5 wrapped in a `<QueryGuard>` that yields a consistent loading/error/empty/restricted state contract.
- **State.** Zustand with namespaced persist keys (`eusolicit-client-auth-store`, `eusolicit-admin-auth-store`).
- **Rich text.** Tiptap. Section anchors are addressable (`#section/<uuid>`).
- **Iconography.** `lucide-react` exclusively, sized 16 / 20 / 24.

---

## 11. Performance & UX Budgets

| Metric | Target | Rationale |
|---|---|---|
| Lighthouse Performance | ≥ 90 | NFR-3 |
| LCP | < 2.0 s on broadband | Above-the-fold-of-the-fold |
| CLS | < 0.05 | Skeleton-driven loading |
| INP | < 200 ms | Editor responsiveness |
| AI TTFB (streaming) | < 500 ms | NFR-2; user-perceived responsiveness |
| First-draft generation | < 2 minutes p95 | J1 acceptance |
| Time-to-first-AI-summary, new user | < 5 minutes | J1 acceptance |
| API p95 | < 200 ms | NFR-1 |

---

## 12. Risks & Open Questions

| # | Risk / Question | Mitigation / Owner |
|---|---|---|
| R1 | AI output quality below user threshold | Human-in-the-loop everywhere; explicit Edit/Reject; confidence chips; rejection feedback into agent registry |
| R2 | Free-tier feels too thin or too generous | Instrument funnel; the paywall preview is the lever; tune "blurred fields" set after first 2 weeks of MVP data |
| R3 | Streaming UX over slow connections | SSE + chunk batching; visible "(Network slow — still streaming)" banner after 3s without progress |
| R4 | Bulgarian + English parity in AI outputs | Bilingual prompt templates; per-locale evaluation set; AI quality gate per locale |
| R5 | Complex RBAC matrix overwhelming admins | Progressive disclosure: per-resource simple toggle by default; advanced grid behind a "Customize" toggle |
| R6 | ESPD legal validity | Generated XML validated against EU ESPD schema in CI; legal review of templates before MVP |
| R7 | Editor performance on 100+ page proposals | Virtualized section rendering; lazy-load sections beyond viewport; server-side compliance/score computation |
| Q1 | Should the score simulator be rate-limited per tier? | Pending — Product to confirm Starter limits |
| Q2 | Which integrations cluster ships first in P2? | Pending — Product to sequence Calendar vs. Slack/Teams vs. CRM |
| Q3 | Do we expose AI model identity to end users? | Pending — Legal + Product on transparency vs. competitive disclosure |

---

## 13. MVP UX Acceptance Checklist

A release candidate is UX-acceptable when all of the following are demonstrably true on staging:

- [ ] A new user (no prior context) can complete journey **J1 steps 1–8** in under 10 minutes.
- [ ] A free-tier user can complete journey **J2 steps 1–6** without encountering any broken/empty paid screens.
- [ ] All MVP screens render an accessible **empty / loading / error / success / paywalled / restricted** state per Section 6.
- [ ] All AI features conform to the **state machine** in §7.1 with visible cancel and source links.
- [ ] All forms expose accessible error messaging (§8.4) and zero `axe-core` serious/critical violations.
- [ ] Lighthouse Performance ≥ 90 on Dashboard, Opportunities List, and Proposal Editor.
- [ ] Bulgarian and English locales produce no missing-string warnings (`pnpm check:i18n` clean).
- [ ] Keyboard-only walkthrough of J1 and J2 completes without traps.
- [ ] Screen-reader smoke test on at least NVDA + VoiceOver passes for J1.
- [ ] Accessibility Statement page is published with VPAT-equivalent report.
- [ ] All copy in Settings → Billing — including downgrade and cancellation — is parity-tested in BG and EN.

---

## 14. Out of Scope (this document)

- Pixel-perfect mockups (delivered in Figma).
- Final color tokens beyond the summary in §9 (live in `frontend/packages/config/tokens`).
- Per-component API contracts (live in `packages/ui` Storybook).
- Marketing site UX (separate scope).
- Phase-3 features (consultant marketplace, full automation, global expansion) beyond IA placeholders.

---

*End of UX Specification — EU Solicit (Draft v2, 2026-04-28).*
