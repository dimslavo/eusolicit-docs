# E06: Opportunity Discovery & Intelligence

**Sprint**: 5-6 | **Points**: 55 | **Dependencies**: E02, E03, E05 | **Milestone**: Demo

## Goal

Enable users to discover, search, and explore EU procurement opportunities with tier-gated access to data, documents, and AI-generated intelligence. The Client API exposes opportunity data ingested by the Data Pipeline (E05) with read-only access to `pipeline.opportunities`, enforcing subscription tier rules that control visible fields, geographic scope, CPV sectors, and budget ranges. Free users see limited metadata; paid tiers unlock progressively richer data, document management, and AI summaries metered via Redis atomic counters. The frontend delivers a responsive listing/detail experience with search, filtering, document upload/download, and streaming AI analysis.

## Tier Access Rules

| Capability | Free | Starter | Professional | Enterprise |
|---|---|---|---|---|
| Fields | name, deadline, location, type, status | All fields | All fields | All fields |
| Regions | -- | Bulgaria only | BG + 3 EU countries | All EU |
| CPV Sectors | -- | 1 sector | Up to 3 sectors | Unlimited |
| Budget Range | -- | Up to 500K EUR | Up to 5M EUR | Unlimited |
| AI Summaries/month | 0 | 10 | 50 | Unlimited |

## Acceptance Criteria

- [ ] Opportunity search returns paginated results using cursor-based pagination with full-text search and all filter combinations (CPV, region, budget range, deadline range, status, type)
- [ ] Free-tier users receive `OpportunityFreeResponse` (name, deadline, location, type, status only); paid-tier users receive `OpportunityFullResponse`
- [ ] TierGate middleware blocks access to opportunities outside the user's region/CPV/budget allowance and returns 403 with upgrade prompt
- [ ] UsageGate middleware uses atomic Redis INCR per feature per billing period; returns 429 with `X-Usage-Remaining` header and upgrade prompt on limit exceeded
- [ ] Opportunity detail endpoint returns full tender data including evaluation criteria, mandatory documents, and linked AI analysis for paid users
- [ ] Document upload generates S3 presigned URL, triggers ClamAV scan, records metadata in `client.documents`; enforces 100MB/file and 500MB/package limits
- [ ] Document download generates S3 presigned URL after access control check against user's tier and opportunity ownership
- [ ] AI summary generation calls Executive Summary Agent via AI Gateway, streams response via SSE, and decrements usage counter atomically
- [ ] Smart matching relevance score from pipeline is displayed on listing cards and detail page
- [ ] Submission guide from `pipeline.submission_guides` renders as step-by-step accordion on detail page
- [ ] Listing page supports table/card view toggle, sortable columns, search bar, filter sidebar, and cursor-based pagination
- [ ] Detail page uses tabbed layout with Overview, Documents, Requirements, AI Analysis, and Submission Guide tabs
- [ ] Upgrade prompt modal appears when tier or usage limit is hit, displaying tier comparison and CTA
- [ ] All API endpoints have OpenAPI documentation and integration tests covering each tier

## Stories

### S06.01: Opportunity Search API with Full-Text Search & Filters
**Points**: 5 | **Type**: backend

Implement `GET /api/v1/opportunities/search` with PostgreSQL full-text `ts_vector`/`ts_query` search against opportunity name, description, and contracting authority. Support filter query params: `cpv_codes` (multi-select, comma-separated), `regions` (multi-select), `budget_min`/`budget_max`, `deadline_from`/`deadline_to`, `status` (enum), `opportunity_type` (enum). Return cursor-based pagination using `after_cursor`/`before_cursor` params with configurable `limit` (default 25, max 100). Query reads from `pipeline.opportunities` with read-only DB session. Include total count in response metadata.

---

### S06.02: Tier-Gated Response Serialization
**Points**: 3 | **Type**: backend

Define Pydantic response models `OpportunityFreeResponse` (fields: id, name, deadline, location, type, status) and `OpportunityFullResponse` (all fields including budget, CPV codes, contracting authority details, evaluation criteria, documents list, relevance score). Implement `TierGate` dependency that inspects the authenticated user's subscription tier from the JWT/session, compares against the opportunity's region, CPV sector, and budget, and selects the appropriate response model. Return 403 with `{"error": "tier_limit", "upgrade_url": "..."}` when a paid-tier user tries to access an opportunity outside their tier scope. Free-tier users always get `OpportunityFreeResponse` regardless of opportunity attributes.

---

### S06.03: Usage Metering Middleware (Redis)
**Points**: 3 | **Type**: backend

Implement `UsageGate` as a FastAPI dependency. On each metered request (AI summaries, future metered features), execute `INCR user:{user_id}:usage:{feature}:{billing_period}` with `EXPIRE` set to end-of-billing-period TTL. Before incrementing, `GET` current count and compare against tier limit. If limit exceeded, return 429 with body `{"error": "usage_limit_exceeded", "limit": N, "used": M, "resets_at": "ISO8601", "upgrade_url": "..."}` and header `X-Usage-Remaining: 0`. On success, set response header `X-Usage-Remaining: N`. Use Redis pipeline for atomic GET+INCR. Write unit tests with fakeredis.

---

### S06.04: Opportunity Listing API Endpoint
**Points**: 3 | **Type**: backend

Implement `GET /api/v1/opportunities` as the primary listing endpoint consumed by the frontend listing page. Accepts same filter params as search but without full-text query (browse mode). Applies TierGate to each result: free users get `OpportunityFreeResponse[]`, paid users get `OpportunityFullResponse[]` filtered to their tier scope. Supports `sort_by` param (deadline, relevance_score, budget, published_date) and `sort_order` (asc/desc). Returns cursor-based paginated response with `next_cursor`, `prev_cursor`, `total_count`, and `results[]`. Include relevance score in response for paid users.

---

### S06.05: Opportunity Detail API Endpoint
**Points**: 3 | **Type**: backend

Implement `GET /api/v1/opportunities/{opportunity_id}`. Free users receive 403 directing to upgrade. Paid users receive full tender data after TierGate validation: contracting authority details, full description, evaluation criteria (weight, type, description), mandatory document list, submission deadline with timezone, estimated budget, CPV codes with labels, linked AI analysis summary (if previously generated), submission guide reference, and relevance score. Include `related_opportunities` field with up to 5 similar opportunities by CPV/region overlap. Return 404 for non-existent IDs.

---

### S06.06: Document Upload API (S3 + ClamAV)
**Points**: 5 | **Type**: backend

Implement `POST /api/v1/opportunities/{opportunity_id}/documents/upload` that returns an S3 presigned URL for direct browser upload. Accept file metadata (name, size, MIME type) in request body. Validate: file size <= 100MB, total package size <= 500MB (sum existing + new), allowed MIME types (PDF, DOCX, XLSX, ZIP). Generate presigned PUT URL with 15-minute expiry. After upload confirmation callback (`POST .../documents/{doc_id}/confirm`), trigger ClamAV scan via async task (Celery/ARQ). Record document metadata in `client.documents` table (id, opportunity_id, user_id, filename, size, mime_type, s3_key, scan_status: pending/clean/infected, uploaded_at). If scan fails, mark as infected and soft-delete S3 object.

---

### S06.07: Document Download API
**Points**: 2 | **Type**: backend

Implement `GET /api/v1/opportunities/{opportunity_id}/documents/{document_id}/download`. Verify requesting user has access: must be document owner or have appropriate tier access to the opportunity. Verify `scan_status = clean`. Generate S3 presigned GET URL with 10-minute expiry and `Content-Disposition: attachment` header. Return 403 if access denied, 404 if document not found, 422 if scan still pending or infected. Log download event for audit trail.

---

### S06.08: AI Summary Generation API (SSE Streaming)
**Points**: 5 | **Type**: backend

Implement `POST /api/v1/opportunities/{opportunity_id}/ai-summary` that generates an executive summary via the AI Gateway. Check UsageGate first (decrement AI summary counter). Call Executive Summary Agent with opportunity data as context. Stream response back to client using Server-Sent Events (`text/event-stream`). Event types: `delta` (text chunk), `metadata` (token count, model), `done` (final summary ID), `error`. Persist completed summary in `client.ai_summaries` table (id, opportunity_id, user_id, content, model, tokens_used, generated_at). Support `GET .../ai-summary` to retrieve cached summary without re-generation. If cached summary exists and is < 24h old, return it directly without metering.

---

### S06.09: Opportunity Listing Page (Table + Card View)
**Points**: 5 | **Type**: frontend

Build the opportunity listing page at `/opportunities`. Implement dual-view toggle: table view with sortable columns (name, deadline, location, budget, relevance score, status) and card view showing opportunity cards with key metadata and relevance badge. Search bar at top triggers full-text search API. Infinite scroll or cursor-based "Load More" pagination. Empty states for no results and no access. Loading skeletons during fetch. Mobile-responsive: cards stack vertically, table collapses to card view on small screens. Wire up to opportunity search/listing API. Free-tier users see limited card content with blurred/locked indicators on restricted fields.

---

### S06.10: Search & Filter Components
**Points**: 5 | **Type**: frontend

Build the filter sidebar panel with: CPV code multi-select with autocomplete (searchable dropdown, loads CPV taxonomy from API), region checkboxes (grouped by EU country with select-all), budget range slider (min/max with EUR formatting), deadline date range picker (from/to with calendar widget), status toggle buttons (Open, Closing Soon, Closed), opportunity type selector (Works, Services, Supplies, Mixed). Filters apply immediately with debounced API calls. Active filters shown as removable chips above results. "Clear all filters" button. Filter state persisted in URL query params for shareable links. Collapsible sidebar on mobile as bottom sheet.

---

### S06.11: Opportunity Detail Page (Tabbed Layout)
**Points**: 5 | **Type**: frontend

Build the opportunity detail page at `/opportunities/:id` with tabbed layout. **Overview tab**: contracting authority, description, budget, deadline countdown timer, CPV codes as tags, relevance score gauge, key dates timeline. **Documents tab**: uploaded documents list with download buttons, upload drop zone (wired to S06.12), scan status badges. **Requirements tab**: evaluation criteria table (criterion, weight, description), mandatory documents checklist. **AI Analysis tab**: AI summary display (wired to S06.13), regenerate button. **Submission Guide tab**: accordion rendering of `pipeline.submission_guides` steps. Tab state in URL hash. Breadcrumb navigation back to listing. 404 page for invalid IDs. Access-denied state for free users with upgrade CTA.

---

### S06.12: Document Upload Component
**Points**: 3 | **Type**: frontend

Build a reusable document upload component used on the opportunity detail Documents tab. Drag-and-drop zone with click-to-browse fallback. File type validation (PDF, DOCX, XLSX, ZIP) with user-friendly error messages. File size validation (100MB) client-side before requesting presigned URL. Multi-file upload queue with individual progress bars. Upload flow: request presigned URL from API, PUT file directly to S3, call confirm endpoint, poll scan status. Status indicators per file: uploading (progress %), scanning (spinner), clean (green check), infected (red X with message). Cancel in-progress uploads. Total package size indicator showing usage against 500MB limit.

---

### S06.13: AI Summary Panel with SSE Streaming
**Points**: 3 | **Type**: frontend

Build the AI summary panel component used on the opportunity detail AI Analysis tab. On tab open, check for cached summary via GET endpoint. If no cached summary, show "Generate Summary" CTA button. On click, connect to SSE endpoint and render streaming text with typing animation. Show token count and generation time on completion. "Regenerate" button triggers new generation (with usage confirmation dialog showing remaining quota). Loading state with pulsing skeleton. Error state with retry button. Usage counter display: "X of Y summaries remaining this month". When usage limit hit, replace CTA with upgrade prompt.

---

### S06.14: Upgrade Prompt Modal & Tier Gating UI
**Points**: 3 | **Type**: frontend

Build a reusable upgrade prompt modal triggered by 403 (tier limit) and 429 (usage limit) API responses. Modal displays: current tier name, limitation hit (e.g., "Bulgaria-only access" or "10/10 AI summaries used"), tier comparison table showing what each tier unlocks, highlighted recommended tier, and primary CTA button linking to billing/upgrade page. Implement global API interceptor that catches 403/429 responses with `tier_limit` or `usage_limit_exceeded` error codes and triggers the modal automatically. Subtle lock icons and "Upgrade to unlock" overlays on restricted content throughout listing and detail pages. Dismissible but re-triggered on next restricted action.
