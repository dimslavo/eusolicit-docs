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
scope: cluster-D-cross-service-notification
batch: P4-c
storyKeys:
  - epic-09-redis-streams-event-bus
  - 9-11-trial-expiry-stripe-usage-sync
  - 18-2-subprocessor-changelog-distribution
  - 20-0-third-party-nps-sdk-and-routing
detectedStack: backend
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/publisher.py
  - eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py
  - eusolicit-app/services/client-api/src/client_api/services/webhook_service.py
  - eusolicit-app/services/client-api/src/client_api/api/v1/nps_feedback.py
  - eusolicit-app/services/notification/src/notification/workers/subscription_consumer.py
  - eusolicit-app/services/notification/src/notification/workers/subprocessor_consumer.py
  - eusolicit-app/tests/cross_service/conftest.py
---

# Automation Summary: P4-c — Cross-service notification flow contract (cluster D, sub-run 3)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P4-c (cluster D, gap-fill API coverage, sub-run 3 — last of cluster D)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

Per the rollout-plan inventory:

> "Cross-service notification flow — `services/notification/tests/...`
> exists; `tests/cross_service/...` may have partial coverage
> → End-to-end Redis Streams flow: client-api emits event →
> notification consumer → outbox row + delivery side effect (mocked
> SendGrid)"

Before this run:

- `tests/cross_service/` had **one** test file
  (`test_generate_sse_headers.py`) and it was **entirely
  `@pytest.mark.skip()`-decorated** RED-phase placeholder.
- `services/notification/tests/integration/` had strong consumer-side
  coverage (4 `test_*_consumer_integration.py` files) — but each test
  hand-builds a stream envelope. There was **no test that verified the
  publisher's envelope shape matches what the consumers' parsers
  accept**, nor any source-level guard that the producer and consumer
  stream-name constants are aligned.

P4-c ships a contract-level cross-service spec that closes the
producer↔consumer wire-contract gap: the canonical 7-key envelope,
two real event types round-tripping through the discriminated
`ServiceEvent` union, correlation_id propagation for distributed
tracing, and source-level stream-name alignment.

---

## Files Created

### `tests/cross_service/test_notification_event_flow.py` (NEW)

5 tests, all marked `@pytest.mark.cross_service`:

| # | Scope | Description |
|---|---|---|
| 1 | Envelope shape | `EventPublisher.publish` writes exactly 7 keys: `event_id`, `event_type`, `payload`, `timestamp`, `correlation_id`, `source_service`, `tenant_id`. `payload` is JSON-encoded; `event_id` + `correlation_id` are UUID4. |
| 2 | `TrialExpiring` round-trip | Publish via `EventPublisher`, read with `xrange`, merge envelope-level fields with `json.loads(payload)`, validate via `TypeAdapter[ServiceEvent]`. Asserts a `TrialExpiring` instance with all fields preserved. Mirrors what `subscription_consumer.process_event` does in production. |
| 3 | `BidOutcomeRecorded` round-trip | Same flow as test 2 but with the bid-outcomes event — pins down `status` literal, `contract_value_eur` float|None, `recorded_by` required. |
| 4 | `correlation_id` propagation | A caller-supplied `correlation_id` MUST survive `publish()` verbatim (distributed-tracing NFR). Catches a regression where the publisher silently regenerates the id. |
| 5 | Stream-name alignment (source inspection) | `inspect.getsource` on the producer modules verifies they reference the canonical stream key, AND the consumer modules' `_STREAM` constants match. Specifically: `webhook_service.py` ↔ `subscription_consumer._STREAM == "eu-solicit:subscriptions"`, `nps_feedback.py` ↔ `subprocessor_consumer._STREAM == "eu-solicit:notifications"`. |

### Test infrastructure

- **In-process Redis** via `fakeredis.aioredis.FakeRedis` (already a
  declared dep in both `services/client-api/pyproject.toml:52` and
  `services/notification/pyproject.toml:44`). No testcontainers /
  Docker required, so the test runs cleanly under
  `make test-integration` and `make test` alike.
- The single `fake_redis` async fixture is local to this file —
  the existing cross-service `conftest.py` provides
  `all_service_clients` for true cross-service-HTTP tests, but our
  contract tests don't need live HTTP.
- `pytestmark = pytest.mark.cross_service` is set at module scope
  (matching the pattern in `test_generate_sse_headers.py`); the
  conftest's docstring promise of "auto-marking" is informational
  only — pytestmark must be declared per-module.

---

## Validation

| Check | Result |
|---|---|
| `ruff check` filtered for new file | ✅ All checks passed |
| `pytest --collect-only` (Python 3.13.5) | ✅ **5 tests collected** in 1 module |
| Active vs skip split | 5 active / 0 skip / 0 xfail |
| Local runtime | ❌ Host venv lacks `eusolicit_common` / `notification` deps (per saved memory `project_test_execution_environment.md`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| 7-key envelope (`event_id`, `event_type`, `payload`, `timestamp`, `correlation_id`, `source_service`, `tenant_id`) | `eusolicit-common/events/publisher.py:50-58` | ✅ |
| `payload` is `json.dumps(payload)` | `publisher.py:53` | ✅ |
| `event_id` + `correlation_id` are UUID4 | `publisher.py:51, 55` | ✅ |
| `TrialExpiring` discriminated-union schema | `eusolicit-models/events.py:129-135` | ✅ |
| `BidOutcomeRecorded` discriminated-union schema | `eusolicit-models/events.py:138-148` | ✅ |
| `ServiceEvent` discriminated union | `eusolicit-models/events.py:252` | ✅ |
| Producer publishes `TrialExpiring` to `eu-solicit:subscriptions` | `client-api/services/webhook_service.py:489` | ✅ |
| Consumer reads `eu-solicit:subscriptions` | `notification/workers/subscription_consumer.py:28` (`_STREAM`) | ✅ |
| Producer publishes NPS detractor alerts to `eu-solicit:notifications` | `client-api/api/v1/nps_feedback.py:200-204` | ✅ |
| Consumer reads `eu-solicit:notifications` | `notification/workers/subprocessor_consumer.py:54` (`_STREAM`) | ✅ |
| `correlation_id` parameter passed verbatim (no regen) | `publisher.py:55` (`correlation_id or str(uuid.uuid4())`) | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make infra                              # postgres + redis (NOT used by these tests)
pytest tests/cross_service/test_notification_event_flow.py -v -m cross_service
```

The 5 tests are self-contained: no Postgres, no Docker testcontainers,
no live HTTP services. They run in-process against fakeredis. CI can
include this file in the standard `make test` pass without spinning
up extra infra — making P4-c the *cheapest* of the three cluster-D
sub-runs to keep green.

### Risks / open questions for the CI pass

1. **`fakeredis.aioredis` version skew** — fakeredis 2.x supports the
   async client but the API surface evolves; if a future bump removes
   `xrange()` or changes its return type, tests #1–#4 break. Pinned at
   `>=2.21` per the existing pyproject deps.

2. **Source-inspection guard breakage** — test #5 grep-asserts on the
   exact string literal `"eu-solicit:subscriptions"` and
   `"eu-solicit:notifications"` in producer source files. If a future
   refactor extracts the constant to a shared module
   (e.g. `eusolicit_common.streams.NOTIFICATIONS_STREAM`), the
   `inspect.getsource` substring check fails. The fix is to switch to
   `getattr(constants, 'NOTIFICATIONS_STREAM') ==
   subprocessor_consumer._STREAM` once the constant is centralised.

3. **Discriminated-union envelope merge** — tests #2 and #3 simulate
   the consumer's "merge envelope + payload" parse step manually. If a
   future consumer refactor changes how the merge is done (e.g. nests
   the payload under a `body` key), the test stays green but the
   production parser breaks. Mitigation: a follow-up "consumer parses
   what publisher emits" integration test that imports the actual
   consumer's parsing function and runs it on a real fakeredis row.

4. **`source_service` / `tenant_id` always-string contract** — the
   publisher coerces `tenant_id=None` to empty string `""`
   (`publisher.py:57`) because Redis Streams field values must be
   strings. The consumer side must therefore round-trip `""` to `None`
   before validating with the Pydantic model — which is what test #2
   asserts via `fields["tenant_id"] or None`. If a consumer ever
   forgets that empty-string-to-None coercion, tenant-scoped routing
   breaks silently.

5. **`pytestmark = pytest.mark.cross_service` is the runner gate** —
   if the project's CI matrix segregates cross-service tests behind a
   `pytest -m cross_service` flag (or excludes them via
   `not cross_service`), this file follows the same gate as
   `test_generate_sse_headers.py`.

6. **Deferred follow-ups for this surface**:
   - **End-to-end with the actual consumer's `process_event` method**:
     publish → fakeredis → invoke `SubscriptionConsumer.process_event`
     directly → assert mocked `send_email.delay` was called with the
     expected `template_type` + `template_data`. This is one step
     beyond the contract tests; the existing
     `test_subscription_consumer_integration.py` already does this
     for the subscription consumer specifically — the gap is only
     for the *coupled* producer + consumer scenario.
   - **DLQ / retry contract**: assert the publisher envelope survives
     a round-trip through the consumer's retry path
     (`EventConsumer.retry_pending` + DLQ xadd in `consumer.py:226`).
   - **Outbox row materialisation**: when a future story adds a true
     transactional outbox table, lock down the contract that
     consumer-side outbox-row writes are atomic with the email
     dispatch.

---

## Coverage Delta

- Before P4-c: **zero** active cross-service tests on this repo —
  `tests/cross_service/test_generate_sse_headers.py` is RED-phase
  module-skip.
- After P4-c: **5 active** cross-service contract tests covering the
  publisher↔consumer envelope shape, two event-type round-trips,
  correlation_id propagation, and source-level stream-name alignment.

---

## Cluster D (P4) — final running totals

| Sub-run | Spec file | Active | Skip / xfail |
|---|---|---:|---:|
| P4-a | `tests/integration/test_nps_feedback_route_contract.py` | 8 | 0 |
| P4-b | `services/client-api/tests/api/test_billing_route_contract.py` | 9 | 0 |
| P4-c | `tests/cross_service/test_notification_event_flow.py` | 5 | 0 |
| **Total (cluster D)** | **3 spec files** | **22** | **0** |

P4 is now complete.

---

## Full-suite rollout — final aggregate

| Phase | Sub-runs | Spec files | Active tests |
|---|---:|---:|---:|
| P1 (Story 11-7 RED→GREEN) | 1 | 1 | (per P1 summary) |
| P2 (cluster A re-enable) | 5 | 5 | (per P2 summaries) |
| P3 (cluster C net-new E2E) | 8 | 9 | 42 |
| P4 (cluster D API gap fill) | 3 | 3 | 22 |
| **Phase totals (P3 + P4 net-new)** | **11** | **12** | **64** |

After P4-c runs green in CI, update the rollout-plan checklist line
and flip the cluster-D aggregate row to ✅ done. With cluster C and
cluster D both complete, the full-suite rollout (P1 → P4-c) is
delivered.
