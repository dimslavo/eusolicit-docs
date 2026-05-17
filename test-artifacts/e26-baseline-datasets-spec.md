---
Status: backlog
Epic: E26 — Agent-Driven Ingestion & Analysis
Spec type: TEA / evaluation baseline dataset
Owner: Murat (TEA) + PM (curation) + SME review
Created: 2026-05-15 (via IR remediation — `implementation-readiness-report-2026-05-15.md` Issue #10)
Source: E26 epic file ACs for S26.01 (qualifier eval-runs ≥85% baseline alignment) + S26.02 (quantifier eval-runs ±25% historical actuals)
Blocks: E26 S26.01 + S26.02 dispatch
---

# E26 Baseline Dataset Specification

## Purpose

E26 acceptance criteria for both `opportunity_qualifier` (S26.01) and `opportunity_quantifier` (S26.02) require validation against a baseline set:

- **S26.01 Acceptance:** *"Eval-run with 20 sample opportunities → ≥85% align with human qualification baseline"*
- **S26.02 Acceptance:** *"Eval-run: estimated values within ±25% of historical actuals on validation set"* (15 sample opportunities + historical outcomes)

The IR found that the **baseline dataset itself doesn't exist yet** — the ACs are aspirational without the data. This spec defines what those datasets are, where they live, who curates them, and how alignment / accuracy is measured. Without it, the eval-runs in S26.01 and S26.02 cannot be operationalised, which means those stories cannot reach `done` honestly.

## Scope

Two distinct dataset specifications, related but not identical:

1. **`qualifier-baseline-v1.json`** — 20 opportunities + human-authored qualification labels.
2. **`quantifier-baseline-v1.json`** — 15 opportunities (subset of the 20 above is acceptable) + historical bid outcomes.

Both live under `services/client-api/tests/data/sirmaai-baselines/` (kept in-repo, NOT committed to canonical Postgres — these are test fixtures, not seed data).

---

## Dataset 1 — Qualifier Baseline (`qualifier-baseline-v1.json`)

### Schema

```json
{
  "version": "v1",
  "created": "2026-05-15",
  "curator": "Deb (PM) + 1 SME bid manager (TBD)",
  "opportunities": [
    {
      "opportunity_id": "OPP-BL-001",
      "source": "AOP | TED | EU-Grants",
      "title": "...",
      "cpv_codes": ["..."],
      "deadline": "YYYY-MM-DD",
      "budget_eur": 12345.67,
      "language": "bg | en",
      "raw_text_path": "fixtures/raw-tenders/OPP-BL-001.pdf",
      "company_profile_ref": "profile-fixtures/sme-environmental-consulting.json",
      "ground_truth": {
        "fit_score_0_100": 78,
        "gap_analysis_keywords": ["missing-cert-iso-14001", "team-size-mismatch"],
        "recommended_action": "pursue | monitor | decline",
        "confidence_band": "low | medium | high",
        "rationale_text": "...",
        "labelled_by": "SME bid manager",
        "labelled_at": "YYYY-MM-DD"
      }
    }
    // ...19 more
  ]
}
```

### Composition Requirements

The 20 opportunities must hit each cell of this stratification grid (≥2 per cell):

| Source | Recommended Action | Min count |
|---|---|---|
| AOP | pursue | 3 |
| AOP | monitor | 2 |
| AOP | decline | 2 |
| TED | pursue | 3 |
| TED | monitor | 2 |
| TED | decline | 2 |
| EU-Grants | pursue | 2 |
| EU-Grants | monitor | 2 |
| EU-Grants | decline | 2 |
| **Total** | | **20** |

Additionally:
- ≥4 opportunities have a **deliberate gap signal** (missing certification, mismatched CPV code, geography exclusion, language barrier) — to exercise gap-analysis output.
- ≥3 opportunities have **high-budget + tight-deadline** combinations — to exercise confidence-band edge cases.
- ≥2 opportunities are **near-edge "monitor"** (could reasonably be argued either way by humans) — to stress the qualifier's calibration.

### Curation Process

1. **PM (Deb)** selects 30 candidate opportunities from production data (anonymised — no real company names, redacted contacts). Stratify per grid.
2. **SME bid manager** (TBD — recommend hiring one Bulgarian consulting firm contact for a paid 4-hour calibration session) reviews and labels each candidate. SME does NOT see any AI output during labelling.
3. **PM** culls 30 → 20, prioritising agreement (high inter-rater confidence) over disagreement, but keeping the 2 near-edge cases.
4. Output committed at `services/client-api/tests/data/sirmaai-baselines/qualifier-baseline-v1.json` + raw tender PDFs at `fixtures/raw-tenders/`.

### Alignment Metric (for S26.01 ≥85% AC)

For each of 20 opportunities, define **alignment** as a 3-component check:

1. **Action match** (binary): does the agent's `recommended_action` exactly match the SME's label? Weight 0.6.
2. **Fit-score band** (binary): does the agent's `fit_score_0_100` fall in the same band (0-33 / 34-66 / 67-100) as the SME's? Weight 0.3.
3. **Gap-analysis overlap** (proportional): fraction of `gap_analysis_keywords` from the SME label that appear in the agent's gap-analysis output (string match, lowercased, stemmed). Weight 0.1.

**Per-opportunity score** = 0.6·(1) + 0.3·(2) + 0.1·(3). Range [0, 1].
**Eval-run aggregate alignment** = mean across 20 opportunities × 100. Must be ≥ 85 for S26.01 AC to pass.

### Regression Protocol

- Eval-runs execute on the `opportunity_qualifier` agent definition committed at the time. Re-run on every change to the agent prompt OR template, OR on every SirmaAI agent-version upgrade.
- Failures (< 85%) bring the agent definition into review; no auto-promotion to `100%` rollout flag.
- A delta-vs-previous-eval-run report is generated automatically and attached to the PR that bumps the agent version.

---

## Dataset 2 — Quantifier Baseline (`quantifier-baseline-v1.json`)

### Schema

```json
{
  "version": "v1",
  "created": "2026-05-15",
  "curator": "Deb (PM) + 1 SME bid manager + historical EU Solicit production data",
  "opportunities": [
    {
      "opportunity_id": "OPP-BL-001",     // overlap with qualifier baseline OK
      "company_profile_ref": "...",
      "historical_actuals": {
        "actual_effort_person_days": 14.5,
        "outcome": "won | lost | withdrawn | not_bid",
        "actual_contract_value_eur": 24500,    // 0 if lost/withdrawn
        "outcome_certainty": "verified_by_company | inferred_from_portal | self_reported",
        "actual_bid_decision_threshold_eur": 18000   // company's stated threshold at time of bid
      },
      "kb_context": {
        "company_past_proposals_count": 12,
        "company_win_rate_pct": 22,
        "kb_completeness_band": "high | medium | low"
      }
    }
    // ...14 more
  ]
}
```

### Composition Requirements

15 opportunities sourced as follows:

- **8 opportunities** from EU Solicit production data (the platform shipped 2026-04-XX onwards) — companies who consented to research use of anonymised bid data. *PM owns the consent process.*
- **5 opportunities** from the SME bid manager's prior client portfolio (anonymised) — covers older bids before EU Solicit existed.
- **2 opportunities** synthetic + adversarial — designed to test "thin KB" (company_past_proposals_count = 0) and "no MCP/CRM data" fallback paths.

Stratification:
| `outcome` | Min count |
|---|---|
| won | 5 |
| lost | 4 |
| withdrawn | 3 |
| not_bid | 3 |
| **Total** | **15** |

### Accuracy Metric (for S26.02 ±25% AC)

For each of 15 opportunities, compute three per-prediction errors:

1. **Effort error** = `|estimated_effort_person_days - actual_effort_person_days| / actual_effort_person_days`.
2. **Win-prob calibration** = Brier score component for `estimated_win_probability` against `outcome ∈ {won=1, lost=0}` (omitted for withdrawn/not_bid).
3. **Expected-value error** = `|expected_value_eur - actual_contract_value_eur| / max(actual_contract_value_eur, 1000)`.

**Eval-run aggregate accuracy**: mean of (1) and (3), separately. Both must be ≤ 0.25 (within ±25%) for S26.02 AC to pass. (2) is reported but not gated until N ≥ 50 (calibration needs sample size).

### Fallback-Confidence Test

For the 2 synthetic "thin KB" opportunities, AC: the quantifier MUST return outputs with `confidence ≤ "medium"` AND include a `caveat: "low_kb_context"` field. Failure to surface low confidence is itself a fail signal — overconfidence is the killer failure mode here.

---

## Curation Timeline + Owners

| Step | Owner | Effort |
|---|---|---|
| Hire SME bid manager (or extend existing contact for paid 4h calibration) | Deb | 1 week wall-clock, ~€600 |
| Select 30 qualifier candidates from production | Deb + PM | ½ day |
| SME labelling pass on 30 candidates | SME (paid) | 4h |
| Cull 30 → 20, commit to repo | Deb | ½ day |
| Source 8 quantifier opps from EU Solicit prod (consent dance) | Deb + Customer Success | 1 week wall-clock |
| Source 5 quantifier opps from SME's portfolio | SME | 2h |
| Author 2 synthetic adversarial opps | Deb | 2h |
| Quantifier dataset commit | Deb | ½ day |
| Murat (TEA) review of both datasets for adequacy | Murat | 2h |

**Total wall-clock**: 2 weeks. **Total cost**: ~€600 SME + Customer Success time.

## Risk

- **R1**: SME availability — if no Bulgarian bid manager will calibrate within budget, qualifier baseline cannot ground-truth. **Fallback**: PM curates labels alone, flag as `single-rater` in v1; v2 adds a second SME post-launch. This degrades S26.01 confidence but not its dispatchability.
- **R2**: EU Solicit production data privacy — consent must be obtained from companies whose bids are anonymised into the quantifier baseline. **Mitigation**: Trust Center sub-processor disclosure (already part of `sirmaai-eu-residency-due-diligence` story scope) extended to cover research use of anonymised data.
- **R3**: Quantifier "ground truth" for `actual_effort` is hard to measure post-hoc. Most companies don't time-track bid prep. **Mitigation**: lean on EU Solicit's `bid_preparation_logs` table where shipped (E19 telemetry); otherwise use self-reported with `outcome_certainty: self_reported` flag.

## Acceptance Signal

- Both JSON datasets committed under `services/client-api/tests/data/sirmaai-baselines/v1/` (NOT canonical seed data).
- README at the same path documents the schema, composition grid, alignment metric, accuracy metric, refresh policy.
- This spec's `Status:` flips to `done` when both files are committed and Murat (TEA) signs off.

## See also

- `eusolicit-docs/planning-artifacts/epics/E26-agent-driven-ingestion.md` ACs for S26.01 / S26.02
- `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-15.md` Issue #10
- `eusolicit-docs/implementation-artifacts/sirmaai-eu-residency-due-diligence.md` (consent / Trust Center scope overlap)
