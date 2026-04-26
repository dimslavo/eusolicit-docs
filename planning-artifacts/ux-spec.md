# UX Design Specification - EU Solicit

**Project:** EU Solicit
**Date:** 2026-04-26
**Status:** Initial Draft

## 1. Key User Personas

### 1.1 Bid Manager
* **Role:** Oversees the bidding process, assigns tasks, and ensures compliance.
* **Goals:** Efficiently discover matching tenders, track team progress, and validate final proposals against frameworks like ZOP.
* **UX Needs:** High-level dashboard, digest configurations, clear pass/fail compliance visual indicators.

### 1.2 Contributor (Subject Matter Expert / Technical Writer)
* **Role:** Drafts specific sections of the proposal.
* **Goals:** Generate high-quality technical content quickly using AI assistance.
* **UX Needs:** Real-time AI generation feedback (streaming text), focused text-editing interface, easy access to extracted requirements and summaries.

### 1.3 Reviewer
* **Role:** Evaluates the generated proposals and summaries.
* **Goals:** Quickly digest 200+ page tenders to make informed reviews.
* **UX Needs:** Split-screen or side-by-side view comparing AI-generated executive summaries against the original document, easy commenting or flagging.

### 1.4 Company Admin
* **Role:** Manages the subscription, billing, and team access.
* **Goals:** Monitor usage, manage role-based access control (RBAC), handle payments.
* **UX Needs:** Clear subscription management portal, user management tables with role dropdowns, audit log access.

---

## 2. Primary User Journeys

### 2.1 Opportunity Discovery & Triage (Bid Manager)
1. **Notification:** Bid Manager receives a daily email digest containing a list of matched tenders.
2. **Review:** User clicks a tender link, landing on the **Tender Detail Page**.
3. **Assessment:** User views the AI-generated executive summary, extracted checklist, and risk analysis.
4. **Decision:** User navigates to the Bid/No-Bid Decision Matrix component.
5. **Action:** User overrides or confirms the AI recommendation and clicks "Proceed to Bid", which transitions the opportunity state and alerts Contributors.

### 2.2 AI-Assisted Proposal Generation (Contributor)
1. **Access:** Contributor logs in and navigates to an active bid workspace.
2. **Drafting:** User selects a specific section to draft and clicks "Generate Draft".
3. **Real-time Feedback:** An empty state transitions to a loading state, followed by streamed text rendering immediately via Server-Sent Events (SSE). A progress/status indicator shows the connection state.
4. **Refinement:** The text stream completes. User edits the generated draft in the WYSIWYG editor.
5. **Save:** User saves the section.

### 2.3 Compliance Validation (Bid Manager / Reviewer)
1. **Trigger Check:** After proposal compilation, the Bid Manager clicks "Run Compliance Check".
2. **Processing:** System displays a scanning animation.
3. **Review Results:** A Compliance Dashboard displays pass/fail criteria based on assigned frameworks (e.g., ZOP). Failed checks are highlighted in red with AI-suggested improvements.
4. **Action:** User addresses the failures by modifying the proposal text or assigning it back to a Contributor.

---

## 3. Key Screens and States

### 3.1 Dashboard (Home)
* **Components:** 
  * Metrics cards (Active Bids, Win Rate, Tenders Matched).
  * Recent activity feed / Audit Log preview.
  * Upcoming deadlines calendar snippet.
* **States:** 
  * Empty state (no matched tenders yet).
  * Loading state with skeleton loaders (`<QueryGuard>` implementation).

### 3.2 Tender Detail & Document Analysis View
* **Components:**
  * Header with key tender metadata (Budget, Deadline, CPV codes).
  * Tabbed interface: Executive Summary, Extracted Checklist, Original Documents, Risk Analysis.
  * File upload dropzone for adding new supplementary documents.
* **States:**
  * File Upload State: Progress bar indicating upload percentage, followed by a "Scanning for viruses..." indicator.
  * Over-limit State: Error banner if file exceeds 100MB or package exceeds 500MB.

### 3.3 Proposal Editor (Streaming Interface)
* **Components:**
  * Document outline sidebar.
  * Main text editor.
  * AI Assistant panel / floating action button.
* **States:**
  * Generating State: Text typing effect with a visual indicator that the stream is active.
  * Timeout/Error State: Graceful degradation displaying "Connection interrupted. Partial draft saved." if the SSE stream fails.

### 3.4 Subscription & Account Management
* **Components:**
  * Pricing tier comparison (Free, Starter, Pro, Enterprise).
  * "Upgrade to Pro" call-to-action button triggering Stripe checkout.
  * Role management table.
* **States:**
  * Pending Payment: Polling or waiting for webhook confirmation before unlocking features.

---

## 4. Accessibility & UI/UX Considerations

* **Real-time Feedback & SSE:** Since proposal generation uses `run-stream` (Server-Sent Events), the UI must smoothly handle incoming chunks without blocking the main thread or causing layout shift. A blinking cursor or typing indicator should be used.
* **Error Handling & Fetching:** All data fetching must be wrapped in the `<QueryGuard>` component to ensure consistent loading and error states across the application.
* **Forms & Validation:** All input forms must utilize `useZodForm` with `<FormField>` wrappers to provide immediate, accessible inline validation errors (especially for password length and email format).
* **Keyboard Navigation:** Crucial for the Proposal Editor and Dashboard tables to support power users (Bid Managers).
* **Color & Contrast:** Ensure compliance visual indicators (Pass/Fail) use both color (Green/Red) and iconography (Checkmark/Cross) to accommodate color blindness.
* **Responsive Design:** While the primary use case is desktop-based due to document complexity, key dashboard metrics and alerts must be responsive for mobile/tablet viewing.
