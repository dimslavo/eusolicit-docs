# Story 25.04: Semantic Search + Parsed-Text + Download Proxy Endpoints

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 5
**Type:** backend
**Dependencies:** S25.02 (files exist in SirmaAI + DB)
**Blocks:** S25.07 (UI completeness), S25.09 (citation surface)
**Created:** 2026-05-15
**Source:** E25 epic §S25.04

## Story

As **Elena researching whether my company has prior experience with environmental impact assessments**,
I want **to semantic-search across my Knowledge Base, view parsed text excerpts, and download source artefacts**,
so that **I can verify what content the AI is grounding in and pull supporting documents into my proposal**.

## Acceptance Criteria

1. **Endpoint** `POST /api/v1/kb/search` accepts `{query: str, limit?: int=20, category_filter?: str}`. Wraps SirmaAI `POST /storage-resources/{id}/search` via `sirmaai-gateway`.
2. **Response shape**:
   ```json
   {
     "results": [
       {
         "kb_file_id": "uuid",
         "filename": "past-proposal-aop-2024-12.pdf",
         "artefact_category": "proposal",
         "relevance_score": 0.87,
         "excerpt": "...environmental impact assessment...",
         "page_anchor": 14
       }
     ],
     "total": 5
   }
   ```
3. **Endpoint** `GET /api/v1/kb/files/{kb_file_id}/parsed-text` streams SirmaAI parsed-text via `GET /files/{sirmaai_file_id}/parsed-text/download`. Content-Type `text/plain`. EU Solicit-side auth check enforces tenant scope before forward.
4. **Endpoint** `GET /api/v1/kb/files/{kb_file_id}/download` streams the original artefact via SirmaAI `/files/{sirmaai_file_id}/download`. Content-Disposition `attachment; filename="{original_filename}"`. Cross-tenant scope-check applied.
5. **Streaming**: parsed-text + download endpoints use FastAPI `StreamingResponse` with httpx async streaming — no full-body buffering. Verified for 100 MB synthetic file.
6. **Per-tenant rate-limit on search**: max 100 search requests / minute / company via atomic Redis Lua (Epic 6 pattern). Returns 429 with `Retry-After` header on exceedance.
7. **Cross-tenant negative**: user-A query for user-B's `kb_file_id` returns 404 (existence-leakage protection — NOT 403). Verified.
8. **Archived-file access**: archived files return 410 Gone on download endpoints (parsed text + body), but still return basic metadata on `GET /api/v1/kb/files/{kb_file_id}` (for audit / citation rendering).
9. **Performance**: search p95 < 2s with 20 results; parsed-text streaming first-byte < 500ms.
10. **Error responses** are uniform: `{error_code, message}` shape.

## Dev Notes

### Pattern reuse
- Streaming proxy: see proposal-export streaming in client-api.
- Rate-limit atomic Lua: `services/client-api/src/client_api/services/usage_metering.py`.
- Existence-leakage protection: scope check returns 404 NOT 403 (existing project pattern).

### Files likely touched
- `services/client-api/src/client_api/routers/kb.py` (extend with 3 endpoints)
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/storage_client.py` (extend with `search`, `parsed_text_stream`, `file_download_stream`)
- `services/client-api/tests/integration/test_kb_search_and_download.py`

### Out of scope
- Citation chips (S25.09)
- Full-text full-document search (only semantic via SirmaAI vector search)
- AI-driven query rewrite (post-launch enhancement)

## Risks

- **R1**: Long-running searches (timeout) — set explicit httpx timeout on the proxy call; surface 504 on SirmaAI timeout.
- **R2**: Search result inflation by archived files — backend filters `archived_at IS NULL` from search-result `kb_file_id` resolution.

## Testing

- Unit: response-shape validation; archived-file 410 path.
- Integration: full search + parsed-text + download against mocked SirmaAI.
- Cross-tenant: scope-check 404.
- Performance: 100MB synthetic download — first-byte latency.

## See also

- Epic file §S25.04
- PRD amendment FR-50
- ADR-019 (KB ownership split)
