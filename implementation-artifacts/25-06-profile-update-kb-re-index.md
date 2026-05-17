# Story 25.06: Profile-Update KB Re-Index

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 2
**Type:** backend
**Dependencies:** S25.02 (upload), S25.05 (replace flow)
**Blocks:** none (quality-of-life; not a Slice 2 DoD gate)
**Created:** 2026-05-15
**Source:** E25 epic §S25.06

## Story

As **Elena who just updated my company profile (added a new certification + sector)**,
I want **the AI agents to start grounding in my updated profile within 5 minutes — no manual re-upload step**,
so that **the next qualification run reflects my company's actual current capabilities**.

## Acceptance Criteria

1. Subscriber on existing Redis Stream `company.profile_updated` (already published by company-profile-CRUD flow): on each event, dispatch Celery task `refresh_company_profile_kb_artefact(company_id)`.
2. The task:
   - Look up the company's KB artefact with `artefact_category='profile'` (there should be at most 1 active).
   - Render the current company profile as Markdown (existing helper `render_company_profile_md(company_id)` — or create one based on the structured profile fields: name, sectors/CPV codes, regions, certifications, description).
   - If a profile artefact exists: use S25.05 replace-flow to swap the body.
   - If none exists: use S25.02 upload-flow to create one with `filename='company-profile-{timestamp}.md'`, `artefact_category='profile'`.
3. **Idempotency**: re-running the task for the same company is safe — replace-flow handles existing rows; multiple profile events within 5 min get coalesced (only the latest matters).
4. **Performance**: profile-updated event → KB profile artefact refreshed within 5 minutes p95.
5. **Test scenario**: company profile update → wait 5 min → query SirmaAI agent that echoes its retrieved KB passages → agent grounds in the NEW profile content (verify by changing a sentinel certification field; mocked agent returns it).
6. **Audit log**: each refresh writes `shared.audit_log` row with `action_type='kb_profile_auto_refresh'`.

## Dev Notes

### Pattern reuse
- Redis Stream consumer: `services/notification/src/notification/consumers/subscription_changed.py`.
- Markdown rendering: existing helper if available; otherwise simple Jinja2 template.
- S25.05 replace-flow service function.

### Files likely touched
- `services/client-api/src/client_api/consumers/company_profile_updated.py` (new)
- `services/client-api/src/client_api/services/sirmaai_kb_service.py` (extend with `refresh_profile_artefact`)
- `services/client-api/src/client_api/services/company_profile_renderer.py` (new — MD renderer)
- `services/client-api/tests/integration/test_profile_kb_refresh.py`

### Out of scope
- Other auto-re-index triggers (past-proposal updates, etc.) — only profile for v1.
- AI-assisted "should we re-index?" judgement — naive blanket refresh on every event.

## Risks

- **R1**: High-frequency profile edits trigger a storm of re-indexes — debounce: coalesce events within a 30s window before dispatching.
- **R2**: SirmaAI re-index latency >5min — flag for operator; SLA is best-effort.

## Testing

- Unit: profile rendering correctness; idempotency.
- Integration: full event → re-index → agent re-grounding flow.

## See also

- Epic file §S25.06
- PRD amendment FR-52
- S25.05 (replace-flow this story consumes)
