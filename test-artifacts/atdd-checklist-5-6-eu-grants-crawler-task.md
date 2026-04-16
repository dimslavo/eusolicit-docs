# ATDD Checklist: Story 5.6 - EU Grants Crawler Task

This checklist translates the acceptance criteria from the user story into concrete, testable scenarios. These tests will fail until the development tasks are complete.

**Epic Test Design Traceability:**
- **E05-P0-006:** EU Grants field completeness
- **E05-P1-016:** EU Grants budget range parsing
- **E05-P1-017:** EU Grants `opportunity_type = grant`
- **E05-P2-007:** EU Grants `crawler_runs` record accuracy

---

### Acceptance Criterion 1: Grant-Specific Fields

> Grant opportunities stored with `opportunity_type='grant'` and populated `evaluation_criteria` JSONB (scoring matrix structure from the agent response) on every upserted record.

| Test ID | Scenario | Status |
|---|---|---|
| **ATDD-5.6-01** | When a crawl is successful, all upserted opportunity records from the EU Grants portal have their `opportunity_type` column set to `'grant'`. | ⬜️ Pending |
| **ATDD-5.6-02** | When the EU Grant Portal Agent provides a JSON structure in the `evaluation_criteria` field, that exact JSON is stored in the `pipeline.opportunities.evaluation_criteria` column for the corresponding record. | ⬜️ Pending |
| **ATDD-5.6-03** | When an existing EU grant opportunity is re-crawled, the `opportunity_type` remains `'grant'` after the `ON CONFLICT` update. | ⬜️ Pending |

### Acceptance Criterion 2: Mandatory Documents

> `mandatory_documents` JSONB contains the structured list of required documents per grant call, as returned by the EU Grant Portal Agent and preserved through normalization.

| Test ID | Scenario | Status |
|---|---|---|
| **ATDD-5.6-04** | When the EU Grant Portal Agent provides a JSON list in the `mandatory_documents` field, that exact JSON is stored in the `pipeline.opportunities.mandatory_documents` column. | ⬜️ Pending |
| **ATDD-5.6-05** | If the agent response for an opportunity omits the `mandatory_documents` field, the corresponding database column is `NULL`. | ⬜️ Pending |

### Acceptance Criterion 3: Budget Parsing

> Budget range correctly parsed into `budget_min` / `budget_max`: when the source provides a single scalar (`budget_value: 2000000`), both `budget_min` and `budget_max` are set to that value; when the source provides a range string (`budget_range: "50000-200000"`), it is split on `-` into `budget_min=50000` and `budget_max=200000`.

| Test ID | Scenario | Status |
|---|---|---|
| **ATDD-5.6-06** | **(Unit Test)** An agent response item with a numeric `budget_value` (e.g., `100000`) is normalized to `budget_min=100000` and `budget_max=100000`. | ⬜️ Pending |
| **ATDD-5.6-07** | **(Unit Test)** An agent response item with a string `budget_range` (e.g., `"50000-200000"`) is normalized to `budget_min=50000` and `budget_max=200000`. | ⬜️ Pending |
| **ATDD-5.6-08** | **(Unit Test)** An agent response item with a `budget_range` string containing extra whitespace (e.g., `" 50000 - 200000 "`) is parsed correctly. | ⬜️ Pending |
| **ATDD-5.6-09** | **(Unit Test)** An agent response item with a standard `budget` dictionary (`{"min": 10000, "max": 50000}`) is parsed correctly. | ⬜️ Pending |
| **ATDD-5.6-10** | **(Unit Test)** An agent response item with no budget-related keys results in `budget_min=NULL` and `budget_max=NULL`. | ⬜️ Pending |

### Acceptance Criterion 4 & 5: Crawl Execution and Deduplication

> Successful crawl inserts/updates opportunities with `source_type='eu_grants'` and records accurate counts in `crawler_runs`. Duplicate source records (same `source_id`, `source_type='eu_grants'`) are updated via atomic `INSERT ... ON CONFLICT ... DO UPDATE`.

| Test ID | Scenario | Status |
|---|---|---|
| **ATDD-5.6-11** | **(Integration Test)** A successful run of `crawl_eu_grants` creates a `crawler_runs` record with `crawler_type='eu_grants'`, `status='completed'`, and `ended_at` populated. | ⬜️ Pending |
| **ATDD-5.6-12** | **(Integration Test)** The `crawler_runs` record accurately reflects the counts: `found` (total from agent), `new_count` (newly inserted), and `updated` (updated on conflict). | ⬜️ Pending |
| **ATDD-5.6-13** | **(Integration Test)** Running a crawl for 5 new opportunities results in `new_count=5` and `updated=0`. | ⬜️ Pending |
| **ATDD-5.6-14** | **(Integration Test)** Running the same crawl again with identical data results in `new_count=0` and `updated=0` (or `updated=5` if fields changed), but no new rows are created. | ⬜️ Pending |

### Acceptance Criterion 6 & 7: Failure and Retry Handling

> AI Gateway failure (`AIGatewayUnavailableError`) triggers Celery-level retry with `max_retries=3`. `crawler_runs.status` is always transitioned to a terminal state (`completed` or `failed`).

| Test ID | Scenario | Status |
|---|---|---|
| **ATDD-5.6-15** | **(Integration Test)** When the AI Gateway call to `eu-grant-portal-agent` fails with `AIGatewayUnavailableError`, the `crawl_eu_grants` task is retried. | ⬜️ Pending |
| **ATDD-5.6-16** | **(Integration Test)** During a retry attempt, the corresponding `crawler_runs` record has its status updated to `'retrying'`. | ⬜️ Pending |
| **ATDD-5.6-17** | **(Integration Test)** After exhausting all 3 retries, the `crawler_runs` record status is set to `'failed'`, and the `errors` field is populated. | ⬜️ Pending |
| **ATDD-5.6-18** | **(Integration Test)** A non-retriable exception during the task correctly invokes the `on_failure` handler, which sets the `crawler_runs` status to `'failed'`. | ⬜️ Pending |

### Acceptance Criterion 8: Test Coverage

> Integration tests cover: (a) happy path — grant-specific fields populated, counts correct; (b) gateway-down — status=`failed` after max retries; (c) `crawler_runs` record accuracy.

| Test ID | Scenario | Status |
|---|---|---|
| **ATDD-5.6-19** | The integration test suite includes a "happy path" test that mocks a full, successful crawl and asserts the database state (opportunities created, fields correct, `crawler_runs` correct). | ⬜️ Pending |
| **ATDD-5.6-20** | The integration test suite includes a "gateway down" test that simulates AI Gateway unavailability and asserts the final `crawler_runs` status is `'failed'` after retries. | ⬜️ Pending |
| **ATDD-5.6-21** | The unit test suite for the normalizer covers all specified budget parsing cases and field passthrough logic. | ⬜️ Pending |
