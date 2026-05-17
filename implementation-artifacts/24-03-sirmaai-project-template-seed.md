# Story 24.03: SirmaAI Project Template Seed (Default Agents + KB)

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 5
**Type:** backend + ops
**Dependencies:** S24.02 (Project exists); coordinated with S11.20 (E11 amendment template content) and S26.01 + S26.02 (qualifier + quantifier defs)
**Blocks:** S24.04 (MCP server stubs are added on top of seeded template); every agent path
**Created:** 2026-05-15
**Source:** `eusolicit-docs/planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md` §S24.03

## Story

As a **platform engineer**,
I want **every newly-provisioned SirmaAI Project to be seeded with a default knowledge-base storage-resource and the standard set of EU Solicit agents**,
so that **the first time Elena uploads a tender, the qualification/drafting/ESPD/grant agents are already available under her tenant scope — no per-tenant manual operator setup**.

## Acceptance Criteria

1. After S24.02 step (a) — Project creation succeeds — dispatch a follow-up Celery task `seed_sirmaai_project_template(company_id)` that applies the v1 template.
2. Template application creates:
   - (a) Default storage-resource named `default-kb` via `POST /api/organizations/{orgId}/projects/{projectId}/storage-resources`.
   - (b) 8+ agent definitions sourced from `services/client-api/config/sirmaai_project_template.yaml`, including (at minimum): `executive_summary`, `requirement_extractor`, `risk_flagger`, `proposal_drafter`, `compliance_checker`, `score_simulator`, `espd_auto_fill`, `grant_eligibility`. Each agent created via `POST /api/organizations/{orgId}/projects/{projectId}/agents` with prompt + tools + model binding from the YAML.
   - (c) `client.sirmaai_projects.agent_map` JSONB populated with `{logical_name: sirmaai_agent_uuid}` per seeded agent.
   - (d) `client.sirmaai_projects.template_version` set to the template version (e.g. `'v1'`).
3. Template application completes within 60s of Project creation (p95). Measured via Prometheus histogram `sirmaai_template_seed_duration_seconds`.
4. Template file `services/client-api/config/sirmaai_project_template.yaml` is PR-reviewed and versioned; structure documented in a top-of-file comment block. Schema:
   ```yaml
   version: "v1"
   storage_resources:
     - name: default-kb
       type: vector_store
   agents:
     - logical_name: executive_summary
       sirmaai_definition:
         model: ...
         system_prompt: ...
         tools: [kb_search]
         structured_output_schema: ...
     # ...7 more
   ```
5. **Integration test** seeds a Project on a SirmaAI staging instance (mocked or real), invokes a templated agent via `sirmaai-gateway.call_agent("executive_summary", company_id, payload)`, asserts a coherent response.
6. **Idempotency**: re-running the seed task on an already-seeded Project is a no-op (checks `agent_map` non-empty + `template_version` matches).
7. **Version-bump path**: if template version on disk > `client.sirmaai_projects.template_version`, the task adds missing agents and updates `template_version`. Doesn't remove or rewrite existing agents (avoids breaking in-flight work).
8. **Failure mode**: if any of the agent creates fails partway, the task leaves `provisioning_status='pending'` and the existing exponential-backoff retry on S24.02 resumes (the seed task is part of the same retry budget).

## Dev Notes

### Template content authoring rule
- The grant/compliance agent definitions (espd_auto_fill, grant_eligibility, budget_builder, etc.) are AUTHORED by S11.20 (E11 amendment). This story (S24.03) provides the *application mechanism*; S11.20 provides the *content* for the grant/compliance subset. Coordination: S11.20 commits content into the YAML; S24.03 reads + applies it. PR-merge order: S11.20's content PR can merge before S24.03 is fully done — the YAML file just sits there.
- Qualifier + quantifier agent definitions (S26.01, S26.02) likewise commit content into the same YAML.
- Sprint 2 reality check: if S24.03 lands BEFORE S11.20 + S26.01 + S26.02 content lands, ship with the minimal legacy set (`executive_summary`, `requirement_extractor`, `risk_flagger`, `proposal_drafter`, `compliance_checker`, `score_simulator`). Add others via template-version-bump in Sprint 3 (`v1` → `v2`).

### Files likely touched
- `services/client-api/config/sirmaai_project_template.yaml` (new)
- `services/client-api/src/client_api/tasks/sirmaai_provisioning.py` (extend with seed task)
- `services/client-api/src/client_api/services/sirmaai_template_loader.py` (new — YAML parsing + version compare)
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/admin_client.py` (extend with `create_agent`, `create_storage_resource`)
- `services/client-api/tests/unit/services/test_sirmaai_template_loader.py`
- `services/client-api/tests/integration/test_sirmaai_template_seed.py`

### YAML schema validation
- Pydantic model `SirmaaiProjectTemplate` validates the YAML on load. Invalid YAML in CI fails build (CI check: `python -c "from client_api.services.sirmaai_template_loader import load; load()"`).
- Schema migration discipline: any breaking change to agent definitions = template_version bump + migration plan for existing tenants.

### Out of scope
- The actual agent prompt + tool content for grant/compliance (S11.20).
- The qualifier + quantifier agent definitions (S26.01, S26.02).
- MCP server stubs (S24.04 — added on top).
- Frontend visibility of seeded agents (no UI — agents are platform-internal).

## Risks

- **R1**: Agent content drift between YAML and SirmaAI side — version-bump discipline + integration test eval-run guard against silent drift.
- **R2**: Template content is sensitive (system prompts) — repo file is fine (no secrets), but PR review for prompt content must include prompt-engineering review.
- **R3**: SirmaAI agent-create endpoint quota — provisioning surge could exceed it; throttle via Celery worker concurrency on this task queue.

## Testing

- Unit: YAML schema validation, version compare, idempotency check, agent_map population.
- Integration: end-to-end seed against staging SirmaAI; eval-run on `executive_summary` returns plausible output.
- Performance: 60s p95 budget verified on staging.

## See also

- Epic file §S24.03
- E11 amendment §S11.20 (template content for grant/compliance)
- E26 §S26.01/S26.02 (qualifier/quantifier definitions)
- Architecture amendment §4.4 (Project template invariants)
