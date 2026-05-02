# Epic 9: Notifications, Alerts, Calendar

Users stay informed of bid deadlines and team activity via timely, idempotent alerts spanning email, in-app notifications, and calendar sync.

### Story 9.1: Daily Tender Digest
As a Bid Manager,
I want a daily digest at my chosen hour,
So that I am aware of the latest opportunities matching my criteria.

**Acceptance Criteria:**
**Given** the daily digest schedule
**When** the notification worker runs
**Then** it queries matches scored above threshold and sends localized SendGrid emails
**And** dispatch is idempotent and honors user opt-outs

### Story 9.2: Calendar Sync (Google / Outlook)
As a Pro user,
I want deadlines on my calendar,
So that I never miss a bid submission date.

**Acceptance Criteria:**
**Given** I have linked Google Calendar or Outlook
**When** a deadline is updated
**Then** the sync updates the external calendar via OAuth tokens encrypted with Fernet
**And** an iCal feed is also available as a fallback

### Story 9.3: Slack / Teams Notifications
As a team channel,
I want bid updates posted in real time,
So that the whole team can collaborate.

**Acceptance Criteria:**
**Given** a connected Slack or Teams channel
**When** a configured event occurs
**Then** a signed payload is sent to the webhook with retry+breaker logic
**And** inbound webhooks use ECDSA validation and cross-user deep links return 404 on mismatch
