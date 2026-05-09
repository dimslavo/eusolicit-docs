---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-identify-targets
  - step-03-generate-tests
  - step-04-validate-and-summarize
lastStep: step-04-validate-and-summarize
lastSaved: '2026-05-09'
workflowType: bmad-testarch-automate
mode: bmad-integrated
scope: cluster-D-nps-feedback-api
batch: P4-a
storyKeys:
  - 20-0-third-party-nps-sdk-and-routing
detectedStack: backend
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/services/client-api/src/client_api/api/v1/nps_feedback.py
  - eusolicit-app/services/client-api/src/client_api/schemas/nps.py
  - eusolicit-app/tests/integration/_nps_helpers.py
  - eusolicit-app/tests/integration/test_nps_feedback_workspace_isolation.py
  - eusolicit-app/tests/integration/test_nps_quarter_cooldown.py
  - eusolicit-app/tests/integration/test_nps_sdk_response_id_dedup.py
  - eusolicit-app/tests/integration/test_nps_detractor_alert_isolation.py
  - eusolicit-app/tests/integration/test_nps_feedback_patch_isolation.py
---

# Automation Summary: P4-a — `nps_feedback` API route contract (cluster D, sub-run 1)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P4-a (cluster D, gap-fill API coverage, sub-run 1)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

Per the rollout-plan inventory:

> "No request/response API test for the actual FastAPI routes — POST
> feedback, GET status, etc."

The existing NPS test suite (8 files in `tests/integration/test_nps_*.py`)
is comprehensive on isolation, tier-gate, dedup, cooldown, detractor
side-effects, and PATCH window enforcement, but each file targets a
specific behavioural assertion and never documents the basic
input/output contract — i.e. **what does the route accept and return
for each of the 3 score buckets, what's the validation surface, what
cache headers does it emit?**

P4-a fills that gap with a single **route-contract** spec that locks down
the canonical request/response shape so any future schema or routing
drift surfaces immediately.

---

## Files Created

### `tests/integration/test_nps_feedback_route_contract.py` (NEW)

8 tests in 1 module, organised into 3 logical groups:

| # | Group | Scope | Description |
|---|---|---|---|
| 1 | Happy path | Promoter | Score=10 → 201, body `{bucket: "promoter", routed_to: "g2_capterra", score: 10, quarter, id, submitted_at}`, plus `Cache-Control: no-store` + `Vary: Authorization` headers (AC-5.10). |
| 2 | Happy path | Passive | Score=8 → 201, `bucket=passive`, `routed_to=passive_thanks`. Patches `redis_client` to assert `xadd` is **not** called for passive scores. |
| 3 | Happy path | Detractor | Score=4 → 201, `bucket=detractor`, `routed_to=csm_alert`. Patches `redis_client` + nulls `send_email`; asserts `xadd("eu-solicit:notifications", …)` is called once. |
| 4 | Boundary | Promoter floor | Score=9 → `bucket=promoter` (NOT passive). Locks down `_compute_bucket` inflection. |
| 5 | Boundary | Detractor ceiling | Score=6 → `bucket=detractor` (NOT passive). Locks down the second inflection. |
| 6 | Validation | Score range | Score=11 → 422 with non-empty `detail` list. Pydantic `Field(ge=0, le=10)` (`schemas/nps.py:12`). |
| 7 | Validation | Quarter format | Quarter=`"2026-Q5"` → 422. Pydantic regex `^\d{4}-Q[1-4]$` (`schemas/nps.py:15`). |
| 8 | Auth | Missing bearer | POST without `Authorization` header → 401 or 403 (depending on `HTTPBearer.auto_error`). Asserts the auth dependency fires before workspace-scope or tier-gate. |

### Shared helpers (reused, no new files)

- `tests/integration/_nps_helpers.py:register_and_verify_with_role` —
  same fixture used by every other `test_nps_*` file; mints a Pro+ user.
- `_nps_helpers.seed_workspace_for_company` + `seed_workspace_membership`
  — required because the NPS endpoint is workspace-scoped via
  `require_workspace_role()`.
- `_nps_helpers.upgrade_subscription_tier` — bumps the company's
  Subscription row to `pro_plus` so `require_pro_plus_tier` passes.

The new file's `app_client` fixture mirrors the rollback-scoped pattern
used by `test_nps_quarter_cooldown.py:43`. Two test-local helpers
(`_seed_pro_plus_user_workspace`, `_payload`, `_assert_no_store_headers`)
remove repetition without leaking into the shared helpers module.

---

## Validation

| Check | Result |
|---|---|
| `ruff check tests/integration/test_nps_feedback_route_contract.py` | ✅ All checks passed |
| `pytest --collect-only` (Python 3.13.5, host venv) | ✅ **8 tests collected** in 1 module |
| Active vs skip split | 8 active / 0 skip / 0 xfail |
| Local runtime | ❌ Host venv lacks service deps (`itsdangerous`, `bcrypt`, …) — defers to CI per saved memory `project_test_execution_environment.md`. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| `POST /api/v1/workspaces/{ws_id}/nps/feedback` route | `nps_feedback.py:283` | ✅ |
| `_compute_bucket(>=9) → promoter` | `nps_feedback.py:94` | ✅ |
| `_compute_bucket(7..8) → passive` | `nps_feedback.py:96` | ✅ |
| `_compute_bucket(<=6) → detractor` | `nps_feedback.py:98` | ✅ |
| `_default_routed_to` mapping (promoter→g2_capterra, passive→passive_thanks, detractor→csm_alert) | `nps_feedback.py:101-108` | ✅ |
| `Cache-Control: no-store` + `Vary: Authorization` | `nps_feedback.py:267-275` | ✅ |
| Pydantic score `Field(ge=0, le=10)` → 422 | `schemas/nps.py:12` | ✅ |
| Pydantic quarter `pattern=r"^\d{4}-Q[1-4]$"` → 422 | `schemas/nps.py:15` | ✅ |
| `redis_client.xadd("eu-solicit:notifications", …)` for detractor | `nps_feedback.py:200-205` | ✅ |
| `redis_client` patched at module scope | `nps_feedback.py:84` (module-level singleton) | ✅ |
| `send_email` patched at module scope | `nps_feedback.py:74-77` (module-level fallback to None) | ✅ |
| Auth dependency surfaces 401/403 before workspace/tier gates | `require_workspace_role` is `Depends`-injected with no path-param coupling (must auth first) | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make infra && make migrate-all
make test-service SVC=client-api      # if service-suite, or:
pytest tests/integration/test_nps_feedback_route_contract.py -v -m integration
```

### Risks / open questions for the CI pass

1. **`HTTPBearer.auto_error` toggle** — test #8 accepts both 401 and 403
   for the missing-bearer case. If the auth dep ever pins `auto_error=False`
   and lets the route handle "no token" itself, the assertion may need to
   tighten. Until then, the dual-accept is robust against minor FastAPI
   surface differences.

2. **`redis_client` patching with `AsyncMock`** — the production singleton
   is created at module import via `get_redis_client()` (`nps_feedback.py:84`)
   and used as `await redis_client.xadd(...)`. `AsyncMock` returns coroutines
   for every attribute, so awaiting `xadd` works without a special spec.
   If the production code ever switches to a sync client, switch the patch
   to a `MagicMock`.

3. **`send_email = None` patch in test #3** — the detractor path enqueues
   email after the Redis publish (`nps_feedback.py:229`). Setting
   `send_email=None` triggers the `pragma: no cover` early-return; the
   route still returns 201 with the right routing. If a future change makes
   `send_email` mandatory, the test would need to provide a `MagicMock`
   substitute with `.delay` instead.

4. **Cache-header case sensitivity** — HTTPX's `Headers.get()` is
   case-insensitive, so `cache-control` and `vary` work. The assertion
   error message preserves the canonical-cased header for log readability.

5. **Validation-error shape coupling** — test #6 asserts
   `detail` is a non-empty list (FastAPI's standard 422 shape). If the
   project ever installs a custom validation handler that flattens detail
   to a string, this assertion would need updating. Today there is no such
   handler in `client_api.main`.

6. **Score boundary tests are independent** — tests #4 and #5 use unique
   `sdk_response_id` values per `_payload()` call, so concurrent execution
   under pytest-xdist won't dedup-collide. Each test seeds its own user
   so the user-quarter UNIQUE constraint can't bite either.

7. **Deferred follow-ups for this surface**:
   - PATCH happy path: edit `feedback_text` within the 5-minute window
     (covered structurally by `test_nps_feedback_patch_isolation.py` but
     no explicit "200 + cache headers + body shape" lock).
   - PATCH validation: `feedback_text` < 10 chars → 422 (`schemas/nps.py:29`).
   - PATCH `routed_to` toggle without `feedback_text` — verify the
     re-dispatch is suppressed (covered tangentially by
     `test_nps_feedback_patch_isolation.py:test_detractor_patch_…` but
     not as a positive "no re-dispatch" assertion).
   - SDK dedup 200 response cache headers (existing dedup test at
     `test_nps_sdk_response_id_dedup.py` checks the body only).

---

## Coverage Delta

- Before P4-a: NPS coverage was strong on **behavioural** axes (isolation,
  tier-gate, cooldown, dedup, alerts, edit window) but had **no explicit
  contract spec** locking down request/response shape, score boundaries,
  validation 422 surface, or cache headers.
- After P4-a: **8 active** request/response contract tests. The bucket /
  routing computation is now pinned across all 3 buckets + 2 boundary
  inflections; validation surface is documented for the two Pydantic
  fields most likely to drift; cache-header AC-5.10 is no longer a
  comment-only requirement.

---

## Cluster D (P4) running totals (through P4-a)

| Sub-run | Spec file | Active | Skip / xfail |
|---|---|---:|---:|
| P4-a | `test_nps_feedback_route_contract.py` | 8 | 0 |
| **Total** | **1 spec file** | **8** | **0** |

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P4-b** — `billing.py` route-level tests (Stripe-mocked, `respx` similar
  to Story 11-7).
- **P4-c** — Cross-service notification flow (largest — needs `make up`,
  not just `make infra`).

After P4-a runs green in CI, update the rollout-plan checklist line.
