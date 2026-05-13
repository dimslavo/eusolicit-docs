# NFR Assessment Report - Epic 22: On-Prem Launch

## 1. Introduction

This document assesses the non-functional requirements (NFRs) for Epic 22: On-Prem Launch. This epic represents a strategic pivot (ADR-010) from a high-availability AWS architecture to a single-host Docker deployment on `www1.endigitalx.com`.

The primary focus of this assessment is on **Reliability**, **Maintainability**, and **Security** within the context of this simplified "best-effort availability" posture. While the PRD and Solution Architecture define broader NFRs (e.g., 99.5% uptime, <200ms latency), Epic 22 explicitly defers these, prioritizing a robust and recoverable single-host launch.

## 2. Loaded Context & Artifacts

### 2.1. Core Documents

*   **Epic Definition**: `eusolicit-docs/planning-artifacts/epics/E22-onprem-launch.md`
*   **Product Requirements**: `eusolicit-docs/EU_Solicit_PRD_v2.md`
*   **Solution Architecture**: `eusolicit-docs/EU_Solicit_Solution_Architecture_v5.md`
*   **Architecture Decisions (ADR-010)**: `eusolicit-docs/planning-artifacts/architecture.md`
*   **Project Context**: `eusolicit-docs/project-context.md`
*   **Epic 22 Retrospective**: `eusolicit-docs/implementation-artifacts/epic-22-retro-2026-05-12.md`

### 2.2. Knowledge Base Fragments

*   `adr-quality-readiness-checklist.md`
*   `ci-burn-in.md`
*   `test-quality.md`
*   `playwright-config.md`
*   `error-handling.md`
*   `playwright-cli.md`

### 2.3. Key Findings from Context Loading

*   **NFR Scope Shift**: Epic 22's primary NFRs are RTO <= 4h and RPO <= 24h. Performance and high-availability targets from the original PRD are explicitly out of scope for this launch.
*   **Implementation Status**: All 6 stories for Epic 22 are in a `ready-for-dev` state. Code has been seeded but not formally developed, tested, or reviewed.
*   **Blocker**: The target host `www1.endigitalx.com` has a root partition at ~85% capacity, which is a blocker for deploying the new observability stack.
*   **Testing**: No ATDD checklists or TEA reviews exist for Epic 22 stories yet. The epic's success is heavily dependent on the successful execution of manual operational drills (backup/restore, failover).

## 3. NFR Categories & Thresholds

Based on the ADR Quality Readiness Checklist and the specific context of Epic 22 (ADR-010), the following NFRs will be assessed.

| # | Category | Threshold for Epic 22 | Source / Justification |
|---|---|---|---|
| 1 | **Disaster Recovery** | **RTO <= 4 hours, RPO <= 24 hours.** This must be validated by a successful manual backup and restore drill (`onprem-01`). | `E22-onprem-launch.md`, ADR-010. This is the core NFR for this epic. |
| 2 | **Monitorability** | Successful deployment of Grafana/Prometheus/Loki stack. Functional alerting via Telegram for critical failures (e.g., container down, disk space >90%). | `E22-onprem-launch.md` (stories `onprem-04`, `onprem-06`). The disk space blocker identified in the retro directly threatens this. |
| 3 | **Deployability** | Repeatable, successful deployment of all services to `www1.endigitalx.com` via Docker. | The fundamental goal of Epic 22. |
| 4 | **Security** | The existing security posture is maintained (JWT, RBAC, encrypted secrets). Admin endpoints remain VPN/IP-restricted. No new vulnerabilities introduced. | `EU_Solicit_Solution_Architecture_v5.md`. The on-prem deployment should not weaken the established security model. |
| 5 | **Scalability & Availability** | **Best-effort availability** on a single host. No public SLA. No horizontal scaling. | ADR-010. This is an explicit deferral of the PRD's original NFRs. The system should remain stable under expected load. |
| 6 | **Testability & Automation** | Successful, documented execution of all manual operational drills defined in Epic 22 stories (backup/restore, monitoring alert test). | `E22-onprem-launch.md`. Given the lack of automated tests for this epic, manual drill success is the primary validation. |
| 7 | **QoS / QoE (Performance)** | **Functional performance**. No strict latency targets (<200ms p95 is deferred). The system must be usable and not exhibit significant degradation. | ADR-010. Performance is secondary to stability for this launch. |
| 8 | **Test Data Strategy** | **UNKNOWN**. The operational drills will depend on having production-like data on the single-host environment to be meaningful. This is a potential concern. | Inferred from the need for realistic DR drills. |

## 4. Evidence Gathering & Gaps

This section details the evidence found for each NFR category and identifies the significant gaps that represent risks to the Epic 22 launch.

| # | Category | Evidence Gathered | Evidence Gaps & Concerns |
|---|---|---|---|
| 1 | **Disaster Recovery** | Story `onprem-01` defines a backup/restore drill. The `docker-compose.prod.yml` includes volume mounts for `pgdata` and `redisdata`, and configurations for WAL archiving and RDB/AOF persistence, which are prerequisites for recovery. | **CONCERN (CRITICAL):** The DR drill has not been performed. The RTO/RPO targets are entirely theoretical until a successful, timed drill is completed and documented. |
| 2 | **Monitorability** | Stories `onprem-04` and `onprem-06` define the work. `docker-compose.observability.yml` exists, showing intent to use Grafana/Prometheus/Loki. | **CONCERN (CRITICAL):** The work is blocked by the host's disk space issue. There is **zero evidence** of a functional monitoring or alerting system for the on-prem environment. |
| 3 | **Deployability** | The project is fully containerized. `docker-compose.yml` and `docker-compose.prod.yml` provide a clear deployment definition for local and production environments respectively. Each service has a `Dockerfile`. | **CONCERN (MEDIUM):** While the artifacts exist, there is no evidence of a successful, repeatable deployment to the actual `www1.endigitalx.com` host. The process appears to be manual, which introduces risk. |
| 4 | **Security** | Code analysis confirms the implementation of the specified security model. `grep` results show widespread use of `Depends(get_current_user)` for route protection in the `client-api`, and the inclusion of `PyJWT` and `authlib` for token handling and OAuth. | **CONCERN (LOW):** The security posture of the host machine (`www1.endigitalx.com`) itself is undocumented. No evidence of a recent security scan or hardening process for the host. |
| 5 | **Scalability & Availability** | ADR-010 explicitly defers these NFRs, setting the expectation for "best-effort availability" on a single host. This is an accepted limitation. | **GAP:** No evidence of even basic load testing to understand the performance ceiling of the single host. The system could fail under a load that is considered "normal" by users. |
| 6 | **Testability & Automation** | The epic definition relies on manual operational drills as the primary form of testing. | **CONCERN (HIGH):** There is a complete lack of automated tests for the on-prem deployment configuration. A manual process is not reliably repeatable and can drift over time. This makes future deployments and patches risky. |
| 7 | **QoS / QoE (Performance)** | ADR-010 defers performance targets. | **CONCERN (HIGH):** There is no performance baseline for the on-prem host. The user experience is a complete unknown. "Functional performance" is not a measurable threshold. |
| 8 | **Test Data Strategy** | None. | **CONCERN (HIGH):** A valid DR drill requires a production-like dataset. There is no evidence of a plan to create, sanitize, and load this data into the on-prem environment for the drills. This invalidates the core NFR test for this epic. |


## 5. Next Steps

The next step is to evaluate and score the findings. Given the significant evidence gaps and critical concerns, the epic is at high risk.
