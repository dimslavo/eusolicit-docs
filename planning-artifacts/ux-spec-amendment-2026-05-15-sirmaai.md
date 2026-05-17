---
stepsCompleted: ["amendment-authoring"]
amendmentTarget: "eusolicit-docs/planning-artifacts/ux-spec.md"
sourceArtefacts:
  - "eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md"
  - "eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md"
  - "eusolicit-docs/planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md"
  - "eusolicit-docs/planning-artifacts/epics/E25-knowledge-base-lifecycle.md"
  - "eusolicit-docs/planning-artifacts/epics/E26-agent-driven-ingestion.md"
  - "eusolicit-docs/planning-artifacts/epics/E27-crm-via-mcp.md"
  - "eusolicit-docs/planning-artifacts/epics/E28-webhook-reconciliation.md"
  - "eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-15.md"
status: "Draft (PM-authored under bmad-agent-pm; intended for Sally finalisation)"
author: "📋 John (PM) — draft on Sally's behalf"
date: "2026-05-15"
---

# UX Spec Amendment — 2026-05-15 (SirmaAI Integration Pivot)

> **Status:** PM-authored draft (📋 John). This amendment is intended to be reviewed, refined, and finalised by Sally (UX) before consumption by dev. It exists because the IR pass on 2026-05-15 found that `ux-spec.md` (regenerated 2026-05-14 *against PRD 2026-04-27*) did not absorb the 2026-05-12 SirmaAI amendment — FR-45 through FR-56 had no UX coverage. This delta supplies that coverage at the same depth as the existing `ux-spec.md` §5 Key Screens & States. It does not replace `ux-spec.md`; on Sally approval, sections below are merged into the base spec.

---

## 0. Why this amendment

The base `ux-spec.md` covers FR-1 through FR-44 (its §9 Traceability table makes the cap explicit). The 2026-05-12 SirmaAI PRD amendment added FR-45..55 and the 2026-05-15 IR remediation added FR-56. Those eleven new FRs introduce **eight user-facing surfaces** that don't exist in the base spec:

1. **Workspace provisioning state** (FR-45) — UI must indicate when a fresh tenant is mid-provisioning vs ready.
2. **Workspace archive cross-substrate confirmation** (FR-46, FR-56) — admin must understand they're destroying state in two places at once, irreversibly.
3. **Opportunity Qualification result panel** (FR-47) — fit-score, gap analysis, pursue/monitor/decline recommendation, citations.
4. **Opportunity Quantification result panel** (FR-48) — effort, win prob, expected value, threshold, citations, tier gate.
5. **Knowledge Base dashboard** (FR-49, FR-50, FR-51, FR-52) — upload, processing badge, quota meter, search, replace/archive/restore, citation deep-links.
6. **Async-run progress UX** (NFR-2 amended) — distinct from the existing sub-15s streaming state; long-running analyses need a different progress model.
7. **Tenant-visible degraded-mode banner** (NFR-26, E28 S28.07) — "AI analysis temporarily unavailable" state.
8. **CRM via SirmaAI MCP** (FR-53) — OAuth-connect flow, MCP-server status, workspace CRM dashboard widget, conflict-log surface.

This document specifies each surface with the same state-table contract used in §5 of the base spec.

---

## 1. Updated principles (delta to §1 of base spec)

Append to existing principle 4 ("Consistent patterns"):

> 4b. **Cross-substrate operations are always confirmed.** Any user action that mutates state across both EU Solicit Postgres AND the SirmaAI substrate (archive workspace, delete KB artefact, sever CRM connection) requires a typed confirmation (user types the resource name) and presents a checklist of what will be deleted on each side. The platform never silently destroys SirmaAI-side state.

Append to existing principle 4 trust overlay:

> **AI-temporary-failure overlay (NEW).** When the SirmaAI substrate is degraded (circuit-breaker open >5min per E28 S28.07), the platform must surface a tenant-visible banner explaining that AI-analysis features are temporarily unavailable while existing analyses are unaffected. The banner is the only acceptable signal — silent feature degradation is forbidden. This principle is the user-facing manifestation of NFR-26.

---

## 2. Persona delta

The existing four personas (Elena, Maria, Operator Ivan, Developer Sasha) cover the new surfaces without addition. However, two of them gain new jobs-to-be-done:

**Elena (Bid Manager) — new JTBD:**
- (e) Upload past proposals + ESPD templates to KB so AI qualification and proposal-drafting agents ground in her firm's institutional memory.
- (f) Read AI qualification + quantification outputs WITH citations to KB sources, so she can sanity-check before pursuing.

**Operator Ivan (Platform Admin) — new JTBD:**
- (e) See orphan SirmaAI Project list (E24 S24.05) and one-click reprovision failed tenants.
- (f) Drain the webhook DLQ (E28 S28.06) when poison events accumulate.
- (g) Confirm cross-substrate erasure completion for GDPR-Article-17 requests (FR-56, E24 S24.07).

No new personas needed.

---

## 3. User journey amendments

### J1 (Elena) — insert KB onboarding before tender upload

Insert between steps 4 and 5 of base spec's J1:

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 4a | S | After wizard completion, dashboard empty-state hero now includes a secondary CTA: "Boost AI quality — upload your past proposals + company profile to your Knowledge Base." Skippable. | Dashboard: Empty state with KB onboarding nudge. |
| 4b | U (optional) | Clicks the KB nudge. Drops 3 past proposals + 1 ESPD template into the KB upload zone. Each shows a "Processing…" badge that flips to "Ready" within ~30s. | KB Upload state (see §5.11 below) → KB List with ready badges. |
| 4c | S | Once KB has artefacts, the existing "Upload your first tender" hero card adds a subtitle: "AI will ground its analysis in your KB." Visual reinforcement of trust. | Dashboard: Empty state with KB-grounded reassurance. |

### J1 — insert qualification + quantification between analysis and proposal start

Insert between steps 8 and 9 of base spec's J1:

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 8a | S | Within ~30s of analysis completion, the **Qualification Panel** auto-runs (per FR-47 auto-trigger on `opportunities.ingested`). Displays: fit-score gauge (0-100, color-banded), recommended action badge (pursue/monitor/decline), gap analysis as a bullet list with citation chips linking back to KB sources, confidence band. | Opportunity Detail: Qualification panel (Analysis-complete substate). |
| 8b | U | (Pro+ tier) Clicks "Run quantification" CTA visible only on pursue-recommended opportunities. | Opportunity Detail: Quantification panel (Running state). |
| 8c | S | Quantification triggers async-run (typically 1-3min). UI shows **async-progress state** (see §5.12 below): stepper showing "Submitted → Estimating effort → Scoring win probability → Computing expected value" with elapsed-time and aria-live announcements. | Opportunity Detail: Quantification panel (Async-progress state). |
| 8d | S | On completion: effort (person-days), win probability (0-1 with confidence band), expected value (EUR), bid/no-bid threshold (EUR), with a "Why this estimate?" affordance expanding citation list. | Opportunity Detail: Quantification panel (Result state). |

### J1 — insert source citations in proposal draft

Update step 12 of base spec's J1:

> | 12 | S | SirmaAI Draft Generator agent streams content section-by-section into the editor. Each section header shows a small spinner that flips to a checkmark on completion. **Generated assertions that ground in KB sources display a small numbered citation marker next to the text**; clicking the marker opens a side panel with the source passage, source file name, and "Open in KB" link. User can read in real time. | Proposal Editor: Streaming state + Citation surface. |

### J3 (Operator Ivan) — insert orphan reconciliation + erasure proof

Append two new flows to base spec's J3:

**J3 step extension — Cross-substrate health check:**

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 11 | U | Sidebar → SirmaAI Health → Orphan Projects. Sees 2 EU Solicit companies without a healthy SirmaAI Project (one failed mid-provisioning, one missing on SirmaAI side after a SirmaAI-side incident). | Admin: Orphan Projects list. |
| 12 | U | Clicks "Reprovision" on the first orphan. Confirms in a modal showing what will be created (Project + KB + agents + MCP stubs). | Admin: Reprovision confirmation modal. |
| 13 | S | Triggers `POST /api/v1/admin/companies/{id}/reprovision` (E24 S24.05). Modal transitions to running state with stepper; on completion, orphan row disappears from list with a toast confirmation. | Admin: Reprovision running → success toast. |

**J3 step extension — GDPR Article 17 cross-substrate erasure verification:**

| # | Actor | Step | State / Screen |
|---|---|---|---|
| 14 | U | Receives a GDPR Art. 17 erasure request for company X. Navigates to Admin → Compliance → Erasure Requests. Initiates erasure for company X. | Admin: Erasure Requests list → Initiate Erasure modal. |
| 15 | S | Confirmation modal lists EVERY substrate that will be touched (Postgres rows, SirmaAI Project, KB files in SirmaAI storage, MCP-server secrets, CRM tokens at provider). Admin must type the company name to confirm. | Admin: Typed-confirmation modal. |
| 16 | S | Erasure executes; UI shows two-ACK progress: ✅ Postgres rows deleted (with count) → ⏳ SirmaAI files deleted (per KB-file count) → ✅ certified. Audit-log entry is link-out for verification. | Admin: Erasure-in-progress → Certified. |

### J2 (Maria) and J4 (Sasha) — no changes

Maria's free-tier journey does not touch SirmaAI surfaces beyond the cosmetic agent rename (covered in base spec). Sasha's enterprise API consumer journey is unaffected — SirmaAI run-execution remains platform-internal per PRD amendment §SaaS B2B Integrations rewrite ("Does not expose direct SirmaAI agent execution — agent runs are platform-internal and not part of the external contract").

---

## 4. Information Architecture deltas

### 4.1 Client app sidebar — insert Knowledge Base entry

Update base spec §4.1 client nav (insert between Content Library and Settings):

```
Dashboard
Opportunities
Proposals
Grants                  (Phase 2)
Analytics
Calendar                (Phase 2)
Content Library
Knowledge Base          NEW — Per-workspace SirmaAI KB management (FR-49..52)
─────
Settings — Profile · Company · Team & Roles · Billing · API Keys · Notifications · Integrations
            · CRM Connections      NEW — Dynamics 365 + HubSpot OAuth (FR-53)
User menu
```

### 4.2 Admin app sidebar — insert SirmaAI Health + Erasure entries

Update base spec §4.2 admin nav (insert between Data Pipelines and Compliance):

```
Admin Dashboard
Organizations
Users
Subscriptions
Data Pipelines
SirmaAI Health          NEW — Tenant Project status, orphan list, DLQ depth, reconciler health
Compliance
   ├ Frameworks         (existing)
   ├ Erasure Requests   NEW — GDPR Art. 17 cross-substrate flow (FR-56, E24 S24.07)
   └ Sub-processors     NEW — SirmaAI residency disclosure source (linked to Trust Center)
White-Label
Audit Logs
System Config
```

### 4.3 Mental model addendum

The client mental model gains a "feed your KB to get better AI" loop running orthogonal to the opportunity → proposal → outcome funnel. The KB is the input substrate; opportunities and proposals are where its value materialises. Surface this to the user via the empty-state nudge in J1 step 4a and via persistent KB-usage hints in qualification/quantification panels ("This estimate is grounded in N artefacts from your KB — review to improve accuracy").

---

## 5. Key Screens & States — deltas

### 5.11 Knowledge Base *(NEW SECTION)*

| Screen | States |
|---|---|
| **KB Dashboard** | Empty (no artefacts) · Loading (skeleton) · Populated (filterable list) · Filtered · Quota approaching (80%) · Quota exceeded (402 paywall) · Error (SirmaAI unreachable — soft fail with retry CTA) |
| **Upload zone** | Idle (drag-and-drop affordance + button) · Validating file type · Uploading (progress bar + cancel) · Processing (yellow "Processing…" badge) · Ready (green badge with checkmark) · Failed (red badge with retry + error reason) · Over-size (rejected pre-upload with format guidance) |
| **Artefact detail** | View metadata (filename, category, tags, size, uploaded-by, SHA-256, processing state) · Edit metadata (tags + category) · Search-within (semantic search filtered to this file) · Replace (uploads new body, preserves row UUID) · Archive (soft-delete) · Restore (within 30-day window) · Restore-expired (410 Gone with "re-upload required" guidance) |
| **Semantic search** | Empty query · Loading · Populated (ranked passages with file name + relevance score + click-through to artefact detail) · No results · Error |

**Components per KB Dashboard:**
- **Quota meter** (top-right): radial progress with used / total bytes; color bands (green <80%, yellow 80-95%, red >95%); click-through to billing if Pro tier and over quota.
- **Filter rail** (left): by `artefact_category` (tender / profile / proposal / espd_template / rubric / other), by tag, by processing state.
- **Artefact list**: card or row toggle; each item shows filename, category badge, processing state badge, size, uploaded-at, action menu (replace / archive / search within).

**Critical state rules:**
- The Processing → Ready badge transition must be visually salient (color + brief shimmer). Users who upload a file and walk away should see a clear "this is ready to be used by AI" signal on return.
- Archive is a soft confirm: a modal warns "this file will be deleted from SirmaAI. You can restore within 30 days by re-uploading the original from your local copy — EU Solicit does not keep the body." Typed confirmation NOT required (low blast radius — one file at a time).
- Cross-tenant: existence-leakage protection means a wrong-tenant artefact ID returns 404, not 403. UI never reveals that a file exists under another tenant.
- WCAG 2.1 AA: drag-and-drop has keyboard fallback (button + native file picker); upload progress announced via `aria-live="polite"`; processing state changes announced.

### 5.12 Async-run progress *(NEW SECTION — applies to Quantification, deep ESPD generation, any SirmaAI run >15s)*

Extends the base spec's "Generating (streaming)" state which only covered sub-15s sync runs.

| State | Description |
|---|---|
| **Submitted** | User triggered. Backend has accepted (202) and returned `run_id`. UI shows: "Analysis queued" + spinner + elapsed time (mm:ss). Cancel button (cancellation may not actually stop the run on SirmaAI side — surface this in tooltip). |
| **Running — stage 1..N** | UI subscribes to SSE channel `/api/v1/opportunities/{id}/analyses/stream` (E26 S26.07). Shows a horizontal stepper with named stages (`Estimating effort → Scoring win probability → Computing expected value` for quantification; equivalent stages for other agents). Current stage pulses; completed stages have checkmarks. `aria-live="polite"` announces stage transitions. |
| **Stuck** | If no SSE event received for >90s AND no terminal webhook arrived, UI shifts to "Still working… (this can take up to 3 minutes)" copy. Does NOT auto-fail — the reconciler (E28 NFR-26) is converging the run; visible failure waits for the reconciler verdict. |
| **Completed** | Result panel rendered. SSE channel terminates. |
| **Failed** | Result panel shows error category (KB context unavailable / SirmaAI rate-limit / structured-output mismatch / unknown). Re-run CTA available. `error_message` from `client.opportunity_analyses` surfaced in expandable detail. |
| **Cancelled** | User cancelled. UI shows "Cancelled — no charge to your tier usage" (since cancellation happens before terminal billing). |

**Critical state rules:**
- The "Stuck" state is essential. Users will refresh the page; the reconciler (FR-55) is the source of truth, so silent backend recovery must be visible to the user as patient progress rather than panic.
- All async-progress states must support tab-switching: the run continues in the background and re-render correctly on focus.
- For Pro+ users on quantification (E26 S26.05 tier gate), an aborted run does NOT count against their quota — surface this in the cancel-confirmation copy.

### 5.13 Tenant degraded-mode banner *(NEW SECTION)*

Triggered by E28 S28.07 when SirmaAI circuit-breaker open >5 minutes.

| State | Description |
|---|---|
| **Hidden** | Default — no banner. `GET /api/v1/system/status` returns healthy. |
| **Degraded** | Banner appears at top of every authenticated client-app page (NOT login / signup / billing — only post-auth surfaces). Copy: "AI analysis temporarily unavailable. Existing analyses are unaffected; new analyses will resume automatically." Color: warning-yellow background, attention but not error-red. Dismissible per-tab (resets on next page load). `role="status"` + `aria-live="polite"`. |
| **Recovering** | (Optional intermediate state) When circuit re-closes, show "AI analysis is recovering — new requests may queue briefly." for up to 1 minute hysteresis. Lower visual priority. |

**Critical state rules:**
- Banner must NOT block any UI action. Users can still browse opportunities, view existing analyses, draft proposals, manage KB metadata. Only NEW SirmaAI calls fail; those failures must point to the banner ("AI analysis is currently unavailable — see banner at top for details").
- Banner must NOT mention the underlying "circuit-breaker open" term. User-facing language only.
- Persisted across pages until status flips back; do not re-show on every page load while degraded.
- Admin app does NOT use this banner (admins need to *see* the degraded state in the dedicated SirmaAI Health screen, not be reassured).

### 5.14 CRM connection (Settings → CRM Connections) *(NEW SECTION — FR-53)*

| Screen | States |
|---|---|
| **CRM Connections list** | Free / Starter tier (locked behind Pro+ paywall with upgrade CTA) · Pro+ tier with no connections · Pro+ with 1-2 active connections (Dynamics 365 + HubSpot) · Connection mid-OAuth · Connection failed health-check |
| **OAuth connect flow** | "Connect" CTA per provider → external OAuth (provider's UI; we never see the password) → callback land state (success / failed-state) → final list state showing connection as "Active" |
| **Connection detail** | Connected since · Last health check · Token-expiry countdown · Disconnect CTA (typed-confirmation modal — disconnecting deletes MCP-server secrets on SirmaAI side) · Conflict log link (E27 S17.34) |
| **Conflict log** | List of MCP-tool invocations with conflict outcomes (LWW resolution); filter by date / provider; export to CSV |

**Critical state rules:**
- OAuth callback failure copy must be specific ("HubSpot returned 'access_denied' — please retry and approve the requested scopes"). Generic "OAuth failed" copy is hostile.
- Token expiry surface: when token has <7 days to expiry, show a yellow badge with "rotating soon" tooltip (informational — the 6h Beat per E27 S17.33 handles it automatically; users don't act here, they just see it).
- Disconnect confirmation lists what will happen: "Your OAuth tokens will be revoked at {provider}. Existing CRM enrichment data already attached to opportunities will be preserved. Future qualification + quantification runs will not enrich via this CRM."

### 5.15 Cross-substrate workspace archive (Settings → Company → Archive) *(NEW SECTION — FR-46 + FR-56)*

| State | Description |
|---|---|
| **Default** | Section visible only to workspace admins. Description: "Archiving this workspace deletes both EU Solicit data and SirmaAI-side data. This action cannot be undone." |
| **Archive button clicked** | Typed-confirmation modal opens. Shows three-column table of what will happen: (1) **EU Solicit Postgres** rows soft-deleted (X proposals, Y opportunities tracked, Z team members) → (2) **SirmaAI Project** soft-deleted, API key revoked, MCP-server secrets deleted (A agents, B workflows, C KB files) → (3) **External CRMs**: OAuth tokens revoked at provider. User must type the workspace name to enable Confirm. |
| **Confirmed → in-progress** | Modal shifts to in-progress state with three-ACK tracker: ✅ EU Solicit rows soft-deleted (instant) → ⏳ SirmaAI Project archival (up to 24h per FR-46) → ⏳ CRM tokens revoked (within 10 min). UI explains: "You can close this tab — completion notification will arrive via email." |
| **Completed** | All three ACKs green. Audit-log entry link surfaced. Workspace switcher (in user menu) removes the archived workspace; user is redirected to the next available workspace OR to a "no workspaces available — create one" screen. |
| **Partially failed** | If SirmaAI archival times out (>24h with no ACK), state freezes at "⏳ SirmaAI archival — investigating". Admin operator alerted via existing notification surface; user is told "We're working on completing SirmaAI-side cleanup. You'll receive an email when complete." Workspace remains soft-deleted in EU Solicit but its switcher entry is hidden. |

**Critical state rules:**
- Typed-confirmation gate is mandatory. This is the most destructive single user action in the platform.
- Three-ACK tracker matches the GDPR Article 17 audit pattern (E24 S24.07) — same UI primitive, different copy.
- The 24h FR-46 SLA cannot be circumvented by user "I want it gone NOW" — explain that SirmaAI side archival is async and audit-traceable. If user demands faster, escalate to admin (who can use the admin reprovision/archive endpoints in Compliance → Erasure Requests for true Article 17 cases).

---

## 6. Interaction Patterns (delta to §6 of base)

### 6.X Citation chips *(NEW PATTERN)*

A citation chip is a numbered marker `[¹]` placed inline next to an AI-generated assertion. Hover reveals the source-passage tooltip; click opens a side panel with:
- Source file name + category badge.
- The exact passage that grounded the assertion (excerpt, ≤300 chars).
- "Open in KB" link → KB Artefact Detail with the passage highlighted (anchor link by char-offset).
- "Source archived" fallback state if the underlying KB file has been archived since the analysis (linked file UUID still resolves, but body is gone — show "Source no longer accessible" + last-known-filename).

**Where citation chips appear:**
- Opportunity qualification panel (gap-analysis bullets)
- Opportunity quantification panel (effort + win-prob justification expander)
- Proposal draft (any AI-generated assertion that grounded in KB)
- ESPD auto-fill output (E11 amendment — each field with provenance)

**State rules:**
- Citation numbers reset per panel (1, 2, 3 within a panel — not platform-wide).
- Hovering a citation highlights the matching passage in the side panel if open.
- Keyboard: citation chips are focusable, Enter opens the side panel, Esc closes.

---

## 7. Accessibility deltas

### 7.5 New surfaces — explicit WCAG commitments

For each new screen group introduced by this amendment:

| Surface | WCAG 2.1 AA requirements |
|---|---|
| KB Dashboard (§5.11) | Keyboard-navigable upload (button + file picker); `aria-live` for upload + processing state; sufficient contrast on processing badges; quota meter has accessible name + value |
| Async-run progress (§5.12) | Stepper stages announced via `aria-live="polite"`; cancel button always reachable via keyboard; "Stuck" state copy is plain language |
| Degraded-mode banner (§5.13) | `role="status"` + `aria-live="polite"`; dismissible via keyboard (Esc); banner is not redundant focus target during normal navigation (focus stays where user was) |
| CRM connections (§5.14) | OAuth callback redirects preserve focus on the connection list; typed-confirmation modal trap-focus correctly; conflict log table sortable via keyboard |
| Workspace archive (§5.15) | Typed-confirmation input has explicit label + describedby of consequences; three-ACK tracker announced via `aria-live="polite"` as ACKs land |
| Citation chips (§6.X) | Focusable + activatable via keyboard; tooltip dismissible via Esc; side panel restores focus on close |

### 7.6 Known gaps to escalate

- Screen-reader pronunciation of "SirmaAI" — verify with JAWS/NVDA/VoiceOver; may need `aria-label` override on first occurrence per page.
- Citation-chip numbered marker UX with screen readers: `[¹]` may be read as "citation 1" which is fine; verify the side-panel relationship is announced via `aria-controls`.

---

## 8. Open questions for Sally / Deb

1. **KB onboarding nudge — opt-in vs opt-out vs required.** Current draft makes it opt-in via dashboard CTA (J1 step 4a). Should it instead be a step 5 of the onboarding wizard? Trade-off: friction vs AI-quality-at-first-run. PM recommendation: opt-in for v1; revisit after seeing onboarding-funnel data.
2. **Citation density.** Open question 3 from base spec §8 — "should every paragraph cite, or only assertions mapped to requirements" — is now sharper. With KB-grounded outputs, the answer needs to be per-surface: gap-analysis bullets always cite; effort estimates always cite if available; proposal draft only on assertions mapped to requirements (preserve readability). Sally confirm?
3. **Degraded-mode banner copy.** Current draft says "AI analysis temporarily unavailable. Existing analyses are unaffected; new analyses will resume automatically." Should this surface a status-page link? PM recommendation: yes, link to `https://status.eusolicit.com/sirmaai` (does that domain exist?).
4. **Workspace archive cross-substrate timing UX.** Current draft says "you can close this tab — email on complete". Should we also push an in-app notification (bell)? PM recommendation: yes — bell + email is the existing pattern for long-running ops (E07 export pattern).
5. **CRM disconnect — preserve enrichment-attached data.** Current draft preserves it. Open: should we offer "also delete CRM enrichment from past opportunities" as a checkbox in the disconnect modal? Affects GDPR posture for B2B contact data. Recommend deferring to v2; flag in §8 of base spec.

---

## 9. Traceability — Amendment to PRD

| New / Amended FR-NFR | UX section providing coverage |
|---|---|
| FR-45 (per-tenant Project provisioning) | §3 J1 (step 5 dashboard empty state implies provisioning complete); §3 J3 (admin orphan list) |
| FR-46 (Project archival) | §5.15 Cross-substrate workspace archive |
| FR-47 (Opportunity Qualification) | §3 J1 step 8a; §5 update to Opportunity Detail tabs |
| FR-48 (Opportunity Quantification) | §3 J1 steps 8b-d; §5.12 async-run progress |
| FR-49 (KB upload + tier quota) | §5.11 KB Dashboard, Upload zone |
| FR-50 (KB semantic search) | §5.11 Semantic search state |
| FR-51 (Agents grounded in KB + citations) | §6.X Citation chips pattern |
| FR-52 (KB lifecycle) | §5.11 Artefact detail, Replace/Archive/Restore states |
| FR-53 (CRM via SirmaAI MCP) | §4.1 Settings nav; §5.14 CRM Connections |
| FR-54 (Webhook receiver) | (Backend-only — no UX surface; verification UI lives in Admin → SirmaAI Health DLQ view per §4.2) |
| FR-55 (Reconciler) | §5.12 "Stuck" state of async-run progress relies on reconciler as authoritative truth |
| FR-56 (Cross-substrate right-to-erasure) | §3 J3 erasure flow; §5.15 cross-substrate archive |
| NFR-2 amended (async-run TTFB exemption) | §5.12 async-run progress |
| NFR-26 (SirmaAI degraded mode) | §5.13 degraded-mode banner |

**Base spec §9 Traceability table coverage gap resolved.** When this amendment lands, update base spec §9 to extend the table through FR-56 with cross-references to this amendment's section numbers.

---

## 10. Merge plan (when ready)

1. Sally reviews this draft; revises any sections that need design judgement (state-table precision, copy nuance, visual-hierarchy decisions).
2. On approval, sections 1-9 above are merged into `ux-spec.md` at the indicated insertion points:
   - §1 deltas appended to existing §1 of base.
   - §2 persona JTBDs appended to Elena and Operator Ivan sections.
   - §3 journey amendments inserted into J1 and J3 tables.
   - §4 IA deltas update §4.1 and §4.2 nav blocks.
   - §5 new screens (§5.11–§5.15) appended to §5 Key Screens & States.
   - §6 citation chip pattern appended to §6 Interaction Patterns.
   - §7 WCAG deltas added to §7.2 / §7.4.
   - §8 open questions added to base §8 list.
   - §9 traceability rows extended in base §9 table.
3. Base spec frontmatter updated: `inputDocuments` adds the SirmaAI amendment + this UX amendment; `status` flips to "Updated 2026-05-XX — SirmaAI amendment merged".
4. This amendment file moves to `ux-spec-amendment-2026-05-15-sirmaai.md.bak` (preserves history alongside `PRD.v2.0.bak.md` and similar).

---

**Author:** 📋 John (PM) — draft authored 2026-05-15 under bmad-agent-pm IR-remediation pass
**Intended reviewer:** 🎨 Sally (UX designer) — final form requires UX judgement
**Pairs with:** `prd-amendment-2026-05-12-sirmaai.md`, `architecture-amendment-2026-05-12-sirmaai.md`
**Status:** Draft for Sally + Deb review
