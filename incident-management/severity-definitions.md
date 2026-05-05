# EU Solicit — Incident Severity Definitions

**Version**: 1.0 | **Story**: PE.06 (21-6) | **Last updated**: 2026-05-05

> **Source of truth**: This file is the canonical reference for severity levels,
> response-time SLAs, and SLA-scope carve-outs. All runbooks reference this file.
> Alertmanager severity labels in PE.05 `alerting-rules.yaml` are the bridge between
> automated detection and these human response definitions.

---

## Severity Levels

### SEV-1 — Critical / Production Outage

**Definition**: Any condition where:
- Any customer cannot complete a **core flow** (opportunity discovery, proposal generation, billing checkout, account access), **OR**
- A **data-integrity breach** is confirmed or strongly suspected, **OR**
- A **security incident** is confirmed or strongly suspected (unauthorised access, credential exposure, injection attempt), **OR**
- Error-budget burn-rate exhaustion within **24 hours** at current rate (fast-burn: >14.4× on 1h + 5m windows).

**Examples**:
- All proposal submissions failing with 5xx errors.
- Database write path completely blocked (disk full, failover in-progress reconnect).
- JWT signing key exposed.
- Error-budget depleted by >50% in <1 hour.

**Response SLA**:
| Step | SLA |
|------|-----|
| PagerDuty ack | 5 minutes |
| Incident Commander (IC) declared | 15 minutes |
| Initial customer communication | 1 hour (via `status-page-comms-templates.md` SEV-1 Initial) |
| Resolution target | As fast as possible (no fixed SLA; SLO budget being burned) |
| Post-mortem scheduled | Within 5 business days |

**Alertmanager severity label**: `severity=page` → routes to PagerDuty (on-call primary paged immediately).

---

### SEV-2 — Degraded Service

**Definition**: Any condition where:
- REST p95 latency exceeds **NFR-2 (0.2s)** sustained for 15+ minutes (`HighLatencyP95` alert), **OR**
- A **partial-tenant impact** — a subset of customers cannot complete a core flow, **OR**
- **Feature-level degradation** — a non-critical feature is broken (e.g., ClamAV scan delay causing upload queue; OAuth provider outage causing new-login failures while existing sessions are unaffected), **OR**
- Error-budget slow-burn rate >6× on 6h + 30m windows (`SustainedErrorBudgetBurnRate` alert).

**Examples**:
- API p95 latency at 350ms (above 200ms NFR-2 threshold).
- ClamAV outage (file uploads queued, not rejected).
- Google OAuth outage (email-password login still working).
- Redis evictions causing cache-miss cascade (latency elevated, not fully broken).

**Response SLA**:
| Step | SLA |
|------|-----|
| Slack `#platform-alerts` notification ack | 15 minutes |
| IC declared (if multi-service impact) | 30 minutes |
| Customer communication (if customer-visible) | 4 hours |
| Resolution target | Within 1 hour of detection |
| Post-mortem scheduled | Within 5 business days (required for sustained SEV-2) |

**Alertmanager severity label**: `severity=ticket` → routes to Slack `#platform-alerts` (no PagerDuty page).

---

### SEV-3 — Warning / Early Signal

**Definition**:
- Slow-burn signals not yet impacting customers (error-budget burn >1× but <6×).
- Non-customer-impacting infrastructure issues (Redis eviction rate starting, RDS replica lag <30s).
- Health-check flaps with no customer impact.
- Informational events for operator awareness.

**Examples**:
- Redis eviction rate at 1/min (low; not yet impacting performance).
- Disk usage at 70% on RDS (early warning; auto-scaling not yet triggered).
- A single pod restart (non-crashloop).

**Response SLA**:
| Step | SLA |
|------|-----|
| Review | Next business day |
| Customer communication | None required |
| Post-mortem | Optional (only if pattern repeats) |

**Alertmanager severity label**: `severity=info` → logged only (Alertmanager discards; no routing to PagerDuty or Slack).

---

## SLA-Scope Table

The platform 99.9% SLA covers **in-scope** infrastructure and services.
**EXEMPT** incidents are caused by upstream vendors and do not count against the platform error budget.

| Incident Type | SLA-Scope | Rationale |
|---------------|-----------|-----------|
| PostgreSQL Multi-AZ failover | **in-scope** | Platform-owned infrastructure |
| Redis ElastiCache failover | **in-scope** | Platform-owned infrastructure |
| nginx-ingress controller restart | **in-scope** | Platform-owned component |
| Application deploy regression | **in-scope** | Platform-controlled change |
| Error-budget burn (all in-scope services) | **in-scope** | Platform SLO definition |
| Node drain / Kubernetes scheduling | **in-scope** | Platform-controlled infrastructure |
| Full-disk on PostgreSQL RDS | **in-scope** | Platform-managed database |
| **KraftData outage** | **EXEMPT** | Upstream vendor; Epic 4 isolation precedent (architecture.md line 762) |
| **Stripe API outage** | **EXEMPT** | Upstream payment vendor; Epic 4 isolation precedent applied to Stripe |
| **ClamAV scanner outage** | **EXEMPT** | Security scanner; degraded scan = degraded feature, not platform outage |
| **Google OAuth outage** | **EXEMPT** | Upstream identity provider; active sessions unaffected (JWT 24h grace) |
| **Microsoft OAuth outage** | **EXEMPT** | Upstream identity provider; Epic 9 calendar-sync degraded (best-effort) |

> **EXEMPT rule**: The platform team owns the *response* to EXEMPT incidents (runbooks, status communication, webhook replay) but the upstream vendor's availability itself does not count against the platform 99.9% SLA.

> **Operator obligation**: EXEMPT incidents MUST still be acknowledged and communicated per the SEV-2 process. The SLA carve-out applies to the error budget, not to the human response requirement.

---

## Severity-to-Alertmanager Label Mapping

**Canonical bridge**: this table maps severity labels in `alerting-rules.yaml` (PE.05) to human response actions.

| Alertmanager `severity` label | Human severity | Routing | Human response |
|------------------------------|----------------|---------|----------------|
| `page` | SEV-1 | PagerDuty (on-call primary) → escalation policy | Immediate ack ≤5 min |
| `ticket` | SEV-2 | Slack `#platform-alerts` | Ack ≤15 min (business hours) |
| `info` | SEV-3 | Alertmanager discard (logged only) | Next business day |

**Anti-pattern fence**: future alerting rules MUST set the correct `severity` label per this table. A missing or incorrect `severity` label means alerts route to the wrong destination — `page` alerts must only fire for SEV-1 conditions (no false paging) and `info` alerts must never be used for conditions requiring same-day response.

---

## Response-Time Clock Start

The response-time SLA clock starts at:
1. **PagerDuty page received** — for `severity=page` alerts (SEV-1).
2. **Slack `#platform-alerts` notification posted** — for `severity=ticket` alerts (SEV-2).
3. **Customer report received** — if the incident is customer-reported before alerting fires.

**PagerDuty ack** = the on-call engineer presses "Acknowledge" in PagerDuty mobile app or web. This is the first measurable response action. Time from page to ack is the KPI for PE.06's 5-minute SEV-1 target.
