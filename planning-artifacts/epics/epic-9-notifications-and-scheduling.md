---
epic: 9
title: Notifications and Scheduling
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-40", "FR-41"]
nfrs_relevant: ["NFR-6", "NFR-15", "NFR-23"]
ux_drs_relevant: ["UX-DR1"]
---

# Epic 9: Notifications and Scheduling

**Phase**: MVP (FR-40) + Post-MVP (FR-41) | **Dependencies**: Epic 1 (Identity), Epic 2 (Tier gates), Epic 3 (Opportunities), Epic 7 (Tasks/comments), `notification` service

## Goal

Users receive timely, configurable notifications (email digests of newly matched opportunities; in-app/email notifications for collaboration events) and can sync opportunity deadlines and proposal tasks to external calendars (Google, Outlook), so they never miss a relevant tender or internal deadline.

## User Outcome

After this epic, a user can configure daily or weekly digest emails of newly matched opportunities, receive in-app notifications when a teammate @mentions them or assigns them a task, and (Pro+ tier) automatically have opportunity deadlines and task due dates appear in their Google or Outlook calendar — kept in sync if dates change.

## Epic Acceptance Criteria

- [ ] All notification dispatch flows through the `notification` service; no service emails directly.
- [ ] Notification preferences are per-user-per-workspace (channel × event-type matrix).
- [ ] Email delivery uses a reputable provider with bounce/complaint feedback wired into preference suppression.
- [ ] Calendar sync uses OAuth refresh tokens encrypted at rest with an extra application-layer key (NFR-6).
- [ ] Calendar updates propagate within 60s of the source change in EU Solicit.
- [ ] Notification delivery failures are logged, retried with backoff, and observable via metrics (NFR-23).

## Stories

### Story 9.1: Email Digests for Opportunities

As a user,
I want to configure and receive email digests of newly matched opportunities,
So that I don't miss relevant tenders even when I'm not logged in.

**FR coverage**: FR-40

**Acceptance Criteria:**

**Given** I have opted into daily or weekly opportunity digests in notification settings
**When** the scheduled time arrives (per my timezone)
**Then** the `notification` service queries the `pipeline.opportunities` records matched to my workspace since the last digest send
**And** it composes a branded HTML+text email with the top N opportunities sorted by relevance score.
**And** if there are zero matches in the window, no email is sent (no "empty digest" spam).
**And** every email includes a one-click unsubscribe link that toggles only that channel/event without disabling other notifications.
**And** delivery, opens, bounces, and complaints are logged for ops visibility.

---

### Story 9.2: External Calendar Sync

As a Pro+ user,
I want to sync opportunity deadlines and proposal tasks to my Google/Outlook calendar,
So that my schedule integrates with my daily workflow tools.

**FR coverage**: FR-41 (Post-MVP)
**NFR coverage**: NFR-6

**Acceptance Criteria:**

**Given** I am on Professional tier or higher
**When** I connect my Google or Outlook calendar via OAuth from settings
**Then** OAuth tokens (access + refresh) are stored encrypted at rest with an extra application-layer encryption key.
**Given** an opportunity I have saved (Story 3.5) or a task assigned to me (Story 7.5) has a due date
**When** the source date is set or changed in EU Solicit
**Then** an event is created or updated in my external calendar within 60s
**And** events are tagged with a stable EU Solicit identifier so duplicates are not created on retry.
**And** disconnecting the integration removes the stored tokens and stops further syncs (existing events remain on the user's calendar — not deleted).
**And** sync failures (revoked token, provider 5xx) are logged and surfaced as a banner in settings inviting re-auth.

## Implementation Notes

- The `notification` service consumes Redis Streams events emitted by other services (e.g., `OpportunityMatched`, `CommentMentioned`, `TaskAssigned`, `ApprovalRequested`).
- Digest scheduling uses a per-user cron driven by the user's timezone — not a global UTC cron — to deliver "morning of" feel.
- Outbound email uses an idempotency key per (user, digest_window) so retries cannot send twice.
- Calendar OAuth tokens are encrypted with `cryptography.fernet` keyed by a KMS-backed master; never stored in plaintext.
- Calendar sync is implemented as outbound-only for MVP+1; bidirectional sync is explicitly out of scope.
- All notification preferences and channel settings are part of the user's settings page; no settings live in service-specific UI.
