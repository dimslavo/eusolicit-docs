# Story 5.25: Tenant-Aware Cross-Tenant Negative Tests for N8N Data Pipeline

**Status:** backlog
**Epic:** E05 amendment — Data Pipeline & Opportunity Ingestion
**Points:** 3
**Type:** integration / test
**Dependencies:** S05.24 (staged-rollout) done; E24 S24.01 schema (for tenant context)
**Blocks:** Slice 1 quality gate; safe legacy Celery retire
**Created:** 2026-05-15
**Source:** E05 epic + sprint-status `5-25-tenant-aware-cross-tenant-negative-tests`

## Story

As **Murat (TEA)**,
I want **cross-tenant negative tests for the N8N-driven data-pipeline confirming a webhook from one tenant cannot land an opportunity row in another tenant's scope**,
so that **the SirmaAI-cutover-introduced webhook path doesn't introduce a tenancy-bypass vulnerability**.

## Acceptance Criteria

1. Test suite at `services/data-pipeline/tests/integration/test_cross_tenant_ingestion_invariants.py` covers ≥8 scenarios:
   - Webhook signed for tenant A's Project lands an opportunity → row's `company_id=A`, NOT any other tenant.
   - Webhook missing tenant scoping → 400 (no fallback to "shared" or "default").
   - Webhook for tenant A but payload mentions tenant B's company_id → rejection (signature scope wins over payload claim).
   - Per-tenant feature-flag (S05.24) for tenant A "v2 enabled" → tenant A workflows use v2; tenant B workflows still v1.
   - Tenant A's `crawler_runs` audit row never visible to tenant B (workspace_id scope).
   - Tenant B consumer cannot read tenant A's `opportunities.ingested` Redis Stream events.
   - HMAC signature for tenant A's webhook secret cannot validate tenant B's payload (single-tenant HMAC scope).
   - Race: tenant A + tenant B simultaneous ingest → both land in correct scopes; no cross-contamination.
2. **Tests run on every CI build** as part of `make test-integration`.
3. **Load test variant**: 100 concurrent tenants ingesting simultaneously → all rows land in correct tenant scope.

## Dev Notes

### Pattern reuse
- Existing cross-tenant negative test fixtures.
- S04.20+S05.24 feature-flag mechanism.

### Files likely touched
- `services/data-pipeline/tests/integration/test_cross_tenant_ingestion_invariants.py` (new)
- `services/data-pipeline/tests/load/test_concurrent_ingest.py` (new)

### Out of scope
- E2E user-facing tests.
- Non-N8N ingestion paths (legacy Celery — retired).

## Testing

- Self-testing.

## See also

- E05 epic §S05.25
- S05.24 (staged-rollout)
- Project memory: "Cross-tenant negative tests are non-negotiable"
