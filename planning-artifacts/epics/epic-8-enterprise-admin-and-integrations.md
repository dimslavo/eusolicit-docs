---
epic: 8
title: Enterprise Admin and Integrations
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-8", "FR-9", "FR-42", "FR-43", "FR-44"]
nfrs_relevant: ["NFR-7", "NFR-8", "NFR-15", "NFR-23"]
ux_drs_relevant: ["UX-DR3"]
---

# Epic 8: Enterprise Admin and Integrations

**Phase**: Phase 2 / Phase 3 | **Dependencies**: Epic 1 (Identity), Epic 2 (Subscriptions), `admin-api`, all data services

## Goal

Enterprise customers can manage multiple isolated client workspaces under a single parent account, switch between them, and access platform data programmatically via a documented REST API. Internal EU Solicit operators get a separate, hardened admin portal to manage tenants, monitor crawler health, oversee subscriptions, and review the immutable audit trail.

## User Outcome

After this epic: a parent enterprise admin can spin up isolated client workspaces and switch context with a top-bar workspace switcher; a developer at an enterprise customer can integrate scored opportunities into Salesforce via a documented REST API; and an internal operator can investigate any tenant action through an immutable audit trail without ever touching the production app shell.

## Epic Acceptance Criteria

- [ ] Sub-tenant workspaces are logically isolated; cross-workspace queries are impossible from the application layer (NFR-7).
- [ ] The admin portal is reachable only over a configured allowlist (VPN/IP) and requires MFA (NFR-8).
- [ ] The audit log is immutable (append-only) and replicated for durability (NFR-15).
- [ ] The Enterprise API exposes OpenAPI 3 docs, supports filtering + keyset pagination, enforces tier-scoped rate limits, and authenticates via API keys with workspace scoping.
- [ ] All admin and API actions emit structured logs with correlation IDs for traceability (NFR-23).

## Stories

### Story 8.1: Sub-Tenant Workspace Management

As an enterprise admin,
I want to create and manage multiple isolated client workspaces under my parent account,
So that I can keep different clients' data strictly segregated.

**FR coverage**: FR-8, FR-9

**Acceptance Criteria:**

**Given** I have an Enterprise subscription
**When** I create a new sub-workspace from the parent account UI
**Then** a new tenant row is provisioned in `shared.tenants` with `parent_tenant_id` set
**And** all data scoped to that workspace is logically isolated and never returned by queries scoped to siblings or the parent.
**And** I can switch context using the workspace switcher (a top-bar dropdown surfaced via Cmd+K per UX-DR3).
**And** users can be members of multiple workspaces; the active workspace is reflected in the JWT `workspace_id` claim.

---

### Story 8.2: Immutable Audit Trail

As the platform,
I want to maintain an immutable audit log of all significant user and system actions,
So that enterprise customers can satisfy compliance and forensic requirements.

**FR coverage**: FR-43
**NFR coverage**: NFR-15

**Acceptance Criteria:**

**Given** a mutating action occurs (login, role change, financial mutation, framework change, document export, proposal submit, etc.)
**When** the action completes successfully
**Then** an append-only record is written to `shared.audit_log` via a background task with `actor_id`, `tenant_id`, `action`, `resource`, `before/after` (where applicable), `correlation_id`, and `created_at`.
**And** the table forbids `UPDATE` and `DELETE` via DB roles.
**And** an admin can search/filter the log by tenant, actor, action, and date range from the admin portal.
**And** failure to write the audit record alerts ops but does not roll back the user-facing action — the event bus retries the audit write.

---

### Story 8.3: Secure REST API Access

As an enterprise developer,
I want to access platform data via a secure REST API,
So that I can integrate EU Solicit with our internal systems (e.g., Salesforce, BI).

**FR coverage**: FR-44

**Acceptance Criteria:**

**Given** I am an Enterprise admin
**When** I generate an API key in workspace settings
**Then** I receive a one-time-revealed token bound to my workspace and tier.
**And** I can name, list, rotate, and revoke API keys; revoked keys stop working within 60s.
**Given** I make a request with a valid key to an Enterprise endpoint
**When** the request hits `client-api`
**Then** the response is correct JSON conforming to the OpenAPI spec
**And** rate limits (per-tier, per-key) are enforced with `429` and `Retry-After`.
**And** the OpenAPI explorer at `/api/docs` is publicly browsable for Enterprise scope.

---

### Story 8.4: Platform Admin Portal

As an internal EU Solicit operator,
I want a dedicated admin portal to manage tenants and monitor system health,
So that I can resolve operational issues securely without touching the public app.

**FR coverage**: FR-42
**NFR coverage**: NFR-8, NFR-23

**Acceptance Criteria:**

**Given** I am on the corporate VPN with my MFA token
**When** I log into the admin portal (Next.js admin app on port 3001 → `admin-api` on 8002)
**Then** access is restricted by IP allowlist enforced at the ingress layer.
**And** I can view crawler statuses, last-run timestamps, and per-source error rates.
**And** I can view & manage tenants (suspend, restore, change tier with audit-logged justification).
**And** I can browse the audit log (Story 8.2) with filtering.
**And** the admin portal never accepts public-internet traffic in production.

## Implementation Notes

- Sub-tenant isolation: every query that touches tenant-scoped tables must include `workspace_id` derived from the request context — DB-level constraints enforce this in addition to application code.
- The Enterprise API is served by `client-api` (not a separate service) but exposes a different prefix (e.g. `/api/v1/enterprise/`) and enforces API-key auth in addition to optional JWT.
- API keys are stored as bcrypt hashes; only the prefix is recoverable.
- Audit log writes go through the shared event bus; failures retry with bounded attempts before paging on-call.
- Admin portal MFA uses TOTP (RFC 6238) plus IP allowlist; consider WebAuthn for hardening in Phase 3.
