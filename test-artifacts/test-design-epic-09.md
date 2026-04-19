---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-19'
---

# Test Design: Epic 09 - Notifications, Alerts & Calendar

**Date:** 2026-04-19
**Author:** BMad TEA Agent
**Status:** Architecture Review Pending
**Project:** EU Solicit
**Epic Reference:** epic-09-notifications-alerts-calendar.md

---

## Executive Summary

**Scope:** Full test design for Epic 09, covering the Celery-based Notification Service, immediate/digest email alerts, third-party calendar integrations (Google, Microsoft, iCal), and Stripe usage reporting.

**Business Context:** Reliable, timely notification dispatch is critical for SaaS user engagement. Calendar integration supports users' external workflows. Accurate Stripe usage reporting is critical for revenue.

**Expected Scale:** High-volume event consumption from Redis streams (tenders, tasks, approvals). Strict rate-limits (SendGrid, Microsoft Graph, Google Calendar API).

**Risk Summary:**
- Total risks identified: 7
- High-priority risks (≥6): 3
- Critical categories: DATA, BUS, SEC

**Coverage Summary:**
- P0 scenarios: 8 (~25 hours)
- P1 scenarios: 7 (~18 hours)
- P2/P3 scenarios: 5 (~12 hours)
- **Total effort**: ~55 hours (~7 days)

---

## Not in Scope

| Item | Reasoning | Mitigation |
| ---------- | -------------- | --------------------- |
| **SendGrid Uptime** | Third-party service | Ensure webhook handles delays and task retries are robust |
| **Google/MS Calendar Outages** | Third-party service | Handle API errors gracefully via `Retry-After` and sync logs |
| **Full Analytics Reporting Content** | E12 dependency | S09.14 creates the pipeline only; content tested in E12 |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| ------- | -------- | ------------- | ----------- | ------ | ----- | ------------ | ------- | -------- |
| R-001 | DATA | Redis stream event loss for immediate notifications | 2 | 3 | 6 | Implement consumer groups with manual ACK; DLQ for failures | Backend | S09.04 |
| R-002 | BUS | Stripe Usage over/under-reporting | 2 | 3 | 6 | Integration tests with Stripe simulator; idempotent clearing | Backend | S09.11 |
| R-003 | SEC | OAuth token exposure & invalidation | 2 | 3 | 6 | Fernet encryption; strict scopes; revocation flows | Security | S09.08, S09.09 |

### Medium-Priority Risks (Score 3-5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| ------- | -------- | ------------- | ----------- | ------ | ----- | ------------ | ------- |
| R-004 | PERF | Daily digest matching logic CPU/DB spike | 2 | 2 | 4 | In-memory preference caching, batch queries | Backend |
| R-005 | TECH | Third-party rate limits (Google/MS/SendGrid) | 3 | 1 | 3 | Exponential backoff retries, Celery rate limits | Backend |
| R-006 | DATA | Race conditions clearing Redis usage counters | 2 | 2 | 4 | Atomic Redis pipeline operations (GET/DEL) | Backend |

### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| ------- | -------- | ------------- | ----------- | ------ | ----- | ------------ | ------- |
| R-007 | UI | Calendar sync polling overhead | 2 | 1 | 2 | 30s interval with cleanup on unmount | Frontend |

---

## Entry Criteria

- [ ] Requirements and assumptions agreed upon by QA, Dev, PM
- [ ] Test environment provisioned with Redis, Celery workers, and PostgreSQL
- [ ] SendGrid sandbox/API keys provided
- [ ] Google and Microsoft OAuth test credentials available
- [ ] Stripe test mode available and configured

## Exit Criteria

- [ ] All P0 and P1 tests passing
- [ ] No open high-priority / high-severity bugs (Severity 1 or 2)
- [ ] External API error handling (429, 5xx) verified
- [ ] Test coverage agreed as sufficient

---

## Test Coverage Plan

### P0 (Critical) - Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ------------- | ---------- | --------- | ---------- | ----- | ------- |
| Immediate Alert Dispatch | E2E | R-001 | 1 | QA | Push to Redis, assert SendGrid task triggered with correct data |
| Daily/Weekly Digest | API/Worker | R-004 | 2 | QA | Validate accurate opportunity grouping & exclusion of zero-match users |
| Stripe Usage Reporting | API/Worker | R-002, R-006 | 2 | QA | End-to-end sync to Stripe API and atomic Redis counter clear |
| Google/MS OAuth Flow | API | R-003 | 2 | QA | Exchange code for tokens, encryption verification |
| Alert Preferences CRUD | API | - | 1 | QA | Validation rules (CPV, Region, Limits) |

**Total P0**: 8 tests, 25 hours

### P1 (High) - Run on PR to main

**Criteria**: Important features + Medium risk (3-4) + Common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ------------- | ---------- | --------- | ---------- | ----- | ------- |
| Calendar Sync Logic | API/Worker | R-005 | 2 | QA | Create/Update/Delete Google and MS events based on diffs |
| Task & Approval Emails | API/Worker | - | 2 | QA | Events from Redis trigger appropriate transactional templates |
| iCal Feed Generation | API | - | 1 | QA | VEVENT parsing, token generation & revocation |
| SendGrid Webhooks | API | - | 1 | QA | Simulate delivered/bounced events updating DB status |
| Frontend Preferences | UI | - | 1 | QA | CRUD operations, 5-limit warning, sliders validation |

**Total P1**: 7 tests, 18 hours

### P2 (Medium) - Run nightly/weekly

**Criteria**: Secondary features + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ------------- | ---------- | --------- | ---------- | ----- | ------- |
| Celery Retry Backoff | Worker | R-005 | 1 | DEV | Mock 429 response from SendGrid, verify 5 retries |
| Trial Expiry Consumer | API/Worker | - | 1 | QA | Triggers 3 days before expiry, correct template |
| Scheduled PDF Report | Worker | - | 1 | QA | Validates PDF generation & email attachment (S09.14) |
| Frontend Calendar Page | UI | R-007 | 1 | QA | Connection state display, disconnect flow, 30s polling |

**Total P2**: 4 tests, 10 hours

### P3 (Low) - Run on-demand

**Criteria**: Nice-to-have + Exploratory + Performance benchmarks

| Requirement | Test Level | Test Count | Owner | Notes |
| ------------- | ---------- | ---------- | ----- | ------- |
| Email Template Rendering | Manual | 1 | QA | Visual verification of SendGrid transactional templates |

**Total P3**: 1 test, 2 hours

---

## Execution Order

### Smoke Tests (<5 min)
**Purpose**: Fast feedback, catch build-breaking issues
- [ ] Worker health check and Redis connection
- [ ] Alert Preferences API basic CRUD

### P0 Tests (<10 min)
**Purpose**: Critical path validation
- [ ] Immediate Alert Dispatch (E2E)
- [ ] Stripe Usage Reporting (Worker)
- [ ] Digest Assemblies (Worker)
- [ ] OAuth Token Exchange (API)

### P1 Tests (<30 min)
**Purpose**: Important feature coverage
- [ ] Calendar Sync Diff Logic (Worker)
- [ ] iCal Feed format and auth (API)
- [ ] Task/Approval consumers (Worker)
- [ ] Webhook state updates (API)

### P2/P3 Tests (<60 min)
**Purpose**: Full regression coverage
- [ ] Celery retry policies (Worker)
- [ ] Scheduled Report generation (Worker)
- [ ] Frontend UI behaviors (UI)

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
| --------- | ----------------- | ---------- | ----------------- | ----------------------- |
| P0 | 8 | 3.0 | 25 | Involves Redis mocking, Stripe mocking, and encrypted DB fields |
| P1 | 7 | 2.5 | 18 | Requires third-party OAuth and SendGrid simulator |
| P2 | 4 | 2.5 | 10 | Includes Playwright UI tests and PDF verification |
| P3 | 1 | 2.0 | 2 | Manual visual checks |
| **Total** | **20** | **-** | **55** | **~7 days (1 QA)** |

### Prerequisites

**Test Data:**
- User profiles with diverse alert preferences
- Opportunity payloads (matching and non-matching)
- Redis usage counters mimicking billing cycles

**Tooling:**
- `pytest` with `pytest-celery` for worker mocking
- Stripe Test Mode / `stripe-mock` container
- WireMock or similar for SendGrid/Google/MS Graph API simulation

**Environment:**
- Redis instance for streams and Celery broker
- PostgreSQL with `notification` and `client` schemas
- Celery worker and beat processes running

---

## Quality Gate Criteria

### Pass/Fail Thresholds
- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: ≥95% (waivers required for failures)
- **P2/P3 pass rate**: ≥90% (informational)
- **High-risk mitigations**: 100% complete or approved waivers

### Non-Negotiable Requirements
- [ ] All P0 tests pass
- [ ] Stripe Usage reporting must strictly not lose or double-count usage (R-002)
- [ ] No OAuth token exposure in logs (R-003)

---

## Mitigation Plans

### R-001: Redis stream event loss
**Mitigation Strategy:** Consumer groups + Manual ACK. Dead Letter Queue for failing events.
**Owner:** Backend
**Timeline:** S09.04
**Verification:** Force a Celery task exception, verify event is not lost and is re-attempted.

### R-002: Stripe Usage over/under-reporting
**Mitigation Strategy:** `action='set'` (not increment) using Stripe API; atomic GET+DEL pipeline in Redis.
**Owner:** Backend
**Timeline:** S09.11
**Verification:** Concurrency tests executing the sync while Redis usage is actively incremented.

### R-003: OAuth token exposure & invalidation
**Mitigation Strategy:** Fernet encryption at rest; strict DB scope access.
**Owner:** Security
**Timeline:** S09.08
**Verification:** Ensure DB records are unreadable without the key, and logs never print tokens.

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
| ----------------- | -------------- | ------------------------------- |
| **Event Bus (E01)** | Producer | Ensure opportunity/task/approval schema changes don't break consumers |
| **Client API** | Shared schema | Client API endpoints interact correctly with shared `notification` and `client` tables |
| **Billing (E08)** | Consumer | Stripe sync accurately reflects billing tiers |
| **Analytics (E12)** | Data Source | Scheduled PDF uses analytics queries properly |

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)