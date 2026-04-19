---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-19'
---

# Test Design: Epic 09 - Notifications, Alerts & Calendar

**Date:** 2026-04-19
**Author:** BMad TEA Agent
**Status:** Draft

---

## Executive Summary

**Scope:** full test design for Epic 09

**Risk Summary:**

- Total risks identified: 4
- High-priority risks (≥6): 2
- Critical categories: TECH, BUS

**Coverage Summary:**

- P0 scenarios: 4 (~18 hours)
- P1 scenarios: 4 (~13 hours)
- P2/P3 scenarios: 4 (~10 hours)
- **Total effort**: ~41 hours (~5 days)

---

## Not in Scope

| Item | Reasoning | Mitigation |
| ---------- | -------------- | --------------------- |
| **SendGrid Uptime** | Third-party service | Ensure webhook handles delays |
| **Google/MS Calendar Outages** | Third-party service | Handle API errors gracefully via Retry-After and sync logs |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| ------- | -------- | ------------- | ----------- | ------ | ----- | ------------ | ------- | -------- |
| R-001 | TECH | Redis stream event loss for immediate notifications | 2 | 3 | 6 | Implement consumer groups with manual ACK | Backend | S09.04 |
| R-002 | BUS | Stripe Usage over/under-reporting | 2 | 3 | 6 | Integration tests with Stripe simulator | Backend | S09.11 |

### Medium-Priority Risks (Score 3-4)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| ------- | -------- | ------------- | ----------- | ------ | ----- | ------------ | ------- |
| R-003 | PERF | Daily digest matching logic CPU/DB spike | 2 | 2 | 4 | Optimize queries, batch processing | Backend |
| R-004 | SEC | OAuth token exposure | 1 | 3 | 3 | Ensure Fernet encryption handles keys securely | Security |

### Low-Priority Risks (Score 1-2)

None identified at this time.

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Entry Criteria

- [ ] Requirements and assumptions agreed upon by QA, Dev, PM
- [ ] Test environment provisioned and accessible with Redis and Celery workers
- [ ] SendGrid sandbox/API keys provided
- [ ] Google and Microsoft OAuth test credentials available
- [ ] Stripe test mode available

## Exit Criteria

- [ ] All P0 tests passing
- [ ] All P1 tests passing (or failures triaged)
- [ ] No open high-priority / high-severity bugs
- [ ] Test coverage agreed as sufficient

## Project Team (Optional)

**Include only if roles/names are known or responsibility mapping is needed; otherwise omit.**

---

## Test Coverage Plan

### P0 (Critical) - Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ------------- | ---------- | --------- | ---------- | ----- | ------- |
| Immediate Notification Match | E2E | R-001 | 1 | QA | Trigger ingested event, check API |
| Daily/Weekly Digest | API | R-003 | 1 | QA | Validate user matching and data grouping |
| Stripe Usage Sync | API | R-002 | 1 | QA | Verify usage numbers sent to Stripe API |
| Alert Preferences CRUD | E2E | - | 1 | QA | Complete CRUD journey via UI |

**Total P0**: 4 tests, 18 hours

### P1 (High) - Run on PR to main

**Criteria**: Important features + Medium risk (3-4) + Common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ------------- | ---------- | --------- | ---------- | ----- | ------- |
| SendGrid Webhook | API | - | 1 | DEV | Mock failure/bounce events |
| iCal Feed Generation | API | - | 1 | QA | Validate .ics format and token auth |
| OAuth Connections | E2E | R-004 | 1 | QA | Google/Microsoft Connect/Disconnect flows |
| Task/Approval emails | API | - | 1 | DEV | Asserts routing logic |

**Total P1**: 4 tests, 13 hours

### P2 (Medium) - Run nightly/weekly

**Criteria**: Secondary features + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ------------- | ---------- | --------- | ---------- | ----- | ------- |
| Celery Retry Backoff | API | - | 1 | DEV | Mock failing task |
| Calendar Sync diff | API | - | 1 | DEV | Validate creates/updates/deletes logic |

**Total P2**: 2 tests, 6 hours

### P3 (Low) - Run on-demand

**Criteria**: Nice-to-have + Exploratory + Performance benchmarks

| Requirement | Test Level | Test Count | Owner | Notes |
| ------------- | ---------- | ---------- | ----- | ------- |
| Email templates render | Component | 1 | DEV | Visual check of emails |
| Calendar UI polling | UI | 1 | QA | Verify status updates automatically |

**Total P3**: 2 tests, 4 hours

---

## Execution Order

### Smoke Tests (<5 min)

**Purpose**: Fast feedback, catch build-breaking issues

- [ ] Alert Preferences CRUD API checks

### P0 Tests (<10 min)

**Purpose**: Critical path validation

- [ ] Immediate Notification Match (E2E)
- [ ] Stripe Usage Sync (API)
- [ ] Daily/Weekly Digest (API)

### P1 Tests (<30 min)

**Purpose**: Important feature coverage

- [ ] OAuth Connections (E2E)
- [ ] SendGrid Webhook (API)

### P2/P3 Tests (<60 min)

**Purpose**: Full regression coverage

- [ ] Celery Retry Backoff (API)
- [ ] Email templates render (Component)

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
| --------- | ----------------- | ---------- | ----------------- | ----------------------- |
| P0 | 4 | 4.5 | 18 | Complex setup, Celery/Redis mocking |
| P1 | 4 | 3.25 | 13 | External OAuth and Webhook mocking |
| P2 | 2 | 3.0 | 6 | Simple API validation scenarios |
| P3 | 2 | 2.0 | 4 | Exploratory, visual |
| **Total** | **12** | **-** | **41** | **~5 days** |

### Prerequisites

**Test Data:**
- User factory (with alert preferences)
- Opportunity factory
- Stripe subscription factory

**Tooling:**
- Playwright Utils for API/E2E
- Celery Task Mocker

**Environment:**
- Redis instance for streams
- Celery beat and worker processes running

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: ≥95% (waivers required for failures)
- **P2/P3 pass rate**: ≥90% (informational)
- **High-risk mitigations**: 100% complete or approved waivers

### Coverage Targets

- **Critical paths**: ≥80%
- **Security scenarios**: 100%
- **Business logic**: ≥70%
- **Edge cases**: ≥50%

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] No high-risk (≥6) items unmitigated
- [ ] Security tests (SEC category) pass 100%

---

## Mitigation Plans

### R-001: Redis stream event loss for immediate notifications (Score: 6)

**Mitigation Strategy:** Implement consumer groups with manual ACK. Celery to process with retries.
**Owner:** Backend
**Timeline:** S09.04
**Status:** Planned
**Verification:** Force worker crash during processing, verify event is re-delivered and processed correctly after restart.

### R-002: Stripe Usage over/under-reporting (Score: 6)

**Mitigation Strategy:** Integration tests with Stripe simulator to ensure correct aggregation and submission.
**Owner:** Backend
**Timeline:** S09.11
**Status:** Planned
**Verification:** Automated test asserting exact reported totals against simulated usage data.

---

## Assumptions and Dependencies

### Assumptions

1. SendGrid dynamic templates will be pre-configured.
2. Redis is accessible from Celery workers with appropriate network configuration.

### Dependencies

1. Shared Redis from E01 event bus.
2. Analytics queries from E12 (for Scheduled Reports).

### Risks to Plan

- **Risk**: Delay in getting verified Google/Microsoft OAuth credentials
  - **Impact**: Inability to test calendar sync E2E
  - **Contingency**: Rely on API mocking for graph endpoints until credentials arrive.

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
| ----------------- | -------------- | ------------------------------- |
| **Event Bus (E01)** | Consumes events | Ensure opportunity publishing still functions |
| **Client API** | Shared schema | Verify Client API endpoints interact correctly with `notification` schema changes |

---

## Appendix

### Knowledge Base References

- `risk-governance.md` - Risk classification framework
- `probability-impact.md` - Risk scoring methodology
- `test-levels-framework.md` - Test level selection
- `test-priorities-matrix.md` - P0-P3 prioritization

### Related Documents

- Epic: eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md
- Architecture: test-design-architecture.md

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)