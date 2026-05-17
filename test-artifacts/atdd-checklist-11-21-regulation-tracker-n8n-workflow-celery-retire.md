---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-05-16'
workflowType: atdd
storyId: '11.21'
storyKey: 11-21-regulation-tracker-n8n-workflow-celery-retire
storyFile: eusolicit-docs/implementation-artifacts/11-21-regulation-tracker-n8n-workflow-celery-retire.md
atddChecklistPath: eusolicit-docs/test-artifacts/atdd-checklist-11-21-regulation-tracker-n8n-workflow-celery-retire.md
tddPhase: RED
generatedTestFiles:
  - eusolicit-app/services/admin-api/tests/unit/test_regulation_tracker_n8n_template.py
  - eusolicit-app/services/admin-api/tests/api/test_regulation_change_webhook.py
  - eusolicit-app/services/admin-api/tests/unit/test_celery_deprecation_and_runbook.py
inputDocuments:
  - eusolicit-docs/implementation-artifacts/11-21-regulation-tracker-n8n-workflow-celery-retire.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-app/services/admin-api/tests/conftest.py
  - eusolicit-app/services/admin-api/tests/api/test_regulatory_changes.py
  - eusolicit-app/services/admin-api/tests/tasks/test_regulation_tracker_task.py
  - eusolicit-app/services/admin-api/src/admin_api/tasks/regulation_tracker.py
  - eusolicit-app/services/admin-api/src/admin_api/services/regulatory_changes_service.py
  - eusolicit-app/services/data-pipeline/tests/fixtures/n8n/crawl-eu-grants-v1-canonical.json
  - eusolicit-docs/project-context.md
---

# ATDD Checklist: Story 11.21 — Regulation Tracker N8N Workflow + Celery Retire

**Date:** 2026-05-16
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Tests written; feature not yet implemented
**Story File:** `eusolicit-docs/implementation-artifacts/11-21-regulation-tracker-n8n-workflow-celery-retire.md`
**Epic Test Design:** `eusolicit-docs/test-artifacts/test-design-epic-11.md` (E11-R-006, E11-P1-016)

---

## Preflight Summary (Step 1)

### Stack Detection

| Check | Result |
|-------|--------|
| `pyproject.toml` found | ✅ `eusolicit-app/services/admin-api/pyproject.toml` |
| `playwright.config.ts` | ✅ Present (fullstack project) |
| Backend indicators | ✅ `pyproject.toml` + `tests/conftest.py` with pytest-asyncio |
| **Detected stack** | **`backend`** (story delivers N8N template + admin-api webhook consumer) |
| Test framework | `pytest` + `pytest-asyncio` (asyncio_mode=auto) |

### Prerequisites Check

| Requirement | Status |
|-------------|--------|
| Story approved with clear acceptance criteria (7 ACs) | ✅ |
| `conftest.py` with `admin_api_session_factory`, `admin_session`, `platform_admin_token` | ✅ |
| `RegulatoryChange` ORM model (admin schema) | ✅ `src/admin_api/models/regulatory_change.py` |
| Existing regulatory changes service | ✅ `src/admin_api/services/regulatory_changes_service.py` |
| Existing Celery task to deprecate | ✅ `src/admin_api/tasks/regulation_tracker.py` |
| Reference N8N template pattern | ✅ `services/data-pipeline/tests/fixtures/n8n/crawl-eu-grants-v1-canonical.json` |

### TEA Config Flags

| Flag | Value |
|------|-------|
| `test_stack_type` | `fullstack` → backend patterns for this story |
| `tea_use_playwright_utils` | `true` (API-only profile; no `page.goto` in scope) |
| `risk_threshold` | `p2` |

---

## Generation Mode (Step 2)

**Mode selected: AI Generation** — backend stack, no browser recording needed.
All scenarios are standard backend patterns: JSON schema validation, HMAC webhook receiver, DB persistence, and file-existence smoke tests.

---

## Test Strategy (Step 3)

### Acceptance Criteria → Test Scenario Mapping

| AC | Description | Scenarios | Level | Priority |
|----|-------------|-----------|-------|----------|
| **AC1** | N8N template at `infra/n8n/templates/regulation-tracker-v1.json`, daily 02:00 UTC | File exists; valid JSON; name field; cron node; `0 2 * * *` expression | Unit/Smoke | P0 |
| **AC2** | Workflow: org-level SirmaAI agent → `regulation.change.detected` → consumer persists to `admin.regulation_changes` | Agent node present; org-level URL (no `$json.projectId`); event string in template; webhook HMAC security; DB persistence | Unit + Integration | P0 |
| **AC3** | Admin dashboard surface unchanged | Covered by existing `test_regulatory_changes.py` (33 tests) — no new ATDD tests needed | Regression | P0 |
| **AC4** | Celery task tagged deprecated | Source file contains deprecation marker text | Unit/Smoke | P0 |
| **AC5** | Staged-rollout via flag `regulation-tracker-n8n` | Feature flag name in template; Feature Flag Guard node present | Unit/Smoke | P1 |
| **AC6** | End-to-end staging test | Manual staging drill — not automated | Manual/P3 | P3 |
| **AC7** | Rollback runbook at `eusolicit-docs/runbooks/regulation-tracker-rollback.md` | File exists and is non-empty | Unit/Smoke | P1 |

---

## TDD Red Phase (Step 4)

### 🔴 Generated Failing Tests

**22 tests skip** (implementation-dependent); **2 tests fail naturally** (artifacts missing).

---

### File 1: `tests/unit/test_regulation_tracker_n8n_template.py`

**Scope:** AC1, AC2, AC5 — N8N workflow template JSON structure

#### `TestN8NTemplateStructure` (11 tests — all `@pytest.mark.skip`)

| Method | Assertion | Why RED |
|--------|-----------|---------|
| `test_template_file_exists` | `infra/n8n/templates/regulation-tracker-v1.json` exists | File not created |
| `test_template_is_valid_json` | Parses without `JSONDecodeError` | File not found |
| `test_template_name_is_regulation_tracker_v1` | `template["name"] == "regulation-tracker-v1"` | File not found |
| `test_cron_trigger_node_present` | Node with `type == "n8n-nodes-base.cron"` exists | File not found |
| `test_cron_schedule_is_daily_0200_utc` | `cronExpression == "0 2 * * *"` in Cron Trigger | File not found |
| `test_feature_flag_guard_node_present` | Node named "Feature Flag Guard" / "Flag Check" exists | File not found |
| `test_feature_flag_name_in_template` | String `"regulation-tracker-n8n"` in raw JSON | File not found |
| `test_sirmaai_agent_invocation_node_present` | Node/URL references `regulation_tracker` / `regulation-tracker` | File not found |
| `test_agent_invocation_is_org_level_not_per_tenant` | URL uses `$env.SIRMAAI_ORG_ID`; does NOT use `$json.projectId` | File not found |
| `test_webhook_emit_node_present` | String `"regulation.change.detected"` in raw JSON | File not found |
| `test_workflow_connections_cover_all_nodes` | No orphaned nodes | File not found |

---

### File 2: `tests/api/test_regulation_change_webhook.py`

**Scope:** AC2 — Standard Webhooks HMAC receiver + DB persistence + Redis idempotency

#### `TestWebhookSignatureVerification` (5 tests, P0 — security)

| Method | Assertion | Why RED |
|--------|-----------|---------|
| `test_missing_signature_returns_401` | No `webhook-signature` → 401 | Endpoint → 404 |
| `test_missing_webhook_id_returns_401` | No `webhook-id` → 401 | Endpoint → 404 |
| `test_tampered_body_invalid_signature_returns_401` | Body changed after signing → 401; enforces `hmac.compare_digest()` | Endpoint → 404 |
| `test_wrong_secret_invalid_signature_returns_401` | Wrong HMAC secret → 401 | Endpoint → 404 |
| `test_valid_signature_returns_200` | Correctly signed request → 200 | Endpoint → 404 |

#### `TestWebhookPersistence` (4 tests, P0 — core functionality)

| Method | Assertion | Why RED |
|--------|-----------|---------|
| `test_valid_webhook_stores_change_in_db` | One change → one `RegulatoryChange` row, correct fields | Endpoint → 404 |
| `test_webhook_sets_status_new_on_new_row` | New row has `status == "new"` | Endpoint → 404 |
| `test_webhook_skips_invalid_change_entries_without_crashing` | 1 valid + 1 no-source → 1 row stored, 200 returned | Endpoint → 404 |
| `test_webhook_with_no_changes_returns_200` | Empty `changes` list → 200, no rows | Endpoint → 404 |

#### `TestWebhookIdempotency` (2 tests, P1 — reliability)

| Method | Assertion | Why RED |
|--------|-----------|---------|
| `test_duplicate_webhook_id_returns_200_without_duplicate_db_row` | Same `webhook-id` twice → 200 both; 1 DB row only | Endpoint → 404 |
| `test_different_webhook_ids_with_same_payload_stores_two_rows` | Two distinct `webhook-id` values → 2 DB rows | Endpoint → 404 |

---

### File 3: `tests/unit/test_celery_deprecation_and_runbook.py`

**Scope:** AC4 + AC7 — natural RED failures (no `@pytest.mark.skip`)

| Method | Assertion | Current Status |
|--------|-----------|----------------|
| `test_celery_regulation_tracker_task_has_deprecation_comment` | `regulation_tracker.py` contains `deprecated` / `DeprecationWarning` / `@deprecated` | ❌ FAILS — no marker in source |
| `test_rollback_runbook_exists` | `eusolicit-docs/runbooks/regulation-tracker-rollback.md` exists and non-empty | ❌ FAILS — file missing |

---

### Test Infrastructure

| Component | Detail |
|-----------|--------|
| `_regulation_webhook_env_setup` | Session-scoped autouse fixture; sets `ADMIN_API_WEBHOOK_SECRET_REGULATION_TRACKER`, DB URL, Redis URL; clears settings cache; restores on teardown |
| `webhook_client_and_session` | Function-scoped; overrides `get_db_session`; rolls back; yields `(AsyncClient, AsyncSession)` — **no admin token** (inbound webhook auth = HMAC only) |
| `_sign_standard_webhooks()` | Mirrors production: `signed_content = f"{wid}.{ts}.".encode() + body`; key = `b64decode(secret)`; output = `f"v1,{b64encode(digest)}"` |
| `_make_signed_headers()` | Returns `{"webhook-id", "webhook-timestamp", "webhook-signature"}` |
| `_TEMPLATE_PATH` | `repo-root/eusolicit-app/infra/n8n/templates/regulation-tracker-v1.json` |
| `_CELERY_TASK_PATH` | `repo-root/eusolicit-app/services/admin-api/src/admin_api/tasks/regulation_tracker.py` |
| `_RUNBOOK_PATH` | `repo-root/eusolicit-docs/runbooks/regulation-tracker-rollback.md` |

### Coverage Statistics (RED Phase)

| Metric | Count |
|--------|-------|
| Total test methods | **24** |
| `@pytest.mark.skip` (implementation-gated) | 22 |
| Natural FAIL (artifact missing) | 2 |
| P0 tests | 16 |
| P1 tests | 6 |
| Acceptance criteria with automated coverage | 6 / 7 (AC6 = manual) |

---

## Acceptance Criteria Coverage Matrix

| AC | Description | Test Methods | Status |
|----|-------------|--------------|--------|
| AC1 | N8N template at correct path, daily 02:00 UTC | `test_template_file_exists`, `test_template_is_valid_json`, `test_template_name_is_regulation_tracker_v1`, `test_cron_trigger_node_present`, `test_cron_schedule_is_daily_0200_utc` | 🔴 RED (skipped) |
| AC2 | Org-level agent → `regulation.change.detected` → webhook consumer persists to DB | `test_sirmaai_agent_invocation_node_present`, `test_agent_invocation_is_org_level_not_per_tenant`, `test_webhook_emit_node_present`, all `TestWebhookSignatureVerification`, all `TestWebhookPersistence`, all `TestWebhookIdempotency` | 🔴 RED (skipped + 404) |
| AC3 | Admin dashboard unchanged | Covered by existing `test_regulatory_changes.py` (33 tests) | ✅ Existing tests (regression guard) |
| AC4 | Celery task tagged deprecated | `test_celery_regulation_tracker_task_has_deprecation_comment` | 🔴 RED (natural FAIL) |
| AC5 | Staged-rollout via `regulation-tracker-n8n` flag | `test_feature_flag_guard_node_present`, `test_feature_flag_name_in_template` | 🔴 RED (skipped) |
| AC6 | End-to-end staging test | Manual staging drill — see E11-P3-005 | Manual |
| AC7 | Rollback runbook exists | `test_rollback_runbook_exists` | 🔴 RED (natural FAIL) |

---

## Security Checklist

| Requirement | Test Coverage |
|-------------|--------------|
| `hmac.compare_digest()` required (never `==`) | `test_tampered_body_invalid_signature_returns_401` docstring + assertion |
| Missing signature → 401 | `test_missing_signature_returns_401` |
| Missing `webhook-id` (part of signed content) → 401 | `test_missing_webhook_id_returns_401` |
| Wrong secret → 401 | `test_wrong_secret_invalid_signature_returns_401` |
| Redis 7-day idempotency (replay protection) | `test_duplicate_webhook_id_returns_200_without_duplicate_db_row` |
| Webhook endpoint unauthenticated inbound (HMAC = auth) | Fixture: no admin token in `webhook_client_and_session` |
| Agent call org-level only (no per-tenant projectId) | `test_agent_invocation_is_org_level_not_per_tenant` |

---

## Implementation Guidance for Developer

### New Files to Create

| File | Purpose |
|------|---------|
| `infra/n8n/templates/regulation-tracker-v1.json` | N8N workflow template (AC1, AC2) |
| `src/admin_api/api/v1/webhooks.py` | `POST /webhooks/regulation-tracker` endpoint (AC2) |
| `eusolicit-docs/runbooks/regulation-tracker-rollback.md` | Rollback runbook (AC7) |

### Files to Modify

| File | Change |
|------|--------|
| `src/admin_api/tasks/regulation_tracker.py` | Add deprecation comment (AC4) |
| `src/admin_api/config.py` | Add `admin_api_webhook_secret_regulation_tracker: SecretStr` |
| `src/admin_api/main.py` | Register webhooks router |

### Critical Notes

1. **HMAC**: use `hmac.compare_digest(expected, received)` — NEVER `==`. Signed content: `f"{webhook_id}.{timestamp}.".encode() + body`. Key: `base64.b64decode(secret_b64)`.
2. **Redis idempotency key**: `f"webhook:regulation-tracker:{webhook_id}"`, TTL = 604800s (7 days). Check before processing; set after DB write.
3. **N8N org-level agent**: URL uses `$env.SIRMAAI_ORG_ID` — no `$json.projectId` in path.
4. **Feature flag guard**: flag name = `regulation-tracker-n8n`. Follow `crawl-eu-grants-v1-canonical.json` pattern exactly (Cron → Feature Flag Guard → Flag Check IF → Agent → Emit Webhook).
5. **Webhook endpoint is unauthenticated inbound** (no `get_admin_user` dependency).
6. **Deprecation comment**: `# DEPRECATED (Story 11.21): superseded by infra/n8n/templates/regulation-tracker-v1.json — remove in S04.30 cleanup`

---

## Next Steps (TDD Green Phase)

After implementing Story 11.21:

1. **Remove `@pytest.mark.skip`** from all 22 test methods
2. **Run tests** (requires `make infra` for integration tests):
   ```bash
   cd eusolicit-app/services/admin-api
   pytest tests/unit/test_regulation_tracker_n8n_template.py \
          tests/api/test_regulation_change_webhook.py \
          tests/unit/test_celery_deprecation_and_runbook.py \
          -v --tb=short
   ```
3. **Verify all 24 tests PASS**
4. **Run regression guard**:
   ```bash
   pytest tests/api/test_regulatory_changes.py tests/tasks/test_regulation_tracker_task.py -v
   ```
5. **Lint + type-check**: `cd eusolicit-app && make lint && make type-check`
6. **Staging E2E drill** (AC6 / E11-P3-005): manually trigger N8N → verify webhook received → verify DB row → admin dashboard reflects

---

## RED Phase Verification Log

```
collected 24 items

tests/unit/test_regulation_tracker_n8n_template.py::TestN8NTemplateStructure — 11 SKIPPED
tests/api/test_regulation_change_webhook.py::TestWebhookSignatureVerification — 5 SKIPPED
tests/api/test_regulation_change_webhook.py::TestWebhookPersistence          — 4 SKIPPED
tests/api/test_regulation_change_webhook.py::TestWebhookIdempotency          — 2 SKIPPED
tests/unit/test_celery_deprecation_and_runbook.py::test_celery_...comment    — FAILED (no deprecation marker)
tests/unit/test_celery_deprecation_and_runbook.py::test_rollback_runbook_exists — FAILED (file missing)

2 failed, 22 skipped — RED phase confirmed ✅
ruff check — All checks passed ✅
```

---

**Generated by:** TEA Master Test Architect — BMAD ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 6.6.0
