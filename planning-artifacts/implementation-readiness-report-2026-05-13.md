---
stepsCompleted:
  - step-01-document-discovery
  - step-02-prd-analysis
  - step-03-epic-coverage-validation
  - step-04-ux-alignment
  - step-05-epic-quality-review
  - step-06-final-assessment
filesUnderReview:
  prd:
    - PRD.md
    - prd-amendment-2026-04-25.md
    - prd-amendment-2026-05-12-sirmaai.md
  architecture:
    - architecture.md
    - architecture-amendment-2026-05-12-sirmaai.md
  epics:
    - epics/E01-infrastructure-foundation.md
    - epics/E02-authentication-identity.md
    - epics/E03-frontend-shell-design-system.md
    - epics/E04-ai-gateway-service.md
    - epics/E05-data-pipeline-ingestion.md
    - epics/E06-opportunity-discovery.md
    - epics/E07-proposal-generation.md
    - epics/E08-subscription-billing.md
    - epics/E09-notifications-alerts-calendar.md
    - epics/E10-collaboration-tasks-approvals.md
    - epics/E11-grants-compliance.md
    - epics/E12-analytics-admin-platform.md
    - epics/E13-hardening-drift-recovery.md
    - epics/E14-multi-client-workspace.md
    - epics/E15-per-bid-sku-pro-plus-tier.md
    - epics/E16-slack-teams-notifications.md
    - epics/E17-crm-integrations.md
    - epics/E18-trust-center-compliance.md
    - epics/E19-outcome-telemetry-renewal-proof.md
    - epics/E20-nps-reviews-onboarding.md
    - epics/E21-platform-reliability-99-9-sla.md
    - epics/E22-onprem-launch.md
    - epics/E23-operational-close-out.md
    - epics/E24-sirmaai-tenant-provisioning.md
    - epics/E25-knowledge-base-lifecycle.md
    - epics/E26-agent-driven-ingestion.md
    - epics/E27-crm-via-mcp.md
    - epics/E28-webhook-reconciliation.md
  ux:
    - ux-spec.md
  context:
    - project-context.md
    - onprem-pivot-decision-2026-05-11.md
    - sprint-change-proposal-2026-05-12-sirmaai.md
filesIgnored:
  - prd.md.bak
  - PRD.v2.0.bak.md
  - epics.md.bak
  - epics.md.ignored
  - 68 stale epic-NN-*.md files in epics/
  - ux-design-specification.md
  - ux-design-directions.html
  - architecture-evaluation-2026-04-25.md
---

# Implementation Readiness Assessment Report

**Date:** 2026-05-13
**Project:** eusolicit

## Document Inventory (Step 1)

User confirmed 2026-05-13: 28 canonical `E##-*.md` epics are authoritative; `ux-spec.md` is the UX source of truth; PRD and Architecture each read as composite (base + amendment(s)).

**Critical issue surfaced:** 68 stale `epic-NN-*.md` files coexist with the 28 canonical `E##-*.md` files in `planning-artifacts/epics/`. Stale files are explicitly excluded from this assessment. Recommend physical cleanup (move to `epics/_archive/`) before next BMAD planning pass to prevent future regenerations from accidentally consuming them.

## PRD Analysis (Step 2)

**Composite read:** `PRD.md` (v1, 2026-04-27) + `prd-amendment-2026-04-25.md` (consulting-firm pivot) + `prd-amendment-2026-05-12-sirmaai.md` (SirmaAI substrate pivot).

### Functional Requirements

#### Base PRD (FR-1 → FR-44)

**User and Tenant Management**
- FR-1: New user registers via email + password
- FR-2: New user registers via Google OAuth
- FR-3: User logs in/out
- FR-4: Email verification required before core features
- FR-5: Admin manages company profile
- FR-6: Admin invites users to company workspace
- FR-7: Admin assigns/modifies workspace roles (admin, bid_manager, contributor, reviewer, read_only)
- FR-8: Enterprise admin manages multiple isolated client workspaces under one parent
- FR-9: User belongs to multiple workspaces and switches between them

**Billing & Subscription**
- FR-10: User subscribes to Starter/Professional/Enterprise via Stripe
- FR-11: Feature-gating by subscription tier and usage limits
- FR-12: 14-day free trial of Professional tier
- FR-13: Self-service subscription mgmt via Stripe customer portal
- FR-14: One-time add-on purchases

**Data Pipeline & Opportunity Discovery**
- FR-15: Auto-ingest opportunities from AOP/TED *(superseded by SirmaAI amendment FR-15-new)*
- FR-16: Search opportunities (full-text + faceted filters)
- FR-17: AI relevance score per opportunity
- FR-18: List opportunities sorted by relevance/deadline
- FR-19: View detailed opportunity
- FR-20: Save/star opportunities

**AI-Powered Analysis**
- FR-21: AI executive summary per tender doc
- FR-22: Extract structured requirements checklist
- FR-23: Flag high-risk clauses
- FR-24: Upload tender docs (PDF, DOCX)
- FR-25: Score simulator

**Proposal Generation & Collaboration**
- FR-26: Create proposal tied to opportunity
- FR-27: AI first-draft generation
- FR-28: Rich text editor
- FR-29: Version history
- FR-30: Restore previous version
- FR-31: Section lock (Post-MVP)
- FR-32: Comments + resolution (Post-MVP)
- FR-33: Export to PDF/DOCX

**Compliance & Workflow**
- FR-34: Validate proposal against compliance checklist
- FR-35: Admin manages regulatory frameworks (e.g., ZOP)
- FR-36: Generate pre-filled ESPD *(amended in SirmaAI amendment — now agent-driven)*
- FR-37: Task management on proposals (Post-MVP)
- FR-38: Task dependencies (Post-MVP)
- FR-39: Multi-stage approval workflow (Post-MVP)

**Notifications & Administration**
- FR-40: Email digests for relevant opportunities
- FR-41: Calendar sync (Google/Outlook) (Post-MVP)
- FR-42: Admin portal (tenant mgmt, subs, settings)
- FR-43: Immutable audit trail
- FR-44: REST API access for Enterprise

#### 2026-04-25 Amendment (sub-FRs + FR8/9/10/11 families)

**Per-Bid Pricing + Pro+ Tier**
- FR2.4: Per-Bid SKU (one-time payment or metered add-on, €99/€199/€299 by tier)
- FR2.5: Per-bid metering bypass (per-opportunity tier-cap bypass)
- FR2.6: Admin-configurable per-bid pricing
- FR2.7: In-product NPS prompt (Pro+ tier, quarterly) routing to G2/Capterra/Gartner

**FR8 — Multi-Client Workspace Management**
- FR8.1: Tenant creates/renames/archives/deletes client workspaces
- FR8.2: Workspace scopes opportunities, proposals, docs, content, usage, audit, frameworks, telemetry
- FR8.3: Per-workspace RBAC roles
- FR8.4: Cross-workspace search + reuse + analytics for tenant admin
- FR8.5: External-collaborator role (read-only/comment-only, no Stripe seat)
- FR8.6: Per-workspace metering rolling up to tenant subscription

**FR9 — Outcome Telemetry & Renewal Proof**
- FR9.1: Platform-attributed tag per opportunity
- FR9.2: Bid outcome capture (won/lost/withdrawn + evaluator score + contract value + effort)
- FR9.3: Workspace Outcome Dashboard (bids tracked, win rate, attribution, content reuse, AI usage, hours saved)
- FR9.4: Monthly auto-generated Outcome Brief (PDF)
- FR9.5: Outcome API for Pro+ (embed in QBR decks)
- FR9.6: Onboarding milestone tracking ("First Bid in 14 Days") + CSM stall alerts

**FR10 — Trust Center & Compliance Posture**
- FR10.1: Public `/trust` page (GDPR, ISO 27001 progress, data residency, encryption)
- FR10.2: Downloadable artefacts (DPA, Security Overview, Sub-Processor List, pen-test summary, BCP, Data Residency)
- FR10.3: Sub-Processor List auto-rendered from `infra/sub-processors.yaml`
- FR10.4: Versioned artefacts + GDPR Art. 28 customer notification on sub-processor changes

**FR11 — CRM & Communications Integrations**
- FR11.1: OAuth2 to HubSpot/Salesforce/Pipedrive (per workspace) *(scope superseded — see SirmaAI FR-53)*
- FR11.2: Opportunity → CRM Deal/Opportunity sync with lifecycle status
- FR11.3: CRM read-back sync ≤5 min
- FR11.4: Slack/Teams webhook integrations
- FR11.5: Tier-gated (Slack/Teams = Professional; CRM = Pro+)

#### 2026-05-12 SirmaAI Amendment

**SirmaAI Tenant Lifecycle**
- FR-45: Auto-provision SirmaAI Project per company create (KB seed + MCP server registrations + retry + nightly reconciliation)
- FR-46: Soft-delete SirmaAI Project on company archive (24h revocation, no auto-restore)

**Replaces FR-15 (whole-line)**
- FR-15-new: N8N workflow templates invoke SirmaAI crawler/normalisation/scoring agents under tenant Project scope; normalised records via Standard Webhooks to `pipeline.opportunities`; `pipeline.crawler_runs` audit retained

**SirmaAI Analysis (new)**
- FR-47: Opportunity Qualification Analysis (fit, gap, recommended action, confidence)
- FR-48: Opportunity Quantification Analysis (effort, p(win), expected value, bid/no-bid threshold)

**Knowledge Base (new subsection)**
- FR-49: Upload KB artefacts (PDF/DOCX/TXT/MD) with tier quotas (50MB/500MB/5GB/unlimited)
- FR-50: Semantic search over tenant KB with ranked passages + citations
- FR-51: All agents read tenant KB; structured citations in outputs
- FR-52: KB lifecycle (replace/archive/delete) with ≤5 min re-index latency; profile updates trigger re-index

**CRM via MCP + Webhooks**
- FR-53: CRM connectivity for **Dynamics 365 + HubSpot** as SirmaAI MCP servers per tenant; OAuth held in EU Solicit, tokens injected at registration; Pipedrive/Salesforce deferred post-launch
- FR-54: Standard Webhooks receiver (HMAC + idempotency + Redis Streams routing + DLQ)
- FR-55: Run-state reconciler polling `GET /jobs/{jobId}/status` every 5 min (authoritative truth)

**Replaces FR-36**
- FR-36-new: ESPD pre-fill via SirmaAI **ESPD Auto-Fill agent** in tenant Project; EU Solicit renders XML + PDF

**Total FR count:** 44 base + 25 sub-FRs and amendments + 11 SirmaAI net-new = **~80 FR slots** (some are amendments/replacements, not net-new).

### Non-Functional Requirements

#### Base PRD (NFR-1 → NFR-23)

- NFR-1 (Performance): API p95 < 200ms
- NFR-2 (Performance): AI streaming TTFB < 500ms *(amended in SirmaAI amendment)*
- NFR-3 (Performance): Lighthouse ≥ 90
- NFR-4 (Concurrency): 100 concurrent users at MVP
- NFR-5 (Security): JWT auth, short-lived access tokens, secure refresh
- NFR-6 (Security): TLS 1.3 + AES-256 + app-level encryption for sensitive data *(amended in SirmaAI amendment)*
- NFR-7 (Security): Strict tenant data isolation
- NFR-8 (Security): Admin portal IP allowlist + MFA
- NFR-9 (Security): No critical/high vuln in deps
- NFR-10 (Scalability): Vertical scale
- NFR-11 (Scalability): Stateless services for horizontal scale
- NFR-12 (Scalability): DB read replicas
- NFR-13 (Scalability): 10k companies / 1M opps in 2y with <20% degradation
- NFR-14 (Reliability): ≥99.5% uptime monthly *(tightened to 99.9% in 2026-04-25 amendment)*
- NFR-15 (Reliability): Zero data loss, daily backups, 24h PITR
- NFR-16 (Reliability): Idempotent financial ops
- NFR-17 (Reliability): DR plan, 4h RTO
- NFR-18 (Accessibility): WCAG 2.1 AA
- NFR-19 (Accessibility): Full keyboard navigation
- NFR-20 (Accessibility): Screen reader support
- NFR-21 (Maintainability): ruff + mypy + eslint + tsc clean on main
- NFR-22 (Maintainability): 80% unit + 90% E2E coverage on critical journeys
- NFR-23 (Maintainability): Structured logs + Prometheus/Grafana metrics

#### 2026-04-25 Amendment
- NFR-14-new (Reliability): **99.9% uptime SLA** (rolling 30 days), service-credit policy, ~43min downtime/month max
- Compliance line: **ISO/IEC 27001:2022 audit by Month 6, certification by Month 12**

#### 2026-05-12 SirmaAI Amendment
- NFR-2-new (Performance): TTFB <500ms p95 applies only to SirmaAI **streaming sync** runs; async runs governed by SirmaAI job-engine latency
- NFR-6-new (Security): Fernet-encrypted credentials extend to **SirmaAI Project API keys, CRM OAuth tokens, SirmaAI webhook HMAC secrets, SirmaAI system API key**
- NFR-24 (Security): SirmaAI Project API-key rotation, default 90 days, zero-downtime double-validation overlap
- NFR-25 (Performance): EU Solicit tier ↔ SirmaAI Org rate-limit sync within 60s of tier change (Free=100/d, Starter=1k/d, Pro=10k/d, Ent=100k/d)
- NFR-26 (Reliability): SirmaAI = critical external dependency, no Multi-AZ failover; `circuit_breaker(retry(http_factory))`; tenant-visible status banner on >5min outage; FR-55 reconciler guarantees no in-flight loss

### Additional Requirements & Constraints

**Domain (GovTech) — base PRD + SirmaAI amendment**
- Public Procurement Law compliance (ZOP + EU directives)
- GDPR (EU residency, right to erasure, processing records)
- **Data residency (amended):** EU only; EU Solicit on `www1.endigitalx.com` per **ADR-010 on-prem pivot (2026-05-11)**; **SirmaAI EU-only residency = launch-blocking due-diligence item**
- Admin VPN/IP allowlist; immutable audit trail; AES-256 + TLS 1.3
- ≥99.5% (→ 99.9%) availability
- Strict tenant isolation
- Government portals: AOP/TED via SirmaAI crawler agents in N8N (post-amendment); EU Solicit holds canonical record
- Standardized formats: XML ESPD

**SaaS B2B Tenancy (amended in SirmaAI amendment)**
- Two cooperating substrates: EU Solicit Postgres (canonical) + SirmaAI (per-Company → Project, singleton Org)
- N8N shared at Org level, parameterized by projectId
- Per-Project API key = tenant-boundary credential

**Subscription Tiers**
- Free / Starter / Professional / **Pro+** (new from 2026-04-25) / Enterprise
- Per-Bid SKU (one-time or metered add-on)
- 14-day Professional free trial
- EU VAT via Stripe Tax

### PRD Completeness Assessment (initial)

| Area | Verdict | Notes |
|---|---|---|
| FRs | ✅ Comprehensive | 80+ FR slots; all user-visible behaviours covered |
| NFRs | ✅ Strong | 26 NFRs; tightened SLA + key-rotation + dependency posture |
| Personas | ✅ Updated | Consulting firms primary ICP per 2026-04-25 amendment |
| Innovation framing | ✅ Documented | Agentic AI + multi-tenant + dual-segment |
| Tenancy model | ⚠️ Two-layer composite | EU Solicit workspace + SirmaAI Project; readiness check will need to verify both layers are decomposed in epics |
| Data residency | ⚠️ Launch-blocking item flagged | SirmaAI EU-only residency confirmation outstanding |
| Open questions | ⚠️ 6 items unresolved | From 2026-04-25 amendment §Open Questions Requiring Deb's Decision — Pro+ pricing, per-bid pricing, Trust Center deadline, ISO budget, CRM prioritisation, existing-customer impact. Some may now be moot post-SirmaAI pivot (e.g., CRM prioritisation overridden to Dynamics+HubSpot). To be verified in Step 3. |

**Carry-forward concerns for Step 3 (Epic Coverage Validation):**
1. Confirm FR-15 amendment is reflected in E05 + E26 (not just E05).
2. Confirm FR-36 amendment is reflected in E11 modification (not double-counted in original E11 + new agent epic).
3. Confirm FR-53 supersedes FR11.1 (CRM scope changed from HubSpot/Salesforce/Pipedrive → Dynamics+HubSpot; Pipedrive/Salesforce deferred) — verify E17 and E27 reflect the swap.
4. Confirm new FRs 45–55 trace to E24–E28 (per traceability matrix in amendment).
5. Confirm ADR-010 (on-prem) and ADR-018/019/020 (SirmaAI substrate) are reflected in architecture and epics.

## Epic Coverage Validation (Step 3)

**Source of truth:** 28 canonical `E##-*.md` files. The flat `epics.md` is **stale** (last touched 2026-05-05, predates SirmaAI pivot, missing E14–E28, FR Coverage Map placeholder unfilled) and is excluded.

**Traceability quality:** Only **9 of 28** epic files carry explicit `FR-NN` references in their text (E04, E05, E11, E14–E21, E24–E28). The other 19 cover FRs **implicitly via goal/story content** — a real traceability gap that should be closed by adding "FRs covered" lines to each epic header.

### Coverage Matrix

| FR/NFR | PRD source | Epic | Status |
|---|---|---|---|
| FR-1 (email/pw register) | base | E02 | ✅ Covered (S02.02) |
| FR-2 (Google OAuth) | base | E02 | ✅ Covered (S02.06) |
| FR-3 (login/logout) | base | E02 | ✅ Covered (S02.03, S02.05) |
| FR-4 (email verification) | base | E02 | ✅ Covered (implicit) |
| FR-5 (company profile) | base | E02 | ✅ Covered |
| FR-6 (invite users) | base | E02 | ✅ Covered |
| FR-7 (assign roles) | base | E02 | ✅ Covered |
| FR-8 (multi-workspace) | base | **E14** | ✅ Covered |
| FR-9 (user spans workspaces) | base | E14 | ⚠️ **Implicit** — verify cross-workspace user assignment is in E14 stories |
| FR-10 (Stripe subscribe) | base | E08 | ✅ Covered (S08.06) |
| FR-11 (tier feature gating) | base | E08 | ✅ Covered (S08.03 + TierGate) |
| FR-12 (14-day trial) | base | E08 | ✅ Covered (S08.03) |
| FR-13 (Stripe customer portal) | base | E08 | ✅ Covered (S08.07) |
| FR-14 (one-time add-ons) | base | E08 + **E15** | ✅ Covered (S08.09 + E15 per-bid SKU) |
| FR-15 (auto-ingest AOP/TED) | base + SirmaAI-replace | **E05 + E26** | ⚠️ Dual coverage — E05 has legacy Celery FR-15, E26 owns the amended SirmaAI/N8N FR-15. Verify E05 was actually amended to defer to E26. |
| FR-16 (search/filter) | base | E06 | ✅ Covered |
| FR-17 (relevance score) | base | E06 (via E05) | ⚠️ Drift risk — relevance scoring now in SirmaAI quantification (FR-48/E26); E06 still presents the score. Verify ownership. |
| FR-18 (sorted list) | base | E06 | ✅ Covered |
| FR-19 (detail view) | base | E06 | ✅ Covered |
| FR-20 (save/star) | base | E06 | ✅ Covered |
| FR-21 (exec summary) | base | E07 | ✅ Covered |
| FR-22 (req checklist) | base | E07 | ✅ Covered |
| FR-23 (risk flagging) | base | E07 | ✅ Covered |
| FR-24 (upload docs) | base | E07 | ✅ Covered (E07 Story 4.1) |
| FR-25 (score simulator) | base | E07 | ✅ Covered |
| FR-26 (create proposal) | base | E07 | ✅ Covered |
| FR-27 (AI first draft) | base | E07 | ✅ Covered |
| FR-28 (rich text editor) | base | E07 | ✅ Covered (Tiptap) |
| FR-29 (version history) | base | E07 | ✅ Covered |
| FR-30 (restore version) | base | E07 | ✅ Covered |
| FR-31 (section lock) | base | **E10** | ✅ Covered |
| FR-32 (comments) | base | E10 | ✅ Covered |
| FR-33 (PDF/DOCX export) | base | E07 | ✅ Covered |
| FR-34 (compliance validation) | base | E07 + **E11** | ✅ Covered (Compliance Checker reused) |
| FR-35 (admin frameworks) | base | E11 | ✅ Covered |
| FR-36 (ESPD pre-fill) | base + SirmaAI-replace | **E11 (amended)** | ⚠️ Verify — E11 file references FR-36; need to confirm content was amended to agent-driven (per SirmaAI amendment), not legacy profile-mapping |
| FR-37 (task mgmt) | base | E10 | ✅ Covered |
| FR-38 (task deps) | base | E10 | ✅ Covered |
| FR-39 (approval workflow) | base | E10 | ✅ Covered |
| FR-40 (email digests) | base | E09 | ✅ Covered |
| FR-41 (calendar sync) | base | E09 | ✅ Covered |
| FR-42 (admin portal) | base | **E12** | ✅ Covered |
| FR-43 (audit trail) | base | E02 + E12 + E14 | ⚠️ Verify E14 added `workspace_id` to every audit_log row per amendment |
| FR-44 (Enterprise API) | base | E12 | ✅ Covered |
| **FR2.4 (per-bid SKU)** | 2026-04-25 | E15 | ✅ Covered |
| **FR2.5 (metering bypass)** | 2026-04-25 | E15 | ✅ Covered |
| **FR2.6 (admin pricing config)** | 2026-04-25 | E15 | ✅ Covered |
| **FR2.7 (NPS prompt)** | 2026-04-25 | **E20** | ✅ Covered |
| **FR8.1–FR8.6 (workspace mgmt)** | 2026-04-25 | **E14** | ✅ Covered |
| **FR9.1–FR9.5 (outcome telemetry)** | 2026-04-25 | **E19** | ✅ Covered |
| **FR9.6 (onboarding milestones)** | 2026-04-25 | E19 + E20 | ✅ Covered (E20 explicitly states absorbed into E19) |
| **FR10.1–FR10.4 (Trust Center)** | 2026-04-25 | **E18** | ✅ Covered |
| **FR11.1–FR11.5 (CRM + Slack/Teams)** | 2026-04-25 | **E16 + E17** | ⚠️ Partially superseded — see FR-53 row below; E17 file explicitly notes "E27 is the forward-looking authoritative spec" |
| **FR-45 (tenant provisioning)** | 2026-05-12 SirmaAI | **E24** | ✅ Covered |
| **FR-46 (tenant archival)** | 2026-05-12 SirmaAI | E24 | ✅ Covered |
| **FR-47 (qualification)** | 2026-05-12 SirmaAI | **E26** | ✅ Covered |
| **FR-48 (quantification)** | 2026-05-12 SirmaAI | E26 | ✅ Covered |
| **FR-49 (KB upload)** | 2026-05-12 SirmaAI | **E25** | ✅ Covered |
| **FR-50 (KB semantic search)** | 2026-05-12 SirmaAI | E25 | ⚠️ Implicit — E25 goal mentions semantic search; verify story exists |
| **FR-51 (agent KB access)** | 2026-05-12 SirmaAI | E25 + E26 | ✅ Covered |
| **FR-52 (KB lifecycle)** | 2026-05-12 SirmaAI | E25 | ✅ Covered |
| **FR-53 (CRM via MCP)** | 2026-05-12 SirmaAI | **E27** | ✅ Covered (E27 supersedes E17 for Dynamics+HubSpot scope) |
| **FR-54 (webhook receiver)** | 2026-05-12 SirmaAI | **E28** | ✅ Covered |
| **FR-55 (run-state reconciler)** | 2026-05-12 SirmaAI | E28 | ✅ Covered |
| NFR-1 (API p95 <200ms) | base | E21 (SLO observability) | ✅ Covered |
| NFR-2 (TTFB <500ms) | base + amended | **E04 + E04-amended** | ⚠️ Verify E04 was amended for SirmaAI streaming sync semantic (per SirmaAI NFR-2) |
| NFR-3 (Lighthouse ≥90) | base | E03 (frontend shell) | ✅ Covered (implicit) |
| NFR-4 (100 concurrent) | base | E21 | ✅ Covered |
| NFR-5 (JWT) | base | E02 | ✅ Covered |
| NFR-6 (TLS + AES-256 + Fernet) | base + amended | E02 + E17 + E24 + E27 | ✅ Covered |
| NFR-7 (tenant isolation) | base | E01 (schema) + E14 (workspace) | ✅ Covered |
| NFR-8 (admin IP+MFA) | base | E12 | ✅ Covered |
| NFR-9 (dependency vuln scan) | base | E13 (Dependabot carry-forward) | ⚠️ **Confirm Dependabot landed** — E13 retro flagged it as still pending in 2026-04-25 retrospective |
| NFR-10 (vertical scale) | base | E21 | ✅ Covered |
| NFR-11 (horizontal scale) | base | E01 (stateless services) | ✅ Covered (implicit) |
| NFR-12 (read replicas) | base | E21 | ⚠️ **Conflict** — read replicas implied AWS RDS; on-prem pivot (E22) makes this NFR aspirational |
| NFR-13 (scale 10k cos / 1M opps) | base | E21 | ⚠️ **Conflict** — single-host topology (E22) cannot credibly back this NFR |
| NFR-14 (≥99.5%, → 99.9% per amendment) | base + amended | **E21 + E22 + E23** | ⚠️ **Conflict** — E22 ships "beta, best-effort availability"; E23 has `public-sla-announcement-soak-gate` still `ready-for-dev`. Public PRD posture mismatches launch posture. |
| NFR-15 (data integrity, daily backups, 24h PITR) | base | E22 (off-site to Hetzner) | ✅ Covered |
| NFR-16 (idempotency) | base | E08 (Stripe) + E28 (webhooks) | ✅ Covered |
| NFR-17 (DR/RTO 4h regional) | base | E22 | ⚠️ **Conflict** — "regional outage" assumes multi-region; on-prem single-host has no regions; E22 honestly defends "≤4h RTO" but it's best-effort |
| NFR-18 (WCAG 2.1 AA) | base | E03 | ✅ Covered (implicit, no dedicated story confirmed) |
| NFR-19 (keyboard nav) | base | E03 | ✅ Covered (implicit) |
| NFR-20 (screen reader) | base | E03 | ✅ Covered (implicit) |
| NFR-21 (ruff+mypy+eslint+tsc) | base | E01 (CI pipeline) | ✅ Covered |
| NFR-22 (80% unit / 90% E2E) | base | Cross-cutting | ⚠️ Implicit — no dedicated epic; lives in `Makefile` `make coverage` per CLAUDE.md. Acceptable. |
| NFR-23 (structured logs + Prometheus) | base | E21 (PE.05 metrics) | ✅ Covered |
| **NFR-24 (SirmaAI key rotation)** | 2026-05-12 | E24 | ✅ Covered |
| **NFR-25 (tier ↔ rate-limit sync)** | 2026-05-12 | E04 + E24 | ✅ Covered |
| **NFR-26 (SirmaAI as critical dep)** | 2026-05-12 | E28 | ✅ Covered |

### Missing Requirements

**No FRs are entirely uncovered.** All 80+ functional requirements have at least one epic home.

**Implicit-only coverage that should be made explicit (traceability hygiene):**
1. FR-9 (user belongs to multiple workspaces) — E14 covers `client_workspaces`; verify cross-workspace user membership is a story, not just a model field.
2. FR-50 (KB semantic search) — E25 goal mentions it; verify dedicated story.
3. NFR-3, NFR-18, NFR-19, NFR-20 (Lighthouse + WCAG/Keyboard/Screen reader) — all assumed covered by E03 design system. No dedicated accessibility story; recommend at minimum a WCAG audit story.
4. NFR-9 (dependency vuln scan / Dependabot) — E13 retro flagged as still pending 2026-04-25. **Confirm Dependabot is live.**

### NFR-vs-Reality Conflicts (4 items — material readiness blockers)

| # | NFR | PRD says | Epic reality | Severity |
|---|---|---|---|---|
| C1 | NFR-12 | "DB read replicas" | Single-host PostgreSQL container on www1 per E22/ADR-010 | **Doc inconsistency** — update NFR-12 to "post-launch hardening" or accept conflict |
| C2 | NFR-13 | "10k companies / 1M opps; <20% degradation over 2y" | Single-host topology has hard ceiling | **Doc inconsistency** — restate as a post-onprem-pivot target |
| C3 | NFR-14 | "99.9% uptime SLA (rolling 30d, service credits)" | E22 ships "beta, best-effort"; E23 `public-sla-announcement-soak-gate` still ready-for-dev | **Public-posture blocker** — cannot publish 99.9% SLA until soak gate clears |
| C4 | NFR-17 | "Restore within 4h after full regional outage" | Single-host = no "regions"; E22 RTO ≤4h is best-effort | **Doc inconsistency** — update NFR-17 to single-host RTO/RPO language |

### Verifications Needed Before Sign-off (5 items)

1. **E05 amendment** — confirm E05 was amended to acknowledge SirmaAI-driven ingestion per FR-15-new, or that FR-15 routes through E26 alone with E05 deprecated.
2. **E11 amendment** — confirm E11 (FR-36) content reflects SirmaAI ESPD Auto-Fill agent invocation, not legacy profile-to-XML mapping.
3. **E04 amendment** — confirm NFR-2 amended-language is present in E04 (sync-stream vs async-run distinction).
4. **E14 audit_log migration** — confirm `workspace_id` was added to `shared.audit_log` per FR-43 amendment dependency.
5. **E17 ↔ E27 reconciliation** — E17 file explicitly says E27 is authoritative for post-pivot CRM. Confirm E17 stories that overlap with E27 are marked deferred/superseded, not double-counted.

### Coverage Statistics

- Total PRD FRs (composite): **80+ slots** (44 base + 11 SirmaAI net-new + ~25 sub-FRs from 4-25 amendment)
- FRs with epic home: **All accounted for** (0 entirely missing)
- FRs with **explicit** epic-file traceability: **9 of 28 epic files** carry inline `FR-NN` refs — **~30% explicit traceability coverage**
- NFRs total: **26** (23 base + 1 amended + 2 new)
- NFRs with NFR-vs-reality conflicts: **4** (NFR-12, NFR-13, NFR-14, NFR-17 — all driven by on-prem pivot)

**Headline gap:** Coverage is *complete* at the epic level, but **traceability is implicit in 19 of 28 epic files**. Combined with `epics.md` being stale and 68 zombie `epic-NN-*.md` files coexisting in the same folder, a fresh BMAD planning pass would have non-trivial trouble reconstructing what's live without operator guidance.

## UX Alignment Assessment (Step 4)

### UX Document Status

**Found:** `ux-spec.md` — Sally (BMAD UX Designer), v3.1, **dated 2026-05-05**, status "autopilot-verified against PRD 2026-04-27". 705 lines, 16 sections, includes explicit §15 PRD traceability matrix.

**Critical timing issue:** **UX spec is 7 days older than the 2026-05-12 SirmaAI amendment.** It is autopilot-verified against PRD v1 (2026-04-27) only; it predates and does not reflect the SirmaAI substrate pivot. It also only partially reflects the 2026-04-25 amendment (Personas + multi-tenant chrome present; per-bid SKU, NPS, Trust Center, Outcome Dashboard surfaces absent).

### UX ↔ PRD Alignment

#### What's well-aligned

| Area | UX evidence | PRD evidence |
|---|---|---|
| Personas | §2.1–§2.5 (Elena, Maria, Stefan, Iliyan, Daniel) | PRD §User Journeys (Journey 1–4 + amendment §3 consulting-firms primary ICP) |
| Multi-tenant chrome | §3.4 Global Layout + P6 "Multi-tenant clarity" principle | FR-8, FR-9 + FR8.x family |
| Bilingual (BG/EN) | §10 Internationalization, P8 "Bilingual native" | GovTech domain reqs |
| Accessibility commit | §9 WCAG 2.1 AA + §9.8 testing gates | NFR-18, NFR-19, NFR-20 |
| Performance budgets | §6 patterns include streaming, skeleton states, TTFB targets | NFR-1, NFR-2, NFR-3, NFR-4 |
| Human-in-the-loop AI | P1 principle (every AI surface editable, regenerate, explain, provenance) | PRD Innovation §Risk Mitigation |
| Tier-gating UX | Journey C, §5.5 paywall card, P2 "time-to-value first" | FR-11, FR-12 |
| Time-to-value SLA | P2 "trial user reaches AI summary in <5min" | PRD Success Criteria "first session" |

UX-spec §15 traceability matrix covers **FR-1, FR-2, FR-4–FR-13, FR-15, FR-16, FR-19, FR-21–FR-30, FR-33–FR-44** and **NFR-1, NFR-2, NFR-3, NFR-4, NFR-7, NFR-8, NFR-18, NFR-19, NFR-20** — **42 of 44 base-PRD FRs** and **9 of 23 base-PRD NFRs**. (FR-3 login/logout and FR-14 add-on purchase weren't explicitly listed but are implicit in Journey A and Journey C respectively.)

#### Missing UX coverage — material gaps

The UX spec **does not contain screen/flow definitions** for the following requirements added after 2026-05-05:

| Missing area | FRs affected | Severity |
|---|---|---|
| **Knowledge Base management UI** — upload, semantic search, lifecycle, citation panel in agent outputs | FR-49, FR-50, FR-51, FR-52 | 🔴 **High** — KB is core to every SirmaAI agent interaction; users need to manage it |
| **Qualification report view** — fit/gap/recommendation/confidence display per opportunity | FR-47 | 🔴 **High** — surfaces on opportunity detail; affects bid/no-bid decision UI |
| **Quantification report view** — effort/p(win)/expected value/bid threshold | FR-48 | 🔴 **High** — same surface as qualification |
| **SirmaAI Project provisioning status UI** — `pending` / `provisioned` / `failed` indicator on company create + admin orphan view | FR-45 | 🟡 Medium — admin + onboarding banner |
| **CRM via MCP connection flow** — OAuth to Dynamics 365 / HubSpot, MCP server registration UI, status indicators | FR-53 | 🟡 Medium — supersedes E17 OAuth UX which UX spec didn't detail anyway |
| **Per-bid SKU purchase flow** — opportunity-detail "Activate per-bid" CTA, Stripe one-time checkout, post-activation badge | FR2.4, FR2.5, FR2.6 | 🟡 Medium — UX spec mentions tier paywall card (§5.5) but not per-bid SKU |
| **Outcome Dashboard** — workspace-level KPIs, win-rate trend, hours saved, content reuse | FR9.1–FR9.5 | 🟡 Medium — UX spec §5.3 Dashboard is generic post-login home, not Outcome Dashboard |
| **Outcome Brief PDF** — preview + download from dashboard | FR9.4 | 🟢 Low — server-side render; surface is just a download button |
| **Onboarding milestone tracker** — "First Bid in 14 Days" widget + CSM stall alerts | FR9.6 | 🟢 Low — widget on dashboard |
| **Public Trust Center page** — `/trust` route content, downloadable artefact list, sub-processor diff view | FR10.1, FR10.2 | 🟡 Medium — public marketing surface; outside §3 IA but should be in §5.1 |
| **NPS prompt** — in-product modal + opt-in to G2/Capterra/Gartner | FR2.7 | 🟢 Low — simple modal flow |
| **External-collaborator magic-link landing** — read-only / comment-only proposal view for end-clients invited without seat consumption | FR8.5 | 🟡 Medium — unique flow, no analog in current UX spec |
| **Long-running agent run progress UI** — async-run jobs with status polling per amended NFR-2 | NFR-2 (SirmaAI) | 🟡 Medium — UX spec §6.5 AI patterns assumes synchronous streaming only |

#### UX ↔ Architecture Alignment

| Area | Verdict |
|---|---|
| Streaming AI (SSE + TTFB <500ms) | ✅ UX §6.5 aligns with architecture AI Gateway SSE proxy; needs **amendment** for SirmaAI sync-stream vs async-run distinction |
| Multi-tenant chrome (workspace switcher) | ✅ Aligns with E14 + frontend `/[locale]/workspace/[workspaceId]/...` routing |
| Responsive web only (no native mobile MVP) | ✅ Aligns with Turborepo apps/client + apps/admin |
| Component library | ✅ shadcn-based per UX §16 → matches frontend `packages/ui` |
| AI provenance labels ("AI draft — review needed") | ✅ Supports the human-in-the-loop pattern |
| WCAG 2.1 AA gate in PR template | ⚠️ Mentioned in UX §9.8 but not confirmed implemented in `.github/workflows/` or coding-standards/ |
| Public Trust Center surface | ⚠️ UX IA (§3.1) doesn't include `/trust`; would need addition to public marketing surfaces (§5.1) |

### Open UX Questions (carry-forward from UX spec §14)

| # | Topic | Status |
|---|---|---|
| Q1 | Section locking concurrency model (pessimistic vs CRDT) | Open — Phase 2 |
| Q2 | Mobile editing scope for Phase 2 | Open |
| Q3 | Score simulator confidence display (band vs point) | Open — A/B candidate |
| Q4 | Bilingual proposal export (combined vs two files) | Open — Legal+Product |
| Q5 | Admin "impersonate tenant" banner UX | Open — security recommendation logged |

UX spec asserts none of these block MVP. Accept as-is.

### Warnings

1. **UX spec must be re-baselined for SirmaAI pivot.** Recommend bmad-create-ux-design or bmad-agent-ux-designer pass to:
   - Add §5.X screens for KB management, Qualification/Quantification reports, CRM via MCP, Outcome Dashboard, Trust Center, per-bid SKU flow, NPS prompt, external-collaborator magic-link landing.
   - Amend §6.5 AI patterns to cover async-run progress UX (long-running quantification, ingestion completion).
   - Update §15 traceability matrix to include FR-45 to FR-55 and the FR2/8/9/10/11 sub-families.

2. **Accessibility gate enforcement** — UX §9.8 references a 25-item per-screen checklist that should be in `coding-standards/` and the PR template. **Verify it exists** before claiming NFR-18 compliance. If absent, this is a real readiness gap.

3. **Public Trust Center IA gap** — `/trust` is launch-blocking per E18 (Month 2 hard deadline). UX spec needs an explicit §5.X entry and a §3.1 IA addition for the public marketing surfaces.

4. **Async-run UX pattern unspecified** — Quantification, deep-scoring, and ingestion are long-running (per amended NFR-2). UX spec only specifies streaming sync patterns. **Pattern needed before E26 stories can be designed UX-first.**

### Assessment Summary

- **UX-PRD alignment:** ✅ Strong for base PRD (v1, 2026-04-27)
- **UX-amendment alignment:** ⚠️ Partial for 2026-04-25 (consulting-firm pivot); ❌ Missing for 2026-05-12 (SirmaAI pivot)
- **UX-Architecture alignment:** ✅ Aligned at the patterns level; needs amendment for SirmaAI async-run flows
- **Material gaps:** 13 missing UX surfaces driven by post-2026-05-05 PRD amendments (3 High, 6 Medium, 4 Low severity)

## Epic Quality Review (Step 5)

**Scope:** 28 canonical `E##-*.md` epics audited against `bmad-create-epics-and-stories` standards.

### A. User Value Focus — ✅ Pass

Every epic delivers user-visible outcome, even the technical ones:

| Epic | User-Value Read |
|---|---|
| E01 Infrastructure Foundation | Greenfield bootstrap — required first epic; ships fully-functional dev env, not a vacuum-named "infra" epic |
| E04 AI Gateway Service | Service-skeleton epic but tied to specific user-facing AI capabilities |
| E13 Hardening & Drift Recovery | Closes carry-forward `[ACTION]` items; user value = NFR conformance for Beta gate |
| E28 Webhook & Run-State Reconciliation | "Reliable AI run lifecycle" — user value = no silent loss of expensive analyses |

No epic is titled "Database Setup" or "API Development" in isolation. ✅

### B. Epic Independence — ✅ Pass (with documented dependencies)

Dependency graph traced from inline references and epic-level `Cross-epic dependencies` sections:

```
E01 → E02 → (E03 frontend, E04 AI gateway)
                 ↓             ↓
              E14, E18      E05 ingest, E07 proposals, E11 grants
                            ↓
                            E06 discovery, E10 collab, E12 admin
                            ↓
                            E13 hardening (cross-cutting)

Post-pivot extension:
  E04 amended → E24 SirmaAI provisioning → (E25 KB, E26 ingestion, E27 CRM, E28 webhooks)
  E14 multi-client → (E15 Pro+/per-bid, E16 Slack/Teams, E17 CRM-original, E18 Trust Center,
                      E19 outcome, E20 NPS)
  E21 reliability ∥ E22 onprem → E23 close-out
```

**No forward "Epic N → Epic N+1" violations found.** All cross-references are backward (lower-numbered) or sideways (parallel).

**Documented dependencies (sample):**
- E05 amendment: "depends on E04 amendment landing (webhook receiver + reconciler)" — clean
- E16: "Helm chart with PDB + min replicas 2 (Epic 21 reliability prerequisite)" — sideways but E21 is parallel track
- E22 → E23: `public-sla-announcement-soak-gate` depends on E22 launch live + 2-week soak ✅ documented
- E27 → E17: E27 is the design narrative; E17 amendment is the execution unit (S17.30–S17.36) ✅ documented

### C. Story Sizing — ⚠️ Mostly Pass, 2 minor concerns

Total stories across all 28 epics: **~250+** (varies by counting convention).

| Epic | Story count | Verdict |
|---|---|---|
| E01–E12 (original 12) | 10–18 each | ✅ Healthy range |
| E13 | 12 (incl. coordinators with sub-stories) | ✅ Pattern documented |
| E14 | 5 | ✅ Foundational refactor, well-scoped |
| E15 | 3 | ✅ Per-bid SKU + Pro+ tier + metering bypass |
| **E16** | **1** | 🟡 **Tiny epic** — Slack/Teams webhook only. Justification valid (smallest viable integration, proves out `integrations-api` service skeleton), but borderline "feature, not epic" |
| E17 | 11 (4 original + 7 amendment delta) | ✅ Healthy |
| E18 | 3 | ✅ Trust Center static-render + WeasyPrint reuse |
| E19 | 3 | ✅ Outcome dashboard + brief + onboarding milestone |
| **E20** | **1** | 🟡 **Tiny epic** — NPS prompt + review routing only (milestones absorbed into E19). Defensible but warrants merge consideration with E19 |
| E21 | 6 PE.NN stories | ✅ |
| E22 | 6 onprem-NN | ✅ |
| E23 | 5 named stories | ✅ |
| E24 | 8 | ✅ |
| E25 | 9 | ✅ |
| E26 | 10 | ✅ |
| E27 | 8 (table-only, defers to S17.30–S17.36) | ⚠️ See §D below |
| E28 | 8 | ✅ |

🟡 **Concern**: E16 + E20 single-story epics could be folded into E15 (Pro+ tier) or E19 (outcome/onboarding) without harm. Not a blocker; documenting as future epic-design hygiene.

### D. Acceptance Criteria & Story Structure — ⚠️ Pass with inconsistency

**Sampled AC quality:**
- E02 stories: Detailed inline AC bullets per story ✅
- E08 stories: Per-story `Description` + AC bullets ✅
- E24/E25/E26/E28 (post-pivot): Rich AC with explicit `Implementation Notes` ✅
- **E27**: Stories in **table format only** — no AC, no Given/When/Then. Explicitly delegates to E17 amendment for orchestrator dispatch. **Documentation gap** even though the work is real and described in E17. Recommend either materialize E27 stories with AC OR mark E27 file as "design narrative — see E17 for execution".

**Story-ID convention inconsistency:** S##.## (default), PE.NN (E21), onprem-NN (E22), named-IDs (E23: `pe-04-chaos-drill-execution`, `drift-recovery-story`), S17.30–S17.36 (E17 amendment for E27 work). Sprint-status keys by the actual story-id, so functionally it works, but raises cognitive load for new contributors.

### E. Database Creation Timing — ✅ Pass

- E01 ships base schema (auth, shared, pipeline) + Alembic scaffold
- Each subsequent epic ships its own migrations:
  - E02 → auth + identity + audit tables
  - E08 → subscriptions + tier_access_policies + add_on_purchases
  - E10 → proposal_collaborators + tasks + dependencies
  - E14 → client_workspaces + workspace_id audit-log migration (atomic via S14.00)
  - E24 → sirmaai_projects table
  - E25 → knowledge base metadata pointers
  - E28 → idempotency cache + dead-letter queue tables

Pattern correctly applied. No "create all tables upfront" violation.

### F. Greenfield Bootstrap — ✅ Pass

- E01.S01.01 = "Monorepo Scaffold & Project Structure" ✅
- E01.S01.02 = "Docker Compose Local Development Environment" ✅
- E01.S01.08 = "GitHub Actions CI Pipeline" ✅
- E03 = full frontend shell with i18n + design system + auth flows ✅

### G. Traceability to FRs — ⚠️ Major Issue (carries forward from Step 3)

**Only 9 of 28 epic files carry explicit `FR-NN` references in their text.** 19 epics map to FRs implicitly via goal/story narrative. This is the most material epic-quality finding in the audit.

**Recommendation:** Add an `## FRs Covered` line near each epic's top, listing FR/NFR numbers. Low-effort, high-impact for orchestrator and future BMAD planning passes.

### H. E13 Coordinator Pattern — ✅ Pass with one open item

E13 documents the "coordinator-with-sub-stories" pattern explicitly (§215). All 12 stories status=done except `drift-recovery-story` which is `in-progress` per sprint-status (code-review verdict 2026-05-13 = Changes Requested, 10 patch items + 2 inline decisions).

**Risk:** E13 AC asserts "zero rollover technical debt and verifiable NFR conformance" for Beta gate. With `drift-recovery-story` open, the Beta gate cannot formally close.

### Findings Summary by Severity

#### 🔴 Critical Violations
- **None.** No technical-only epics, no forward dependencies, no impossible-to-complete stories.

#### 🟠 Major Issues
1. **Implicit FR traceability in 19 of 28 epic files** (carries forward from Step 3) — orchestrator and reviewers cannot mechanically verify FR coverage without manual reading.
2. **E27 stories in table-only format** — no AC, no Given/When/Then; relies on E17 amendment for execution detail. Either materialize or mark E27 as design narrative.
3. **NFR-12, NFR-13, NFR-14, NFR-17 conflicts** (from Step 3) — PRD claims AWS-grade scaling/availability that single-host on-prem (E22) cannot back. Update PRD or amend epics.
4. **`drift-recovery-story` open** — blocks formal Beta-gate per E13 AC.

#### 🟡 Minor Concerns
5. E16 and E20 are single-story epics — defensible but borderline; consider folding into E15/E19.
6. Story-ID convention inconsistency (S##.##, PE.NN, onprem-NN, named) — readability cost only.
7. `epics.md` is stale (2026-05-05, missing E14–E28, FR Coverage Map placeholder unfilled) — should be regenerated or marked deprecated.
8. UX spec predates SirmaAI pivot (2026-05-05) — 13 missing UX surfaces (from Step 4).
9. NFR-22 (test coverage) has no dedicated epic — implicit in Makefile. Acceptable but flag.

### Best Practices Compliance Checklist

| Epic | User value | Independent | Story sizing | No fwd-deps | DB timing | AC quality | FR traceability |
|---|---|---|---|---|---|---|---|
| E01–E13 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Mostly ❌ |
| E14 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit FR8 |
| E15 | ✅ | ✅ | 🟡 small | ✅ | ✅ | ✅ | ✅ explicit FR2.4 |
| E16 | ✅ | ✅ | 🟡 1 story | ✅ | n/a | ✅ | ✅ explicit FR11.4 |
| E17 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit FR11.1, FR-53 |
| E18 | ✅ | ✅ | ✅ | ✅ | n/a | ✅ | ✅ explicit FR10 |
| E19 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit FR9, FR7.2 |
| E20 | ✅ | ✅ | 🟡 1 story | ✅ | n/a | ✅ | ✅ explicit FR2.7, FR9.6 |
| E21 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit NFR-2, NFR-13 |
| E22 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ implicit |
| E23 | ✅ | ✅ | ✅ | ✅ | n/a | ✅ | ❌ implicit |
| E24 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit FR-45/46 |
| E25 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit FR-49/52 |
| E26 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit FR-15/47/48 |
| E27 | ✅ | ✅ | ✅ (via E17) | ✅ | ✅ | ❌ table only | ✅ explicit FR-53 |
| E28 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ explicit FR-54/55/NFR-26 |

## Summary and Recommendations

### Overall Readiness Status

**🟡 NEEDS WORK — but not from scratch.** Implementation is ~95% done (191 of ~210 stories status=done per `sprint-status.yaml`). The blockers to formal launch readiness are **documentation-and-doc-alignment defects + 13 open-status stories + 4 PRD↔reality conflicts** introduced by the on-prem pivot (2026-05-11) and the SirmaAI pivot (2026-05-12). None of these require greenfield work — they are surgical fixes to documents already on disk.

A more candid phrasing: **the code is launch-ready; the planning documents lag the code by two pivots' worth of edits.**

### Critical Issues Requiring Immediate Action

#### 🔴 Launch Blockers (must close before public launch)

1. **`drift-recovery-story` open** (E13 carry-forward, code-review verdict 2026-05-13 = Changes Requested with P2 VAT-validation circuit-open gap). VAT may be incorrectly charged to B2B EU customers whose `tax_exempt` didn't sync. Tax-compliance risk. **Action:** dispatch review-fix per the 10 patch items already authored in story §Review Findings.

2. **6× `onprem-NN` stories in `review` status** (E22: postgres backup, redis persistence, monitoring, disk/resource monitoring, deploy guardrails, www1-as-code). All gate the on-prem launch on `www1.endigitalx.com`. **Action:** parallelize review; single-reviewer bottleneck is the immediate risk.

3. **`pe-06-pagerduty-rotation-provisioning` in `review`** + **`public-sla-announcement-soak-gate` ready-for-dev** + **`pe-04-chaos-drill-execution` ready-for-dev** (E23). Together these gate the 99.9% SLA public posture. **Action:** confirm soak clock has started against E22 launch-live.

4. **PRD↔reality conflicts (4 items) from on-prem pivot** — NFR-12, NFR-13, NFR-14, NFR-17 promise AWS-grade scaling/HA/multi-region DR that single-host www1 cannot back. **Action:** amend PRD `prd-amendment-2026-05-13-onprem-nfrs.md` to align NFR language with single-host topology (beta-posture availability, off-site backups, no multi-region, single-host scaling ceilings).

5. **SirmaAI EU-only residency confirmation** — flagged as launch-blocking due-diligence item in SirmaAI amendment §Data Residency. **Action:** obtain written confirmation from SirmaAI vendor; file under `eusolicit-docs/compliance/`.

#### 🟠 Major Defects (close before next BMAD planning pass)

6. **UX spec predates SirmaAI pivot.** 13 missing UX surfaces (3 High severity: KB management, Qualification, Quantification report views). **Action:** invoke `bmad-create-ux-design` or `bmad-agent-ux-designer` to add §5.X screens + §6.5 async-run UX pattern + §15 traceability update.

7. **Implicit FR traceability in 19 of 28 epic files.** Orchestrator cannot mechanically verify coverage. **Action:** add `## FRs Covered` section to each epic header (low-effort batch edit).

8. **`epics.md` is stale** (2026-05-05, missing E14–E28, FR Coverage Map placeholder unfilled). **Action:** either regenerate from `E##-*.md` source-of-truth or mark deprecated and point readers to the folder.

9. **68 zombie `epic-NN-*.md` files in `planning-artifacts/epics/`.** Will trip a fresh BMAD planning pass. **Action:** `mkdir -p epics/_archive && mv epic-*.md epics/_archive/` to clear the folder.

10. **5 epic amendments need verification** (Step 3 §Verifications Needed) — E04, E05, E11, E14 audit_log, E17↔E27 reconciliation. **Action:** spot-check each file for amended content (15-min audit).

#### 🟡 Minor Concerns (hygiene)

11. **NFR-9 (Dependabot)** — confirm landed; flagged pending in 2026-04-25 E13 retrospective. Verify in `.github/dependabot.yml` or equivalent.
12. **E27 stories in table-only format, no AC** — either materialize stories with AC or mark E27 file as design-narrative.
13. **E16 + E20 are single-story epics** — consider folding into E15 / E19 for future planning passes.
14. **Story-ID convention inconsistency** (S##.##, PE.NN, onprem-NN, named-IDs) — cognitive load only; sprint-status handles all.
15. **5 open UX questions** (UX spec §14) — non-blocking; defaults documented.
16. **6 open PRD-amendment questions** (2026-04-25 §Open Questions) — Pro+ pricing, per-bid pricing, Trust Center deadline, ISO budget, CRM prioritisation, existing-customer impact. Some moot post-SirmaAI (e.g., CRM prioritisation overridden).

### Recommended Next Steps

In execution order:

1. **Triage `drift-recovery-story`** (today) — dispatch review-fix on the 10 patch items; P2 VAT path is the unblock-first item.
2. **Parallelize the 6× `onprem-NN` reviews** (this week) — appoint reviewers; track via sprint-status.
3. **Write `prd-amendment-2026-05-13-onprem-nfrs.md`** (this week) — surgical amendment of NFR-12/13/14/17 to match single-host topology.
4. **Obtain SirmaAI EU-residency letter** (this week, vendor-paced) — file in compliance folder.
5. **Re-baseline UX spec for SirmaAI** (next sprint) — Sally pass adding 13 missing surfaces.
6. **Add `## FRs Covered` lines to 19 epics** (1-day batch edit) — close implicit-traceability gap.
7. **Archive zombie epics** (15 minutes) — `mkdir epics/_archive && mv epic-*.md epics/_archive/`.
8. **Spot-check 5 epic amendments** (15 minutes) — confirm E04/E05/E11/E14/E17 are amended in place.
9. **Drive `public-sla-announcement-soak-gate`** (calendar-paced, 2-week soak post-E22 live) — do not publish 99.9% SLA early.

### Final Note

This assessment identified **16 distinct issues across 6 categories** (Status, NFR Alignment, Traceability, UX Coverage, Doc Hygiene, Epic Structure). **No issue requires net-new feature work** — every blocker is a surgical edit to existing artefacts, a status transition on an existing story, or a vendor sign-off.

The product itself is in launch-ready shape. The documentation needs two pivots' worth of catching-up. With the 9-step plan above, this is **3–5 days of focused operator work plus a 2-week SLA soak** to a clean launch posture.

---

**Assessment Date:** 2026-05-13
**Assessor:** 📋 John (Product Manager) via `bmad-check-implementation-readiness`
**Method:** 6-step PRD/UX/Architecture/Epics traceability audit against canonical sources
**Source documents:** `PRD.md` + 2 amendments, `architecture.md` + 1 amendment, `ux-spec.md`, 28 canonical `E##-*.md` epics, `sprint-status.yaml`
**Excluded (out of scope or stale):** `epics.md` (stale 2026-05-05), 68 zombie `epic-NN-*.md` files, `ux-design-specification.md` (superseded), backup files
