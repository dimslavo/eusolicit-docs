---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-17'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 6
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-05.md'
  - 'eusolicit-docs/test-artifacts/test-design-progress.md'
  - 'eusolicit/_bmad/bmm/config.yaml'
---

# Test Design: Epic 6 — Opportunity Discovery & Intelligence

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**Status:** Draft
**Epic:** E06 | **Sprint:** 5–6 | **Points:** 55 | **Dependencies:** E02, E03, E05

---

## Executive Summary

**Scope:** Epic-level test design for E06 — Opportunity Discovery & Intelligence. Covers the full client-facing opportunity access layer built on top of the E05 data pipeline: opportunity search and listing with cursor-based pagination and full-text filters (S06.01, S06.04), tier-gated response serialization enforcing subscription rules for region/CPV/budget/fields (S06.02), atomic Redis usage metering for AI features (S06.03), opportunity detail with related opportunities and submission guides (S06.05), S3-presigned document upload with ClamAV scanning (S06.06), access-controlled document download (S06.07), AI executive summary generation via SSE streaming (S06.08), and the complete frontend experience: listing page with table/card views (S06.09), search and filter sidebar (S06.10), tabbed detail page (S06.11), document upload component (S06.12), AI summary panel (S06.13), and upgrade prompt modal with global tier/usage interceptor (S06.14).

This epic is the **primary revenue-sensitive surface of EU Solicit**: all four subscription tiers are enforced here. Incorrect tier enforcement means revenue leakage (free users accessing paid data), incorrect billing (over-quota AI summaries consumed), or legal exposure (unauthenticated document access). The document upload/scan flow introduces a direct security surface via S3 presigned URLs and async ClamAV execution.

**Risk Summary:**

- Total risks identified: 10
- High-priority risks (≥6): 4 (TierGate bypass, Redis counter race, S3/ClamAV pre-scan gap, SSE exhaustion)
- Critical categories: SEC (TierGate bypass, ClamAV gap), DATA (Redis race), PERF (SSE exhaustion)

**Coverage Summary:**

- P0 scenarios: 10 (~20–35 hours)
- P1 scenarios: 30 (~30–45 hours)
- P2 scenarios: 15 (~12–20 hours)
- P3 scenarios: 5 (~3–6 hours)
- **Total effort:** ~65–106 hours (~1.5–2.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **Source data quality from AOP/TED/EU Grants** | E05 owns ingestion and normalization; E06 reads `pipeline.opportunities` as-is | E05 test design covers crawler correctness; E06 tests treat pipeline data as trusted fixture input |
| **KraftData Executive Summary Agent internal logic** | E04 AI Gateway owns the proxy and model routing; E06 only tests SSE streaming behavior at the client boundary | Mock AI Gateway at the `httpx` call boundary; E04 integration tested separately |
| **Billing / subscription tier upgrade flow** | A future Billing epic owns tier changes, Stripe integration, and VAT; E06 only tests the **consequences** of an already-assigned tier claim in the JWT | Seed test users with pre-assigned tier claims in JWT fixtures; upgrade CTA destination URL is verified as present, not functionally tested |
| **S3 infrastructure reliability** | AWS-managed; E06 verifies presigned URL generation and expiry parameters, not S3 availability or durability | Use localstack or boto3-mocked presigned URL generation in CI |
| **ClamAV internal scan classification accuracy** | Platform/DevOps concern; E06 verifies scan status transitions and access control enforcement based on reported status | Mock ClamAV scan callback returning clean/infected payloads; real scan accuracy tested in infra smoke tests |
| **Redis infrastructure reliability** | Platform concern; E06 verifies atomic INCR semantics and TTL behaviour using fakeredis/testcontainers Redis | Use fakeredis for unit tests; testcontainers Redis for integration tests |
| **Actual payment / upgrade destination page** | Billing epic owns the destination; E06 verifies the upgrade CTA is present with a non-empty URL | Assert CTA `href` is non-empty and points to billing path; do not navigate to or test the billing page |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E06-R-001** | **SEC** | TierGate bypass via direct ID enumeration — free-tier users may call `GET /api/v1/opportunities/{id}` directly (bypassing the listing gate) or manipulate cursor tokens to retrieve full `OpportunityFullResponse` if TierGate is not applied consistently at every layer: route handler, dependency injection, and response serializer; any gap between layers allows a free user to receive paid-tier data, representing both a revenue bypass and a data access control violation | 2 | 3 | **6** | Apply `TierGate` as a FastAPI dependency on every endpoint that returns opportunity data (not just the listing endpoint); never trust the response model selection to the caller; integration test: call detail and listing endpoints with a free-tier JWT and assert `OpportunityFreeResponse` shape at the field level; also assert 403 when free-tier attempts detail endpoint | Backend / QA | Sprint 5 |
| **E06-R-002** | **DATA** | Redis usage counter race condition — S06.03 specifies "atomic GET+INCR via Redis pipeline", but if the implementation uses two separate commands (GET to check, then INCR to increment) rather than a Lua script or `MULTI/EXEC` pipeline, two concurrent AI summary requests can both read the same counter value (e.g., 9 of 10), both pass the limit check, then both decrement — resulting in 11 summaries consumed against a 10-limit quota; this directly corrupts billing signal, erodes subscription value, and may trigger downstream quota alerts | 2 | 3 | **6** | Implement GET+INCR as a single atomic operation using a Redis Lua script or `MULTI/EXEC` pipeline; never separate the check and the increment across two round-trips; integration test with fakeredis: fire two concurrent usage-gated requests when counter is at N-1 (one below limit) and verify only one succeeds, the other receives 429 | Backend / QA | Sprint 5 |
| **E06-R-003** | **SEC** | S3 presigned URL / ClamAV pre-scan download gap — the upload flow generates a 15-minute presigned PUT URL; after the client calls the confirm callback (`POST .../documents/{doc_id}/confirm`), `scan_status` is set to `pending` and ClamAV runs asynchronously; if `scan_status` defaults to `clean` (rather than `pending`) before the scan completes — or if the download endpoint serves files in `pending` state — an infected file can be downloaded before it is caught; additionally, the presigned PUT URL remains valid for 15 minutes and can be shared or reused, bypassing the per-user upload rate and package size limits | 2 | 3 | **6** | Confirm that `scan_status` is set to `pending` (not `clean`) immediately after the confirm callback; `GET .../download` must return 422 for any document not in `clean` state; integration test: call confirm, immediately attempt download, verify 422; also test that ClamAV `infected` callback marks document infected and soft-deletes S3 object | Backend / QA | Sprint 5–6 |
| **E06-R-004** | **PERF** | SSE connection exhaustion under concurrent AI summary load — `POST /api/v1/opportunities/{id}/ai-summary` keeps an HTTP connection open for the full streaming duration of the AI Gateway response; Professional and Enterprise tier users (50 and unlimited summaries/month) can trigger simultaneous SSE connections; with no concurrency cap or backpressure on the SSE endpoint, a burst of concurrent summary requests may exhaust uvicorn worker slots or upstream AI Gateway concurrency limits, blocking all API traffic for other tenants | 2 | 3 | **6** | Define and enforce a per-endpoint concurrency cap (e.g., max N active SSE streams); document in architecture decision; load test: fire 20 concurrent SSE requests, verify response time SLA for non-SSE endpoints is unaffected; verify 503 or queuing behavior when cap is exceeded | Backend / QA | Sprint 6 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E06-R-005 | SEC | Cursor forgery — opaque pagination cursors that encode DB row IDs or offsets could be reconstructed or manipulated by clients to skip tier-scope filtering between pages; if TierGate is only applied at the first page and cursor-based continuation bypasses it, a paid user in a limited region could craft a cursor pointing to out-of-scope opportunities | 2 | 2 | 4 | Re-apply TierGate on every paginated request, not just the first; cursors must encode only the sort key and not grant implicit scope elevation; test: obtain valid cursor for in-scope page, mutate cursor to reference an out-of-scope row, verify 403 or filtered response | Backend / QA |
| E06-R-006 | BUS | Concurrent upload package size race — S06.06 specifies total package ≤ 500MB checked against existing `client.documents` sum; if two concurrent upload requests both query the current total (e.g., 450MB), both pass the 500MB check, and both are confirmed, the actual package reaches 550MB, violating the storage constraint | 2 | 2 | 4 | Use a DB-level advisory lock or `SELECT ... FOR UPDATE` on the `client.documents` size aggregation before issuing the presigned URL; test: concurrent upload requests from the same user for 300MB + 300MB starting from 200MB used, verify only the first is accepted | Backend / QA |
| E06-R-007 | DATA | Stale AI summary cache — the cached summary is returned for < 24h without checking whether the underlying opportunity data has been updated (e.g., deadline extended, budget revised) since the summary was generated; users receive outdated intelligence on a materially changed opportunity | 2 | 2 | 4 | On summary retrieval, compare `ai_summaries.generated_at` against `pipeline.opportunities.updated_at`; if opportunity was updated after summary generation, mark summary stale and require regeneration (with metering); test: generate summary, update opportunity, fetch again, verify regeneration triggered | Backend / QA |
| E06-R-008 | SEC | ClamAV async timing — if the `scan_status` transitions from `pending` to `clean` before the actual ClamAV result is received (e.g., a timeout or no-callback scenario), documents could become downloadable without being scanned; no timeout-to-failed transition is specified | 2 | 2 | 4 | Add a background job that marks `scan_status = failed` for documents stuck in `pending` beyond a configurable timeout (e.g., 15 minutes); test: trigger confirm callback without sending ClamAV result, advance clock past timeout, verify scan_status = failed and download returns 422 | Backend / QA |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E06-R-009 | OPS | Filter state URL param exposure — budget range, CPV codes, and region filters are persisted as query params for shareable links; these may appear in server access logs or browser history, potentially exposing competitive intelligence about a company's procurement focus area | 1 | 2 | 2 | Document: sensitive filters should be treated as non-PII business data; ensure logs are access-controlled; no test required |
| E06-R-010 | OPS | fakeredis parity — INCR pipeline atomic behaviour in fakeredis may differ from real Redis under concurrent loads in edge cases; unit tests using fakeredis may not catch the race scenario from E06-R-002 | 1 | 1 | 1 | Use testcontainers Redis (not just fakeredis) for the concurrency race tests (E06-P0-004); fakeredis is acceptable for serial unit tests only |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- **E06-R-001** (TierGate bypass) → extends system **R-01** (Multi-tenancy Isolation). Subscription tier is a revenue-critical form of access control; a tier bypass is equivalent in impact to a tenant data leak — it grants access to paid-tier data without entitlement.
- **E06-R-002** (Redis counter race) → extends system **R-02** (Billing Inaccuracy). Consuming AI summaries beyond quota without billing enforcement is a direct financial integrity violation.
- **E06-R-004** (SSE exhaustion) → extends system **R-04** (Search Latency). Long-lived SSE connections are a generalized API performance concern; shared worker slot exhaustion degrades all endpoints for all tenants.
- **E06-R-003** and **E06-R-008** (ClamAV pre-scan gap) → new epic-level SEC risks not covered by system-level assessment. Flag for architecture security review regarding async scan completion guarantees.

---

## Entry Criteria

- [ ] E02 complete: Auth service with JWT issuance including `subscription_tier` claim (`free` | `starter` | `professional` | `enterprise`); JWT fixtures available for all four tiers
- [ ] E03 complete: Next.js frontend scaffold with app shell, routing, Zustand stores, TanStack Query, and i18n setup operational
- [ ] E05 complete: `pipeline.opportunities` and `pipeline.submission_guides` tables populated with test seed data covering multiple regions, CPV sectors, and budget ranges
- [ ] E04 complete: AI Gateway operational in Docker Compose; SSE proxy endpoint `/execute/stream` stable; `httpx`-based client importable
- [ ] Database migrations applied for `client.documents` and `client.ai_summaries` tables
- [ ] Redis available in CI (testcontainers or Docker Compose); fakeredis configured for unit test layer
- [ ] S3 presigned URL generation mocked in CI via localstack or `boto3` mock; no real AWS credentials required for CI
- [ ] ClamAV scan callback mocked in CI (no real ClamAV process required)

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95%; failures triaged with PM sign-off before release)
- [ ] No open high-severity bugs in TierGate enforcement, UsageGate atomicity, or document access control
- [ ] Line coverage ≥80% on `services/client_api/routers/` (opportunity, document, ai_summary endpoints) and `services/client_api/middleware/` (tier_gate, usage_gate)
- [ ] Component test coverage ≥75% on frontend components (S06.09–S06.14 components)
- [ ] All 8 backend API endpoints have OpenAPI documentation (per E06 acceptance criteria)
- [ ] Integration test suite (backend) passes in CI within 5 minutes
- [ ] No document downloadable with `scan_status != clean` in any test scenario

---

## Test Coverage Plan

> **Note:** P0/P1/P2/P3 denote **priority and risk level**, NOT execution timing. Execution scheduling is defined separately in the Execution Strategy section.

### P0 (Critical)

**Criteria:** Blocks core tier-gated access + High risk (≥6) + No safe workaround (revenue or security impact)

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E06-P0-001 | TierGate blocks free-tier from detail endpoint — call `GET /api/v1/opportunities/{id}` with free-tier JWT; verify 403 with `{"error": "tier_limit", "upgrade_url": "..."}` | S06.02, S06.05 | API | E06-R-001 | Confirms detail endpoint never serves free-tier users regardless of call path |
| E06-P0-002 | Free-tier receives `OpportunityFreeResponse` with only allowed fields — call listing API with free-tier JWT; assert response fields are exactly `{id, name, deadline, location, type, status}`; assert `budget`, `cpv_codes`, `evaluation_criteria`, `relevance_score` are absent | S06.02, S06.04 | API | E06-R-001 | Test field-level enforcement, not just HTTP status |
| E06-P0-003 | TierGate blocks paid-tier from out-of-scope opportunity — Starter JWT (Bulgaria-only) requests opportunity in France; verify 403 with `tier_limit` error; also verify Professional JWT (BG + 3 countries) cannot access opportunity in a 4th non-BG country | S06.02 | API | E06-R-001 | Parameterized across Starter + Professional scope boundaries |
| E06-P0-004 | UsageGate atomic counter — two concurrent AI summary requests when counter is at limit−1 (9 of 10 for Starter); verify exactly one succeeds (200), the other receives 429; verify `ai_summaries` table has exactly one new row | S06.03, S06.08 | Integration | E06-R-002 | Use testcontainers Redis (not fakeredis) for concurrency guarantee; fire requests with `asyncio.gather` |
| E06-P0-005 | UsageGate returns 429 with correct headers and body when limit exceeded — exhaust Starter quota (10 summaries); call AI summary endpoint; verify 429, body `{"error": "usage_limit_exceeded", "limit": 10, "used": 10, "resets_at": "..."}`, header `X-Usage-Remaining: 0`, and `upgrade_url` present | S06.03 | API | E06-R-002 | Verify both response body schema and header in one test |
| E06-P0-006 | Document download blocked before ClamAV scan — call confirm callback, immediately call download endpoint; verify 422 with appropriate error (scan pending); verify that `scan_status = pending` record is not served | S06.06, S06.07 | Integration | E06-R-003 | Confirm the download gate checks `scan_status` correctly before generating presigned GET URL |
| E06-P0-007 | ClamAV infected scan blocks download and soft-deletes S3 object — mock ClamAV callback returning `infected`; verify `scan_status = infected`, download returns 422, and S3 soft-delete is triggered (mock assertion on delete call) | S06.06, S06.07 | Integration | E06-R-003 | Core security gate for malware protection |
| E06-P0-008 | AI summary usage counter decrements atomically; cached summary (< 24h) returned without decrement — generate summary (counter decrements by 1); call GET endpoint within 24h; verify counter unchanged and summary returned from cache | S06.03, S06.08 | Integration | E06-R-002 | Verifies the caching exemption from metering per S06.08 AC |
| E06-P0-009 | SSE streaming delivers all event types in correct order — call AI summary SSE endpoint for Enterprise-tier user; verify event sequence: one or more `delta` events → `metadata` event → `done` event; verify `done` event includes `summary_id`; verify completed summary persisted in `client.ai_summaries` | S06.08 | Integration | E06-R-004 | Use `httpx` streaming client; assert each event type present and schema-valid |
| E06-P0-010 | End-to-end tier enforcement: free vs. paid user journey — E2E: sign in as free user → navigate to `/opportunities` → verify blurred/locked fields on cards → attempt to click detail page → verify upgrade prompt; sign in as Starter user → verify in-scope opportunities accessible with full fields; verify out-of-scope opportunities return upgrade prompt | S06.02, S06.04, S06.09, S06.14 | E2E | E06-R-001 | Playwright test covering the full UI tier-gate flow for two distinct tiers |

**Total P0: 10 tests, ~20–35 hours**

---

### P1 (High)

**Criteria:** Critical feature paths + Medium/high risk + Common execution paths across all stories

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E06-P1-001 | Full-text search returns paginated results for query matching opportunity name and description | S06.01 | API | — | Use seeded `pipeline.opportunities` with known text; verify result contains expected records |
| E06-P1-002 | All filter combinations active simultaneously — CPV + region + budget_min/max + deadline range + status + type — returns correctly filtered results | S06.01 | API | — | Parameterized test with at least 3 filter combinations; verify zero false positives |
| E06-P1-003 | Cursor-based pagination: `after_cursor` advances to next page, `before_cursor` retrieves prior page; total_count consistent across pages | S06.01 | API | E06-R-005 | Seed ≥50 opportunities; verify non-overlapping pages; verify total_count stable |
| E06-P1-004 | Search `limit` parameter defaults to 25, accepts up to 100, rejects values > 100 | S06.01 | API | — | Test boundary values: 1, 25, 100, 101 |
| E06-P1-005 | `OpportunityFreeResponse` Pydantic model excludes budget, CPV codes, contracting authority details, evaluation criteria, relevance score | S06.02 | Unit | E06-R-001 | Test model serialization directly with a full opportunity object |
| E06-P1-006 | `OpportunityFullResponse` Pydantic model includes all fields including evaluation_criteria JSONB, mandatory_documents list, relevance_score | S06.02 | Unit | — | Validates JSONB deserialization shape |
| E06-P1-007 | TierGate FastAPI dependency reads `subscription_tier` from JWT claims and routes to correct response model | S06.02 | Unit | E06-R-001 | Test dependency in isolation with mocked JWT payload for each tier |
| E06-P1-008 | UsageGate INCR sets EXPIRE to end-of-billing-period TTL — verify Redis key TTL is ≤ seconds remaining in billing period after INCR | S06.03 | Unit | E06-R-002 | Use fakeredis; assert TTL is correct relative to mocked billing period end |
| E06-P1-009 | UsageGate sets `X-Usage-Remaining` header on successful metered request; value decrements correctly across sequential calls | S06.03 | API | — | Call 3 times against Starter (10-limit) tier; verify header values: 9, 8, 7 |
| E06-P1-010 | Listing API browse mode (no `q` param) applies TierGate to each result; supports `sort_by=relevance_score,deadline,budget,published_date` and `sort_order=asc,desc` | S06.04 | API | — | 4 sort_by × 2 sort_order = 8 sort combinations; at minimum test relevance_score and deadline |
| E06-P1-011 | Listing API response includes `next_cursor`, `prev_cursor`, `total_count`, and `results[]`; free users get `OpportunityFreeResponse[]` | S06.04 | API | — | Verify pagination metadata fields present for both free and paid tier responses |
| E06-P1-012 | Detail API returns full tender data including evaluation_criteria, mandatory_documents, submission deadline with timezone for paid tier | S06.05 | API | — | Verify field presence and type; timezone field must include offset |
| E06-P1-013 | Detail API returns `related_opportunities` — up to 5 opportunities with CPV/region overlap | S06.05 | API | — | Seed 6+ related opportunities; verify max 5 returned; verify CPV/region overlap |
| E06-P1-014 | Detail API returns 404 for non-existent `opportunity_id` | S06.05 | API | — | Use random UUID; assert 404 status |
| E06-P1-015 | Detail API returns linked AI analysis summary (when previously generated) without triggering metering | S06.05, S06.08 | API | E06-R-002 | Pre-seed `client.ai_summaries`; call detail endpoint; assert summary present, counter unchanged |
| E06-P1-016 | Document upload validates allowed MIME types (PDF, DOCX, XLSX, ZIP); rejects JPEG and EXE | S06.06 | API | — | Test 4 allowed + 2 rejected MIME types |
| E06-P1-017 | Document upload generates S3 presigned PUT URL with 15-minute expiry; metadata recorded with `scan_status: pending` after confirm | S06.06 | Integration | E06-R-003 | Assert presigned URL expiry parameter; assert DB record scan_status=pending after confirm |
| E06-P1-018 | Document download generates S3 presigned GET URL with 10-minute expiry and `Content-Disposition: attachment` header | S06.07 | Integration | — | Assert URL parameters; assert Content-Disposition value |
| E06-P1-019 | Document download denies non-owner without tier access — User B attempts to download User A's document; verify 403 | S06.07 | API | — | Two separate user JWTs; verify ownership check |
| E06-P1-020 | Document download logs audit event — successful download triggers audit log entry with user_id, document_id, and timestamp | S06.07 | Integration | — | Inspect structured log output or audit table after download |
| E06-P1-021 | AI summary GET endpoint returns cached summary without re-generating — verify no AI Gateway call made and no counter decrement | S06.08 | API | E06-R-002 | Use respx mock to assert zero outbound calls to AI Gateway |
| E06-P1-022 | AI summary error event streamed on AI Gateway failure — mock AI Gateway returning 5xx; verify SSE `error` event type delivered; verify response terminates gracefully | S06.08 | Integration | — | Client must receive error event, not an orphaned connection |
| E06-P1-023 | Opportunity listing page renders table view with sortable columns and card view; toggle switch works | S06.09 | Component | — | React Testing Library; verify column headers and card elements render; toggle changes layout |
| E06-P1-024 | Infinite scroll / "Load More" pagination fires next page request and appends results without duplicates | S06.09 | Component | — | Mock TanStack Query; simulate scroll or click Load More; assert request fired and results appended |
| E06-P1-025 | Filter sidebar: CPV autocomplete loads taxonomy; region checkboxes with select-all; budget slider; deadline date range picker — all filter changes fire debounced API calls | S06.10 | Component | — | Test each filter control type; assert API call fired after debounce delay |
| E06-P1-026 | Active filter chips render above results; clicking chip removes filter and re-fetches; "Clear all" removes all chips | S06.10 | Component | — | Interaction test |
| E06-P1-027 | Detail page tabbed layout renders all 5 tabs: Overview, Documents, Requirements, AI Analysis, Submission Guide | S06.11 | Component | — | Assert each tab panel renders expected content elements |
| E06-P1-028 | Document upload component: drag-drop upload flow — presigned URL requested, S3 PUT initiated, confirm called, scan status polling shows pending → clean | S06.12 | Component | — | Mock API calls; simulate file drop; assert status badge transitions |
| E06-P1-029 | AI summary panel: "Generate Summary" CTA triggers SSE connection; streaming text renders; typing animation visible; usage counter shows remaining quota | S06.13 | Component | — | Mock SSE EventSource; simulate delta events; assert text appended progressively |
| E06-P1-030 | Upgrade prompt modal triggered by 403 `tier_limit` response — global API interceptor catches 403, displays modal with tier comparison table and primary CTA; modal dismissible | S06.14 | Component | — | Mock Axios/fetch interceptor; fire 403 response; assert modal rendered |

**Total P1: 30 tests, ~30–45 hours**

---

### P2 (Medium)

**Criteria:** Secondary flows + Low/medium risk (1–4) + Edge cases and configuration validation

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E06-P2-001 | Search with no matching results returns empty `results[]`, `total_count = 0`, and no cursors | S06.01 | API | — | Empty-state correctness |
| E06-P2-002 | Search with invalid cursor returns 400 or safe empty result (no crash or data leak) | S06.01 | API | E06-R-005 | Mutated cursor string; assert no 500 and no data outside scope |
| E06-P2-003 | Cursor re-applies TierGate — Starter user obtains valid cursor; mutates cursor to reference out-of-scope French opportunity; verify 403 or filtered empty result | S06.01, S06.02 | API | E06-R-005 | Direct test of per-page tier re-enforcement |
| E06-P2-004 | Concurrent upload package size race — two concurrent upload requests (300MB each) from a user with 200MB existing; verify only one is accepted; total package stays ≤ 500MB | S06.06 | Integration | E06-R-006 | Use `asyncio.gather`; assert DB sum ≤ 500MB |
| E06-P2-005 | Document upload rejects file with size > 100MB at API validation layer (before presigned URL issuance) | S06.06 | API | — | Client-size pre-check validation; 400 expected |
| E06-P2-006 | ClamAV scan timeout — document stuck in `pending` beyond configurable timeout is marked `failed`; download returns 422 | S06.06, S06.07 | Integration | E06-R-008 | Advance clock using freezegun; assert status transition |
| E06-P2-007 | AI summary cache invalidation — summary generated, then `pipeline.opportunities.updated_at` advanced; next GET detects staleness and triggers regeneration (with metering) | S06.08 | Integration | E06-R-007 | Assert counter decrements on staleness-triggered regen |
| E06-P2-008 | AI summary "Regenerate" button shows confirmation dialog with remaining quota count before submitting | S06.13 | Component | — | Assert dialog renders with quota value |
| E06-P2-009 | AI summary panel shows "X of Y summaries remaining this month" usage counter; updates after generation | S06.13 | Component | — | Mock GET usage endpoint; assert counter text matches |
| E06-P2-010 | Submission guide step-by-step accordion renders from `pipeline.submission_guides.steps` JSONB | S06.11 | Component | — | Seed mock steps array; assert accordion items rendered in order |
| E06-P2-011 | Smart matching relevance score visible on listing cards and detail page for paid users; hidden for free users | S06.09, S06.11 | Component | — | Two fixtures (free vs. paid); assert score element visible/absent |
| E06-P2-012 | Filter state persisted in URL query params — apply filters, copy URL, navigate to URL in fresh context, verify filters restored | S06.10 | Component | — | Assert URL params and restored filter state |
| E06-P2-013 | Lock icons and "Upgrade to unlock" overlays visible on restricted fields for free-tier users in listing and detail | S06.09, S06.14 | E2E | — | Playwright screenshot assertion or DOM element check |
| E06-P2-014 | Mobile responsive: listing page collapses to card-only view at 375px; filter sidebar renders as bottom sheet | S06.09, S06.10 | Component | — | Resize viewport; assert layout changes |
| E06-P2-015 | Upgrade prompt modal triggered by 429 `usage_limit_exceeded` response — interceptor catches 429, renders modal with "10/10 AI summaries used" and recommended tier highlighted | S06.14 | Component | — | Symmetric with P1-030; tests usage limit path |

**Total P2: 15 tests, ~12–20 hours**

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Benchmarks and configuration correctness

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E06-P3-001 | OpenAPI schema coverage — all 8 backend endpoints appear in `/openapi.json` with correct request/response schemas for each tier variant | S06.01–S06.08 | Unit | — | `fastapi.testclient` schema assertion |
| E06-P3-002 | Deadline countdown timer on detail page decrements in real time — mock `Date.now()`; advance 1 second; assert countdown display changes | S06.11 | Unit | — | Component unit test; no E2E timer dependency |
| E06-P3-003 | Tab state persisted in URL hash — navigate to Documents tab; URL hash = `#documents`; reload; verify Documents tab is active | S06.11 | Component | — | URL hash preservation |
| E06-P3-004 | fakeredis atomic INCR — verify sequential usage gate calls in fakeredis match expected counter values; compare to testcontainers Redis baseline for serial case | S06.03 | Unit | E06-R-010 | Documents fakeredis sufficiency for serial unit tests |
| E06-P3-005 | Loading skeletons and empty states render correctly — simulate loading state (delayed API response) and empty results; assert skeleton and empty-state components visible | S06.09 | Component | — | UX completeness check |

**Total P3: 5 tests, ~3–6 hours**

---

## Execution Strategy

**Philosophy:** Backend API and integration tests run on every PR; frontend component tests run on every PR if suite completes in <10 minutes. E2E tests (Playwright) run nightly due to browser startup overhead. Real S3/ClamAV smoke tests are manual only.

| When | What | Expected Duration |
|------|------|------------------|
| **Every PR** | All P0, P1, P2 API + integration + unit tests (pytest with testcontainers + fakeredis + boto3 mock) | ~5–8 minutes |
| **Every PR** | All P1, P2, P3 frontend component tests (Vitest + React Testing Library) | ~3–5 minutes |
| **Nightly** | P0/P1 E2E tests (Playwright — E06-P0-010, E06-P1-023–E06-P1-030) | ~10–20 minutes |
| **Weekly / Manual** | Real S3 presigned URL + ClamAV integration smoke tests (tagged `@skip-ci`) | ~15–30 minutes |
| **Performance (Manual)** | SSE concurrency load test (20 concurrent streams — E06-R-004 mitigation verification) | ~30 minutes |

---

## Resource Estimates

| Priority | Test Count | Estimated Effort | Notes |
|----------|------------|-----------------|-------|
| P0 | 10 | ~20–35 hours | Mix of API, integration (concurrent), and E2E; testcontainers + real Redis for race tests |
| P1 | 30 | ~30–45 hours | Mix of API, unit, integration, component, and E2E; heavy cross-story coverage |
| P2 | 15 | ~12–20 hours | Mostly edge cases, configuration, and secondary UI flows |
| P3 | 5 | ~3–6 hours | Benchmarks, schema checks, timer unit tests |
| **Total** | **60** | **~65–106 hours** | **~1.5–2.5 weeks, 1 QA** |

### Prerequisites

**Test Fixtures and Factories:**

- `OpportunityFactory` — generates `pipeline.opportunities` records with CPV codes, regions, budget ranges, deadlines, and relevance scores; configurable per tier scope (in-scope, out-of-scope per tier)
- `UserJWTFactory` — generates signed JWT tokens with `subscription_tier` claim for all four tiers (`free`, `starter`, `professional`, `enterprise`); includes `user_id` and `billing_period_end` claims
- `DocumentFactory` — generates `client.documents` records in any `scan_status` state; configurable `owner_user_id` and `opportunity_id`
- `AISummaryFactory` — generates `client.ai_summaries` records with configurable `generated_at` offset for cache staleness tests

**Tooling:**

- `pytest` + `pytest-asyncio` for async FastAPI endpoint tests
- `testcontainers` (PostgreSQL + Redis) — shared session scope across integration test modules
- `fakeredis` for serial unit tests of UsageGate logic
- `respx` for mocking `httpx`-based AI Gateway SSE and sync calls
- `boto3` mock (`moto` library) for S3 presigned URL generation
- `freezegun` for ClamAV scan timeout tests and billing period TTL assertions
- `Vitest` + `React Testing Library` for frontend component tests
- `Playwright` for E2E tier-gate flow tests
- `asyncio.gather` for concurrency race condition tests (E06-P0-004, E06-P2-004)

**Environment:**

- `DATABASE_URL` pointing to testcontainers PostgreSQL with `pipeline` and `client` schema migrations applied
- `REDIS_URL` pointing to testcontainers Redis (for race tests); fakeredis for serial unit tests
- `AI_GATEWAY_BASE_URL` pointing to `respx` mock router
- `AWS_DEFAULT_REGION=eu-central-1` + moto S3 mock active
- `JWT_SECRET` configured for test JWT signing
- `CLAMAV_SCAN_TIMEOUT=900` (configurable in integration tests; overridden with short values in timeout tests)
- `AI_SUMMARY_CACHE_TTL_HOURS=24` (overridden in P2-007 stale cache test)

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions — tier enforcement is revenue-critical; any breach is a release blocker)
- **P1 pass rate:** ≥95% (failures must be triaged and accepted by PM + Security Lead before release)
- **P2/P3 pass rate:** ≥90% (informational; failures tracked as tech debt)
- **High-risk mitigations complete:** E06-R-001, E06-R-002, E06-R-003 all verified by P0 tests before release

### Coverage Targets

- **Endpoint/middleware layer (`client_api/routers/`, `client_api/middleware/`):** ≥80% line coverage
- **TierGate + UsageGate dependencies:** ≥90% line coverage (security-critical)
- **Frontend components (S06.09–S06.14):** ≥75% line coverage
- **Security scenarios (tier bypass, pre-scan download):** 100%

### Non-Negotiable Requirements

- [ ] All P0 tests pass before Sprint 6 close
- [ ] TierGate enforcement verified for all four tiers across listing, search, and detail endpoints
- [ ] UsageGate race condition test (E06-P0-004) passes with testcontainers Redis (not fakeredis)
- [ ] Document download returns 422 for `scan_status != clean` — verified in CI
- [ ] AI summary cached path does not decrement usage counter — verified in CI

---

## Mitigation Plans

### E06-R-001: TierGate Bypass (Score: 6)

**Mitigation Strategy:**
1. Apply `TierGate` as a FastAPI dependency on **every** route that returns opportunity data — not as middleware, but as a per-route dependency so it is impossible to accidentally omit from new routes
2. Response model selection (Free vs. Full) must happen inside the `TierGate` dependency, not in the route handler, so callers cannot override it
3. On each paginated continuation request, re-evaluate tier scope using the cursor's embedded tier-scope hash — invalidate cursors that don't match the current JWT tier
4. Tests E06-P0-001 (free-tier blocked from detail), E06-P0-002 (field-level enforcement), and E06-P0-003 (scope boundary enforcement) must all pass before any opportunity endpoint ships

**Owner:** Backend / QA
**Timeline:** Sprint 5
**Status:** Planned
**Verification:** E06-P0-001/002/003 and E06-P2-003 all passing; security code review gate for any route adding opportunity data

---

### E06-R-002: Redis Usage Counter Race Condition (Score: 6)

**Mitigation Strategy:**
1. Implement GET+INCR as a **single atomic Redis Lua script** or `MULTI/EXEC` pipeline — never two separate round-trips; the Lua script should: check current value, return 429 if ≥ limit, otherwise INCR and set EXPIRE atomically
2. Unit test with fakeredis verifies correct body/header schema (E06-P1-008/009)
3. Concurrency test with testcontainers Redis (E06-P0-004) fires two concurrent requests at limit−1 and verifies exactly one succeeds
4. Cache bypass (no decrement for cached summaries < 24h) tested in E06-P0-008

**Owner:** Backend / QA
**Timeline:** Sprint 5
**Status:** Planned
**Verification:** E06-P0-004 (concurrent race) and E06-P0-008 (cache no-decrement) passing in CI with real Redis

---

### E06-R-003: S3 Presigned URL / ClamAV Pre-Scan Download Gap (Score: 6)

**Mitigation Strategy:**
1. `scan_status` must be set to `pending` (not `clean` or null) immediately after the confirm callback — never default to clean
2. Download endpoint checks `scan_status == 'clean'` before generating presigned GET URL; returns 422 for `pending` and `infected` with distinct error messages
3. ClamAV infected callback must: (a) set `scan_status = infected`, (b) call S3 delete (soft-delete, object tag or actual delete based on retention policy), (c) never return a presigned URL for an infected document
4. Tests E06-P0-006 (pending blocks download), E06-P0-007 (infected → soft-delete), and E06-P2-006 (timeout → failed) must all pass

**Owner:** Backend / QA
**Timeline:** Sprint 5–6
**Status:** Planned
**Verification:** After E06-P0-006, no document with `scan_status != clean` is ever downloadable in any test scenario

---

### E06-R-004: SSE Connection Exhaustion (Score: 6)

**Mitigation Strategy:**
1. Define a configurable `MAX_CONCURRENT_SSE_STREAMS` env var; implement semaphore-based concurrency cap on the AI summary SSE endpoint
2. When cap is exceeded, return 503 with `Retry-After` header rather than queuing indefinitely
3. Route SSE endpoint to a dedicated uvicorn worker pool (or separate process) to prevent SSE load from blocking non-streaming endpoints
4. Performance test: 20 concurrent SSE requests; verify non-SSE endpoints respond within SLA; verify 503 behaviour when cap exceeded

**Owner:** Backend
**Timeline:** Sprint 6
**Status:** Planned
**Verification:** P3 load test (manual) showing non-SSE endpoint p95 latency unaffected during SSE burst

---

## Assumptions and Dependencies

### Assumptions

1. `subscription_tier` claim is present in all JWT tokens issued by E02; the claim is one of `free`, `starter`, `professional`, `enterprise`
2. `pipeline.opportunities` is a read-only data source from E06's perspective — the Client API service uses a read-only DB connection to the `pipeline` schema; no writes occur
3. `respx` mock is sufficient for AI Gateway SSE testing in CI — no real KraftData Executive Summary Agent calls are required for test execution
4. S3 presigned URL generation can be mocked via `moto` (or equivalent) in CI without real AWS credentials
5. ClamAV scan result is delivered via an HTTP callback to the API; the callback mechanism can be triggered synchronously in integration tests by calling the callback endpoint directly
6. `fakeredis` is acceptable for serial unit tests of UsageGate logic; testcontainers Redis is required only for concurrency race tests

### Dependencies

| Dependency | Required By | Status |
|------------|-------------|--------|
| E02 complete: Auth service with `subscription_tier` JWT claim for all four tiers | Sprint 5 start | Assumed complete |
| E03 complete: Next.js scaffold with app shell, routing, Zustand, TanStack Query | Sprint 5 start | Assumed complete |
| E05 complete: `pipeline.opportunities` + `pipeline.submission_guides` tables with test seed data | Sprint 5 start | Assumed complete (per Sprint 3–4 completion) |
| E04 complete: AI Gateway SSE proxy stable; `httpx` client importable | Sprint 5, S06.08 | Assumed complete |
| `client.documents` and `client.ai_summaries` DB migration applied | Sprint 5 start | Requires migration in this epic |

### Risks to Plan

- **Risk:** AI Gateway SSE endpoint undergoes schema changes post-E04 before E06 S06.08 is implemented
  - **Impact:** SSE event type assertions in E06-P0-009 and E06-P1-022 may need to be updated
  - **Contingency:** Pin respx mock to the E04 SSE contract version; treat SSE schema as a contract test boundary; escalate to E04 owner if schema changes

- **Risk:** ClamAV callback mechanism not finalized by Sprint 5 start
  - **Impact:** E06-P0-006, E06-P0-007, and E06-P2-006 cannot be executed until callback endpoint design is confirmed
  - **Contingency:** Use a stub callback mechanism for CI tests; defer real ClamAV integration to manual smoke test tier

---

## Interworking & Regression

| Service / Component | Impact | Regression Scope |
|--------------------|--------|-----------------|
| **E02 — Auth / JWT** | TierGate reads `subscription_tier` from JWT; UsageGate scopes Redis counter key to `user_id` from JWT | Verify JWT claim names unchanged; `subscription_tier` and `user_id` fields present in all issued tokens |
| **E03 — Frontend Shell** | Listing and detail pages built on E03 app shell layout, sidebar, top bar, design tokens, Zustand stores, and TanStack Query setup | Verify app shell navigation renders; route guards from E03 still apply before opportunity routes |
| **E05 — Data Pipeline** | `pipeline.opportunities` (source for all search/listing/detail data) and `pipeline.submission_guides` (rendered on detail page Submission Guide tab) — read-only from E06 perspective | Verify schema of `pipeline.opportunities` unchanged; `relevance_scores` JSONB, `evaluation_criteria`, `mandatory_documents` fields still present; `pipeline.submission_guides.steps` JSONB structure stable |
| **E04 — AI Gateway** | AI summary generation calls Executive Summary Agent via AI Gateway SSE proxy; E06 is a consumer of E04's `/execute/stream` endpoint | E04 SSE event schema (`delta`, `metadata`, `done`, `error`) unchanged; `agents.yaml` UUID for Executive Summary Agent still valid |
| **E11 — ESPD / Proposal** | Opportunity data and submission guides from E06 are used as proposal context; `pipeline.submission_guides` schema must remain stable | `steps` JSONB structure backward compatible; `reviewed` flag respected downstream |
| **E12 — Analytics** | Opportunity view events and AI summary usage events from E06 feed analytics dashboards | Event bus publish calls (if any) emit correct stream/topic names and payload schema; opportunity view counts not broken by tier-gating changes |

---

## Follow-on Workflows

- Run `/bmad-testarch-atdd` to generate failing P0 ATDD tests for TierGate enforcement and UsageGate atomicity (highest-value starting points)
- Run `/bmad-testarch-automate` to expand backend API test coverage beyond P0/P1 once E06 stories are implemented
- Run `/bmad-testarch-ci` to wire E06 backend and frontend test suites into the CI pipeline alongside E05

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework
- `probability-impact.md` — P×I scoring methodology
- `test-levels-framework.md` — Unit / Integration / Component / E2E level selection
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

- Epic: `eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md`
- System Architecture Test Design: `eusolicit-docs/test-artifacts/test-design-architecture.md`
- QA Test Design (System-Level): `eusolicit-docs/test-artifacts/test-design-qa.md`
- E05 Test Design (Data Source Dependency): `eusolicit-docs/test-artifacts/test-design-epic-05.md`
- E04 Test Design (AI Gateway Dependency): `eusolicit-docs/test-artifacts/test-design-epic-04.md`

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
