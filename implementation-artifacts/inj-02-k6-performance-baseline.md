# INJ-02: k6 Performance Baseline
**Status:** ready-for-dev
**Epic:** 13
**Priority:** High

As a platform operator,
I want k6 load test scripts written, executed, and committed,
so that PRD SLAs (p95 <200ms REST, <500ms SSE TTFB) are verifiable.

## Acceptance Criteria:
1. k6 script: REST p95 latency for GET /opportunities (search with FTS) — must pass <200ms at 50 VUs
2. k6 script: SSE TTFB for POST /opportunities/:id/summary (AI summary stream) — first byte <500ms at 10 VUs
3. k6 script: Redis usage metering counter under 100 concurrent INCR operations
4. Results written to eusolicit-docs/implementation-artifacts/load-test-results.md
5. Scripts committed to eusolicit-app/tests/load/

## Tasks:
- [ ] Create tests/load/ directory
- [ ] Write k6 script for REST search (k6 run with thresholds)
- [ ] Write k6 script for SSE TTFB (k6 experimental streams or http.get + check timing)
- [ ] Write k6 script for Redis metering
- [ ] Execute against running services (make up)
- [ ] Fill load-test-results.md with actual p50/p95/p99 values
- [ ] Commit all scripts
