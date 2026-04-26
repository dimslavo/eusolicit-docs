# E20: NPS, Reviews & Onboarding (Residual)

**Sprint**: 18 | **Points**: 5 | **Dependencies**: E14, E19 | **Milestone**: Review-pipeline + retention optimisation
**Source:** PRD v1.1 §6 FR2.7; architecture-evaluation §2 Change-7

## Goal

Plug the review-collection pipeline so satisfied Pro+ customers route opt-in to G2/Capterra/Gartner Peer Insights review forms. Onboarding milestone tracking and CSM stall alerts are absorbed into Epic 19 (FR9.6); this epic is residual — just the NPS prompt + review routing. Use a third-party NPS SDK (Delighted, Wootric, or similar) to keep audit-trail clean for ISO 27001 evidence purposes (offload survey storage and response handling to vendor; we add the vendor to sub-processor list).

## Acceptance Criteria

- [ ] Third-party NPS SDK integrated into client app (Delighted/Wootric/equivalent)
- [ ] Tier-gated to Pro+ and above via TierGate Depends (Epic 6 pattern)
- [ ] Quarterly cadence (configurable, opt-out by tenant_admin); shown to active users (≥1 login/week)
- [ ] Score 9–10 (promoter) → opt-in routing to G2/Capterra review form via deep link with prefilled product version + customer logo
- [ ] Score ≤6 (detractor) → routed to CSM contact via structured-feedback form
- [ ] NPS history retained per workspace (analytics, not surveillance — opt-in disclosure shown on first prompt)
- [ ] NPS vendor added to `infra/sub-processors.yaml` (sub-processor change flow per Epic 18)
- [ ] Privacy notice updated to reference NPS vendor as sub-processor
- [ ] Workspace-scoped negative test: W1's NPS history does not leak into W2

## Stories

### S20.00: Third-Party NPS SDK + Opt-In Routing to G2/Capterra + Privacy Disclosure
**Points**: 5 | **Type**: fullstack + content

**SDK selection:** Evaluate Delighted vs. Wootric vs. SatisMeter; pick based on (a) GDPR compliance posture, (b) EU data residency option, (c) audit-trail for ISO 27001 evidence, (d) Pro+ tier price compatibility (recommended: Delighted, ~€100–200/mo).

**Frontend integration:**
- Vendor SDK integrated into `apps/client/` initialised on authenticated session start (post-AuthGuard hydration)
- TierGate Depends Pro+ enforced before SDK fires
- Quarterly trigger logic respected by vendor; first-prompt opt-in disclosure with link to privacy notice
- Score capture handler: 9–10 → modal with G2/Capterra deep links (configurable per workspace; default G2 first); 7–8 → "Thanks!"; ≤6 → modal routes to structured-feedback form posting to CSM email

**Backend:**
- `POST /api/v1/workspaces/:id/nps/feedback` (called by detractor flow) — captures structured feedback, posts to CSM contact via notification service
- Sub-processor entry added to `infra/sub-processors.yaml` for the chosen NPS vendor (triggers Epic 18 customer-DPA notification flow)
- Privacy notice updated on `/legal/privacy` page; deployment includes pre-launch DPA notification per GDPR Art. 28

**Tests:**
- Tier-gate E2E (Playwright): Free user does not see NPS prompt; Pro+ user does
- Score-routing test: 10 → G2 deep link with correct product version; 5 → CSM-feedback form
- Workspace-scoped negative test
- Privacy disclosure reachable from NPS prompt
- Sub-processor entry valid (Epic 18 lint passes)
