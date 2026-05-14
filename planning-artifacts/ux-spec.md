---
stepsCompleted: ["step-01-init", "step-02-discovery", "step-03-personas", "step-04-journeys", "step-05-ia", "step-06-screens", "step-07-states", "step-08-accessibility", "step-09-complete"]
inputDocuments:
  - "eusolicit-docs/planning-artifacts/PRD.md"
  - "eusolicit-docs/project-context.md"
  - "eusolicit-docs/EU_Solicit_UX_Supplement_v1.md"
  - "eusolicit-docs/EU_Solicit_User_Journeys_and_Workflows_v1.md"
  - "eusolicit-docs/EU_Solicit_Requirements_Brief_v5.md"
project_name: "eusolicit"
author: "Deb"
date: "2026-05-14"
status: "Draft (autopilot, regenerated against PRD 2026-04-27)"
---

# UX Design Specification — EU Solicit

**Author:** Deb
**Date:** 2026-05-14
**Status:** Draft — autopilot-generated, derived from PRD + UX Supplement v1
**Companion documents:** `PRD.md`, `EU_Solicit_UX_Supplement_v1.md` (design system, screen-level specs), `EU_Solicit_User_Journeys_and_Workflows_v1.md` (full E2E dataflows)

---

## 0. Purpose & Scope

This document is the consolidated UX specification for EU Solicit. It defines:

1. **Personas** — who the platform is built for, what they need, what blocks them.
2. **Primary user journeys** — the step-by-step flows that the product must support end-to-end, from first contact through outcome.
3. **Information architecture & key screens/states** — the navigational model and the screens (with their states) that fulfill the journeys.
4. **Accessibility considerations** — the WCAG 2.1 AA commitments and the patterns that make them concrete.

The detailed visual design system (tokens, components, breakpoints, exhaustive screen anatomy) lives in the UX Supplement. This document focuses on UX intent and flow — it is the brief a designer, a frontend engineer, or a QA author should be able to read end-to-end in one sitting.

**Out of scope for this document:** marketing site, public landing pages, design-token-level specifications, brand identity guidelines, white-label theming (covered in the UX Supplement and brand guidelines).

---

## 1. Design Principles

Four principles inform every screen and state in this spec. They exist to keep designers and engineers aligned when a trade-off needs to be resolved.

1. **Clean, data-dense, professional SaaS aesthetic.** Linear / Vercel / Notion lineage. Restrained palette, typography-driven hierarchy, no decorative chrome. Density without claustrophobia: 15–25 table rows visible without scrolling on desktop; cards carry 4–6 data points before they feel overloaded.
2. **Content-first.** The chrome (sidebar, top bar, inspector) serves the content. Charts, tables, and the proposal editor extend to container edges. Skeletons mirror incoming layout so the page does not reflow.
3. **Progressive disclosure.** EU procurement is deep; show only what the user needs at this step. Primary information is always visible (title, deadline, budget, status, source). Secondary information is one click (full requirements list, evaluation weights). Tertiary is explicit drill-down (audit history, raw source data).
4. **Consistent patterns.** One data table, one filter pattern, one AI operation state machine, one empty-state template. A user who learns to filter opportunities filters proposals, grants, and audit logs the same way.

**Trust-first overlay (GovTech-specific).** Because users are about to commit to public-procurement bids, every AI output must be traceable to source, every destructive action must be confirmable, and every compliance status must be explainable. AI is presented as an assistant, never an oracle — confidence indicators, source citations, and a visible "Why?" affordance are required wherever the system asserts something.

---

## 2. Personas

Four personas drive product decisions. They are derived from the PRD journeys and validated against the market research underpinning the product brief.

### 2.1 Elena Petrova — The Overwhelmed Bid Manager *(Primary persona — Professional tier)*

| Attribute | Value |
|---|---|
| Role | Bid manager at a 40-person Bulgarian consulting firm in Sofia |
| Age / context | 34, MSc in EU public administration, 6 years in bidding |
| Team | Manages 5 contributors; 10–15 bids in flight at any time |
| Tools today | Excel, SharePoint, email threads, Word, AOP/TED portal logins |
| Tech proficiency | High — comfortable with SaaS, expects keyboard shortcuts |
| Time per bid (today) | 40–80 hours of partner + analyst time |
| Win rate (today) | ~22%, with bids lost to administrative defects (missed attachments, page-limit violations, missed deadlines) — not on merit |
| Primary pain | Volume and chaos: too many parallel bids, no single source of truth, fear of administrative rejection |
| Top jobs-to-be-done | (a) Triage incoming tenders in <30 minutes each; (b) get a defensible first draft in front of partners on day 1, not day 7; (c) ensure every submission is administratively clean |
| Success looks like | Submits 30% more bids in the same time, win rate climbs from 22% to 30%+, no administrative losses |
| Decision authority | Recommends tools; budget approval up to €5k/year without C-level sign-off |
| Quote | *"I'm not afraid of writing proposals. I'm afraid of missing something stupid in a 150-page PDF at 11pm on a Friday."* |

**Why she anchors the product.** Elena is the buyer and the daily power user. The MVP is built for her workflow; everything else (Maria's exploration, the admin operator, the enterprise developer) extends out from there.

### 2.2 Maria Dimitrova — The Cautious Solopreneur *(Acquisition persona — Free → Starter)*

| Attribute | Value |
|---|---|
| Role | Solo environmental-impact consultant, Plovdiv |
| Age / context | 41, ex-ministry analyst, 3 years independent |
| Team | Just herself; occasionally subcontracts |
| Tools today | Government portals, Google search, Word, a paid consultant once per year (€800–€1500) |
| Tech proficiency | Medium — comfortable with web apps, uneasy with anything that asks for a card upfront |
| Bids per year | 3–6 attempts, ~1 win |
| Primary pain | The procurement world is opaque. She does not know what she does not know. Every portal demands registration before she can even see a budget. |
| Top jobs-to-be-done | (a) See what's out there in her niche without committing money; (b) for one promising tender, get enough analysis to decide go/no-go; (c) pay only when she has a real opportunity in hand |
| Success looks like | Pays €29 for one month, wins a single €25k contract, keeps the subscription |
| Decision authority | Sole decision-maker; price-sensitive |
| Quote | *"I'll pay, but only after I've seen the thing I'm paying for."* |

**Why she matters.** Maria is the freemium funnel. The Free tier must give her enough signal to convert; the Starter price point (€29) must feel like a coffee-money risk against a four-figure prize. If Maria's path breaks, the bottom of the funnel collapses.

### 2.3 Operator Ivan — The Platform Admin *(Internal persona — admin portal)*

| Attribute | Value |
|---|---|
| Role | Platform operations / data-quality lead at EU Solicit |
| Age / context | 30, software-ops background, joined to keep the crawlers and compliance frameworks healthy |
| Tools today | Admin portal, Grafana, Sentry, ticketing |
| Tech proficiency | Very high |
| Primary pain | Source portals (AOP, TED) change HTML without warning; compliance regulations change without warning; tenants raise edge-case support tickets |
| Top jobs-to-be-done | (a) See crawler health at a glance and drill into failures; (b) edit compliance frameworks (ZOP, Horizon Europe) when regulations change; (c) impersonate a tenant to reproduce a support ticket; (d) audit any action across the platform |
| Success looks like | Crawler error rate stays <2%, compliance frameworks are current within 48h of a regulation change, every tenant ticket can be reproduced and resolved without engineering escalation |
| Decision authority | Operational; escalates schema changes to engineering |
| Quote | *"If I can't see it on the dashboard, it's broken and we don't know yet."* |

### 2.4 Developer Sasha — The Enterprise API Consumer *(Tertiary persona — Enterprise tier)*

| Attribute | Value |
|---|---|
| Role | Senior engineer at a large consulting firm with an Enterprise contract |
| Age / context | 29, integrates third-party APIs into the firm's Salesforce + BI stack |
| Tools today | Postman, GitHub, Salesforce APEX, OpenAPI specs |
| Tech proficiency | Expert |
| Primary pain | Bad docs, unstable schemas, rate limits that aren't documented |
| Top jobs-to-be-done | (a) Read OpenAPI docs and try a call in <10 minutes; (b) sync scored opportunities into Salesforce hourly; (c) get a notified webhook when an opportunity changes status |
| Success looks like | Partner dashboard in Salesforce shows scored EU Solicit opportunities in near real-time without a single support ticket |
| Decision authority | Influences renewal; if API is unstable, contract churns |

---

## 3. Primary User Journeys

The four journeys below are the spine of the product. Each is a step-by-step flow with entry triggers, system responses, and exit conditions. Together they cover ~85% of the platform's daily activity.

**Journey legend.** *U:* user action. *S:* system response. *State:* the resulting screen/component state. *AC:* acceptance criteria checkpoint.

### Journey J1 — Elena: First-time tender triage → AI-assisted proposal → submission *(MVP-critical)*

**Entry.** Elena receives a tender notification (LinkedIn / colleague / inbox) and signs up.

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 1 | U | Lands on `/signup`. Enters email + password OR clicks "Continue with Google". | Auth: Signup screen (empty state). |
| 2 | S | Sends verification email. Shows "Check your inbox" screen with resend timer. | Auth: Verify-email state. |
| 3 | U | Clicks the verification link. | Auth: Verified → Welcome wizard. |
| 4 | U | Welcome wizard (4 steps): (a) company name + VAT, (b) sectors via CPV-code picker, (c) primary languages, (d) "Are you bidding for AOP, TED, or both?". Skippable but a yellow banner reminds the user that AI quality depends on profile completeness. | Onboarding: Wizard steps 1–4. Progress bar at top. |
| 5 | S | Creates `Company` + `User` (role: Admin). Auto-enrolls 14-day Professional trial. Routes to Dashboard with an empty-state hero: "Add your first tender to see EU Solicit in action — upload a PDF or pick one from the live feed." | Dashboard: Empty state. |
| 6 | U | Clicks "Upload tender PDF" and drags in the 150-page document. | Opportunity Intake: Drop-zone state → Uploading state with progress bar. |
| 7 | S | Streams the file to MinIO, dispatches a `tender.uploaded` event. ClamAV scan runs. SirmaAI Document Parser agent begins. Live progress: *Parsing → Extracting requirements → Generating summary → Flagging risks.* TTFB <500ms. | Opportunity Detail: Analysis-in-progress state with 4-stage progress strip + live token stream into the summary panel. |
| 8 | S | Within ~90s: (a) executive summary appears (1 page, structured: scope, budget, deadline, evaluation criteria, key risks); (b) requirements checklist (auto-extracted, severity-tagged); (c) risk panel (3–8 flagged clauses with explanation + source-text excerpt + page anchor). | Opportunity Detail: Analysis-complete state. Tabs: Summary, Requirements, Risks, Documents, History. |
| 9 | U | Reviews the summary, expands one risk to see source-text quote and page anchor. Clicks "Looks promising — start proposal". | Opportunity Detail: Risk panel expanded; CTA bar at top. |
| 10 | S | Creates a `Proposal` linked to the opportunity. Pre-fills sections from the requirements checklist. Routes to the Proposal Editor with an empty body and section outline in the left rail. | Proposal Editor: Empty state with section outline. |
| 11 | U | Clicks "Generate first draft". Confirms model + tone in a small modal (default: Professional, Bulgarian language). | Modal: Generation-options state. |
| 12 | S | SirmaAI Draft Generator agent streams content section-by-section into the editor. Each section header shows a small spinner that flips to a checkmark on completion. User can read in real time. | Proposal Editor: Streaming state. Section-level progress indicators. |
| 13 | U | When draft is complete, invites two contributors via Settings → Team. Assigns them sections. | Team Invite modal → Editor with section assignees visible in left rail. |
| 14 | U *(later)* | Returns next day. Opens the proposal. Sees co-author presence avatars in the top right. Locks the "Financials" section while she edits. *(Phase 2 — single-user editing in MVP.)* | Proposal Editor: Active collaboration state. Section-lock indicator. |
| 15 | U | One week before deadline, clicks "Run compliance check". | Compliance Drawer: Running state → Results state. |
| 16 | S | Compares the proposal against the requirements checklist + the active ZOP compliance framework. Returns 3 findings: 2 missing attachments (high severity), 1 page-limit violation (medium). Each finding has a "Jump to source" affordance. | Compliance Drawer: 3 findings, severity-coloured. |
| 17 | U | Fixes findings. Re-runs check. All green. Clicks "Run score simulator". | Score Simulator modal: Running → Result state. |
| 18 | S | Returns a simulated evaluator score: 85/100 with a per-criterion breakdown and one suggestion ("Strengthen Methodology with concrete examples — current section lacks measurable outcomes."). | Score Simulator: Result state with criterion bars + suggestion card. |
| 19 | U | Applies the suggestion. Re-simulates: 92/100. Clicks "Export → PDF + DOCX". | Export modal: Format-selection → Generating → Ready state. |
| 20 | S | Generates artefacts in MinIO, returns signed URLs. | Export modal: Ready state with download buttons. |
| 21 | U | Downloads, submits manually to the source portal, marks proposal status "Submitted". | Proposal Detail: Submitted state. |
| 22 | S | Adds calendar reminder (24h before deadline). Adds proposal to "Pipeline Forecast" dashboard. | Dashboard: Pipeline-forecast widget updated. |

**Exit.** Elena has submitted one proposal end-to-end, converted from trial to paid before day 14. *AC: 14-day trial → paid conversion ≥20% requires this path to feel effortless on first run.*

**Critical UX checkpoints.**
- Step 5: empty-state hero must do real teaching, not generic welcome copy.
- Step 8: the analysis-complete state is the "aha" moment — must arrive in <120s p95 or the magic evaporates.
- Step 12: streaming must be visible and feel alive; a spinner with no content is not enough.
- Step 16: every compliance finding must link back to source text. Otherwise the user mistrusts the tool.
- Step 18: the score simulator must show *why*, not just *what*. A number without a criterion breakdown is hostile.

### Journey J2 — Maria: Free exploration → paywall → Starter upgrade → win

**Entry.** Maria signs up with just an email after hearing about the Free tier.

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 1 | U | `/signup` with email-only. No card required. | Auth: Signup state, Free-tier badge visible. |
| 2 | U | Lightweight onboarding (2 steps): sectors + region. | Onboarding: Compact 2-step wizard. |
| 3 | S | Routes to Opportunities list, filtered to her sectors. | Opportunities: List view, Free-tier overlay. |
| 4 | U | Filters by sector = "Environment", region = "Bulgaria". | Opportunities: Filtered list, 47 results. |
| 5 | S | For each row: name, deadline, contracting authority visible. Budget, full description, and AI analysis are blurred behind a lock icon with tooltip: "Upgrade to Starter to see full details." | Opportunities: List with Free-tier paywall overlays. |
| 6 | U | Opens a promising tender (Environmental Impact Assessment). | Opportunity Detail: Free-tier locked state. |
| 7 | S | Shows: title, deadline, authority, source. AI summary, requirements, budget, documents are gated. CTA: "Unlock for €29/month — cancel anytime." Below CTA, a "Why upgrade?" expander lists the unlocked features. | Opportunity Detail: Paywall state. |
| 8 | U | Clicks "Unlock". Stripe Checkout opens in a modal. | Billing: Stripe Checkout modal. |
| 9 | U | Pays. Returns to the Opportunity Detail. All fields unlock with a brief shimmer animation. | Opportunity Detail: Analysis-complete state. |
| 10 | U | Reads the summary, decides to bid. Clicks "Start proposal" (Starter tier allows 1 proposal in flight). | Proposal Editor: New proposal state. |
| 11 | — | (Follows steps 11–21 of Journey J1, single-user mode.) | — |

**Exit.** Maria submits the bid and keeps the €29/month subscription. *AC: Free → paid conversion ≥5% requires step 7's paywall to feel like an invitation, not a wall.*

**Critical UX checkpoints.**
- Step 5: locked fields must show *shape* (e.g., a blurred budget number with the € symbol still visible) so the user senses what's behind the lock.
- Step 7: "Why upgrade?" must be expandable inline, not a separate page — Maria will not navigate away.
- Step 9: the unlock shimmer is a 400ms confidence moment. Skipping it makes the purchase feel transactional rather than rewarding.

### Journey J3 — Operator Ivan: Compliance-framework update + crawler triage

**Entry.** A new EU regulation drops. Ivan needs to update the Horizon Europe framework and verify pipelines are healthy.

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 1 | U | Accesses `admin.eusolicit.com` over VPN. MFA challenge. | Admin Auth: MFA state. |
| 2 | S | Lands on Admin Dashboard: tenant count, crawler health, recent audit events, open support tickets. | Admin Dashboard: Default state. |
| 3 | U | Sidebar → Compliance Frameworks → Horizon Europe. | Compliance Framework Detail: View state. |
| 4 | U | Clicks "Edit". Adds 3 new rules with description, severity, source citation. Saves. | Compliance Framework Detail: Edit state → Save → View state with version bump. |
| 5 | S | Versions the framework (immutable history). Logs change to audit log. Notifies affected tenants via in-app banner. | Audit Log: New entry. Tenant in-app notification: New compliance rules in effect. |
| 6 | U | Sidebar → Data Pipelines. Sees AOP crawler at "yellow" (8% error rate, threshold 5%). | Data Pipelines: List with status pills. |
| 7 | U | Drills into AOP crawler. Sees error log: 14 of last 50 jobs failed with selector-not-found. | Crawler Detail: Logs state. |
| 8 | U | Clicks "File engineering ticket". A pre-filled ticket with logs and recent diff link opens. Submits. | Ticket modal: Pre-filled state → Submitted state. |
| 9 | S | Files ticket via Jira integration. Marks crawler in "investigating" status. | Crawler Detail: Investigating banner. |
| 10 | U | Spot-check: searches audit log for "Elena Petrova" to verify her recent actions look normal (responding to a tenant ticket). | Audit Log: Filtered search state. |

**Exit.** Compliance frameworks current, crawler degradation triaged, audit trail clean.

**Critical UX checkpoints.**
- Step 4: framework edits must be *versioned*, not destructive. Operators need to roll back without engineering help.
- Step 6–7: crawler health needs three states at a glance — green, yellow, red — with a one-click drill into logs.
- Step 10: audit search must be fast (sub-second) and tenant-scoped or global, with a clear toggle.

### Journey J4 — Developer Sasha: Enterprise API integration

**Entry.** Sasha's firm just signed an Enterprise contract; partners want EU Solicit opportunities in Salesforce.

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 1 | U | Logs into the client app as Enterprise Admin. Settings → API Keys. | Settings: API Keys, empty state. |
| 2 | U | Clicks "Create key". Names it "Salesforce sync". Selects scopes: `opportunities:read`, `proposals:read`. Sets rotation reminder = 90 days. | API Key modal: Create state. |
| 3 | S | Generates key. Shows it *once* with a "Copy & I've saved it" gate. Adds to API Keys list (masked). | API Key modal: Reveal-once state. |
| 4 | U | Opens `/api/docs` (OpenAPI / Swagger UI). Tries the `GET /opportunities?score_gte=80&status=new` call. Works. | API Docs: Try-it state. |
| 5 | U | Writes a script. Schedules it hourly. Configures a webhook endpoint in Settings → Webhooks for `opportunity.score_changed`. | Settings: Webhooks state. |
| 6 | S | Verifies the webhook URL by sending a `ping` event with HMAC signature. Sasha's endpoint replies 200. | Webhook Detail: Verified state. |
| 7 | U *(partner-side)* | Opens Salesforce dashboard, sees a tender scored 95 appear in real-time. Clicks the source link — deep links to Opportunity Detail in EU Solicit. | Opportunity Detail: Standard view. |

**Exit.** External integration live without a support ticket.

**Critical UX checkpoints.**
- Step 3: secret reveal must be *once only*. The UI must make this impossible to misunderstand.
- Step 5: webhook configuration needs an explicit verification step before events flow.
- Step 7: deep links from external systems must land on the right tenant context, with SSO handling the auth transparently.

### Cross-journey: Notifications & deadline pressure

A fifth meta-journey runs across all four: **the deadline never sleeps.** Notifications are not a feature — they are the connective tissue.

- New high-relevance opportunity → email digest (daily, 07:00 user-local) + in-app bell.
- Proposal deadline 7 days out → in-app + email; 24h out → in-app + email + (Pro+) calendar event reminder.
- Compliance check failed on submit attempt → in-app blocking modal, never silent.
- Co-author @mention → in-app + email (debounced 5min). *Phase 2.*
- Webhook delivery failed 3x → email to admin.

Notification preferences live in Settings → Notifications, with channel toggles per event type. Defaults are opinionated (deadlines on, marketing off).

---

## 4. Information Architecture

### 4.1 Top-level navigation — Client app (`eusolicit.com`)

```
Dashboard         — Personal home: deadlines, pipeline, recommended opportunities
Opportunities     — Searchable tender feed (AOP + TED); detail; saved searches
Proposals         — In-flight + submitted proposals; templates; lessons learned
Grants            — EU grants feed + grant-specific workflows  (Phase 2)
Analytics         — ROI, win rate, team performance, pipeline forecast
Calendar          — Deadlines, milestones, sync state with Google/Outlook (Phase 2)
Content Library   — Reusable content blocks; consortium partners
─────
Settings          — Profile, Company, Team & Roles, Billing, API Keys, Notifications, Integrations
User menu         — Profile, Switch workspace, Sign out
```

### 4.2 Top-level navigation — Admin app (`admin.eusolicit.com`)

```
Admin Dashboard      — Tenant count, system health, recent audit, support tickets
Organizations        — Tenants list, drill-down, impersonation (audited)
Users                — Cross-tenant user search
Subscriptions        — Plan health, churn risk, expansion candidates
Data Pipelines       — Crawler status (AOP, TED, EU Grants); job logs
Compliance           — Framework editor (ZOP, Horizon Europe, …); versioning
White-Label          — Tenant theming for Enterprise
Audit Logs           — Cross-tenant immutable audit search
System Config        — Feature flags, rate limits, model routing
```

### 4.3 Mental model

The client app is organized around the **opportunity → proposal → outcome** funnel. The Dashboard surfaces *what needs my attention today*; Opportunities is *what's out there*; Proposals is *what I'm working on*; Analytics is *how I'm doing*. Cross-cutting (Calendar, Content Library, Settings) supports the funnel rather than competing with it.

The admin app is organized around **operational health**: tenants, pipelines, compliance, audit. Admin users never need to bid; client users never need to operate. The two apps share design tokens but have different sidebars, different routing namespaces, and different auth requirements (admin = VPN + MFA).

### 4.4 Deep-link conventions

- Opportunities: `/opportunities/{public_id}` (e.g., `/opportunities/OPP-2026-0847`) — stable across crawler re-ingestion.
- Proposals: `/proposals/{uuid}` — UUID, not slug, because titles change.
- Tenant context: subdomain (`acme.eusolicit.com`) for Enterprise; path-based for SMB. The login flow resolves the active workspace and persists it in the auth store key `eusolicit-client-auth-store`.
- Deep links from emails / webhooks: must round-trip through the auth gate without losing the target URL.

---

## 5. Key Screens & States

This section enumerates the screens that materialize the journeys above, each with the states the UI must support. The UX Supplement contains pixel-level anatomy; what follows is the contract a screen must fulfill.

### 5.1 Auth & onboarding

| Screen | States |
|---|---|
| **Signup** | Empty · Validating · Error (email taken / weak password / Google failed) · Success → Verify-email |
| **Verify email** | Pending (with resend countdown 60s) · Verified · Expired link · Already verified |
| **Login** | Empty · Validating · Error · 2FA challenge · Success → workspace router |
| **Workspace router** | Single workspace (auto-route) · Multi-workspace (chooser) · No workspace (offer to create) |
| **Onboarding wizard** | Step 1 Company · Step 2 Sectors (CPV picker) · Step 3 Languages · Step 4 Region · Review · Skipped (with banner) |

**Critical state rules.**
- Verify-email must offer "Wrong email? Change it." inline; not buried in support.
- Login error messages must not leak account existence (anti-enumeration).
- The wizard must be *resumable* — a user who closes their browser mid-wizard returns to the same step.

### 5.2 Dashboard (Client)

| Variant | When | Composition |
|---|---|---|
| **Empty state — new user** | 0 opportunities saved, 0 proposals | Big hero: "Upload your first tender or browse the live feed." Two CTAs. Tour-mode launcher. |
| **Empty state — no activity** | Has used the product but no in-flight work | Compact hero: "Nothing in flight. Here are 3 opportunities matching your sectors." |
| **Active state** | Has in-flight work | Four widgets: (a) Deadlines this week, (b) Pipeline forecast, (c) Recommended opportunities, (d) Recent activity. |
| **Trial banner** | User on Day 1–14 of trial | Persistent top banner: "X days left on Professional trial — Add billing details." Dismissible per session. |
| **Trial-expired** | Trial ended without payment | Read-only overlay over the dashboard with "Reactivate" CTA; existing proposals viewable but not editable. |

### 5.3 Opportunities

| Screen | States |
|---|---|
| **List** | Loading (skeleton rows) · Empty (no matches) · Populated · Filtered · Free-tier paywall (per row) · Bulk-selection mode (for star, export) · Error (data-pipeline down) |
| **Detail** | Loading · Free-tier locked · Analysis-not-yet-run · Analysis-in-progress (streaming, 4-stage progress strip) · Analysis-complete · Analysis-error (with retry) · Archived |
| **Filter panel** | Closed (chips visible in bar) · Open (sheet on mobile, side panel on desktop) · Active (chips show count) |
| **Saved searches** | None · 1–N saved · Editing |

**Key components per detail screen.** Tabs: *Summary* (one-pager with citations) · *Requirements* (checklist with severity tags + page anchors) · *Risks* (clauses with explanation + excerpt + page anchor) · *Documents* (file list with parse status) · *History* (audit of who-did-what).

**Trust requirements.** Every AI assertion in Summary, Requirements, Risks must link back to source text (page + paragraph anchor). A "Why this score?" affordance is required on the relevance score and the risk severity.

### 5.4 Proposal editor

The proposal editor is the most complex screen in the product. It must support a wide range of states without becoming a maze.

| State | Description |
|---|---|
| Empty / outline-only | Just created. Left rail shows section outline derived from requirements; body is empty. |
| Generating (streaming) | AI is producing content. Section headers show spinner → checkmark. Body fills in real time. User can read but not edit until a section completes. |
| Editing — single user | Standard rich text editing (TipTap). No collaboration indicators. |
| Editing — multi-user | Co-presence avatars in top right. Section-lock indicators in the outline. @mention autocomplete. Comment thread side rail (toggle). *Phase 2.* |
| Section locked by other user | Body is read-only with a banner "Locked by Maria — Unlock available in 14 min." (or "Request unlock"). |
| Compliance check running | Compliance drawer open with running state. Editor remains usable. |
| Compliance findings present | Top banner: "3 issues found — Review." Click → drawer with findings, each with "Jump to source" anchor in the editor. |
| Score simulator open | Modal overlay. Editor underneath dimmed but not disabled. |
| Export in progress | Export modal with progress; editor remains usable in background. |
| Submitted (read-only) | Editor switches to read-only with a "Submitted on {date}" banner and an "Open new version" affordance. |
| Restored from history | Banner: "Viewing version from {date} by {user} — Restore | Discard." |

**Section outline (left rail).** Always visible at ≥1024px. Each section: title, assignee avatar (if any), lock indicator (if any), compliance issue badge (if any). Click to scroll-anchor. Drag to reorder (Phase 2).

**Right inspector (optional).** Contextual panel for the focused section: source-text citations, related requirements, lessons-learned snippets, content-block suggestions. Pinnable.

### 5.5 Compliance & scoring

| Screen / drawer | States |
|---|---|
| **Compliance check drawer** | Idle · Running (with stage progress) · Results — all pass · Results — N findings (grouped by severity) · Error |
| **Finding card** | Default · Expanded (shows source-rule text + jump-to-source) · Resolved (struck through, with restore) |
| **Score simulator modal** | Idle · Running · Result (overall score + per-criterion bars + 1–3 suggestions) · Suggestion-applied state |
| **ESPD builder** | Empty (auto-fill from company profile) · Editing · Validation errors · Ready to export · Exported XML |

### 5.6 Billing & subscription

| Screen | States |
|---|---|
| **Plan & billing** | Free tier · Trial (countdown) · Paid (active) · Past due (banner + CTA to update card) · Cancelled (read-only until grace period ends) |
| **Upgrade flow** | Tier-comparison page · Stripe checkout modal · Success state (with confetti, briefly) · Failure state (with retry) |
| **Usage meters** | Per-tier limits (proposals in flight, AI tokens, seats) with progress bars. Soft state at 80%, warning state at 100%, hard cap at 110% (with upgrade nudge). |
| **Invoices** | List · Empty · Drill-in (line items, VAT, download PDF) |

### 5.7 Team & permissions

| Screen | States |
|---|---|
| **Team members** | Empty (just the admin) · 1–N members · Invite pending · Invite expired · Role-edit state |
| **Invite modal** | Empty · Sending · Sent · Error |
| **Per-entity permission overrides** | Default-inherits state · Overridden state (with clear "remove override" affordance) |

### 5.8 Settings (catch-all)

Sub-pages: Profile · Company · Team & Roles · Billing · API Keys · Webhooks · Notifications · Integrations (Google Calendar, Outlook, Slack, Teams). Each follows the same form pattern: section header, descriptive subtitle, form fields with inline validation, sticky save bar.

### 5.9 Notifications

| Surface | Behavior |
|---|---|
| **Bell drawer** | Slides from top-right. Tabs: Unread · All. Each item: icon, title, snippet, timestamp, mark-read affordance. Click → deep link to the target entity. |
| **In-app banner** | Top of viewport, dismissible. Reserved for critical state changes (trial expiring, payment failed, compliance change). |
| **Toast** | Bottom-right, auto-dismiss 5s. For action confirmations ("Proposal saved", "Score simulator complete"). |
| **Modal (blocking)** | Reserved for: destructive confirmations, irreversible compliance violations on submit, payment failures mid-action. |

### 5.10 Admin screens

| Screen | States |
|---|---|
| **Admin dashboard** | Healthy · Degraded (one or more systems yellow) · Critical (red) |
| **Tenant list** | Filter, search, export. Drill into tenant view (audited). |
| **Tenant view** | Read-only by default. Impersonation mode requires explicit click + reason + audit entry. |
| **Compliance framework editor** | List · Detail (view) · Detail (edit) · Version history · Rollback confirmation |
| **Crawler dashboard** | Per-source health pill (green/yellow/red) + recent jobs + error log + manual re-run |
| **Audit log** | Search · Filtered (by tenant, actor, action, time range) · Export to CSV |

### 5.11 Cross-cutting states (apply to all screens)

| State | Pattern |
|---|---|
| **Loading** | Skeleton mirroring final layout. Never a centered spinner on a full page. |
| **Empty** | Icon + headline + 1–2 sentence description + primary CTA. Optional secondary link to docs. |
| **Error — recoverable** | Inline error region with retry button and "Contact support" link. |
| **Error — fatal** | Full-page error with error code, request-id (for support), and "Go home" CTA. |
| **Offline** | Persistent top banner: "You're offline. Changes will sync when you reconnect." Editor caches in IndexedDB. |
| **AI operation states** | One unified state machine: `idle → requesting → streaming\|processing → complete \| error \| cancelled`. Same visual vocabulary across all 11 AI-driven features. |
| **Confirmation — destructive** | Modal with action verb in the button (e.g., "Delete proposal", not "Confirm"), and a 3-second hold for irreversible actions. |
| **Trial / paywall / usage-cap** | Single, consistent paywall pattern with a clear CTA, the value proposition expanded inline, and a "Maybe later" exit. |

---

## 6. Interaction Patterns (selected)

A short list of patterns that recur and must be consistent.

1. **Command palette (Cmd+K).** Global search + actions. Fuzzy-matches opportunities, proposals, settings pages, and named actions ("Create proposal", "Run compliance check on current proposal"). Always available; first thing learned by power users.
2. **Keyboard shortcuts.** Documented in `?` overlay. At minimum: `Cmd+K` palette, `Cmd+B` sidebar toggle, `g d` go to dashboard, `g o` opportunities, `g p` proposals, `n` new proposal, `/` focus search, `Esc` close drawer/modal.
3. **Filter & chips.** Filter bar above every list. Active filters render as removable chips. The same component is used in Opportunities, Proposals, Audit log, etc.
4. **Inspector panel.** A right-side panel for contextual deep-dive without leaving the list/editor. Pinnable; persists across navigation within a section.
5. **Source citations.** Every AI claim has a citation. Hover or focus on a claim → tooltip with source. Click → scroll-anchor to source document with the relevant span highlighted.
6. **Streaming AI output.** Token-by-token render with cursor. Cancel button is always present and stops generation within ~1s. Partial output is preserved.
7. **Optimistic UI** for low-risk actions (star opportunity, mark notification read). Pessimistic UI for high-risk actions (delete proposal, change role, submit, pay).
8. **Form validation.** Inline + on submit. Errors appear below the field with a red icon. Submit button disabled until critical fields valid. Sticky save bar at the bottom of long forms with unsaved-changes count.
9. **Undo where possible.** Toast with "Undo" for: archive, delete (5s grace), bulk operations. Hard-delete confirmations explicit.

---

## 7. Accessibility (WCAG 2.1 AA — and a bit beyond)

NFR-18 mandates WCAG 2.1 Level AA across the user-facing application. Government-facing software in the EU is also expected to meet EN 301 549 (which references WCAG 2.1 AA). The following is non-negotiable.

### 7.1 Conformance commitments

| Area | Commitment |
|---|---|
| Standard | WCAG 2.1 Level AA across all user-facing pages (client + admin client-side) |
| Keyboard | Every interactive element reachable and operable by keyboard alone (NFR-19) |
| Screen readers | JAWS, NVDA, VoiceOver fully supported (NFR-20); ARIA used semantically, not decoratively |
| Color contrast | Text ≥ 4.5:1 (3:1 for ≥18px or bold ≥14px). UI components & graphics ≥ 3:1. Verified against design tokens in CI. |
| Motion | Respect `prefers-reduced-motion`: skeleton shimmer, page transitions, score-simulator chart animations, unlock shimmer all degrade gracefully. |
| Language | `lang` attribute set per page; `dir` attribute supports Bulgarian/English; mixed-language proposal content uses inline `lang` spans. |

### 7.2 Concrete patterns

1. **Focus management.**
   - Visible focus ring on every interactive element (`outline: 2px solid var(--color-primary-600)` with 2px offset); never `outline: none` without an alternative.
   - When a modal opens, focus moves to the first interactive element; on close, focus returns to the trigger.
   - In the proposal editor, section-jump (from compliance findings or outline) moves focus to the section heading, not the body, to preserve context.
2. **Semantic structure.**
   - One `<h1>` per page (the page title). Section headings step down without skipping levels.
   - Landmarks: `<nav>` (sidebar), `<main>` (content), `<aside>` (inspector), `<footer>` (where present).
   - Lists are `<ul>` / `<ol>`. Data tables are `<table>` with `<th scope>` and captions; cards-as-list use `role="list"` only when semantics require.
3. **Forms.**
   - Every input has a programmatically associated `<label>`. Placeholder text is never a label substitute.
   - Errors announced via `aria-live="polite"` regions; error text linked to the input via `aria-describedby`.
   - Required fields marked both visually and programmatically (`aria-required="true"`).
4. **Tables & lists.**
   - Sortable headers expose `aria-sort`. Bulk-selection state announces selection count to screen readers.
   - When a data table transforms to a card list on mobile, the underlying semantics still convey row identity.
5. **AI streaming output.**
   - Streaming regions use `aria-live="polite"` with `aria-busy="true"` during generation and `aria-busy="false"` on completion. Token-by-token updates are throttled in the live region so screen readers don't fire on every token.
   - A user-controlled "Pause announcements" affordance is available in the assistive-tech settings.
6. **Documents & PDFs.**
   - Uploaded PDFs are surfaced with their parsed text in the UI (not iframe-only) so screen readers can access content.
   - Exported PDFs are tagged PDFs with reading-order metadata where the generator supports it.
7. **Compliance findings & risks.**
   - Severity is communicated by color *and* by text label *and* by icon — never by color alone (WCAG 1.4.1).
   - "Jump to source" anchors the user with appropriate `aria-label` describing the destination ("Jump to clause 4.2 — page 18").
8. **Time-out and session expiry.**
   - Auth session warnings appear ≥30s before expiry with an option to extend.
   - The editor never silently times out; unsaved work is preserved in IndexedDB.
9. **Touch & pointer.**
   - Minimum target size 44×44 CSS pixels on touch devices.
   - Hover-only affordances (tooltips, row actions) have a tap-equivalent on touch.
   - Drag-and-drop has a keyboard alternative (move via menu) per WCAG 2.5.7.
10. **Internationalization.**
    - All copy externalized via `next-intl`. `pnpm check:i18n` blocks merges with missing translations.
    - Right-to-left support not in MVP but the layout tokens (logical properties: `margin-inline-start`, `padding-block-end`) are RTL-ready.

### 7.3 Testing & verification

| Layer | Approach |
|---|---|
| Automated CI | `@axe-core/playwright` on every E2E test of a critical user journey; build fails on Serious / Critical violations. |
| Component tier | Each shared UI component in `packages/ui` has a basic axe assertion in its test file. |
| Manual audit | Quarterly screen-reader walkthrough of: signup, opportunity detail, proposal editor, compliance drawer, billing flow, admin compliance framework editor. |
| Keyboard audit | Same six flows traversed with keyboard only before each major release. |
| Color contrast | Design-token-level contrast verified in `packages/ui` Storybook with a contrast-check addon. |
| User testing | At least one user with a visual or motor disability included in beta test cohorts. |

### 7.4 Known accessibility risks

| Risk | Mitigation |
|---|---|
| Rich text editor (TipTap) keyboard support gaps | Audit + supplement with explicit shortcut documentation; add toolbar fallback for shortcuts that fail with assistive tech. |
| AI streaming creates noisy screen-reader output | Throttled `aria-live`, completion-only announcement option, "Pause announcements" toggle. |
| PDF previews from source tenders may be image-based scans | Surface OCR'd text alongside the preview; never make critical info available only inside the PDF. |
| Color-dense charts in Analytics (Phase 2) | Pair every color with a pattern fill; data tables exposed as accessible alternatives. |

---

## 8. Open Questions & Decisions to Resolve

These are flagged for the next design / PM review. They do not block MVP build, but each affects a screen state above.

1. **Trial → paid conversion UI:** is the payment-required gate triggered by an explicit action (e.g., generating a proposal) or by a date-driven banner-then-overlay sequence? Recommendation: date-driven, with a softer gate that allows read-only access for 7 days post-trial.
2. **Proposal section-locking UX (Phase 2):** how is unlock-request communicated — banner-in-section, in-app notification, or both? Recommendation: both.
3. **Source citation density:** should every paragraph in a generated draft cite, or only assertions that map to requirements? Recommendation: only assertions mapped to requirements, to keep readability intact.
4. **Admin impersonation:** is impersonation read-only, or can the operator act-as the user? Recommendation: read-only by default, escalate via support workflow for action.
5. **Cross-language proposals:** if a tender is Bulgarian but the company drafts in English, do we auto-translate the final export or surface a translation step? Recommendation: explicit translation step with diff preview — never silent.

---

## 9. Traceability

| PRD section | Reflected in this UX spec |
|---|---|
| Success Criteria → User Success | §2 Personas, §3.J1 (time-to-value, confidence checkpoints) |
| Functional Requirements FR-1..FR-9 (Users & Tenants) | §5.1 Auth, §5.7 Team, §3.J1 onboarding |
| FR-10..FR-14 (Billing) | §5.6 Billing, §3.J2 upgrade path |
| FR-15..FR-20 (Discovery) | §5.3 Opportunities, §3.J1 step 6–9, §3.J2 step 4–7 |
| FR-21..FR-25 (AI Analysis) | §5.3 Opportunity detail (Analysis-in-progress, -complete), §3.J1 step 7–8, §6.5 source citations |
| FR-26..FR-33 (Proposal generation) | §5.4 Proposal editor, §3.J1 step 10–22 |
| FR-34..FR-39 (Compliance & workflow) | §5.5 Compliance & scoring, §3.J1 step 15–18 |
| FR-40..FR-44 (Notifications & admin) | §5.9 Notifications, §5.10 Admin, §3.J3, §3.J4 |
| NFR-18..NFR-20 (Accessibility) | §7 in full |
| Domain-specific GovTech requirements | §1 trust overlay, §5.10 admin, §7 audit & contrast |

---

## 10. Glossary

| Term | Meaning |
|---|---|
| Opportunity | An external tender or grant ingested from AOP, TED, or EU Grants. Read-only from the user's perspective. |
| Proposal | A user-owned draft authored against an opportunity. Mutable; versioned. |
| Compliance check | Validation of a proposal against (a) the opportunity's extracted requirements and (b) the applicable compliance framework. |
| Compliance framework | An admin-managed ruleset (ZOP, Horizon Europe, …) that proposals are checked against. Versioned. |
| Score simulator | An AI estimator of evaluator scoring; not a guarantee, presented with criterion-level breakdown. |
| Workspace | A tenant: one company. Users may belong to multiple workspaces and switch between them. |
| ESPD | European Single Procurement Document — a standardized self-declaration form. Generated XML. |
| Inspector | The right-side contextual panel pattern used across lists and the editor. |

---

*End of UX Specification — eusolicit.*
