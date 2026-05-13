# E23: Operational Close-Out

**Sprint**: 2026-05-15 → 2026-05-29 (operator-paced, parallel to E22 tail) | **Points**: low (operator execution, not engineering) | **Dependencies**: E22 onprem-03 (alertmanager); E22 launch live for `public-sla-soak` | **Milestone**: launch readiness gate cleared; long-standing carry-forwards closed

**Source**: `eusolicit-docs/implementation-artifacts/epic-21-retro-2026-05-05.md` (operator deferrals D-1, D-3, calendar gate); `eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md` (`drift-recovery`, `inj-03` carry-forwards).

## Goal

Close out the operator-execution work that doesn't fit any prior epic's engineering scope but must complete before launch sign-off. Five stories: three from E21 retro (chaos drill execution, paging provisioning, SLA-disclosure soak gate), two long-standing E13 retro carry-forwards (Stripe circuit-breakers + TEA-review backlog). All five are at `ready-for-dev` with partial code landed; the work shape is operator execution + verification, not greenfield engineering.

## Acceptance Criteria

- [ ] First chaos drill executed against the single-host topology; post-mortem authored using the blameless template at `eusolicit-docs/incident-management/post-mortem-template.md`; runbooks revised based on drill findings
- [ ] Email + Telegram-bot paging operational via Alertmanager (replacing PagerDuty); rotation calendar documented; first synthetic page verified end-to-end
- [ ] Trust Center disclosure live with "Service is in beta. Best-effort availability." posture card + RTO ≤ 4h / RPO ≤ 24h commitment; 2-week post-launch soak observed before any uptime claim
- [ ] Stripe circuit-breakers added (AC4 of `drift-recovery-story`); billing metrics dashboard live (AC5 of `drift-recovery-story`); circuit-breaker state observable in Grafana
- [ ] 6 TEA-review tickets cleared (Epic 8 + Epic 9 backlog from `inj-03`); coverage gaps documented in `test_artifacts/`
- [ ] All 5 stories transition `ready-for-dev` → `done` with two-gate close (`bmad-code-review` Approve verdict + operator-completion signal in each story's runbook §Results section)

## Stories

### pe-04-chaos-drill-execution (operator-paced, ~1 drill window)

Execute the first single-host chaos drill against www1: kill-and-recover container; fill-disk drill; Postgres crash recovery; Redis AOF replay. Post-mortem the drill using the blameless template. Revise runbooks based on findings. Source story: `implementation-artifacts/pe-04-chaos-drill-execution.md` (rescoped from AWS-EKS chaos drill to single-host per ADR-010). Skeleton at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` already exists.

### pe-06-pagerduty-rotation-provisioning (rescoped: email + Telegram, ~½ day operator)

Originally PagerDuty provisioning; rescoped 2026-05-11 to: operator activates email + Telegram-bot paging via the `alertmanager.yaml` receivers that landed in code under onprem-03. Verifies first synthetic page reaches both channels. Documents the single-engineer rotation as a calendar reminder, not a platform. Source story: `implementation-artifacts/pe-06-pagerduty-rotation-provisioning.md`.

### public-sla-announcement-soak-gate (operator calendar gate, 2-week soak post-launch)

Originally gated on 2-week soak before public 99.9% SLA announcement; rescoped 2026-05-11 to: publish "Service is in beta. Best-effort availability." disclosure on Trust Center with RTO/RPO commitment. **No numeric SLA.** Story file landed 2 posture cards in `frontend/apps/client/content/trust/compliance-posture.mdx`. Closure signal: 2 weeks post-launch with disclosure live and no incidents requiring posture revision. Source story: `implementation-artifacts/public-sla-announcement-soak-gate.md`.

### drift-recovery-story (E13 carry-forward, ~1-2 days)

Long-standing E13 carry-forward. AC1-3 already done; AC4 (Stripe circuit-breakers) and AC5 (billing metrics dashboard) outstanding. AC4 lands two-layer resilience pattern (project-context Rule 47) on Stripe SDK calls in `client-api/services/billing`. AC5 surfaces circuit-breaker state in a new Grafana dashboard. Source story: `implementation-artifacts/drift-recovery-story.md`.

### inj-03-tea-review-backlog-epic8-epic9 (E13 carry-forward, ~1 day per ticket)

Long-standing E13 carry-forward. 6 TEA-review tickets covering test-coverage gaps in Epic 8 (Subscription Billing) and Epic 9 (Notifications/Calendar). Each ticket: targeted test addition + traceability matrix update at `test_artifacts/`. Source story: `implementation-artifacts/inj-03-tea-review-backlog-epic8-epic9.md`.

## Tests

- For pe-04: drill must be real (not synthetic); §Drill Results section in `runbooks/chaos-drill-single-host.md` records measured recovery times.
- For pe-06: first synthetic page must reach email + Telegram inboxes; screenshots captured in §Verification section.
- For drift-recovery AC4/AC5: pytest unit + integration tests on circuit-breaker behaviour; Grafana dashboard JSON committed and provisioned.
- For inj-03: 6 PRs (one per TEA ticket), each closing a coverage gap with passing tests.

## Anti-patterns / known deviations

- **AP23-D1** Treating this epic as "engineering" — most work is operator execution. Don't over-engineer.
- **AP23-D2** Publishing any numeric SLA before the 2-week soak — explicit calendar gate.
- **AP23-D3** Reverting to PagerDuty — pivot decision is ratified; do not re-introduce.
- **AP23-D4** Skipping the chaos drill post-mortem — blameless template enforces structure.

## Cross-epic dependencies

- **E22 onprem-03**: pe-06 consumes alertmanager email/Telegram receivers landed under onprem-03.
- **E22 onprem-03 + onprem-04**: pe-04 chaos drill consumes monitoring + alerting to observe drill effects.
- **E22 launch live**: public-sla-soak depends on launch being live.

## Out-of-scope

- New SLA commitments (deferred to Phase-2 HA migration if pursued).
- ISO 27001 / SOC 2 audit work (separate parallel programme).
- Re-introducing PagerDuty (explicitly ratified out per ADR-010).
