# E24: AgenticSAI Tenant Provisioning

**Sprint:** post-pivot S+1 (first sprint after E04 amendment lands) | **Points:** 21 | **Dependencies:** E04 amendment | **Milestone:** AgenticSAI Pivot

> **Source:** `sprint-change-proposal-2026-05-12-agenticsai.md`, `prd-amendment-2026-05-12-agenticsai.md` (FR-45, FR-46), `architecture-amendment-2026-05-12-agenticsai.md` (ADR-018).

## Goal

Make EU Solicit's company lifecycle and AgenticSAI's Project lifecycle move in lockstep. On company creation in EU Solicit, automatically create a corresponding AgenticSAI Project under the singleton EU Solicit Organisation; provision a Project-scoped API key (Fernet-encrypted at rest); seed a default knowledge-base storage-resource; register tenant-default MCP servers (Dynamics + HubSpot stubs, `inactive` until OAuth-connected); apply the AgenticSAI Project template that seeds grant/compliance/qualification/quantification agents; publish a `tenant.provisioned` event. On company archive, soft-delete the AgenticSAI Project, revoke the API key, and delete MCP-server secrets. A nightly reconciliation job verifies every active EU Solicit company has a healthy AgenticSAI Project; orphans surface to admin attention.

Without this epic, no tenant can use the post-pivot platform — every other AgenticSAI-facing capability (KB, agents, MCP, workflows) has no parent to attach to.

## Acceptance Criteria

- [ ] Company creation in EU Solicit triggers automatic AgenticSAI Project creation within 30s (p95)
- [ ] Project name follows pattern `eusolicit-company-{company_id}`; description includes EU Solicit company name + creation timestamp for AgenticSAI-side operability
- [ ] Project-scoped api-key issued via `POST /api/organizations/{orgId}/keys`, stored Fernet-encrypted in `client.agenticsai_projects.api_key_encrypted`
- [ ] Default KB storage-resource seeded: one vector store per Project named `default-kb`, ready for artefact uploads (E25)
- [ ] AgenticSAI Project template applied: grant/compliance/qualification/quantification agents seeded under the new Project; `agent_map` JSONB populated in `client.agenticsai_projects.agent_map`
- [ ] Tenant-default MCP servers registered (Dynamics 365 + HubSpot stubs, status `inactive` until OAuth-connected per E27)
- [ ] `provisioning_status` field on `client.agenticsai_projects`: `pending` → `provisioned` on success; `failed` on terminal failure; retry path for transient failures
- [ ] Exponential-backoff retry on transient AgenticSAI failures: 5 attempts over 4 hours; after exhaustion, status `failed` and admin alert
- [ ] `tenant.provisioned` event published to Redis Streams on success; consumed by `notification` (welcome email enriched with KB-upload CTA) and `client-api` (UI badge update)
- [ ] Nightly reconciliation job (Celery Beat): for every `client.companies` not archived, verify corresponding `client.agenticsai_projects` row exists with `provisioning_status='provisioned'`; orphans logged + admin-API endpoint surfaces them
- [ ] Admin-API endpoint `POST /api/v1/admin/companies/{id}/reprovision`: manually retry a failed provisioning; idempotent
- [ ] Company archive flow: AgenticSAI Project soft-deleted within 24h (`DELETE /api/organizations/{orgId}/projects/{projectId}`); api-key revoked; MCP-server secrets deleted; `client.agenticsai_projects.archived_at` set
- [ ] Right-to-Erasure (GDPR Art. 17) flow: erasure for a company traverses `pipeline.opportunities` deletion + `client.agenticsai_kb_files` deletion + AgenticSAI Project archival, with two-ACK audit-log proof (`erasure_step: postgres_rows_deleted`, `erasure_step: agenticsai_files_deleted`) per ADR-019
- [ ] Cross-tenant negative test: company-A provisioning failure does not block or affect company-B provisioning
- [ ] Failure-mode coverage: AgenticSAI API key revoked mid-provisioning; AgenticSAI rate-limit during provisioning; partial-state recovery (Project created but key generation failed)
- [ ] **Free-tier policy (Deb decision 2026-05-12):** Free-tier company provisioning follows the **standard flow** — Project created, KB seeded, MCP stubs registered, default agents seeded. Free-tier rate limit (100 runs/day per NFR-25) is applied automatically by E04 amendment S04.28 tier-sync flow on company create. No tier-conditional code path in E24 itself; tier behavior is rate-limit-driven, not provisioning-flow-driven. Closes readiness Concern #10.

## Stories

### S24.01: `client.agenticsai_projects` + `client.agenticsai_mcp_servers` schema + provisioning state machine
**Points:** 3 | **Type:** backend

Alembic migration for **two `client` schema tables** per architecture amendment §4.1: `client.agenticsai_projects` (with `agent_map` JSONB, Fernet-encrypted `api_key_encrypted`, partial index on non-`provisioned` rows) and `client.agenticsai_mcp_servers` (per-provider state + UNIQUE `(company_id, provider)`). SQLAlchemy models with `provisioning_status` field as a `StrEnum` (`pending`, `provisioned`, `failed`, `archived`); MCP server `status` as StrEnum (`inactive`, `registered`, `error`). State-machine helper module for provisioning lifecycle: transition rules, audit-log writes on every transition. Unit tests for all transitions. Closes readiness Concern #2 (client-schema migrations).

**Acceptance:**
- Migration creates both tables; downgrade cleanly drops them
- StrEnum enforces valid status values (per ADR-012) on both tables
- State-machine transitions validated; invalid transitions raise `InvalidProvisioningTransition`
- Partial index on `client.agenticsai_projects` used by `EXPLAIN ANALYZE` reconciler query
- UNIQUE constraint on `client.agenticsai_mcp_servers (company_id, provider)` enforced

---

### S24.02: AgenticSAI Project creation on company create
**Points:** 5 | **Type:** backend

Hook into the existing `client.companies` create flow: on commit, dispatch a Celery task `provision_agenticsai_project(company_id)`. Task body: (1) call `POST /api/organizations/{orgId}/projects` with project name + description; (2) call `POST /api/organizations/{orgId}/keys` to issue Project-scoped api-key; (3) Fernet-encrypt and persist to `client.agenticsai_projects.api_key_encrypted`; (4) set `provisioning_status='provisioned'`. On any step failure: increment retry counter, exponential backoff (1m, 5m, 30m, 2h, 4h), terminal `failed` after 5 attempts.

**Acceptance:**
- Company create triggers provisioning within 30s p95 (measured by Prometheus)
- Provisioning runs in background; company creation API response does not block on AgenticSAI call
- Idempotent: re-invocation with same company_id is no-op if status is `provisioned`
- Cross-tenant negative: failure on company-A retry doesn't block company-B provisioning

---

### S24.03: AgenticSAI Project template seed (default agents + KB)
**Points:** 5 | **Type:** backend

After Project creation (S24.02 step 1), apply the Project template: (a) create default storage-resource `default-kb` via `POST /api/organizations/{orgId}/projects/{projectId}/storage-resources`; (b) seed 8+ grant/compliance/qualification/quantification agents per templates authored in S11.20 amendment + new authoring per E26; (c) populate `client.agenticsai_projects.agent_map` JSONB with `{logical_name: agenticsai_agent_uuid}` for each seeded agent. Template authoring lives in `services/client-api/config/agenticsai_project_template.yaml` — versioned and PR-reviewed.

**Acceptance:**
- Template applied within 60s of Project creation
- `agent_map` populated for every agent in template; `agenticsai-gateway.call_agent("logical", company_id)` resolves correctly
- Template version recorded in `client.agenticsai_projects.template_version` for forward migrations
- Integration test: seed → invoke a templated agent → expected response

---

### S24.04: Tenant-default MCP server registrations (inactive stubs)
**Points:** 3 | **Type:** backend

After template seed (S24.03), register Dynamics 365 + HubSpot MCP-server stubs in the Project: `POST /api/organizations/{orgId}/projects/{projectId}/mcp-servers` with `status='inactive'` (per E27 contract). Persist `client.agenticsai_mcp_servers` rows. No secrets pushed yet (those come at OAuth completion per E27 S17.32 amended).

**Acceptance:**
- Both MCP-server rows created per company
- `client.agenticsai_mcp_servers.status='inactive'` until E27 OAuth flow completes
- Stubs queryable via admin-API endpoint for ops visibility

---

### S24.05: Nightly reconciliation + admin reprovision endpoint
**Points:** 3 | **Type:** backend

Celery Beat task `reconcile_agenticsai_projects` (daily 03:00 UTC): for each non-archived `client.companies`, verify `client.agenticsai_projects` row exists with `provisioning_status='provisioned'` and that AgenticSAI-side Project still exists (`GET /api/organizations/{orgId}/projects/{projectId}` returns 200). Orphans (EU Solicit company without provisioned Project, or Project missing on AgenticSAI side) logged + surfaced via `GET /api/v1/admin/agenticsai-projects/orphans`. Admin endpoint `POST /api/v1/admin/companies/{id}/reprovision` triggers retry; idempotent.

**Acceptance:**
- Daily reconciliation runs; orphans count exported as Prometheus gauge `agenticsai_provisioning_orphans_total`
- Admin endpoint surfaces orphan list with `last_attempt_at`, `last_error`
- Reprovision endpoint succeeds on previously-failed companies after underlying issue resolved

---

### S24.06: Company archive → AgenticSAI Project soft-delete + key revocation
**Points:** 2 | **Type:** backend

On `company.archived` event (existing Redis Stream): dispatch `archive_agenticsai_project(company_id)`. Steps: (a) revoke api-key via `DELETE /api/organizations/{orgId}/keys/{keyId}`; (b) delete MCP-server secrets per `client.agenticsai_mcp_servers` row; (c) soft-delete Project via `DELETE /api/organizations/{orgId}/projects/{projectId}`; (d) set `client.agenticsai_projects.archived_at` and `provisioning_status='archived'`. Failure path: alert admin, do NOT mark archived until AgenticSAI side confirms.

**Acceptance:**
- Archive completes within 24h
- Audit log records each step with timestamps
- Failed archive (e.g. AgenticSAI 500) blocks final state transition; admin alerted; retry available

---

### S24.07: Right-to-Erasure cross-substrate completion proof
**Points:** 3 | **Type:** backend

Extend existing GDPR erasure flow (Epic 1/9): for each erased company, after the standard Postgres deletion path, invoke AgenticSAI cleanup: iterate `client.agenticsai_kb_files` and call `DELETE /storage-resources/{id}/files/{fileId}` for each; archive the Project per S24.06. Audit log writes two ACKs: `erasure_step: postgres_rows_deleted` and `erasure_step: agenticsai_files_deleted`. Erasure not certified until both present. Nightly sweep job catches stale erasure rows missing the AgenticSAI ACK.

**Acceptance:**
- Erasure certificate (audit-log query) shows both ACKs for every erased company
- Sweep job surfaces stale erasure attempts > 48h old; admin alert
- Negative test: AgenticSAI ACK missing → erasure status remains `in_progress` (not `certified`)
- **Verify `shared.audit_log.details->>'erasure_step'` queries are performant; add GIN index on `(details)` (or expression index on `details->>'erasure_step'`) if EXPLAIN ANALYZE shows sequential scan over audit_log on the sweep query. Closes readiness Concern #9.**

---

### S24.08: One-off backfill — provision AgenticSAI Projects for existing companies
**Points:** 3 | **Type:** backend + ops

One-off Celery task `backfill_agenticsai_projects()` invoked manually (admin endpoint `POST /api/v1/admin/agenticsai-projects/backfill`) at pivot rollout time. Enumerates all non-archived `client.companies` rows that lack a corresponding `client.agenticsai_projects` row. For each, invokes the standard provisioning flow (S24.02 path). Throttle: 1 concurrent provisioning per minute to avoid AgenticSAI rate-limit storm. Idempotent: re-running mid-failure resumes from where it stopped (skips companies with `provisioned` status). Each backfill action writes an audit row with `event_type='backfill_provisioned'`. Pre-flight check returns company count + estimated duration before backfill starts. Operator runbook at `eusolicit-docs/runbooks/agenticsai-backfill.md`: pre-flight (count companies, verify AgenticSAI quota), throttle override flag for staging speed-runs, monitoring during backfill, failure-recovery procedure. Closes readiness Concern #6.

**Acceptance:**
- Pre-flight endpoint returns `{eligible_companies: N, estimated_duration_minutes: M}` without side effects
- Backfill triggered → companies provisioned at throttled rate; audit row per success
- Crash recovery: kill mid-backfill → re-trigger → resumes idempotently
- Failure during one company's provisioning does not block others; failed companies surfaced via existing orphan endpoint (S24.05)
- Runbook tested on staging with synthetic 50-company batch

---

## Salvaged patterns

- Fernet encryption module (Epic 9 canonical) — same module that holds OAuth tokens.
- Celery Beat reconciliation pattern (Epic 5/8 — already applied to Stripe webhook reconciliation).
- Soft-delete uniqueness via partial index (Epic 14 pattern).
- `tenant.provisioned` event consumer pattern matches existing `subscription.changed` flow (Epic 8).

## Out of scope

- Per-tenant N8N provisioning (N8N is org-scoped per ADR-018 — shared EUSolicit-Org instance only).
- Migration of pre-pivot existing companies (separate one-off migration task, not part of the steady-state provisioning epic — owned by E22 onprem-launch close-out).
- AgenticSAI Project agent versioning beyond template version (covered in E11 amendment S11.20).
