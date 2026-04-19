---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-finalize']
lastStep: 'step-05-finalize'
lastSaved: '2026-04-18'
workflowType: 'testarch-atdd'
storyId: '8-2-subscription-tier-database-schema'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/8-2-subscription-tier-database-schema.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-08.md'
  - 'eusolicit-app/services/client-api/src/client_api/core/tier_gate.py'
  - 'eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py'
  - 'eusolicit-app/services/client-api/src/client_api/models/subscription.py'
  - 'eusolicit-app/packages/eusolicit-models/src/eusolicit_models/enums.py'
  - 'eusolicit-app/services/client-api/tests/integration/test_020_migration.py'
  - 'eusolicit-app/services/client-api/tests/unit/test_tier_gate_context.py'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-1-3-postgresql-schema-design-role-based-access.md'
---

# ATDD Checklist: Story 8.2 — Subscription & Tier Database Schema

## Story Summary

**Epic:** E08 — Subscription & Billing
**Story:** 8.2 — Subscription & Tier Database Schema
**Sprint:** Current | **Priority:** P0 (Infrastructure prerequisite for all billing Stories 8.3–8.14)
**Risk-Driven:** Yes — R-001 (webhook race), R-003 (usage metering drift), R-004 (trial manipulation)

**As a** platform engineer,
**I want** a complete billing database schema (`subscriptions` extended, `tier_access_policies`,
`add_on_purchases`) in place with accurate seed data,
**So that** all subsequent billing stories (8.3–8.14) have a solid, type-safe foundation to build
on without redefining tables or guessing column names.

---

## TDD Red Phase (Current)

**Status:** 🔴 RED — All tests use `@pytest.mark.skip(reason="ATDD RED: ...")` and will be
skipped until Story 8.2 is implemented. Remove skip decorators to enter GREEN phase.

| Test Class / Function Group | File | Tests | AC | Priority | Status |
|-----------------------------|------|-------|----|----------|--------|
| `test_tier_access_policy_table_name`, `test_tier_access_policy_schema`, etc. | `tests/unit/test_tier_access_policy_model.py` | 7 | AC5 | P1 | Skipped (RED) |
| `test_add_on_purchase_table_name`, `test_add_on_purchase_schema`, etc. | `tests/unit/test_add_on_purchase_model.py` | 7 | AC5, AC6 | P1 | Skipped (RED) |
| `test_subscription_has_tier_column`, `test_subscription_has_all_seven_new_columns`, etc. | `tests/unit/test_subscription_orm_updates.py` | 9 | AC5 | P1 | Skipped (RED) |
| `test_require_paid_tier_uses_tier_column`, `test_require_professional_plus_tier_uses_tier_column`, etc. | `tests/unit/test_tier_gate.py` | 6 | AC7, AC8 | P0 | Skipped (RED) |
| `test_get_opportunity_tier_gate_queries_tier_column`, etc. | `tests/unit/test_opportunity_tier_gate.py` | 5 | AC7, AC8 | P0 | Skipped (RED) |
| `test_subscription_status_enum_has_all_six_values`, `test_tier_access_policy_dto_*`, etc. | `tests/unit/test_billing_enums_dtos.py` | 16 | AC6 | P1 | Skipped (RED) |
| `test_migration_027_*`, `test_subscriptions_*`, `test_tier_access_policies_*`, etc. | `tests/integration/test_subscription_billing_schema.py` | 25 | AC1–AC4, AC8 | P0/P1 | Skipped (RED) |
| **Total** | **7 files** | **75 tests** | **AC1–AC8** | | **All skipped** |

---

## Acceptance Criteria Coverage

| AC | Description | Test File(s) | Tests | Priority | Coverage |
|----|-------------|--------------|-------|----------|----------|
| AC1 | Migration 027 applies cleanly; all columns/tables created; downgrade reverts | `test_subscription_billing_schema.py` | 8 | P0 | Full — apply, column existence, downgrade, structure |
| AC2 | subscriptions UNIQUE on company_id, index on stripe_subscription_id, existing index preserved | `test_subscription_billing_schema.py` | 5 | P0 | Full — constraint exists, duplicate rejected, both indexes verified |
| AC3 | tier_access_policies seeded 4 rows; UNIQUE on tier | `test_subscription_billing_schema.py` | 7 | P0 | Full — count, tier names, all seed values per tier, unique constraint |
| AC4 | add_on_purchases FKs (company, opportunity) + 2 indexes | `test_subscription_billing_schema.py` | 5 | P0 | Full — table, indexes, FK enforcement for both FK columns |
| AC5 | Subscription ORM updated; TierAccessPolicy and AddOnPurchase ORM created; exported | `test_subscription_orm_updates.py`, `test_tier_access_policy_model.py`, `test_add_on_purchase_model.py` | 23 | P1 | Full — table name, schema, all columns, nullable constraints, __init__ export |
| AC6 | SubscriptionStatus, AddOnType enums; SubscriptionDTO updated; new DTOs added; exported | `test_billing_enums_dtos.py`, `test_add_on_purchase_model.py` | 19 | P1 | Full — enum values, StrEnum, DTO fields, type annotations, __init__ exports |
| AC7 | tier_gate.py and opportunity_tier_gate.py use Subscription.tier not Subscription.plan | `test_tier_gate.py`, `test_opportunity_tier_gate.py` | 11 | P0 | Full — SQL compiled output verified for all 3 gate deps + opportunity gate |
| AC8 | Unit tests for ORM models + tier gates; integration test for migration | All files | 75 | P0 | Full — all ACs covered by targeted tests |

**Total coverage: 75 tests across 8 acceptance criteria — 100% AC coverage.**

---

## Risk Mitigation Verification

### E08-R-001: Webhook Race Conditions (Score: 6 — HIGH)

| Mitigation | Test | Status |
|------------|------|--------|
| UNIQUE constraint on subscriptions.company_id prevents duplicate rows | `test_subscriptions_unique_company_id_rejects_duplicate` | RED |
| UNIQUE constraint exists in DB | `test_subscriptions_unique_company_id_constraint_exists` | RED |

### E08-R-003: Redis Usage Metering Drift (Score: 6 — HIGH)

| Mitigation | Test | Status |
|------------|------|--------|
| tier_access_policies provides authoritative per-tier limits for Story 8.8 | `test_tier_access_policies_seed_values_match_spec[*]` (4 parametrized) | RED |
| Seed values exactly match spec (ai_summaries_limit, etc.) | `test_tier_access_policies_enterprise_row_uses_unlimited_sentinel` | RED |

### E08-R-004: Trial Manipulation (Score: 4 — MEDIUM)

| Mitigation | Test | Status |
|------------|------|--------|
| One subscription per company enforced at DB level | `test_subscriptions_unique_company_id_rejects_duplicate` | RED |
| UNIQUE tier prevents duplicate seed rows | `test_tier_access_policies_unique_tier_rejects_duplicate` | RED |

---

## Test Strategy

### Stack Detection

- **Detected stack:** `fullstack` (Python backend + Node.js/Playwright frontend)
- **Story scope:** Backend only — database schema, ORM models, enums/DTOs, tier gate logic
- **No E2E/browser tests:** This story is pure infrastructure (no UI components)
- **Test framework:** pytest + pytest-asyncio + SQLAlchemy async

### Generation Mode

**AI Generation** — backend/infrastructure story, no browser recording needed.

### Test Levels

| Level | Justification | Count |
|-------|--------------|-------|
| **Unit** | ORM model metadata (no DB), enum values, DTO fields, mock-based SQL column verification | 50 |
| **Integration** | Migration apply/downgrade, column existence, constraint enforcement, FK enforcement | 25 |

### Priority Distribution

| Priority | Count | Criteria |
|----------|-------|----------|
| **P0** | 43 | Migration correctness (AC1–4), tier gate column (AC7) — blocks all billing Stories 8.3–8.14 |
| **P1** | 32 | ORM models, enums, DTOs — needed for type-safe coding of downstream stories |

---

## Failing Tests Created (RED Phase)

### Unit Tests

---

#### File: `eusolicit-app/services/client-api/tests/unit/test_tier_access_policy_model.py`

**Coverage:** AC5, AC3 (UNIQUE on tier)

| Test | AC | Priority | Failure Reason |
|------|----|----------|----------------|
| `test_tier_access_policy_table_name` | AC5 | P1 | `tier_access_policy.py` file does not exist |
| `test_tier_access_policy_schema` | AC5 | P1 | `tier_access_policy.py` file does not exist |
| `test_tier_access_policy_required_columns_present` | AC5 | P1 | File does not exist → ImportError |
| `test_tier_access_policy_tier_column_has_unique_constraint` | AC3 | P0 | File does not exist → ImportError |
| `test_tier_access_policy_unlimited_sentinel_integer_columns` | AC5 | P1 | File does not exist → ImportError |
| `test_tier_access_policy_boolean_columns` | AC5 | P1 | File does not exist → ImportError |
| `test_tier_access_policy_exported_from_models_init` | AC5 | P1 | Not yet exported from models/__init__.py |

---

#### File: `eusolicit-app/services/client-api/tests/unit/test_add_on_purchase_model.py`

**Coverage:** AC5 (AddOnPurchase ORM), AC6 (AddOnType enum)

| Test | AC | Priority | Failure Reason |
|------|----|----------|----------------|
| `test_add_on_purchase_table_name` | AC5 | P1 | `add_on_purchase.py` does not exist |
| `test_add_on_purchase_schema` | AC5 | P1 | File does not exist |
| `test_add_on_purchase_required_columns_present` | AC5 | P1 | File does not exist |
| `test_add_on_purchase_company_id_has_fk` | AC4 | P0 | File does not exist |
| `test_add_on_purchase_opportunity_id_has_fk` | AC4 | P0 | File does not exist |
| `test_add_on_purchase_exported_from_models_init` | AC5 | P1 | Not exported from models/__init__.py |
| `test_add_on_type_enum_has_exactly_three_values` | AC6 | P1 | AddOnType not in eusolicit_models.enums |
| `test_add_on_type_enum_string_values` | AC6 | P1 | AddOnType not in eusolicit_models.enums |
| `test_add_on_type_exported_from_eusolicit_models_init` | AC6 | P1 | Not exported from eusolicit_models |

---

#### File: `eusolicit-app/services/client-api/tests/unit/test_subscription_orm_updates.py`

**Coverage:** AC5 (Subscription ORM updates)

| Test | AC | Priority | Failure Reason |
|------|----|----------|----------------|
| `test_subscription_has_tier_column` | AC5 | P0 | `tier` column not yet declared in subscription.py |
| `test_subscription_tier_column_server_default_is_free` | AC5 | P1 | Column not yet added |
| `test_subscription_has_stripe_subscription_id_column` | AC5 | P1 | Column not yet added |
| `test_subscription_has_trial_start_column` | AC5 | P1 | Column not yet added |
| `test_subscription_has_trial_end_column` | AC5 | P1 | Column not yet added |
| `test_subscription_has_is_trial_boolean_column` | AC5 | P1 | Column not yet added |
| `test_subscription_has_started_at_column_nullable` | AC5 | P1 | Column not yet added |
| `test_subscription_has_expires_at_column` | AC5 | P1 | Column not yet added |
| `test_subscription_plan_column_preserved_for_backward_compat` | AC5 | P1 | Guard test — passes if plan not dropped |
| `test_subscription_has_all_seven_new_columns` | AC5 | P0 | Multiple columns missing |

---

#### File: `eusolicit-app/services/client-api/tests/unit/test_tier_gate.py`

**Coverage:** AC7 (tier_gate.py uses Subscription.tier), AC8

| Test | AC | Priority | Failure Reason |
|------|----|----------|----------------|
| `test_require_paid_tier_uses_tier_column` | AC7 | P0 | Compiled SQL references `plan` not `tier` |
| `test_require_paid_tier_grants_access_when_tier_found` | AC7 | P1 | N/A until SQL fixed |
| `test_require_professional_plus_tier_uses_tier_column` | AC7 | P0 | Compiled SQL references `plan` |
| `test_require_enterprise_tier_uses_tier_column` | AC7 | P0 | Compiled SQL references `plan` |
| `test_paid_tiers_constant_uses_subscription_tier_enum` | AC7 | P1 | PAID_TIERS constant does not exist |
| `test_tier_gate_exports_paid_tiers_constant` | AC7 | P1 | PAID_TIERS not in __all__ |

---

#### File: `eusolicit-app/services/client-api/tests/unit/test_opportunity_tier_gate.py`

**Coverage:** AC7 (opportunity_tier_gate.py uses Subscription.tier), AC8

| Test | AC | Priority | Failure Reason |
|------|----|----------|----------------|
| `test_get_opportunity_tier_gate_queries_tier_column` | AC7 | P0 | Compiled SQL references `plan` not `tier` |
| `test_get_opportunity_tier_gate_resolves_tier_correctly` | AC7 | P1 | Depends on column fix |
| `test_get_opportunity_tier_gate_falls_back_to_free_when_no_subscription` | AC7 | P1 | Depends on column fix |
| `test_opportunity_tier_gate_has_paid_tiers_constant` | AC7 | P1 | `_PAID_TIERS` not yet defined |

---

#### File: `eusolicit-app/services/client-api/tests/unit/test_billing_enums_dtos.py`

**Coverage:** AC6 (SubscriptionStatus, AddOnType, SubscriptionDTO update, TierAccessPolicyDTO, AddOnPurchaseDTO)

| Test | AC | Priority | Failure Reason |
|------|----|----------|----------------|
| `test_subscription_status_enum_has_all_six_values` | AC6 | P1 | SubscriptionStatus not in enums.py |
| `test_subscription_status_is_str_enum` | AC6 | P1 | SubscriptionStatus not in enums.py |
| `test_subscription_status_exported_from_eusolicit_models` | AC6 | P1 | Not exported |
| `test_subscription_dto_has_status_field` | AC6 | P1 | SubscriptionDTO missing `status` field |
| `test_subscription_dto_status_type_is_subscription_status` | AC6 | P1 | Field not added yet |
| `test_subscription_dto_has_trial_end_field` | AC6 | P1 | SubscriptionDTO missing `trial_end` field |
| `test_subscription_dto_has_tier_field` | AC6 | P1 | SubscriptionDTO may lack `tier` field |
| `test_tier_access_policy_dto_exists` | AC6 | P1 | TierAccessPolicyDTO not in dtos.py |
| `test_tier_access_policy_dto_has_all_required_fields` | AC6 | P1 | Not yet created |
| `test_tier_access_policy_dto_tier_uses_subscription_tier_enum` | AC6 | P1 | Not yet created |
| `test_tier_access_policy_dto_exported_from_eusolicit_models` | AC6 | P1 | Not exported |
| `test_add_on_purchase_dto_exists` | AC6 | P1 | AddOnPurchaseDTO not in dtos.py |
| `test_add_on_purchase_dto_has_all_required_fields` | AC6 | P1 | Not yet created |
| `test_add_on_purchase_dto_add_on_type_uses_enum` | AC6 | P1 | Not yet created |
| `test_add_on_purchase_dto_optional_fields_default_to_none` | AC6 | P1 | Not yet created |
| `test_add_on_purchase_dto_exported_from_eusolicit_models` | AC6 | P1 | Not exported |

---

### Integration Tests

---

#### File: `eusolicit-app/services/client-api/tests/integration/test_subscription_billing_schema.py`

**Coverage:** AC1–AC4, AC8

| Test | AC | Priority | Failure Reason |
|------|----|----------|----------------|
| `test_migration_027_file_exists` | AC1 | P0 | Migration file 027_subscription_billing_schema.py not yet created |
| `test_migration_027_has_correct_down_revision` | AC1 | P0 | File not created |
| `test_migration_027_applies_cleanly` | AC1 | P0 | Migration not yet implemented |
| `test_migration_027_downgrades_cleanly` | AC1 | P0 | Migration not yet implemented |
| `test_subscriptions_has_all_new_billing_columns` | AC1 | P0 | 7 new columns not yet added by migration |
| `test_subscriptions_tier_column_default_is_free` | AC1 | P1 | tier column does not exist |
| `test_subscriptions_unique_company_id_constraint_exists` | AC2 | P0 | Constraint not yet created by migration |
| `test_subscriptions_unique_company_id_rejects_duplicate` | AC2 | P0 | Constraint not yet present |
| `test_subscriptions_stripe_subscription_id_index_exists` | AC2 | P0 | Index not yet created |
| `test_subscriptions_stripe_customer_id_index_preserved` | AC2 | P1 | Guard test — verifies no regression from 026 |
| `test_tier_access_policies_table_exists` | AC3 | P0 | Table not yet created |
| `test_tier_access_policies_seeded_exactly_four_rows` | AC3 | P0 | Migration not run |
| `test_tier_access_policies_seed_tier_names` | AC3 | P0 | Migration not run |
| `test_tier_access_policies_seed_values_match_spec[free]` | AC3 | P0 | Seed not run |
| `test_tier_access_policies_seed_values_match_spec[starter]` | AC3 | P0 | Seed not run |
| `test_tier_access_policies_seed_values_match_spec[professional]` | AC3 | P0 | Seed not run |
| `test_tier_access_policies_seed_values_match_spec[enterprise]` | AC3 | P0 | Seed not run |
| `test_tier_access_policies_enterprise_row_uses_unlimited_sentinel` | AC3 | P0 | Seed not run |
| `test_tier_access_policies_unique_tier_constraint_exists` | AC3 | P0 | Constraint not yet created |
| `test_tier_access_policies_unique_tier_rejects_duplicate` | AC3 | P0 | Constraint not yet present |
| `test_add_on_purchases_table_exists` | AC4 | P0 | Table not yet created |
| `test_add_on_purchases_stripe_payment_intent_id_index_exists` | AC4 | P0 | Index not yet created |
| `test_add_on_purchases_composite_index_exists` | AC4 | P0 | Index not yet created |
| `test_add_on_purchases_company_id_fk_enforced` | AC4 | P0 | Table does not exist |
| `test_add_on_purchases_required_columns_exist` | AC4 | P1 | Table does not exist |

---

## Data Factories / Fixtures

No new test factories required for this story. Tests use:
- `create_async_engine(TEST_DATABASE_URL)` with direct SQL via `text()` — no ORM required
- `uuid.uuid4()` for synthetic test data
- Parametrization over `TIER_SEED_EXPECTATIONS` dict for seed value spot-checks

---

## Developer Notes — Removing RED Phase Skip Markers

When implementing Story 8.2, remove `@pytest.mark.skip` decorators in this order:

1. **Task 1 (Migration):** Remove skips from `test_subscription_billing_schema.py` after `027_subscription_billing_schema.py` is created and `alembic upgrade head` succeeds.

2. **Task 2 (Subscription ORM):** Remove skips from `test_subscription_orm_updates.py` after `subscription.py` is updated.

3. **Tasks 3–4 (New ORM models):** Remove skips from `test_tier_access_policy_model.py` and `test_add_on_purchase_model.py` after the model files are created.

4. **Task 5 (models/__init__.py):** All "exported_from_models_init" tests become active.

5. **Task 6 (eusolicit-models):** Remove skips from `test_billing_enums_dtos.py` and `test_add_on_purchase_model.py` (enum section) after `enums.py` and `dtos.py` are updated.

6. **Tasks 7–8 (Tier gates):** Remove skips from `test_tier_gate.py` and `test_opportunity_tier_gate.py` after `tier_gate.py` and `opportunity_tier_gate.py` are updated.

**Regression guard:** Existing tier gate tests in `tests/unit/test_tier_gate_context.py` and `tests/api/test_opportunity_tier_gate.py` (currently GREEN from S06.02) must remain GREEN after Tasks 7–8. Run `make test-unit` and `make test-service SVC=client-api` to verify.

---

## Quality Gate

| Gate | Target | Notes |
|------|--------|-------|
| P0 tests pass rate | 100% | Blocks all billing story dev |
| P1 tests pass rate | ≥95% | Required before Story 8.3 kickoff |
| No regression in existing tier gate tests | 100% | test_tier_gate_context.py + api/test_opportunity_tier_gate.py |
| Coverage ≥80% on new modules | `tier_access_policy.py`, `add_on_purchase.py`, updated `tier_gate.py` | `make coverage` |
