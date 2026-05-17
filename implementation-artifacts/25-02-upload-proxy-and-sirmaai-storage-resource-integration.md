# Story 25.02: KB Upload Proxy + SirmaAI Storage-Resource Integration

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 5
**Type:** backend
**Dependencies:** S25.01 (schema), S24.02 (Project exists), S24.03 (default-kb storage-resource seeded)
**Blocks:** S25.03, S25.04, S25.05, S25.07
**Created:** 2026-05-15
**Source:** E25 epic §S25.02

## Story

As **Elena uploading past proposals to my company's Knowledge Base**,
I want **a fast, streaming upload that doesn't blow EU Solicit's memory and respects my tier quota**,
so that **my 25 MB tender PDFs land in the KB end-to-end in under 10 seconds without server-side overflow**.

## Acceptance Criteria

1. **Endpoint** `POST /api/v1/kb/files` (`client-api`) accepts multipart form-data with fields: `file`, `artefact_category` (required), `tags` (optional, JSON array string).
2. **Streaming proxy**: FastAPI `UploadFile` stream is forwarded to SirmaAI's `POST /storage-resources/{vector_store_id}/files` via httpx streaming upload — body never fully loaded into memory.
3. **Incremental SHA-256**: hash computed via `hashlib.sha256().update(chunk)` for each streamed chunk, then `.hexdigest()` at stream end. Verified against post-upload SirmaAI metadata if SirmaAI returns hash; otherwise EU-Solicit-computed hash is authoritative.
4. **Tier-gate Depends** (existing pattern) checks BEFORE forwarding: (a) per-file size cap (50 MB hardcoded for v1); (b) per-tenant total quota — `compute_total_size_for_company(company_id) + new_file_size ≤ tier_quota` where quotas are Free 50MB, Starter 500MB, Pro 5GB, Enterprise unlimited.
5. **Validation BEFORE forward**:
   - `artefact_category` ∈ StrEnum values
   - Content-Type ∈ {`application/pdf`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, `text/plain`, `text/markdown`}
   - Pre-stream size check via `Content-Length` header (rejected at 50 MB)
6. **Error responses**:
   - 402 with `{"error_code": "KB_QUOTA_EXCEEDED", "current_size_bytes": X, "limit_bytes": Y}` on over-quota.
   - 415 with `{"error_code": "UNSUPPORTED_CONTENT_TYPE", "allowed_types": [...]}` on wrong type.
   - 413 with size info on > 50 MB.
   - 422 on bad `artefact_category`.
7. **On SirmaAI success**: persist `client.sirmaai_kb_files` row via `record_uploaded_file`; emit `kb.file.uploaded` event to Redis Stream `kb.file.uploaded` with `{company_id, kb_file_id, sirmaai_file_id, filename, category, size_bytes}`.
8. **On SirmaAI failure**: do NOT persist a DB row; return 502 with `{error_code: "SIRMAAI_UPLOAD_FAILED", retriable: true}`.
9. **Performance**: 25 MB PDF upload end-to-end completes in < 10s p95 on staging (measured via Prometheus histogram `kb_upload_duration_seconds`).
10. **Cross-tenant**: user-A's upload cannot target user-B's storage-resource_id — verified via integration test where the resolved `sirmaai_storage_resource_id` always comes from `client.sirmaai_projects` for the calling tenant (not from request body).

## Dev Notes

### Pattern reuse
- httpx streaming upload: see how proposal-export streams to MinIO in `services/client-api/src/client_api/services/proposal_export.py`.
- TierGate Depends factory: existing in `services/client-api/src/client_api/core/tier_gating.py`.
- Atomic Redis Lua quota check: see Epic 15 canonical `_USAGE_LUA` pattern; same module, new resource key `{company_id}:kb_size_bytes` (NOT required for v1 — DB `compute_total_size_for_company` is sufficient; revisit if hot-path latency becomes an issue).

### Files likely touched
- `services/client-api/src/client_api/routers/kb.py` (new)
- `services/client-api/src/client_api/services/sirmaai_kb_service.py` (extend with upload-proxy logic)
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/storage_client.py` (new — streaming upload)
- `services/client-api/src/client_api/api/v1/__init__.py` (register router)
- `services/client-api/tests/integration/test_kb_upload.py`
- `eusolicit-models/src/eusolicit_models/events.py` (add `KbFileUploadedEvent`)

### Out of scope
- Frontend upload UI (S25.07)
- Processing-state badge (S25.03 — that's the webhook side)
- Search endpoints (S25.04)

## Risks

- **R1**: SirmaAI streaming upload may have a chunk-size requirement — verify against SirmaAI OpenAPI docs.
- **R2**: Concurrent uploads racing on quota — atomic check + insert. Use SELECT ... FOR UPDATE on the quota query OR accept best-effort race (slight over-quota tolerance) for v1.
- **R3**: Filename injection / path traversal — sanitise filename before persisting (slash + ".." stripped).

## Testing

- Unit: tier-gate logic, content-type validation, size validation.
- Integration: full upload flow with mocked SirmaAI 200; tier-gate 402 path; oversize 413 path.
- Performance: 25 MB upload < 10s on staging.
- Cross-tenant: user-A upload + user-B query → user-B sees nothing.

## See also

- Epic file §S25.02
- PRD amendment FR-49
- ADR-019 (KB ownership split)
- Epic 15 (TierGate canonical pattern)
