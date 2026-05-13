# PRD Amendment — 2026-05-12 (SirmaAI Integration Pivot)

**Source PRD:** `eusolicit-docs/planning-artifacts/PRD.md` (Author: Deb, 2026-04-27)
**Trigger:** Sprint Change Proposal `sprint-change-proposal-2026-05-12-sirmaai.md` (approved 2026-05-12)
**Author:** 📋 John (PM)
**Status:** Draft for review — paired with `architecture.md` amendment + 3 new ADRs (Winston) in parallel

> This amendment is a **delta document**. It does not replace the PRD. On approval, sections below are merged into `PRD.md` in a tracked edit; the original PRD file moves to `PRD.v2.0.bak.md` (file already reserved). Section headings reference the live PRD's `##` and `###` anchors.

---

## Why this amendment

The product is at the same line on the wall it was at on 2026-04-27 — automate the EU procurement and grant lifecycle for SMBs and consulting firms. What changes is **where the AI substrate lives**: from "thin proxy fronting KraftData logical-named agents called from our Celery pipeline" to "SirmaAI Projects owning agent runtime, with EU Solicit orchestrating and holding the canonical record." Customer-facing capability stays largely intact; behind the line, ingestion inverts, tenancy gains a SirmaAI dimension, and CRM scope re-shapes.

---

## §Executive Summary (whole-paragraph amendment)

**§Executive Summary, paragraph 2 (line 32):**

> **OLD:** *"The core differentiator is a unified, end-to-end workflow powered by KraftData's sophisticated multi-agent AI system. This approach replaces a fragmented market of single-point solutions and manual consulting services. The core insight is that modern Agentic AI is now capable of handling the complex, unstructured data and nuanced analytical tasks of tender evaluation and proposal writing, which previously required significant, high-cost human expertise."*

> **NEW:** *"The core differentiator is a unified, end-to-end workflow in which the **SirmaAI agentic substrate** (at `https://agenticsai.endigitalx.com/`) is the dominant runtime for qualification, quantification, analysis, and content generation, while EU Solicit acts as the canonical record keeper, orchestrator, and customer-facing surface. Each EU Solicit company maps to a dedicated SirmaAI Project under a single EU Solicit Organization, giving every tenant an isolated agent / team / workflow / knowledge-base scope. This approach replaces a fragmented market of single-point solutions and manual consulting services. The core insight is that modern Agentic AI — composed of specialised agents, persistent vector knowledge bases, and per-tenant workflow orchestration in N8N — is now capable of handling the complex, unstructured data and nuanced analytical tasks of tender evaluation and proposal writing, which previously required significant, high-cost human expertise."*

**Rationale:** Names SirmaAI explicitly; introduces the Org/Project tenancy model in the customer-facing framing; foreshadows the KB and workflow constructs that subsequent FRs depend on.

---

## §Project Scoping & Phased Development — §MVP Strategy & Philosophy (resource line)

**Line 80 (Resource Requirements):**

> **OLD:** *"…leveraging the managed services of the KraftData AI platform."*
> **NEW:** *"…leveraging the SirmaAI agentic substrate (Organisation + per-tenant Projects, vector knowledge bases, agent / team / workflow execution, and the shared EU-Solicit-owned N8N instance)."*

---

## §Phase 1: MVP Feature Set (capability-line amendments)

**Line 88 (Data Pipeline capability):**

> **OLD:** *"Automated daily crawling of AOP and TED portals."*
> **NEW:** *"Automated daily ingestion of AOP and TED opportunities via N8N workflow templates invoking SirmaAI crawler / normalisation / scoring agents; normalised opportunity rows persisted to EU Solicit's canonical Postgres via Standard Webhooks."*

**Insert new Must-Have line after line 89:**

> **NEW (FR-traceable):** *"Per-tenant SirmaAI Project auto-provisioning on company create, with a seeded knowledge base; tenant-default MCP server registrations for CRM connectivity (Dynamics 365 + HubSpot)."*

**Rationale:** Brings the Phase-1 MVP definition into alignment with the new substrate; auto-provisioning becomes a launch-critical capability (without it, no tenant can use the platform).

---

## §Phase 3: Expansion (CRM/ERP integrations line)

**Line 111 (Marketplace & Ecosystem):**

> **OLD:** *"…and deep integrations with CRM/ERP systems."*
> **NEW:** *"…and additional CRM/ERP MCP servers beyond the v1 set (Pipedrive, Salesforce, NetSuite, SAP) added as SirmaAI MCP-server registrations without changing the EU Solicit contract."*

**Rationale:** Maps the previously-vague "Phase 3 CRM" line to the now-explicit MCP-server extension model.

---

## §Functional Requirements — §User and Tenant Management (insert)

**Insert after FR-9:**

> **NEW — FR-45 (Per-tenant SirmaAI Project provisioning):** *"On company creation in EU Solicit, the system shall automatically create a corresponding SirmaAI Project under the singleton EU Solicit Organisation. Provisioning shall include: generation of a Project-scoped API key (Fernet-encrypted at rest in EU Solicit); seeding of a default knowledge-base storage-resource; registration of tenant-default MCP servers (Dynamics 365 + HubSpot stubs, inactive until OAuth-connected); publication of a `tenant.provisioned` event. Provisioning shall be retried with exponential backoff on transient SirmaAI failures and surfaced via a `provisioning_status` field on the company record (`pending` / `provisioned` / `failed`). A nightly reconciliation job shall verify every active EU Solicit company has a healthy SirmaAI Project and flag orphans for admin attention."*

> **NEW — FR-46 (Per-tenant SirmaAI Project archival):** *"On company archive in EU Solicit, the system shall soft-delete the corresponding SirmaAI Project and revoke its API key within 24 hours. Archived Projects shall not be re-provisioned on company restoration without explicit admin action."*

**Rationale:** Tenant lifecycle in EU Solicit and SirmaAI must remain in lockstep; without this, every other SirmaAI-facing capability (KB, agents, MCP servers) has no parent to attach to.

---

## §Functional Requirements — §Data Pipeline & Opportunity Discovery (replace FR-15)

**FR-15 (whole-line replace):**

> **OLD:** *"FR-15: The system can automatically ingest new procurement opportunities from configured public sources (AOP, TED)."*

> **NEW — FR-15:** *"The system shall automatically ingest new procurement opportunities from configured public sources (AOP, TED, and EU Grants portals) via N8N workflow templates running in the shared EU Solicit-owned N8N instance. Each workflow shall invoke SirmaAI crawler, normalisation, relevance-scoring, and submission-guide agents under the calling tenant's Project scope. Normalised opportunity records shall be delivered to EU Solicit's canonical `pipeline.opportunities` Postgres table via Standard Webhooks (`workflow.completed`, `agent.run.completed`). The system shall retain the `pipeline.crawler_runs` audit table for historical run visibility and shall publish `opportunities.ingested` events to the internal Redis Streams event bus on each successful cycle."*

**Rationale:** Reflects the inversion from EU Solicit-driven Celery crawl to SirmaAI-agent-driven crawl orchestrated by N8N; preserves canonical-record ownership and downstream event contracts so existing consumers (Opportunity Discovery, Notifications, Analytics) remain unchanged.

---

## §Functional Requirements — §AI-Powered Analysis & Insights (insert + amend)

**Replace introductory subsection text (currently implicit):**

> **NEW preamble:** *"All AI analysis and insight functions in this subsection are executed by SirmaAI agents and teams within the calling company's SirmaAI Project. Long-running analyses (qualification, quantification, deep-scoring) shall use the async `run-async` + `jobs/{jobId}/status` polling pattern via `sirmaai-gateway`; sub-15-second analyses (executive summary, requirement extraction) shall use synchronous run. All agents have access to the tenant's knowledge-base storage-resource for grounded context."*

**Insert after FR-25:**

> **NEW — FR-47 (Opportunity Qualification Analysis):** *"For every newly ingested or user-triggered opportunity, the system shall invoke a SirmaAI qualification agent that produces a structured qualification report: fit assessment against the company profile, gap analysis, recommended action (`pursue` / `monitor` / `decline`), and confidence score. Results shall be persisted in EU Solicit Postgres and surfaced in the opportunity detail view."*

> **NEW — FR-48 (Opportunity Quantification Analysis):** *"On user request or on `qualification = pursue`, the system shall invoke a SirmaAI quantification agent producing: estimated effort (person-days), estimated probability of win, expected value (EUR), and a recommended bid/no-bid threshold. The agent shall consume the company's KB resources and past-proposal outcomes."*

**Rationale:** Lifts qualification and quantification from implicit "we'll figure it out" to explicit FRs aligned with Deb's stated requirement: *"all data ingestion flows are dominated by the AI agents in SirmaAI, for qualification, quantification analysis, as well as further processing."*

---

## §Functional Requirements — NEW SUBSECTION §Knowledge Base

**Insert as a new `###` subsection between `### AI-Powered Analysis & Insights` and `### Proposal Generation & Collaboration`:**

> ### Knowledge Base
>
> **FR-49 (KB artefact upload):** Users may upload knowledge-base artefacts (tender PDFs, ESPD templates, company profile documents, past proposals, qualification rubrics) to their company's SirmaAI Project storage-resource. Supported formats: PDF, DOCX, TXT, MD. Maximum file size and per-tenant quota tier-gated (Free: 50MB, Starter: 500MB, Professional: 5GB, Enterprise: unlimited).
>
> **FR-50 (KB semantic search):** Users may execute semantic search across their tenant KB and receive ranked passages with source-document references.
>
> **FR-51 (KB consumed by agents):** All AI agents executing within a tenant Project shall have read access to that tenant's KB and shall ground their outputs in retrieved passages where applicable. Citations to KB sources shall be returned in structured agent outputs.
>
> **FR-52 (KB lifecycle):** Users may replace, archive, or delete KB artefacts; agent runs shall reflect the current KB state within 5 minutes of mutation (re-index latency budget). Profile updates (company description, sectors, key personnel) shall trigger automatic KB re-index.

**Rationale:** First-class FR for KB makes the requirement *"artefacts stored in knowledge base for operability and further use by the agents"* enforceable; tier-gating quota lines fold cleanly into existing Stripe-tier metering.

---

## §Functional Requirements — §Notifications and Administration (insert)

**Insert after FR-44:**

> **NEW — FR-53 (CRM enrichment via SirmaAI MCP):** *"The system shall expose CRM connectivity for **Microsoft Dynamics 365** and **HubSpot** as SirmaAI MCP servers registered under each tenant's Project. Each MCP server shall expose a stable tool surface (`find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`) callable by qualification, opportunity-lifecycle, and proposal agents. OAuth flow: EU Solicit hosts the OAuth callback, stores access + refresh tokens Fernet-encrypted in `client.crm_connections`, and injects tokens into the MCP-server configuration at registration. Pipedrive and Salesforce connectors are deferred to post-launch (post-MVP) as additional MCP-server registrations following the same pattern. Tier-gated to Pro+ and above."*

> **NEW — FR-54 (Standard Webhooks receiver):** *"The system shall host an authenticated, public webhook receiver consuming SirmaAI Standard Webhooks events (`workflow.completed`, `agent.run.completed`, `storage.file.processed`, `policy.violation`). Receiver shall verify HMAC signatures using `hmac.compare_digest`, deduplicate via a Redis-backed idempotency cache (7-day TTL), route events to internal Redis Streams, and write poison events to a dead-letter queue for admin review."*

> **NEW — FR-55 (Run-state reconciliation):** *"A scheduled job shall poll non-terminal SirmaAI agent / workflow runs every 5 minutes via `GET /jobs/{jobId}/status` and converge `gateway.workflow_runs` to terminal state. This reconciler is the authoritative truth for run lifecycle, with webhooks treated as latency optimisation."*

**Rationale:** Makes the CRM-via-MCP requirement explicit (with the v1 scope swap recorded in the FR itself), gives the webhook receiver and reconciler the requirement weight they need to land in epics E26 and E28.

---

## §Functional Requirements — §Compliance & Workflow (amend FR-36)

**FR-36 (whole-line replace):**

> **OLD:** *"FR-36: The system can generate a pre-filled European Single Procurement Document (ESPD) from a company's profile."*

> **NEW — FR-36:** *"The system shall generate a pre-filled European Single Procurement Document (ESPD) by invoking the SirmaAI **ESPD Auto-Fill agent** within the tenant Project, supplying the company profile and target-opportunity context. The agent's structured output shall be rendered as ESPD-compliant XML (per the EU ESPD schema) and PDF (via reportlab) by EU Solicit. ESPD profile records, framework assignments, and exported artefacts remain canonical in EU Solicit Postgres."*

**Rationale:** Reflects the agent-definition migration to SirmaAI (E11 modification) while preserving the EU-Solicit-owned XML/PDF rendering responsibility.

---

## §Non-Functional Requirements — §Security (amend NFR-6, insert NFR-24)

**NFR-6 (whole-line replace):**

> **OLD:** *"NFR-6 (Encryption): All data must be encrypted in transit using TLS 1.3+ and at rest using AES-256. Sensitive data like external API keys must be additionally encrypted at the application level."*

> **NEW — NFR-6:** *"All data shall be encrypted in transit using TLS 1.3+ and at rest using AES-256. Sensitive credentials — per-tenant SirmaAI Project API keys, CRM OAuth tokens, SirmaAI webhook HMAC secrets, SirmaAI system API key — shall be additionally encrypted at the application level via the Fernet canonical module."*

**Insert as NFR-24 in §Security:**

> **NFR-24 (SirmaAI key rotation):** *"Per-tenant SirmaAI Project API keys shall be rotated on a configurable schedule (default 90 days) by an `sirmaai-gateway`-owned job; rotation shall be zero-downtime (new key issued and verified before old key revoked). Webhook HMAC shared secrets shall be rotated on the same cadence with double-validation overlap."*

---

## §Non-Functional Requirements — §Performance (amend NFR-2, insert NFR-25)

**NFR-2 (whole-line replace):**

> **OLD:** *"NFR-2 (AI Generation): The Time to First Byte (TTFB) for streaming AI-generated content (e.g., summaries, proposal drafts) must be less than 500ms."*

> **NEW — NFR-2:** *"For streaming AI-generated content (executive summaries, proposal drafts) invoked through SirmaAI synchronous-stream endpoints (`/agents/{id}/run/stream`), TTFB measured at the EU Solicit edge shall be < 500ms at p95. For long-running analyses invoked through async-run (`run-async` + job polling), first-result-available time is governed by SirmaAI job-engine latency and is not subject to the 500ms SLO; user-facing UX shall present progress state during these runs."*

**Insert as NFR-25 in §Performance:**

> **NFR-25 (Tier-to-rate-limit synchronisation):** *"EU Solicit tier upgrades and downgrades shall be mirrored to the SirmaAI Organization rate-limit policy via `PATCH /api/admin/organizations/{orgId}/rate-limit` within 60 seconds of the tier change event. Default mappings: Free → 100 agent-runs/day, Starter → 1k/day, Professional → 10k/day, Enterprise → 100k/day (subject to commercial calibration)."*

---

## §Non-Functional Requirements — §Reliability (insert NFR-26)

**Insert as NFR-26:**

> **NFR-26 (SirmaAI as external dependency):** *"SirmaAI is treated as a critical external dependency with no Multi-AZ failover. Outbound calls shall use the two-layer resilience pattern `circuit_breaker(retry(http_factory))` per logical agent/team/workflow name. On SirmaAI unavailability >5 minutes, EU Solicit shall surface a tenant-visible status banner ("AI analysis temporarily unavailable") and queue affected operations for retry. The run-state reconciler (FR-55) shall guarantee no in-flight run is silently lost."*

---

## §Domain-Specific Requirements (GovTech) — §Compliance & Regulatory (amend Data Residency)

**Data Residency line:**

> **OLD:** *"Data Residency: All customer data, without exception, must be stored and processed within EU data centers (specifically AWS eu-central-1 as per architecture)."*

> **NEW — Data Residency:** *"All customer data — including KB artefacts and agent execution traces — shall be stored and processed within EU data centres. EU Solicit's application data resides on `www1.endigitalx.com` (per ADR-010 on-prem pivot, 2026-05-11). SirmaAI's data residency for the EU Solicit Organisation shall be contractually confirmed to be EU-only; this is a launch-blocking due-diligence item."*

**Rationale:** Captures the on-prem pivot already recorded in ADR-010 and flags the SirmaAI residency confirmation as launch-blocking.

---

## §Domain-Specific Requirements (GovTech) — §Integration Requirements (amend Government Portals)

**Government Portals line:**

> **OLD:** *"The data pipeline must reliably integrate with official procurement portals like Bulgaria's AOP and the EU's TED. This requires robust crawlers that are actively maintained to handle changes in the source portals."*

> **NEW — Government Portals:** *"The data pipeline shall reliably integrate with official procurement portals (Bulgaria's AOP, EU's TED, EU Grants portals) via SirmaAI crawler agents executed within N8N workflow templates. Maintenance ownership for crawler resilience to source-portal HTML changes shall rest with the SirmaAI agent-definition repository (per-tenant Project template); EU Solicit's responsibility is workflow trigger orchestration, normalised-record persistence, and operational monitoring."*

---

## §SaaS B2B Platform Specific Requirements — §Tenant & Subscription Model (amend Multi-Tenancy)

**Multi-Tenancy line:**

> **OLD:** *"Multi-Tenancy: The platform is architected as a multi-tenant system. Each 'Company' or 'Workspace' is a distinct tenant. All data is strictly isolated at the database level using a `workspace_id` or `company_id` on all relevant tables, with database policies enforcing this separation. There are no cross-tenant queries at the application layer."*

> **NEW — Multi-Tenancy:** *"The platform is architected as a multi-tenant system spanning two cooperating substrates: (1) **EU Solicit Postgres** — canonical tenant boundary enforced via `workspace_id` / `company_id` on all relevant tables with database policies; no cross-tenant queries at the application layer; (2) **SirmaAI** — each EU Solicit company maps 1:1 to a SirmaAI Project under the singleton EU Solicit Organisation (per ADR-018). SirmaAI Project scope provides agent / team / workflow / KB / trace / memory isolation. N8N is shared at the Organisation level; workflow templates are parameterised by `projectId` and tenant-scoped through the SirmaAI Project context. Per-Project API keys constitute the tenant-boundary credential."*

---

## §SaaS B2B Platform Specific Requirements — §Integrations (whole subsection rewrite)

> ### Integrations (rewritten)
>
> *   **Data Ingestion:** Public-portal opportunity ingestion (AOP, TED, EU Grants) is delivered via SirmaAI crawler agents orchestrated by N8N workflow templates running in the shared EU Solicit-owned N8N instance. EU Solicit consumes the results via Standard Webhooks and persists normalised opportunities to canonical Postgres.
> *   **User Productivity:** Google Calendar and Outlook Calendar integrations for deadline syncing are provided for paid tiers. These remain native EU Solicit OAuth integrations (not delegated to SirmaAI).
> *   **Notifications:** Real-time notifications for team collaboration can be sent to Slack and Microsoft Teams channels (Post-MVP). Native EU Solicit integrations.
> *   **CRM (NEW SCOPE):** Bi-directional connectivity to **Microsoft Dynamics 365** and **HubSpot** is provided via SirmaAI MCP servers registered per tenant Project; agents call MCP tools during qualification and lifecycle transitions. OAuth tokens are held in EU Solicit and injected into the MCP-server config at registration. Pipedrive and Salesforce connectors are deferred to post-launch and follow the same MCP-server pattern. Tier-gated Pro+.
> *   **Enterprise API:** An enterprise-grade REST API is available for the highest tier. Exposes EU Solicit canonical data (opportunities, scores, proposals, KB metadata). Does **not** expose direct SirmaAI agent execution — agent runs are platform-internal and not part of the external contract.
> *   **External AI Substrate:** SirmaAI itself is a critical external integration. Outbound calls follow `circuit_breaker(retry(http_factory))`; inbound webhooks are HMAC-verified; run lifecycle is reconciled by polling. See NFR-26.

---

## Traceability Matrix — Amendment to Epic

| New / Amended FR-NFR | Owning Epic |
|---|---|
| FR-45, FR-46 (tenant provisioning + archival) | **E24** SirmaAI Tenant Provisioning |
| FR-49, FR-50, FR-51, FR-52 (KB lifecycle) | **E25** Knowledge Base Lifecycle |
| FR-15 (amended), FR-47, FR-48 (qualification + quantification) | **E26** Agent-Driven Ingestion & Analysis + modified **E05** |
| FR-53 (CRM via MCP) | **E27** CRM via MCP (modified **E17**) |
| FR-54, FR-55 (webhook + reconciliation) | **E28** Webhook & Run-State Reconciliation |
| FR-36 (amended ESPD via SirmaAI agent) | modified **E11** Grants & Compliance |
| NFR-2 (amended), NFR-6 (amended), NFR-24, NFR-25, NFR-26 | Cross-cutting; verified in **E04** modified + **E28** |

---

## Approval & Merge Plan

1. **Review** this amendment alongside Winston's architecture amendment + ADR-018/019/020.
2. On joint approval, edits are merged into `PRD.md`:
   - Section replacements per the OLD → NEW deltas above.
   - New subsections (§Knowledge Base) inserted at indicated positions.
   - FR / NFR numbers FR-45 through FR-55, NFR-24 through NFR-26 appended to their sections.
3. Original PRD archived to `PRD.v2.0.bak.md` (already reserved in the planning-artifacts directory).
4. Stepscompleted frontmatter in PRD.md updated to record the amendment pass: append `"step-13-sirmaai-amendment-2026-05-12"`.
5. `bmad-check-implementation-readiness` runs against the merged PRD + amended architecture + modified epics + new epics; readiness PASS unblocks orchestrator sprint replan.

---

**Author:** 📋 John (PM) — 2026-05-12
**Pairs with:** `architecture.md` amendment + ADR-018 (SirmaAI substrate, Topology A) + ADR-019 (KB ownership split) + ADR-020 (CRM via MCP) — pending from Winston.
**Status:** Draft for Deb review.
