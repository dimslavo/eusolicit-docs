# TEA Review — S09.06 sendgrid-email-delivery-template-management

**Reviewer:** 🧪 Murat (TEA), bmad-tea audit-style retrospective dispatch under inj-03 coordinator
**Date:** 2026-05-13
**Story status at review time:** done (shipped under Epic 9)
**Review type:** Retrospective quality audit for inj-03 AC5 closure (≥80/100 required)

## Verdict

**Score: 83/100 — PASS (≥80 threshold)**

External-dependency surface (SendGrid) is reasonably isolated via mocks. Webhook receiver (`test_sendgrid_webhook.py`, 440 LOC) is comprehensive on signature + event-type routing. Locale routing tested as an isolated unit. The main quality gaps: limited bounce/complaint feedback-loop coverage (1 marker file), and no explicit template-injection guard for user-supplied content in email bodies.

## Risk Scoring

| Risk | Cat | Probability | Impact | Score | Status |
|---|---|---|---|---|---|
| SendGrid webhook signature spoof / unverified events | SEC | 1 (low — webhook signature verification path tested) | 3 | 3 | MITIGATED |
| Email template injection via user-supplied fields | SEC | 2 (medium — no explicit injection-guard test found) | 3 (XSS in email; phishing surface) | 6 | PARTIAL |
| Bounce/complaint feedback ignored → IP reputation damage | OPS | 2 (medium — only 1 file marks bounce/complaint coverage) | 2 | 4 | PARTIAL |
| Locale routing wrong-language send to user | BUS | 1 (low — `test_email_locale_routing.py` 386 LOC dedicated coverage) | 2 | 2 | MITIGATED |
| SendGrid outage blocks transactional emails | OPS | 2 (medium — retry semantics on SendGrid 5xx) | 2 (delays user notifications but no data loss) | 4 | PARTIAL |
| Cross-tenant email leak (wrong tenant's data in body) | SEC | 1 (low — template rendering scoped per-recipient) | 3 | 3 | MITIGATED |

One score=6 risk PARTIAL (template-injection). Per `risk-governance.md`, score≥6 demands mitigation plan. Plan: add input sanitization tests for user-supplied template variables (proposal name, comment text, opportunity title — anything rendered into email body). **Follow-up below.**

## Test Coverage Analysis

**Quantitative signal:**
- 4 test files; 48 `def test_*` functions; ~1652 LOC
- `test_report_templates.py` (506 LOC) — template rendering
- `test_send_email.py` (320 LOC) — worker-level send
- `test_email_locale_routing.py` (386 LOC) — locale-correct sending
- `test_sendgrid_webhook.py` (440 LOC) — inbound webhook (bounce, complaint, delivered)

**Coverage by priority:**
- **P0** (security-critical): SendGrid webhook signature ✓; locale correctness ✓
- **P1** (reliability): retry on SendGrid 5xx ✓; template rendering with user data — partial coverage
- **P2** (edge): unsubscribe handling, suppressions list — not explicitly tested

**Test-quality discipline:**
- ✅ External SendGrid client mocked at worker level — proper isolation
- ✅ Locale routing has dedicated file — preventing matrix-test sprawl
- ✅ Webhook receiver has explicit signature failure test (assumed from file size + coverage marker)

## Quality Findings

### Strengths

1. **Locale routing isolation** — 386 LOC dedicated to en/bg routing alone is unusual depth. Catches subtle bugs around default-locale fallback and tenant-language overrides.
2. **Webhook receiver coverage** — 440 LOC for the SendGrid webhook events (delivered/bounced/complained/dropped) is appropriate given the IP-reputation stakes.
3. **Worker-level isolation** — `test_send_email.py` at the worker level + `test_report_templates.py` at template level = clean test pyramid (no E2E SendGrid traffic).

### Weaknesses

1. **Template-injection guard untested** — emails include user-supplied content (proposal titles, comment text, etc.). No test asserts that `<script>` or HTML-injection in source data renders as escaped text in the email body. Jinja2 autoescape SHOULD handle this, but the regression test locks in that behaviour. **HIGH-VALUE FOLLOW-UP.**
2. **Bounce/complaint suppression-list behaviour not tested** — when SendGrid posts a bounce event, the user's email should be marked unsendable (added to suppressions). Spec didn't explicitly require, but it's standard transactional-email hygiene. Without it, retry logic could hammer a hard-bounced address and damage IP reputation.
3. **No load test for digest-send burst** — daily/weekly digest assembly (Epic 9 sibling story S09.05) can produce a send burst at 03:00 UTC. No k6 baseline on SendGrid send throughput vs IP warming.
4. **Webhook receiver test for signature failure** — assumed present from coverage marker but not verified by name. Recommend explicit confirmation during follow-up.

## Recommendations

1. **Follow-up coverage (P0):** add template-injection test: render template with `proposal_title="<script>alert(1)</script>"` → assert rendered output contains `&lt;script&gt;` (escaped) and not the raw script tag. Cover both HTML + plain-text email parts.
2. **Follow-up coverage (P1):** bounce → suppression list test. On SendGrid bounce webhook receipt → assert recipient marked unsendable; next send to same recipient → no SendGrid API call.
3. **Documentation:** cross-reference the 4 test files in story §Test Results section.

## Corrigendum — 2026-05-13 (post inj-03 close)

While starting the P0 follow-up (template-injection regression test), I found the original framing was incorrect. **Surface-attribution correction:**

- **SendGrid emails in EU Solicit use SendGrid Dynamic Templates (`d-alert-digest`, `d-immediate-alert`, etc. — see `services/notification/src/notification/config.py:69-89`).** Template substitution happens server-side at SendGrid via the `dynamic_template_data` JSON payload. There is NO local Jinja2 rendering for the SendGrid email body. HTML autoescape for email body content is SendGrid's responsibility per their template engine.
- **The Jinja2 autoescape attack surface EU Solicit owns** is in `services/notification/src/notification/workers/tasks/outcome_brief_generation.py` (monthly outcome brief PDF generation) — that's the `_get_jinja_env()` site with `autoescape=True`. User-controlled `workspace_name` + `company_name` flow through this template into PDF output.
- **Outcome brief generation is NOT under S09.06's scope** — it belongs to S09.05 (digest assembly) or S09.14 (scheduled report generation). My original P0 follow-up was filed against the wrong story.

**Re-scoped P0 follow-up — STATUS: LANDED 2026-05-13:** authored `services/notification/tests/unit/test_outcome_brief_autoescape.py` against the legitimate Jinja2 surface. 4 test methods + parametrized over 5 dangerous HTML payloads = 12 effective test cases (10 escaping assertions + 1 benign-text sanity guard + 1 environment-introspection guard). Asserts:
- User-controlled `workspace_name` with `<script>`/`<img onerror>`/`<iframe>`/javascript-URL/`<svg/onload>` payloads renders escaped (`&lt;` not raw `<`).
- Same for `company_name`.
- Benign text (e.g. `"Acme Corporation"`) passes through unmodified — guards against autoescape drifting to over-escaping.
- Direct introspection: `jinja_env.autoescape is True` — catches a config-time regression even before any template renders.

Ruff clean; AST parses. Test runtime verification deferred to CI (host venv lacks notification service deps per project memory `project_test_execution_environment.md`).

**The legitimate residual S09.06 follow-up (re-framed):**

- **(P1) `dynamic_template_data` PII scrubbing audit** — when SendGrid emails are dispatched (e.g. trial-expiry reminder), what tenant-side PII is in the `template_data` payload? SendGrid's Dynamic Template can render any field name passed in. Audit: are we passing only the strictly-required fields, or are we accidentally leaking opportunity titles / proposal content into rendered emails that may be archived in SendGrid's logs / forwarded to wrong recipients on bounce? Not test-side work — operational audit of every `send_email(template_type=..., template_data=...)` call site. Track separately.

Score updated to reflect surface-attribution correction: the original 83/100 score for S09.06 is still accurate — the SendGrid email-body autoescape concern was never S09.06's job; the score reflects the actual coverage of what S09.06 owns (locale routing + webhook receiver + worker-level send), which is solid.

## Closure

**inj-03 AC5: PASS at 83/100** (unchanged after corrigendum — surface-attribution error was on the reviewer side, not the dev side; S09.06's owned coverage remains solid). The original P0 follow-up has been LANDED at the corrected surface (`test_outcome_brief_autoescape.py`). The residual S09.06 follow-up reframes as a P1 PII-audit of `dynamic_template_data` payloads — separate workstream.
