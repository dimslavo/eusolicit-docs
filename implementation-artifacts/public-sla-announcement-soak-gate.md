# Story: public-sla-announcement-soak-gate

**Epic:** E23 — Operational Close-Out
**Status:** ready-for-dev
**Story key:** `public-sla-announcement-soak-gate`
**Type:** full-stack (Next.js MDX + i18n + monitoring alert) + cross-functional review
**Sprint:** E23 close-out
**Created:** 2026-05-13 by `bmad-create-story` (📋 John PM dispatch under `proceed with all` autopilot) per sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md §3.1.3 + §4.3 (approved by Deb 2026-05-13)

> **Provenance** — this is a re-create of the public-sla story spec after the 2026-05-13 audit surfaced that the sprint-status row had been injected at `ready-for-dev` without a formal story file at `story_location` (E22-AP01 / E23-AP02 anti-pattern recurrence flagged in the 2026-05-12 retros). Pull-back to `backlog` landed via Edit 1 of the proposal; this `bmad-create-story` dispatch lands the spec and flips to `ready-for-dev`.

## Goal

Complete the public-facing Trust Center disclosure for EU Solicit's post-pivot availability posture. Two MDX posture cards (`availability`, `rtoRpo`) already render in `compliance-posture.mdx` but their `description_key` lookups currently 404 on both English and Bulgarian locales because the i18n keys were never authored (verified 2026-05-13 — `grep "trust.posture.availability\|trust.posture.rtoRpo" frontend/apps/client/messages/{en,bg}.json` returns empty). Story closes when:

1. Both posture cards render correctly with full bg/en i18n parity
2. WCAG 2.1 AA accessibility verified
3. A monitoring alert covers sustained 404s on the Trust Center routes
4. Product + Legal review the disclosure card CONTENT before public commit

**Out of scope (DELIBERATELY split to follow-up story):**
- **7-day Trust Center soak observation** — see §See also for `public-sla-7-day-soak-observation` follow-up. Calendar-paced (≥7 consecutive days on stage with zero P0/P1 incidents touching the SLA disclosure surface) — splitting it keeps this story dispatchable in normal flow without calendar-blocking the close. PM-approved split per sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md §3.1.3 + §4.3.

**Scope explicitly NOT included** (per ADR-010 + project memory `project_onprem_pivot_2026_05_11.md`):
- **99.9% SLA numeric promise** — DEFERRED INDEFINITELY. Numeric uptime promise can be reconsidered after ≥1 quarter of measured uptime data + potential HA-migration epic, IF commercial pressure justifies it. The current public-facing posture is the explicit beta declaration (`availability` card status=in_progress, value="Beta — best effort"). DO NOT reintroduce 99.9% language anywhere in this story's deliverables.

---

## Acceptance Criteria

### AC1 — Trust Center MDX cards render and reflect the beta posture

- [ ] `eusolicit-app/frontend/apps/client/content/trust/compliance-posture.mdx` renders both posture cards under the post-pivot beta posture:
  - `availability` card: status=`in_progress`, value="Beta — best effort", slug=`availability`
  - `rtoRpo` card: status=`compliant`, value="RTO ≤ 4h, RPO ≤ 24h", slug=`rto-rpo`
- [ ] No `99.9%` SLA language anywhere in the rendered output (sanity-check via `grep -rE "99\.9|99 ?%" eusolicit-app/frontend/apps/client/content/trust/`)
- [ ] Pre-existing `encryptionAtRest` + `encryptionInTransit` cards continue to render correctly (regression guard — pre-existing landed work must not break)

### AC2 — i18n keys present with full bg/en parity

- [ ] `eusolicit-app/frontend/apps/client/messages/en.json` contains:
  - `trust.posture.availability.title` (or whatever the card-title pattern is — verify by inspecting how `encryptionAtRest` is keyed)
  - `trust.posture.availability.description` — the user-facing prose explaining beta posture (status-page link required per AC7 product+legal review)
  - `trust.posture.rtoRpo.title`
  - `trust.posture.rtoRpo.description` — the user-facing prose explaining RTO/RPO commitment
- [ ] `eusolicit-app/frontend/apps/client/messages/bg.json` contains the same 4 keys with Bulgarian translations
- [ ] Total key count delta: en.json grows by exactly +2 description keys (+2 title keys if titles weren't already there); bg.json same delta — full parity
- [ ] Existing key count (1799 keys both per 2026-05-13 audit) preserved — no accidental deletions

### AC3 — `pnpm check:i18n` passes

- [ ] `pnpm --filter @eusolicit-client check:i18n` (or `cd eusolicit-app/frontend/apps/client && pnpm check:i18n`) exits 0
- [ ] Lint output shows no missing keys, no orphaned keys, no parity violations between en.json + bg.json
- [ ] Pre-existing pattern verified: this is the existing project i18n discipline gate (per CLAUDE.md "i18n: `next-intl`; run `pnpm check:i18n` after adding strings")

### AC4 — WCAG 2.1 AA accessibility on the rendered MDX

- [ ] Keyboard navigation: card status badges (`in_progress` / `compliant`) reachable via Tab; descriptions readable in tab order
- [ ] Color contrast: status-badge text contrast ratio ≥ 4.5:1 for normal text, ≥ 3:1 for large text or graphical elements (per WCAG 2.1 AA §1.4.3 + §1.4.11). The `in_progress` status visual treatment in particular must not rely on color alone (per §1.4.1)
- [ ] Screen-reader announcement: status badges have `aria-label` or equivalent so screen readers announce "Availability: in progress" vs "Encryption at rest: compliant" without forcing the user to interpret the visual badge
- [ ] No keyboard trap, no missing focus indicators, no `:hover`-only affordances
- [ ] Spot-check via axe-core / Lighthouse accessibility audit on the rendered Trust Center page

### AC5 — DEFERRED to follow-up story `public-sla-7-day-soak-observation`

> **NOT IN THIS STORY'S CLOSE PATH.** Originally proposal §3.1.3 AC5 covered the 7-day Trust Center soak — explicitly split during create-story per PM recommendation in sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md §3.1.3 + §4.3. See §See also for the follow-up.

### AC6 — Monitoring alert for SLA disclosure page 404

- [ ] New alert rule in `eusolicit-app/infra/observability/prometheus/rules/` (recommended file: `frontend-routes-alerts.yaml`, new) — alert fires when any of the Trust Center MDX-rendered routes (`/<locale>/trust/*`) return sustained 404 (rate > 0 sustained for ≥ 5min)
- [ ] Alert metadata: `severity: ticket`, `slo_target: platform`, runbook_url present (per pe-06 §AC7 routing discipline) — pointing at a new runbook stub `eusolicit-docs/runbooks/trust-center-404.md` (minimal — operator escalation procedure; "page does not render" remediation)
- [ ] Alert routes via pe-06 `slack-platform-alerts` receiver (severity=ticket per pe-06 routing model)
- [ ] `scripts/check_runbook_url_coverage.py` continues to pass (existing onprem-04 lint gate, would fail if alert ships without runbook_url)
- [ ] Alert rule syntax-valid: `python3 -c "import yaml; yaml.safe_load(open('infra/observability/prometheus/rules/frontend-routes-alerts.yaml'))"` succeeds

### AC7 — Product + Legal review of disclosure card content

- [ ] PM (📋 John) circulates the proposed `availability` + `rtoRpo` description strings (en + bg) to product owner (Deb) for tone/business-posture sign-off
- [ ] Legal review on the `availability` description specifically — the "Beta — best effort" framing must not inadvertently create a binding SLA commitment or contradict the master service agreement (legal stakeholder per company practice)
- [ ] Status-page link added to the `availability` card description (existing status-page route under `apps/client/app/[locale]/` — confirm exact path during dev; per public-sla sprint-status row audit comment: "Status page link is implicit … add link to the availability card description if not present")
- [ ] Sign-off recorded inline in the story file's §Review Findings section (or commit-message annotation if no formal §Review Findings section exists at time of sign-off)
- [ ] **NOT a code gate** — this is a cross-functional review gate. PM dispatches the review request; product + legal respond on their own cadence. Story stays at `review` until sign-off arrives. Sign-off is calendar-paced but typically <5 business days.

---

## Dev Notes

### What's on disk (auditable artefacts — INPUTS, not deliverables)

1. **`eusolicit-app/frontend/apps/client/content/trust/compliance-posture.mdx`** — already has 2 posture cards landed under 2026-05-11 partial PM dev pass. Lines 36-44 (availability) + 45-49 (rtoRpo). MDX frontmatter establishes the `description_key` pattern (`trust.posture.<card>.description`). Pre-existing `encryptionAtRest` + `encryptionInTransit` cards (lines 25-35) follow the same pattern — REFERENCE them for i18n key naming consistency.
2. **`eusolicit-app/frontend/apps/client/messages/en.json`** + **`messages/bg.json`** — both at 1799 keys (per 2026-05-13 audit). The 4 new keys (2 cards × {title, description}) land here. Inspect the existing `encryptionAtRest` keys for naming/structure pattern before authoring.
3. **`eusolicit-app/frontend/apps/client/package.json`** — `check:i18n` script: `"check:i18n": "node scripts/check-i18n-keys.mjs"`. Run from `apps/client/` directory.
4. **`eusolicit-app/scripts/check_runbook_url_coverage.py`** — existing onprem-04 lint gate. New alert rule under AC6 MUST satisfy this (every alert ships `runbook_url:` annotation).

### Files this story modifies (UPDATE — preserve existing behaviour)

1. **`compliance-posture.mdx`** — only if i18n key naming under AC2 forces an MDX frontmatter adjustment (currently `description_key: "trust.posture.availability.description"` — confirm the renderer reads from this key vs. some convention-derived key). If MDX is already correct (likely — the description_key pattern looks intentional), this file is UNCHANGED by AC2.
2. **`messages/en.json`** + **`messages/bg.json`** — APPEND 4 new keys (or 2 if titles were already added). Preserve all 1799 existing keys verbatim. JSON formatting per project style (likely 2-space indent, sorted by key — verify).
3. **`infra/observability/prometheus/rules/`** — NEW file `frontend-routes-alerts.yaml` per AC6. Follow `host-alerts.yaml` structural pattern: groups[0].name, interval, rules[] with alert/expr/for/labels/annotations including runbook_url.

### Files this story creates (NEW)

1. **`infra/observability/prometheus/rules/frontend-routes-alerts.yaml`** — Prometheus alert rule file per AC6
2. **`eusolicit-docs/runbooks/trust-center-404.md`** — minimal operator runbook per AC6 (referenced by alert's runbook_url)

### Anti-pattern guards

- **DO NOT reintroduce 99.9% SLA language** — superseded by ADR-010 beta posture (2026-05-11). Sanity check: `grep -rE "99\.9|99 ?%" eusolicit-app/frontend/apps/client/content/trust/ eusolicit-app/frontend/apps/client/messages/` must NOT match any string in this story's deliverables.
- **DO NOT inflate scope to include the 7-day soak observation** — AC5 EXPLICITLY split to a follow-up story (`public-sla-7-day-soak-observation`). This story closes on code-Approve + i18n + monitoring alert + product/legal sign-off. The soak is OUT.
- **DO NOT skip bg.json parity** — every key added to en.json MUST have a corresponding bg.json entry. `pnpm check:i18n` enforces this; AC3 is the gate.
- **DO NOT bypass product/legal sign-off** to close the story — AC7 is a cross-functional review gate. Sign-off is required even if calendar-paced. PM owns dispatch + tracking.
- **DO NOT rewrite the MDX card structure** — pre-existing pattern (status / value / description_key) is canonical. Match `encryptionAtRest` + `encryptionInTransit` shape exactly.
- **DO NOT couple this story to pe-06 alertmanager routing** — pe-06 already closed today 2026-05-13 (review→done). AC6's alert SHOULD route through pe-06's existing `slack-platform-alerts` receiver via `severity: ticket` label; verify but don't modify pe-06 routing config.

### Carry-forward inputs (proposal §3.1.3 + §4.3)

- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md` §3.1.3 + §4.3 — this story's parent disposition + AC5 split decision
- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-12.md` — original epic-23 injection that created the orphan row
- `eusolicit-docs/implementation-artifacts/epic-21-retro-2026-05-05.md` — carry-forward origin (E21 retro flagged public-sla-soak as operator-deferred under the original 99.9% SLA scope, now superseded)
- `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md` — ADR-010 establishes the "best-effort availability" posture this story formalizes for public consumption
- `eusolicit-docs/implementation-artifacts/pe-06-pagerduty-rotation-provisioning.md` — sibling story; AC6 monitoring alert routes through pe-06's `slack-platform-alerts` receiver per `severity: ticket` label

### Salvaged patterns

- **MDX posture card structure** — `encryptionAtRest` + `encryptionInTransit` cards already at `compliance-posture.mdx` lines 25-35 define the canonical shape; new `availability` + `rtoRpo` cards already follow it (lines 36-49)
- **`next-intl` i18n key pattern** — existing `trust.posture.encryptionAtRest.description` (verify in en.json/bg.json) is the namespace pattern for the new keys
- **`pnpm check:i18n` discipline** — per CLAUDE.md frontend section; existing gate, no new tooling needed
- **Prometheus alert rule + runbook_url pattern** — `host-alerts.yaml` (onprem-04) is the structural reference; same annotation discipline
- **pe-06 severity:ticket routing** — `slack-platform-alerts` receiver established and validated under today's pe-06 close

### Test approach

Component / unit:
- (Optional, recommended) Playwright E2E test: render `/en/trust/compliance-posture` + `/bg/trust/compliance-posture`; assert no missing i18n keys (no `[MISSING_KEY]` placeholder text); assert all 4 posture cards visible with their expected status badges
- `pnpm check:i18n` is the primary i18n gate (AC3)
- Axe-core / Lighthouse audit for AC4 — either as a CI step or operator-run pre-merge

Monitoring alert:
- Prometheus rule syntax validation (yaml.safe_load) is the minimum gate (AC6)
- Test-fire the alert via synthetic 404 against the rendered route (block route access at nginx OR send a forced 404 response) — verify routing reaches `#platform-alerts` Slack channel; record evidence in commit message or follow-up runbook stub

### Risk acknowledgment

Per proposal §3.3:

- **Calendar-paced AC7 review** — product + legal sign-off may take >5 business days depending on stakeholder availability. Mitigation: PM (📋 John) dispatches the review request EARLY (parallel with code work), not at end of cycle. Story stays at `review` post-code-Approve until sign-off lands.
- **i18n drift if check:i18n CI gate isn't blocking** — confirm during dev whether `check:i18n` runs in CI workflow (`.github/workflows/`). If not blocking, add it (small scope, contained in this story's AC3 closure if discovered).
- **WCAG audit ambiguity** — AC4 spot-check via axe-core may surface edge-case violations that aren't strict regressions. Mitigation: scope AC4 strictly to badge contrast + keyboard nav + screen-reader announcement; broader Trust Center page-level audits roll into a separate accessibility-sweep story if needed.

---

## Closure path

1. **Dev (Amelia)** implements AC1-AC4 + AC6:
   - Author i18n keys in en.json + bg.json (AC2)
   - Verify `pnpm check:i18n` clean (AC3)
   - WCAG 2.1 AA validation (AC4)
   - Author Prometheus alert rule + runbook stub (AC6)
2. **PM (📋 John)** dispatches AC7 review request to Deb (product) + legal stakeholder (parallel with dev work to absorb calendar latency)
3. **`bmad-code-review`** dispatched on the story spec + the dev deliverables; on Approve verdict → story Status flips `ready-for-dev → review` (story-file Status only; sprint-status stays `ready-for-dev` until AC7 sign-off lands per AP17-C1)
4. **AC7 sign-off** lands inline in §Review Findings (or commit annotation); PM records the sign-off
5. **AP17-C1 two-gate close** — story Status `review → done` + sprint-status `public-sla-announcement-soak-gate: ready-for-dev → done` (AP18-C2 atomic) in same commit-equivalent
6. **Follow-up dispatch** — PM (or operator) dispatches `bmad-create-story` for `public-sla-7-day-soak-observation` once THIS story is `done`. That follow-up story owns the 7-day stage soak gate before public flip to prod.
7. **Epic 23 close** — public-sla done is one of 4 remaining gates (alongside drift-recovery code-Approve, pe-04 normal dispatch + drill execution, inj-03 TEA escalation)

---

## See also

- **Follow-up story (to author post code-Approve):** `public-sla-7-day-soak-observation` — owns the deferred AC5 (7-day stage soak + 0 P0/P1 incidents touching SLA disclosure surface + flip to prod). Author via `bmad-create-story` once THIS story is `done`.
- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md` §3.1.3 + §4.3 — parent disposition + AC5 split rationale
- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-12.md` — original epic-23 injection
- `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md` — ADR-010 (establishes the "Beta — best effort" public-facing posture)
- `eusolicit-docs/implementation-artifacts/pe-06-pagerduty-rotation-provisioning.md` — sibling story; AC6 monitoring alert routes through pe-06's `slack-platform-alerts` receiver
- `eusolicit-app/frontend/apps/client/content/trust/compliance-posture.mdx` — the carry-forward MDX file
- `eusolicit-app/frontend/apps/client/scripts/check-i18n-keys.mjs` — the i18n parity gate enforced by AC3
- `eusolicit-docs/implementation-artifacts/epic-22-retro-2026-05-12.md` § E22-AP01 — anti-pattern this re-create closes
- `eusolicit-docs/planning-artifacts/epics/E23-operational-close-out.md` — owning epic
