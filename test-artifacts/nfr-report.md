---
artifact: nfr-report
epic: 23
epic_title: Operational Close-Out
assessed_by: Murat (bmad-testarch-nfr, autopilot)
assessed_on: 2026-05-12
project_root: /home/debian/Projects/eusolicit
config_source: _bmad/bmm/config.yaml
inputDocuments:
  - eusolicit-docs/planning-artifacts/epics/E23-operational-close-out.md
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/EU_Solicit_Solution_Architecture_v5.md
  - eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md
  - eusolicit-docs/project-context.md
  - eusolicit-docs/implementation-artifacts/sprint-status.yaml
  - eusolicit-docs/implementation-artifacts/epic-21-retro-2026-05-05.md (referenced)
  - eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md (referenced)
stepsCompleted:
  - step-01-load-context
  - step-02-define-thresholds
  - step-03-gather-evidence
  - step-04-evaluate-and-score
  - step-05-generate-report
lastStep: step-05-generate-report
lastSaved: 2026-05-12
overall_gate: CONCERNS
critical_failures: 0
---

# Epic 23 — NFR Assessment Report

## 1. Executive Summary

**Overall NFR Gate:** **CONCERNS** (no critical failures; release-gated on verifiable evidence still in flight)

Epic 23 ("Operational Close-Out") is **the** launch-readiness NFR epic — its entire purpose is to discharge the operational/non-functional gates that the engineering epics deliberately deferred. Three of the five stories are *operator-paced execution* (chaos drill, paging rotation, public-SLA soak), and the remaining two close long-standing carry-forwards (Stripe two-layer resilience, TEA backlog).

Because the epic is intentionally scoped against a **"beta, best-effort availability"** posture (per `onprem-pivot-decision-2026-05-11.md` + ADR-010) with **RTO ≤ 4h / RPO ≤ 24h** and **no numeric SLA**, the NFR thresholds applied here are the beta-posture thresholds, not the originally planned 99.9 % production SLOs (those are explicitly deferred to a Phase-2 HA migration per AP23-D2).

No CRITICAL failures detected — autopilot does **not** emit `HALT`. Four CONCERN items are listed below; each is closeable inside the existing epic scope.

## 2. NFR Thresholds (Sourced)

| NFR | Threshold | Source |
|---|---|---|
| Availability posture | "Service is in beta. Best-effort availability." (no numeric SLA pre-soak) | E23 AC; `onprem-pivot-decision-2026-05-11.md` |
| RTO | ≤ 4 hours | E23 AC; ADR-010 |
| RPO | ≤ 24 hours | E23 AC; ADR-010 |
| Soak gate before any uptime claim | 2 weeks post-launch, no posture-revising incidents | E23 AC; AP23-D2 |
| Paging delivery | First synthetic page reaches email + Telegram channels end-to-end | E23 AC `pe-06` |
| Chaos drill scope | container kill/recover, fill-disk, Postgres crash recovery, Redis AOF replay | E23 `pe-04` |
| External-API resilience | Two-layer (circuit breaker OUTER, retry INNER); breaker only counts 5xx/network/timeout/connection (not 4xx) | project-context Rule 47; epic-5 OBS-001; drift-recovery AC4 |
| Webhook signature security | `hmac.compare_digest`; raw body before JSON parse; timing diff <1 ms (verified by unit test) | project-context Rule 48 |
| Observability for billing | 5 Prometheus metric families wired + Grafana dashboard JSON committed under `infra/observability/grafana/dashboards/` | drift-recovery AC5 |
| Test-coverage minimum | 80 % coverage gate, per-service | `eusolicit-app/Makefile`, `make coverage` |
| Cross-tenant negative tests | Required on any endpoint exposing tenant-scoped state | project delivery instructions |

## 3. Evidence Inventory

| Source | Status | Notes |
|---|---|---|
| Stripe two-layer resilience reference wiring | PARTIAL | `services/client-api/src/client_api/services/billing_service.py:~149` (customer.create) landed; **8 remaining call sites + vies_service.py pending** (sprint-status 2026-05-12) |
| `register_billing_metrics()` | PARTIAL | Landed in `billing_metrics.py` + wired in `main.py`; **4 metric wirings still pending** (webhook duration, usage-sync drift, active subs gauge, trial→paid increment) |
| Grafana dashboard JSON for billing/circuit-breaker | MISSING | Not yet committed under `infra/observability/grafana/dashboards/` |
| Chaos drill execution evidence | NOT YET RUN | Post-mortem skeleton exists (`post-mortems/2026-MM-DD-pe-04-chaos-drill.md`); §Drill Results empty — RTO/RPO claims **unverified empirically** |
| Alertmanager email + Telegram receivers | CODE LANDED, UNVERIFIED | Landed under E22 `onprem-03`; first synthetic page not yet observed end-to-end |
| Trust Center beta disclosure | CODE LANDED | 2 posture cards in `frontend/apps/client/content/trust/compliance-posture.mdx`; 2-week soak clock starts at E22 launch-live (not yet started) |
| TEA-review backlog (Epic 8/9) | OPEN | 6 PRs open; coverage-gap docs in `test_artifacts/` not yet updated |
| Existing resilience reference | PRESENT | `ai-gateway.call_kraftdata()` confirms two-layer pattern is in-codebase and tested (Rule 47 reference impl) |
| Webhook HMAC pattern | ENFORCED | Rule 48; carry-forward applies to Stripe webhook handler under this epic |
| k6 load baselines | PRESENT | Re-homed into PE.01 / Story 21-1 (per sprint-status); not re-run for E23 scope (no new endpoints) |

## 4. NFR Evaluation

### 4.1 Performance — **PASS (with note)**

- **Scope check:** E23 introduces no new user-facing endpoints, no new DB hot paths, no new background workers. No perf regression surface.
- **Circuit-breaker latency cost:** Two-layer wrapping adds bounded overhead (~µs-level decorator dispatch + breaker state check). Inner retry uses exponential backoff (existing reference impl) — worst-case tail latency on a downed Stripe is bounded by the breaker open-threshold rather than retry exhaustion, which is a **latency improvement** over status quo.
- **Note:** No new k6 run required for E23 since no new endpoints. PE.01 baselines remain the authoritative perf evidence.

### 4.2 Security — **PASS**

- **Rule 47** (two-layer external resilience) **and Rule 48** (HMAC `compare_digest`, raw body before JSON) carry forward and are restated in drift-recovery story §Anti-patterns.
- **Composition order discipline:** Story explicitly mandates `circuit_breaker(retry(call))` — preventing retry from burning the breaker's failure budget on a single request. This is a defensive-coding posture, not a vulnerability surface.
- **Webhook handler:** The pending AC5 `webhook_processing_duration_seconds` wiring **must not** alter the existing HMAC path; flagged for code review.
- **4xx exclusion from breaker counter** (epic-5 OBS-001) is a security-adjacent correctness rule — it prevents an attacker who can elicit 401/403 from forcing the breaker open as a DoS vector. Story carries this anti-pattern forward; verify in code review.
- **Cross-tenant negative tests:** Any new endpoint exposing `billing_metrics` state must have a per-tenant negative test (delivery instructions). Currently no such endpoint is proposed (metrics live on the existing `/metrics` Prometheus scrape, scoped at infra layer) — flagged as a check-in-review item, not a gap.

### 4.3 Reliability — **CONCERNS**

This is the dominant NFR axis for E23. Status:

| Capability | State | Risk |
|---|---|---|
| Stripe circuit-breaker (8 remaining call sites) | PARTIAL | Until all sites migrated, a Stripe partial outage still has 8 retry-storm vectors → tail-latency + thread/worker exhaustion exposure on `client-api`. **Highest active reliability risk in the epic.** |
| Billing metrics (4 pending wirings + dashboard JSON) | PARTIAL | Operator cannot observe breaker state in Grafana → MTTD inflated on a real Stripe degradation event |
| Chaos drill (real, not synthetic) | NOT RUN | RTO ≤ 4h is a **claim**, not a measurement. Cannot be released to Trust Center until drill measures actual recovery times |
| Paging end-to-end verification | NOT RUN | Alert-to-page path unproven; first-page latency unknown |
| 2-week soak window | NOT STARTED | Gated on E22 launch-live; cannot be parallelised |

**Reliability gate:** CONCERNS. None of these is a *critical* failure (the epic is in-progress and these are the in-flight deliverables), but **none of the AC items are currently closeable** with evidence on hand. The closure path is clear and the epic owns it.

### 4.4 Maintainability — **PASS**

- **Operational artefacts** are template-driven: blameless post-mortem template, runbook §Drill Results / §Verification sections, traceability matrix updates per inj-03 ticket — structural enforcement (E21 retro pattern E21-P05) discourages ad-hoc drift.
- **Two-layer resilience pattern** has a canonical reference impl (`call_kraftdata`) — migrating 8 Stripe call sites is mechanical, low-variance work.
- **Idempotent `register_billing_metrics()`** is already verified against test app rebuilds (sprint-status 2026-05-11) — preserves existing test-isolation discipline.
- **PagerDuty rescope** removes a non-essential vendor dependency, simplifying on-call surface (AP23-D3 explicitly forbids reintroduction).

### 4.5 Scalability — **PASS (not in scope, no regression)**

- Single-host posture is intentional and ratified (ADR-010); horizontal scale is out-of-scope for E23 and deferred to Phase-2.
- No new fan-out or back-pressure surface introduced.

## 5. Findings (Prioritised)

| ID | Severity | NFR | Finding | Action |
|---|---|---|---|---|
| E23-NFR-01 | Medium | Reliability | 8 Stripe call sites + `vies_service.py` not yet migrated to `resilient_stripe_call` → retry-storm vector remains | Complete migration as part of `drift-recovery-story` dev pass; verify in code review |
| E23-NFR-02 | Medium | Reliability/Observability | 4 billing metric wirings + Grafana dashboard JSON pending → no operator visibility into breaker state | Land wirings + commit dashboard JSON under `infra/observability/grafana/dashboards/`; provision via PE.05 / Story 21-5 pattern (do NOT invent new provisioning path) |
| E23-NFR-03 | Medium | Reliability | Chaos drill not yet executed → RTO ≤ 4h is unmeasured | Run real drill (not synthetic) per `pe-04`; record measured recovery times in `runbooks/chaos-drill-single-host.md` §Drill Results; complete post-mortem |
| E23-NFR-04 | Low | Reliability | Paging end-to-end unverified | Trigger synthetic page; capture email + Telegram screenshots in story §Verification |
| E23-NFR-05 | Low | Quality/Coverage | 6 TEA-review tickets open; traceability matrix not yet updated | Close per `inj-03-tea-review-backlog-epic8-epic9`; one PR per ticket, each closing a coverage gap with passing tests |
| E23-NFR-06 | Info | Reliability | 2-week soak gate cannot begin until E22 launch-live | Calendar-driven; tracked outside this assessment |
| E23-NFR-07 | Info | Security | New `webhook_processing_duration_seconds` wiring must not alter HMAC verification path | Verify in code review (Rule 48 unchanged) |
| E23-NFR-08 | Info | Security | If billing_metrics state ever becomes tenant-scoped, per-tenant negative test required | Currently scrape-scoped at infra layer; flag if scope changes |

## 6. Anti-Patterns to Enforce During Execution

(Imported from E23 §Anti-patterns + project-context + sprint-status PM proposal)

- **AP23-D1** Treating E23 as engineering — most work is operator execution. Don't over-engineer.
- **AP23-D2** Publishing any numeric SLA before the 2-week soak.
- **AP23-D3** Reverting to PagerDuty.
- **AP23-D4** Skipping the chaos drill post-mortem.
- **Composition order**: `circuit_breaker` OUTER, `retry` INNER (otherwise retry burns breaker's failure budget).
- **OBS-001**: 4xx must not increment the breaker failure counter.
- **Idempotency**: `register_billing_metrics()` must remain re-registration-safe.
- **HMAC**: `hmac.compare_digest`, raw body before JSON parse (Rule 48).
- **Explicit `httpx` timeout** on every outbound call.
- **Grafana provisioning path**: reuse PE.05 / Story 21-5 pattern; do not invent new provisioning surface.
- **Cross-tenant negative test** on any new tenant-scoped endpoint.

## 7. Gate Decision

**Overall NFR Gate: CONCERNS**

- **Critical failures:** 0 → **HALT not emitted**.
- **Medium concerns:** 3 (E23-NFR-01, E23-NFR-02, E23-NFR-03) — all owned inside the epic and have clear closure criteria in the existing story files.
- **Low concerns:** 2 (E23-NFR-04, E23-NFR-05).
- **Release-gating items:** All 6 ACs of E23 itself; specifically AC3 (Trust Center disclosure + 2-week soak), AC4 (Stripe circuit-breakers complete), and AC5 (TEA backlog cleared) must each transition to done with two-gate close before launch sign-off.

**Recommended close order:**
1. drift-recovery AC4 + AC5 (engineering — unblock observability before drill)
2. pe-06 paging verification (need pages to observe drill)
3. pe-04 chaos drill execution (RTO/RPO measurement)
4. inj-03 TEA backlog (parallelisable; no ordering constraint)
5. public-sla-soak (calendar gate; starts at E22 launch-live)

## 8. Reassessment Triggers

Re-run this NFR assessment if any of the following occur:
- A real incident during the 2-week soak that requires posture revision.
- Chaos drill measures RTO > 4h or RPO > 24h → posture must downgrade further or remediation required before launch claim.
- Any reintroduction of PagerDuty or any numeric-SLA claim.
- A new endpoint exposing billing/breaker state with tenant scope.
- Phase-2 HA migration is greenlit (thresholds change to 99.9 % SLO regime).

---

*Assessment produced by `bmad-testarch-nfr` skill in autopilot. No HALT condition met.*
