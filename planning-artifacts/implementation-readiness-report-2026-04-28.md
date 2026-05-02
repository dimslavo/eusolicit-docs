---
stepsCompleted: ["step-01-document-discovery", "step-02-prd-analysis", "step-03-epic-coverage-validation", "step-04-ux-alignment", "step-05-epic-quality-review", "step-06-final-assessment"]
inputDocuments:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/ux-spec.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/ux-design-specification.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E14-multi-client-workspace.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E15-per-bid-sku-pro-plus-tier.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E16-slack-teams-notifications.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E17-crm-integrations.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/sprint-status.yaml"
focus: "Readiness for next-up Epic 17 (CRM Integrations) kickoff with sprint-state and planning-artifact reconciliation"
date: 2026-04-28
assessor: "PM (autopilot)"
---

# Implementation Readiness Assessment Report

**Date:** 2026-04-28
**Project:** eusolicit
**Mode:** BMAD autopilot (no operator prompts)
**Assessor role:** Product Manager — requirements traceability + planning-hygiene critique
**Focus:** Validate readiness for the next epic the orchestrator will dispatch (Epic 17 — CRM Integrations) and reconcile drift between regenerated planning artifacts (epics.md, 2026-04-28) and the operational sprint-status.yaml.

---

## Executive Verdict

**Overall Readiness: NEEDS WORK — proceed with kickoff caveats**

| Dimension | Verdict | Notes |
|---|---|---|
| PRD coverage of next epic (E17) | ⚠ Split-source | FR11.x lives in `prd-amendment-2026-04-25.md`, not in `PRD.md`. Functionally complete, but base PRD never re-merged. |
| Architecture coverage of E17 | ✅ Strong | ADR-009 + `integrations-api` schema + Fernet token vault + Rule-47 resilience all documented. |
| UX coverage of E17 | ⚠ Thin | `ux-spec.md` covers per-workspace settings shell; CRM-specific connection/conflict-log surfaces not explicitly specified. |
| Epic 17 spec quality | ✅ Adequate to start | 4 stories sized (13/13/13/16 pts), dependencies clear, test patterns inherited from E14/E15 carry-forwards. |
| Planning-artifact hygiene | 🛑 **Critical** | 50+ epic files in `planning-artifacts/epics/` across ≥3 naming conventions; regenerated `epics.md` (10-epic view) does not reference operational E11–E21 work. |
| Sprint-state vs planning-artifact alignment | 🛑 **Critical** | `epics.md` (regenerated today) lists 10 epics; `sprint-status.yaml` tracks 21. The regenerated view erases visibility of in-flight Epic 16/17 in the planning surface. |
| Epic 13 carry-forwards | ⚠ Outstanding | 6 stories at `ready-for-dev` (drift-recovery, dw-01..03, inj-01..03) still un-dispatched; not blocking E17 but accumulating debt. |
| Epic 16 closure integrity | ⚠ Suspect | `epic-16: in-progress` with only `16-0` done **and** `epic-16-retrospective: done` — status combination is internally inconsistent. |

---

## Step 1 — Document Discovery

### Canonical documents (operative source of truth)

| Document | Path | Mtime | Status |
|---|---|---|---|
| PRD (base, v1.x) | `planning-artifacts/PRD.md` | 2026-04-27 | Active, but missing FR8–FR11 amendment scope |
| PRD amendment | `planning-artifacts/prd-amendment-2026-04-25.md` | 2026-04-25 | Defines FR8/FR9/FR10/FR11 — **must be read in conjunction with PRD.md** |
| Architecture | `planning-artifacts/architecture.md` | 2026-04-28 | v2.0, supersedes v1.x; reflects PRD amendment |
| UX spec (current) | `planning-artifacts/ux-spec.md` | 2026-04-28 | Today's regeneration |
| Epics master | `planning-artifacts/epics.md` | 2026-04-28 | Today's regeneration — **10-epic view, does not include E11–E21** |
| Per-epic files (operational) | `planning-artifacts/epics/E01..E21-*.md` | 2026-04-05 → 2026-04-27 | Match sprint-status.yaml structure |
| Per-epic files (regenerated 2026-04-28) | `planning-artifacts/epics/epic-01-..epic-10-*.md` | 2026-04-28 | Match the new `epics.md` 10-epic view |
| Sprint state | `implementation-artifacts/sprint-status.yaml` | 2026-04-27 | 21 epics tracked; orchestrator-managed (per MEMORY.md, surgical edits only) |

### CRITICAL — Document hygiene issues

1. **🛑 Epic file proliferation.** `planning-artifacts/epics/` contains **80+ files** with at least 4 distinct naming conventions:
   - `E01-...md` … `E21-...md` (operational; matches sprint-status)
   - `epic-01-...-2026-04-28` … `epic-10-...-2026-04-28` (regenerated today; 10-epic view)
   - `epic-1-onboarding.md`, `epic-2-intelligence.md`, `epic-3-document-analysis.md`, `epic-4-proposal-generation.md`, `epic-5-compliance.md`, `epic-6-workflows.md` (mid-April drafts)
   - `epic-1-user-registration-subscription.md`, `epic-2-opportunity-intelligence-discovery.md`, … (a third draft series, mid-April)
   - `epic-14.md`, `epic-15.md` … `epic-20.md` (yet another series, 2026-04-26)
   - `epic-01-workspace-access-management.md`, `epic-02-frontend-shell-design-system.md`, … `epic-12-admin-platform.md` (another series, 2026-04-27)
   - `epic-01-core-platform.md` … `epic-10-consortium-discovery.md` (another series, 2026-04-27)
   
   **Risk:** any agent — orchestrator, dev-story, or human — pattern-matching on `epic-*` will load conflicting scopes. Already triggered story-misroute incidents previously implied by sprint-status comments.

2. **🛑 PRD/epics.md backups still present in canonical folder.**
   - `epics.md.bak` (2026-04-22), `epics.md.ignored` (2026-04-26)
   - `prd.md.bak`, `PRD.v2.0.bak.md`
   - `ux-design-directions.html`, `ux-design-specification.md` (older), `ux-spec.md` (newest)
   - These are not in a `_archive/` or `.bak/` folder — they sit alongside live artifacts.

3. **⚠ Two PRD versions in flight.** `PRD.md` is base; `prd-amendment-2026-04-25.md` introduces FR8–FR11. The amendment was never re-merged into PRD.md. Per amendment §1, the PRD is officially "v1.1" but the file on disk does not show that version stamp. Epic E17 cites `PRD v1.1 §6 FR11.1–FR11.3`; that text only exists in the amendment file.

4. **⚠ Two UX specs in flight.** `ux-spec.md` (2026-04-28) and `ux-design-specification.md` (2026-04-22) both present. The regenerated `epics.md` `inputDocuments` frontmatter cites `ux-design-specification.md` (the older), not `ux-spec.md` — drift between regeneration sources.

### Missing documents
None — all four required artifact types (PRD, Architecture, Epics, UX) are present.

---

## Step 2 — PRD Analysis

### Functional Requirements (extracted from `epics.md` Requirements Inventory + amendment cross-reference)

**Base PRD.md FRs (FR-1 … FR-44):** complete. All 44 FRs are mapped to Epic 1–10 in the regenerated `epics.md` "FR Coverage Map" (lines 138–181).

**Amendment FRs (not in base PRD):**
- **FR8 — Multi-Client Workspaces** → Epic 14 (delivered)
- **FR9 — Outcome Telemetry** → Epic 19 (backlog)
- **FR10 — Trust Center & Compliance Posture** → Epic 18 (backlog)
- **FR11 — CRM and Communications Integrations** → Epics 16 (delivered) + 17 (next up)
  - FR11.1: OAuth2 with HubSpot/Salesforce/Pipedrive per workspace → E17 AC bullets 1, 2
  - FR11.2: Bi-directional Deal/Opportunity sync on status change → E17 ACs 3, 4
  - FR11.3: 5-min CRM→EU Solicit read-back → E17 AC 5
  - FR11.4: Slack/Teams webhooks for opportunity/deadline/proposal/compliance triggers → E16 ACs 4, 5 (closed)
  - FR11.5: Tier gating (Slack/Teams=Pro, CRM=Pro+) → E16/E17 tier-gate ACs

### Non-Functional Requirements (NFR-1 … NFR-23)
All 23 NFRs documented in `epics.md` (lines 69–91). E17-relevant NFRs:
- NFR-5/6 (auth, encryption) — Fernet token vault honors NFR-6 application-level encryption requirement
- NFR-7 (data isolation) — workspace-scoped negative test specified in E17 ACs
- NFR-9 (third-party deps) — `inj-01-dependabot-configuration` carry-forward still ready-for-dev (gap)
- NFR-23 (logging/monitoring) — E17 sync_logs + Prometheus metrics in scope per architecture.md line 161

### PRD Completeness Assessment for next epic (E17)

✅ **Functional scope is fully specified** (when amendment is read alongside base PRD).
⚠ **Source-of-truth fragmentation:** New developers/agents will read `PRD.md` and miss FR11 entirely unless they know to load `prd-amendment-2026-04-25.md`. This is a discoverability footgun.

---

## Step 3 — Epic Coverage Validation

### What's in flight per orchestrator (`sprint-status.yaml`)

| Epic | Status | Stories | Notes |
|---|---|---|---|
| Epic 1–12 | done | 100+ stories all done | MVP + initial post-MVP set |
| Epic 13 (Hardening) | done | 7-17 done; **drift-recovery + dw-01..03 + inj-01..03 = 6 stories at ready-for-dev** | 6 carry-forwards un-dispatched |
| Epic 14 (Workspaces) | done | 14-0..14-4 all done | Closed 2026-04-27 |
| Epic 15 (Per-Bid + Pro+) | done | 15-0..15-2 all done | Closed 2026-04-27; retrospective flagged S15.1 review-fix gap |
| Epic 16 (Slack/Teams) | **in-progress (suspect)** | 16-0 done; retrospective also done | Status combination internally inconsistent — see CR-3 |
| **Epic 17 (CRM)** | **backlog (next up)** | 17-0, 17-1, 17-2, 17-3 (all backlog) | Story files not yet in implementation-artifacts/ |
| Epic 18 (Trust Center) | backlog | 18-0, 18-1, 18-2 backlog | |
| Epic 19 (Outcome Telemetry) | backlog | 19-0, 19-1, 19-2 backlog | |
| Epic 20 (NPS/Reviews) | backlog | 20-0 backlog | |
| Epic 21 (Reliability/SLA) | backlog | 21-1..21-6 backlog | k6 baseline (21-1) feeds NFR-1/4/13 closure |

### CRITICAL — Coverage gap between `epics.md` and operational state

The regenerated `epics.md` (today, 2026-04-28) lists exactly 10 epics:
1. Core Platform & User Foundation
2. Opportunity Ingestion & Discovery
3. AI-Powered Tender Analysis
4. AI-Assisted Proposal Drafting
5. Advanced Proposal Collaboration & Workflow
6. Platform Administration & Governance
7. Advanced Compliance & Generation (ESPD)
8. Enterprise & API Features
9. Post-Award Management & Reporting
10. Consortium & Partner Discovery

**Missing from `epics.md` entirely:**
- Multi-Client Workspaces (operational E14, FR8) — partially mapped to "Epic 8: Enterprise & API Features" line 145, but the workspace switcher / external-collaborator scope is not reflected
- Per-Bid SKU + Pro+ tier (operational E15) — not mapped
- Slack/Teams notifications (operational E16, FR11.4–5) — not mapped
- CRM integrations (operational E17, FR11.1–3) — **NOT MAPPED → next epic to start has no presence in the regenerated planning view**
- Trust Center (operational E18, FR10) — not mapped
- Outcome Telemetry (operational E19, FR9) — not mapped
- NPS / Reviews / Onboarding (operational E20) — not mapped
- Platform Reliability / 99.9% SLA (operational E21, NFR-14 uplift) — not mapped
- Hardening / Drift Recovery (operational E13) — not mapped

**Implication:** The regenerated `epics.md` is a clean PRD-FR-aligned view, but it does **not** reflect the in-flight roadmap. If the orchestrator later loads `epics.md` rather than the per-epic files, it will lose visibility of E11–E21 entirely.

### FR Coverage for next epic (E17 — CRM)

| AC reference | PRD/Amendment source | Coverage |
|---|---|---|
| `client.crm_connections` table + Fernet vault | Amendment FR11.1; arch §5 line 499 | ✅ |
| OAuth2 flow per provider | Amendment FR11.1; arch §5 ADR-009 | ✅ |
| Auto-create CRM Deal on opportunity add | Amendment FR11.2 | ✅ |
| Bi-directional sync ≤60s outbound, ≤5min inbound | Amendment FR11.2/FR11.3 | ✅ |
| Conflict resolution LWW + audit log | Amendment §122 (decision item 4) | ✅ |
| Pro+ tier gate | Amendment FR11.5 | ✅ |
| Two-layer resilience (Rule 47) | Architecture line 696 | ✅ |
| Per-provider rate limits | Architecture line 1036 risk #10 | ✅ |
| Token rotation Beat task | Architecture pattern from Epic 9 | ✅ |
| Workspace-scoped negative test | E14 carry-forward pattern | ✅ |
| sync_logs 30-day retention | Architecture line 266 | ✅ |
| CRM dashboard widget | E17 AC bullet 13 | ⚠ UX spec does not show this surface explicitly |

---

## Step 4 — UX Alignment

### What `ux-spec.md` (2026-04-28) covers
- App shell, navigation, design tokens, accessibility (WCAG 2.1 AA)
- Workspace switcher (E14 surface) — covered
- Per-workspace settings page (where E16 integrations panel lives) — covered
- Notification preferences — covered

### Gaps for next epic (E17)
1. **CRM connection cards** — no spec for the per-provider connection card layout (status badge, last-sync timestamp, "Reconnect" / "Disconnect" affordances).
2. **Conflict log viewer** — `integrations.conflict_log` is in the data model and AC bullet 6 of E17, but no UX surface specified for how a user reviews/acks conflicts.
3. **Stage-mapping configuration** — E17 S17.01 mentions "Stage mapping configurable per workspace via Admin API; default mapping seeded" but no admin UI is specified for editing the mapping table.
4. **Rate-limit "paused for N minutes" workspace UI state** — E17 S17.00 lists this as an outcome of the circuit-breaker, but no UX spec exists for how that state is surfaced (toast? settings banner? dashboard widget?).
5. **OAuth callback success/failure pages** — generic OAuth handling is established in Epic 9 (calendar) but provider-specific copy not specified.

### UX-spec source drift (warning)
`epics.md` frontmatter cites `ux-design-specification.md` (older), but `ux-spec.md` (newer) exists. The two files differ in scope. The orchestrator/PM should pick one as canonical and archive the other.

---

## Step 5 — Epic Quality Review (focus: E17)

### E17 spec quality scorecard

| Dimension | Score | Evidence |
|---|---|---|
| Goal clarity | A | "Bi-directional CRM sync … Without CRM, EU Solicit becomes shelfware" — concrete deal-blocker framing |
| Acceptance criteria specificity | A− | 13 testable ACs with vendor-specific rate-limits, retention windows, latency targets |
| Story decomposition | B+ | 4 stories (S17.00 scaffold, S17.01 HubSpot, S17.02 Pipedrive, S17.03 Salesforce); story sizes 13/13/13/16 = 55 pts (matches header) |
| Dependency declaration | A | "Dependencies: E14, E15, E16" stated; intra-epic order via locked architect decision §11.6 |
| Test design carry-forward | B | Patterns inherited from E14/E15 retros (parametrised cross-tenant, canonical ORM seeding, no production-side test routes). No `test-design-epic-17.md` yet — story files will need to fill the gap (mirror the S15.1 pattern) |
| Source-of-truth citations | B+ | Cites "PRD v1.1 §6 FR11.1–11.3", "architecture-evaluation §2 Change-3", "§11 locked decision §11.6" — all verifiable |
| Net-new fence (BMM Rule) | A | Each story explicitly scoped (HubSpot first, Salesforce daily-quota-aware) — no scope creep across stories |
| Anti-pattern guard rails | A | Story files will need to inherit: canonical ORM seeding (E14.2 BLOCKING #3), no test-routes-in-prod (S15.0 B1), parametrised cross-tenant symmetry (S15.0 B3), HMAC `compare_digest` (project-context Rule 48), state-nonce validation (Rule 39) |

### E17 risks flagged for the dev pass
- **R-001:** `S17.00` introduces both a token-vault data model AND a generic adapter framework AND OAuth flow AND Celery worker AND conflict resolution = 13-point story with high blast radius. Recommend pre-flight architecture sign-off before story file creation.
- **R-002:** Salesforce daily-quota state machine (`S17.03`) is the most complex new pattern in the platform's history. The 16-pt sizing assumes an existing `simple-salesforce` SDK pattern. Validate during `bmad-create-story`.
- **R-003:** `integrations.conflict_log` is referenced from both `S17.00` (table created) and `epic-15` retrospective notes — confirm migration ownership.
- **R-004:** "Per-org rate-limit (different from per-user limits)" in `S17.03` requires Salesforce sandbox account procurement. Procurement lead time may slip dev kickoff.

### Epic 16 closure inconsistency (CR-3, blocker for clean E17 kickoff record)

`sprint-status.yaml` shows:
```
epic-16: in-progress
16-0-...: done
epic-16-retrospective: done
```

Either:
- Epic 16 has only one story by design (then `epic-16` should be `done`), OR
- Epic 16 has additional stories that were never created (then retrospective is premature).

Operator action required: confirm Epic 16 scope-of-completion and either flip `epic-16: done` or open the missing stories.

---

## Step 6 — Final Assessment

### Summary

The **next epic (E17 — CRM Integrations) is fundamentally ready to kick off**. PRD coverage exists (in the amendment), architecture coverage is strong (ADR-009 + schema + resilience patterns), and the epic spec itself is high quality with verifiable source citations.

However, **planning-artifact hygiene has degraded sharply** — the same folder now contains 80+ epic files in 6+ naming conventions, two PRD files (base + amendment), two UX specs, and a regenerated `epics.md` that erases visibility of all in-flight post-MVP work (Epics 11–21). This is a high-probability source of agent misroute incidents.

### Critical Issues Requiring Immediate Action

1. **CR-1 (planning-artifact hygiene) — 🛑 CRITICAL**
   - Move all `epic-*.md` files in `planning-artifacts/epics/` that do **not** match the operational `E01-...md` … `E21-...md` convention into `planning-artifacts/epics/_archive/`.
   - Specifically archive: `epic-1-*.md`, `epic-2-*.md`, …, `epic-6-*.md`, `epic-14.md`–`epic-20.md`, `epic-01-...-2026-04-27.md` series, and `epic-01-...-2026-04-28.md` … `epic-10-...-2026-04-28.md` series.
   - Move `epics.md.bak`, `epics.md.ignored`, `prd.md.bak`, `PRD.v2.0.bak.md`, `ux-design-directions.html`, `ux-design-specification.md` to `planning-artifacts/_archive/`.

2. **CR-2 (planning view drift) — 🛑 CRITICAL**
   - The regenerated `epics.md` (today) reflects only the original 10-epic PRD-FR view and erases Epics 11–21. Decision required:
     - **Option A:** Treat `epics.md` as historical-MVP reference; rename to `epics-mvp-baseline.md`. Establish a new `epics.md` (or `roadmap.md`) that includes E11–E21 as the authoritative full-roadmap view.
     - **Option B:** Regenerate `epics.md` to merge the 10-epic PRD view with the post-MVP E11–E21 set (preferred — single source of truth).
   - Until resolved, instruct all BMAD agents to use `planning-artifacts/epics/E*.md` per-epic files as authoritative, not the rolled-up `epics.md`.

3. **CR-3 (Epic 16 status integrity) — ⚠ HIGH**
   - Reconcile `epic-16: in-progress` with `epic-16-retrospective: done`. If Epic 16 is single-story (S16.00 only per E16 spec), update status to `done`. If additional stories were planned, surface them and open story files.

4. **CR-4 (PRD source unification) — ⚠ HIGH**
   - Merge `prd-amendment-2026-04-25.md` into `PRD.md` and stamp `PRD.md` as v1.1 (or v2.0 to match architecture). Until merged, every BMAD agent that loads `PRD.md` will be working from an incomplete spec — they will not see FR8/FR9/FR10/FR11 unless they know to also load the amendment.

5. **CR-5 (UX gap for E17) — ⚠ MEDIUM**
   - Before `bmad-create-story` for `17-0`, add the following surfaces to `ux-spec.md` (or a small E17-specific UX supplement):
     - CRM connection card (per provider)
     - Conflict log viewer
     - Stage-mapping admin UI
     - Rate-limit "paused" workspace banner
     - Provider-specific OAuth callback copy

6. **CR-6 (Epic 13 carry-forwards) — ⚠ MEDIUM (background)**
   - 6 stories at `ready-for-dev`: drift-recovery, dw-01..03, inj-01..03. Not blocking E17 but accumulating debt across two epic boundaries (E13 → E14 → E15 → E16 → E17). Recommend a side-channel orchestrator dispatch in parallel with E17 work.

### Recommended Next Steps (in order)

1. **Resolve CR-1 + CR-2 + CR-4 first** (planning hygiene). 30–60 min of file moves + a single `epics.md` regeneration that includes E11–E21. This unblocks all subsequent BMAD agents from misroute risk.
2. **Resolve CR-3** (flip `epic-16: done` if S16.00 was the only planned story).
3. **Resolve CR-5** (UX supplement for E17 surfaces) — should be a 1–2 hour `bmad-ux-designer` pass.
4. **Begin E17 kickoff** with `bmad-create-story` for `17-0` (CRM Connection Model + Fernet Token Vault + Sync Engine Scaffold + Conflict-Resolution Framework). Per operator workflow guidance: this MUST be followed by `[VS] Validate Story` (non-negotiable) before dev.
5. **In parallel:** dispatch one or more of the 6 Epic-13 carry-forwards (`inj-01` Dependabot is a 1-pt quick-win; `inj-02` k6 baseline is also called for by E21 prerequisites).

### Final Note

This assessment identified **6 issues across 3 categories** (planning-artifact hygiene = 2, status integrity = 1, spec coverage = 3).

E17 (CRM Integrations) is **specification-ready**: PRD scope (via amendment) + architecture + epic AC quality are all sufficient to begin story creation. The blockers are organisational/hygienic, not technical.

**Recommendation:** **Proceed to `bmad-create-story` for 17-0 only after CR-1, CR-2, CR-3, CR-4 are resolved.** CR-5 (UX gap) can be resolved during `[VS] Validate Story` rather than blocking kickoff. CR-6 is background.

---

## HALT determination

**No HALT.** None of the issues identified are blocking in the sense that E17 specification is broken or incoherent. The critical issues are file-organisation problems that must be cleaned up to prevent agent misroute, but the underlying specs (architecture.md, prd-amendment-2026-04-25.md, E17-crm-integrations.md) are coherent and aligned.

**Operator decision point:** Resolve CR-1 through CR-4 in a 30–60 min hygiene pass before invoking `bmad-create-story` for 17-0, OR accept the misroute risk and proceed.

---

*Generated by `bmad-check-implementation-readiness` skill in BMAD autopilot mode (no operator prompts) on 2026-04-28.*
