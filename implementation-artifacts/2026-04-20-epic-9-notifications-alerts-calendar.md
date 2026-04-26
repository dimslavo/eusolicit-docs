---
epic: 9
title: 'Notifications, Alerts & Calendar'
date: '2026-04-20'
stories: 14
points: 55
sprints: '9–10'
milestone: Beta
generated_by: 'bmad-retrospective'
gate_decisions:
  trace_gate: 'CONCERNS'
  nfr_gate: 'PASS (with CONCERNS)'
  adr_score: '19/29'
  high_risks_mitigated: 4
  blockers: 0
---

# Retrospective — Epic 9: Notifications, Alerts & Calendar

**Date:** 2026-04-20
**Epic:** E09 — Notifications, Alerts & Calendar
**Stories Delivered:** 14 of 14 (55 story points, Sprints 9–10, Beta Milestone)
**Total Automated Tests:** ~810 (unit + integration + API + ATDD)
**All P0 Risks:** Mitigated ✅
**Release Blockers:** 0

---

## Executive Summary

Epic 9 delivered EU Solicit's full outbound communications subsystem: a Celery-based event-driven Notification Service consuming Redis Streams, SendGrid email delivery with 7 transactional templates, Google Calendar and Microsoft Outlook OAuth2 sync with Fernet-encrypted token storage, iCal feed generation, task/approval lifecycle notifications, Stripe usage reporting, and scheduled report generation — backed by 14 fully shipped stories.

**Highlights:**
- All four test-design HIGH risks (E09-R-001 Fernet token exposure, E09-R-002 consumer duplicate dispatch, E09-R-003 SendGrid webhook spoof, E09-R-004 Stripe counter atomicity) are **fully mitigated with P0 tests** — the strongest security posture of any epic to date alongside E07.
- ~810+ automated tests across 14 stories; 150 ACs inventoried, 141 covered.
- ADR quality score 19/29 — on par with E07 (best prior epic).

**Concerns requiring Sprint 11 action:**
1. Story 9.3 story-file status inconsistency (story file = `ready-for-dev`, sprint-status = `done`, ATDD checklist shows 12 tests complete)
2. Epic status `epic-9: in-progress` not closed — 4th recurrence of this anti-pattern
3. 60-second immediate-alert SLA unmeasured on staging
4. Frontend component tests for S09.12 / S09.13 deferred to TEA with no named activation story
5. No Prometheus metrics or Grafana dashboard for the Notification Service

---

## What Went Well

### 1. Security Engineering Discipline
All four HIGH-priority risks were addressed before story completion:
- **E09-R-001**: Fernet encryption of OAuth refresh tokens — `token_crypto.py` canonical module; P0 test asserting stored bytes ≠ plaintext.
- **E09-R-002**: `INSERT ... ON CONFLICT DO NOTHING` on `(user_id, opportunity_id, digest_type)` prevents duplicate dispatch under at-least-once delivery.
- **E09-R-003**: SendGrid Event Webhook ECDSA signature verified on raw request bytes; fail-closed on missing public key.
- **E09-R-004**: Strict `read → Stripe-with-idempotency-key → GETDEL` ordering; counter survives crash-before-clear.

### 2. Idempotency Patterns Consistently Applied
Redis `SETNX` idempotency guard appeared across S09.10 (task consumers), S09.11 (subscription consumer), and per-admin fanout isolation. `INSERT ... ON CONFLICT DO NOTHING` complemented it at the DB layer. The pattern is now battle-tested across alert matching, task notifications, and subscription events.

### 3. Async-Safe Consumer Dispatch Pattern Discovered
`asyncio.to_thread(send_email.delay, ...)` wrapper emerged as the correct pattern for calling synchronous Celery `.delay()` from within async event-loop consumer contexts. This was not in the original epic spec and was discovered during implementation — caught and tested (S09.04, S09.10, S09.11).

### 4. Retry Exception Propagation Bug Caught Early (S09.09)
Story 9.9 identified that broad `except Exception:` clauses can swallow `celery.exceptions.Retry`, preventing Celery from re-scheduling the task. The fix and tests WU-11/WU-12 landed before story review. This is a subtle but high-impact pattern for all Celery consumer stories.

### 5. Cross-Service Schema Reuse Without Coupling
The `TrackedOpportunity` ORM mirror pattern (S09.08, AC12) was used for cross-schema JOINs instead of inline Core Table declarations, maintaining schema isolation while allowing efficient queries. This extends the Rule 1 / schema-isolation convention cleanly to read-only cross-schema access.

### 6. ATDD Checklist Quality
The 4 ATDD checklists generated (9.3, 9.4, 9.5, 9.6) show RED → GREEN discipline with 100% green phase completion. S09.3's checklist (12 tests, P0–P2 coverage) documents the tests that the traceability matrix failed to surface.

---

## What Went Wrong

### 1. Story 9.3 State Inconsistency (Governance Gap)

The story file `9-3-alert-preferences-crud-api-client-api.md` still has status `ready-for-dev` with all task checkboxes unchecked. `sprint-status.yaml` shows `9-3: done`. The ATDD checklist (step-06-verification, 2026-04-19) shows 12 tests implemented and passing.

**This is the same RC2 anti-pattern from the Story 5-5 retrospective** (dev phase completes but story file post-conditions are not updated). Despite playbooks delivered in April 2026 (dev-phase-acceptance.md), the anti-pattern recurred on Story 9.3. The traceability matrix, generated from story-file status, correctly recorded 0 tests — but was wrong about the actual state.

**Impact:** TRACE_GATE was `CONCERNS` (not `PASS`); NFR HIGH priority item; governance trust in story-file state is degraded.

### 2. Epic Status Not Closed (4th Recurrence)

`epic-9: in-progress` in sprint-status.yaml with all 14 stories `done`. This is the **4th consecutive epic** where the epic-level status was not updated to `done` at story completion. This was explicitly documented as an anti-pattern in the E12 retrospective and a CI enforcement action was planned but not implemented.

### 3. 60-Second Immediate-Alert SLA Never Measured

The P3 E2E "Full alert flow ≤ 60s" scenario in the test design is tagged `@skip-ci` and was never executed on staging. The in-memory preference cache refreshes every 60 seconds, meaning a newly-enabled preference may miss the SLA on the first event. The SLA is a headline product commitment for Beta — shipping without a measured baseline is a governance risk.

### 4. Frontend Component Test Deferral Without Activation Story

Stories 9.12 (224 ATDD tests) and 9.13 (145 ATDD tests) deferred component-level Vitest + Testing Library tests to TEA phase. Unlike the E12 retrospective's anti-pattern for RED-phase specs, no activation story was named at deferral time. Regression detection on frontend relies entirely on manual ATDD walkthrough.

### 5. Prometheus Metrics Absent for New Service

The Notification Service shipped with ~810 tests and strong functional coverage, but no Prometheus metrics and no Grafana dashboard. Alert-on-SLA-breach monitoring (e.g., `immediate_alert_dispatch_duration_seconds p95 > 60s`) cannot be implemented without instrumentation.

### 6. ATDD Artifact Location Inconsistency

Only 4 of 14 stories have ATDD checklist files in `test_artifacts/` (and only 4 in the root-project-level `test_artifacts/`). The remaining 10 stories have pytest files under `eusolicit-app/services/notification/tests/`. There is no consistent location convention across backend vs. frontend stories, making it difficult to enumerate ATDD artifacts programmatically.

### 7. Spec Ambiguity — Report Schedule Time

Story 9.14's epic spec said "08:00 UTC" for scheduled report generation; the implementation asserts "06:00 UTC". The deviation was caught and documented, but reveals that schedule constants in the epic spec were not reviewed against the Beat schedule table in S09.01 before story creation. A pre-story spec alignment check would have prevented the deviation.

---

## Structured Findings

### [PATTERN] Idempotency Guard: DB Constraint + Redis SETNX Dual Layer
Applying both `INSERT ... ON CONFLICT DO NOTHING` at the DB level and `SETNX` per-recipient at the Redis level prevents duplicate email dispatch under at-least-once Celery delivery. Use this dual-layer pattern in all future consumer stories.
- **First instance:** S09.04 (alert matching), S09.10 (task consumers), S09.11 (subscription consumer)

### [PATTERN] Fernet Encryption for Third-Party OAuth Tokens
Store encrypted ciphertext, assert `stored_bytes != plaintext`, run roundtrip test, scrub from structlog. The `notification/core/token_crypto.py` canonical module is the reference implementation. Reuse in any future OAuth2 integration.

### [PATTERN] `asyncio.to_thread` Wrapper for Celery Dispatch in Async Consumers
When a Celery `.delay()` call is issued from within an `async` Redis Stream consumer, wrap it in `await asyncio.to_thread(task.delay, ...)`. Without this, the synchronous Celery dispatch can block the async event loop. This pattern is mandatory for all future async consumer stories.
- **Test evidence:** `test_asyncio_to_thread_wraps_send_email_delay`, `test_dispatch_uses_asyncio_to_thread`

### [PATTERN] 404 (Not 403) for Cross-User Resource Access
Return 404 (not 403) when a user attempts to access another user's preference, calendar connection, or similar scoped resource. This prevents existence leakage. Test both the 200 (own resource) and 404 (other user's resource) paths in every CRUD story.

### [PATTERN] ECDSA Webhook Signature Validation on Raw Request Body
Validate the raw `request.body()` bytes before JSON parsing for all inbound webhooks. Use the provider's verification library (SendGrid: `EventWebhook.verify_signature`). Fail-closed on missing key. Three tests: valid → 200, invalid → 401, empty-key → 401.

### [PATTERN] Stripe Usage Counter: read → API(idempotency-key) → GETDEL
The correct atomic ordering for Stripe usage reporting: (1) read counter, (2) call Stripe with `idempotency_key={item_id}:{period}`, (3) `GETDEL` only on confirmed success. Crash-before-DEL leaves counter intact for the next run. P0 tests must verify both paths.

### [PATTERN] Narrow `except` Clauses in Celery Tasks — Never Swallow `celery.exceptions.Retry`
`celery.exceptions.Retry` must propagate past all `except Exception:` handlers in Celery tasks. Broad catch clauses swallow the retry signal and convert retryable errors into permanent failures. Always use narrow exception types or explicitly re-raise `Retry`. Test this with `test_retry_exception_propagates`.
- **Discovery:** S09.09 Microsoft Outlook sync, E09-R-009

### [ACTION] Close epic-9 status in sprint-status.yaml
`epic-9: in-progress` → `done`. All 14 stories confirmed `done`.
- `IMPACT: config_tuning`
- `SEVERITY: high`

### [ACTION] Reconcile Story 9.3 story-file status and task checkboxes
Update `eusolicit-docs/implementation-artifacts/9-3-alert-preferences-crud-api-client-api.md`: flip status from `ready-for-dev` → `done`, check all task boxes, add Dev Agent Record referencing the ATDD checklist. Update traceability matrix row with 12 test IDs.
- `IMPACT: standards_update`
- `SEVERITY: high`

### [ACTION] Execute staging E2E for 60-second immediate-alert SLA
Run the P3 "Full alert flow ≤ 60s" scenario on staging. Publish `load-test-results-epic-9.md` with p50/p95 dispatch latency over 20 trials. Document the 60-second cache-refresh corner case as accepted product behaviour or mitigate with pub/sub invalidation.
- `IMPACT: story_injection`
- `SEVERITY: high`

### [ACTION] Add component tests for S09.12 (Alert Preferences) and S09.13 (Calendar Connection)
Create Vitest + Testing Library component tests covering: CPV multi-select, 5-preference limit enforcement, optimistic toggle with rollback (9.12); OAuth button state transitions, copy-to-clipboard, 30s polling behavior (9.13). Name the activation story in Sprint 11.
- `IMPACT: story_injection`
- `SEVERITY: high`

### [ACTION] Add Prometheus metrics and Grafana dashboard for Notification Service
Instrument: `immediate_alert_dispatch_duration_seconds`, `sendgrid_send_total{template,status}`, `calendar_sync_duration_seconds{provider}`, `stripe_usage_report_errors_total`. Wire 5 alerting hooks (SLA breach, deliverability regression, consumer lag, third-party API anomaly, billing integrity).
- `IMPACT: story_injection`
- `SEVERITY: high`

### [ACTION] Add mandatory story-file post-condition check to dev-phase playbook
Despite `dev-phase-acceptance.md` playbook delivery in April 2026, story 9.3's file was left at `ready-for-dev`. Escalate enforcement: add a structured `## Dev Agent Record` check to the phase-completion gate. The story file status field must be updated before `HALT`. If `Layer B (2b-dev-story-verify)` from the Story 5-5 retrospective is not yet shipped, prioritise it.
- `IMPACT: prompt_adjustment`
- `SEVERITY: critical`

### [ACTION] Standardise ATDD checklist artifact location across all story types
All ATDD checklists (backend and frontend) must be written to `test_artifacts/atdd-checklist-{story_id}.md` in the project root's `test_artifacts/` directory. The current split (some in `test_artifacts/`, most as pytest files) makes programmatic enumeration impossible. Update the ATDD checklist template and the traceability phase to read from this single location.
- `IMPACT: standards_update`
- `SEVERITY: medium`

### [ACTION] Add calendar sync partial-failure transactional integration test (E09-R-007)
Inject mid-batch Google Calendar API failure and assert `client.calendar_events` is consistent with pre-sync state. This MEDIUM risk was documented but no test was enumerated in the matrix.
- `IMPACT: story_injection`
- `SEVERITY: medium`

### [ACTION] Cross-service contract test: E06 → E09 `subscription.trial_expiring`
Verify that E06 publishes the event envelope fields that E09's `SubscriptionConsumer` expects. This is required before the Beta milestone to confirm the E06↔E09 integration path.
- `IMPACT: story_injection`
- `SEVERITY: medium`

### [ACTION] Enforce CI rule: epic-N status must be `done` when all stories are `done`
This is the 4th consecutive epic where the epic status was left open. Add a CI lint step (`quality-gates.yml`) that reads `sprint-status.yaml` and fails if any epic has all stories in terminal state (`done`/`completed`/`approved`) but epic status is not `done`.
- `IMPACT: config_tuning`
- `SEVERITY: medium`

### [ANTI-PATTERN] Story file left in pre-dev state after implementation completes (RC2 Recurrence)
Story 9.3's story file retains `Status: ready-for-dev` with unchecked task boxes despite the implementation and tests being complete per the ATDD checklist. This is the same Root Cause 2 identified in the Story 5-5 retrospective. The `dev-phase-acceptance.md` playbook was not sufficient to prevent recurrence. Stronger enforcement (automated or gated) is required.

### [ANTI-PATTERN] Deferring frontend component tests without naming an activation story
Stories 9.12 and 9.13 deferred Vitest component tests to "TEA phase" without specifying the sprint or story that would activate them. Per the E12 retrospective rule: *"RED-phase ATDD specs require an owner and an activation story at creation time."* Apply the same discipline to any deferred test category.

### [ANTI-PATTERN] Omitting Prometheus metrics when shipping a new long-running service
The Notification Service deployed ~810 tests worth of functional coverage with zero observability instrumentation. A service that runs Celery workers handling SLA-bounded operations (60s dispatch, 15m calendar sync) must have metrics from Sprint 1, not Sprint N+1.

### [ANTI-PATTERN] Epic spec schedule constants not cross-checked with Beat schedule table
Story 9.14 implemented "06:00 UTC" while the epic spec said "08:00 UTC". The deviation was harmless but reveals that story creators do not reconcile schedule constants against the S09.01 Beat schedule dict before writing acceptance criteria. Add a "Beat schedule alignment" check to the story creation preflight for any story that mentions UTC time constants.

---

## TEA Score Summary (from NFR Report)

| Category | Score | Notes |
|---|---|---|
| Testability & Automation | PASS (4/4) | testcontainers + respx + freezegun complete stack |
| Test Data Strategy | PASS (3/3) | Full factory set; rollback-only DB isolation |
| Scalability & Availability | CONCERNS (2/4) | Cache staleness, Graph 429 throttling undocumented |
| Disaster Recovery | CONCERNS (0/3) | Inherited platform posture; no E09 drill |
| Security | PASS (4/4) | Strongest security posture in the project alongside E07 |
| Monitorability | CONCERNS (2/4) | No Prometheus metrics; structlog present |
| QoS & QoE | CONCERNS (1/4) | 60s SLA unmeasured; functional correctness proven |
| Deployability | PASS (3/3) | Additive migrations; downgrade path present |
| **Overall ADR** | **19/29 (66%)** | **PASS — on par with E07** |

---

## Carry-Forward to Epic 10

The following items from Epic 9 must be surfaced in Epic 10's NFR assessment as automatic CONCERNS if not resolved:

| Item | Priority | Owner | Target |
|---|---|---|---|
| Story 9.3 story-file reconciliation | HIGH | Dev Lead | Sprint 11 Day 1 |
| Staging 60s SLA E2E baseline | HIGH | QA | Before Beta |
| Frontend component tests S09.12 / S09.13 | HIGH | Frontend Lead | Sprint 11 |
| Prometheus metrics + Grafana dashboard | HIGH | Backend Lead | Sprint 11 |
| Calendar sync partial-failure test (E09-R-007) | MEDIUM | QA | Sprint 11 |
| Cross-service contract test E06↔E09 | MEDIUM | QA | Sprint 11 |
| Epic status CI enforcement | MEDIUM | DevOps | Sprint 11 |
| iCal access log token redaction | LOW | Backend Lead | Pre-Beta |
| CALENDAR_ENCRYPTION_KEY rotation playbook | LOW | DevOps | Pre-Beta |
| Pub/sub cache invalidation for preference writes | LOW | Backend Lead | Backlog |

---

## Risk Mitigation Ledger

| Risk ID | Description | Score | Status |
|---|---|---|---|
| E09-R-001 | Fernet calendar token key exposure | 6 | ✅ MITIGATED |
| E09-R-002 | Consumer-group duplicate dispatch | 6 | ✅ MITIGATED |
| E09-R-003 | SendGrid webhook signature bypass | 6 | ✅ MITIGATED |
| E09-R-004 | Stripe usage counter atomicity | 6 | ✅ MITIGATED |
| E09-R-009 | Microsoft Retry exception propagation | 0 | ✅ FIXED (S09.09) |
| E09-R-007 | Calendar sync partial-failure | 4 | ⚠️ OPEN — test missing |
| E09-R-010 | E06↔E09 trial_expiring contract | 4 | ⚠️ OPEN — no contract test |
| E09-R-011 | Digest window cursor error | 2 | ⚠️ OPEN — 100-user timing test not run |
| E09-R-012 | iCal token in access logs | 2 | ⚠️ OPEN — log redaction unverified |

---

## Quick Wins Applied in Sprint 11

- [ ] Update `sprint-status.yaml`: `epic-9: done`
- [ ] Fix story 9.3 file status + task checkboxes + Dev Agent Record
- [ ] Add iCal log redaction rule (`/api/v1/calendar/ical/` path scrubbing)
- [ ] Add structlog assertion: no token substring in OAuth callback logs
- [ ] Add product release note: "First alert to a newly-enabled preference may be delayed up to 60s (cache refresh)"

---

## Velocity & Capacity

| Metric | Value |
|---|---|
| Stories | 14 |
| Points | 55 |
| Sprints | 2 (9–10) |
| Points/Sprint | 27.5 |
| Tests/Point | ~14.7 |
| Stories with Full Coverage | 12 / 14 |
| P0 Risks Mitigated | 4 / 4 |
| New Service Shipped | Notification Service (Celery + Beat) |

---

*Retrospective generated by `bmad-retrospective` skill — Epic 9, eusolicit project, 2026-04-20.*
