# INJ-03: TEA Review Backlog (E08 + E09)
**Status:** done
**Epic:** 13
**Priority:** High

<!-- 2026-05-13 — bmad-tea audit-style retrospective dispatch (🧪 Murat persona, claude-opus-4-7[1m] autopilot via 'proceed with all' carry-through). VERDICT: ALL 6 STORIES PASS at ≥80/100. Average score 85.7. Six review artefacts authored at eusolicit-docs/implementation-artifacts/tea-reviews/. AC1-AC6 closed on score; AC7 closed on artefact presence. AP18-C2 atomic: story Status `ready-for-dev → done` + sprint-status `inj-03-tea-review-backlog-epic8-epic9: ready-for-dev → done` in same commit. Closes the 18th-consecutive-epic terminal TEA escalation flagged in epic-23-retro-2026-05-12.md § E23-AP01. -->

As a quality gate enforcer,
I want TEA scored reviews (≥80/100) completed for 6 high-risk stories,
so that the 7-epic TEA gap is partially closed and the billing + notification
paths have independent quality validation.

## Acceptance Criteria:
1. TEA review for S08.04 (stripe-webhook) — score ≥80/100, findings documented
2. TEA review for S08.08 (usage-metering-redis-counters) — score ≥80/100
3. TEA review for S08.10 (eu-vat-vies-validation) — score ≥80/100
4. TEA review for S09.04 (alert-matching-immediate-dispatch) — score ≥80/100
5. TEA review for S09.06 (sendgrid-email-delivery) — score ≥80/100
6. TEA review for S09.08 (google-calendar-oauth2-sync) — score ≥80/100
7. All review artifacts saved to eusolicit-docs/implementation-artifacts/tea-reviews/

## Tasks:
- [x] Run bmad-tea on 08-4-stripe-webhook-endpoint-subscription-lifecycle-sync.md → **88/100 PASS**
- [x] Run bmad-tea on 08-8-usage-metering-with-redis-counters-stripe-sync.md → **90/100 PASS** (strongest of batch)
- [x] Run bmad-tea on 08-10-eu-vat-handling-via-stripe-tax-vies-validation.md → **86/100 PASS**
- [x] Run bmad-tea on 09-4-alert-matching-immediate-dispatch.md → **82/100 PASS** (marginal — test-file fragmentation noted)
- [x] Run bmad-tea on 09-6-sendgrid-email-delivery-template-management.md → **83/100 PASS** (template-injection follow-up flagged P0)
- [x] Run bmad-tea on 09-8-google-calendar-oauth2-sync.md → **85/100 PASS**
- [x] Save all review artifacts → `eusolicit-docs/implementation-artifacts/tea-reviews/<story-key>-tea-review-2026-05-13.md` (6 files authored)

## Review Findings — bmad-tea 2026-05-13 (terminal TEA escalation close)

**Average score: 85.7/100. All 6 stories PASS the ≥80 threshold.** Closes the 18th-consecutive-epic TEA gap per E23-AP01.

| Story | Score | Review artefact |
|---|---|---|
| S08.04 stripe-webhook | 88 | `tea-reviews/08-4-stripe-webhook-endpoint-subscription-lifecycle-sync-tea-review-2026-05-13.md` |
| S08.08 usage-metering-redis | 90 | `tea-reviews/08-8-usage-metering-with-redis-counters-stripe-sync-tea-review-2026-05-13.md` |
| S08.10 EU-VAT-VIES | 86 | `tea-reviews/08-10-eu-vat-handling-via-stripe-tax-vies-validation-tea-review-2026-05-13.md` |
| S09.04 alert-matching | 82 | `tea-reviews/09-4-alert-matching-immediate-dispatch-tea-review-2026-05-13.md` |
| S09.06 sendgrid-email | 83 | `tea-reviews/09-6-sendgrid-email-delivery-template-management-tea-review-2026-05-13.md` |
| S09.08 google-calendar | 85 | `tea-reviews/09-8-google-calendar-oauth2-sync-tea-review-2026-05-13.md` |

### High-priority follow-ups surfaced (NOT inj-03 gating — track separately)

1. **S09.06 template-injection guard test (P0)** — emails render user-supplied content (proposal titles, comments). No test asserts Jinja2 autoescape behaviour against `<script>` input. SECURITY surface. Author this regardless of inj-03 close.
2. **S08.10 `vat.sync.pending` consumer-drain integration test (P1)** — score=6 risk PARTIAL on Stripe-circuit-open VAT reconciliation. The enqueue is tested (drift-recovery P2 today); the drain is not. Closes a real tax-compliance gap.
3. **S09.04 malformed-event ACK regression test (P1)** — spec requires "logs error, ACKs to avoid redelivery loop" but the regression test for this AC was not located by name. Verify or author.
4. **S09.08 OAuth scope minimization test (P1)** — least-privilege assertion missing. Affects Google app-verification compliance + user consent screen trust.

### Anti-pattern observations (across the batch)

- **Test-file fragmentation** (S09.04 was the clearest example): two test files cover the same story without cross-reference. A reviewer landing on one file gets a false sparse-coverage signal. Recommend: top-of-file docstring with sibling test-file references for any story whose coverage spans 2+ files.
- **External-dependency contract drift risk** (S08.04 + S08.10): tests mock SDK objects (Stripe Event, VIES response). No contract tests against published API schemas. SDK upgrade can silently break without test signal. Recommend Pact-style contract testing layer per `contract-testing.md` knowledge fragment for the next SDK upgrade window.
- **Test-pyramid balance is healthy** across the batch — appropriate use of unit < integration < worker test boundaries; no over-reliance on E2E for behaviour that belongs at lower levels.
- **Cross-tenant negative tests present in all 6 stories** — project memory rule satisfied across the batch. This is the strongest single quality signal in the inj-03 corpus.

### Closure

AC1-AC6 closed at ≥80/100 each. AC7 closed on artefact presence (6 files at canonical `tea-reviews/` path). AP17-C1 + AP18-C2 atomic close: story Status `done` + sprint-status row flip in same commit.
