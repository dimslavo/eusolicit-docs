# E25: Knowledge Base Lifecycle

**Sprint:** post-pivot S+2 | **Points:** 34 | **Dependencies:** E24 | **Milestone:** AgenticSAI Pivot

> **Source:** `sprint-change-proposal-2026-05-12-agenticsai.md`, `prd-amendment-2026-05-12-agenticsai.md` (FR-49 to FR-52), `architecture-amendment-2026-05-12-agenticsai.md` (ADR-019).

## Goal

Make the per-Project AgenticSAI knowledge base a first-class, user-managed surface in EU Solicit. Users upload tenders, ESPD templates, company profile documents, past proposals, and qualification rubrics to their company's KB; agents (qualification, quantification, ESPD auto-fill, grant tools, proposal drafter) ground their outputs in retrieved KB passages with citations; users semantic-search across their KB and inspect the documents agents grounded against. Per ADR-019, **AgenticSAI is canonical for unstructured artefacts** — EU Solicit holds metadata pointers (`agenticsai_storage_resource_id`, `agenticsai_file_id`, `sha256`) but not the artefact bodies after upload. Tier-gated quotas enforced. Profile updates trigger automatic re-index.

## Acceptance Criteria

- [ ] Users can upload artefacts to their company KB via multipart form upload through `client-api`; supported formats: PDF, DOCX, TXT, MD; per-file max 50MB; per-tenant total quota tier-gated (Free 50MB, Starter 500MB, Professional 5GB, Enterprise unlimited)
- [ ] Each artefact tagged with `artefact_category` (tender / profile / proposal / espd_template / rubric / other) and free-form `tags` array
- [ ] Upload flow: stream file from `client-api` to AgenticSAI `POST /storage-resources/{vectorStoreId}/files` (multipart proxy, no buffering); record `client.agenticsai_kb_files` row with SHA-256 + metadata; emit `kb.file.uploaded` event
- [ ] AgenticSAI `storage.file.processed` webhook updates `client.agenticsai_kb_files.parsed_text_available_at`; UI badge transitions from "Processing…" to "Ready"
- [ ] Semantic search endpoint `POST /api/v1/kb/search` wraps AgenticSAI `POST /storage-resources/{id}/search`; returns ranked passages with file metadata, source file ID, and parsed-text excerpt
- [ ] Parsed-text download proxy `GET /api/v1/kb/files/{file_id}/parsed-text` wraps AgenticSAI `/files/{fileId}/parsed-text/download`
- [ ] Signed-URL artefact download proxy `GET /api/v1/kb/files/{file_id}/download` wraps AgenticSAI `/files/{fileId}/download` with EU Solicit auth check
- [ ] Replace flow: replacing a file deletes the old AgenticSAI file, uploads new, preserves `client.agenticsai_kb_files` row UUID + tags
- [ ] Archive flow: soft-delete in `client.agenticsai_kb_files` + delete in AgenticSAI storage-resource; user can restore within 30 days (re-upload from local backup if needed — bodies aren't kept in EU Solicit)
- [ ] Tier-gate quota check at upload time via existing TierGate Depends + per-tenant size sum query; over-quota returns 402 with upgrade prompt
- [ ] Profile-update re-index: on `company.profile_updated` event, refresh the company-profile artefact in the KB within 5 minutes; agents called within that window must ground against the new profile
- [ ] Agent grounding: all qualification, quantification, ESPD, grant tools, and proposal drafter agents have read access to their tenant's KB (already in AgenticSAI Project scope per ADR-018); structured agent outputs include source-citation references back to `client.agenticsai_kb_files.id`
- [ ] Nightly SHA-256 reconciliation: hash on file in `client.agenticsai_kb_files.sha256` compared against AgenticSAI inventory metadata; mismatch → admin alert + audit entry
- [ ] Right-to-Erasure path: company erasure deletes all KB files (per E24 S24.07 cross-substrate completion proof)
- [ ] Cross-tenant negative test: tenant A's API key cannot search or download tenant B's KB
- [ ] WCAG 2.1 AA: upload UI keyboard-navigable, progress announced via `aria-live`, file-type errors announced, drag-and-drop has keyboard fallback

## Stories

### S25.01: `client.agenticsai_kb_files` schema + upload metadata model
**Points:** 3 | **Type:** backend

Alembic migration for `client.agenticsai_kb_files` (per architecture amendment §4.1). SQLAlchemy model with `artefact_category` as StrEnum. Partial index `(company_id, artefact_category) WHERE archived_at IS NULL`. Service-layer functions: `record_uploaded_file`, `mark_processed`, `soft_delete`, `restore`, `compute_total_size_for_company`.

**Acceptance:**
- Migration up + down clean
- StrEnum validates category at write
- `compute_total_size_for_company` returns sum of `size_bytes WHERE archived_at IS NULL` — used for quota gate

---

### S25.02: Upload proxy + AgenticSAI storage-resource integration
**Points:** 5 | **Type:** backend

Endpoint `POST /api/v1/kb/files` (multipart). Streaming proxy to AgenticSAI without buffering the body in memory (FastAPI streaming + httpx streaming upload). On AgenticSAI success: persist `client.agenticsai_kb_files` row with SHA-256 computed during stream (incremental hash). Emit `kb.file.uploaded` event. Validate `artefact_category` + content-type before forwarding. Tier-gate Depends checks total-size + per-file size.

**Acceptance:**
- Upload of 25MB PDF succeeds end-to-end within 10s p95 on staging
- SHA-256 computed without buffering full file (incremental update)
- Over-quota upload returns 402 with structured error code `KB_QUOTA_EXCEEDED`
- Wrong content-type returns 415 with allowed-types list

---

### S25.03: Storage-processed webhook handler + UI state badge
**Points:** 3 | **Type:** backend + frontend

Webhook handler (in `data-pipeline` or `agenticsai-gateway` consumer per E28 routing): on `agenticsai.kb.file.processed` event, look up `client.agenticsai_kb_files` row by `agenticsai_file_id`, set `parsed_text_available_at = now()`. Frontend KB list page subscribes to TanStack Query invalidation; row state badge transitions `Processing… → Ready`. Idempotent.

**Acceptance:**
- Webhook delivery → UI state update within 5s p95 in dev
- Duplicate webhook → no-op
- Test: file uploaded; AgenticSAI webhook delayed; reconciler (per E28) catches missing processed state and converges

---

### S25.04: Semantic search + parsed-text + download proxy endpoints
**Points:** 5 | **Type:** backend

Three endpoints, all wrapping `agenticsai-gateway` calls with EU Solicit auth + cross-tenant scope check:
- `POST /api/v1/kb/search` → `POST .../storage-resources/{id}/search`; returns ranked passages
- `GET /api/v1/kb/files/{file_id}/parsed-text` → AgenticSAI parsed-text download (streaming response)
- `GET /api/v1/kb/files/{file_id}/download` → AgenticSAI file download (streaming response with original filename)

Per-tenant rate-limit on search endpoint (atomic Redis Lua, Epic 6 pattern).

**Acceptance:**
- Search returns within 2s p95 with 20 results
- Cross-tenant negative: user-A query for user-B's KB returns 404 (existence leakage protection)
- Parsed-text and download endpoints use streaming to avoid memory blow-up on large files
- Rate-limit returns 429 + Retry-After

---

### S25.05: Replace + archive + restore lifecycle
**Points:** 3 | **Type:** backend + frontend

Replace endpoint `PUT /api/v1/kb/files/{file_id}`: deletes old AgenticSAI file, uploads new, preserves the EU Solicit row UUID + tags + category. Archive `DELETE /api/v1/kb/files/{file_id}`: soft-delete in EU Solicit + delete in AgenticSAI; sets `archived_at`. Restore `POST /api/v1/kb/files/{file_id}/restore`: only valid within 30 days of archive, re-upload required from local source (UI prompts user — body isn't kept).

**Acceptance:**
- Replace preserves row UUID so historical links don't break
- Archive deletes AgenticSAI side within 10s
- Restore beyond 30 days returns 410 Gone

---

### S25.06: Profile-update KB re-index
**Points:** 2 | **Type:** backend

On `company.profile_updated` event (existing Redis Stream): identify the "company profile" KB artefact (`artefact_category='profile'`) — if exists, replace it with the new profile content (rendered to markdown). If none exists, create one. Idempotent.

**Acceptance:**
- Profile update → KB profile artefact refreshed within 5 minutes p95
- Subsequent agent runs ground against new profile (test by mocking AgenticSAI agent that echoes its retrieved KB passages)

---

### S25.07: Tier-gated quota + workspace KB dashboard
**Points:** 5 | **Type:** full-stack

Quota check via TierGate Depends. Frontend KB dashboard page per workspace: list view with filter by `artefact_category`, sort by `created_at` / `size_bytes`, search-within-KB-list, per-tenant total usage vs. quota with progress bar. Upgrade CTA on over-quota.

**Acceptance:**
- Pro tenant with 5GB quota: upload succeeds up to 5GB total, blocks at 5GB
- WCAG 2.1 AA: keyboard nav, `aria-live` for upload state, contrast OK
- E2E Playwright: upload → see in list → archive → restore (within 30d) → search

---

### S25.08: Nightly SHA-256 reconciliation
**Points:** 3 | **Type:** backend

Celery Beat (daily 04:00 UTC): for each company's active KB files, query AgenticSAI metadata, compare hash. Mismatch → audit log entry + admin alert via existing notification surface. Reconciliation result exported as Prometheus gauge `agenticsai_kb_hash_mismatches_total`.

**Acceptance:**
- 7-day rolling test: synthetic file insertion + AgenticSAI-side modification triggers alert
- Reconciliation completes within 10 minutes for 10K files across 100 tenants (load-test baseline)

---

### S25.09: Agent grounding + citation surface
**Points:** 5 | **Type:** backend + integration

Update agent response handling in `agenticsai-gateway`: when an agent run includes `kb_citations` in its structured output (AgenticSAI returns citation references for grounded responses), enrich them with `client.agenticsai_kb_files.id` + filename for UI rendering. Update qualification + quantification + ESPD endpoints to surface citations to the frontend. Integration test: agent returns 3 citations; UI shows 3 source file references with click-through to parsed-text view.

**Acceptance:**
- Citation enrichment adds EU Solicit-side metadata (filename, category, archived_at) to every AgenticSAI citation
- UI renders citation chips with hover + click-through
- Citation to archived file shows "Source archived" instead of broken link

## Salvaged patterns

- Atomic Redis Lua quota check (`_USAGE_LUA`, Epic 15 canonical) — same module, new resource key (`{company_id}:kb_size_bytes`).
- TierGate Depends factory (Epic 6 pattern) — unchanged.
- Streaming file upload (Epic 7 pattern for proposal exports) — same httpx streaming approach.
- Soft-delete uniqueness via partial index (Epic 14).
- Webhook idempotency cache (Epic 9 / new E28).

## Out of scope

- AI-assisted KB curation (automatic tagging, deduplication, similarity grouping) — post-launch.
- Cross-tenant content sharing within an enterprise account (`tenant_scope=true` artefacts) — separate epic post-launch.
- Vector embedding inspection / model selection for the KB — AgenticSAI manages this; EU Solicit doesn't expose it.
