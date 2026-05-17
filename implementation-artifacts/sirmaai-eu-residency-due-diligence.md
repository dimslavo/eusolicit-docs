---
Status: backlog
Epic: E23 — Operational Close-Out
Story type: operator / legal due-diligence (not engineering)
Owner: PM (Deb) + legal counsel
Created: 2026-05-15 (via IR remediation — `implementation-readiness-report-2026-05-15.md` Critical Issue #2)
Source: `prd-amendment-2026-05-12-sirmaai.md` §Domain-Specific Requirements — Data Residency (amended)
Blocks: 2026-06-01 launch (per ADR-010 + PRD amendment)
---

# sirmaai-eu-residency-due-diligence

## Goal

Obtain signed contractual confirmation from SirmaAI (operator of `agenticsai.endigitalx.com`) that all data stored or processed within the EU Solicit Organisation (and every per-tenant Project under it) — including knowledge-base artefacts (E25), agent execution traces, agent memory, prompt logs, vector embeddings, MCP-server secrets, and any derivative analytics — resides exclusively in EU data centres.

Without this confirmation, EU Solicit cannot honestly defend GDPR Article 44 compliance (transfers outside the EEA) to enterprise prospects, and the PRD amendment's amended Data Residency requirement is unenforceable. PRD amendment §Domain-Specific Requirements flags this as **"launch-blocking due-diligence item"**.

## Acceptance Criteria

- [ ] **AC1** — Written confirmation from SirmaAI (signed amendment to the existing service agreement, OR a standalone Data Processing Addendum) stating that:
  - (a) All customer data, agent traces, memory, KB storage-resource bodies, vector embeddings, prompt logs, and derivative analytics for the EU Solicit Organisation are stored exclusively in EU data centres (named in the document).
  - (b) Sub-processors used by SirmaAI that touch EU Solicit data are EU-resident or covered by Standard Contractual Clauses (SCCs); the sub-processor list is named in the addendum.
  - (c) SirmaAI shall notify EU Solicit at least 30 days before adding or changing any sub-processor that processes EU Solicit data.
- [ ] **AC2** — SCC clauses included where any onward transfer is unavoidable (Art. 46 GDPR safeguards documented).
- [ ] **AC3** — A copy of the signed document is archived in `eusolicit-docs/legal/sirmaai-dpa-2026-XX.pdf` (or equivalent legal-docs path) and committed to the repo with `legal-counsel` reviewer approval on the PR.
- [ ] **AC4** — Trust Center (`frontend/apps/client/content/trust/sub-processors.mdx` or equivalent) updated to declare SirmaAI as a sub-processor with EU residency confirmation, sub-processor change-notification commitment, and a link to the public DPA summary.
- [ ] **AC5** — `eusolicit-docs/planning-artifacts/PRD.md` §Domain-Specific Requirements — Data Residency amended (or PRD amendment annotated) to flip the "launch-blocking due-diligence item" line to "confirmed on 2026-XX-XX (see legal/sirmaai-dpa-2026-XX.pdf)".
- [ ] **AC6** — `project-context.md` updated with the residency-confirmation reference for future-agent context (memory rule observed: surgical add to the existing SirmaAI pivot section, no structural regeneration).
- [ ] **AC7** — `bmad-code-review` Approve verdict on the AC4 + AC5 + AC6 PR (two-gate close per AP17-C1); operator completion signal in this story file's Status header transitioning `ready-for-dev → done` only after the legal addendum is signed.

## Dev / Operator Notes

This is **operator + legal** work, not engineering. The story is dispatchable through the BMAD `ready-for-dev` → `review` → `done` lifecycle but the bulk of the effort is:

1. **PM (Deb)** opens a conversation with SirmaAI's commercial / legal contact requesting a Data Processing Addendum scoped to the EU Solicit Organisation. Anchor the request in (a) GDPR Art. 28 (processor) + Art. 44 (cross-border transfers), (b) ADR-010 EU residency commitment, (c) PRD amendment 2026-05-12 §Domain-Specific Data Residency.
2. **Legal counsel** reviews the resulting addendum against the AC1 (a)/(b)/(c) clauses + AC2.
3. **PM** lands the AC3/AC4/AC5/AC6 edits as a single PR; tags `legal-counsel` for sign-off; requests `bmad-code-review` for the doc/MDX/JSON changes.
4. **Operator** flips story status to `done` only after AC1 + AC2 are signed and AC3..AC7 are merged.

## Risk

- If SirmaAI refuses or cannot honour AC1 (a)/(b)/(c) within a commercially-acceptable timeframe, this becomes a **hard launch blocker** — EU Solicit must either (1) renegotiate the ADR-010 launch date or (2) raise an architecture amendment to add a residency-attestation layer (e.g. self-hosted SirmaAI agent fleet on EU infra, which is out of scope for the 2026-06-01 launch posture).
- Backup posture: if AC1 (a) is granted but AC1 (c) sub-processor notification is refused, the Trust Center disclosure (AC4) must be downgraded to "best-effort sub-processor change notification — confirm before contractually committing".

## Out of Scope

- ISO 27001 certification for SirmaAI (separate audit programme).
- SOC 2 Type II coverage of SirmaAI (separate vendor due-diligence track).
- Renegotiation of SirmaAI commercial terms beyond residency (rate-limit ceilings, support SLAs).

## See also

- `eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md` §Domain-Specific Requirements — Data Residency (amended)
- `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §SirmaAI substrate (ADR-018)
- `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md` (ADR-010)
- `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-15.md` Critical Issue #2
