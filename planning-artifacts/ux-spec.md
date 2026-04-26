---
title: UX Specification — EU Solicit
project_name: EU Solicit
author: Deb
date: 2026-04-25
version: '2.0'
status: canonical
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/epics.md
  - eusolicit-docs/planning-artifacts/project-context.md
supersedes: ux-spec.md v1.0 (2026-04-24)
---

# UX Specification — EU Solicit

**Author:** Deb
**Date:** 2026-04-25
**Version:** 2.0 (Canonical — supersedes initial draft of 2026-04-24)
**Status:** Final — synthesized from PRD v2.0 (2026-04-25) and the detailed UX Design Specification (`ux-design-specification.md`).

---

## 1. Purpose & Scope

This document is the canonical UX specification for the EU Solicit MVP web application. It defines the personas the product must serve, the primary user journeys, the screens and states required to support those journeys, and the accessibility considerations that gate release.

It is intentionally complementary to — not a replacement for — `ux-design-specification.md`, which holds the visual design foundation, design tokens, component anatomies, and emotional/strategic narrative. Where this document references tokens, breakpoints, or component atoms, the source of truth is the design specification.

**Scope:** Web application (`EUSolicit.com`) — Next.js 14 client portal (port 3000) and admin portal (port 3001). Out of scope for the MVP: native mobile apps, electronic bid submission, on-prem deployments.

---

## 2. Key User Personas

The platform supports six personas across two surfaces (client portal and internal admin portal). Personas are not roles in the RBAC sense; one human may wear multiple persona hats, and the company-level role (`admin`, `bid_manager`, `contributor`, `reviewer`, `read_only`) governs what they can do in the system.

### 2.1 Bid Manager (Primary persona — drives the workflow)

- **Demographic:** 35–55, often the head of business development at a SMB or a consulting firm; juggles 5–20 active opportunities at once.
- **Context:** Coordinates the entire bid lifecycle — pipeline triage, bid/no-bid decisions, deadline discipline, team assignment, executive sign-off.
- **Pain points:** Tracking dozens of portals (AOP, TED, EU Funding Portal); missed deadlines; opaque go/no-go decisions; inability to forecast win-rate to leadership.
- **Goals:** Never miss a relevant tender; convert pipeline into wins with measurable ROI; defend pursuit decisions with data.
- **UX needs:** A pipeline-centric dashboard, AI-scored matches, deadline alerts (in-app + email + calendar), a defensible Bid/No-Bid scorecard, ROI analytics.
- **Frequency:** Daily, multiple sessions per day. Heavy desktop user.
- **Mapped role:** `bid_manager` or `admin`.

### 2.2 Technical Writer / Proposal Author

- **Demographic:** 28–50, technical writer or domain expert (engineer, consultant) who authors proposal narrative.
- **Context:** Drafts the actual response document — technical methodology, project plan, team CVs, references.
- **Pain points:** "Blank page paralysis"; manually mining past proposals for reusable boilerplate; reformatting boilerplate to match each RFP's structure.
- **Goals:** Produce a high-scoring first draft fast; reuse proven content blocks safely; never overwrite a teammate's work.
- **UX needs:** AI draft generation (RAG over Company Profile Vault + past bids), a clean Tiptap split-pane editor, slash commands for inserting content blocks, inline AI assistance, conflict-safe collaboration.
- **Frequency:** Bursts of 4–8 hour drafting sessions during active bids.
- **Mapped role:** `contributor` (most common) or `bid_manager`.

### 2.3 Legal / Compliance Reviewer

- **Demographic:** 30–60, in-house counsel or compliance officer.
- **Context:** Reviews tender documents and proposal drafts for legal risk and regulatory compliance (ZOP, EU Directive 2014/24/EU, programme rules).
- **Pain points:** Scanning 200+ page contracts for a single problematic clause; ESPD transcription; chasing missing mandatory documents.
- **Goals:** Spot risky clauses before signature; verify ESPD accuracy; certify compliance before submission.
- **UX needs:** Auto-flagged clauses with severity badges and citations back to the source paragraph; pass/fail compliance validator; ESPD auto-fill with provenance.
- **Frequency:** Episodic per bid (1–4 hours per opportunity).
- **Mapped role:** `reviewer` (comment-only) or `bid_manager`.

### 2.4 Financial Analyst / Budget Builder

- **Demographic:** 28–55, financial controller or grant accountant.
- **Context:** Builds bid pricing and EU grant budgets under co-financing rules and cost-category constraints.
- **Pain points:** Spreadsheet errors; eligibility-rule violations discovered post-submission; pricing without competitive intelligence.
- **Goals:** Submit a budget that passes programme validation on the first try; price to win.
- **UX needs:** Structured budget builder with real-time rule validation, competitive pricing assistant grounded in historical award data, clear allocation summaries.
- **Frequency:** 1–2 deep sessions per bid.
- **Mapped role:** `contributor` with entity-level access to financial sections.

### 2.5 Company Admin

- **Demographic:** Owner/manager at a small firm; CTO/COO at a larger one.
- **Context:** Manages the tenant — users, RBAC, company profile vault content, billing, calendar integrations.
- **Pain points:** Surprise quota exhaustion; manual onboarding; confusing tier upgrade pathways.
- **Goals:** Predictable billing; clear visibility into team activity; self-serve subscription management.
- **UX needs:** Usage dashboard against tier limits, in-app Stripe Customer Portal link, RBAC matrix, vault management UI, audit log access.
- **Frequency:** Weekly check-ins; episodic during user/billing events.
- **Mapped role:** `admin`.

### 2.6 Free Explorer (Lead-gen persona)

- **Demographic:** Anyone evaluating the platform — prospective customer, journalist, public-sector observer.
- **Context:** Browses the public catalogue with severely restricted metadata (six fields only).
- **Pain points:** Wants to evaluate fit before paying; needs enough signal to convert.
- **Goals:** Decide whether to sign up for a paid tier or 14-day Professional trial.
- **UX needs:** A frictionless catalogue browse experience, clear tier-comparison and upgrade CTAs, no surprise paywalls mid-flow.
- **Frequency:** One-time or sporadic.
- **Mapped role:** Unauthenticated or `Free` tier authenticated user.

### 2.7 Platform Admin (Internal — admin portal)

- **Demographic:** EU Solicit operations engineer or compliance officer.
- **Context:** Manages tenants, compliance frameworks, AI quotas, crawler health. Operates the VPN-restricted admin portal (`localhost:3001` in dev).
- **Pain points:** Dispersed operational signals; opaque tenant state; manual framework configuration.
- **Goals:** Catch crawler regressions early; configure compliance frameworks per opportunity class; investigate billing or RBAC incidents.
- **UX needs:** System-wide health dashboards, tenant inspector with audit log, framework editor, crawler run timeline, override tools with mandatory justification.
- **Frequency:** Daily.
- **Mapped role:** Platform-level admin (separate auth boundary from tenant `admin`).

---

## 3. Primary User Journeys

Each journey is described as a step-by-step flow. Steps reference the PRD's functional requirements (FRn) and acceptance criteria (AC-x.y.z) where the journey enforces a specific contract.

### 3.1 Journey A — Opportunity Discovery & Qualification

**Persona:** Bid Manager. **Trigger:** Daily digest email or in-app notification. **Outcome:** Opportunity moved to active pipeline with a workspace, or archived with rationale.

1. **Alert.** User receives a tier-appropriate digest email or in-app notification listing new AI-scored matches (FR5).
2. **Land on dashboard.** Clicking the email opens the Dashboard's "Top AI Matches" widget; the opportunity is visually distinguished as "new since last visit."
3. **Browse list.** User filters the catalogue by CPV / country / value band / deadline window (FR2). Results render in < 200 ms p95 (AC-6.1.4).
4. **Open detail.** Selecting a row opens the **Opportunity Detail Workspace** on the *Overview* tab.
5. **Read AI summary.** A streaming SSE-rendered executive summary (FR4) appears with first token < 500 ms (AC-6.2.3). Key fields (deadline, budget, CPVs, contracting authority, submission channel) are extracted and pinned above the fold.
6. **Skim risks & checklist.** User glances at the *Documents* tab (risk flags) and *Checklist* tab (mandatory requirements) for a fit gut-check.
7. **Open Bid/No-Bid modal.** User triggers the **Bid/No-Bid Decision** scorecard (FR19). Strategic-fit, capability, cost-to-bid, probability-of-win sliders feed a live score gauge.
8. **Decide.** User clicks **Pursue** (creates active workspace, surfaces in pipeline, schedules deadline reminders, optionally syncs to calendar) or **Pass** (archives with optional rationale; counted in lost-opportunity analytics).
9. **(Admin override)** A Company Admin can override a "Pass" with a mandatory justification field; override is captured in the immutable audit log (AC-6.6.1, AC-6.7.1).

**Success criteria:** Decision reached in < 5 minutes for a typical opportunity. Zero accidental data loss on cancel.

### 3.2 Journey B — Document Analysis & Proposal Drafting

**Persona:** Technical Writer (with Bid Manager observing). **Trigger:** "Pursue" decision from Journey A. **Outcome:** A reviewable first-draft proposal in the Tiptap editor.

1. **Workspace initiation.** From the active opportunity workspace, user uploads any supplementary documents (drawings, annexes). Each upload streams through ClamAV (AC-6.2.2); infected files are quarantined visibly.
2. **AI ingestion.** A streaming "AI is analyzing 3 of 12 documents…" indicator appears on the *Documents* tab (FR7). Each document's row updates with a confidence score and risk-flag count as parsing completes.
3. **Review extraction.** User opens the *Checklist* tab to inspect the auto-generated requirement checklist (FR8). Each item links back to the source paragraph in the parsed PDF (cited evidence — never a black box).
4. **Acknowledge risks.** User opens the *Risks* sub-view; high-severity clauses (uncapped liability, unilateral termination) are listed with "Acknowledge / Escalate to Legal" actions (AC-6.2.4).
5. **Generate first draft.** User clicks **Generate Proposal Draft** (FR11). The *Proposal Editor* tab activates and the AI streams content section-by-section into Tiptap. Cancellation is always possible; the cancel button is keyboard-reachable.
6. **Edit & collaborate.** User edits the draft. Slash commands (`/`) insert reusable content blocks from the Vault. The right-side AI Assistant panel offers contextual regeneration, citations, and section-level scoring suggestions.
7. **Concurrent editing.** A teammate joins. Section presence indicators show who is editing where. Conflicting saves resolve via optimistic locking (`content_hash` SHA-256) — stale hash returns 409 with a structured conflict dialog (AC-6.3.1).
8. **Inline scoring.** User triggers the **Scoring Simulator** (FR12) on a section; a localized score (e.g., 85/100) and prescriptive improvement prompts surface inline.
9. **Save & continue.** Auto-save runs continuously; the editor never loses work on browser refresh. Quota for AI draft generations decrements only on success and is refunded on 5xx (AC-6.3.4).

**Success criteria:** First draft visible in Tiptap within 60 seconds of click. No black-box AI output (every AI block is citable).

### 3.3 Journey C — Compliance Check & Submission Export

**Persona:** Bid Manager + Legal Reviewer. **Trigger:** Draft considered "ready for review." **Outcome:** Final PDF/DOCX export and (separately) ESPD XML.

1. **Run validator.** Bid Manager clicks **Run Compliance Validator** (FR13). The system validates the proposal against the opportunity's assigned framework (ZOP, EU Directive 2014/24/EU, programme rules).
2. **Review report.** A pass/fail report renders per requirement, grouped by framework section. Each unmet item links to the offending location in the proposal and the source paragraph in the tender.
3. **Resolve flags.** Legal Reviewer (with `reviewer` role — comment-only) leaves threaded comments on flagged sections. Technical Writer addresses each, marking them resolved.
4. **ESPD auto-fill.** Legal Reviewer opens the **ESPD Builder** (FR26). Fields auto-populate from the Company Profile Vault; provenance badges (`Auto-filled from vault: Certifications.pdf v3`) appear next to each filled field. Manual fields are flagged for completion.
5. **ESPD validation.** User clicks **Validate**; output is checked against the official EU XSD (AC-6.7.2). Errors surface inline with offending field highlighted.
6. **ESPD export.** User downloads the validated XML.
7. **Final compliance pass.** Bid Manager re-runs the validator. The compliance metric card shows a green ring at 100%; the accordion is empty.
8. **Approval pipeline.** The proposal moves through any configured approval workflow (FR19). Approvers receive in-app + email notifications.
9. **Export proposal.** User clicks **Export** and chooses PDF (WeasyPrint) or DOCX (python-docx). Rendering runs off the event loop (AC-6.3.2). A success toast offers "Download" and "Copy share link."
10. **Outcome capture.** After submission, the Bid Manager logs the outcome (won/lost, evaluator feedback) in the **Bid History** view, feeding the Lessons Learned engine for future drafts (FR23, KB-2).

**Success criteria:** ESPD validates first try ≥ 90% of cases. Exports never block the API. Audit log row exists for every state transition.

### 3.4 Journey D — EU Grant Application (Specialization)

**Persona:** Financial Analyst + Bid Manager. **Trigger:** Opportunity classified as EU grant (Horizon Europe, Digital Europe, CEF, ESF+, ERDF). **Outcome:** Eligible budget + logframe + consortium plan attached to the proposal.

1. **Eligibility match.** From the opportunity Overview, system displays the **Eligibility Matcher** result (FR21) — programme name, eligibility rationale, gap items.
2. **Build consortium.** Bid Manager opens the **Consortium Finder** (FR22); recommended partner organizations rank by CPV overlap, geography, prior co-participation.
3. **Generate logframe.** User clicks **Generate Logframe**; a programme-templated DOCX skeleton renders with editable cells (AC-6.4.2).
4. **Build budget.** Financial Analyst opens the **Budget Builder**. Cost categories (personnel, travel, subcontracting, overheads) appear as line groups; co-financing rules and category caps are enforced inline (AC-6.4.1).
5. **Validate & refine.** Live validator highlights any over-cap line item with the specific rule violated.
6. **Attach to proposal.** Budget and logframe attach as proposal sections; subsequent compliance validator runs include programme-specific rules.

### 3.5 Journey E — Free → Paid Conversion

**Persona:** Free Explorer → Starter/Professional. **Trigger:** Catalogue browse hits a paywall. **Outcome:** Stripe Checkout completed; user lands on onboarded dashboard.

1. **Browse public catalogue.** Free user sees opportunities with the six allowed metadata fields only (AC-6.1.2).
2. **Hit paywall.** User clicks "View full tender" or "Generate AI summary"; an in-line upsell card explains what the action unlocks per tier.
3. **Compare tiers.** User opens the **Pricing** page; the four-tier comparison table highlights the recommended tier for their use.
4. **Start trial or checkout.** Professional offers a 14-day no-card trial; Starter/Pro/Enterprise route to Stripe Checkout. VAT validation runs via VIES; if VIES is unavailable, registration proceeds with `vat_validation_status: pending` (AC-6.9.3).
5. **Stripe Checkout.** User completes payment.
6. **Post-checkout onboarding.** User returns to a streamlined onboarding wizard: company profile → vault seeding → CPV/country preferences → first AI match.
7. **Activation moment.** Within 5 minutes of checkout, user generates their first AI summary on a real tender — the "Aha!" moment.

### 3.6 Journey F — Platform Admin Operations (Admin portal)

**Persona:** Platform Admin. **Trigger:** Daily ops review or alert. **Outcome:** Crawler issue triaged or compliance framework updated.

1. **Land on admin dashboard.** Active-tier distribution, SSE concurrency, webhook latency, usage-sync drift gauges visible (NFR13).
2. **Inspect crawler.** Admin opens the *Crawlers* timeline; failing run shows red. Drill-down reveals the structured error and last-success timestamp.
3. **Tenant inspector.** Admin opens a tenant; sees usage vs. tier, audit-log feed, RBAC matrix, billing state.
4. **Framework edit.** Admin opens the **Compliance Framework Editor** (FR24); edits a rule; change is captured in the immutable audit log.
5. **Override.** If a tenant requests a manual quota grant, admin issues it with mandatory justification (audit-logged).

---

## 4. Key Screens & States

This section enumerates the screens that must exist for the MVP, the layout pattern each follows, and the states each must explicitly handle. Visual tokens, component anatomies, and breakpoint specifics are in `ux-design-specification.md`.

### 4.1 Global Application Shell

- **Pattern:** Persistent collapsible left sidebar + sticky top bar + optional right inspector panel (Sheet).
- **Sidebar:** Dashboard · Opportunities · Active Pipeline · Knowledge Base (Vault) · Calendar · Analytics · Settings · Billing.
- **Top bar:** Global `Cmd+K` search · breadcrumb · notification bell · tenant/tier badge · user avatar with menu.
- **States:** Default · sidebar collapsed (icons only) · search palette open · notification drawer open · loading (skeleton shell on first paint).
- **Breakpoints:** ≥ `xl` (1280 px) full 3-pane editor; `lg` (1024 px) sidebar + content; `md` (768 px) sidebar collapses to drawer; `sm` (640 px) read-only/light-action mode.

### 4.2 Authentication Screens

- **Login:** Email + password, Google OAuth2 button, "Forgot password" link.
- **Register:** Email, password (≤ 128 chars; NFR6), company name, VAT ID (optional), tier intent (Free/Trial).
- **Email verification:** Success state, expired-link state, resend CTA.
- **Forgot / Reset password:** Token-bearing reset form; expired/invalid token state.
- **States across all auth screens:** idle · validating · submitting · field-level error · global error · success/redirect.

### 4.3 Dashboard

- **Layout:** 3-column responsive grid.
- **Widgets:**
  - *Top AI Matches* — cards with match-percentage ring, deadline countdown, primary "Open" CTA.
  - *Upcoming Deadlines* — switchable list/calendar mini-view.
  - *Active Pipeline Snapshot* — stage funnel with counts and total pipeline value.
  - *Win Rate & ROI* — sparkline + headline metric.
  - *AI Usage* — quota ring against tier limit; upgrade CTA when threshold crossed (AC-6.8.1).
- **States:** First-run empty state (with "Set your CPV preferences" CTA) · loading (skeleton matching widget anatomy) · error (per-widget retry) · degraded (Redis fail-open notice on usage widget).

### 4.4 Opportunity Catalogue (Listing)

- **Layout:** Filter rail (left) · data table (center, `@tanstack/react-table`) · optional inspector panel.
- **Columns:** title · buyer · CPV · country · value · deadline · match score · status.
- **Filters:** country · CPV · value band · deadline window · status · saved-filter dropdown.
- **States:** Idle · loading skeleton (table rows) · empty (with "broaden filters" CTA) · server error · partial results (Free-tier banner explaining truncated metadata).
- **Tier behaviour:** Free users see exactly the six allowed fields; the "view tender" cell renders an upsell card instead of a link (AC-6.1.2).

### 4.5 Opportunity Detail Workspace

- **Layout:** Tabbed workspace.
  - **Tabs:** *Overview* · *Documents* · *Checklist* · *Risks* · *Proposal Editor* · *Tasks & Team* · *Activity*.
- **Header:** title · buyer · deadline countdown · framework badge · primary CTA (varies by state — "Run Bid/No-Bid", "Generate Draft", "Run Validator", "Export").
- **States per tab:**
  - *Overview*: AI summary streaming · summary ready · summary failed (retry).
  - *Documents*: pre-upload · upload-in-progress · scanning (ClamAV) · parsed (with confidence) · failed.
  - *Checklist*: empty · partial · complete · with overdue items (red badge).
  - *Risks*: no-risk-found · severity-grouped list · acknowledged/escalated chip per item.
  - *Proposal Editor*: see §4.7.
  - *Tasks & Team*: kanban or list toggle · DAG dependency visualization (FR18).
  - *Activity*: chronological audit log feed.

### 4.6 Bid/No-Bid Decision Modal

- **Pattern:** Multi-step modal (or Sheet on smaller viewports) with sticky **Score Gauge** that recalculates on every input.
- **Sections:** Strategic fit · Capability · Cost to bid · Probability of win · Justification (for overrides).
- **States:** In-progress · validation error (missing required slider) · score below pursue threshold (warning) · admin-override mode (justification field becomes mandatory).
- **Decision actions:** **Pursue** (primary) · **Pass** (secondary) · **Save & continue later** (tertiary).

### 4.7 Proposal Editor (Split-Pane)

- **Pattern:** Three-pane split (UX-DR2).
  - **Left rail:** Document outline · per-section completion status · linked requirements badge.
  - **Center canvas:** Tiptap WYSIWYG with slash commands (UX-DR5) and AI diff blocks (UX-DR3).
  - **Right rail:** Tabbed contextual panels — *AI Assistant* · *Compliance* · *Comments* · *Pricing Assistant*.
- **States:**
  - *Draft generation streaming* — section-level skeletons (UX-DR10) replaced as tokens arrive.
  - *Idle editing* — auto-save indicator in the toolbar.
  - *Conflict (409)* — modal showing server hash, server version, base version with three actions (Take theirs · Keep mine · Merge manually).
  - *Section locked by another user* — lock icon + presence avatar.
  - *AI suggestion pending* — diff block with green/red highlights and per-chunk Accept/Reject (UX-DR3).
  - *Cancellation* — clean cancel with terminal SSE event (NFR3); no orphan locks.
  - *Quota exhausted* — inline upsell card, no destructive interruption.

### 4.8 ESPD Builder

- **Pattern:** Guided wizard (TurboTax-style) with section progress stepper.
- **Sections:** Identification · Exclusion grounds · Selection criteria · Reduction of candidates · Concluding statements.
- **State per field:** auto-filled (with provenance tooltip) · auto-filled but stale · manual entry required · validation error.
- **Footer:** **Validate XML** · **Export XML** · **Save draft**.
- **States:** Draft · validating · valid (XSD pass) · invalid (per-field errors) · exported.

### 4.9 Knowledge Base / Company Profile Vault

- **Pattern:** Folder tree (left) · file grid/list (center) · file inspector (right).
- **File card:** type icon · name · version · upload date · uploader · "used by N proposals" badge.
- **States:** Empty · uploading · scanning (ClamAV) · ready · superseded (older versions accessible — AC-6.5.1) · quarantined.

### 4.10 Bid History & Lessons Learned

- **Pattern:** Filterable table + drilldown.
- **Per-row:** opportunity · outcome (won/lost/withdrawn) · evaluator score · key lesson summary.
- **Detail view:** Win/loss reason editor · attached evaluator feedback · "feed into next draft" toggle.

### 4.11 Calendar & Deadlines

- **Pattern:** Month/week/list toggle.
- **Events:** opportunity deadlines · internal milestones · approval deadlines.
- **Integration UI:** "Connect Google Calendar" / "Connect Outlook" with re-auth banner if refresh token has permanently failed (AC-6.6.2).

### 4.12 Analytics & Reporting

- **Sections:** Win rate over time · pipeline forecast · ROI per bid · sector intelligence (CPV / country / buyer volumes).
- **States:** Loading · empty (insufficient data) · partial (gated metrics on lower tiers) · error.

### 4.13 Settings & Administration (Tenant)

- **Sub-screens:** Profile · Team & RBAC · Billing & Tier · Integrations · Notifications · Audit log · API keys (Enterprise).
- **Billing state machine surface:** active · past_due (revoked entitlements banner) · canceled · trialing (countdown).
- **RBAC matrix:** users × roles × entity overrides (FR17).

### 4.14 Notification Center

- **Pattern:** Right-side drawer.
- **Categories:** Deadlines · AI completions · Approvals · Comments · System (regulation changes — FR25).
- **States:** Unread · read · grouped (e.g., "5 new matches today").

### 4.15 Pricing & Upgrade Surfaces

- **Pricing page:** Four-tier comparison table; recommended tier highlight.
- **Inline upsell card:** Triggered at every paywall; shows the specific feature gated and the minimum tier to unlock.
- **Stripe handoff:** Loading state during checkout-session creation; explicit failure state if Stripe is unreachable (R-2 mitigation).

### 4.16 Admin Portal (Internal — port 3001)

- **System dashboard:** Crawler health timeline · SSE concurrency gauge · webhook latency histogram · active-tier distribution · recent audit-log feed.
- **Tenant inspector:** Per-tenant usage, RBAC, billing state, audit-log filter, override actions.
- **Compliance framework editor:** CRUD on frameworks, rule library, framework-to-opportunity-class mapping (FR24).
- **AI agent registry:** From `config/agents.yaml` — circuit-breaker state, rate-limiter state, recent run telemetry.

### 4.17 Cross-cutting States (must exist on every screen)

- **Loading:** Structure-matching skeletons (UX-DR10), never generic spinners except for sub-second transitions.
- **Empty:** A single primary action + brief explanation; never a blank container.
- **Error:** Recoverable (retry CTA) or fatal (link to status page / support).
- **Permission denied:** RBAC failures show a friendly "You don't have access" panel; cross-tenant access returns the same screen as "not found" (AC-9 / NFR10) — never leaks ownership.
- **Quota exhausted:** Non-destructive upsell card.
- **Streaming:** SSE-driven views show first token < 500 ms p95, a heartbeat indicator, and a guaranteed terminal event (success / cancel / error).
- **Offline / network failure:** Toast + retry; persisted local state where applicable.
- **i18n:** Every screen renders cleanly in EN and BG; no string concatenation; no untranslated keys (`pnpm check:i18n` enforced).

---

## 5. Accessibility Considerations

EU Solicit serves municipalities, regulated industries, and public-sector users; accessibility is not optional. The platform targets **WCAG 2.1 Level AA** at minimum, with selected AAA enhancements for critical flows (compliance, billing).

### 5.1 Standards & Compliance

- **WCAG 2.1 AA** baseline across all screens.
- **EN 301 549** alignment (EU public-sector accessibility regulation) — relevant because municipalities are a target segment.
- **GDPR + accessibility intersection:** Cookie / consent surfaces are keyboard-navigable and screen-reader announced.
- **Compliance verification:** Automated axe-core checks in CI on every PR; manual audit before each major release.

### 5.2 Perceivable

- **Color contrast:** Text ≥ 4.5:1; large text and non-text UI ≥ 3:1. Indigo `#4f46e5` on white passes AA at body sizes; verified for the entire palette in the design tokens.
- **Color is never the sole signal.** Risk severity badges combine color + icon + text label. Compliance ring uses color + percentage + state label.
- **Reduced motion:** `prefers-reduced-motion` honoured (UX-DR8) — slide transitions, parallax effects, and confetti micro-interactions disable cleanly.
- **Text resize:** Layout supports 200% browser zoom without horizontal scroll on the primary content column.
- **Alt text:** Every meaningful image, icon, and chart has an accessible name. Decorative icons use `aria-hidden="true"`.
- **Charts:** Recharts visualizations expose tabular alternatives via "View as table" toggle.

### 5.3 Operable

- **Keyboard navigation:** Every interactive element is reachable and operable via keyboard. Tab order matches visual order. Focus traps in modals/sheets release on close.
- **Visible focus:** A 2-px indigo focus ring on a 1-px white offset is mandatory (UX-DR7); never `outline: none` without a replacement.
- **Skip links:** "Skip to main content" and "Skip to sidebar" appear on focus at the top of every page.
- **Keyboard shortcuts:** `Cmd+K` for global search; slash commands inside the editor; all shortcuts discoverable via `?` help overlay; all shortcuts can be disabled in user preferences.
- **No keyboard traps in editor:** Tiptap editor escapes via `Esc` or `Cmd+E` to the surrounding shell.
- **Pointer alternatives:** Drag-and-drop in the kanban / DAG views has a keyboard-equivalent move-to-status menu.
- **Timing:** No session timeouts under 20 minutes without warning. Auto-save + 5-minute warning toast on idle expiry.

### 5.4 Understandable

- **Language attribution:** Document `lang` attribute set; mixed-language content (e.g., Bulgarian tender excerpt inside an English UI) wrapped in `lang="bg"`.
- **Error messages:** Specific, actionable, and announced to assistive tech. Form errors include the field name and the corrective action.
- **Plain-language defaults:** Default microcopy avoids jargon; expert terms (CPV, ESPD, ZOP) link to a glossary on first use per session.
- **Predictability:** Navigation order is consistent across the shell. Destructive actions require explicit confirmation (export, delete, override).
- **i18n:** EN and BG ship at GA with full string parity (`pnpm check:i18n`); RTL is not in scope for MVP but the layout architecture leaves no LTR-only assumptions in place.

### 5.5 Robust

- **Semantic HTML.** Headings form a logical outline; landmarks (`<header>`, `<nav>`, `<main>`, `<aside>`) are present.
- **ARIA only where native semantics are insufficient.** Radix primitives provide WCAG-correct popovers, dialogs, listboxes, comboboxes; we do not re-roll our own.
- **Live regions:**
  - SSE streaming AI output uses `aria-live="polite"` for incremental tokens; `aria-busy="true"` while in progress.
  - Compliance failures and quota-exhaustion warnings use `aria-live="assertive"` (interrupt).
  - Chat-like AI assistant turns use `role="log"` with `aria-relevant="additions"`.
- **Naming:** Every actionable element has an accessible name (icon-only buttons require `aria-label`).
- **Form labels:** Every input has a programmatically associated `<label>`; placeholder text is never the only label.

### 5.6 Specific to Streaming AI UX

- **Announcement etiquette.** Token-level changes are not announced individually (would flood screen readers); instead, section-level updates are announced ("AI generated section: Project Plan") via a dedicated polite live region.
- **Cancellation.** A keyboard-reachable Cancel button is present whenever a stream is active and announces "Generation cancelled" on activation.
- **Failure.** SSE stream failures are announced with an assertive live region and offer a retry button focused after announcement.

### 5.7 Specific to the Tiptap Proposal Editor

- **Editor role:** Tiptap exposes the `textbox` role with multiline support and rich-text capabilities.
- **Slash command menu:** Implemented as a `combobox` with `aria-activedescendant`; arrow-key navigable; `Esc` to close.
- **Section locking:** Lock-state changes announce ("Section locked by Maria Petrova") via a polite live region.
- **AI diff blocks:** Each diff chunk is a labelled region with Accept/Reject buttons that clearly identify the chunk being affected ("Accept change to paragraph 3 of Methodology").

### 5.8 Responsive Strategy & Accessibility Intersection

- **Desktop-first** for complex flows (proposal editing, compliance review).
- **Tablet (`md` 768 px–`lg` 1024 px):** Read-mostly with light actions (approve Bid/No-Bid, comment). Right-rail panels collapse to bottom sheets.
- **Mobile (`sm` ≤ 640 px):** Notifications, deadline review, summary reading, approval gestures. The full proposal editor degrades gracefully — AI sidebar collapses behind a floating action button; section navigation moves to a top sheet.
- **Touch targets:** Minimum 44 × 44 CSS pixels on all interactive elements at touch breakpoints.
- **Orientation:** No content depends on a single orientation.

### 5.9 Acceptance Gates

A release is accessible only if **all** the following hold:

1. axe-core CI run is clean across primary screens (Dashboard, Catalogue, Detail Workspace, Editor, ESPD Builder, Settings).
2. Manual keyboard-only walkthrough completes Journeys A–D end-to-end.
3. A screen-reader walkthrough (NVDA + VoiceOver) of Journey B (drafting + streaming AI) completes without unannounced state changes.
4. Color tokens verified against contrast thresholds at every text/background pairing in production CSS.
5. `prefers-reduced-motion` walkthrough has no rejected animations.
6. EN / BG i18n parity check (`pnpm check:i18n`) passes; no untranslated keys in production bundles (PRD §10.10).

---

## 6. Open Questions & Forward Work

- **Mobile native app:** Out of scope for MVP; revisit post-GA based on usage analytics on `sm` breakpoint.
- **DE / FR / RO localization:** Behind feature flag; UX patterns must remain LTR-only safe but copy expansion (~30%) reserved in container widths.
- **Voice input for editor:** Not in MVP; keep editor toolbar extensible to add later without layout breakage.
- **Custom theming for white-label (Enterprise):** Architecturally supported via CSS custom properties; UX guidance for partners is a post-MVP deliverable.
- **AAA contrast for compliance-critical surfaces:** Aspirational; explicit elevated contrast for the compliance ring + ESPD validator is recommended for v2.1.

---

*End of canonical UX Specification v2.0. Supersedes v1.0 (2026-04-24). Visual design tokens, component anatomies, and emotional/strategic narrative live in `ux-design-specification.md`. Functional contracts and acceptance criteria are authoritative in `PRD.md`.*
