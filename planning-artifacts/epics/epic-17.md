## Epic 17: Slack & Teams Notifications

Enable workspace-level notification routing to Slack and Microsoft Teams to keep bid teams aligned on deadlines and new opportunities.
**FRs covered:** FR11.4, FR11.5

### Story 17.1: Incoming Webhook Configuration

As a Bid Team Lead,
I want to configure a Slack or Teams webhook URL for my workspace,
So that alerts can be routed to our project channel.

**Acceptance Criteria:**

**Given** a workspace on Professional tier or above
**When** the user provides a valid Slack/Teams webhook URL
**Then** the configuration is saved
**And** a test message can be successfully sent to the channel

### Story 17.2: Alert Triggering and Formatting

As a Bid Team Lead,
I want to receive formatted alerts for upcoming deadlines and new matched opportunities,
So that my team can respond quickly without constantly checking the platform.

**Acceptance Criteria:**

**Given** an active webhook configuration
**When** an opportunity reaches 48 hours before its deadline
**Then** a formatted message containing the opportunity name and a direct link is sent to the webhook
**And** the system ensures no duplicate notifications are sent for the same event
