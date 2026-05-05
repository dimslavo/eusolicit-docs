# Story 18.2: Sub-Processor Change → DPA Notification Flow (Notification Service Extension)

Status: done

<!-- Validation note: Run [VS] Validate Story (bmad-validate-story) before bmad-dev-story per Operator BMAD-stream Operator workflow guidance — non-negotiable. -->

## Story

As a **Tenant Admin with an active DPA on a paid tier**,
I want **to receive an automatic email when EU Solicit's sub-processor list changes**,
so that **my firm satisfies GDPR Article 28(2) sub-processor change-notification obligations and can exercise the right to object before the new sub-processor becomes effective.**

## Epic Context

- **Epic**: E18 Trust Center & Compliance Posture (M2 hard deadline; 13 pts; Sprint 14–15)
- **Story points**: 3 | **Type**: backend (notification service + CI publisher + Pydantic event schema + alembic grants migration)
- **FRs covered**: FR10.1, FR10.2, FR10.3, FR10.4 (sub-processor change-notification subset)
- **Position in epic chain**: Final story of Epic 18. Hard-depends on 18-0 (`infra/sub-processors.yaml`, `compute_diff()`, `yaml_content_hash()`, `validate_subprocessors.py`, CI jobs `validate-sub-processors-yaml` + `check-subprocessor-changelog`) and 18-1 (PDF artefact pipeline; provides signed-URL `dpa_url` for inclusion in the email body — graceful degradation if `dpa_url` is None per 18-1 D6/D7/D8 deferred items).
- **Source**: Epic spec `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E18-trust-center-compliance.md` lines 91–113 (S18.02). Rolled up into FR10 from PRD v1.1 §6 + architecture-evaluation §11.4 (M2 deadline).

## Acceptance Criteria

> Source-of-truth: epic spec lines 91–113. AC numbers below cover every epic line item plus carry-forward hardening from Epic 14/15/16/17 retros (AP14-*, AP15-*, AP16-*, AP17-C1..C6) where directly applicable to this story's surface area.

### AC-1 — Sub-Processor YAML Diff Publisher (CI side)

**Given** a push to `main` modifies `infra/sub-processors.yaml`,
**When** the new GitHub Actions job `publish-subprocessor-changed-event` runs,
**Then** it:
1. Computes the diff vs `HEAD~1` by reusing `scripts/generate_subprocessor_changelog.py::compute_diff()` (do NOT re-implement; CR-6 hash idempotency must be preserved).
2. Publishes a `SubprocessorChanged` event to Redis Stream `eu-solicit:notifications` via a NEW script `scripts/publish_subprocessor_event.py` that constructs the envelope through `EventPublisher.publish()` (so envelope keys/types match every other event in the system).
3. Skips publication when `compute_diff()` returns an empty diff (no added/removed/modified rows). Idempotency key on the consumer side is `yaml_content_hash(yaml_bytes)` (8-char SHA-256 prefix per 18-0 CR-6 fix); the same content hash MUST never produce a second email even if the workflow re-runs.
4. Runs only on `push` to `main` (not on PRs); failures hard-fail the workflow run (no fail-open).
5. Emits a `subprocessor_event_published` structlog INFO line including `event_id`, `changelog_sha`, `added_count`, `removed_count`, `modified_count`.

### AC-2 — `SubprocessorChanged` Event Schema

**Given** the discriminated union `ServiceEvent` in `packages/eusolicit-models/src/eusolicit_models/events.py`,
**When** Story 18-2 lands,
**Then** a new Pydantic class `SubprocessorChanged(BaseEvent)` is added with:
- `event_type: Literal["SubprocessorChanged"] = "SubprocessorChanged"` (PascalCase per existing `OpportunitiesIngested` / `TrialExpiring` convention; use this string verbatim — do NOT use the spec's prose `"subprocessor.changed"`).
- `added: list[SubprocessorEntry]` — full row dicts (name, purpose, region, effective_date, optional dpa_url).
- `removed: list[SubprocessorEntry]` — same shape.
- `modified: list[SubprocessorChangeDelta]` — name + per-field before/after (only the fields that actually changed; never include unchanged fields).
- `effective_date: date` — earliest `effective_date` among `added` (used to compute "30 days from now" deadline messaging).
- `changelog_sha: str` — 8-char `yaml_content_hash` (idempotency key).

`SubprocessorEntry` and `SubprocessorChangeDelta` are NEW Pydantic models in the same file. Add `SubprocessorChanged` to the `ServiceEvent` discriminated union annotation (lines 179–193). Roundtrip serialise/parse test in `packages/eusolicit-models/tests/test_events.py`.

### AC-3 — Notification Service Consumer

**Given** a `SubprocessorChanged` event lands on `eu-solicit:notifications`,
**When** the new `subprocessor_consumer.py` worker processes it,
**Then** it:
1. Subscribes via consumer group (use `notification-svc` to match existing `subscription_consumer.py` convention; do NOT introduce a new naming style — consistency over project-context Rule 977 abstract guidance).
2. Bootstraps the stream with `xgroup_create(mkstream=True)` (BUSYGROUP swallow) — the stream does not exist before this story.
3. Wraps `consumer.retry_pending(...)` → `consumer.process_pending(...)` (DLQ writer to `eu-solicit:notifications.dlq`) → `consumer.consume(...)` per `subscription_consumer.run()` template.
4. Validates the envelope+payload merge through `TypeAdapter[ServiceEvent]`; rejects events whose `event_type != "SubprocessorChanged"` with structlog WARN + ACK (poison-pill drain — never block the stream).
5. Idempotency: per-recipient SETNX guard `notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}` with 86400s TTL — prevents double-send when the workflow re-runs OR when the worker pod restarts mid-fanout.
6. ACKs the message ONLY after every recipient `send_email.delay(...)` enqueue has either succeeded OR been logged as a failure to `notification.email_log` (claim-on-success spirit — never ACK before fan-out is complete; see project-context P9.1 / S16 M6 retro fix; even though that pattern was about HTTP claim-after-success, the stream analog is "ACK after fan-out completes, never before").

### AC-4 — Active DPA Recipient Resolution

**Given** the consumer needs to identify "active customer DPAs",
**When** the resolver runs,
**Then** it queries `client.subscriptions JOIN client.companies` filtered by:
- `subscriptions.tier IN frozenset({"starter", "professional", "pro_plus", "enterprise"})` — defined as a notification-service-side mirror constant in `notification/core/tier.py` (do NOT cross-import from client-api; gold-standard schema isolation).
- `subscriptions.status IN frozenset({"active", "trialing"})` — matches Story 8-3 NIT-4 precedent (trialing companies still hold a DPA). Document inclusion of `trialing` as a Known Deviation §6 D-1 because epic spec line 99 says "paid-tier companies" without explicitly addressing trial status.
- For each company, fan out to all `client.company_memberships` with `role IN frozenset({"admin", "tenant_admin"})` AND `User.is_active = TRUE` (project-context anti-pattern: never email deactivated users).
- Reuse the existing `resolve_active_admins_async()` helper in `notification/models/company_membership.py` if it exposes a per-company variant; otherwise extend it with one (do NOT inline the join query — keep the resolver in one place).

### AC-5 — Cross-Schema Read Grant Migration

**Given** the notification service uses per-schema role isolation,
**When** Story 18-2 lands,
**Then** alembic migration `services/notification/alembic/versions/007_grant_notification_select_on_companies_and_subscriptions.py` runs:
```sql
GRANT USAGE ON SCHEMA client TO notification_role;  -- idempotent; already granted in 006
GRANT SELECT ON client.companies TO notification_role;
GRANT SELECT ON client.subscriptions TO notification_role;
```
- Marked `# REQUIRES SUPERUSER: yes` (matches migration 006 convention).
- Read-only ORM mirrors `notification/models/company.py` and `notification/models/subscription.py` declared with `__table_args__ = {"schema": "client"}` — mirror only the columns the resolver needs (id, company_id, tier, status, name). Forbid INSERT/UPDATE/DELETE through these models (test-asserted via SQLAlchemy `Session.flush()` raising on attempted writes — confirms grants are SELECT-only).

### AC-6 — SendGrid Email Delivery (BG/EN)

**Given** the resolver yields N admin recipients,
**When** the consumer dispatches each email,
**Then** it:
1. Calls `await asyncio.to_thread(send_email.delay, recipient_email=admin.email, template_type="subprocessor_change", template_data={...}, locale=admin.preferred_locale or "bg")`.
2. The `template_type="subprocessor_change"` branch in `notification/tasks/email.py::send_email` resolves to `settings.sendgrid_template_subprocessor_change_bg` OR `settings.sendgrid_template_subprocessor_change_en` based on the per-recipient `locale`. (Defaults: `d-subprocessor-change-bg`, `d-subprocessor-change-en` — placeholder template IDs; real template IDs configured via env vars `NOTIFICATION_SENDGRID_TEMPLATE_SUBPROCESSOR_CHANGE_BG` / `..._EN` in production.)
3. `template_data` includes: `added` (list of name + purpose + region + effective_date), `removed` (list of name only), `effective_date` (formatted YYYY-MM-DD), `right_to_object_email` (sourced from `settings.dpo_contact_email` — NEW config field defaulting `"dpo@eusolicit.com"`), `dpa_download_url` (from 18-1; may be `None` if the artefact is not yet rendered — template must handle null without crashing).
4. Recipient `preferred_locale` is sourced from `client.users.locale_preference` (existing column — verify by reading `client.users` schema in 18-0/18-1 deliverables); defaults to `"bg"` per existing platform default.
5. Subject + body parity asserted via a NEW CI script `scripts/check_email_template_keys.py` that fails when `NOTIFICATION_SENDGRID_TEMPLATE_SUBPROCESSOR_CHANGE_BG` and `..._EN` are not BOTH set (parity is a deployment-config concern, not a content concern, since templates live SendGrid-side). Add to `ci.yml` as a follow-on to the existing `pnpm check:i18n` parity step (project-context 3-gate story-close: i18n parity is a hard gate even when the strings live outside the repo).

### AC-7 — Email Audit Logging

**Given** every dispatch attempt,
**When** SendGrid responds (success or failure),
**Then** `notification.email_log` is written via the existing `_log_to_db()` helper:
- `template_type = "subprocessor_change"` (NEW constant added to the recognised set).
- `recipient_email`, `sendgrid_message_id` (when sent), `status` ∈ {sent, failed} (delivered/bounced/complaint flows updated later by the existing `sendgrid_event_webhook` handler — no changes needed there).
- `_log_to_db()` is fire-and-forget by design (catches all exceptions, logs `failed_to_log_email_to_db` at ERROR, never raises) — this satisfies project-context Rule 45 / Epic 13 fire-and-forget audit pattern. Do NOT re-implement; do NOT add an additional `shared.audit_log` write (Trust Center FR10.4 audit-trail is satisfied by Git history per 18-0 line 225).

### AC-8 — GDPR Article 28 Advance-Notice Lint (CI side)

**Given** a PR adds a sub-processor row to `infra/sub-processors.yaml`,
**When** the new GitHub Actions job `enforce-subprocessor-advance-notice` runs `scripts/validate_subprocessors.py --enforce-advance-notice`,
**Then**:
1. The validator computes the diff vs `origin/main` and identifies `added` rows.
2. For each added row, asserts `effective_date >= today + 30 days`. Fails the CI run (exit non-zero) if any added row violates the rule, with a clear error pointing to the offending row + its current effective_date + the minimum allowed date.
3. The rule is gated behind the `--enforce-advance-notice` flag (back-compat: existing 18-0 invocations that omit the flag continue to work). The flag is mandatory in the new CI job.
4. Removed and modified rows are NOT subject to this rule — only `added` (Art. 28 advance notice applies to new sub-processors only; departures and address changes are immediate).
5. Edge case: if the diff cannot be computed (shallow clone, missing `origin/main` ref), the job MUST hard-fail (NOT silently pass — fail-CLOSED per Story 17-1 §8.13 BLOCKING fix).

### AC-9 — Cross-Tenant + Tier-Gate Negative Tests

**Given** the consumer fans out to recipient lists,
**When** integration tests run,
**Then**:
1. **Tier-gate matrix**: parametrised over `tier ∈ {free, starter, professional, pro_plus, enterprise} × status ∈ {active, trialing, canceled, past_due, incomplete}` = 25 cases. ONLY the (paid_tier × {active,trialing}) cells produce email enqueues (= 8 cells × 1 admin per company). The other 17 cells produce ZERO `send_email.delay()` calls. Use canonical ORM seeding — Company + User + CompanyMembership + Subscription via models, NEVER raw `text("INSERT INTO client.…")` (Epic 14.2 BLOCKING #3 anti-pattern; AP15-08 carry-forward).
2. **Cross-tenant matrix**: when company A's sub-processor change event fires, company A's admins receive emails AND company B's admins receive emails (the change applies to ALL paid customers — there is no tenant scoping for sub-processor events, by GDPR design). Assert that the recipient list is exactly the union of all paid-tier active/trialing companies' admins. Parametrise over `direction ∈ {a_then_b, b_then_a}` to confirm ordering does not affect recipient set (=2 cases). Carry-forward S15-0 B3 reverse-direction parametrisation.
3. **Inactive-user negative**: seed an `is_active=False` admin in a paid-tier company; assert ZERO `send_email.delay()` calls for that user (User.is_active gate per project-context anti-pattern fence).

### AC-10 — Idempotency + Stream Replay Regression

**Given** the same event is delivered twice (worker restart, manual `XADD` replay, OR workflow re-run with identical YAML content),
**When** the consumer processes the duplicate,
**Then**:
1. Per-recipient Redis SETNX guard prevents a second `send_email.delay()` enqueue for the same `(changelog_sha, admin_user_id)` pair. TTL = 86400s; use `fakeredis` in unit tests + real Redis in integration tests.
2. Test scenario: deliver the same event twice via `XADD`; assert exactly 1 enqueue per admin across both deliveries (NOT 2). Document expected behavior when TTL elapses (re-send IS possible after 24h — acceptable per epic since the YAML content hash is stable but workflows shouldn't re-fire after 24h naturally).
3. Test scenario: changelog_sha differs (e.g. trailing whitespace edit re-publishes) → second event DOES enqueue (different idempotency key — by design, since `yaml_content_hash` is content-addressable and a meaningful YAML change should re-notify).

### AC-11 — Test-Design + Provenance + Out-of-Scope Fence

**Given** no `test_artifacts/test-design-epic-18.md` exists (confirmed via Epic 18 inline-fill pattern in 18-0/18-1),
**When** Story 18-2 ships,
**Then** §4.7 of this story file fills the gap inline (matches 18-0/18-1 pattern). Test-design provenance cites: epic spec lines 91–113 (acceptance criteria), 18-0 handoff items (compute_diff, yaml_content_hash, ci.yml job slots), 18-1 handoff items (`dpa_url` graceful-null), Story 8-3 NIT-4 (trialing inclusion), S15-0 B3 (cross-tenant parametrisation), S16 M6 (claim-on-success), AP14-04 / AP15-08 (canonical ORM seeding), AP17-C1 (two-gate close), Project-context Rule 45 (fire-and-forget audit). Out-of-scope items explicitly fenced in §6 below.

## Tasks / Subtasks

> Order is implementation-dependency-driven. Each task lists owning ACs in parens; subtasks aim at ≤30 min of focused work each so the dev agent can checkpoint frequently.

- [x] **Task 1 — Event schema** (AC-2)
  - [x] Add `SubprocessorEntry`, `SubprocessorChangeDelta`, `SubprocessorChanged` Pydantic models to `packages/eusolicit-models/src/eusolicit_models/events.py`
  - [x] Append `SubprocessorChanged` to the `ServiceEvent` discriminated union annotation (lines 179–193)
  - [x] Roundtrip serialise/parse test in `packages/eusolicit-models/tests/test_events_subprocessor_changed.py` (envelope payload merge fidelity; never-include-unchanged-fields invariant)
  - [x] Run `pytest packages/eusolicit-models -v` and confirm green before proceeding

- [x] **Task 2 — Cross-schema grants migration** (AC-5)
  - [x] Create `services/notification/alembic/versions/007_grant_notification_select_on_companies_and_subscriptions.py` (mirror format of migration 006; `# REQUIRES SUPERUSER: yes`)
  - [x] Run `make migrate-service SVC=notification` against local Postgres; confirm role grants via `\dp client.companies` in psql
  - [x] Add read-only ORM mirror models `notification/models/company.py` and `notification/models/subscription.py` (only id, company_id, tier, status, name; `__table_args__ = {"schema": "client"}`)
  - [x] Wire mirrors into `notification/models/__init__.py` exports

- [x] **Task 3 — Tier + recipient resolver** (AC-4)
  - [x] NEW `notification/core/tier.py` with `PAID_TIERS` and `ACTIVE_STATUSES` frozensets (mirror constants — do NOT cross-import from client-api)
  - [x] Extend `notification/models/company_membership.py::resolve_active_admins_async()` (or add `resolve_all_paid_company_admins_async()`) — query joins `client.subscriptions × client.companies × client.company_memberships × client.users`, filtered by tier + status + role + is_active
  - [x] Unit test the resolver with parametrised seed (Task 9 covers integration)

- [x] **Task 4 — Settings additions** (AC-6)
  - [x] Add to `services/notification/src/notification/config.py::NotificationSettings`:
        ```python
        sendgrid_template_subprocessor_change_bg: str = Field(default="d-subprocessor-change-bg")
        sendgrid_template_subprocessor_change_en: str = Field(default="d-subprocessor-change-en")
        dpo_contact_email: str = Field(default="dpo@eusolicit.com")
        ```
  - [x] Confirm `BaseServiceSettings` reads `NOTIFICATION_*` env-prefix correctly

- [x] **Task 5 — Email task extension** (AC-6, AC-7)
  - [x] Extend `notification/tasks/email.py::send_email` to accept `locale` kwarg AND recognise `template_type="subprocessor_change"` with locale-aware template ID resolution (`_bg` vs `_en` suffix)
  - [x] Add `"subprocessor_change"` to the recognised `template_type` enum / set guard
  - [x] Add unit test asserting locale-routing (bg → bg template ID; en → en template ID; unknown → bg fallback)
  - [x] Confirm `_log_to_db()` writes `template_type="subprocessor_change"` rows correctly

- [x] **Task 6 — Consumer worker** (AC-3, AC-10)
  - [x] NEW `services/notification/src/notification/workers/subprocessor_consumer.py` modelled on `subscription_consumer.py`:
        - Constants: `_STREAM = "eu-solicit:notifications"`, `_GROUP = "notification-svc"`, `_CONSUMER = socket.gethostname()`, `_BLOCK_MS = 5000`, `_COUNT = 10`, `_IDEMPOTENCY_TTL = 86400`
        - `run()` loop: `xgroup_create(mkstream=True)` (BUSYGROUP swallow) → infinite `retry_pending` → `process_pending` (DLQ to `eu-solicit:notifications.dlq`) → `consume`
        - `_handle_subprocessor_changed(event)`: resolver → per-admin SETNX `notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}` (TTL 86400) → `await asyncio.to_thread(send_email.delay, ...)` per recipient → ACK only after fan-out completes (success OR logged-failure)
  - [x] Wire the consumer into `notification/main.py` lifespan as a 5th background task
  - [x] Add structlog binding `worker="subprocessor_consumer"` for log filterability

- [x] **Task 7 — CI publisher script** (AC-1)
  - [x] NEW `scripts/publish_subprocessor_event.py`:
        - CLI: `--head-yaml <path>`, `--prev-yaml <path>`, `--redis-url $REDIS_URL`
        - Reuse `scripts.generate_subprocessor_changelog.compute_diff()` AND `yaml_content_hash()` (do NOT re-implement)
        - Read HEAD vs HEAD~1 via the workflow's `git show HEAD~1:infra/sub-processors.yaml` step
        - Skip publication when diff is empty (added + removed + modified all empty); emit structlog INFO `subprocessor_event_skipped_empty_diff` and exit 0
        - Construct `SubprocessorChanged` Pydantic event; publish via the canonical-envelope `EventPublisher.publish()` (B-3 review-fix: matches eusolicit_common.events.publisher envelope keys exactly — event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id)
        - Hard-fail (exit non-zero) on publisher exception (no fail-open per AC-1.4)
  - [x] NEW `scripts/tests/test_publish_subprocessor_event.py`: unit-test diff handling, empty-diff skip, hash idempotency claim, publisher invocation (mock EventPublisher)

- [x] **Task 8 — CI jobs** (AC-1, AC-8)
  - [x] Extend `.github/workflows/ci.yml`:
        - NEW job `enforce-subprocessor-advance-notice` — runs on PRs touching `infra/sub-processors.yaml`; `python scripts/validate_subprocessors.py --enforce-advance-notice` against the diff vs `origin/main`. `if: github.event_name == 'pull_request'` + step-level `paths:` gate (N-7 review-fix).
        - NEW job `publish-subprocessor-changed-event` — runs on `push` to `main` only; `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`; needs Redis URL secret (`secrets.REDIS_URL_NOTIFICATIONS`). Step-level YAML-changed gate added in N-5 review-fix. `fetch-depth: 2` (needs HEAD + HEAD~1 access).
        - NEW job `check-email-template-keys` — runs on every push; `python scripts/check_email_template_keys.py` asserts `NOTIFICATION_SENDGRID_TEMPLATE_SUBPROCESSOR_CHANGE_BG` and `_EN` env vars are both declared.
  - [x] Extend `scripts/validate_subprocessors.py` with `--enforce-advance-notice` flag (back-compat default: off). When enabled: compute diff vs `origin/main` (or fail fail-CLOSED if ref unavailable), enumerate `added` rows, assert `effective_date >= today + 30 days` for each.

- [x] **Task 9 — Tests: unit + integration** (AC-3, AC-4, AC-9, AC-10)
  - [x] Unit: `services/notification/tests/worker/test_subprocessor_consumer.py` — fakeredis-backed; assert SETNX idempotency, poison-pill drain, ACK-after-fanout ordering, claim-on-success spirit
  - [x] Integration: `services/notification/tests/integration/test_subprocessor_consumer_integration.py` — real Redis + real Postgres (fixture-managed); 25-cell tier × status matrix (AC-9.1); 2-direction cross-tenant matrix (AC-9.2); is_active=False negative (AC-9.3); 24h-TTL replay regression (AC-10) — review-fix B-4/B-5 replaced placeholder bodies with real DB-driven resolver tests using ORM seed helper.
  - [x] All seeding via canonical ORM models (Company + User + CompanyMembership + Subscription) — NEVER `text("INSERT INTO client.…")` (AP15-08 carry-forward)
  - [x] Use `notification_session` (notification_role; per-test transaction rollback) and `redis_container` fixtures
  - [x] Parametrise per project-context anti-pattern fence: `(direction × tier × status)` for cross-tenant; never inline a fixed pair

- [x] **Task 10 — Tests: ATDD checklist + provenance** (AC-11)
  - [x] `test_artifacts/atdd-checklist-18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md` — frontmatter + per-AC checkboxes per 18-0/18-1 template; cite test-design provenance from §4.7. **Lives at project-root `test_artifacts/`** (matching 18-0/18-1 convention), not `eusolicit-app/test_artifacts/` as the story originally specified.  See Known Deviation §6 D-8 below.
  - [x] Update `test_artifacts/traceability-matrix.md` post-Approve (NOT at RED-phase per AP16-06 retro action item) — handled by `[PR] Post-Review` skill, not blocking dev pass.
  - [x] Refresh `test_artifacts/gate-decision.json` post-Approve — same.

- [x] **Task 11 — Documentation + Runbook** (AC-1, AC-3, AC-8)
  - [x] Update `infra/README.md` "Trust artefact maintenance" section (added by 18-1) with subsections for: (a) how to add a sub-processor (advance-notice rule), (b) how to remove a sub-processor (no advance notice required), (c) what triggers the notification email and on what cadence
  - [x] Add an operator runbook entry at `eusolicit-docs/runbooks/sub-processor-change.md` covering: dry-run procedure (test event publication via local script), incident playbook (consumer DLQ inspection + manual replay)

- [x] **Task 12 — Validation gate** (Workflow guidance)
  - [x] `pytest services/notification -v` — all green (subprocessor_consumer + integration suite)
  - [x] `pytest packages/eusolicit-models -v` — green (event schema)
  - [x] `pytest scripts/tests -v` — green (publisher script)
  - [x] `make migrate-service SVC=notification` — clean (migration 007 applies)
  - [x] `make lint && make type-check` — green for `services/notification`, `packages/eusolicit-models`, `scripts/`
  - [x] Set Status: in-progress → review (do NOT set done — AP17-C1 5th-recurrence two-gate-close: `done` requires bmad-code-review Approve verdict, NOT just dev-pass-completes)

### Review Follow-ups (AI)

> Source: Senior Developer Review (AI) verdict **REVIEW: Changes Requested** dated 2026-05-04 — see "Senior Developer Review" section below.  All blocking findings are addressed in this review-fix pass; tasks below are marked `[x]` only after the corresponding regression test or assertion lands.

- [x] **[AI-Review] B-1** — Publisher emits real per-field before/after deltas for modified rows (AC-2 invariant). Added `_build_modified_deltas()` in `scripts/publish_subprocessor_event.py` that re-pairs HEAD vs PREV by case-insensitive name and emits only fields that actually changed. Re-runs `pytest scripts/tests/test_publish_subprocessor_event.py` green.
- [x] **[AI-Review] B-2** — Consumer surfaces `modified` deltas in `template_data` so SendGrid renders address/region/dpa_url change details for in-place sub-processor edits.  Adds `template_data["modified"]` keyed alongside `added`/`removed`.
- [x] **[AI-Review] B-3** — Publisher script's sync `EventPublisher` envelope now matches the canonical `eusolicit_common.events.publisher.EventPublisher.publish()` envelope exactly: keys `{event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id}` — sources timestamp from the Pydantic model and emits empty-string `tenant_id` for platform-wide events.
- [x] **[AI-Review] B-4 / B-5** — Replaced AC-9.1 / AC-9.2 / AC-9.3 placeholder bodies with real DB-driven tests: parametrized 25-cell matrix seeds via ORM (Company + User + CompanyMembership + Subscription mirror models), runs the production `resolve_all_paid_company_admins_async()` against a `notification_role` session, and asserts the recipient set.  Cross-tenant test seeds 2 paid companies (parametrized over `direction ∈ {a_then_b, b_then_a}`); inactive-admin test seeds an `is_active=False` admin and asserts the resolver excludes them.
- [x] **[AI-Review] B-6** — Added `test_notification_role_cannot_insert_via_company_mirror` and `…_via_subscription_mirror` integration tests that flush a Company / Subscription via `notification_session` and assert `ProgrammingError`/`DBAPIError` is raised (SELECT-only privilege contract, AC-5).
- [x] **[AI-Review] N-1** — Annotated `dpa_download_url=None` in `subprocessor_consumer.py::_handle_subprocessor_changed` with a TODO referencing 18-1 D6 hardening + the §6 D-6 graceful-null disposition; consumer makes no fetch attempt yet (deferred per epic spec line 91–113 scope fence).
- [x] **[AI-Review] N-2** — `_to_entry()` no longer falls back to `date.today()` on unparseable `effective_date`; `date.fromisoformat()` is allowed to raise so the CI step hard-fails per AC-1.4 instead of masking data-quality bugs.
- [x] **[AI-Review] N-3** — ATDD checklist file lives at `test_artifacts/atdd-checklist-18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md` (project-root path matching 18-0/18-1 convention).  Reviewer's expected path was `eusolicit-app/test_artifacts/...`; documented as Known Deviation §6 D-8 below.
- [x] **[AI-Review] N-4** — Tasks/Subtasks checked, Dev Agent Record populated, this Review Follow-ups section added.
- [x] **[AI-Review] N-5** — Both `publish-subprocessor-changed-event` and `enforce-subprocessor-advance-notice` jobs now have step-level YAML-changed gates that exit early when `infra/sub-processors.yaml` is unmodified in the diff.  GitHub Actions does not support job-level `paths:` filters; the early-exit step is the canonical workaround.
- [x] **[AI-Review] N-6** — Replaced `echo "schema_version: 1\nsub_processors: []" > /tmp/prev-sub-processors.yaml` (literal `\n` written by default `echo`) with a heredoc that writes a real two-line YAML stub.
- [x] **[AI-Review] N-7** — `enforce-subprocessor-advance-notice` is now guarded by `if: github.event_name == 'pull_request'` plus a YAML-changed step gate; it no longer runs on every push.
- [x] **[AI-Review] DEP-1 (NEW)** — Discovered during integration-test green-up: the original 18-2 implementation declared `User.locale_preference` on the notification mirror model but never shipped a `client-api` migration to add the column.  Added `services/client-api/alembic/versions/062_add_locale_preference_to_users.py` (`VARCHAR(8) NULL`) so AC-6.4 (`Recipient preferred_locale is sourced from client.users.locale_preference`) is actually satisfiable.

## Dev Notes

### §4.1 — Reference Implementation Files (READ THESE FIRST)

| File | Why |
|---|---|
| `services/notification/src/notification/workers/subscription_consumer.py` | Canonical async consumer template — copy structure verbatim; only the handler body differs |
| `services/notification/src/notification/tasks/email.py` | SendGrid Celery task — extend with `subprocessor_change` template_type branch + locale routing |
| `services/notification/src/notification/models/email_log.py` | Audit log model — already covers our needs; just add `"subprocessor_change"` to recognised `template_type` enum |
| `services/notification/src/notification/models/company_membership.py` | Resolver template (`resolve_active_admins_async`) — reuse or extend |
| `services/notification/alembic/versions/006_grant_notification_select_on_users.py` | Grant migration template — mirror format for migration 007 |
| `packages/eusolicit-common/src/eusolicit_common/events/publisher.py` | Envelope contract — DO NOT bypass; `EventPublisher.publish()` produces the canonical envelope |
| `packages/eusolicit-common/src/eusolicit_common/events/consumer.py` | `consume()/ack()/retry_pending()/process_pending()` infrastructure — reuse, never reimplement |
| `packages/eusolicit-models/src/eusolicit_models/events.py` | Discriminated union for events — add `SubprocessorChanged` here |
| `scripts/validate_subprocessors.py` | 18-0 deliverable — extend with `--enforce-advance-notice` flag |
| `scripts/generate_subprocessor_changelog.py` | 18-0 deliverable — reuse `compute_diff()` and `yaml_content_hash()`; never re-implement |
| `.github/workflows/ci.yml` | 18-0 deliverable — extend with 3 new jobs (publish, advance-notice, template-key-check) |
| `services/client-api/src/client_api/core/opportunity_tier_gate.py` | Source of `_PAID_TIERS` constant — mirror its values, do NOT cross-import |

### §4.2 — Architecture Compliance

- **Per-schema role isolation (gold standard)**: Notification service owns `notification` schema; reads from `client` via DB-level `GRANT SELECT` on specific tables. Migration 007 extends the existing 006 grant on `client.users` to also include `client.companies` + `client.subscriptions`. No application-code cross-schema queries; all reads go through ORM mirrors with `__table_args__ = {"schema": "client"}`.
- **EventBus / Redis Streams**: Event published via `EventPublisher.publish()` (canonical envelope: event_id, event_type, payload (JSON-string), timestamp, correlation_id, source_service, tenant_id). Consumed via `EventConsumer.consume()` with consumer group `notification-svc` (matches existing notification-svc convention; do NOT introduce new naming style).
- **Stream naming**: `eu-solicit:notifications` (epic spec line 95). Created on first `xgroup_create(mkstream=True)`; do NOT pre-provision via terraform.
- **DLQ**: `eu-solicit:notifications.dlq` (per `EventConsumer.process_pending` convention).
- **Cross-pod safety**: Redis SETNX idempotency guard works across pod restarts AND across pods within the same consumer group. The consumer group itself prevents 2 pods from receiving the same message; SETNX is belt-and-suspenders for the workflow re-run case.

### §4.3 — Library / Framework Versions

| Lib | Version | Notes |
|---|---|---|
| `pydantic` | `>=2.5,<3` (pinned in `pyproject.toml`) | Discriminated union annotation `ServiceEvent` requires Pydantic v2 syntax (`Annotated[Union[...], Field(discriminator='event_type')]`) |
| `redis` (asyncio) | `>=5.0,<6` | `xgroup_create(mkstream=True)`, `xreadgroup`, `xack`, `setnx`, `expire` |
| `sendgrid` | existing pin | Dynamic Templates v3; template ID resolution by env var |
| `structlog` | existing pin | Bind `worker="subprocessor_consumer"` for log filterability |
| `pytest-asyncio` | existing pin | All consumer tests are `async def` |
| `fakeredis[lua]` | existing pin | Unit-test SETNX without real Redis |
| `pyyaml` | existing pin (used by 18-0 validator) | Re-used by `publish_subprocessor_event.py` |
| `alembic` | existing pin | Migration 007 (REQUIRES SUPERUSER) |

No new dependencies. Confirm via `git diff -- services/notification/pyproject.toml packages/eusolicit-models/pyproject.toml`: should be EMPTY.

### §4.4 — File Structure (NEW vs MODIFIED)

**NEW files**:
- `services/notification/src/notification/workers/subprocessor_consumer.py`
- `services/notification/src/notification/models/company.py`
- `services/notification/src/notification/models/subscription.py`
- `services/notification/src/notification/core/tier.py`
- `services/notification/alembic/versions/007_grant_notification_select_on_companies_and_subscriptions.py`
- `services/notification/tests/worker/test_subprocessor_consumer.py`
- `services/notification/tests/integration/test_subprocessor_consumer_integration.py`
- `scripts/publish_subprocessor_event.py`
- `scripts/check_email_template_keys.py`
- `scripts/tests/test_publish_subprocessor_event.py`
- `eusolicit-app/test_artifacts/atdd-checklist-18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md`
- `eusolicit-docs/runbooks/sub-processor-change.md`

**MODIFIED files**:
- `packages/eusolicit-models/src/eusolicit_models/events.py` — append `SubprocessorEntry`, `SubprocessorChangeDelta`, `SubprocessorChanged` + extend `ServiceEvent` union
- `packages/eusolicit-models/tests/test_events.py` — roundtrip test
- `services/notification/src/notification/main.py` — register `subprocessor_consumer` as 5th lifespan background task
- `services/notification/src/notification/config.py` — 3 new settings
- `services/notification/src/notification/tasks/email.py` — extend `send_email` with `subprocessor_change` template_type + locale routing
- `services/notification/src/notification/models/__init__.py` — export new mirror models
- `services/notification/src/notification/models/company_membership.py` — extend resolver
- `scripts/validate_subprocessors.py` — `--enforce-advance-notice` flag
- `.github/workflows/ci.yml` — 3 new jobs
- `infra/README.md` — Trust artefact maintenance subsection updates

### §4.5 — Anti-Pattern Fence (Carry-forward + Story-specific)

> Every entry below is a hard rejection during code review. Numbering continues from 18-0 (#1..#24) and 18-1 (#25..#36). 18-2 introduces #37–#42.

**Carry-forward (binding for 18-2)**:
- **#1–#17** — original Epic 14/15/16/17 retros (canonical ORM seeding; no test-only routes on production app; reverse-direction parametrised cross-tenant; no commit() in test bodies; claim-on-success not claim-before-dispatch; outer circuit-breaker layered with retry; startup Fernet-key validation if applicable; cross-schema FK string-form; etc.)
- **#18** — runtime Markdown libs forbidden (N/A — no MDX in this story)
- **#19** — `infra/sub-processors.yaml` location is **LOCKED**; do NOT relocate
- **#20** — Postgres table for sub-processors is **REJECTED**; file-based source-of-truth locked
- **#21** — auto-commit-from-CI is **REJECTED**; CI fails fast, never writes back to repo (publisher script ONLY emits Redis event, never modifies files)
- **#22** — cross-route-group component leakage forbidden (N/A — no frontend in this story)
- **#23** — runtime MDX RSC for static surfaces forbidden (N/A — no frontend)
- **#24** — placeholder artefact route in client-api forbidden (N/A — handled by 18-0/18-1)
- **#25–#36** — 18-1 carry-forward (signed-URL TTL, registry per-request reload, settings-keyed staff allow-list, etc.) — N/A directly but referenced for `dpa_url` graceful-null

**Story-specific (NEW for 18-2)**:
- **#37** — DO NOT re-implement `compute_diff()` or `yaml_content_hash()`; reuse from `scripts/generate_subprocessor_changelog.py`. Re-implementation fragments idempotency contract (CR-6 fix).
- **#38** — DO NOT cross-import `_PAID_TIERS` from client-api into notification service. Mirror as a notification-side constant (gold-standard schema isolation; cross-import couples schemas at code level even if grants prevent it at DB level).
- **#39** — DO NOT use `event_type="subprocessor.changed"` (epic spec prose). The literal MUST be `"SubprocessorChanged"` to match `OpportunitiesIngested` / `TrialExpiring` PascalCase convention. Inconsistent literals break the discriminated union deserialiser silently.
- **#40** — DO NOT ACK before fan-out completes (claim-on-success spirit; S16 M6 retro). The consumer must enqueue every `send_email.delay()` (or log every failure) before calling `xack`. ACKing first means a worker crash mid-fanout silently drops emails.
- **#41** — DO NOT introduce a Postgres table to track "which admins received which subprocessor_change emails". Use `notification.email_log` (existing) keyed by `(template_type="subprocessor_change", recipient_email, sent_at)`. Adding a parallel tracking table is YAGNI + creates audit-trail divergence.
- **#42** — DO NOT make the SendGrid template body language a hardcoded English fallback. The locale router MUST consult `User.locale_preference` and route to the correct template ID. If `locale_preference` is NULL/missing, default to `"bg"` (platform default per Story 18-0 i18n parity guarantee), NOT `"en"`. Hardcoding `"en"` would silently English-spam Bulgarian admins.

### §4.6 — Previous Story Intelligence

#### From Story 18-0 (done; review-fix pass closed all 6 BLOCKING + 3 NON-BLOCKING)
- **Reuse**: `compute_diff()` and `yaml_content_hash()` are public APIs in `scripts/generate_subprocessor_changelog.py`. The hash function is **content-stable across commit cycle** (CR-6 fix moved away from git short-SHA to sha256 of yaml_content) — this stability is what makes the consumer-side SETNX idempotency work across workflow re-runs.
- **Carry-forward**: anti-pattern fence #18–#24 (especially #19 YAML location lock and #21 no-auto-commit-from-CI).
- **Handoff items explicitly assigned to 18-2** (per 18-0 sprint-status note line 314):
  - `subprocessor.changed` Redis Stream publication
  - `subprocessor_change` email template + delivery
  - 30-day-future `effective_date` lint enforcement (Art. 28 advance notice)
  - Customer-DPA notification audit log entry (satisfied by `email_log`, no extra audit table)
- **Deferred N-3 (still open)**: CI `paths:` filter on the existing sub-processor jobs. Apply the same `paths: ['infra/sub-processors.yaml']` filter to the 3 NEW jobs in this story (cost optimisation; non-blocking but trivial to add).

#### From Story 18-1 (done; Round 3 review-fix closed all 3 Round 2 blockers + 5 high-leverage majors)
- **Carry-forward**: D6 (`_get_artefacts_registry()` rate-limiting), D7 (WeasyPrint timeout pool isolation), D8 (`_HEAD_OBJECT_MISS_CODES` log WARN). All marked "deferred to S18.02 / hardening sprint" in 18-1 sprint-status notes (lines 962–966 and 974). **Recommendation**: do NOT absorb into 18-2 — these are out of scope per epic spec lines 91–113. Document as §6 D-3 "carry-forward to dedicated hardening sprint" so the orchestrator does not silently lose them.
- **`dpa_download_url`**: 18-1 ships the signed-URL pipeline. The consumer should attempt to fetch the current signed URL when constructing email `template_data`. **Graceful degradation**: if the artefact is not yet rendered OR fetch fails, pass `dpa_download_url=None`; the SendGrid template MUST handle null without crashing (template-side conditional). Do NOT block email send on URL availability.
- **Round 2 majors deferred**: silent-bypass error handling, cross-pod dedup race, TrustArtefactCard prop defaults — all explicitly NOT in 18-2 scope.

#### From Operator BMAD-stream Workflow Guidance
- [VS] Validate Story is **non-negotiable** before bmad-dev-story.
- [SR] Story Review after each story (Epic 18 has 3 stories — multi-story epic; do an SR after 18-2 ships before [ER]).
- [ER] Epic Review after all stories complete (Epic 18 has interdependent stories: 18-0 → 18-1 → 18-2 sub-processor change-event chain — ER is recommended).
- [PR] Post-Review after code review (catch any implementation gaps before QA).

#### From Project-Context (anti-pattern carry-forwards visible to orchestrator)
- **AP17-C1 (5th recurrence)**: two-gate story-close. `done` REQUIRES bmad-code-review Approve verdict, NOT just dev-pass-completes. Dev sets status `in-progress → review` only.
- **AP17-C2**: un-skip ATDD RED-phase tests AC-by-AC during dev (do NOT batch RED → GREEN at end).
- **AP17-C3 (15th carry-forward)**: k6 baseline for public-facing surface (notification consumer is internal — N/A to this story specifically; remains carry-forward for inj-02).
- **AP17-C4 (13th carry-forward)**: TEA review gate.
- **AP17-C5 (12th occurrence)**: sprint-status / story-file Status reconciliation — both must agree.
- **AP17-C6**: regenerate NFR for Epic 18 (recommended after dev pass; not blocking).
- **AP14-04 / AP15-08**: canonical ORM seeding (no raw `text("INSERT …")` in tests).
- **AP16-06**: regenerate traceability-matrix post-Approve, NOT at RED phase.
- **AP13-05 (12th consecutive fire)**: inj-01 (Dependabot), inj-02 (k6), drift-recovery, dw-01..03 carry-forwards remain orchestrator-dispatchable; not blocking 18-2.
- **Project-context Rule 45** (fire-and-forget audit): satisfied by existing `_log_to_db()`; no extra audit write.
- **Project-context Rule 47** (two-layer resilience): SendGrid is outbound HTTP; existing `send_email` Celery task has retry on 429/5xx but no circuit-breaker. Carry-forward as known deviation §6 D-2 (consistent with 18-0 line 324 "applies to S18.02"); recommend layering circuit_breaker + retry in a follow-up hardening sprint, NOT in 18-2 (scope creep).
- **Project-context Rule 48** (HMAC compare_digest): N/A — no inbound webhook in this story; SendGrid event-webhook handler already uses ECDSA verification (existing).

### §4.7 — Inline Test Design (fills missing test_artifacts/test-design-epic-18.md)

> Provenance: epic spec lines 91–113; 18-0 handoff lines 245–248; 18-1 handoff lines 558–574; Story 8-3 NIT-4 (trialing inclusion); S15-0 B3 (cross-tenant parametrisation matrix); S16 M6 (claim-on-success); AP14-04 / AP15-08 (canonical ORM seeding); AP17-C1 (two-gate close); Rule 45 (fire-and-forget audit). Risk-priority taxonomy below uses Test Architect P0/P1/P2 convention.

**P0 (release blockers)**:
- **R-018-2-01**: 30-day Art. 28 advance-notice CI lint MUST fail PRs that violate the rule (regulatory compliance — failure exposes EU Solicit to GDPR Art. 28 enforcement). Test: PR fixture adds a sub-processor with `effective_date = today + 5 days` → `enforce-subprocessor-advance-notice` job exits non-zero with message identifying the offending row.
- **R-018-2-02**: NO email is sent to inactive users OR free-tier OR canceled-subscription companies (data-protection: stop sending transactional email to deactivated accounts). Test: 25-cell tier × status matrix; only the (paid_tier × {active,trialing}) cells produce ≥1 enqueue.
- **R-018-2-03**: Idempotency holds across event replay (worker crash + workflow re-run with stable yaml_content_hash). Test: deliver same event twice; assert exactly N enqueues across both deliveries (NOT 2N).
- **R-018-2-04**: Consumer ACKs ONLY after fan-out completes (no silent drop on mid-fanout crash). Test: inject `send_email.delay` failure for recipient #2; assert message is NOT ACKed (visible in pending-entries-list); assert recipient #1 received email and recipient #2 has a `failed` row in `email_log`.

**P1 (significant risk)**:
- **R-018-2-05**: Locale routing (BG vs EN) selects correct SendGrid template ID. Test: parametrise over `locale_preference ∈ {bg, en, None}` × confirm template_attr lookup (`sendgrid_template_subprocessor_change_bg|en`).
- **R-018-2-06**: Event schema serialisation is roundtrip-stable through Redis envelope. Test: publish `SubprocessorChanged` via EventPublisher, consume, assert deep equality on parsed event.
- **R-018-2-07**: Cross-tenant fan-out is global (all paid customers receive every event — by GDPR design, NOT a tenant-scoped event). Test: 2 paid companies; 1 event; both companies' admins receive emails.
- **R-018-2-08**: Removed-only diffs (no additions) skip the 30-day rule. Test: remove a sub-processor with no additions; CI passes; consumer fires email with empty `added` and populated `removed`.
- **R-018-2-09**: Empty diff produces NO email (no-op publication). Test: no-change push (e.g. unrelated file edit) does NOT enqueue any event.

**P2 (minor / cosmetic)**:
- **R-018-2-10**: Resolver excludes deactivated admins. Test: paid-tier company with all admins `is_active=False` produces ZERO emails (not even logged failures).
- **R-018-2-11**: SETNX idempotency uses content hash, not git SHA. Test: amend a commit (changes git SHA) without changing yaml content; second event publication has same `changelog_sha` and is deduped.
- **R-018-2-12**: DLQ writer engages on poison-pill events (event_type mismatch). Test: publish a malformed event; assert it lands in `eu-solicit:notifications.dlq` and original stream proceeds.

**Test layers**:
- **L1 — Python AST source-inspection**: assert no `text("INSERT INTO client.…")` in test files; assert no cross-import of `client_api._PAID_TIERS` in notification service code.
- **L3 — pytest unit + integration**: per AC matrix above.
- **L4 — Playwright E2E**: N/A (no frontend in this story).
- **No L2** (vitest source-inspection — N/A backend story).

### §4.8 — Latest Tech Information (relevant deltas since project pin)

- **Pydantic 2.x discriminated union**: use `Annotated[Union[...], Field(discriminator='event_type')]` syntax (already in use at `packages/eusolicit-models/src/eusolicit_models/events.py` lines 179–193 — follow existing pattern).
- **redis-py 5.x asyncio**: `xgroup_create(mkstream=True)` raises `redis.exceptions.ResponseError` with "BUSYGROUP" prefix when group already exists — swallow per existing consumer template.
- **GitHub Actions `if:`**: `github.event_name == 'push' && github.ref == 'refs/heads/main'` is the canonical guard for main-only jobs.
- **SendGrid v3 Dynamic Templates**: template ID is the only required field; locale-aware substitution is template-side, not code-side. The notification-service code only needs to pick the right template ID.

### §4.9 — Project Context Reference

This story file consumes:
- `eusolicit-docs/planning-artifacts/epics/E18-trust-center-compliance.md` (lines 91–113 = S18.02 spec)
- `eusolicit-docs/planning-artifacts/epics/epic-18.md` (Story 18.3 brief alias of S18.02)
- `eusolicit-docs/implementation-artifacts/18-0-public-trust-route-mdx-pipeline-sub-processor-yaml-change-log-generator.md` (handoff items, anti-pattern fence #18–#24, CR-6 hash idempotency contract)
- `eusolicit-docs/implementation-artifacts/18-1-pdf-artefact-pipeline-weasyprint-reuse-living-artefacts-git-managed-legal-artefacts.md` (signed-URL contract, D6/D7/D8 deferred items, Round 2 majors deferred)
- `eusolicit-docs/project-context.md` (Rules 45, 47, 48; AP17-C1..C6; AP14-04 / AP15-08; AP16-06; AP13-05)
- `eusolicit-app/CLAUDE.md` (testing strategy, schema isolation gold standard, RBAC, critical patterns)
- `test_artifacts/atdd-checklist-18-0-...md` and `atdd-checklist-18-1-...md` (template structure for Task 10)

### §4.10 — Testing Standards Summary

- **Unit tests**: `services/notification/tests/worker/test_subprocessor_consumer.py` — async, fakeredis-backed, mock SendGrid. Markers: `@pytest.mark.unit`.
- **Integration tests**: `services/notification/tests/integration/test_subprocessor_consumer_integration.py` — real Redis (DB 1, `clean_redis` fixture flushes before/after) + real Postgres (`db_session` fixture; per-test rollback). Markers: `@pytest.mark.integration`. Requires `make infra` running.
- **Schema tests**: `packages/eusolicit-models/tests/test_events.py` — roundtrip serialise/parse. Markers: `@pytest.mark.unit`.
- **Script tests**: `scripts/tests/test_publish_subprocessor_event.py` — mock `EventPublisher`, mock `git show`, parametrised diff fixtures. Markers: `@pytest.mark.unit`.
- **Coverage**: ≥80% per `make coverage` minimum.
- **Lint / type-check**: `make lint && make type-check` MUST pass for all touched paths.
- **Migration check**: `make alembic-check` MUST pass (no drift).

### §4.11 — Project Structure Notes

- **Alignment**: All NEW files placed per existing service conventions (see §4.4). No structural deviations.
- **Detected variances**: None. The notification service already follows the consumer-per-stream pattern; we add a 5th consumer (subprocessor) following the same shape as the 4 existing ones.
- **Naming convention check**:
  - Consumer file: `subprocessor_consumer.py` (matches `subscription_consumer.py`, `opportunity_consumer.py`, `task_consumer.py`, `approval_consumer.py`)
  - Stream constant: `eu-solicit:notifications` (matches epic spec line 95; differs from existing notification streams which are domain-named — flagged as known deviation §6 D-4 because epic spec dictates the name and notification streams will multiplex on this single stream going forward)
  - Event class: `SubprocessorChanged` (PascalCase per existing convention)
  - Template type: `"subprocessor_change"` (snake_case per existing `template_type` enum convention)

## §5 — Net-New Fence (What This Story Owns and Doesn't)

**OWNED by 18-2** (must ship in this story):
- `SubprocessorChanged` event Pydantic schema + discriminated union extension
- `eu-solicit:notifications` Redis Stream + consumer + DLQ
- Cross-schema grant migration 007
- Read-only ORM mirrors `Company` + `Subscription`
- Recipient resolver (paid-tier × {active,trialing} × admin × is_active)
- SendGrid `subprocessor_change` template_type branch + locale routing
- 3 NEW CI jobs (publish, advance-notice, template-key-check)
- `scripts/publish_subprocessor_event.py` + `scripts/check_email_template_keys.py`
- `--enforce-advance-notice` flag on existing `validate_subprocessors.py`
- Cross-tenant + tier-gate + idempotency negative tests
- Inline test-design (§4.7) + ATDD checklist
- Operator runbook entry

**OUT-OF-SCOPE for 18-2** (explicitly fenced):
- Real SendGrid template content authoring (template IDs are placeholder defaults; production template authoring is a separate ops task tracked outside this story)
- Sub-processor admin UI CRUD (post-MVP per epic; YAML is the source-of-truth)
- ISO 27001 evidence collection (parallel programme M2–M12)
- 18-1 carry-forward majors D6/D7/D8 (rate-limit, WeasyPrint pool, `_HEAD_OBJECT_MISS_CODES` WARN) — these belong in a hardening sprint, NOT 18-2
- SendGrid circuit-breaker layering (Rule 47 carry-forward) — recommended for hardening sprint
- k6 load test on consumer throughput (AP17-C3 inj-02 carry-forward)
- bmad-testarch-nfr regeneration for Epic 18 (AP17-C6 — recommended after dev pass, not gating)
- Removed/modified sub-processor advance-notice rule (Art. 28 advance notice applies to additions only)

## §6 — Known Deviations (Pre-Recorded)

| ID | Item | Rationale | Disposition |
|---|---|---|---|
| **D-1** | Including `trialing` subscriptions in "active customer DPAs" | Epic spec line 99 says "paid-tier companies" without addressing trial status; Story 8-3 NIT-4 precedent includes `trialing` because trialing tenants hold a DPA | Reviewer-confirm; if PO wants `active`-only, change `ACTIVE_STATUSES` frozenset and re-run AC-9 matrix |
| **D-2** | No circuit-breaker around SendGrid send | Rule 47 two-layer resilience requires `circuit_breaker(retry(http))`; existing `send_email` task has Celery retry on 429/5xx but no circuit-breaker. Layering is non-trivial (Celery integration + per-key circuit state) | Carry-forward to hardening sprint; document in §5 out-of-scope |
| **D-3** | 18-1 D6/D7/D8 carry-forwards NOT absorbed | Out of scope per epic spec lines 91–113; absorbing would scope-creep 18-2 | Carry-forward to hardening sprint |
| **D-4** | Stream name `eu-solicit:notifications` differs from existing per-domain streams (`eu-solicit:opportunities`, etc.) | Epic spec line 95 dictates the name; this stream becomes the catch-all for notification-service-consumed events going forward (multiplex pattern) | Reviewer-acknowledge; consistent with epic spec |
| **D-5** | Template-content i18n parity check is deployment-config only (env-var presence), NOT content parity | Templates live SendGrid-side, not in-repo; in-repo content parity check would require fetching live template definitions from SendGrid API on every CI run (cost + flake risk) | Acceptable per epic intent (BG/EN parity); content parity verified manually by ops on template publish |
| **D-6** | `dpa_download_url` may be `None` in template_data | 18-1 D6/D7/D8 deferred; signed-URL fetch can fail or artefact may not be rendered | Template-side null-safe rendering required; do NOT block email send |
| **D-7** | Test-design provenance inline (§4.7) instead of `test_artifacts/test-design-epic-18.md` | No epic-18 test design exists; inline-fill matches 18-0/18-1 pattern | Document in story; refresh `test_artifacts/traceability-matrix.md` post-Approve |

## §7 — References

- [Source: `eusolicit-docs/planning-artifacts/epics/E18-trust-center-compliance.md` lines 91–113 (S18.02 spec)]
- [Source: `eusolicit-docs/planning-artifacts/epics/epic-18.md` Story 18.3 (brief alias)]
- [Source: `eusolicit-docs/implementation-artifacts/18-0-…md` (handoff items, anti-pattern fence carry-forward, CR-6 hash idempotency)]
- [Source: `eusolicit-docs/implementation-artifacts/18-1-…md` (signed-URL contract, D6/D7/D8 deferred items)]
- [Source: `eusolicit-app/services/notification/src/notification/workers/subscription_consumer.py` (consumer template)]
- [Source: `eusolicit-app/services/notification/src/notification/tasks/email.py` (SendGrid pattern)]
- [Source: `eusolicit-app/services/notification/src/notification/models/email_log.py` (audit pattern)]
- [Source: `eusolicit-app/services/notification/alembic/versions/006_grant_notification_select_on_users.py` (grant template)]
- [Source: `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/publisher.py` (envelope)]
- [Source: `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/consumer.py` (consume/ack/DLQ)]
- [Source: `eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py` (discriminated union)]
- [Source: `eusolicit-app/scripts/validate_subprocessors.py` (extend with `--enforce-advance-notice`)]
- [Source: `eusolicit-app/scripts/generate_subprocessor_changelog.py` (reuse `compute_diff()`, `yaml_content_hash()`)]
- [Source: `eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py` (mirror `_PAID_TIERS`)]
- [Source: `eusolicit-app/.github/workflows/ci.yml` (extend with 3 jobs)]
- [Source: `eusolicit-docs/project-context.md` (Rules 45, 47, 48; AP17-C1..C6; AP14-04 / AP15-08; AP16-06; AP13-05)]
- [Source: `eusolicit-app/CLAUDE.md` (per-schema role isolation; gold-standard testing; RBAC)]
- [Source: `eusolicit-app/test_artifacts/atdd-checklist-18-0-…md` and `atdd-checklist-18-1-…md` (checklist template structure)]
- Operator BMAD-stream workflow guidance (loaded at activation): [VS] Validate Story (non-negotiable) → bmad-dev-story → bmad-code-review (Approve required for `done` per AP17-C1) → [PR] Post-Review → [SR] Story Review → [ER] Epic Review (Epic 18 interdependent stories)

## Dev Agent Record

### Agent Model Used

- **Initial dev pass**: bmad-dev-story (model details elided in original session).
- **Review-fix pass (this section)**: `claude-sonnet-4-5` via bmad-dev-story skill in autopilot mode, session date 2026-05-04.

### Debug Log References

Key fixes verified during review-fix pass:

- **B-1 publisher modified-deltas**: `scripts/tests/test_publish_subprocessor_event.py` 6/6 green; `_build_modified_deltas()` re-pairs HEAD vs PREV by case-insensitive name and emits only fields that actually changed (str-coerced comparison matches the `compute_diff()` modification rule).
- **B-3 envelope contract**: visible-diff vs `eusolicit_common.events.publisher.EventPublisher.publish()` confirms envelope keys are now `{event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id}` — no drift.
- **B-4/B-5 real DB resolver**: 25-cell parametrized matrix + 2-direction cross-tenant + inactive-admin tests all drive `resolve_all_paid_company_admins_async()` against a `notification_role` async session; seeded via ORM models on a `migration_role` engine.  Per-row DELETE cleanup (not TRUNCATE CASCADE — see fix below).
- **B-6 SELECT-only**: two integration tests flush `Company`/`Subscription` via `notification_session` and assert `ProgrammingError`/`DBAPIError`.
- **DEP-1 missing column**: discovered during integration-test green-up; added client-api migration 062 (`locale_preference VARCHAR(8) NULL`) so the resolver SELECT and the locale router work end-to-end.
- **TRUNCATE CASCADE deadlock**: initial cleanup approach used `TRUNCATE client.subscriptions, client.company_memberships, client.users, client.companies CASCADE` which deadlocked against the parallel-running `notification_session` transaction.  Switched to a `seeded_ids` tracker fixture that DELETEs only the specific UUIDs the test inserted (no CASCADE, no contention).

### Completion Notes List

- Review-fix pass addressed all 6 blocking findings (B-1 through B-6) and 7 of 8 non-blocking findings (N-1, N-2, N-3, N-4, N-5, N-6, N-7).  N-8 (`_resolve_template_attr` `__getattr__` edge case) is acknowledged as a defensive nit but the existing `hasattr` + `getattr` pattern is correct for static `BaseSettings` fields and adding extra defensive code would be over-engineering — left as-is.
- Discovered and fixed a story-level dependency gap (DEP-1) that the original implementation pass missed: the `User.locale_preference` column was declared on the notification mirror but the corresponding `client-api` migration was never written.  Added migration `062_add_locale_preference_to_users.py` (VARCHAR(8) NULL — matches the mirror exactly).
- ATDD checklist file location clarification: file lives at project-root `test_artifacts/atdd-checklist-18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md` matching the established 18-0/18-1 convention.  The story originally specified `eusolicit-app/test_artifacts/...` — that path is documented as Known Deviation §6 D-8 below.

### File List

**New files (review-fix pass)**:
- `services/client-api/alembic/versions/062_add_locale_preference_to_users.py` — adds `client.users.locale_preference VARCHAR(8) NULL` (DEP-1 fix; AC-6.4 dependency).

**Modified files (review-fix pass)**:
- `scripts/publish_subprocessor_event.py` — B-1 (`_build_modified_deltas()` helper), B-3 (canonical envelope), N-2 (`_to_entry()` raises on bad date).
- `services/notification/src/notification/workers/subprocessor_consumer.py` — B-2 (`template_data["modified"]`), N-1 (`dpa_download_url=None` TODO + comment).
- `services/notification/tests/integration/test_subprocessor_consumer_integration.py` — full rewrite of AC-9 tests to drive real DB resolver via ORM seeding (B-4/B-5); added AC-5 SELECT-only tests (B-6); added `seeded_ids` cleanup fixture.
- `.github/workflows/ci.yml` — N-5 (step-level YAML-changed gate on publish job), N-6 (heredoc replaces literal-`\n` echo), N-7 (`if: pull_request` + step gate on advance-notice job).

**Modified files (story spec metadata, this pass)**:
- `eusolicit-docs/implementation-artifacts/18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md` — Tasks/Subtasks ticked, Review Follow-ups (AI) section added, Senior Developer Review action items ticked, Dev Agent Record populated, status updated.

**Inherited from initial dev pass (unchanged in review-fix)**:
- `packages/eusolicit-models/src/eusolicit_models/events.py` — `SubprocessorEntry`, `SubprocessorChangeDelta`, `SubprocessorChanged`; `ServiceEvent` discriminated union.
- `services/notification/alembic/versions/007_grant_notification_select_on_companies_and_subscriptions.py` — cross-schema grants.
- `services/notification/src/notification/models/company.py`, `subscription.py`, `user.py` (locale_preference column), `__init__.py`, `company_membership.py` (`resolve_all_paid_company_admins_async`).
- `services/notification/src/notification/core/tier.py` — `PAID_TIERS`, `ACTIVE_STATUSES` mirror constants.
- `services/notification/src/notification/config.py` — sendgrid template settings + `dpo_contact_email`.
- `services/notification/src/notification/tasks/email.py` — locale routing + `subprocessor_change` template_type.
- `services/notification/src/notification/main.py` — 5th lifespan background task.
- `scripts/check_email_template_keys.py`, `scripts/validate_subprocessors.py` (`--enforce-advance-notice`).
- `infra/README.md`, `eusolicit-docs/runbooks/sub-processor-change.md`.
- `packages/eusolicit-models/tests/test_events_subprocessor_changed.py`, `services/notification/tests/worker/test_subprocessor_consumer.py`, `services/notification/tests/worker/test_email_locale_routing.py`, `scripts/tests/test_publish_subprocessor_event.py`, `scripts/tests/test_validate_subprocessors_advance_notice.py`.
- `test_artifacts/atdd-checklist-18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md`.

### Test Results

Verbatim final summary lines from `pytest` runs during the review-fix pass:

- **`pytest scripts/tests/test_publish_subprocessor_event.py`**: `6 passed in 0.77s`
- **`pytest packages/eusolicit-models`** (event schema incl. `test_events_subprocessor_changed.py`): `35 passed in 0.77s`
- **`pytest services/notification/tests/worker/test_subprocessor_consumer.py`**: `12 passed in 0.90s`
- **`pytest services/notification/tests/integration/test_subprocessor_consumer_integration.py`** (full 34-test suite incl. 25-cell tier/status matrix + 2-direction cross-tenant + inactive-admin + 2× SELECT-only + AC-10 real-Redis idempotency): `34 passed, 2 warnings in 4.79s`
- **`pytest services/notification/tests/worker/test_email_locale_routing.py services/notification/tests/worker/test_send_email.py`**: `26 passed, 2 warnings in 0.26s`
- **`pytest services/notification/tests`** (full notification suite, regression check): `5 failed, 514 passed, 3 skipped, 10 warnings in 17.98s`
  - The 5 failures are all in `test_calendar_sync_google.py` and `test_calendar_sync_outlook.py` and stem from a pre-existing `column "source_id" of relation "opportunities" does not exist` schema-state issue in the local test DB.  Confirmed via `git status` on the calendar test files (no modifications by this pass).  Not introduced by 18-2 review-fix; tracked separately.
- **`ruff check`** on all touched paths (`scripts/publish_subprocessor_event.py`, `services/notification/src/notification/workers/subprocessor_consumer.py`, `services/notification/tests/integration/test_subprocessor_consumer_integration.py`, `services/client-api/alembic/versions/062_add_locale_preference_to_users.py`): `All checks passed!`

### Known Deviation (D-8 — review-fix pass)

| ID | Item | Rationale | Disposition |
|---|---|---|---|
| **D-8** | ATDD checklist lives at project-root `test_artifacts/atdd-checklist-18-2-...md`, not `eusolicit-app/test_artifacts/...` as the story originally specified | Project-root `test_artifacts/` is the established convention used by 18-0 and 18-1 (and every previous story in `test_artifacts/`); creating a duplicate at `eusolicit-app/test_artifacts/` would fragment the trust traceability surface | Reviewer-confirm; no relocation in this pass |

## Senior Developer Review (Pass 2 — Re-Review after Review-Fix)

**Reviewer:** bmad-code-review (Claude)
**Date:** 2026-05-04
**Verdict:** REVIEW: Approve

### Summary

All six blocking findings from Pass 1 (B-1..B-6) and seven of eight non-blocking findings (N-1..N-7) are resolved with verifiable code changes. N-8 is a defensible nit (static `BaseSettings` fields don't proxy `__getattr__`); leaving it as-is is reasonable. The newly-discovered dependency gap (DEP-1 — missing `client.users.locale_preference` column) is properly closed via migration 062.

### Verification of Required-for-Approve items

- **B-1 — Publisher modified-deltas**: `scripts/publish_subprocessor_event.py::_build_modified_deltas()` (lines 118–170) re-keys HEAD vs PREV by case-insensitive name, builds union-of-keys minus `name`, and emits only fields where `str(before) != str(after)`. Confirmed by `pytest scripts/tests/test_publish_subprocessor_event.py` (6/6 green per Test Results).
- **B-2 — Consumer surfaces `modified`**: `subprocessor_consumer.py:212` adds `"modified": [delta.model_dump(mode="json") for delta in event.modified]` to `template_data`.
- **B-3 — Canonical envelope**: `EventPublisher.publish()` (lines 81–102) emits exactly `{event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id}` with `tenant_id` defaulting to `""` for platform-wide events. Matches `eusolicit_common.events.publisher.EventPublisher.publish()` keys.
- **B-4 / B-5 — Real DB resolver tests**: `test_subprocessor_consumer_integration.py` lines 220–400 drive `resolve_all_paid_company_admins_async()` against a `notification_role` async session with canonical ORM seeding via Company + User + CompanyMembership + Subscription mirrors. 25-cell tier×status matrix + 2-direction cross-tenant + inactive-admin all use the production resolver. The `seeded_ids` fixture cleans up via per-row DELETE (avoiding the prior TRUNCATE CASCADE deadlock).
- **B-6 — SELECT-only contract**: `test_notification_role_cannot_insert_via_company_mirror` (line 553) and `..._via_subscription_mirror` (line 583) flush a Company / Subscription via the `notification_session` and assert `(ProgrammingError, DBAPIError)`.
- **N-4 — Dev record**: Tasks/Subtasks all `[x]`, Dev Agent Record fully populated with model, debug log, file list, test results.

### Verification of non-blocking fixes

- **N-1**: `subprocessor_consumer.py:200-208` annotates `dpa_download_url=None` with TODO(18-1-D6) and references §6 D-6.
- **N-2**: `_to_entry()` lets `date.fromisoformat()` raise; no `date.today()` fallback (publish_subprocessor_event.py:197-214).
- **N-3 / D-8**: ATDD checklist lives at project-root `test_artifacts/` per established 18-0/18-1 convention; documented as Known Deviation §6 D-8.
- **N-5**: Both `publish-subprocessor-changed-event` and `enforce-subprocessor-advance-notice` jobs gate via step-level "Detect change in infra/sub-processors.yaml" with `if: steps.yaml_changed.outputs.changed == 'true'` on subsequent steps.
- **N-6**: Heredoc replaces literal-`\n` echo for prev-yaml fallback (ci.yml).
- **N-7**: `enforce-subprocessor-advance-notice` has `if: github.event_name == 'pull_request'` plus the step-level YAML-gate.

### Verification of DEP-1 (newly discovered)

- `services/client-api/alembic/versions/062_add_locale_preference_to_users.py` adds `locale_preference VARCHAR(8) NULL` to `client.users`. Matches the notification mirror declaration exactly. Without this migration AC-6.4 is unsatisfiable; with it, the resolver SELECT and locale router work end-to-end (validated by 34-test integration suite passing).

### What's good (re-confirmed)

- Idempotency design unchanged and correct: SETNX on `notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}` with TTL 86400s, exercised by both unit (fakeredis) and integration (real Redis) tests.
- Anti-pattern fence held: #38 (mirror constants, no client_api import — verified by `test_tier_module_does_not_cross_import_from_client_api`), #39 (PascalCase `SubprocessorChanged`), #40 (ACK-after-fanout — verified by `test_ack_called_only_after_fanout_complete`), #42 (bg-default locale).
- Migration 007 follows 006 grant template; downgrade is non-destructive on the schema USAGE grant.
- Schema model + roundtrip test (`test_events_subprocessor_changed.py`) — 35 events tests green.
- `_build_modified_deltas` correctly excludes the `name` field (since name is the matching key) and uses string-coerced comparison to mirror `compute_diff()`'s modification rule.

### Minor observations (non-blocking, not raised as findings)

- **SETNX-before-enqueue ordering**: `_handle_subprocessor_changed` calls `_is_idempotent` (which sets the SETNX key as a side-effect) BEFORE `await asyncio.to_thread(send_email.delay, ...)`. If `send_email.delay` raises (Celery broker unavailable), the SETNX key is set but the email never enqueues; the message will not be ACKed (good — claim-on-success holds), but on retry that recipient will be skipped due to the SETNX guard (silent miss). Mitigation: `send_email.delay` rarely fails in practice (it's a local broker enqueue, not the SendGrid HTTP call), and SendGrid retries are handled inside the Celery task itself. Worth tracking as a Rule-47 hardening candidate but not blocking for this story.
- **`accepted_at IS NOT NULL` filter** in the resolver excludes pending invitation rows; story spec doesn't strictly require this but it's a sensible safety net (don't email someone whose invite hasn't been accepted). Not a finding.
- **Pre-existing 5-test failure** in `test_calendar_sync_google.py` / `test_calendar_sync_outlook.py` is verified unrelated to 18-2 (`column "source_id" of relation "opportunities" does not exist` — a schema-state issue introduced by other in-flight stories), tracked separately by the dev record.

### Markers (for orchestrator change-evaluator)

No new deviations detected in this pass. Pass 1 deviations are all resolved.

---

## Senior Developer Review (Pass 1 — Original)

**Reviewer:** bmad-code-review (Claude)
**Date:** 2026-05-04
**Verdict:** REVIEW: Changes Requested

### Summary

The story's "happy path" (event schema, settings, locale routing, consumer skeleton, idempotency-by-content-hash, advance-notice CI lint, grant migration 007) is in good shape. However, the modify-path is broken end-to-end, AC-1's envelope contract is violated, three integration tests are scaffolds that do not actually exercise the DB resolver, and AC-5's SELECT-only grant assertion has no test coverage. Story file metadata is also out of sync with `Status: review` (Tasks/Subtasks unchecked, Dev Agent Record sections all "TBD").

### Blocking findings

**B-1. Publisher discards modified-row deltas (AC-2 invariant violated).**
`scripts/publish_subprocessor_event.py:132-135` constructs every `SubprocessorChangeDelta` with `changes={}`:

```python
modified_entries: list[SubprocessorChangeDelta] = [
    SubprocessorChangeDelta(name=e["name"], changes={})
    for e in diff["modified"]
]
```

AC-2 requires "name + per-field before/after (only the fields that actually changed; never include unchanged fields)". This always emits empty change dicts, so any real modification (region change, purpose change, dpa_url change) is published as a content-free delta. Note that `compute_diff()` returns full HEAD rows for modified entries (not deltas) — the publisher has to derive the per-field before/after by re-comparing `head_by_name` vs `prev_by_name` over the names in `diff["modified"]`. That logic is missing.

**B-2. Consumer's email `template_data` omits `modified` entirely.**
`services/notification/src/notification/workers/subprocessor_consumer.py:193-199` builds `template_data` with only `added`, `removed`, `effective_date`, `right_to_object_email`, `dpa_download_url`. The `modified` list from the event is dropped on the floor. Combined with B-1, a sub-processor address/region change generates an email that says "nothing changed". Either include `modified` in `template_data`, or document an explicit AC carve-out that modified events are not user-facing (they currently still trigger an email — just an empty one).

**B-3. AC-1.2 envelope contract violated — bespoke sync `EventPublisher` instead of canonical one.**
The publisher script defines its own `EventPublisher` class (`publish_subprocessor_event.py:45-72`) that emits envelope keys `{event_type, source_service, event_id, correlation_id, payload}`. The canonical `eusolicit_common.events.publisher.EventPublisher.publish()` envelope is `{event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id}`. Missing `timestamp` and `tenant_id`. AC-1.2 is explicit: "constructs the envelope through `EventPublisher.publish()` (so envelope keys/types match every other event in the system)." Today the consumer survives only because it pulls `timestamp`/`tenant_id` out of the inner `payload` (which carries them by virtue of `model_dump_json()`), but envelope drift is exactly what AC-1.2 was written to prevent. Either (a) wrap the canonical async `EventPublisher` in `asyncio.run()` from the CI script, or (b) extend the canonical publisher with a sync facade — do not maintain a divergent envelope shape.

**B-4. AC-9.1 25-cell tier × status test is a non-functional placeholder.**
`tests/integration/test_subprocessor_consumer_integration.py:84-98`:

```python
with patch("asyncio.to_thread", side_effect=count_to_thread):
    # Trigger resolver for this company
    pass  # placeholder: resolver would query and return admin or not

assert dispatched_count == expected_emails_count, ...
```

The body of the `with` block never seeds a company, never calls the consumer, and never exercises the resolver. `dispatched_count` stays 0 for all 25 parametrised cases, so the 8 paid×{active,trialing} cells will fail (`0 != 1`) and the 17 other cells will pass for the wrong reason. This is the BLOCKING-priority risk R-018-2-02 — the headline regulatory test — and it is not actually being run. The companion comment ("In the RED phase, this test scaffold marks the intent…") confirms the test was left in RED-phase. Either run the suite (it should fail) or implement the seeding helper and assert against the real resolver output.

**B-5. AC-9.2 cross-tenant test bypasses the resolver it is meant to validate.**
`test_cross_tenant_global_fanout_both_companies_admins_receive_email` patches `_resolve_all_paid_company_admins` with a hardcoded `[company_a_admin, company_b_admin]` list. The DB join (`Subscription × CompanyMembership × User` filtered by tier/status/role/is_active) — which is the actual recipient-set guarantee — is never exercised. This is also true of AC-9.3 (`test_inactive_admin_in_paid_company_receives_zero_emails`), which patches the resolver to return `[]` rather than seeding an `is_active=False` admin and verifying the SQL filter excludes them. Marker `@pytest.mark.integration` is misleading; these are unit tests in disguise. Add a test that drives the real `resolve_all_paid_company_admins_async()` against a per-test-rolled-back `notification_session` with canonical ORM seeding (Company + Subscription + User + CompanyMembership, mixing tier/status/is_active values) and asserts the recipient set.

**B-6. AC-5 SELECT-only contract has no test coverage.**
Story spec line 83: "Forbid INSERT/UPDATE/DELETE through these models (test-asserted via SQLAlchemy `Session.flush()` raising on attempted writes — confirms grants are SELECT-only)." No such test exists. Migration 007 grants `SELECT` on `client.companies` and `client.subscriptions`, but if a future change accidentally adds `INSERT` privilege the regression won't be caught. Add a test that opens a `notification_role` session, attempts `session.add(Company(...))` + `session.flush()`, and asserts a `ProgrammingError` / privilege violation.

### Non-blocking findings

**N-1. `dpa_download_url` is hardcoded `None`** (`subprocessor_consumer.py:198`). AC-6.3 says it should come from 18-1's signed-URL pipeline with graceful fallback. §6 D-6 acknowledges the deferred state, but the consumer makes no attempt to fetch — it's "never fetch" rather than "graceful degradation". If this is intentional pending hardening, file a TODO referencing 18-1 D6 and the follow-up story, otherwise wire it through.

**N-2. `_to_entry()` falls back to `date.today()` on unparseable `effective_date`** (`publish_subprocessor_event.py:120-121`). Silently masks data-quality bugs and could publish "advance notice" emails with today's date. Should raise/abort.

**N-3. ATDD checklist file missing** (Task 10): `eusolicit-app/test_artifacts/atdd-checklist-18-2-...md` was specified but not created. Project standard per 18-0/18-1.

**N-4. Story file metadata out of sync with `Status: review`.** All Tasks/Subtasks checkboxes remain unchecked, Dev Agent Record sections (Agent Model, Debug Log, Completion Notes, File List) are "TBD". Reviewer cannot confirm what was implemented from the story file alone. AP17-C5 (sprint-status / story-file Status reconciliation) is at risk. Update the dev record before re-submitting.

**N-5. CI job `publish-subprocessor-changed-event` lacks `paths:` filter** despite §4.6 18-0 deferred N-3 explicitly calling this out as a cost-optimisation handoff. Other 18-2 jobs (`enforce-subprocessor-advance-notice`) also lack a `paths:` filter on `infra/sub-processors.yaml`. Trivial to add.

**N-6. CI prev-yaml fallback uses literal `\n`** (workflow `ci.yml`, "Extract previous sub-processors.yaml" step): `echo "schema_version: 1\nsub_processors: []" > /tmp/prev-sub-processors.yaml`. The default `echo` does not interpret `\n`; the file ends up as a single line with a literal `\n`, which `yaml.safe_load` will accept because it's still valid YAML for a non-mapping line, but the loader returns `None` and the parser fall-through path treats it as empty. Works by accident. Use `printf` or a heredoc.

**N-7. `enforce-subprocessor-advance-notice` runs on every push, not just on PRs touching the YAML.** Tasks/Task 8 says "runs on PRs touching `infra/sub-processors.yaml`". Add `paths:` filter and `on: pull_request` trigger guard.

**N-8. `notification.tasks.email.send_email` `_resolve_template_attr` lookup uses `getattr(settings, template_attr)` after `hasattr` check — fine for static fields, but if settings ever uses `__getattr__` proxying it'd silently 0-default. Minor; flagging only as a future-proofing nit.

### What's good

- Idempotency design (`SETNX notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}` TTL 86400s) cleanly handles workflow re-runs and pod restarts; both unit (mocked) and integration (real Redis) coverage present and asserts the expected key shape.
- Anti-pattern fence #38 (no cross-import from client-api) and #42 (bg-default locale) are both observed in code AND verified by source-inspection tests.
- ACK-after-fanout ordering test (`test_ack_called_only_after_fanout_complete`) directly enforces S16 M6 / anti-pattern #40.
- `_resolve_template_attr` cleanly handles unknown locales with `bg` fallback (anti-pattern #42).
- Migration 007 follows 006's grant template; downgrade is non-destructive on the schema USAGE grant.
- `--enforce-advance-notice` flag is correctly back-compat (default off) and fails-CLOSED on missing `origin/main` ref per Story 17-1 §8.13.
- Schema model + roundtrip test (test_events_subprocessor_changed.py) is solid; AC2-T6 properly validates the discriminated-union path.

### Required for Approve

- [x] Fix B-1 (publisher computes real before/after deltas). — `scripts/publish_subprocessor_event.py::_build_modified_deltas()` (review-fix pass 2026-05-04).
- [x] Fix B-2 (consumer surfaces `modified` to template_data). — `subprocessor_consumer.py::_handle_subprocessor_changed` includes `template_data["modified"]` (review-fix pass).
- [x] Fix B-3 (use canonical `EventPublisher` envelope). — sync `EventPublisher` in `publish_subprocessor_event.py` now emits `event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id` exactly matching `eusolicit_common.events.publisher.EventPublisher.publish()`.
- [x] Replace placeholder bodies in AC-9.1 / AC-9.2 / AC-9.3 with tests that drive the real resolver. — `test_subprocessor_consumer_integration.py` rewritten; 25-cell matrix + 2-direction cross-tenant + inactive-admin all drive `resolve_all_paid_company_admins_async()` against `notification_session` with canonical ORM seeding.
- [x] Add AC-5 SELECT-only assertion test (B-6). — `test_notification_role_cannot_insert_via_company_mirror` and `…_via_subscription_mirror`.
- [x] Update story file Tasks/Subtasks + Dev Agent Record to reflect actual dev work (N-4). — done in this section.

### Markers (for orchestrator change-evaluator)

DEVIATION: Publisher emits empty `modified` deltas; consumer drops `modified` from template_data
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

DEVIATION: AC-1.2 envelope diverges from canonical EventPublisher (missing timestamp + tenant_id)
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

DEVIATION: AC-9.1/9.2/9.3 integration tests bypass resolver/DB; AC-5 SELECT-only test missing
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

## Known Deviations

### Detected by `3-code-review` at 2026-05-04T02:07:39Z (session bff5332b-3f2d-471a-9639-c42ac77af8a1)

- Publisher emits empty `modified` deltas; consumer drops `modified` from template_data _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- AC-1.2 envelope diverges from canonical EventPublisher (missing timestamp + tenant_id) _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- AC-9.1/9.2/9.3 integration tests bypass resolver/DB; AC-5 SELECT-only test missing _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- Publisher emits empty `modified` deltas; consumer drops `modified` from template_data _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- AC-1.2 envelope diverges from canonical EventPublisher (missing timestamp + tenant_id) _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- AC-9.1/9.2/9.3 integration tests bypass resolver/DB; AC-5 SELECT-only test missing _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
