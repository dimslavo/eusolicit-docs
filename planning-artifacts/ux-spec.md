---
document: UX Specification
project: EU Solicit
author: Sally (BMAD UX Designer) — autonomous run
date: 2026-05-05
sourcePRD: eusolicit-docs/planning-artifacts/PRD.md
status: Draft v3.1 (autopilot-verified against PRD 2026-04-27 on 2026-05-05)
scope: MVP (Phase 1) with forward-compatible patterns for Phase 2/3
audiences: ["Product", "Engineering", "QA", "Design"]
---

# EU Solicit — UX Specification

> Companion to `PRD.md`. This document translates PRD personas, journeys and functional requirements into a concrete UX blueprint: information architecture, key screens and states, interaction patterns, content rules, and accessibility (WCAG 2.1 AA) commitments. It is the source of truth for screen-level expectations consumed by epics/stories.

---

## 1. UX Vision & Principles

### 1.1 UX Vision

EU Solicit must feel like **"a calm, expert co-worker, not a chatbot"** — opinionated where it has data, transparent where it doesn't, and always keeping the human bid manager in command. Every AI surface is a glass box: users see what was extracted, why a score was assigned, and where to push back. The product collapses a 2-day reading task into a 10-minute review while leaving every decision recoverable.

### 1.2 Design Principles

| # | Principle | What it means in practice |
|---|---|---|
| P1 | **Human-in-the-loop, always** | No AI output is final. Every generated artifact is editable, has a "regenerate", an "explain", and a clear provenance label ("AI draft — review needed"). |
| P2 | **Time-to-value first** | A trial user must reach an AI summary in < 5 minutes from sign-up. The empty-state of every screen ships with a ready-to-click CTA. |
| P3 | **Progressive disclosure** | Lists show 4–6 signals max; detail panes carry depth. Advanced features (locking, approvals, score sim) revealed contextually, not in primary nav. |
| P4 | **Confidence over completeness** | When the AI is unsure, say so. Confidence chips, "low signal" warnings, and "review recommended" callouts beat false precision. |
| P5 | **Compliance is a first-class citizen** | ESPD, ZOP/TED rules, and deadline math get persistent UI affordances — not buried in menus. Compliance state is always visible on the proposal workspace header. |
| P6 | **Multi-tenant clarity** | Workspace identity (logo + name) is permanent in the chrome. Switching workspaces is two clicks and visually unmistakable. |
| P7 | **Accessible by default** | WCAG 2.1 AA is met before a screen ships, not retrofitted. Every interactive element is keyboard-reachable and screen-reader labelled. |
| P8 | **Bilingual native** | Bulgarian and English are equal citizens in copy, dates, currency, and document templates. No machine-translated strings in primary surfaces. |

### 1.3 Non-Goals (UX scope boundaries)

- No native mobile app in MVP — responsive web only, optimized for desktop ≥ 1280 px and tablet ≥ 768 px.
- No real-time collaborative cursor presence in MVP (Phase 2).
- No marketplace, white-label theming, or admin-side per-tenant theming in MVP.

---

## 2. Personas

Personas are derived from the PRD journeys and refined for screen-level decisions.

### 2.1 Elena — The Overwhelmed Bid Manager *(primary)*

| | |
|---|---|
| **Role** | Bid Manager, mid-sized consulting firm (Sofia, 5-person team) |
| **Subscription** | Professional (14-day trial → paid) |
| **Tech literacy** | High (lives in spreadsheets, Notion, Google Workspace) |
| **Goals** | Win more bids; stop missing deadlines; cut prep time per bid from 2 days to < 4 hours |
| **Frustrations** | Fragmented tools, version-controlled chaos in shared drives, no visibility on team workload |
| **Quote** | *"I don't need another tool. I need fewer tools that actually work together."* |
| **Devices** | 14" laptop (primary), occasionally a 27" external monitor; rarely uses mobile for work |
| **UX implications** | Dense data tables OK; needs keyboard shortcuts; multi-pane editor; portfolio dashboard; @-mentions and assignment patterns |
| **Accessibility considerations** | None known — but power-user keyboard shortcuts must coexist with screen-reader semantics |

### 2.2 Maria — The Cautious Solopreneur *(secondary, freemium funnel)*

| | |
|---|---|
| **Role** | Independent environmental consultant |
| **Subscription** | Free → Starter (€29/mo) |
| **Tech literacy** | Medium (comfortable with web apps, not technical) |
| **Goals** | Find a couple of relevant grants/tenders per quarter; submit competitive bids without hiring a consultant |
| **Frustrations** | Government portals are opaque; consultants too expensive; jargon overload |
| **Quote** | *"If I can't see what I'm getting before paying, I'm out."* |
| **Devices** | 13" MacBook + iPhone; sometimes browses opportunities on phone during commute |
| **UX implications** | Mobile-readable opportunity list; aggressive empty-state hand-holding; transparent paywalls (lock icons, not hidden fields); upgrade flow under 3 clicks |
| **Accessibility considerations** | Larger default font; reduced jargon; bilingual labels visible together for legal/document terms |

### 2.3 Stefan — The Reviewer / Approver *(tertiary, Phase 2)*

| | |
|---|---|
| **Role** | Director / Partner, signs off bids before submission |
| **Subscription** | Professional / Enterprise (Reviewer role) |
| **Tech literacy** | Medium |
| **Goals** | Approve/reject quickly; see compliance status and risks at a glance; not get pulled into editing |
| **Frustrations** | Being asked to read 80-page PDFs at midnight |
| **UX implications** | Read-only proposal view, comment-only mode, single-click approve/reject, mobile-friendly review surface |

### 2.4 Iliyan — The Platform Admin *(internal)*

| | |
|---|---|
| **Role** | EU Solicit internal operator |
| **Subscription** | n/a — internal admin portal |
| **Goals** | Maintain crawler health; update compliance frameworks; manage tenants and incidents |
| **UX implications** | Information-dense ops dashboards, drill-down logs, IP-restricted entry, MFA-gated actions, immutable audit trail visible on every mutation |

### 2.5 Daniel — The Enterprise Developer *(API consumer)*

| | |
|---|---|
| **Role** | Developer at a large consulting firm with Enterprise plan |
| **Goals** | Pull scored opportunities into Salesforce; manage API keys; understand quotas |
| **UX implications** | Self-serve API key management UI, rotate/revoke flow, OpenAPI/Swagger console embedded in admin section, copy-paste-friendly docs |

---

## 3. Information Architecture

### 3.1 Top-level IA — Client Portal (`apps/client`, port 3000)

```
[Workspace switcher] → [Global Nav]
├── Dashboard                     (Elena's home; portfolio + alerts)
├── Opportunities                 (search/list/detail)
│   ├── List
│   ├── Detail
│   └── Saved
├── Proposals                     (list, kanban, detail/editor)
│   ├── All proposals
│   ├── My drafts
│   └── Proposal workspace        (split-pane editor)
├── Calendar (Phase 2)
├── Compliance Library            (ESPD, frameworks, document vault)
├── Reports / Analytics (Phase 2)
└── Settings
    ├── Company profile
    ├── Members & roles
    ├── Billing & subscription
    ├── Integrations (Calendar, Slack/Teams)
    └── Notifications
```

### 3.2 Top-level IA — Admin Portal (`apps/admin`, port 3001)

```
├── Operations Dashboard          (uptime, queue depth, AI quotas, error rates)
├── Tenants                        (list, detail, suspend, impersonate-with-audit)
├── Users (cross-tenant search)
├── Compliance Frameworks         (ZOP, Horizon Europe, etc.)
├── Crawlers                       (status, schedule, logs, manual re-run)
├── AI Agents                      (registry, models, rate-limit policies, circuit breaker state)
├── Billing Operations             (Stripe events, refunds, dunning)
├── Audit Log
└── Settings (system config, feature flags)
```

### 3.3 URL Conventions

- `/{lang}/dashboard` — `lang` ∈ `bg | en`, default by `Accept-Language`, persisted on user profile.
- Stable, shareable URLs for every entity: `/opportunities/:id`, `/proposals/:id`, `/proposals/:id/section/:sectionId`.
- Workspace context lives in a cookie + JWT claim, not the URL — switching workspace does not break deep links to entities the user can access.

### 3.4 Global Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  [LOGO]  [Workspace ▾]    [Search]              [🔔] [Help] [👤] │  Top bar (sticky)
├──────────┬───────────────────────────────────────────────────────┤
│ Sidebar  │                                                        │
│  Dashboard│            Page content                                │
│  Opps    │            (max-width 1440 px, fluid below)            │
│  Props   │                                                        │
│  ...     │                                                        │
│  Settings│                                                        │
└──────────┴───────────────────────────────────────────────────────┘
```

- Sidebar collapses to icon-rail at `< 1024 px`; full overlay drawer at `< 768 px`.
- Top bar is **always** keyboard-reachable in a single Tab cycle: skip-link → workspace → search → notifications → user.

---

## 4. Primary User Journeys (Step-by-Step)

Each journey lists screens, states, error/empty/loading paths, and a11y notes. Journeys map directly to PRD `FR-*` and the epics in `epics.md`.

### Journey A — New Trial User to First AI Summary (Elena)

**Goal:** Sign up, set up company profile, upload a tender, see an AI executive summary. Target time-to-value < 5 minutes.
**Functional refs:** FR-1, FR-2, FR-4, FR-5, FR-12, FR-21, FR-22, FR-24.

```
Step 1  Marketing → Sign-up                     [Public]
Step 2  Email verification                       [Email]
Step 3  Welcome / role chooser                   [Onboarding]
Step 4  Company profile wizard (3 steps)         [Onboarding]
Step 5  Trial activation banner                  [Dashboard]
Step 6  Empty Opportunities OR "Upload tender"   [Opportunities]
Step 7  Upload modal → progress                  [Modal]
Step 8  AI processing — streaming summary        [Opportunity detail]
Step 9  Review summary + checklist               [Opportunity detail]
Step 10 CTA: "Generate proposal draft"           [→ Journey B]
```

**Step details:**

1. **Sign-up screen.** Email + password OR "Continue with Google". Inline password rules. Submit triggers verification email. State: idle / submitting / error (`email exists`, `weak password`, `invalid email`).
2. **Verify-email gate.** Persistent banner if user lands without clicking link. "Resend" with 60s cooldown. A11y: live region announces resend success.
3. **Welcome page.** "Are you bidding for…" radio: a company / yourself / evaluating. Drives default plan and onboarding copy.
4. **Company profile wizard.**
   - 4a — Company basics: legal name, EIK/VAT, country, bilingual name fields.
   - 4b — Capabilities: CPV codes (multi-select with Bulgarian + English labels), regions, languages.
   - 4c — Key personnel: optional, can skip (deferred to Settings).
   - "Skip for now" available on every step except 4a; reminder card persists on dashboard until complete.
5. **Trial activation.** Top-of-dashboard banner: "Professional trial — 14 days left. [Manage plan]". Dismissible per session, never permanently hidden.
6. **Empty Opportunities.** Dual CTA: "Browse pre-loaded opportunities" or "Upload your own tender (PDF/DOCX, ≤ 50 MB)". Shows 3 sample tenders matched to user's CPV codes.
7. **Upload modal.** Drag-and-drop + file picker. Validates type, size, and runs ClamAV scan (visible step). States: idle → uploading → scanning → analyzing → ready / error (`virus_detected`, `unsupported_format`, `oversize`, `parse_failed`).
8. **AI processing.** Inline streaming pane with skeleton sections (Summary / Requirements / Risks). TTFB target < 500 ms (NFR-2). Cancel button visible. If cancelled, partial output is preserved as draft.
9. **Opportunity detail.** Tabs: Summary | Requirements | Risks | Documents | Timeline. Provenance footer per AI block: model, timestamp, confidence chip.
10. **CTA.** Sticky right-rail action card: "Generate proposal draft" → Journey B.

**Error states (whole journey):**

- Email already in use → inline link: "Sign in instead".
- Verification email not received in 60 s → "Try a different address" link.
- Upload fails virus scan → modal with rationale + support link; no retry.
- AI generation fails → skeleton replaced by error card with "Retry" and "Contact support"; preserves uploaded document.

**Accessibility notes:**

- Skip-to-main link on every onboarding screen.
- Wizard progress announced via `role="progressbar"` with `aria-valuenow/valuemax`.
- Streaming summary uses `aria-live="polite"` to announce sectional completions, not every token.
- All modals trap focus and restore on close.

---

### Journey B — Proposal Drafting & Compliance Validation (Elena)

**Goal:** Generate AI draft, edit collaboratively (MVP single-user; Phase 2 multi-user), validate against checklist, export.
**Functional refs:** FR-26 → FR-34, FR-23, FR-25.

```
Step 1  Click "Generate proposal draft"         [Opportunity detail]
Step 2  Confirm scope modal                      [Modal]
Step 3  Streaming draft into editor              [Proposal workspace]
Step 4  Section-by-section review/edit           [Proposal workspace]
Step 5  Run compliance check                     [Proposal workspace]
Step 6  Resolve compliance issues                [Proposal workspace]
Step 7  Run score simulator                      [Proposal workspace]
Step 8  Apply suggestions / iterate              [Proposal workspace]
Step 9  Export to PDF/DOCX                       [Proposal workspace]
Step 10 Mark as submitted                        [Proposal workspace]
```

**Proposal workspace layout** (the most important screen in the product):

```
┌───────────────────────── Proposal Header ────────────────────────────┐
│  Title  ·  Opportunity link  ·  Deadline (countdown)                  │
│  Status: Draft | In review | Submitted                                │
│  Compliance: 12/14 ✓     Score: 85/100     [Export ▾] [Validate]      │
├──────────────┬───────────────────────────────────────────────────────┤
│  Outline     │   Editor (TipTap, max 90ch line)                       │
│  ─────────   │   ┌────────────────────────────────────────────────┐   │
│  ✓ Cover     │   │ Section: Methodology                           │   │
│  ✓ Summary   │   │ [AI-generated]   Last edited 3m ago — Elena    │   │
│  ⚠ Methodology│  │                                                │   │
│  · Pricing   │   │  Lorem ipsum...                                │   │
│  · Annexes   │   │                                                │   │
│              │   └────────────────────────────────────────────────┘   │
│              │   Toolbar: B I U • Heading • List • Insert ▾ • AI ✦   │
├──────────────┴───────────────────────────────────────────────────────┤
│  Right rail (tabs): Compliance | Score | Comments (P2) | Versions     │
└───────────────────────────────────────────────────────────────────────┘
```

**Step details:**

1. **Generate trigger.** Confirms data sources (company profile, tender doc, optional past proposals). Estimates tokens / cost-tier (silent for trial).
2. **Confirm scope.** "Generate draft for: [all sections | selected sections]". Default = all.
3. **Streaming draft.** Each section appears with a section divider, "AI draft — review needed" badge, and a "Stop / Regenerate this section" control. User can start editing already-streamed sections immediately.
4. **Section review.** Editor supports: headings, lists, tables, image insert (Phase 2), citation pills. Inline AI assist: select text → "Improve / Shorten / Translate / Make more formal". Each AI rewrite shows a diff overlay with Accept/Reject.
5. **Compliance check.** Right-rail "Compliance" tab lists every requirement extracted in Journey A, status: ✓ met / ⚠ partial / ✗ missing / ➖ n/a, with deep-link to the responsible section. Severity chip (Critical / Major / Minor). FR-34.
6. **Resolve issues.** Click a ⚠/✗ row → editor scrolls + highlights candidate section + opens a "Suggested fix" panel. User can accept fix, edit, or mark as "Manually resolved" (audit-logged).
7. **Score simulator.** FR-25. Runs on demand (cost-aware). Output: total score, per-criterion breakdown, top 3 improvement suggestions ranked by expected score lift. Confidence band shown explicitly.
8. **Iterate.** Apply suggestion → re-run score → diff vs. previous run side-by-side.
9. **Export.** Modal: PDF or DOCX, include/exclude annexes, language (BG/EN/both), template (Default / Company branded — Phase 2). Export jobs go to a queue with toast on completion.
10. **Mark submitted.** Status change is irreversible without admin support. Triggers audit log entry, archives version, removes from "Active" board.

**Key states across Journey B:**

- **Draft saving:** Auto-save every 5 s + on blur. State chip: "Saved 2 s ago" / "Saving…" / "Offline — changes pending". On reconnect, conflict resolution = last-writer-wins for MVP single-user, with version snapshot.
- **AI unavailable / over quota:** Editor stays fully usable. AI buttons disabled with tooltip "AI temporarily unavailable — your work is safe".
- **Export failure:** Toast with "Retry" and link to Notifications center; never blocks editor.
- **Section locking (Phase 2):** Padlock icon, presence avatars, ghost cursor of locker.

**Accessibility notes:**

- Editor is fully keyboard operable (TipTap defaults extended): `Ctrl+B/I/U`, `Ctrl+K` link, `Alt+H` heading, `Esc` exits inline AI menu.
- Compliance issue list is a `role="list"` with `aria-live="polite"` for status changes after re-validation.
- Score simulator announces "New score: 92 of 100, up from 85" via live region.
- All AI badges include screen-reader-only text: "AI-generated content, review required".

---

### Journey C — Free → Paid Upgrade (Maria)

**Goal:** Discover an opportunity, hit the paywall, decide to upgrade, complete checkout, unlock detail.
**Functional refs:** FR-10, FR-11, FR-13, FR-16, FR-19.

```
Step 1   Sign up free                            [Public]
Step 2   Browse opportunities                    [Opportunities list]
Step 3   Open opportunity                         [Opportunity detail (locked)]
Step 4   Hit paywall on Budget / AI Summary       [Paywall card]
Step 5   Compare plans                            [Plans page]
Step 6   Stripe Checkout                          [External]
Step 7   Return → unlock                          [Opportunity detail (unlocked)]
Step 8   Use Starter features                     [Proposal workspace, limited]
```

**Paywall pattern (critical UX):** Locked content is **visible-but-fogged**, not hidden. Each lock has:

- a small 🔒 icon,
- a one-line value pitch ("Budget data unlocks on Starter — €29/mo"),
- two buttons: `Compare plans` (primary) and `Upgrade now` (secondary).

This is deliberately Maria-shaped: she will not pay for an unknown.

**Plan comparison page** uses a 4-column matrix (Free / Starter / Professional / Enterprise) with clear feature ✓/✗, sticky CTA per column, and an "EU VAT will be calculated at checkout" disclaimer.

**Stripe Checkout** runs in Stripe's hosted environment (PCI-out-of-scope). On return:

- Success → toast "Welcome to Starter" + confetti (low-key, respects `prefers-reduced-motion`).
- Failure → return to plans page with error banner referencing Stripe's reason code.
- Pending (3DS) → polling page with live status; auto-redirect on resolution.

**Edge states:**

- VAT-id not provided → field appears in checkout, not before.
- Card declined → user-friendly copy ("Your bank declined the charge — try another card or contact your bank.") with retry CTA.
- Idempotency: a refresh during checkout will not double-charge (FR-10, NFR-16).

**A11y:** paywall cards announce "Locked content. Upgrade to Starter to unlock." Lock icons are decorative (`aria-hidden`) because text already conveys it.

---

### Journey D — Team Collaboration (Phase 2 preview)

**Goal:** Bid Manager invites contributors, assigns sections, runs an approval workflow.
**Functional refs:** FR-6, FR-7, FR-31, FR-32, FR-37, FR-38, FR-39.

Steps and screens (summary, full spec deferred to Phase 2 design pass):

1. Settings → Members & roles → Invite (email, role).
2. Proposal workspace → outline → "Assign" per section → user picker.
3. Contributor receives notification, opens proposal in their assigned section.
4. Reviewer is added with read+comment permission; receives review request when proposal hits "In review".
5. Approval workflow: linear N-step approver list configurable per company; each approver gets Approve / Request changes; on final approval, status → "Ready to submit".

**MVP equivalent:** single-user editing, single Admin role + Read-Only invites. Approval workflow is hidden, replaced by a manual "Mark as submitted" with a confirm dialog.

---

### Journey E — Enterprise API Setup (Daniel)

**Goal:** Generate API key, read docs, make first call, monitor usage.
**Functional refs:** FR-44, NFR-7.

1. Settings → Integrations → API (Enterprise only; tooltip on lower tiers explaining gate).
2. "Create API key" modal: name, scopes (read-only by default), expiration.
3. Key shown **once**, with copy button, "I have saved this key" gating dismissal.
4. Side-panel embeds OpenAPI/Swagger UI scoped to the workspace.
5. Usage dashboard: requests/hour, top endpoints, last 50 calls with status codes; rotate / revoke buttons.

**A11y:** the one-time key reveal must use a non-color signal (a yellow warning box with text) and a "Copy" button reachable by keyboard with `aria-label="Copy API key — visible once"`.

---

### Journey F — Platform Admin Operations (Iliyan)

**Goal:** Update a compliance framework and triage a failing crawler.
**Functional refs:** FR-35, FR-42, FR-43, NFR-8.

1. VPN/IP check → MFA gate → admin login.
2. Operations Dashboard: tiles for crawler health, queue depth, AI gateway error rate, billing webhook lag. Each tile is a link to its detail screen.
3. Compliance Frameworks → "Horizon Europe" → Edit → Add 3 new rules with severity & description → Save (requires re-confirm + audit reason).
4. Crawlers → "AOP-BG" status: red → drill into 24h error log → file ticket inline.
5. All mutations write to Audit Log with actor, timestamp, before/after diff.

**A11y:** dense data tables follow the WCAG-compliant table pattern (caption, scope, sortable headers announce direction).

---

## 5. Key Screens & States — Catalogue

For each screen: purpose, primary content, key states, and which `FR-*` it serves.

### 5.1 Public marketing & auth

| Screen | Purpose | Key states | FR |
|---|---|---|---|
| Landing | Convert visitor to sign-up | default, BG/EN | — |
| Pricing | Plan comparison | logged-out, logged-in, currency-by-locale | FR-10 |
| Sign-up | Create account | idle, submitting, success, errors (email exists, weak pwd) | FR-1, FR-2 |
| Sign-in | Authenticate | idle, error, MFA challenge, locked-out | FR-3 |
| Verify email | Gate to core | unverified-banner, verifying, verified | FR-4 |
| Forgot/Reset password | Recover | request, sent, reset, expired | FR-3 |

### 5.2 Onboarding

| Screen | Purpose | Key states |
|---|---|---|
| Welcome / role chooser | Personalize onboarding | a/b/c choice |
| Company wizard 1/3 | Basics | empty, validating, error |
| Company wizard 2/3 | Capabilities (CPV) | empty, populated, "no codes match — search wider" empty |
| Company wizard 3/3 | Key personnel (skippable) | empty, populated, skipped |
| Trial activated | Confirmation + tour CTA | one-time |

### 5.3 Dashboard (post-login home)

**Purpose:** Elena's portfolio at a glance.
**Sections:**
1. Trial / billing banner (conditional).
2. "Active proposals" cards — title, opportunity, deadline countdown, score, compliance %, owner avatars.
3. "Upcoming deadlines" timeline (next 14 days).
4. "Recommended opportunities" list (top 5 by AI relevance).
5. "Team activity" feed (Phase 2).

**Key states:** first-run empty, populated, all-overdue (visual warning).

### 5.4 Opportunities — List

**Layout:** left filter rail · main table · right preview pane.
**Filters:** keyword, CPV (multi), country/region, budget range, deadline window, source (AOP/TED), status (new/saved/dismissed).
**Columns:** ★ save · Title · Buyer · Country · Deadline · Budget · AI score · Source.
**States:** loading skeleton, empty (with sample suggestions), filtered-empty (with "clear filters" CTA), error.
**Pagination:** server-side, 25/page default; infinite scroll alternative behind feature flag.
**FR:** FR-15 → FR-20.

### 5.5 Opportunities — Detail

**Tabs:** Summary · Requirements · Risks · Documents · Timeline · Activity.
**Header:** title, buyer, deadline countdown, score, CPV chips, language flags, "Save" / "Generate proposal" CTAs.
**AI Summary block:** streamed in, with model+timestamp+confidence footer; "Regenerate" requires admin or Bid Manager role on Free/Starter quota.
**Requirements tab:** structured table; each row: requirement text, type (admin/technical/financial), severity, source page link in original PDF.
**Risks tab:** flagged clauses with severity chip and explanation.
**Documents tab:** original PDFs/DOCX with viewer; download per role; ESPD pre-fill action (FR-36).
**States:** locked (free tier), partially analyzed, fully analyzed, analysis-failed.

### 5.6 Proposal Workspace

(See Journey B for layout.) **Key sub-screens / panels:**

- **Outline panel** — drag-to-reorder sections (Bid Manager+).
- **Editor** — TipTap with a custom toolbar group: AI (Improve / Shorten / Translate / Cite).
- **Compliance panel** — interactive checklist, deep-linked to sections.
- **Score panel** — current score, last run timestamp, suggestions.
- **Versions panel** — list of snapshots, per-version diff vs. current, restore (FR-29, FR-30).
- **Export modal** — format, language, template, annexes.

**Empty/edge states:**

- No draft yet → "Generate first draft" CTA card occupies the editor area.
- AI quota exhausted → AI buttons disabled with usage-meter tooltip linking to billing.
- Network offline → top banner, autosave queues locally (IndexedDB) and replays on reconnect.

### 5.7 Compliance Library

- **ESPD generator** — pre-fill from company profile (FR-36), preview, export XML + human-readable PDF.
- **Frameworks browser** — read-only client view of frameworks (ZOP, Horizon Europe etc.).
- **Document vault** — re-usable artifacts (certificates, CVs, references) with tags and expiry warnings.

### 5.8 Settings

| Section | Notes |
|---|---|
| Company profile | Mirror of onboarding wizard; edits trigger re-index for relevance scoring. |
| Members & roles | Table of members, role dropdown (gated by admin), invite modal, pending invites. |
| Billing & subscription | Current plan card, usage meters (proposals, AI tokens, seats), "Manage in Stripe portal" link, invoices list. |
| Integrations | Cards: Google Calendar, Outlook, Slack, Teams, API (Enterprise). Each with connect / disconnect / scope-displayed. |
| Notifications | Channel × event matrix (in-app / email / Slack); per-row toggle. |
| Audit log (Phase 2 client-side) | Filterable list of actions in this workspace. |

### 5.9 Admin Portal screens

Operations Dashboard · Tenants list/detail · Users search · Compliance Frameworks editor · Crawlers monitor · AI Agents registry · Billing operations · Audit log · Feature flags. Each has explicit MFA-gated actions for destructive operations (suspend tenant, force re-bill, edit framework).

### 5.10 Cross-cutting screens

- **404 / 403 / 500** — branded, with workspace-aware navigation back, support link, and screen-reader-friendly headings.
- **Maintenance mode** — read-only banner, queues writes where safe.
- **In-app help** — slide-over with search, contextual articles, "Contact support" form.

---

## 6. Interaction & Component Patterns

### 6.1 Navigation

- **Workspace switcher** is a button (not a hover menu) opening a popover; keyboard-navigable; shows last 5 + search.
- **Global command palette** (`Ctrl/Cmd + K`): jump to opportunity, proposal, settings page, run an action ("Validate this proposal").

### 6.2 Tables

- Sortable columns indicate state via icon + `aria-sort`.
- Row selection via checkbox column with "select all on page" (never silently selects across pages).
- Bulk actions appear in a sticky action bar above the table when ≥ 1 row selected.

### 6.3 Forms

- All forms use `useZodForm` + `<FormField>` wrappers (per CLAUDE.md frontend pattern).
- Inline validation on blur; submit-time validation aggregated in a `role="alert"` summary at top of form.
- Required fields marked with `*` AND `aria-required="true"`; never color-only.
- Long forms (e.g. company profile) save per-step or on tab change; never lose user input on validation error.

### 6.4 Feedback

- **Toasts** for transient confirmations (4 s default, dismissible, never the only channel for critical info).
- **Banners** for persistent state (trial, billing failed, maintenance).
- **Inline messages** for field-level issues.
- **Modals** only for: confirm destructive action, capture short structured input (invite member), view Stripe-redirect-style flows. Never for primary content.

### 6.5 AI-specific patterns

- **Streaming output** — every streamed response shows a section skeleton, an animated cursor, and a `Stop` button.
- **Provenance footer** — `Model · Generated YYYY-MM-DD HH:mm · Confidence: High/Medium/Low · [Why this?]`.
- **Inline AI menu** — `Ctrl+J` over selected text opens it; keyboard arrow + Enter selects an action; `Esc` cancels.
- **Diff acceptance UX** — green-add / red-remove tracks, "Accept all" / "Accept this hunk" / "Reject" controls.
- **Quota meters** — visible in account header for Pro/Enterprise; hover reveals breakdown by agent.

### 6.6 Loading, empty, error pattern

Every data surface ships **four** designs: **default, loading, empty, error.** Empty states must show one primary CTA and one link to learn more. Error states must show a recovery action plus a support reference ID for log correlation.

### 6.7 Bilingual content

- Strings managed via `next-intl`. Translation keys are mandatory; raw strings are a lint error.
- Domain terms (CPV codes, ZOP article references, ESPD field labels) are presented bilingual side-by-side, e.g. `Доказателства за технически възможности (Technical capability evidence)`.
- Date format follows locale (`dd.MM.yyyy` BG, `MMM d, yyyy` EN). Currency always EUR with explicit code.

---

## 7. Visual Design Foundations

> Visual identity is owned by the design team; this section captures the constraints derived from accessibility, brand, and engineering reality.

- **Color** — primary brand (deep blue), accent (warm amber for AI surfaces). Minimum contrast ratio 4.5:1 for body text, 3:1 for large text. Never encode meaning by color alone — always pair with icon or text.
- **Typography** — system font stack with optional Inter for headings. Base size 16 px, scale 1.25 (Major Third). Line-height 1.5 for body. Bilingual fallbacks must include Cyrillic-supporting weights.
- **Spacing** — 4 px base scale (4, 8, 12, 16, 24, 32, 48, 64).
- **Iconography** — Lucide / Heroicons; outlined style; 20 px default. All icons get a text label in primary nav and dense tables.
- **Motion** — < 200 ms for micro-interactions; respect `prefers-reduced-motion`. AI streaming animations capped at 2 Hz to avoid distraction.
- **Components** — shadcn/ui in `frontend/packages/ui` is the source of truth; custom components must justify deviation in PR description.

---

## 8. Responsive Behaviour

| Breakpoint | Width | Layout intent |
|---|---|---|
| Desktop XL | ≥ 1440 px | All three panes (outline · editor · right rail) visible. |
| Desktop | 1024–1439 px | Outline collapses to icon rail; editor + right rail. |
| Tablet | 768–1023 px | Right rail becomes a tabbed sheet at the bottom or drawer on demand. |
| Phone | < 768 px | Read-mostly: opportunity browse, proposal review, comments, approve/reject. **Editing is degraded** (single column, AI assist disabled by default). Onboarding fully supported. |

Touch targets ≥ 44 × 44 px. Hover-only affordances are duplicated as visible buttons at touch sizes.

---

## 9. Accessibility (WCAG 2.1 AA)

NFR-18, NFR-19, NFR-20 must be met before any screen ships. This section defines the contract.

### 9.1 Conformance targets

- **WCAG 2.1 Level AA** across every authenticated and public surface.
- Voluntary AAA where feasible: line-height, image-of-text avoidance, error-prevention on financial actions.

### 9.2 Keyboard

- Every interactive element reachable via Tab in a logical reading order.
- Visible focus ring (≥ 3:1 contrast) on every focusable element; never `outline: none` without a replacement.
- Skip-to-main-content link on every page.
- Editor, command palette, modals, popovers all trap focus appropriately and restore it on close.
- Documented shortcuts: `Ctrl/Cmd+K` palette, `Ctrl+S` save, `Ctrl+J` AI menu, `?` open shortcut help.

### 9.3 Screen readers

- Semantic HTML first; ARIA only where semantics fall short.
- Landmarks: `header`, `nav`, `main`, `aside`, `footer` on every page.
- Headings form a single hierarchical outline per page (one `h1`).
- Live regions: `aria-live="polite"` for save status, score changes, AI section completions; `aria-live="assertive"` for errors blocking submission.
- Decorative icons use `aria-hidden="true"`. Icon-only buttons must include `aria-label`.

### 9.4 Color & contrast

- Body text 4.5:1; large text and icons 3:1; UI component states (focus, hover) 3:1 against adjacent colors.
- Status conveyed by **icon + text + color**, never color alone (compliance ✓/⚠/✗ rows are the canonical example).
- Both light and dark themes must independently pass contrast tests.

### 9.5 Forms & errors

- Labels are visible; placeholders never substitute for labels.
- Errors are announced (live region), associated with their field via `aria-describedby`, and retain user input.
- Destructive actions require typed confirmation OR a 5-second undo toast (whichever fits the action).

### 9.6 Media & documents

- Uploaded PDFs/DOCX are not accessible by default; the platform clearly communicates this and provides text-extracted views (the AI summary and requirements list) as accessible alternatives.
- Any platform-authored video/screencast must include captions and transcripts.

### 9.7 Cognitive accessibility

- Plain-language toggle in onboarding (defers jargon-heavy CPV labels to tooltips).
- Time-limited actions (e.g. Stripe 3DS challenges) display the limit and offer extension where the underlying provider allows.
- No auto-advancing carousels in the product (marketing site exempt with controls).

### 9.8 Testing & gates

- **Per-PR:** automated `axe` checks on changed routes; failing checks block merge.
- **Per-release:** keyboard-only walkthrough of critical journeys (A, B, C); NVDA + VoiceOver smoke test.
- **Quarterly:** external accessibility audit; remediations triaged into the next sprint.

---

## 10. Internationalization & Localization

- Default language: Bulgarian (`bg`); fully supported alongside English (`en`).
- All strings via `next-intl` keys; missing key = build-time error.
- Date, time, number, currency via Intl APIs and locale-aware components.
- Document templates (proposals, ESPD) ship in BG and EN; AI generation respects user-chosen output language.
- RTL not in scope for MVP; component library should not preclude future RTL support.

---

## 11. Content & Microcopy Guidelines

- **Tone:** clear, expert, calm. Avoid hype ("magical", "revolutionary"). Prefer "AI suggests" over "AI knows".
- **AI uncertainty:** when confidence is low, copy must say so — "We're not sure about this — please double-check the budget figure on page 12."
- **Errors:** name the problem, what to try, who to contact. Never blame the user. Include a reference ID for support correlation.
- **Empty states:** lead with the value, not the void — "Find tenders matched to your CPV codes — start with these 3 we picked for you."
- **Money & legal:** never invent legal advice; always link to the source document for compliance claims.

---

## 12. Privacy, Trust & Transparency

- Every AI surface explains, on demand, **what data was used** (company profile, this tender, prior proposals) — a "Why this?" link.
- Per-tenant data export and deletion accessible from Settings → Privacy (GDPR right to portability and erasure).
- Cookie banner: technical-only by default; analytics opt-in; bilingual.
- Audit log entries shown to admin users include actor identity and timestamp; PII is masked for non-admins.

---

## 13. UX Metrics & Acceptance

The PRD success metrics are tracked through the following UX-instrumented events:

| PRD metric | UX event(s) |
|---|---|
| Time-to-Value | `signup_completed` → `first_ai_summary_seen` |
| Trial-to-paid 20% | `trial_started` → `subscription_active` |
| Free-to-paid 5% | `free_signup` → `subscription_active` |
| 200 proposals in 90 days | `proposal_marked_submitted` |
| Human-rejection rate < 10% | `ai_suggestion_rejected / ai_suggestion_offered` |
| CSAT ≥ 90% | post-export micro-survey (1 question, dismissible) |

Per-screen acceptance criteria for each journey are owned by the relevant epic; this spec defines the screens and states they must cover.

---

## 14. Open Questions & Risks

| # | Topic | Owner | Notes |
|---|---|---|---|
| Q1 | Section locking concurrency model in Phase 2 — pessimistic lock vs. CRDT? | Engineering + UX | Affects editor UX significantly. |
| Q2 | Mobile editing scope — is "review + comment" enough for Phase 2 or do we need light edits? | Product | Decision drives investment in mobile editor. |
| Q3 | Score simulator confidence display — band vs. point + caveat? | Data Science + UX | A/B candidate post-launch. |
| Q4 | Bilingual proposal export — both languages in one document, or two files? | Legal + Product | Some buyers require both. |
| Q5 | Admin "impersonate tenant" — does it surface a banner inside the impersonated session? | Security + UX | Recommended: yes, persistent red banner. |

None of these block MVP delivery; each has a sensible default in this spec that we can iterate on.

---

## 15. Traceability — UX → PRD

| Journey / Screen | PRD Functional Requirements covered |
|---|---|
| Journey A (Onboarding → AI summary) | FR-1, FR-2, FR-4, FR-5, FR-12, FR-15, FR-16, FR-19, FR-21, FR-22, FR-23, FR-24 |
| Journey B (Proposal drafting) | FR-26, FR-27, FR-28, FR-29, FR-30, FR-33, FR-34, FR-25 |
| Journey C (Free → Paid) | FR-10, FR-11, FR-13 |
| Journey D (Collaboration, P2) | FR-6, FR-7, FR-31, FR-32, FR-37, FR-38, FR-39 |
| Journey E (Enterprise API) | FR-44 |
| Journey F (Admin operations) | FR-35, FR-42, FR-43 |
| Compliance Library | FR-36, FR-34, FR-43 |
| Notifications settings | FR-40, FR-41 |
| Multi-tenant chrome / RBAC | FR-7, FR-8, FR-9, NFR-7, NFR-8 |
| Accessibility section | NFR-18, NFR-19, NFR-20 |
| Performance budgets in patterns | NFR-1, NFR-2, NFR-3, NFR-4 |

---

## 16. Hand-off & Next Steps

1. **Design system pass** — produce shadcn-based component variants for AI badges, compliance chips, score meter, paywall card, streaming skeletons. Tracked under Epic FE-01 (frontend foundation).
2. **High-fidelity mocks** — Journey A and B end-to-end in Figma; Journey C from paywall onward.
3. **Per-epic UX reviews** — every epic's first story includes a "UX walk-through" acceptance check against this document.
4. **Accessibility checklist** — a 25-item per-screen checklist will be added to the PR template (see also `coding-standards/`).
5. **Living document** — this spec is amended via PR; major changes (new top-level IA entry, new persona) require sign-off from Product + Engineering leads.

---

*End of UX Specification v3.0.*
