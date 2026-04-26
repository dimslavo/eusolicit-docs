## Epic 20: NPS, Reviews & Onboarding Milestones

Integrate in-app NPS prompts and onboarding milestone tracking to improve user retention and drive positive reviews on G2/Capterra.
**FRs covered:** FR2.7, FR9.6

### Story 20.1: NPS Prompt and Review Routing

As a Pro+ User,
I want to provide feedback on the platform via an NPS prompt,
So that I can share my experience and optionally leave a public review.

**Acceptance Criteria:**

**Given** an active user who has not seen an NPS prompt in 90 days
**When** they log into the platform
**Then** an NPS survey appears
**And** if they score 9-10, they are offered an opt-in link to review on G2/Capterra
**And** if they score <= 6, feedback is routed internally to the CSM team

### Story 20.2: Onboarding Milestone Tracking

As a CSM,
I want to track a customer's progress towards their "First Bid in 14 Days" milestone,
So that I can intervene if they stall during onboarding.

**Acceptance Criteria:**

**Given** a newly created workspace
**When** the user completes sub-tasks (e.g., content uploaded, first AI summary generated)
**Then** the milestone tracker updates
**And** if the workspace stalls for >7 days without progress, an alert is sent to the CSM channel
