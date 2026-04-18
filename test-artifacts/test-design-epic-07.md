---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-17'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 7
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-06.md'
  - 'eusolicit-docs/test-artifacts/test-design-progress.md'
  - 'eusolicit/_bmad/bmm/config.yaml'
---

# Test Design: Epic 7 — Proposal Generation & Document Intelligence

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**Status:** Draft
**Epic:** E07 | **Sprint:** 7–8 | **Points:** 55 | **Dependencies:** E04, E06

---

## Executive Summary

**Scope:** Epic-level test design for E07 — Proposal Generation & Document Intelligence. Covers the full proposal workspace: proposal lifecycle CRUD and company-scoped RLS (S07.02), Alembic schema migrations for `proposals`, `proposal_versions`, and `content_blocks` tables (S07.01), section-based versioning with diff and rollback (S07.03), section-level auto-save PATCH and full-save PUT with optimistic locking (S07.04), AI-powered draft generation proxied through the AI Gateway as an SSE stream with `generation_status` guard (S07.05), Requirement Checklist Agent integration (S07.06), Compliance Checker / Clause Risk Analyzer / Scoring Simulator agent integrations (S07.07), Pricing Assistant and Win Theme Extractor agent integrations (S07.08), Content Blocks CRUD with PostgreSQL full-text search and approval workflow (S07.09), PDF and DOCX export with company branding and table of contents (S07.10), and the complete frontend experience: workspace page layout (S07.11), Tiptap rich-text section editor with auto-save and conflict reconciliation (S07.12), AI draft generation panel with SSE streaming and accept/discard actions (S07.13), Requirement Checklist and Compliance panels (S07.14), Scoring Simulator / Pricing / Win Themes panels (S07.15), and Version History / Content Blocks Library / Export Dialog components (S07.16).

This epic is **EU Solicit's flagship AI-powered capability** and the **Demo Milestone closer**. It integrates six AI agents through the AI Gateway, introduces multi-user collaborative editing semantics via optimistic locking, and produces legally significant output (tender proposals) that must accurately reflect user intent. Any data-loss bug in the save/version flow, any RLS bypass exposing a competitor's proposal content, or any SSE stream failure that loses generated content could directly damage a company's tender submission and constitute a business-critical defect. The export path produces documents that may be submitted to public procurement bodies, meaning format correctness is not cosmetic — it carries contractual weight.

**Risk Summary:**

- Total risks identified: 10
- High-priority risks (≥6): 3 (SSE stream persistence failure, auto-save optimistic locking race, RLS bypass)
- Critical categories: TECH (SSE persistence), DATA (auto-save race, rollback conflict, duplicate trigger), SEC (RLS bypass)

**Coverage Summary:**

- P0 scenarios: 11 (~25–40 hours)
- P1 scenarios: 31 (~35–50 hours)
- P2 scenarios: 15 (~12–20 hours)
- P3 scenarios: 5 (~3–6 hours)
- **Total effort:** ~75–116 hours (~2–3 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **KraftData AI agent internal logic** (Requirement Checklist, Compliance Checker, Clause Risk Analyzer, Scoring Simulator, Pricing Assistant, Win Theme Extractor) | E04 AI Gateway owns agent routing, model selection, and output quality; E07 only tests the request/response contract at the `POST /proposals/:id/*` endpoint boundary and the persistence of results on the proposal | Mock all AI Gateway responses at the `httpx` call boundary using `respx`; E04 integration tested separately |
| **AI Gateway SSE routing and model selection** | E04 owns the `/execute/stream` SSE endpoint and prompt construction for the Proposal Generator Workflow; E07 only verifies that section events streamed from the gateway are proxied correctly to the client and persisted | Use `respx` to mock a realistic SSE stream (multiple `delta`, `metadata`, `done` events); assert E07 proxy behavior, not E04 internals |
| **Billing / subscription tier enforcement for AI usage** | Billing epic and E06 UsageGate own metering; E07 AI calls go through the AI Gateway which is presumed to enforce quotas upstream; E07 only tests the consequences of a successful or failed gateway call | Seed test state with gateway mocks that return 429 to test E07 error handling for over-quota scenarios |
| **reportlab and python-docx rendering quality / visual fidelity** | PDF/DOCX visual layout, font rendering, and brand color accuracy are Design/Product concerns; E07 verifies structural correctness (sections present, TOC generated, file is a valid PDF/DOCX) | Assert section titles present in export; validate file magic bytes and Python parsability; visual review is manual |
| **S3 infrastructure availability for export downloads** | Platform concern; E07 verifies file generation and streaming response, not S3 durability or availability | Use in-memory file generation returning streaming `BytesIO` response in CI; no real S3 required for export tests |
| **Tiptap editor internals (ProseMirror schema, custom extensions)** | Tiptap is a third-party library; E07 tests the integration surface (section blocks render, auto-save fires, content block insertion works) — not ProseMirror schema correctness | Treat Tiptap as a black box; test observable DOM states and API call side-effects |
| **Real-time collaboration / multi-user concurrent editing** | E07 specifies single-user section editing with optimistic locking as a conflict signal; true collaborative CRDT or operational transform is out of scope for Sprint 7–8 | Test the two-user concurrent-edit scenario as an optimistic lock conflict detection test (P0-002), not a collaboration correctness test |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E07-R-001** | **TECH** | SSE draft generation stream loss on client disconnect — `POST /proposals/:id/generate` proxies the AI Gateway SSE stream back to the client; the S07.05 acceptance criterion specifies "on stream completion, persist the full generated content as a new proposal version"; if this persistence happens only after the client receives the `done` event (client-triggered), then any client disconnect (browser refresh, network drop, tab close) during generation will result in the generated content never being persisted; the user loses the entire AI-generated draft and must re-trigger generation at AI quota cost; the `generation_status` field would also remain stuck at `generating` permanently, blocking future generation attempts | 2 | 3 | **6** | Persist generated content server-side independent of client connection: buffer incoming AI Gateway SSE section chunks in memory (or Redis) on the backend; on AI Gateway `done` event, persist the complete content as a new `proposal_version` and reset `generation_status` to `completed` before the SSE response is closed toward the client; implement a cleanup job that resets `generation_status = failed` for proposals stuck in `generating` beyond a configurable timeout (e.g., 5 minutes); integration test: trigger generation, immediately close the client SSE connection, verify proposal version was persisted and `generation_status = completed` | Backend / QA | Sprint 7 |
| **E07-R-002** | **DATA** | Optimistic locking race condition on concurrent section auto-saves — S07.04 specifies optimistic locking via "content hash or version check" to prevent overwrites; if the implementation checks the content hash in a separate `SELECT` followed by a separate `UPDATE` (two round-trips), two concurrent PATCH requests for the same section can both read the same hash value (stale), both pass the check, and both write — last-write-wins with silent data loss for the earlier write; the 1.5s debounce in S07.12 means rapid typing across multiple section keys can produce concurrent PATCH bursts; additionally, if the hash is computed from only the target section's body (not the full JSONB), a hash collision between different section bodies could silently allow a write that overwrites concurrent sibling-section changes | 2 | 3 | **6** | Implement the hash check + update atomically in a single SQL statement: `UPDATE proposal_versions SET content = jsonb_set(content, '{sections,<key>,body}', :new_body) WHERE id = :vid AND md5(content::text) = :expected_hash RETURNING id`; check rows-affected = 1; if 0, return 409 with the full latest content for client reconciliation; the hash must cover the entire JSONB content object, not just the target section; integration test (asyncio.gather): fire two concurrent PATCHes to the same section from the same version hash; verify exactly one returns 200, the other returns 409 with `latest_content` body | Backend / QA | Sprint 7 |
| **E07-R-003** | **SEC** | Proposal RLS bypass via ID enumeration across company boundaries — `proposals` and `proposal_versions` are created with a `company_id` derived from the JWT; if any endpoint (detail GET, PATCH, version list, generate, checklist, compliance, scoring, pricing, win-themes, rollback) accepts `proposal_id` without enforcing `company_id = jwt.organization_id` at the DB layer, a malicious user with a valid JWT from Company A can access Company B's proposals by guessing or enumerating UUIDs; given that proposals contain commercially sensitive tender response content, this is a critical revenue and legal exposure beyond the general multi-tenancy risk; the `content_blocks` table has the same exposure vector | 2 | 3 | **6** | Apply company-scoped RLS on `proposals`, `proposal_versions`, and `content_blocks` tables identical to the E06 pattern; `company_id` must be derived exclusively from the JWT `organization_id` claim and injected as a DB session variable for RLS evaluation — never accepted from request body or path; all foreign-key traversals (proposal → versions → content → checklist → compliance results → etc.) must cascade the RLS check; integration test: generate proposal IDs for Company A, then call every proposal endpoint using Company B's JWT and assert 404 (not 403 — avoids confirming ID existence) | Backend / QA | Sprint 7 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E07-R-004 | DATA | `generation_status` duplicate trigger race — S07.05 specifies a `generation_status` field to prevent duplicate triggers; if two concurrent `POST /generate` requests both read `generation_status = idle` before either updates it, both will update to `generating` and both will invoke the AI Gateway stream simultaneously; this doubles AI Gateway cost, creates two competing versions, and may leave `generation_status` in an inconsistent state | 2 | 2 | 4 | Use a single atomic SQL update: `UPDATE proposals SET generation_status = 'generating' WHERE id = :id AND generation_status = 'idle' RETURNING id`; check rows-affected; return 409 if 0; integration test: two concurrent POST /generate requests fired simultaneously with asyncio.gather; verify exactly one succeeds, other returns 409 | Backend / QA |
| E07-R-005 | PERF | PDF/DOCX export blocking uvicorn event loop — reportlab and python-docx are CPU-bound synchronous operations; rendering a multi-section proposal with branding assets inline a uvicorn async worker blocks the event loop, degrading API latency for all concurrent users during export; large proposals (10+ sections with tables) may exceed the default 30s request timeout | 2 | 2 | 4 | Offload export generation to a Celery task or run in a `ProcessPoolExecutor` via `asyncio.run_in_executor`; return a streaming response from in-memory buffer (small proposals) or a job-polling endpoint (large proposals); performance test: concurrent export requests while measuring non-export API p95 latency | Backend / QA |
| E07-R-006 | DATA | Version rollback creating silent conflict with concurrent auto-save — `POST /proposals/:id/versions/:vid/rollback` creates a new version from a past version's content; if an auto-save PATCH fires concurrently during rollback, the new rollback version and the patched version may both be created with the same `version_number`, or `current_version_id` may point to the wrong version depending on commit ordering | 2 | 2 | 4 | Rollback must acquire a row-level advisory lock on the proposal row (or use `SELECT FOR UPDATE`) before creating the new version; test: fire rollback + concurrent PATCH simultaneously; verify only one new version created and `current_version_id` is consistent | Backend / QA |
| E07-R-007 | TECH | AI agent cascade failure with no per-endpoint timeout — S07.07 and S07.08 implement five separate AI agent calls (compliance, clause-risk, scoring, pricing, win-themes); if the AI Gateway is slow, overloaded, or returns a partial response, all five workspace panels may hang in indefinite loading states; no per-endpoint timeout is specified in the story acceptance criteria | 2 | 2 | 4 | Define a configurable `AI_AGENT_TIMEOUT_SECONDS` env var (default 30s); apply as `httpx.AsyncClient` timeout on all agent calls; on timeout, return 504 with `{"error": "agent_timeout", "retry_after": 30}`; frontend must render error state with "Retry" CTA; tests: mock AI Gateway with delayed response exceeding timeout; verify 504 returned within timeout window | Backend / QA |
| E07-R-008 | TECH | Tiptap editor content desync after partial SSE acceptance — S07.13 specifies "Accept Section" (individual sections) and "Accept All" actions during streaming; if a user accepts individual sections while other sections are still streaming, the editor local state may contain mixed accepted+streaming sections; on "Accept All" after partial accepts, the backend version created may not match the editor content if the frontend does not reconcile partial accepts before persisting | 2 | 2 | 4 | Disable "Accept Section" for individual sections while the SSE stream is in-flight (allow only after stream completes); enable "Accept All" only when all section events have been received; test: simulate partial SSE delivery, invoke Accept All, verify backend version content exactly matches all streamed sections | Frontend / QA |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E07-R-009 | DATA | tsvector not updated on content block body PATCH — if the `content_blocks.search_vector` tsvector column is populated only on INSERT (not on PATCH/UPDATE), searching after an edit returns stale results until the next full-text index refresh | 1 | 2 | 2 | Add DB trigger or application-level tsvector update on PATCH; test: create block, PATCH body with new keywords, search for new keywords, verify result returned |
| E07-R-010 | PERF | Export size denial-of-service — a proposal with many long sections and embedded tables could produce a very large file that exhausts memory or times out; no section count or content size limit is specified | 1 | 2 | 2 | Document: enforce max section count (e.g., 50) and max total content size (e.g., 500KB) at API validation; test: submit export request with oversized proposal (mock); verify 400 or graceful degradation |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- **E07-R-003** (RLS bypass) → extends system **R-01** (Multi-tenancy Isolation). Proposal content contains commercially sensitive tender response text — a cross-company data leak here has legal and competitive consequences beyond general tenant isolation.
- **E07-R-002** (auto-save race) → extends system **R-03** (AI Hallucinations). Data loss from optimistic locking failure corrupts the human-curated edits that correct or complement AI-generated content, making the final proposal unreliable.
- **E07-R-001** (SSE stream persistence failure) → extends system **R-05** (Tender Sync Reliability). The Proposal Generator Workflow SSE stream is an AI-driven event pipeline; persistence failure on disconnect is analogous to event loss in the Kafka consumer pattern (R-05 mitigation: idempotent consumers).
- **E07-R-007** (AI agent timeout cascade) → extends system **R-03** (AI Hallucinations). Uncontrolled AI Gateway timeouts degrade user trust in AI-generated content and may cause users to submit unvalidated proposals.

---

## Entry Criteria

- [ ] E04 complete: AI Gateway operational in Docker Compose; `POST /execute/stream` SSE endpoint stable; `httpx`-based client importable; agent IDs for Proposal Generator Workflow, Requirement Checklist, Compliance Checker, Clause Risk Analyzer, Scoring Simulator, Pricing Assistant, and Win Theme Extractor registered in `agents.yaml`
- [ ] E06 complete: Opportunity discovery API operational; `pipeline.opportunities` and `client.documents` tables available; `GET /opportunities/:id` returns opportunity data usable as proposal input
- [ ] E02 complete: Auth service with JWT issuance including `organization_id` and `user_id` claims; JWT fixtures available for test users in at least two distinct companies
- [ ] E03 complete: Next.js frontend scaffold with app shell, routing, Zustand stores, TanStack Query, and Tiptap dependency installed
- [ ] Database migrations applied: `client.proposals`, `client.proposal_versions`, `client.content_blocks`, and `client.proposal_checklists` tables created with all specified indexes and RLS policies
- [ ] Alembic migration reversibility verified: `alembic upgrade head` then `alembic downgrade -1` completes without error
- [ ] `respx` mock library available for mocking AI Gateway SSE and sync agent calls in CI
- [ ] CI environment: PostgreSQL testcontainers available; no real AI Gateway calls required for CI execution

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95%; failures triaged with PM sign-off before demo milestone)
- [ ] No open high-severity bugs in proposal RLS enforcement, auto-save optimistic locking, or SSE persistence
- [ ] Line coverage ≥80% on `services/client_api/routers/` (proposal, version, content, agent-integration endpoints) and `services/client_api/services/proposal_*.py`
- [ ] Component test coverage ≥75% on frontend proposal workspace components (S07.11–S07.16)
- [ ] All 12+ backend proposal endpoints have OpenAPI documentation
- [ ] Integration test suite (backend) passes in CI within 8 minutes
- [ ] PDF and DOCX exports produce valid, parsable files verified in CI (magic bytes + Python parsing)
- [ ] No proposal or content block accessible across company boundaries in any test scenario
- [ ] `generation_status` never permanently stuck in `generating` (cleanup job tested)

---

## Test Coverage Plan

> **Note:** P0/P1/P2/P3 denote **priority and risk level**, NOT execution timing. Execution scheduling is defined separately in the Execution Strategy section.

### P0 (Critical)

**Criteria:** Blocks core proposal creation/generation + High risk (≥6) + No safe workaround (data loss, security, or demo-blocking impact)

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E07-P0-001 | SSE stream persistence on client disconnect — trigger `POST /proposals/:id/generate`, mock AI Gateway to stream 3 section events + `done`; close the client SSE connection immediately after the first section event; verify proposal version was created server-side with all 3 sections' content; verify `generation_status = completed` | S07.05 | Integration | E07-R-001 | Use `httpx.AsyncClient` with `respx` mock; close response early; assert DB state independent of client receipt |
| E07-P0-002 | `generation_status` stuck cleanup — trigger generate, mock AI Gateway to never send `done` event (timeout); advance clock past cleanup threshold (5 min); verify background job resets `generation_status = failed`; verify subsequent generate call succeeds | S07.05 | Integration | E07-R-001 | Use `freezegun`; assert `generation_status` transitions idle → generating → failed |
| E07-P0-003 | Concurrent section PATCH optimistic lock race — acquire current content hash; fire two concurrent `PATCH /proposals/:id/content/sections/:key` requests with the same hash using `asyncio.gather`; verify exactly one returns 200 and exactly one returns 409; verify 409 response body contains `latest_content` for reconciliation; verify DB has only one write | S07.04 | Integration | E07-R-002 | Use testcontainers PostgreSQL; assert rows affected count; verify no silent data loss |
| E07-P0-004 | Stale hash full-save conflict — submit `PUT /proposals/:id/content` with an outdated content hash (proposal was auto-saved by concurrent PATCH after hash was read); verify 409 with latest content; verify DB content is not overwritten with stale full-save payload | S07.04 | Integration | E07-R-002 | Seed proposal with version N; PATCH section externally to advance to version N+1; PUT with hash from version N; assert 409 |
| E07-P0-005 | Proposal RLS cross-company isolation — create proposals for Company A; authenticate as Company B user; call `GET /proposals/:a_id`, `PATCH /proposals/:a_id`, `DELETE /proposals/:a_id`, `GET /proposals/:a_id/versions`, and `POST /proposals/:a_id/generate` with Company B's JWT; verify all return 404 | S07.02, S07.03, S07.05 | Integration | E07-R-003 | Parameterized over 5 endpoint types; assert 404 not 403 to avoid ID confirmation; use two distinct DB-seeded companies |
| E07-P0-006 | Content blocks RLS cross-company isolation — create content blocks for Company A; authenticate as Company B user; call `GET /content-blocks/:a_block_id`, `PATCH /content-blocks/:a_block_id`, and `GET /content-blocks/search?q=<a_block_title>` with Company B's JWT; verify 404 for direct access and zero results for search | S07.09 | Integration | E07-R-003 | Search must not leak even partial block titles or IDs across companies |
| E07-P0-007 | Duplicate trigger prevention — two concurrent `POST /proposals/:id/generate` requests fired via `asyncio.gather` when `generation_status = idle`; verify exactly one returns 200/202 and begins streaming; the other returns 409 with `{"error": "generation_in_progress"}`; verify only one AI Gateway SSE invocation was made (assert via respx call count) | S07.05 | Integration | E07-R-004 | Confirms atomic status guard prevents dual invocation and double AI cost |
| E07-P0-008 | Version rollback creates new version from target content — create 3 versions; rollback to version 1; verify a new version 4 is created with version 1's content; verify `current_version_id` on the proposal points to version 4; verify versions 2 and 3 still exist (no deletion) | S07.03 | API | — | Core version integrity test; verifies rollback does not mutate historical versions |
| E07-P0-009 | Section-level diff accuracy — create version A with sections `{intro: "Hello", body: "World"}`; create version B with sections `{intro: "Hi", body: "World", conclusion: "End"}`; call `GET /proposals/:id/versions/diff?from=A&to=B`; verify diff shows `intro` as `changed` with before/after, `body` as `unchanged`, and `conclusion` as `added` | S07.03 | API | — | Diff correctness is core UX for version history; incorrect diff misdirects user review |
| E07-P0-010 | PDF export produces valid parsable file with section headers and TOC — `POST /proposals/:id/export` with `format: pdf`; verify response `Content-Type: application/pdf`; verify `Content-Disposition: attachment` header; parse response body with `PyPDF2` or `pypdf`; verify at least one page; verify each section title appears in the text layer; verify table of contents page is present | S07.10 | Integration | — | Export correctness is demo-critical and has contractual weight for tender submissions |
| E07-P0-011 | End-to-end proposal workspace — E2E: create proposal from opportunity; click "Generate Draft"; verify SSE progress panel shows section checkmarks as sections complete; click "Accept All"; verify editor renders all sections; open Version History; verify version 2 appears with change_summary; click Export → PDF → Download; verify download triggered | S07.11, S07.13, S07.16 | E2E | E07-R-001 | Playwright test covering the demo-critical path end-to-end; mocked AI Gateway SSE in Playwright network intercept |

**Total P0: 11 tests, ~25–40 hours**

---

### P1 (High)

**Criteria:** Critical feature paths + Medium/high risk + Common execution paths across all stories

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E07-P1-001 | Proposal creation from opportunity — `POST /proposals` with valid `opportunity_id`; verify proposal created with `status = draft`, `current_version_id` pointing to first empty version, `company_id` derived from JWT (not request body) | S07.02 | API | E07-R-003 | Verify `company_id` never accepted from body |
| E07-P1-002 | Proposal list filtering — `GET /proposals?opportunity_id=X&status=draft` returns matching proposals; free-form list without filters returns all company proposals paginated | S07.02 | API | — | Verify pagination and filter correctness |
| E07-P1-003 | Proposal detail includes current version content — `GET /proposals/:id` returns joined current version sections; verify `title`, `status`, `current_version_id`, and `content.sections[]` present | S07.02 | API | — | Verify JOIN is performed, not a separate fetch |
| E07-P1-004 | Proposal soft-archive — `DELETE /proposals/:id` sets `status = archived`; subsequent `GET /proposals` does not return archived proposal by default; GET with `status=archived` filter returns it | S07.02 | API | — | Verify soft-delete semantics; data not physically deleted |
| E07-P1-005 | Version creation on snapshot — `POST /proposals/:id/versions` with optional `change_summary`; verify new version has `version_number = previous + 1`; verify content copied from current version; verify `change_summary` stored | S07.03 | API | — | Core versioning contract |
| E07-P1-006 | Version list sorted newest first — create 3 versions; `GET /proposals/:id/versions` returns them newest first; verify `version_number` sequence descending | S07.03 | API | — | Sort order is required by spec |
| E07-P1-007 | Section auto-save updates only target section key — PATCH section key `body` with new content; GET proposal versions; verify `intro` section unchanged; verify `body` section updated | S07.04 | API | E07-R-002 | Confirms JSONB partial update; other sections intact |
| E07-P1-008 | Auto-save updates `updated_at` on proposal — successful PATCH advances `updated_at` timestamp on the proposal row | S07.04 | API | — | Timestamp hygiene for staleness detection |
| E07-P1-009 | SSE draft generation streams section events — `POST /proposals/:id/generate`; mock AI Gateway SSE with 3 section events (key, title, body chunk) + `metadata` + `done`; verify client receives all events in order; verify proposal version created after `done` event; verify `generation_status = completed` | S07.05 | Integration | E07-R-001 | Nominal streaming path; complements P0-001 which tests disconnect |
| E07-P1-010 | AI Gateway error propagates as SSE error event — mock AI Gateway returning 500; verify SSE `error` event delivered to client; verify `generation_status = failed`; verify no partial version created | S07.05 | Integration | — | Error path must not leave orphaned state |
| E07-P1-011 | Requirement Checklist Agent generate, retrieve, toggle — `POST /proposals/:id/checklist/generate`; mock agent response with 3 checklist items; verify items stored; `GET /proposals/:id/checklist` returns structured items; `PATCH .../items/:itemId` with `{is_checked: true}` persists state | S07.06 | Integration | — | Three operations in one integration test; mock respx for agent call |
| E07-P1-012 | Compliance check persists per-criterion results — `POST /proposals/:id/compliance-check`; mock agent response with 2 pass + 1 fail criteria; verify stored on proposal; `GET /proposals/:id/compliance-check` returns same results | S07.07 | Integration | — | Verify persistence shape; mock AI Gateway |
| E07-P1-013 | Clause risk analyzer returns flagged clauses — `POST /proposals/:id/clause-risk`; mock agent response with 2 high-risk clauses; verify stored; GET returns correct risk levels (low/medium/high) | S07.07 | Integration | — | Risk level enum validation |
| E07-P1-014 | Scoring simulation returns per-criterion scorecard — `POST /proposals/:id/scoring-simulation`; mock response with 3 criteria (score, max_score, suggestion); verify stored and retrievable | S07.07 | Integration | — | Scorecard shape validation |
| E07-P1-015 | Pricing assistant returns recommendation with market range — `POST /proposals/:id/pricing-assist`; mock response with recommended_price, market_range (min/median/max), justification; verify stored | S07.08 | Integration | — | Data shape validation for pricing result |
| E07-P1-016 | Win themes returns ranked list — `POST /proposals/:id/win-themes`; mock response with 3 ranked themes (title, description); verify stored and retrievable in rank order | S07.08 | Integration | — | Rank order preservation |
| E07-P1-017 | Content block CRUD lifecycle — create with title/category/body/tags; GET by ID; PATCH body (verify version incremented); approve (verify `approved_at` + `approved_by` populated); unapprove (verify fields cleared); DELETE | S07.09 | API | — | Full CRUD + approval workflow in sequence |
| E07-P1-018 | Content block full-text search — create 3 blocks with distinct keywords; search for keyword unique to one block; verify only that block returned; verify relevance ordering with multiple hits | S07.09 | API | — | tsvector search correctness |
| E07-P1-019 | DOCX export produces valid parsable file — `POST /proposals/:id/export` with `format: docx`; verify `Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document`; parse with `python-docx`; verify section titles present as headings | S07.10 | Integration | — | DOCX validity companion to P0-010 (PDF) |
| E07-P1-020 | Export includes company branding and section headers — verify company logo reference / color class present in exported PDF/DOCX structure; verify each section has a heading with the section title; verify page numbers present (PDF) | S07.10 | Integration | — | Branding correctness for demo; use seeded company profile with logo |
| E07-P1-021 | Proposal workspace page renders layout with all panels — navigate to `/proposals/:id`; verify Tiptap editor center area renders; verify left panel (checklist, content blocks) toggle works; verify right panel (AI, compliance, scoring, pricing, win themes) toggle works; verify toolbar with title, status badge, version indicator visible | S07.11 | Component | — | React Testing Library; verify DOM elements for each panel zone |
| E07-P1-022 | Tiptap section blocks render with section headers — load proposal with 2 sections; verify each section renders as a distinct block with a heading; verify formatting toolbar visible (bold, italic, heading buttons) | S07.12 | Component | — | Section-block rendering correctness |
| E07-P1-023 | Auto-save fires after 1.5s debounce and shows save indicator — simulate typing in section; advance timer 1.5s; verify PATCH API call fired; verify save indicator transitions: typing → saving → saved | S07.12 | Component | — | Debounce timer + indicator state machine |
| E07-P1-024 | Conflict reconciliation dialog shown on 409 — mock PATCH returning 409 with `latest_content`; verify reconciliation dialog appears with "Overwrite" / "Discard my changes" options | S07.12 | Component | E07-R-002 | Frontend conflict handling for E07-R-002 mitigation |
| E07-P1-025 | SSE streaming panel renders sections progressively — mock SSE EventSource with 3 section events; verify each section appears in editor as events arrive; verify progress indicator shows section checkmarks completing | S07.13 | Component | E07-R-001 | Progressive rendering UX; no Accept until done (per E07-R-008 mitigation) |
| E07-P1-026 | Accept All and Discard actions behave correctly — after SSE stream completes: "Accept All" fires version creation API call with complete content; "Discard" clears streamed content from editor without API call | S07.13 | Component | — | Action correctness; assert API call count |
| E07-P1-027 | Requirement checklist panel renders items grouped by category with completion progress bar — mock checklist API; verify items grouped; check one item (PATCH fires); verify progress bar updates | S07.14 | Component | — | Checklist interaction + optimistic update |
| E07-P1-028 | Compliance panel shows pass/fail per criterion with expandable detail — mock compliance API response; verify pass items show green indicator; fail items show red; expand detail text on click | S07.14 | Component | — | Compliance results rendering |
| E07-P1-029 | Scoring simulator panel shows radar chart and table — mock scoring API; verify Recharts radar chart rendered; verify table rows for each criterion with score/max/suggestion; verify criteria below 70% highlighted amber | S07.15 | Component | — | Recharts integration; threshold highlighting |
| E07-P1-030 | Version history sidebar lists versions and supports side-by-side diff view — mock version list API; verify each version shows number, date, author; click "Compare"; verify diff viewer renders with insertions (green) and deletions (red) | S07.16 | Component | — | Diff viewer rendering correctness |
| E07-P1-031 | Content blocks library modal: search and insert at cursor — open library modal; type search query (mock real-time search API); verify results render with title, preview, approval badge; click "Insert"; verify block body inserted at cursor position in Tiptap editor | S07.16 | Component | — | Full content block insertion flow |

**Total P1: 31 tests, ~35–50 hours**

---

### P2 (Medium)

**Criteria:** Secondary flows + Low/medium risk (1–4) + Edge cases and configuration validation

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E07-P2-001 | Proposal not found returns 404 — GET/PATCH/DELETE on non-existent or archived proposal UUID returns 404 | S07.02 | API | — | Edge case: soft-deleted proposals must also return 404 |
| E07-P2-002 | Version number is sequential and non-repeating — create 5 versions; verify version_numbers are 1, 2, 3, 4, 5 (no gaps, no duplicates); rollback creates version 6 | S07.03 | API | — | Sequence integrity |
| E07-P2-003 | Diff on identical versions returns all-unchanged — diff two versions with identical content; verify all sections are `unchanged` | S07.03 | API | — | Edge case for diff logic |
| E07-P2-004 | Concurrent rollback + auto-save produces consistent state — fire rollback and PATCH concurrently via asyncio.gather; verify exactly one new version created; verify `current_version_id` points to the winner | S07.03, S07.04 | Integration | E07-R-006 | Race condition mitigation test for E07-R-006 |
| E07-P2-005 | AI Gateway timeout returns 504 with retry-after — mock AI Gateway with 31s response delay; verify `POST /proposals/:id/compliance-check` returns 504 within timeout window; verify `{"error": "agent_timeout", "retry_after": 30}` body | S07.07 | Integration | E07-R-007 | Timeout behavior for all five agent endpoints (compliance used as representative) |
| E07-P2-006 | tsvector updated after content block body PATCH — create block with keyword "procurement"; PATCH body replacing keyword with "sourcing"; search "procurement"; verify old block not returned; search "sourcing"; verify block returned | S07.09 | Integration | E07-R-009 | tsvector freshness after update |
| E07-P2-007 | Content block pagination and tag/category filtering — seed 10 blocks; GET with `?category=legal` returns only legal-category blocks; GET with `?tags=EU,grants` returns only blocks with both tags; verify pagination `limit` and `offset` params work | S07.09 | API | — | Filter correctness |
| E07-P2-008 | Export rejects invalid format parameter — `POST /proposals/:id/export` with `format: xml`; verify 422 with validation error | S07.10 | API | — | Input validation |
| E07-P2-009 | Export handles empty proposal gracefully — export a proposal with no sections; verify 200 with a valid (but minimal) PDF/DOCX file, not a 500 | S07.10 | Integration | — | Edge case: first-time export before draft generation |
| E07-P2-010 | Pricing panel renders market range bar and recommended price — mock pricing API; verify recommended price rendered as large-text element; verify min/median/max bar rendered; verify justification paragraph present | S07.15 | Component | — | Pricing visualization correctness |
| E07-P2-011 | Win themes panel renders ranked cards; drag-to-reorder changes priority — mock win themes API; verify 3 theme cards in rank order; simulate drag reorder; verify card order changes in DOM | S07.15 | Component | — | Drag-and-drop interaction test |
| E07-P2-012 | Export dialog format selector and download trigger — open export dialog; select DOCX; verify DOCX icon highlighted; click "Download"; verify export API called with `format: docx`; verify progress indicator shown during download | S07.16 | Component | — | Export dialog interaction |
| E07-P2-013 | Rollback confirmation dialog prevents accidental rollback — click "Rollback" in version history; verify confirmation dialog appears; click Cancel; verify no rollback API called; click Rollback again → Confirm; verify API called | S07.16 | Component | — | Destructive action guard |
| E07-P2-014 | Proposal workspace toolbar inline title edit — click proposal title in toolbar; verify input field appears; type new title; press Enter; verify PATCH /proposals/:id called with updated title | S07.11 | Component | — | Inline edit UX |
| E07-P2-015 | Generation in-progress state disables editor — during active SSE stream (mock delayed stream); verify editor `contenteditable=false`; verify "Generate" button is disabled; verify "Stop" button is present and wired to cancel | S07.13 | Component | — | UX guard during generation |

**Total P2: 15 tests, ~12–20 hours**

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Benchmarks and configuration correctness

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E07-P3-001 | OpenAPI schema coverage — all proposal, version, content-save, agent-integration, content-block, and export endpoints appear in `/openapi.json` with correct request/response schemas | S07.01–S07.10 | Unit | — | `fastapi.testclient` schema assertion |
| E07-P3-002 | Alembic migration reversibility — `alembic upgrade head` creates all 3 tables with indexes; `alembic downgrade -1` drops them cleanly; re-running `upgrade head` succeeds | S07.01 | Unit | — | Migration hygiene |
| E07-P3-003 | Content block search relevance ordering — seed 5 blocks where block A contains query keyword 3× and block B contains it 1×; verify block A ranked before block B in tsvector search results | S07.09 | API | — | Search relevance correctness |
| E07-P3-004 | Loading skeletons and error states render in all AI panels — simulate loading state (TanStack Query `isLoading: true`); verify skeleton components visible in checklist, compliance, scoring, pricing, win-themes panels; simulate API error; verify error CTA rendered | S07.14, S07.15 | Component | — | UX completeness; all panels symmetric |
| E07-P3-005 | Inline diff toggle (side-by-side vs. inline) in version history — click "Compare"; verify side-by-side view renders; toggle to "Inline"; verify single-column diff renders with `+/-` markers; toggle back; verify side-by-side re-rendered | S07.16 | Component | — | Diff viewer toggle interaction |

**Total P3: 5 tests, ~3–6 hours**

---

## Execution Strategy

**Philosophy:** Backend API and integration tests run on every PR; frontend component tests run on every PR if suite completes within 10 minutes. E2E tests (Playwright) run nightly due to browser startup overhead. PDF/DOCX structural validation runs in CI using Python parsing libraries (no visual rendering). Export performance tests are manual only.

| When | What | Expected Duration |
|------|------|------------------|
| **Every PR** | All P0, P1, P2 API + integration + unit tests (pytest with testcontainers PostgreSQL + respx AI Gateway mocks) | ~6–10 minutes |
| **Every PR** | All P1, P2, P3 frontend component tests (Vitest + React Testing Library) | ~4–6 minutes |
| **Nightly** | P0 E2E test (E07-P0-011 — Playwright proposal workspace end-to-end) | ~10–15 minutes |
| **Nightly** | P1 E2E component smoke (key panel interactions via Playwright) | ~10–20 minutes |
| **Performance (Manual)** | Concurrent export load test (CPU-bound blocking test) + API p95 latency baseline | ~30–45 minutes |
| **Manual / Pre-demo** | Full visual review of exported PDF/DOCX with company branding | ~15–30 minutes |

---

## Resource Estimates

| Priority | Test Count | Estimated Effort | Notes |
|----------|------------|-----------------|-------|
| P0 | 11 | ~25–40 hours | Mix of integration (SSE, concurrent race), API, and E2E; testcontainers PostgreSQL for race tests; respx for AI Gateway mocks |
| P1 | 31 | ~35–50 hours | Heavy integration coverage (8 agent endpoints × mock+persist), API CRUD, and component tests for all 6 frontend stories |
| P2 | 15 | ~12–20 hours | Edge cases, concurrency variants, secondary UI flows |
| P3 | 5 | ~3–6 hours | Schema checks, migration test, search relevance, UX completeness |
| **Total** | **62** | **~75–116 hours** | **~2–3 weeks, 1 QA** |

### Prerequisites

**Test Fixtures and Factories:**

- `ProposalFactory` — generates `client.proposals` records with configurable `company_id`, `status` (draft/active/archived), `current_version_id`, and associated first version
- `ProposalVersionFactory` — generates `client.proposal_versions` records with configurable `version_number`, JSONB `content` (sections array), `change_summary`, and `created_by`
- `ContentBlockFactory` — generates `client.content_blocks` records with configurable `category`, `tags[]`, `body`, and `approval_status`
- `CompanyJWTFactory` — generates signed JWT tokens with `organization_id` and `user_id` claims for multiple distinct companies; extends `UserJWTFactory` from E06 with company-scoped claims

**Tooling:**

- `pytest` + `pytest-asyncio` for async FastAPI endpoint tests
- `testcontainers` (PostgreSQL) — session scope for integration and concurrency tests
- `respx` for mocking all `httpx`-based AI Gateway calls (SSE streaming and sync agent calls)
- `asyncio.gather` for concurrency race condition tests (E07-P0-003, E07-P0-004, E07-P0-007, E07-P2-004)
- `freezegun` for `generation_status` cleanup job timeout tests (E07-P0-002)
- `PyPDF2` or `pypdf` for PDF structural validation (E07-P0-010)
- `python-docx` for DOCX structural validation (E07-P1-019)
- `Vitest` + `React Testing Library` for frontend component tests
- `Playwright` for E2E workspace test (E07-P0-011)

**Environment:**

- `DATABASE_URL` pointing to testcontainers PostgreSQL with `client` schema migrations applied (proposals, versions, content_blocks, checklists)
- `AI_GATEWAY_BASE_URL` pointing to `respx` mock router for all AI agent calls
- `JWT_SECRET` configured for test JWT signing
- `AI_AGENT_TIMEOUT_SECONDS=30` (overridden in E07-P2-005 to `1` to test timeout path)
- `GENERATION_CLEANUP_TIMEOUT_SECONDS=300` (overridden in E07-P0-002 to `5` for fast clock-advance test)
- `EXPORT_MAX_SECTIONS=50` and `EXPORT_MAX_CONTENT_BYTES=524288` (configurable size limits)

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions — data integrity in proposal save/version and RLS enforcement are demo-critical blockers)
- **P1 pass rate:** ≥95% (failures must be triaged and accepted by PM before Demo Milestone)
- **P2/P3 pass rate:** ≥90% (informational; failures tracked as tech debt)
- **High-risk mitigations complete:** E07-R-001, E07-R-002, E07-R-003 all verified by P0 tests before demo

### Coverage Targets

- **Proposal/version/content endpoint routers (`client_api/routers/proposal*.py`):** ≥80% line coverage
- **Optimistic locking service layer:** ≥90% line coverage (data-integrity critical)
- **Frontend proposal workspace components (S07.11–S07.16):** ≥75% line coverage
- **RLS enforcement (all proposal and content-block endpoints):** 100% cross-company scenario coverage

### Non-Negotiable Requirements

- [ ] All P0 tests pass before Sprint 8 close (Demo Milestone)
- [ ] SSE persistence test (E07-P0-001) passes with server-side version creation verified independent of client connection
- [ ] Optimistic locking race test (E07-P0-003) passes with testcontainers PostgreSQL (not SQLite or mock)
- [ ] RLS cross-company test (E07-P0-005) verifies 404 (not 403) for all 5 endpoint types
- [ ] PDF export validated as parsable file with section titles in CI (E07-P0-010 automated)
- [ ] `generation_status` cleanup tested (E07-P0-002) — no permanently stuck proposals in any test scenario

---

## Mitigation Plans

### E07-R-001: SSE Draft Generation Stream Loss (Score: 6)

**Mitigation Strategy:**
1. Persist generated content **server-side**, triggered by the AI Gateway `done` event — never by client acknowledgment; the SSE response to the client is a read-only stream, not the persistence trigger
2. Buffer incoming AI Gateway section chunks (in memory or a Redis key scoped to `proposal_id`) as they arrive; flush to `proposal_versions` on `done` event regardless of client connection state
3. Implement a background cleanup job (`generate_status_cleanup`) that sets `generation_status = failed` for any proposal stuck in `generating` beyond `GENERATION_CLEANUP_TIMEOUT_SECONDS`; job runs every 60 seconds
4. Tests E07-P0-001 (disconnect mid-stream), E07-P0-002 (cleanup job), and E07-P1-009 (nominal path) must all pass before S07.05 ships

**Owner:** Backend / QA
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** E07-P0-001 (disconnect → version persisted) and E07-P0-002 (timeout → `failed`) passing in CI

---

### E07-R-002: Optimistic Locking Race Condition (Score: 6)

**Mitigation Strategy:**
1. Implement section PATCH as a **single atomic SQL statement**: `UPDATE proposal_versions SET content = jsonb_set(content, '{sections,<idx>,body}', :new_body) WHERE proposal_id = :pid AND md5(content::text) = :expected_hash RETURNING id`; never split into SELECT + UPDATE
2. The hash must cover the **entire JSONB content** (all sections), not just the target section, to prevent collisions where different section combinations produce identical hashes
3. On `rows_affected = 0`, return HTTP 409 with `{"error": "version_conflict", "latest_content": <current_content>}` so the frontend can invoke the reconciliation dialog (S07.12)
4. Full-save PUT uses the same atomic hash-check pattern on the entire content object
5. Tests E07-P0-003 (concurrent PATCH race), E07-P0-004 (stale-hash full-save), and E07-P1-007 (nominal PATCH) must all pass

**Owner:** Backend / QA
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** E07-P0-003 (concurrent race) passing with testcontainers PostgreSQL; zero silent data-loss scenarios in any test

---

### E07-R-003: Proposal RLS Cross-Company Bypass (Score: 6)

**Mitigation Strategy:**
1. Apply PostgreSQL Row Level Security policy on `client.proposals`, `client.proposal_versions`, and `client.content_blocks` identical in structure to the E06 `client.documents` RLS pattern: `USING (company_id = current_setting('app.company_id')::uuid)`
2. `company_id` must be derived **exclusively** from the JWT `organization_id` claim and injected as a DB session variable via FastAPI middleware — never accepted from request body, path params, or query params
3. All endpoints that traverse proposal FK chains (versions → checklists → compliance results → agent results) must use DB connections with the RLS session variable set; never use a superuser connection for proposal reads
4. Return **404** (not 403) for cross-company access to avoid confirming ID existence to a potential attacker
5. Test E07-P0-005 (5 proposal endpoint types) and E07-P0-006 (content blocks) must pass with two distinct companies seeded in testcontainers PostgreSQL

**Owner:** Backend / QA
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** E07-P0-005 and E07-P0-006 both passing; security code review gate for any new proposal endpoint added post-sprint

---

## Assumptions and Dependencies

### Assumptions

1. All AI agent calls (checklist, compliance, clause-risk, scoring, pricing, win-themes, proposal generator) go through the AI Gateway `POST /execute` or `POST /execute/stream` endpoints; no direct model API calls from the proposal service
2. `respx` mock is sufficient for all AI Gateway calls in CI — no real KraftData agent calls required for any automated test
3. The optimistic locking hash is computed server-side from the JSONB content at the time of the PATCH request (not client-computed); the client sends the hash it last received from the server
4. `python-docx` and `PyPDF2`/`pypdf` are available in the CI test environment as dev dependencies
5. Company branding (logo URL, color scheme) is available on the `company_profile` record seeded by the E02 test fixtures
6. The `generation_status` field cleanup is implemented as a periodic Celery Beat task (not a cron outside the app process), so it can be triggered deterministically in tests via direct Celery task invocation

### Dependencies

| Dependency | Required By | Status |
|------------|-------------|--------|
| E04 complete: AI Gateway SSE proxy stable; all 7 agent UUIDs registered in `agents.yaml` | Sprint 7 start | Assumed complete (per E04 test design) |
| E06 complete: `GET /api/v1/opportunities/:id` returns opportunity data with requirements for proposal input | Sprint 7 start | Assumed complete (per E06 test design) |
| E02 complete: JWT with `organization_id` claim for company-scoped RLS; at least 2 test companies seeded | Sprint 7 start | Assumed complete (per E02 test design) |
| E03 complete: Next.js scaffold with Zustand, TanStack Query, and Tiptap dependency (`@tiptap/react`) installed | Sprint 7 start | Assumed complete (per E03 test design) |
| Alembic migration applied: all 3 tables with RLS policies and indexes live in CI DB | Sprint 7 start (S07.01) | Requires implementation in this epic |

### Risks to Plan

- **Risk:** AI Gateway SSE event schema (`section_key`, `title`, `body_chunk` fields in delta events) changes post-E04 before S07.05 is implemented
  - **Impact:** E07-P0-001, E07-P0-009, and E07-P1-009 SSE event assertions may need updating
  - **Contingency:** Pin `respx` mock to the E04 SSE contract version; treat as a contract test boundary; escalate to E04 owner if schema changes

- **Risk:** reportlab or python-docx CPU-bound export blocks async worker before E07-R-005 mitigation (Celery offload) is implemented
  - **Impact:** Export API tests may time out in CI if a large test proposal is used
  - **Contingency:** Use a minimal 3-section test proposal for CI export tests; flag large-proposal performance test as manual-only

---

## Interworking & Regression

| Service / Component | Impact | Regression Scope |
|--------------------|--------|-----------------|
| **E02 — Auth / JWT** | Proposal and content-block RLS derives `company_id` from JWT `organization_id` claim; version author stored as `user_id` from JWT | Verify `organization_id` and `user_id` fields present in all issued tokens; JWT claim names unchanged from E02 |
| **E03 — Frontend Shell** | Proposal workspace page (`/proposals/:id`) is built on E03 app shell layout, routing, Zustand stores, and TanStack Query | Verify app shell navigation renders; route guard from E03 applies before `/proposals` route; Zustand store namespacing does not conflict with new proposal stores |
| **E04 — AI Gateway** | All 7 AI agent calls (draft generation, checklist, compliance, clause-risk, scoring, pricing, win-themes) route through AI Gateway; E07 is a heavy consumer of E04 SSE and sync endpoints | E04 SSE event schema unchanged (`delta`, `metadata`, `done`, `error`); `agents.yaml` UUIDs for all 7 agents still valid; AI Gateway concurrency limits unchanged (affects E07-R-001 scenarios) |
| **E06 — Opportunity Discovery** | `GET /opportunities/:id` provides the opportunity requirements and company profile used as input to the Proposal Generator Workflow; `POST /proposals` accepts `opportunity_id` FK | E06 opportunity detail endpoint schema stable; `requirements`, `evaluation_criteria`, and `submission_deadline` fields present; `pipeline.opportunities.updated_at` not changing between E06 and E07 consumption |
| **E11 — ESPD / Grants Compliance** | E07 content blocks and compliance check results may be consumed or referenced by E11 grant-eligibility workflows; `proposal_id` FK relationships must remain stable | No schema changes to `client.proposals` that would break E11 FK references; `proposal.status` enum values backward compatible |
| **E12 — Analytics** | Proposal creation, generation, and export events feed analytics dashboards; E07 event bus publishes (if any) must emit correct stream/topic names and payload schema | Event payload schema stable; proposal CRUD events not broken by RLS changes; AI agent call count events correctly attributed to `company_id` |

---

## Follow-on Workflows

- Run `/bmad-testarch-atdd` to generate failing P0 ATDD tests for SSE persistence (E07-P0-001), optimistic locking race (E07-P0-003), and RLS cross-company (E07-P0-005) — highest-value starting points for TDD implementation
- Run `/bmad-testarch-automate` to expand backend API test coverage beyond P0/P1 once S07.01–S07.10 stories are implemented
- Run `/bmad-testarch-ci` to wire E07 backend and frontend test suites into the CI pipeline alongside E06 and E04

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework
- `probability-impact.md` — P×I scoring methodology
- `test-levels-framework.md` — Unit / Integration / Component / E2E level selection
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

- Epic: `eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md`
- System Architecture Test Design: `eusolicit-docs/test-artifacts/test-design-architecture.md`
- QA Test Design (System-Level): `eusolicit-docs/test-artifacts/test-design-qa.md`
- E06 Test Design (Immediate Dependency): `eusolicit-docs/test-artifacts/test-design-epic-06.md`
- E04 Test Design (AI Gateway Dependency): `eusolicit-docs/test-artifacts/test-design-epic-04.md`

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
