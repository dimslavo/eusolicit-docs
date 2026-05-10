# Auto-sync Incidents — Log

**Purpose:** Track production outages or near-misses caused by `chore: auto-sync …` commits. Acceptance criterion of Story `onprem-05`: **zero new incidents in the 30 days following the close of onprem-05** (workflow guardrails + branch protection live).

**Format:** Most recent first. One row per incident. Cross-link to post-mortems where available.

| Detected | Trigger commit | Type | Duration (live impact) | Resolution commit(s) | Post-mortem | Notes |
|---|---|---|---|---|---|---|
| 2026-05-09 ~18:53 UTC | `ac2fd08 chore: auto-sync 2026-05-09 18:31:59` | SEV1 — full prod outage | ~4h (until first green deploy at 2026-05-09 19:15:57 + further fixes through 2026-05-10 08:56 UTC) | `e52bb40 fix(frontend): unblock production build — type/lint errors from auto-sync`; `27487a7 fix(deploy): unbreak prod /api/v1, retire admin host TLS error, clear ac2fd08 lint/type debt`; `bc051a9 fix(deploy): allocate dedicated 1xxxx host-port range …`; `3cba326 fix(nginx): strip /admin-api/ prefix before forwarding to admin-api` | TBD — open ticket | Direct cause: auto-sync push triggered ungated `deploy.yml`. Frontend build failed (TS/lint errors); admin TLS broke (cert SAN); `/api/v1` returned 404 (nginx strip mismatch). Recovery required 9 manual commits across ~14 hours. **Story `onprem-05` opened in response.** |

---

## Process

Whenever a `chore: auto-sync …` commit causes (or nearly causes) a production issue:

1. **Append a new row above this section** within 24h of detection.
2. **Open a post-mortem** in `eusolicit-docs/post-mortems/` if SEV1 or SEV2.
3. **Tag the trigger commit and resolution commit(s)** for traceability.
4. **Update the project memory note** "Auto-sync ships unverified code" if the failure mode is new or has evolved.

## Close-out metric

**`onprem-05` acceptance gate AC7:** zero new rows added between the merge date of `onprem-05` and 30 days later. If a row appears in that window, `onprem-05` fails its long-term AC and a follow-up ticket is opened to harden the gate further.

**Close-out target window:** TBD (set when onprem-05 lands in `done`).

## References

- Story: `eusolicit-docs/implementation-artifacts/onprem-05-deploy-guardrails.md`
- Branch protection runbook: `eusolicit-docs/runbooks/branch-protection-policy.md`
- Workflow: `eusolicit-app/.github/workflows/deploy.yml`
- Project memory: `On-prem pivot 2026-05-11` + `Auto-sync ships unverified code`
