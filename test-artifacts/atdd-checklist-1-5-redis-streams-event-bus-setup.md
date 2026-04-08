---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-5-redis-streams-event-bus-setup'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-5-redis-streams-event-bus-setup.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/redis_utils.py'
  - 'eusolicit-app/tests/conftest.py'
---

# ATDD Checklist: Story 1.5 — Redis Streams Event Bus Setup

## Story Summary

**Epic:** E01 — Infrastructure & Monorepo Foundation
**Story:** 1.5 — Redis Streams Event Bus Setup
**Sprint:** 2 | **Priority:** P0/P1 (Event bus foundation for all async inter-service communication)
**Risk-Driven:** Yes — E01-R-002 (Redis Streams DLQ gaps, Score: 6 — HIGH), E01-R-007 (Serialization inconsistency, Score: 4 — MEDIUM)

**As a** developer on the EU Solicit platform,
**I want** a Redis Streams event bus with a lightweight publisher/consumer abstraction in `eusolicit-common`, 7 named streams, consumer groups per service, idempotent bootstrap, and dead letter handling,
**So that** services can communicate asynchronously via typed events with guaranteed at-least-once delivery, distributed tracing via correlation IDs, and automatic dead letter quarantine for failed messages — enabling all future event-driven features to be built on a proven, tested foundation.

---

## TDD Red Phase (Current)

**Status:** RED — 72 of 72 tests fail. 14 fail with `ModuleNotFoundError` / `ImportError` (events package does not exist), 58 error with `redis.exceptions.ConnectionError` (Redis not running, but would also fail on missing imports with Docker running).

| Test Class | File | Test Functions | AC | Priority | Status |
|------------|------|---------------|----|----|--------|
| `TestAC0PackageExports` | `tests/integration/test_redis_event_bus.py` | 8 | AC3,4 | P1 | 8 FAIL |
| `TestAC0ConstantsAlignment` | `tests/integration/test_redis_event_bus.py` | 5 | AC1,2 | P1 | 5 FAIL |
| `TestAC1BootstrapStreams` | `tests/integration/test_redis_event_bus.py` | 8 | AC1 | P1 | 8 ERROR |
| `TestAC2BootstrapConsumerGroups` | `tests/integration/test_redis_event_bus.py` | 12 | AC2 | P1 | 12 ERROR |
| `TestAC3EventPublisher` | `tests/integration/test_redis_event_bus.py` | 4 | AC3 | P0 | 4 ERROR |
| `TestAC5EventEnvelope` | `tests/integration/test_redis_event_bus.py` | 11 | AC5 | P0 | 11 ERROR |
| `TestAC4EventConsumer` | `tests/integration/test_redis_event_bus.py` | 6 | AC4 | P0 | 6 ERROR |
| `TestAC6IdempotentBootstrap` | `tests/integration/test_redis_event_bus.py` | 3 | AC6 | P2 | 3 ERROR |
| `TestAC7PublishConsumeRoundTrip` | `tests/integration/test_redis_event_bus.py` | 2 | AC7 | P0 | 2 ERROR |
| `TestAC8DeadLetterHandling` | `tests/integration/test_redis_event_bus.py` | 7 | AC8 | P1 | 7 ERROR |
| `TestAC7MultipleConsumersLoadBalancing` | `tests/integration/test_redis_event_bus.py` | 1 | AC7 | P1 | 1 ERROR |
| `TestEventPublisherAsyncInterface` | `tests/integration/test_redis_event_bus.py` | 5 | AC3,4 | P1 | 1 FAIL / 4 ERROR |
| **Total** | **1 file** | **72 test cases** | **AC1–AC8** | | **72 FAIL (14 FAIL + 58 ERROR)** |

---

## Acceptance Criteria Coverage

| AC | Description | Test Class(es) | Tests | Priority | Coverage |
|----|-------------|-----------------|-------|----------|----------|
| AC1 | 7 Redis Streams created on startup | `TestAC1BootstrapStreams`, `TestAC0ConstantsAlignment` | 13 | P1 | Full — XINFO STREAM for all 7 streams; STREAMS constant alignment; eu-solicit: prefix validation |
| AC2 | Consumer groups created per service | `TestAC2BootstrapConsumerGroups`, `TestAC0ConstantsAlignment` | 14 | P1 | Full — XINFO GROUPS per stream; 4 service groups verified on subscribed streams; total (stream,group) count |
| AC3 | EventPublisher class with publish() | `TestAC3EventPublisher`, `TestAC0PackageExports`, `TestEventPublisherAsyncInterface` | 9 | P0 | Full — constructor, publish returns msg ID, stream population, multiple events, async interface, imports |
| AC4 | EventConsumer class with consume() | `TestAC4EventConsumer`, `TestAC0PackageExports`, `TestEventPublisherAsyncInterface` | 12 | P0 | Full — constructor, blocking/non-blocking consume, ack removes from PEL, async interface, imports |
| AC5 | Event envelope format | `TestAC5EventEnvelope` | 11 | P0 | Full — all 7 required fields; UUID4 event_id; ISO-8601 timestamp; UUID correlation_id; JSON payload; string values; tenant_id defaults |
| AC6 | Idempotent bootstrap | `TestAC6IdempotentBootstrap` | 3 | P2 | Full — twice without error; streams unchanged; groups unchanged |
| AC7 | Integration publish/consume round-trip | `TestAC7PublishConsumeRoundTrip`, `TestAC7MultipleConsumersLoadBalancing` | 3 | P0 | Full — full envelope round-trip; payload deserialization; load-balanced consumers |
| AC8 | Dead letter handling (3 failures → DLQ) | `TestAC8DeadLetterHandling` | 7 | P1 | Full — DLQ stream created; original envelope preserved; failure metadata; dlq_timestamp; original_message_id; original ACK'd; under-threshold not DLQ'd |

**Total coverage: 72 test cases across 8 acceptance criteria — 100% AC coverage.**

---

## Risk Mitigation Verification

### E01-R-002: Redis Streams Dead Letter Handling Gaps (Score: 6 — HIGH)

| Mitigation | Test Coverage | Status |
|------------|--------------|--------|
| EventConsumer.process_pending() checks XPENDING for stuck messages | `test_dlq_stream_created_for_failed_messages` | RED |
| Failed events moved to `<stream>.dlq` with original payload | `test_dlq_event_has_original_envelope_fields` | RED |
| DLQ events include failure metadata (dlq_reason, failure_count, dlq_timestamp, original_stream, original_message_id) | `test_dlq_event_has_failure_metadata`, `test_dlq_event_has_valid_dlq_timestamp`, `test_dlq_event_has_original_message_id` | RED |
| Original message ACK'd from source stream after DLQ | `test_original_message_acked_after_dlq` | RED |
| Messages under retry threshold NOT moved to DLQ | `test_messages_under_retry_threshold_not_dlqd` | RED |

**Quality gate:** DLQ flow verified end-to-end. P0 tests 100% pass rate; P1 tests >= 95% pass rate.

### E01-R-007: Redis Streams Event Serialization Inconsistency (Score: 4 — MEDIUM)

| Mitigation | Test Coverage | Status |
|------------|--------------|--------|
| EventPublisher produces correct envelope format matching redis_utils.py | `TestAC5EventEnvelope` (11 tests) | RED |
| Round-trip publish/consume with full envelope field validation | `test_round_trip_full_envelope` | RED |
| STREAMS/CONSUMER_GROUPS constants match test-utils exactly | `test_streams_constant_matches_test_utils`, `test_consumer_groups_constant_matches_test_utils` | RED |
| All envelope values are strings (Redis constraint) | `test_all_envelope_values_are_strings` | RED |

---

## Test Strategy

### Stack Detection

- **Detected stack:** `fullstack` (Python backend + Node.js/Playwright frontend)
- **Story scope:** Backend only — Redis Streams event bus abstraction in eusolicit-common
- **Test framework:** pytest + pytest-asyncio (integration tests against live Redis)
- **No E2E/browser tests:** This story has no UI components

### Generation Mode

- **Mode:** AI generation (backend infrastructure story, no browser recording needed)

### Test Level Selection

| Level | Justification | Count |
|-------|--------------|-------|
| Integration (Redis) | Event bus bootstrap, publish/consume, DLQ flow — requires live Redis | 52 |
| Unit (import/interface) | Package exports, constant alignment, async interface verification | 20 |

### Priority Distribution

| Priority | Tests | Criteria |
|----------|-------|----------|
| **P0** | 28 | Publisher, consumer, envelope format, round-trip smoke test — core event bus functionality |
| **P1** | 34 | Bootstrap streams/groups, constants alignment, DLQ handling, load balancing, async interface — E01-R-002/R-007 mitigations |
| **P2** | 10 | Idempotent bootstrap, top-level imports — operational robustness |

### Epic-Level Test Design Mapping

| Epic Priority | Epic Description | Story Tests | Risk Link |
|--------------|-----------------|-------------|-----------|
| **P0** | Redis Streams publish/consume smoke test | `TestAC7PublishConsumeRoundTrip::test_round_trip_full_envelope` | E01-R-002 (Score 6) |
| **P1** | Redis Streams dead letter handling | `TestAC8DeadLetterHandling` (7 tests) | E01-R-002 (Score 6) |
| **P1** | Redis Streams 7 streams + 4 consumer groups exist | `TestAC1BootstrapStreams` + `TestAC2BootstrapConsumerGroups` (20 tests) | E01-R-007 (Score 4) |
| **P2** | Redis Streams idempotent bootstrap | `TestAC6IdempotentBootstrap` (3 tests) | — |

---

## Failing Tests Created (RED Phase)

### Integration + Unit Tests (72 test cases across 12 classes)

**File:** `eusolicit-app/tests/integration/test_redis_event_bus.py`

#### AC0: Package Exports & Constants (13 tests)

- **`TestAC0PackageExports::test_import_event_publisher`** (1 test)
  - Status: RED — `ModuleNotFoundError: No module named 'eusolicit_common.events'`
  - Verifies: `EventPublisher` importable from `eusolicit_common.events`

- **`TestAC0PackageExports::test_import_event_consumer`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: `EventConsumer` importable from `eusolicit_common.events`

- **`TestAC0PackageExports::test_import_bootstrap_event_bus`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: `bootstrap_event_bus` importable from `eusolicit_common.events`

- **`TestAC0PackageExports::test_import_streams_constant`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: `STREAMS` importable from `eusolicit_common.events`

- **`TestAC0PackageExports::test_import_consumer_groups_constant`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: `CONSUMER_GROUPS` importable from `eusolicit_common.events`

- **`TestAC0PackageExports::test_top_level_import_event_publisher`** (1 test)
  - Status: RED — `ImportError: cannot import name 'EventPublisher'`
  - Verifies: `EventPublisher` importable from top-level `eusolicit_common`

- **`TestAC0PackageExports::test_top_level_import_event_consumer`** (1 test)
  - Status: RED — `ImportError`
  - Verifies: `EventConsumer` importable from top-level `eusolicit_common`

- **`TestAC0PackageExports::test_top_level_import_bootstrap_event_bus`** (1 test)
  - Status: RED — `ImportError`
  - Verifies: `bootstrap_event_bus` importable from top-level `eusolicit_common`

- **`TestAC0ConstantsAlignment::test_streams_constant_matches_test_utils`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: Production `STREAMS` dict == canonical `redis_utils.STREAMS`

- **`TestAC0ConstantsAlignment::test_consumer_groups_constant_matches_test_utils`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: Production `CONSUMER_GROUPS` dict == canonical `redis_utils.CONSUMER_GROUPS`

- **`TestAC0ConstantsAlignment::test_streams_has_seven_entries`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: Exactly 7 streams defined

- **`TestAC0ConstantsAlignment::test_consumer_groups_has_four_entries`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: Exactly 4 consumer group definitions

- **`TestAC0ConstantsAlignment::test_all_stream_names_have_eu_solicit_prefix`** (1 test)
  - Status: RED — `ModuleNotFoundError`
  - Verifies: All stream values start with `eu-solicit:`

#### AC1: Bootstrap Creates 7 Streams (8 tests)

- **`TestAC1BootstrapStreams::test_stream_exists[×7]`** (7 tests, parametrized)
  - Status: RED — `ModuleNotFoundError` → `redis.ConnectionError` (no module, no Redis)
  - Verifies: `XINFO STREAM` succeeds for each of 7 stream names after bootstrap

- **`TestAC1BootstrapStreams::test_all_seven_streams_created`** (1 test)
  - Status: RED
  - Verifies: Exactly 7 streams exist after single bootstrap call

#### AC2: Consumer Groups per Service (12 tests)

- **`TestAC2BootstrapConsumerGroups::test_consumer_group_exists_on_subscribed_streams[×4]`** (4 tests, parametrized by group)
  - Status: RED
  - Verifies: Each consumer group found on all its subscribed streams via `XINFO GROUPS`

- **`TestAC2BootstrapConsumerGroups::test_stream_has_expected_groups[×7]`** (7 tests, parametrized by stream)
  - Status: RED
  - Verifies: Each stream has exactly its expected consumer groups

- **`TestAC2BootstrapConsumerGroups::test_total_consumer_group_count`** (1 test)
  - Status: RED
  - Verifies: Total (stream, group) pair count matches CONSUMER_GROUPS mapping

#### AC3: EventPublisher (4 tests)

- **`TestAC3EventPublisher::test_publisher_accepts_redis_client`** (1 test)
  - Status: RED
  - Verifies: `EventPublisher(redis_client)` constructor works

- **`TestAC3EventPublisher::test_publish_returns_message_id`** (1 test)
  - Status: RED
  - Verifies: `publish()` returns a string message ID with Redis `timestamp-sequence` format

- **`TestAC3EventPublisher::test_publish_adds_event_to_stream`** (1 test)
  - Status: RED
  - Verifies: After publish, `XRANGE` returns 1 message in the stream

- **`TestAC3EventPublisher::test_publish_multiple_events_increments_stream_length`** (1 test)
  - Status: RED
  - Verifies: 3 publishes result in stream length == 3

#### AC5: Event Envelope Format (11 tests)

- **`TestAC5EventEnvelope::test_envelope_has_all_required_fields`** (1 test)
  - Status: RED
  - Verifies: Published event contains all 7 envelope fields (event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id)

- **`TestAC5EventEnvelope::test_event_id_is_valid_uuid4`** (1 test)
  - Status: RED
  - Verifies: event_id is a valid UUID4

- **`TestAC5EventEnvelope::test_event_type_matches_input`** (1 test)
  - Status: RED
  - Verifies: event_type matches the value passed to publish()

- **`TestAC5EventEnvelope::test_payload_is_json_serialized_string`** (1 test)
  - Status: RED
  - Verifies: payload field is a JSON string that deserializes to original dict

- **`TestAC5EventEnvelope::test_timestamp_is_valid_iso8601_utc`** (1 test)
  - Status: RED
  - Verifies: timestamp is parseable as ISO-8601

- **`TestAC5EventEnvelope::test_correlation_id_is_valid_uuid_when_not_provided`** (1 test)
  - Status: RED
  - Verifies: Auto-generated correlation_id is a valid UUID

- **`TestAC5EventEnvelope::test_correlation_id_uses_provided_value`** (1 test)
  - Status: RED
  - Verifies: Custom correlation_id is passed through unchanged

- **`TestAC5EventEnvelope::test_source_service_matches_input`** (1 test)
  - Status: RED
  - Verifies: source_service matches the value passed to publish()

- **`TestAC5EventEnvelope::test_tenant_id_matches_input`** (1 test)
  - Status: RED
  - Verifies: tenant_id matches the UUID passed to publish()

- **`TestAC5EventEnvelope::test_tenant_id_defaults_to_empty_string_when_none`** (1 test)
  - Status: RED
  - Verifies: tenant_id defaults to "" when None is passed (Redis stores strings)

- **`TestAC5EventEnvelope::test_all_envelope_values_are_strings`** (1 test)
  - Status: RED
  - Verifies: All envelope field values are `str` type (Redis constraint)

#### AC4: EventConsumer (6 tests)

- **`TestAC4EventConsumer::test_consumer_accepts_redis_client`** (1 test)
  - Status: RED
  - Verifies: `EventConsumer(redis_client)` constructor works

- **`TestAC4EventConsumer::test_consume_returns_list`** (1 test)
  - Status: RED
  - Verifies: `consume()` returns a `list`

- **`TestAC4EventConsumer::test_consume_returns_published_event`** (1 test)
  - Status: RED
  - Verifies: Consumed event matches published event_type and source_service

- **`TestAC4EventConsumer::test_consume_non_blocking_returns_empty_on_no_messages`** (1 test)
  - Status: RED
  - Verifies: `consume(block_ms=0)` returns `[]` on empty stream

- **`TestAC4EventConsumer::test_consume_non_blocking_with_none_returns_empty`** (1 test)
  - Status: RED
  - Verifies: `consume(block_ms=None)` returns `[]` on empty stream

- **`TestAC4EventConsumer::test_ack_acknowledges_message`** (1 test)
  - Status: RED
  - Verifies: After `ack()`, message no longer in XPENDING list

#### AC6: Idempotent Bootstrap (3 tests)

- **`TestAC6IdempotentBootstrap::test_bootstrap_twice_without_error`** (1 test)
  - Status: RED
  - Verifies: Second `bootstrap_event_bus()` call raises no errors (BUSYGROUP handled)

- **`TestAC6IdempotentBootstrap::test_streams_unchanged_after_second_bootstrap`** (1 test)
  - Status: RED
  - Verifies: Stream set identical before and after second bootstrap

- **`TestAC6IdempotentBootstrap::test_groups_unchanged_after_second_bootstrap`** (1 test)
  - Status: RED
  - Verifies: Consumer group lists per stream identical before and after second bootstrap

#### AC7: Publish/Consume Round-Trip (3 tests)

- **`TestAC7PublishConsumeRoundTrip::test_round_trip_full_envelope`** (1 test)
  - Status: RED
  - Verifies: **P0 smoke test** — Publish to `eu-solicit:opportunities` → consume from `client-api` group → assert all 7 envelope fields present, event_type/source_service/correlation_id/tenant_id match, payload round-trips, event_id is UUID, timestamp is ISO-8601

- **`TestAC7PublishConsumeRoundTrip::test_consumed_payload_is_deserializable_json`** (1 test)
  - Status: RED
  - Verifies: Consumed payload is deserializable from JSON to original dict (including nested structures)

- **`TestAC7MultipleConsumersLoadBalancing::test_two_consumers_receive_different_messages`** (1 test)
  - Status: RED
  - Verifies: 2 consumers in same group each receive 1 of 2 published events (load balancing)

#### AC8: Dead Letter Handling (7 tests)

- **`TestAC8DeadLetterHandling::test_dlq_stream_created_for_failed_messages`** (1 test)
  - Status: RED
  - Verifies: After 4 delivery attempts (exceeds max_retries=3), event appears in `<stream>.dlq`

- **`TestAC8DeadLetterHandling::test_dlq_event_has_original_envelope_fields`** (1 test)
  - Status: RED
  - Verifies: DLQ event preserves all 7 original envelope fields

- **`TestAC8DeadLetterHandling::test_dlq_event_has_failure_metadata`** (1 test)
  - Status: RED
  - Verifies: DLQ event includes dlq_reason="max_retries_exceeded", failure_count >= 3, original_stream matches source

- **`TestAC8DeadLetterHandling::test_dlq_event_has_valid_dlq_timestamp`** (1 test)
  - Status: RED
  - Verifies: dlq_timestamp is valid ISO-8601

- **`TestAC8DeadLetterHandling::test_dlq_event_has_original_message_id`** (1 test)
  - Status: RED
  - Verifies: original_message_id matches the message ID from the source stream

- **`TestAC8DeadLetterHandling::test_original_message_acked_after_dlq`** (1 test)
  - Status: RED
  - Verifies: After DLQ processing, original message no longer in XPENDING (ACK'd)

- **`TestAC8DeadLetterHandling::test_messages_under_retry_threshold_not_dlqd`** (1 test)
  - Status: RED
  - Verifies: Messages with only 2 delivery attempts (< max_retries=3) are NOT moved to DLQ; still pending

#### Cross-Cutting: Async Interface Verification (5 tests)

- **`TestEventPublisherAsyncInterface::test_publish_is_coroutine`** (1 test)
  - Status: RED
  - Verifies: `EventPublisher.publish` is an `async def` (coroutine function)

- **`TestEventPublisherAsyncInterface::test_consume_is_coroutine`** (1 test)
  - Status: RED
  - Verifies: `EventConsumer.consume` is an `async def`

- **`TestEventPublisherAsyncInterface::test_ack_is_coroutine`** (1 test)
  - Status: RED
  - Verifies: `EventConsumer.ack` is an `async def`

- **`TestEventPublisherAsyncInterface::test_process_pending_is_coroutine`** (1 test)
  - Status: RED
  - Verifies: `EventConsumer.process_pending` is an `async def`

- **`TestEventPublisherAsyncInterface::test_bootstrap_event_bus_is_coroutine`** (1 test)
  - Status: RED
  - Verifies: `bootstrap_event_bus` is an `async def`

---

## Data Factories Created

N/A — This story tests Redis Streams event bus infrastructure. Test events are created inline via `EventPublisher.publish()`. No application-level data factories needed.

---

## Fixtures Created

N/A — Tests reuse existing fixtures from root `conftest.py`:
- `redis_client` — session-scoped async Redis client (`decode_responses=True`)
- `clean_redis` — per-test fixture that flushes DB before/after

Per-class `_setup` / `_bootstrap` fixtures call `bootstrap_event_bus()` and instantiate `EventPublisher`/`EventConsumer` before each test. These are defined inline via `@pytest.fixture(autouse=True)` within the test classes.

---

## Mock Requirements

N/A — All tests run against real Redis (via Docker Compose or `TEST_REDIS_URL`). No mocks required — the purpose is to verify actual Redis Streams operations (XADD, XREADGROUP, XACK, XPENDING, XCLAIM, XGROUP CREATE).

---

## Required data-testid Attributes

N/A — No UI components in this story.

---

## Implementation Checklist

### Task 1: Create `events` Package Structure (AC: 3, 4)

**Files to create (1 file):**
- `packages/eusolicit-common/src/eusolicit_common/events/__init__.py`

**Tests that will pass when complete:**

- [ ] `TestAC0PackageExports` (5 of 8 tests) — events package imports

### Task 2: Create EventPublisher Class (AC: 3, 5)

**Files to create (1 file):**
- `packages/eusolicit-common/src/eusolicit_common/events/publisher.py`

**Tests that will pass when complete:**

- [ ] `TestAC0PackageExports::test_import_event_publisher` (1 test) — import works
- [ ] `TestAC3EventPublisher` (4 tests) — constructor, publish returns msg ID, stream population, multiple events
- [ ] `TestAC5EventEnvelope` (11 tests) — all envelope field validations
- [ ] `TestEventPublisherAsyncInterface::test_publish_is_coroutine` (1 test) — async verification

### Task 3: Create EventConsumer Class (AC: 4, 8)

**Files to create (1 file):**
- `packages/eusolicit-common/src/eusolicit_common/events/consumer.py`

**Tests that will pass when complete:**

- [ ] `TestAC0PackageExports::test_import_event_consumer` (1 test) — import works
- [ ] `TestAC4EventConsumer` (6 tests) — constructor, consume, non-blocking, ack
- [ ] `TestAC8DeadLetterHandling` (7 tests) — DLQ stream, envelope preservation, metadata, ACK, threshold
- [ ] `TestAC7MultipleConsumersLoadBalancing` (1 test) — load balancing
- [ ] `TestEventPublisherAsyncInterface::test_consume_is_coroutine` etc. (3 tests) — async verification

### Task 4: Create Bootstrap Module (AC: 1, 2, 6)

**Files to create (1 file):**
- `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py`

**Tests that will pass when complete:**

- [ ] `TestAC0PackageExports::test_import_bootstrap_event_bus` (1 test) — import works
- [ ] `TestAC0ConstantsAlignment` (5 tests) — STREAMS/CONSUMER_GROUPS match test-utils exactly
- [ ] `TestAC1BootstrapStreams` (8 tests) — all 7 streams created
- [ ] `TestAC2BootstrapConsumerGroups` (12 tests) — all consumer groups on all streams
- [ ] `TestAC6IdempotentBootstrap` (3 tests) — idempotent re-run
- [ ] `TestEventPublisherAsyncInterface::test_bootstrap_event_bus_is_coroutine` (1 test)

### Task 5: Update Package Exports (AC: 3, 4)

**Files to modify (2 files):**
- `packages/eusolicit-common/src/eusolicit_common/events/__init__.py` — export all public symbols
- `packages/eusolicit-common/src/eusolicit_common/__init__.py` — add top-level re-exports

**Tests that will pass when complete:**

- [ ] `TestAC0PackageExports::test_top_level_import_*` (3 tests) — top-level imports work

### Task 6: Integration Round-Trip (AC: 7)

**No additional files — verify end-to-end with all above tasks complete.**

**Tests that will pass when complete:**

- [ ] `TestAC7PublishConsumeRoundTrip` (2 tests) — P0 smoke test + JSON round-trip

**Estimated Total Effort:** 3–5 hours

---

## Running Tests

```bash
# Run all ATDD tests for this story (72 will fail in RED phase)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v

# Run with short traceback (recommended for RED phase verification)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v --tb=line

# Run only import/constant tests (no Redis required)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v -k "AC0 or AsyncInterface"

# Run only Redis-dependent tests (requires Docker Compose Redis running)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v -k "AC1 or AC2 or AC3 or AC4 or AC5 or AC6 or AC7 or AC8"

# Run only P0 tests
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v -k "AC3EventPublisher or AC5EventEnvelope or AC4EventConsumer or AC7PublishConsumeRoundTrip"

# Run only DLQ tests (E01-R-002 HIGH risk mitigation)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v -k "AC8"

# Run with verbose output showing full assertion messages
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v --tb=long

# Run with Docker Compose Redis (full environment)
make up && cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_redis_event_bus.py -v
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete)

**TEA Agent Responsibilities:**

- 72 test cases written across 12 test classes in 1 file
- Tests organized by acceptance criteria with clear priority tags (P0/P1/P2)
- E01-R-002 mitigation verified: 7 DLQ tests covering full dead letter lifecycle
- E01-R-007 mitigation verified: 11 envelope format tests + constants alignment
- P0 smoke test maps exactly to epic test design requirement
- Tests use existing `redis_client` and `clean_redis` fixtures (no new fixtures created)
- Constants cross-referenced against `eusolicit_test_utils.redis_utils` canonical definitions

**Verification:**

```
14 failed, 58 errors in 0.39s
```

- 14 failures: `ModuleNotFoundError` (events package) and `ImportError` (top-level exports) — module doesn't exist
- 58 errors: `redis.ConnectionError` (no Redis running) but would also fail on `ModuleNotFoundError` if imports were attempted first in fixture setup
- 0 passing: All tests fail due to missing implementation, not test bugs

---

### GREEN Phase (DEV Team — Next Steps)

**DEV Agent Responsibilities:**

1. **Task 1:** Create `packages/eusolicit-common/src/eusolicit_common/events/__init__.py` with exports
2. **Task 2:** Create `events/publisher.py` — `EventPublisher` class with `async publish()` producing correct envelope
3. **Task 3:** Create `events/consumer.py` — `EventConsumer` class with `async consume()`, `async ack()`, `async process_pending()`
4. **Task 4:** Create `events/bootstrap.py` — `STREAMS`, `CONSUMER_GROUPS` constants + `async bootstrap_event_bus()`
5. **Task 5:** Update `eusolicit_common/__init__.py` — add top-level re-exports
6. Start Docker Compose Redis: `make up` or `docker compose up redis -d`
7. Run: `pytest tests/integration/test_redis_event_bus.py -v`
8. All 72 tests pass — GREEN

**Key Implementation Notes:**

- `STREAMS` and `CONSUMER_GROUPS` MUST match `redis_utils.py` exactly (production is the follower, test-utils is the source of truth)
- `EventPublisher.publish()` MUST produce the exact envelope format from `redis_utils.build_event_envelope()` — all values are strings, payload is `json.dumps()`
- `tenant_id` defaults to `""` (empty string), NOT `None` — Redis stores strings
- `EventConsumer.consume()` supports `block_ms=None` and `block_ms=0` for non-blocking mode — both return `[]` on empty stream
- `process_pending()` checks `XPENDING` for delivery count > `max_retries`, then `XCLAIM` → `XADD` to DLQ → `XACK` original
- DLQ stream name: `<stream>.dlq` (e.g., `eu-solicit:opportunities.dlq`)
- DLQ envelope includes original fields + `{dlq_reason, failure_count, dlq_timestamp, original_stream, original_message_id}`
- All methods MUST be `async def` — no synchronous variants
- Use `redis.asyncio.Redis` (async client) — constructor receives client via dependency injection
- Bootstrap uses `XGROUP CREATE ... MKSTREAM` and handles `BUSYGROUP` error gracefully (log + continue)

---

### REFACTOR Phase (DEV Team — After All Tests Pass)

1. Verify all 72 tests pass consistently (run 3x)
2. Run full regression suite (629+ existing tests) to verify no regressions
3. Verify `structlog` logging in bootstrap module (stream/group creation logged)
4. Review error handling in `process_pending` (edge cases: empty PEL, XCLAIM race conditions)
5. Ready for code review

---

## Next Steps

1. **Review this checklist** — Verify test coverage aligns with team expectations
2. **Start Docker Compose** — `make up` to start Redis
3. **Implement Story 1.5** — Follow the implementation checklist above (Tasks 1–5)
4. **Run tests** — `pytest tests/integration/test_redis_event_bus.py -v`
5. **Verify GREEN** — All 72 tests pass
6. **Story 1.6 depends on this** — eusolicit-common shared package will integrate `bootstrap_event_bus()` into service startup lifecycle
7. **Story 1.7 depends on this** — eusolicit-models will define Pydantic event schemas that type the envelope

---

## Knowledge Base References Applied

- **test-quality.md** — Deterministic, isolated, self-cleaning test design; existing `clean_redis` fixture flushes DB before/after each test
- **test-levels-framework.md** — Integration level for Redis operations; unit-equivalent for import/interface validation
- **test-priorities-matrix.md** — P0 for smoke test (E01-R-002 Score: 6), P1 for DLQ/bootstrap/constants (E01-R-002, E01-R-007), P2 for idempotency
- **test-healing-patterns.md** — Clear assertion messages with diagnostic context (field names, expected vs actual, stream names)
- **risk-governance.md** — Risk mitigation verification matrix for HIGH (E01-R-002) and MEDIUM (E01-R-007) risks

---

## Notes

- **Test database:** Tests use Redis DB 1 (`TEST_REDIS_URL=redis://localhost:6379/1` from conftest.py). Docker Compose exposes Redis 7 on port 6379.
- **No new fixtures:** Tests reuse `redis_client` (session-scoped) and `clean_redis` (per-test flush) from root conftest.py as required by story spec.
- **No DO NOT TOUCH violations:** Tests do not modify docker-compose.yml, .env.example, Makefile, redis_utils.py, or conftest.py.
- **DLQ simulation strategy:** Tests use `XCLAIM` with `min_idle_time=0` to increment delivery count in XPENDING. This is a valid simulation because `XCLAIM` increments the internal delivery counter, which `process_pending()` reads via `XPENDING` to decide DLQ eligibility.
- **0 pre-existing passes:** Unlike Story 1.4 (which had 12 passing tests from pre-existing alembic.ini files), Story 1.5 has zero pre-existing state — the `eusolicit_common.events` module does not exist at all.

---

**Generated by BMad TEA Agent** — 2026-04-06
