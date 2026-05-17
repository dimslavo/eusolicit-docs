# Story 26.03: Opportunity-Analysis-v1 N8N Workflow Template

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 5
**Type:** workflow + backend
**Dependencies:** S26.01 (qualifier def), S26.02 (quantifier def), S05.24 (staged-rollout enforcement) done
**Blocks:** S26.05 (user trigger), S26.06 (auto-trigger)
**Created:** 2026-05-15
**Source:** E26 epic §S26.03

## Story

As a **platform engineer**,
I want **an N8N workflow template that orchestrates qualification → quantification → post-processing for a given opportunity**,
so that **EU Solicit's invocation surface is a single workflow trigger rather than a chained client-side dance, and we get N8N's retry + observability for free**.

## Acceptance Criteria

1. Author N8N workflow template `opportunity-analysis-v1` at `infra/n8n/templates/opportunity-analysis-v1.json`. Parameterised inputs: `(projectId, opportunityId, analysis_type ∈ {qualification, quantification, both}, force, eusolicit_run_id)`.
2. **Workflow steps** in order:
   - Receive trigger.
   - Fetch canonical opportunity row from EU Solicit `pipeline.opportunities` via internal API.
   - Invoke `opportunity_qualifier` agent (async-run) under tenant Project scope.
   - On `recommended_action='pursue'` AND `analysis_type ∈ {quantification, both}`: invoke `opportunity_quantifier` agent (async-run).
   - Invoke `opportunity_post_processor` agent (small wrap-up agent that finalises structured output).
   - Emit `agent.run.completed` Standard Webhook back to EU Solicit (per S28.03 receiver path).
3. **Versioning**: workflow is semver-tagged (`v1` initial; future `v2` lives alongside `v1` until 100% staged-rollout). Per-tenant feature-flag selects version (per S05.24 staged-rollout mechanism).
4. **Rollback runbook** at `eusolicit-docs/runbooks/opportunity-analysis-rollback.md` covering: flip per-tenant feature flag back to `v1`; next run uses old version; no data migration needed.
5. **Staged-rollout discipline**:
   - Canary tenant first (1 tenant for 24h).
   - 10% of tenants for 7 days.
   - 100% after 7 days clean.
   - Per S05.24 feature-flag mechanism + ADR-018 §3.4.
6. **End-to-end staging test**: trigger workflow with a sample opportunity → both agents fire → webhook returns to EU Solicit with structured result.
7. **PR-review discipline**: workflow JSON is in repo (`infra/n8n/templates/`); every change is PR-reviewed with named prompt-engineering + ops reviewer. NO dashboard-edited workflow changes.
8. **N8N → EU Solicit auth**: webhook call uses HMAC signed with `gateway.webhook_subscriptions.hmac_secret_encrypted` per S28.03.

## Dev Notes

### Pattern reuse
- S05.20 N8N workflow templates pattern.
- S05.22 phase-1 equivalence test harness pattern.
- Staged-rollout enforcement (S05.24).

### Files likely touched
- `infra/n8n/templates/opportunity-analysis-v1.json` (new)
- `infra/n8n/templates/README.md` (extend with the new template)
- `eusolicit-docs/runbooks/opportunity-analysis-rollback.md` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/services/workflow_trigger.py` (extend with `trigger_opportunity_analysis_v1`)
- `tests/staging/test_opportunity_analysis_e2e.py`

### Out of scope
- The `opportunity_post_processor` agent — minimal wrapping; defined in template.
- User-trigger endpoint (S26.05).
- Auto-trigger (S26.06).
- Result persistence (S26.07).

## Risks

- **R1**: N8N workflow JSON drift between staging and prod — committed-to-repo enforces, but operator must apply via `n8n import:workflow`.
- **R2**: Async-run polling within N8N may time out if SirmaAI is slow — N8N workflow has per-step timeout config; tune to 10min for quantifier step.

## Testing

- Staging E2E: trigger → both agents → webhook ack.
- Versioning: deploy `v2` alongside `v1`; rollback works.

## See also

- Epic file §S26.03
- S05.24 (staged rollout)
- ADR-018 (N8N workflow versioning)
- E26 §S26.06 (auto-trigger consumer)
