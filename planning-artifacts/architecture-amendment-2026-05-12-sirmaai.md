# Architecture Amendment — 2026-05-12 (SirmaAI Integration Pivot)

**Source architecture:** `eusolicit-docs/planning-artifacts/architecture.md` v2.0
**Trigger:** Sprint Change Proposal `sprint-change-proposal-2026-05-12-sirmaai.md` (approved 2026-05-12)
**Paired with:** `prd-amendment-2026-05-12-sirmaai.md` (John, draft)
**Author:** 🏗️ Winston (System Architect)
**Status:** Draft for review

> Delta document. On approval (paired with the PRD amendment), sections below merge into `architecture.md`; the original moves to `architecture.v2.0.bak.md`. Section headings reference the live architecture's `##` and `###` anchors.

---

## Why this amendment

EU Solicit's AI tier was originally designed around a thin `ai-gateway` proxy wrapping KraftData logical-named agents, with Celery-driven crawlers in `data-pipeline`. That design ships on a clean separation of concerns: **EU Solicit owns orchestration; the external platform owns inference.** It works. What changes now is the surface area we delegate.

The pivot is **not** a redesign of the architecture's first principles — schema isolation, two-layer resilience, per-route Depends() gating, SSE lifecycle controls, fire-and-forget audit, event-bus discipline all remain intact. What changes is **where the agent runtime, agent definitions, knowledge bases, and workflow orchestration live**:

- Agent runtime, definitions, traces, memories, vector stores → **SirmaAI** (per-tenant Project, under one EU Solicit Org).
- Workflow orchestration → **N8N**, provisioned by SirmaAI, **shared at the Org level** (one N8N for the whole platform).
- Tenant-canonical structured data (opportunities, proposals, billing, RBAC) → **EU Solicit Postgres** (unchanged).
- Tenant-canonical unstructured artefacts (tenders, profiles, past proposals, ESPD templates, rubrics) → **SirmaAI storage-resources** (per-Project KB).

This document records three new ADRs justifying the call, one addendum to ADR-004, and the architectural-surface deltas that follow.

Trade-offs were considered, not avoided. They are surfaced inline in each ADR's *Consequences* section, not glossed.

---

## ADR-018 — SirmaAI as agentic substrate, Topology A (singleton Org, Project-per-company)

**Status:** Proposed (2026-05-12)
**Supersedes:** §3.4 framing of "AI Gateway abstraction" as the integration locus. ADR-018 narrows the gateway's responsibility from "broker every KraftData call" to "broker every SirmaAI call, hold per-tenant mapping, host webhooks, reconcile run state."

**Decision.** EU Solicit delegates its agent runtime to **SirmaAI** (at `https://agenticsai.endigitalx.com/`, the production deployment of the platform formerly known as KraftData). Tenancy is modelled as **Topology A**: a **single SirmaAI Organisation** owned by EU Solicit, with **one SirmaAI Project per EU Solicit company**. Per-Project API keys are the tenant credential boundary, stored Fernet-encrypted in `client.sirmaai_projects.api_key_encrypted`. SirmaAI-provisioned N8N runs at the Organisation level — **one shared N8N instance** for the whole platform, workflow templates parameterised by `projectId`.

**Options considered.**

| Option | Tenancy mapping | Cost profile | Isolation | N8N | Decision |
|---|---|---|---|---|---|
| **A. Singleton Org, Project-per-company** | 1 Org + N Projects | Low — single billing line | Project-scope (agents, KB, traces, memories) | Shared (org-scoped) | **Selected** |
| B. Org-per-company | N Orgs + N Projects | High — per-Org overhead | Org-scope (incl. N8N + rate-limits + audit) | Per-tenant | Rejected — cost + provisioning surface |
| C. Hybrid by tier | Free/Starter → A; Pro/Enterprise → B | Medium | Mixed | Mixed | Rejected — two code paths, tier-upgrade migration is hostile |

**Rationale.**

*Boring choice for the boring question.* Topology B is the "right" answer if money were free and SirmaAI were our crown jewel; it isn't. We're a single-team platform with a single-host on-prem launch (ADR-010). Operating N N8N instances and N billing relationships in lockstep with EU Solicit's company lifecycle is operational debt that doesn't buy what the product needs at launch. Project-level isolation in SirmaAI is genuine — agents, KB vector stores, traces, memories, eval-runs, policies, and run logs are all Project-scoped. The N8N concession is the only meaningful give.

*Per-Project API key as tenant boundary* fits the existing Fernet pattern (ADR-009 / Epic 9 — OAuth token vault). Same encryption module, same rotation cadence, same dependency-injection shape. Rule of Three holds: company secrets, OAuth tokens, SirmaAI keys — third use of the pattern is the trigger to elevate it from "vendor-specific" to "canonical Fernet vault" in the project context.

**Consequences.**

- *N8N as shared infrastructure is a known blast-radius concession.* One badly-authored workflow template can affect every tenant simultaneously. Mitigation: workflow versioning + staged rollout (canary tenants → 10% → 100%) + per-tenant feature flag gate on new template versions. **This must be a story AC in E26, not a hand-wave.** Treating workflow templates as production code (PR review, semver, rollback plan) is non-negotiable.
- *SirmaAI is now the second critical external dependency* (after Stripe). EU Solicit's effective availability becomes `min(EU Solicit, SirmaAI)` for any user flow that crosses the boundary. Mitigation: run-state reconciler is authoritative (ADR-018 §run-reconciliation), async-run + job-poll model means transient SirmaAI outages don't lose runs, graceful-degradation UX banner ("AI analysis temporarily unavailable") for outages >5 minutes.
- *Tenant provisioning becomes synchronous with company create.* Auto-create SirmaAI Project + seed KB + register MCP stubs within 30s of company create (FR-45). Failure → reconciliation path with retry; admin-API "re-provision" endpoint for manual recovery.
- *EU residency is now an ownership-split question.* EU Solicit data on www1 is EU; SirmaAI residency for the EU Solicit Organisation must be **contractually confirmed**. See §11.3 risk row update.
- *The `agents.yaml` logical-name registry retires.* SirmaAI Projects own agent identity natively. Duplicating it client-side was acceptable when the upstream was opaque; with per-Project agent records exposed via API, the registry is duplication.
- *Logical agent names are still useful for code clarity.* They survive as an **adapter pattern**: `sirmaai-gateway` resolves `("proposal_drafter", project_id)` → SirmaAI agent UUID via a per-Project agent-name index built at provisioning time. The lookup table lives in `client.sirmaai_projects.agent_map` JSONB.

---

## ADR-019 — Knowledge Base ownership split: SirmaAI canonical for unstructured artefacts, EU Solicit canonical for structured records

**Status:** Proposed (2026-05-12)

**Decision.** EU Solicit's data tier is **split by content type**:

- **EU Solicit Postgres remains canonical for structured records**: opportunities, proposals, ESPD profiles, compliance frameworks, billing, subscriptions, users, companies, workspaces, memberships, audit log.
- **SirmaAI storage-resources is canonical for unstructured artefacts**: tender PDFs, ESPD template documents, company profile documents (org charts, capability statements, certifications), past proposals (uploaded as reference), qualification rubrics.

Vector embeddings and parsed-text representations live exclusively in SirmaAI; EU Solicit holds metadata pointers (`sirmaai_file_id`, `sirmaai_storage_resource_id`, `parsed_text_available_at`) but not the artefact bodies after upload.

**Options considered.**

| Option | Where artefacts live | Search | Agent grounding | Decision |
|---|---|---|---|---|
| **A. KB canonical in SirmaAI** | SirmaAI storage-resources only | SirmaAI semantic search | Native (agents read KB in Project scope) | **Selected** |
| B. Dual write (EU Solicit S3 + SirmaAI KB) | Both | SirmaAI for semantic, S3 for raw retrieval | Native | Rejected — sync gap risk, double cost |
| C. KB canonical in EU Solicit S3 + agents pull at runtime | S3 | Custom semantic layer | Per-agent custom retriever | Rejected — re-implements what SirmaAI does natively |

**Rationale.**

*The platform that does the inference should own the index it queries.* Mirroring artefacts in EU Solicit S3 buys nothing the user notices and costs us: storage duplication, sync drift, two retention policies, two delete paths for Right-to-Erasure. SirmaAI's storage-resources gives parsed-text download (`/files/{fileId}/parsed-text/download`) and signed-URL file download — both are sufficient for any EU Solicit-side use case (re-export, audit, legal hold).

*Right-to-Erasure (GDPR Art. 17) needs an explicit cross-substrate path.* When a tenant exercises erasure: EU Solicit Postgres rows are deleted/anonymised in the existing flow; **a sibling job must call SirmaAI `DELETE /storage-resources/{id}/files/{fileId}` for every artefact in the tenant's Project**. This is FR-46-adjacent and must be a story AC in E24 archival flow.

**Consequences.**

- *No more in-house parsed-text or vector indexing for unstructured content.* Saves the build of a separate retrieval substrate — and saves the bug surface that comes with it. Boring.
- *Right-to-Erasure traverses two substrates.* New cross-substrate erasure-completion proof required in audit log: `erasure_step: postgres_rows_deleted`, `erasure_step: sirmaai_files_deleted`. Single missing step = erasure not certified.
- *Data export for tenant offboarding crosses two substrates.* Export job pulls structured records from EU Solicit Postgres + iterates SirmaAI `/storage-resources/{id}/files` to fetch artefacts and parsed text. Bundled into a single tarball. New endpoint: `POST /api/v1/companies/{id}/export-archive`.
- *Operational pain shifts to "what if SirmaAI loses a file?"* Mitigation: pre-upload SHA-256 hash recorded in `client.sirmaai_kb_files.sha256`; periodic reconciliation job verifies SirmaAI inventory against the EU Solicit hash table. Mismatch → admin alert. This is the storage-resources analogue of the run-state reconciler.
- *Local-dev story is uglier.* `make up` cannot stand up a fake SirmaAI cheaply. Two options: (1) point local dev at SirmaAI staging (`stage.sirma.ai`) with a shared dev Org; (2) write a `sirmaai-mock` thin FastAPI that implements the dozen endpoints we care about with in-memory state. Recommendation: **(1) for early E24/E25/E26 work, (2) later when test-isolation pain forces it**. Don't pre-build the mock.

---

## ADR-020 — CRM integration via SirmaAI MCP servers (Dynamics 365 + HubSpot v1; Pipedrive + Salesforce deferred)

**Status:** Proposed (2026-05-12)
**Supersedes:** §3.4 and ADR-009's CRM-related scope (HubSpot → Pipedrive → Salesforce as direct adapters in `integrations-api`). ADR-009 itself **remains accepted** for Slack/Teams + `integrations` schema + OAuth callback hosting; CRM-specific direct-adapter language is retired by this ADR.

**Decision.** CRM connectivity in v1 ships as **two MCP servers per SirmaAI Project**: **Microsoft Dynamics 365** and **HubSpot**. Each MCP server exposes a stable tool surface — `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note` — callable by SirmaAI agents during qualification, lifecycle transitions, and on-demand enrichment. EU Solicit hosts the OAuth callback and stores access + refresh tokens Fernet-encrypted in `client.crm_connections` (existing table). Tokens are injected into the MCP-server configuration **at registration time** (and on rotation) via SirmaAI's `secrets` API; SirmaAI's secrets store becomes a downstream extension of EU Solicit's trust boundary for these credentials.

**Pipedrive** and **Salesforce** are deferred to post-launch as additional MCP-server registrations following the same pattern (no architectural change required).

**Options considered.**

| Option | Where CRM HTTP lives | Agent integration | Per-provider rate-limit state | Decision |
|---|---|---|---|---|
| **A. MCP server per provider, per Project** | SirmaAI MCP server | Native (agent invokes MCP tool) | SirmaAI-side | **Selected** |
| B. Direct adapter in `integrations-api` (the v1 plan) | EU Solicit `integrations-api` | Indirect — agent → ai-gateway → integrations-api | EU Solicit-side, per-provider Celery queues | Rejected — agents can't enrich in-flight |
| C. N8N HTTP-request nodes in workflows | N8N (org-shared) | Workflow-step only | N8N-side | Rejected — agents need tool access, not workflow steps; credential isolation is harder in shared N8N |

**Rationale.**

*Agents need to enrich leads during qualification, not after it.* In Option B, qualification → workflow ends → separate enrichment step → workflow restarts → re-qualification with enriched data. Round-trip + state machinery + sync drift. In Option A, the qualification agent calls `enrich_contact` mid-run, gets richer context, and folds it into the same output. The bid/no-bid recommendation comes out the other side already-enriched.

*Pipedrive + Salesforce deferral is a v1 scope call, not a permanent architectural cut.* The previous E17 plan correctly identified Salesforce as a deal-blocker for the Loopio-shaped competitive set. With on-prem launch ADR-010 in beta posture and the 8–10 week launch slip from the pivot, Salesforce is a Phase 2 add — same pattern, additional MCP server registration, no contract change. **Dynamics 365 is net-new** to the plan and reflects the rebrand opportunity for the v1 enterprise pursuit list.

**Consequences.**

- *OAuth tokens leave EU Solicit's Fernet vault into SirmaAI's secrets store at registration.* This is a deliberate trust-boundary expansion: EU Solicit remains the custodian (rotation, revocation, audit) but SirmaAI gets a copy needed for MCP-server runtime. Rotation must be **double-sided**: rotate in EU Solicit's vault → push to SirmaAI secrets → verify reachability → revoke old token. Per FR-53.
- *Conflict resolution remains Last-Write-Wins (LWW) with audit.* Repurposes the existing `integrations.conflict_log` table; conflict source changes from "EU Solicit direct adapter vs CRM" to "SirmaAI MCP tool call vs CRM webhook reflection." Same shape.
- *Per-provider rate limits now belong to SirmaAI.* EU Solicit's existing per-provider circuit-breaker + Celery-queue machinery for CRM (`integrations-api` S17.00) **retires for these two providers**. The two-layer resilience pattern (ADR-004) applies to the EU Solicit → SirmaAI call only.
- *Audit trail spans two substrates.* MCP-tool invocations are traced inside SirmaAI (per-Project traces); EU Solicit's `shared.audit_log` records the trigger (`agent_run_initiated`, `mcp_tool_invoked_via_agent`) and the user-visible outcome. Both are retrievable.
- *Slack / Teams remain native EU Solicit integrations* (per ADR-009, unchanged). They are notification surfaces, not agent-callable tools. If we ever expose them as agent tools, that's a separate MCP-server addition — same pattern.

---

## ADR-004 addendum

**Append to ADR-004 *Decision* paragraph:**

> *Addendum 2026-05-12 (per ADR-018):* Outbound calls to **SirmaAI** (replacing prior "KraftData" framing) use the same two-layer composition. The pattern is unchanged. Logical-name keys for circuit-breaker buckets shift from agent-name-from-yaml-registry (retired) to a composite of `(eusolicit_logical_name, sirmaai_project_id)` resolved at call time by `sirmaai-gateway`. Open-circuit fallback for AI paths remains **fail-open with degraded result + tenant-visible banner** for outages >5 minutes; payment-path circuit breakers (Stripe) remain **fail-closed**.

---

## §1.1 Architectural Style (amendment)

**Bullet 3 (whole-bullet replace):**

> **OLD:** *"Human-in-the-loop AI orchestration: an AI Gateway brokers all calls to KraftData's Agentic AI platform (multi-agent system); the user always reviews/approves AI output."*
>
> **NEW:** *"Human-in-the-loop AI orchestration with **SirmaAI as the agentic substrate**: a per-tenant SirmaAI Project owns agent definitions, runs, traces, memories, and the knowledge base; **`sirmaai-gateway`** (renamed from `ai-gateway`) brokers EU Solicit ↔ SirmaAI calls, holds tenant↔Project mapping, hosts the Standard Webhooks receiver, and reconciles run state. **N8N** (shared org-scoped, provisioned by SirmaAI for the EU Solicit Organisation) orchestrates multi-step workflows with agents as native steps. The user always reviews/approves AI output."*

**Insert new bullet after bullet 4:**

> *"**Two-substrate canonical-data model**: structured records (opportunities, proposals, billing, RBAC) canonical in EU Solicit Postgres; unstructured artefacts (tenders, profiles, past proposals, ESPD templates, rubrics) canonical in SirmaAI storage-resources per tenant Project. Tenant operations (provisioning, archive, Right-to-Erasure, export) traverse both substrates with explicit cross-substrate completion proof in `shared.audit_log` (ADR-019)."*

**Bullet 6 (Resilient outbound integration) — no change.** ADR-004 addendum covers the SirmaAI scope.

---

## §1.2 Architectural Drivers (amendment)

**Replace the row "EU-only data residency (Domain-Specific)" `Implication` cell:**

> **OLD:** *"AWS eu-central-1 single-region; KraftData EU-region storage resources only"*
>
> **NEW:** *"On-prem single-host `www1.endigitalx.com` (per ADR-010); SirmaAI EU-region storage resources only for the EU Solicit Organisation — **contractually confirmed** as launch-blocking due-diligence (see §11.3 risk #1, post-amendment)"*

---

## §2.1 High-Level Topology (whole ASCII-diagram replacement)

**Replace the diagram block (lines 66-157) with:**

```
                                            ┌──────────────────────────┐
                                            │       INTERNET            │
                                            └─┬────────┬────────┬───────┘
                                              │        │        │
                                          (CloudFlare CDN — Trust Center only)
                                              │        │        │
                              ┌───────────────┼────────┼────────┼───────────┐
                              │               │        │        │           │
                              ▼               ▼        ▼        ▼           ▼
                        ┌─────────────┐  ┌─────────┐  ┌────────────┐  ┌────────────┐
                        │ apps/client │  │apps/    │  │ /trust     │  │  Stripe    │
                        │ Next.js 14  │  │admin    │  │ (public    │  │  webhooks  │
                        │ :3000       │  │Next.js  │  │  static)   │  │ (HMAC)     │
                        └──────┬──────┘  │:3001    │  └────────────┘  └─────┬──────┘
                               │         │ (VPN +  │                        │
                               │         │ IP-AL)  │                        │
                               │         └────┬────┘                        │
                               │              │                             │
                  ┌────────────┴──────────────────────────────────┐         │
                  │      host nginx on www1 (TLS 1.3 + certbot)   │         │
                  └──┬────────┬─────────┬────────┬────────┬───────┘         │
                     │        │         │        │        │                 │
                     ▼        ▼         ▼        ▼        ▼                 │
              ┌──────────┐┌────────┐┌───────────┐┌──────┐┌──────────┐      │
              │client-api││admin-  ││sirmaai-   ││data- ││integ-    │      │
              │  :8001   ││api     ││gateway    ││pipe- ││rations-  │◄─────┤
              │ FastAPI  ││:8002   ││:8004      ││line  ││api :8007 │      │
              │ + Stripe │└────────┘│ (renamed) ││:8003 │└────┬─────┘      │
              │ + RBAC   │          │ tenant→   ││(thin,│     │            │
              │ + Tier   │          │  Project  ││ recvr│     │            │
              │ + tenant │          │  mapping  ││ only)│     │            │
              │  provis. │          │  + WH recv│└──┬───┘     │            │
              └────┬─────┘          │  + reconc.│   │         │            │
                   │                └─────┬─────┘   │         │            │
                   │                      │         │         │            │
                   │                      ▼         ▼         ▼            │
                   │       ╔═══════════════════════════════════════════╗  │
                   │       ║              SirmaAI (EXTERNAL)            ║  │
                   │       ║      https://agenticsai.endigitalx.com/   ║  │
                   │       ║                                            ║  │
                   │       ║  ┌──────────────────────────────────────┐ ║  │
                   │       ║  │ EU Solicit Organisation (singleton)  │ ║  │
                   │       ║  │ ┌──────────────────────────────────┐ │ ║  │
                   │       ║  │ │ Project per EU Solicit company   │ │ ║  │
                   │       ║  │ │  - Agents (ESPD, qual, quant…)   │ │ ║  │
                   │       ║  │ │  - Teams + Workflows             │ │ ║  │
                   │       ║  │ │  - Storage-resources (KB) ◄──────┼─┼─║──┐ artefact uploads
                   │       ║  │ │  - MCP servers (Dynamics,HubSpot)│ │ ║  │ from client-api
                   │       ║  │ │  - Traces, memories, eval-runs   │ │ ║  │
                   │       ║  │ │  - Per-Project api-key (Fernet)  │ │ ║  │
                   │       ║  │ └──────────────────────────────────┘ │ ║  │
                   │       ║  │  ┌────────────────────────────────┐  │ ║  │
                   │       ║  │  │ Shared N8N (org-scoped)        │  │ ║  │
                   │       ║  │  │  Workflow templates by         │  │ ║  │
                   │       ║  │  │  projectId — AOP/TED/EUGrants/ │  │ ║  │
                   │       ║  │  │  Qualification/Quantification/ │  │ ║  │
                   │       ║  │  │  CRM-enrichment                │  │ ║  │
                   │       ║  │  └────────────────────────────────┘  │ ║  │
                   │       ║  └──────────────────────────────────────┘ ║  │
                   │       ║                                            ║  │
                   │       ║  Webhooks (Standard Webhooks spec):       ║  │
                   │       ║   workflow.completed                       ║  │
                   │       ║   agent.run.completed                      ║──┴──► HTTPS in to
                   │       ║   storage.file.processed                   ║      sirmaai-gateway
                   │       ║   policy.violation                         ║      /webhooks/sirmaai
                   │       ╚═══════════════════════════════════════════╝
                   │
                   │                                  ┌────────────┐
                   │                                  │ Microsoft  │
                   │                                  │ Dynamics + │◄── via MCP from
                   │                                  │ HubSpot    │    SirmaAI agents
                   │                                  │ (CRM)      │    (Pipedrive/SF
                   │                                  └────────────┘     deferred)
                   │
                   ▼
       ┌──────────────────────┐         ┌──────────────────────────┐
       │  notification :8005  │◄────────┤ Redis Streams            │
       │  Email / Slack /     │         │ (events)                 │
       │  Teams / SP-change / │         │ + DLQ                    │
       │  onboarding alerts   │         └──────────┬───────────────┘
       └──────────┬───────────┘                    │
                  │                                │
                  │   ┌────────────────────────────┴─────────────────┐
                  │   │              Redis 7 (single container)       │◄──┘
                  │   │  - Caching (tier policy, RBAC, session)        │
                  │   │  - Rate limiting + atomic usage Lua            │
                  │   │  - Event Bus (Streams) + DLQ                   │
                  │   │  - Idempotency keys (SETNX) — webhook dedup    │
                  │   └────────────────────────────────────────────────┘
                  │
                  ▼
        ┌─────────────────────────────────────────────────────────────────┐
        │   PostgreSQL 16 (single container on www1, daily Hetzner BU)    │
        │  Schemas (logical isolation; cross-schema queries forbidden):   │
        │  ┌──────────┬──────────┬──────────┬─────────┬────────────────┐  │
        │  │  client  │  admin   │ pipeline │ gateway │  notification  │  │
        │  ├──────────┴──────────┴──────────┴─────────┴────────────────┤  │
        │  │  integrations                                              │  │
        │  ├───────────────────────────────────────────────────────────┤  │
        │  │  shared (audit_log, lookups)                               │  │
        │  └───────────────────────────────────────────────────────────┘  │
        │  NEW (per this amendment):                                      │
        │   client.sirmaai_projects        (tenant↔Project mapping)       │
        │   client.sirmaai_kb_files        (artefact metadata + SHA256)   │
        │   client.sirmaai_mcp_servers     (MCP registration state)       │
        │   gateway.webhook_subscriptions  (Standard Webhooks subs)       │
        │   gateway.workflow_runs          (run lifecycle + reconciler)   │
        └─────────────────────────────────────────────────────────────────┘

        ┌─────────────────────────────────────────────────────────────────┐
        │   Local object storage on www1 (Trust Center artefacts)         │
        │   - Tender raw downloads cached briefly before upload to        │
        │     SirmaAI storage-resources                                   │
        │   - Trust Center artefacts (signed URLs)                        │
        │   NOTE: S3/MinIO no longer canonical for tender/proposal        │
        │   content (per ADR-019 — SirmaAI KB is canonical)               │
        └─────────────────────────────────────────────────────────────────┘
```

**Rationale.** Three changes vs. the prior diagram: (1) `ai-gateway` renamed to `sirmaai-gateway` with explicit responsibilities for tenant mapping, webhook receipt, reconciliation; (2) the SirmaAI box is now an explicit external-platform region with its own internal structure (Org / Projects / N8N / webhooks); (3) the EU Solicit data tier shows the new tables this amendment introduces. AWS-specific elements (ALB, CloudFront, RDS Multi-AZ, ElastiCache) are gone — already replaced by host nginx + Docker containers per ADR-010, but the prior diagram hadn't been updated.

---

## §2.2 Service Inventory (table amendments)

**Replace `ai-gateway` row:**

> | `sirmaai-gateway` (renamed from `ai-gateway`) | 8004 | `gateway` (rw) | Single broker for SirmaAI. Tenant↔Project mapping cache (Fernet-encrypted per-Project api-keys), Standard Webhooks receiver, async-run + job-poll, run-status reconciler (5-min cadence), SSE proxy for streaming agent runs. **Retires:** `agents.yaml` logical-name registry, hard-coded KraftData routes |

**Replace `data-pipeline` row:**

> | `data-pipeline` | 8003 | `pipeline` (rw) | **Re-scoped.** Webhook event router + opportunity normaliser + canonical `pipeline.opportunities` writer. **Retires:** Celery Beat crawlers (AOP/TED/EUGrants), per-pipeline `ai_gateway_client` retry module. Crawler responsibility moves to N8N workflow templates calling SirmaAI crawler agents. Retains: `crawler_runs` history table (read-only), `opportunities.ingested` Redis Streams publication |

**Replace `integrations-api` row:**

> | `integrations-api` | 8007 | `integrations` (rw), `client.crm_connections` (read-only) | **Re-scoped.** OAuth callback hosting for Dynamics 365 + HubSpot; Fernet token storage in `client.crm_connections`; MCP-server registration + token-injection to SirmaAI; token rotation Celery Beat. Slack/Teams incoming-webhook templates **retained**. **Retires:** HubSpot/Pipedrive/Salesforce direct HTTP adapters, per-provider sync Celery queues for CRM (Slack/Teams Celery queues retained) |

**Backend shared packages — replace `eusolicit-kraftdata` row:**

> | `eusolicit-sirmaai` (renamed from `eusolicit-kraftdata`) | Typed Python client for SirmaAI Org/Project/Agent/Team/Workflow/Storage/MCP/Webhook APIs. Generated from `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json`. Pins SirmaAI base URL via env (`SIRMAAI_BASE_URL`). **Boring-tech directive:** keep this a thin generated client + a handful of hand-written convenience wrappers — no business logic in the client package |

---

## §3.4 AI Platform (whole-subsection rewrite)

> ### 3.4 AI Platform (rewritten)
>
> | Component | Choice | Rationale |
> |---|---|---|
> | **SirmaAI agentic substrate** | external managed at `https://agenticsai.endigitalx.com/` | Multi-agent runtime (per-Project): agents, teams, workflows, vector storage-resources (KB), traces, memories, eval-runs, policies. Per ADR-018 |
> | **Tenancy mapping** | Singleton EU Solicit Org + Project-per-company | Topology A, ADR-018. `client.sirmaai_projects` table holds (`company_id ↔ sirmaai_project_id`, Fernet-encrypted api-key, n8n-subdomain, agent_map JSONB) |
> | **`sirmaai-gateway` abstraction** | in-house FastAPI service | Single broker per ADR-018: tenant↔Project mapping cache, per-Project api-key vault, Standard Webhooks receiver, run-state reconciler, SSE proxy. Logical agent names resolved per `(eusolicit_logical_name, sirmaai_project_id)` from `agent_map` |
> | **Calling conventions** | `SirmaaiGatewayClient.run_agent(logical_name, payload, company_id)` (sync), `run_agent_stream(...)` (SSE), `submit_agent_run_async(...)` + `get_job_status(job_id)` (async-poll) | Frozen contract. Flat 503 body `{"message", "code": "AGENT_UNAVAILABLE"}` retained (Epic 11 standard). `X-Caller-Service` header retained. **No client-side retry**, gateway owns retry/backoff per ADR-004 |
> | **N8N orchestration** | shared org-scoped (one instance for EU Solicit Org) | Workflow templates parameterised by `projectId`. Per-template versioning (semver) + staged rollout (canary tenant → 10% → 100%) per ADR-018 *Consequences*. Templates are production code: PR review + rollback plan |
> | **Knowledge Base** | SirmaAI storage-resources per Project | Per ADR-019. Artefacts canonical in SirmaAI; EU Solicit holds `(sirmaai_storage_resource_id, sirmaai_file_id, sha256)` metadata only |
> | **CRM tooling** | SirmaAI MCP servers per Project (Dynamics + HubSpot v1) | Per ADR-020. OAuth tokens custodied in EU Solicit Fernet vault, injected into MCP-server config at registration + rotation |
> | **Webhooks (SirmaAI → EU Solicit)** | Standard Webhooks specification | Subscribed event types: `workflow.completed`, `agent.run.completed`, `storage.file.processed`, `policy.violation`. HMAC SHA-256 verified via `hmac.compare_digest()` (existing canonical pattern). 7-day Redis idempotency cache. DLQ for poison events |
> | **Run-state reconciliation** | scheduled job in `sirmaai-gateway`, 5-min cadence | Polls `GET /jobs/{jobId}/status` for non-terminal rows in `gateway.workflow_runs` and converges to terminal. **Reconciler is authoritative**; webhooks are latency optimisation. Decision rationale: SirmaAI is the second critical external dependency (after Stripe) — we cannot accept silent run loss |
> | **Streaming (SSE)** | unchanged per ADR-005 | SSE lifecycle invariants (quota check before `StreamingResponse`, generator closed in finally, terminal event, fresh `session_factory`, single-point error sanitization, `except asyncio.CancelledError: raise`) all retained. Upstream URL shifts to SirmaAI `/agents/{id}/run/stream` |

**Rationale.** Reframes the section around SirmaAI as the substrate (vs. KraftData as a dependency); makes tenant↔Project mapping a first-class concern; pins the async-run + job-poll model as the long-running-analysis path (qualification, quantification — FR-47, FR-48); records the N8N staged-rollout discipline as architecture-level (not buried in epic ACs).

---

## §4.1 Schema Layout (additions)

**Append new tables to the existing `client` and `gateway` schemas:**

```sql
-- =====================================================================
-- SIRMAAI TENANT MAPPING (per ADR-018)
-- =====================================================================

CREATE TABLE client.sirmaai_projects (
    id                    UUID PRIMARY KEY,
    company_id            UUID NOT NULL UNIQUE REFERENCES client.companies(id),
    sirmaai_org_id        TEXT NOT NULL,                    -- the singleton EU Solicit Org
    sirmaai_project_id    TEXT NOT NULL UNIQUE,
    api_key_encrypted     BYTEA NOT NULL,                   -- Fernet, Epic 9 canonical module
    api_key_rotated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    n8n_subdomain         TEXT,                              -- nullable; org-shared but project-routed
    agent_map             JSONB NOT NULL DEFAULT '{}',      -- {logical_name: sirmaai_agent_uuid}
    provisioning_status   TEXT NOT NULL DEFAULT 'pending',  -- pending | provisioned | failed | archived
    provisioning_error    TEXT,                              -- last error if failed
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    archived_at           TIMESTAMPTZ
);
CREATE INDEX ix_sirmaai_projects_status ON client.sirmaai_projects(provisioning_status)
    WHERE provisioning_status != 'provisioned';   -- partial: hot rows only

-- =====================================================================
-- KB ARTEFACT METADATA (per ADR-019 — bodies live in SirmaAI)
-- =====================================================================

CREATE TABLE client.sirmaai_kb_files (
    id                          UUID PRIMARY KEY,
    company_id                  UUID NOT NULL REFERENCES client.companies(id),
    sirmaai_storage_resource_id TEXT NOT NULL,
    sirmaai_file_id             TEXT NOT NULL UNIQUE,
    filename                    TEXT NOT NULL,
    content_type                TEXT NOT NULL,
    size_bytes                  BIGINT NOT NULL,
    sha256                      TEXT NOT NULL,                   -- for reconciliation
    artefact_category           TEXT NOT NULL,                   -- tender|profile|proposal|espd_template|rubric|other
    parsed_text_available_at    TIMESTAMPTZ,                     -- nullable; set when SirmaAI completes processing
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    archived_at                 TIMESTAMPTZ
);
CREATE INDEX ix_sirmaai_kb_files_company_category
    ON client.sirmaai_kb_files (company_id, artefact_category)
    WHERE archived_at IS NULL;

-- =====================================================================
-- CRM MCP REGISTRATION (per ADR-020)
-- =====================================================================

CREATE TABLE client.sirmaai_mcp_servers (
    id                          UUID PRIMARY KEY,
    company_id                  UUID NOT NULL REFERENCES client.companies(id),
    provider                    TEXT NOT NULL,                  -- dynamics365|hubspot (v1); pipedrive|salesforce post-launch
    sirmaai_mcp_server_id       TEXT NOT NULL UNIQUE,
    crm_connection_id           UUID REFERENCES client.crm_connections(id),  -- token source
    status                      TEXT NOT NULL DEFAULT 'inactive',  -- inactive|registered|error
    last_token_pushed_at        TIMESTAMPTZ,
    last_error                  TEXT,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (company_id, provider)
);

-- =====================================================================
-- WEBHOOK SUBSCRIPTIONS + RUN-STATE RECONCILIATION (per ADR-018)
-- =====================================================================

CREATE TABLE gateway.webhook_subscriptions (
    id                          UUID PRIMARY KEY,
    sirmaai_subscription_id     TEXT NOT NULL UNIQUE,
    event_types                 TEXT[] NOT NULL,                -- subscribed Standard Webhooks event types
    hmac_secret_encrypted       BYTEA NOT NULL,                 -- Fernet
    hmac_rotated_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE gateway.workflow_runs (
    id                          UUID PRIMARY KEY,
    company_id                  UUID NOT NULL REFERENCES client.companies(id),
    eusolicit_run_id            UUID NOT NULL UNIQUE,           -- our correlation id
    sirmaai_run_id              TEXT,                            -- null until SirmaAI assigns
    sirmaai_job_id              TEXT,                            -- async-run job id, if applicable
    run_type                    TEXT NOT NULL,                  -- agent|team|workflow
    status                      TEXT NOT NULL,                  -- pending|running|completed|failed|cancelled
    started_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at                TIMESTAMPTZ,
    last_polled_at              TIMESTAMPTZ,                    -- for reconciler
    error_message               TEXT,
    payload_excerpt             JSONB                            -- redacted; for forensics
);
CREATE INDEX ix_workflow_runs_nonterminal
    ON gateway.workflow_runs(last_polled_at)
    WHERE status IN ('pending','running');  -- partial: reconciler scan target
```

**Rationale.**

- `client.sirmaai_projects.agent_map JSONB`: the **adapter pattern** mentioned in ADR-018. Logical names survive in EU Solicit code (`run_agent("proposal_drafter", ...)`); per-tenant resolution at call time. Cheap, debuggable.
- Partial indexes on `provisioning_status != 'provisioned'` and `status IN ('pending','running')`: hot rows only. The reconciler scans this index every 5 minutes; full-table scans on `workflow_runs` would be a real cost at scale.
- `gateway.workflow_runs.payload_excerpt JSONB`: redacted, **bounded size** (e.g. 16KB) — for forensics on failed runs. Full payloads live in SirmaAI's traces; we keep a breadcrumb.
- No FKs from `client.sirmaai_projects` or `gateway.workflow_runs` to anything in `pipeline` or `gateway` schemas beyond `client.companies` — schema isolation invariant (ADR-001) preserved.

---

## §4.4 Key Data Patterns (additions)

**Append the following bullets:**

- **Per-Project SirmaAI API key as tenant credential boundary** (ADR-018): Fernet-encrypted at rest in `client.sirmaai_projects.api_key_encrypted`. Rotation on 90-day cadence with overlap: new key issued → smoke test → old key revoked. Rotation event publishes `sirmaai.key_rotated` to Redis Streams for downstream observability.
- **Per-Project agent-name resolution** (ADR-018 *Consequences*): `agent_map JSONB` on `client.sirmaai_projects`. Logical-name lookup at call time; missing key triggers re-sync against SirmaAI Project agent inventory before falling back to 503.
- **KB artefact integrity via SHA-256 reconciliation** (ADR-019): `client.sirmaai_kb_files.sha256` recorded at upload; nightly reconciliation against SirmaAI inventory; mismatches → admin alert + audit entry.
- **Cross-substrate Right-to-Erasure** (ADR-019): erasure flow has two ACK steps in `shared.audit_log` (`erasure_step: postgres_rows_deleted`, `erasure_step: sirmaai_files_deleted`); erasure not certified complete until both present.
- **Workflow-run reconciliation as authoritative** (ADR-018 *Consequences*): `gateway.workflow_runs.status` is converged by the 5-min reconciler polling `GET /jobs/{jobId}/status`; webhooks are latency optimisations and may be lost without correctness impact. Test design must include `webhook_dropped, reconciler_recovers` scenario.

---

## §5.1 External APIs (table amendment)

**Replace the row:**

> | AOP / TED / KraftData / Stripe / VIES / Google OAuth / HubSpot / Salesforce / Pipedrive / Slack / MS Teams | Outbound | Provider-specific | All wrapped in `circuit_breaker(retry(http_factory))` with explicit `httpx` timeout |

**With two rows:**

> | **SirmaAI** (`https://agenticsai.endigitalx.com/`) | Outbound + inbound webhooks | Per-Project api-key (Bearer) outbound; HMAC SHA-256 inbound (Standard Webhooks spec) | All outbound wrapped in `circuit_breaker(retry(http_factory))` with explicit `httpx` timeout. Logical-name circuit-breaker keys = `(eusolicit_logical_name, sirmaai_project_id)`. Inbound webhooks: signature verified via `hmac.compare_digest()`, 7-day idempotency cache, DLQ for poison events. Per ADR-018 |
> | AOP / TED / Stripe / VIES / Google OAuth / Microsoft Dynamics / HubSpot / Slack / MS Teams (Pipedrive/Salesforce deferred) | Outbound | Provider-specific | All wrapped in `circuit_breaker(retry(http_factory))`. Dynamics + HubSpot are invoked via SirmaAI MCP servers (per ADR-020); EU Solicit's direct HTTP only used by OAuth-callback flow + token-rotation push to SirmaAI secrets |

---

## §5.3 Event Catalog (additions)

**Append inbound webhook event types (SirmaAI → EU Solicit), mapped to internal Redis Streams:**

- SirmaAI `workflow.completed` → internal `sirmaai.workflow.completed` → `data-pipeline` (opportunity ingestion completion handler), `client-api` (user-facing notification)
- SirmaAI `agent.run.completed` → internal `sirmaai.agent.run.completed` → `gateway` (workflow_runs status converge), `notification` (user-facing run-completion alerts)
- SirmaAI `storage.file.processed` → internal `sirmaai.kb.file.processed` → `client-api` (set `client.sirmaai_kb_files.parsed_text_available_at`)
- SirmaAI `policy.violation` → internal `sirmaai.policy.violation` → `notification` (admin alert), `shared.audit_log` (immutable record)

**Append new internal Redis Streams events (EU Solicit → EU Solicit consumers):**

- `tenant.provisioned` (published by `client-api` after E24 S24.02 completes) → `notification` (welcome email enriched with KB-upload CTA), `client-api` (UI badge update for provisioning completion). Closes readiness Concern #5.
- `sirmaai.key_rotated` (published by `sirmaai-gateway` after E04 S04.22 rotation) → `sirmaai-gateway` mapping cache invalidation, `notification` (admin audit notification on rotation).

**Append to *Event handler discipline*:**

- *SirmaAI-origin events must be idempotent against the reconciler* — both webhook and reconciler may converge the same `workflow_runs` row. Use `UPDATE ... WHERE status IN ('pending','running')` guards rather than blind state writes.

---

## §3.4 AI Platform addendum — N8N workflow template source-of-truth (per ADR-018 *Consequences*)

**Decision (2026-05-12, closes readiness Concern #7):** N8N workflow templates are **canonical in EU Solicit's Git repo** at `eusolicit-app/infra/n8n-templates/<workflow-name>-v<semver>.json` (exported JSON from SirmaAI N8N). Templates are versioned (semver-tagged) and PR-reviewed like production code. Deploy mechanism: a deploy-time script (`scripts/sync_n8n_templates.py`) diffs committed JSON against SirmaAI N8N instance via the SirmaAI N8N API and applies updates; refuses to deploy on drift unless `--force` flag is set (with audit log). Rollback: revert the commit + re-run the script to restore the previous template version. Per-tenant feature flag (`enable_n8n_workflow_<source>_<version>`) — per E05 S05.24 — gates which version a tenant runs against. **Owning epic:** E05 amendment story to be amended to include the deploy-mechanism scope.

---

## §8.1 Multi-Tenancy (amendment)

**Append after step 5:**

> 6. **SirmaAI Project scope** propagates as a sixth tenant-context dimension: `sirmaai-gateway` resolves `(company_id) → sirmaai_project_id + api_key` from the `client.sirmaai_projects` cache (Redis-backed, 5-min TTL, invalidated on key rotation event). Every outbound SirmaAI call carries the per-Project bearer token; **a request with a mismatched Project token is a server-side bug, not a tenant-isolation violation** — the api-key is the tenant boundary at the SirmaAI side.
> 7. **KB artefact upload scope**: artefact uploads from `client-api` to SirmaAI `storage-resources` use the calling company's Project api-key. `client.sirmaai_kb_files` rows carry `company_id` and are scoped by the existing RBAC `Depends()` factories (Epic 2 pattern, unchanged).
> 8. **MCP-tool invocations** carry the Project scope implicitly — agents run inside the Project; MCP tools the agent calls execute against that Project's MCP-server config (which holds *this* tenant's OAuth tokens, not a sibling tenant's). Cross-tenant negative tests for MCP invocations are a story AC in E27.

---

## §11.3 Risks (amendments)

**Update existing row #1 (KraftData → SirmaAI):**

> | 1 | **SirmaAI EU data residency / GDPR sub-processor evidence** | **Launch-blocking**: contractually confirm EU-only data residency for the EU Solicit Organisation, including storage-resources, traces, and N8N execution. Add SirmaAI to the sub-processor list with the confirmed residency posture. ADR-010 on-prem pivot's GDPR rationale is wasted if SirmaAI residency cannot be confirmed |

**Append new rows:**

> | 11 | **N8N org-scope blast radius** (per ADR-018) | One badly-authored workflow template can affect every tenant. Mitigation: workflow versioning (semver) + canary-tenant + 10% + 100% staged rollout + per-tenant feature flag gate on new template versions. **Story AC in E26**, not hand-wave |
> | 12 | **SirmaAI as second critical external dependency** | Effective availability = `min(EU Solicit, SirmaAI)`. Mitigations: run-state reconciler is authoritative; async-run + job-poll preserves runs across transient outages; tenant-visible degraded-mode banner on outages >5 minutes; AI summary paths fail-open with degraded result, payment paths remain fail-closed |
> | 13 | **SirmaAI secrets-store as extended trust boundary** (per ADR-020) | OAuth tokens for Dynamics + HubSpot leave EU Solicit's Fernet vault into SirmaAI secrets at MCP registration. Mitigations: rotation is double-sided (EU Solicit vault → SirmaAI push → verify → old-key revoke); audit log records both sides; periodic verification job (weekly) confirms SirmaAI MCP-server config reachability |
> | 14 | **Cross-substrate Right-to-Erasure completion** (per ADR-019) | Erasure spans EU Solicit Postgres + SirmaAI storage-resources. Risk: partial erasure → compliance breach. Mitigation: two-ACK audit-log pattern (`erasure_step: postgres_rows_deleted`, `erasure_step: sirmaai_files_deleted`); erasure not certified until both present; nightly job sweeps for stale erasure rows missing the second ACK |
> | 15 | **Tenant provisioning failure modes** (per ADR-018, FR-45) | SirmaAI unavailable at company create → `provisioning_status='pending'` with retry. Bounded retry window (24h); after that, admin-flagged for manual recovery. UX: trial-tier signups can land in `pending` and degrade gracefully (no AI features until provisioned), paid signups must block at billing until provisioning succeeds |

**Append a note to §11.1 *Architecture validation*:**

> **2026-05-12 SirmaAI pivot pass:** Re-validated coherence and requirements coverage against the post-pivot architecture. PRD amendment (`prd-amendment-2026-05-12-sirmaai.md`) introduces FR-45 through FR-55 and NFR-24 through NFR-26; each maps to a service/epic/ADR per the traceability matrix in that document. Architecture readiness for the post-pivot scope is **DEFERRED PENDING `bmad-check-implementation-readiness`** against the modified E04/E05/E11/E17 + new E24-E28 epic set. No retreat from the underlying invariants (schema isolation, two-layer resilience, per-route Depends(), SSE lifecycle, fire-and-forget audit, event-bus discipline) — the pivot reshapes the upstream surface, not the platform's spine.

---

## ADR Index update (architecture.md §7 header)

When merged, the ADR list grows from ADR-001..ADR-017 to ADR-001..ADR-020. ADR-010 keeps its superseded-historical note unchanged. ADR-004 carries the 2026-05-12 addendum.

---

## Diagram & cross-document consistency check

- `prd-amendment-2026-05-12-sirmaai.md` FR-45 ↔ ADR-018 + `client.sirmaai_projects` table ✓
- PRD FR-49..FR-52 (KB) ↔ ADR-019 + `client.sirmaai_kb_files` table ✓
- PRD FR-53 (CRM via MCP) ↔ ADR-020 + `client.sirmaai_mcp_servers` table ✓
- PRD FR-54..FR-55 (webhooks + reconciler) ↔ ADR-018 *run-state reconciliation* + `gateway.webhook_subscriptions` + `gateway.workflow_runs` tables ✓
- PRD NFR-24 (key rotation) ↔ §4.4 new pattern + `api_key_rotated_at` column ✓
- PRD NFR-25 (tier→rate-limit) ↔ §3.4 calling-convention row (admin-API extension) — implementation detail; no architectural change beyond what `sirmaai-gateway` already brokers ✓
- PRD NFR-26 (SirmaAI as external dep) ↔ §11.3 risk #12 ✓

---

## Implementation handoff

**Routing back to John:** ADRs + architecture amendments ready. Pairs cleanly with the PRD amendment. Next step is **epic delta** (modify E04/E05/E11/E17; author E24-E28). Either John drives it directly or hands to Amelia via `/bmad-agent-dev`. Once epic files are aligned, `bmad-check-implementation-readiness` becomes the final gate before orchestrator sprint replan.

**One operational thing for Deb to action while the epic delta is drafted:**

1. **Get the SirmaAI EU residency answer in writing.** §11.3 risk #1 is launch-blocking and outside any agent's control. Without it, ADR-010 is half a decision.
2. **Get the SirmaAI per-Org / per-Project idle cost number.** ADR-018 assumes Topology A is cost-defensible at free-tier scale. If a dormant Project costs more than ~€2/month, free-tier acquisition economics need a model. This isn't an ADR-blocker but it is a "before we onboard our first 100 free tenants" question.

---

**Author:** 🏗️ Winston (System Architect) — 2026-05-12
**Status:** Draft for Deb review.
**Pairs with:** `prd-amendment-2026-05-12-sirmaai.md` (John). Both merge into live PRD/architecture on joint approval.
