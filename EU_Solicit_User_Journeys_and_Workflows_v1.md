# EU Solicit — User Journeys, UX Workflows & E2E Dataflows

**Platform**: EU Solicit | **Domain**: EUSolicit.com | **Version**: 1.0 | **Date**: 2026-04-04 | **Status**: Draft

**Companion documents**: Requirements Brief v4, Solution Architecture v3.1, Service Decomposition v1.1

---

## 1. User Personas

### 1.1 Persona: Free Explorer

| Attribute | Detail |
|---|---|
| **Name** | Maria — Junior Business Development Associate |
| **Company** | Small IT consultancy (8 employees), Sofia, Bulgaria |
| **Goal** | Evaluate whether EU Solicit is worth paying for by browsing available tenders |
| **Tier** | Free |
| **Key frustration** | Spends 3–4 hours weekly scanning AOP and TED manually; misses deadlines |
| **Success metric** | Finds a relevant opportunity within first session, understands what she'd get by upgrading |

### 1.2 Persona: Starter User

| Attribute | Detail |
|---|---|
| **Name** | Georgi — Owner-operator of a construction firm |
| **Company** | Regional construction firm (25 employees), Plovdiv, Bulgaria |
| **Goal** | Monitor Bulgarian public works tenders under €500K, get AI summaries to decide which to bid on |
| **Tier** | Starter (€29/mo) |
| **Key frustration** | Can't afford a bid consultant for every tender; often bids blind |
| **Success metric** | Uses AI summaries to pre-qualify 3–5 tenders/month; spends less time reading docs |

### 1.3 Persona: Professional User

| Attribute | Detail |
|---|---|
| **Name** | Elena — Bid Manager at a mid-size engineering firm |
| **Company** | Infrastructure engineering firm (120 employees), Sofia + Bucharest offices |
| **Goal** | Full proposal lifecycle — discovery through AI-drafted proposals and compliance checks across BG + RO + GR |
| **Tier** | Professional (€59/mo) |
| **Key frustration** | Preparing a competitive proposal takes 2–3 weeks and €5K–€15K in consultant fees |
| **Success metric** | Cuts proposal prep time by 50%; increases win rate from 15% to 25% |

### 1.4 Persona: Enterprise User

| Attribute | Detail |
|---|---|
| **Name** | Nikolai — Director of EU Programmes at a consulting firm |
| **Company** | Public sector consulting firm (50 consultants), managing 30+ client bids/year across all EU |
| **Goal** | White-label the platform for his clients; run unlimited proposals; API access for internal tools |
| **Tier** | Enterprise (€109+/mo) |
| **Key frustration** | Coordination across 10+ concurrent bids; no institutional memory between projects |
| **Success metric** | Manages all client bids from one dashboard; lessons from past bids auto-feed new proposals |

### 1.5 Persona: Platform Administrator

| Attribute | Detail |
|---|---|
| **Name** | Dimitar — Product Owner / Platform Admin |
| **Company** | EU Solicit (operator) |
| **Goal** | Configure compliance frameworks, monitor crawler health, oversee tenant subscriptions, manage white-label configs |
| **Tier** | Admin (no tier — full access) |
| **Key frustration** | Regulatory changes can invalidate compliance checks; must react quickly |
| **Success metric** | All active opportunities have correct compliance frameworks; crawlers run reliably on schedule |

---

## 2. User Journey Maps

### Journey 1: Free Discovery → Upgrade Conversion

**Persona**: Maria (Free Explorer)

```
Phase         │ Discover          │ Explore           │ Hit Paywall        │ Convert
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Action        │ Google "Bulgarian │ Browse opportunity │ Click on a tender  │ Click "Upgrade"
              │ tender monitoring"│ catalogue, filter  │ to see full docs   │ on upgrade prompt
              │ → lands on        │ by region, CPV     │ + AI summary       │ → Stripe Checkout
              │ eusolicit.com     │ sector, deadline   │                    │ → 14-day trial
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Touchpoint    │ Landing page      │ Opportunity list   │ Opportunity detail │ Stripe Checkout
              │ SEO / SSR         │ (limited metadata) │ (gated content)    │ Trial activation
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Emotion       │ Curious           │ Impressed by       │ Frustrated: can    │ Relieved: trial
              │                   │ coverage breadth   │ see the name but   │ is free, no card
              │                   │                    │ not the details    │ required
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
System        │ Next.js SSR page  │ Client API: tier-  │ Client API: 403   │ Client API →
              │ served via CDN    │ gated search       │ with upgrade_url   │ Stripe → webhook
              │                   │ returns Free       │ returns limited    │ → subscription
              │                   │ metadata only      │ response + CTA     │ created event
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
UX Req        │ UX-F01: Public    │ UX-F02: Clearly   │ UX-F03: Gated     │ UX-F04: Seamless
              │ landing with SEO  │ show what's hidden │ content must show  │ trial activation
              │ metadata for      │ behind paywall     │ a compelling       │ with zero-
              │ crawl tenders     │ (blur/lock icons)  │ upgrade prompt     │ friction
              │                   │ on restricted data │ with tier benefits │ onboarding wizard
```

**UX Requirements surfaced**:

- **UX-F01**: Public landing pages must be SSR for SEO. Tender titles and metadata visible to search engines.
- **UX-F02**: Free-tier opportunity list must visually indicate locked content (blur, lock icons, "Upgrade to view" badges) — not simply hide it. The user must feel the value they're missing.
- **UX-F03**: Gated content screen must show a contextual upgrade prompt: "This tender has 42 pages of requirements. Upgrade to Starter to read the AI summary." Include tier comparison inline.
- **UX-F04**: Trial activation completes in ≤3 clicks: "Start Free Trial" → email confirmation → redirect to full opportunity detail. No credit card required for 14-day Professional trial.
- **UX-F05** (supplementary): First-time login wizard collects company profile, CPV sector preferences, and region interests to pre-configure alert preferences and tier-gated filters.

---

### Journey 2: Starter — Opportunity Monitoring & AI Summaries

**Persona**: Georgi (Starter User)

```
Phase         │ Configure         │ Monitor            │ Triage             │ Decide
──────────────┼───────────────────┼────────────────────┼────────────────────┼──────────────────
Action        │ Set alert prefs:  │ Receive daily      │ Open opportunity   │ Read AI summary
              │ CPV 45 (constr.), │ email digest with  │ detail from email  │ → decide bid or
              │ Bulgaria, ≤500K   │ matching tenders   │ link               │ skip
──────────────┼───────────────────┼────────────────────┼────────────────────┼──────────────────
Touchpoint    │ Alert prefs UI    │ Email inbox        │ Opportunity detail │ AI Summary panel
              │ + iCal subscribe  │ (SendGrid)         │ page               │ (usage counter
              │                   │                    │                    │ shown: 3/10 used)
──────────────┼───────────────────┼────────────────────┼────────────────────┼──────────────────
Emotion       │ Empowered:        │ Satisfied: no      │ Focused: full docs │ Confident: can
              │ exactly my niche  │ more manual        │ available inline   │ decide in 5 min
              │                   │ portal scanning    │                    │ instead of 2 hrs
──────────────┼───────────────────┼────────────────────┼────────────────────┼──────────────────
System        │ Client API saves  │ Notification Svc   │ Client API returns │ Client API →
              │ prefs → publishes │ matches opps →     │ full opp data      │ AI Gateway →
              │ alert_preference  │ generates digest   │ (region/CPV/       │ KraftData
              │ .updated event    │ via SendGrid       │ budget checked)    │ Executive Summary
              │                   │                    │                    │ Agent → usage
              │                   │                    │                    │ meter incremented
──────────────┼───────────────────┼────────────────────┼────────────────────┼──────────────────
UX Req        │ UX-S01: CPV       │ UX-S02: Email      │ UX-S03: Tier      │ UX-S04: AI
              │ sector picker     │ digest must be     │ boundary is        │ summary panel
              │ with search +     │ mobile-responsive  │ invisible to       │ with usage counter
              │ autocomplete      │ with deep links    │ authorized users   │ and "upgrade for
              │                   │ to opp detail      │                    │ more" prompt at
              │                   │                    │                    │ limit
```

**UX Requirements surfaced**:

- **UX-S01**: CPV sector picker must support hierarchical search (e.g., type "45" → "Construction work" with sub-categories). Region picker shows Bulgaria's districts for Starter, EU countries for Pro/Enterprise.
- **UX-S02**: Email digest must be mobile-responsive HTML with deep links directly into the opportunity detail page (authenticated via magic link or session).
- **UX-S03**: When a Starter user accesses an opportunity within their tier limits, the experience should feel unlimited — no "you're on Starter" friction. Tier enforcement is invisible when within bounds.
- **UX-S04**: AI summary panel shows remaining usage count ("3 of 10 summaries used this month"). At limit, button switches to "Upgrade to Professional for 50/month".
- **UX-S05** (supplementary): iCal subscription URL displayed prominently on alerts settings page with copy-to-clipboard and QR code for mobile calendar apps.

---

### Journey 3: Professional — Full Proposal Lifecycle

**Persona**: Elena (Professional User)

```
Phase         │ Discover          │ Analyze           │ Bid Decision       │ Propose
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Action        │ Alert email →     │ Upload full tender│ Run Bid/No-Bid    │ Generate AI
              │ review matched    │ package → AI      │ decision agent →  │ proposal draft →
              │ opportunities     │ parses + extracts │ review scorecard  │ stream in real-
              │                   │ requirements      │ → approve bid     │ time via SSE
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Touchpoint    │ Email + Dashboard │ Document upload   │ Bid/No-Bid        │ Proposal editor
              │                   │ + analysis panel  │ scorecard modal   │ (Tiptap)
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Emotion       │ Excited: 3 new    │ Impressed: 200-   │ Confident: data-  │ Amazed: first
              │ tenders match     │ page doc parsed   │ driven decision   │ draft in 3 min
              │                   │ in 30 seconds     │ not gut feeling   │ not 3 weeks
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
System        │ Notification Svc  │ Client API →      │ Client API →      │ Client API →
              │ → email           │ S3 + ClamAV →     │ AI Gateway →      │ AI Gateway SSE
              │                   │ AI Gateway →      │ KraftData         │ → KraftData
              │                   │ KraftData Doc     │ Bid/No-Bid Agent  │ Proposal Workflow
              │                   │ Parser Agent      │ → result stored   │ → streamed to UI
```

```
Phase         │ Validate          │ Optimize          │ Export & Submit
──────────────┼───────────────────┼───────────────────┼──────────────────
Action        │ Run compliance    │ Run scoring       │ Export PDF/DOCX →
              │ checker against   │ simulator →       │ follow portal
              │ assigned          │ review per-       │ submission
              │ framework         │ criterion scores  │ instructions
              │                   │ → iterate         │
──────────────┼───────────────────┼───────────────────┼──────────────────
Touchpoint    │ Compliance        │ Scorecard view    │ Export dialog +
              │ report panel      │ with improvement  │ submission
              │                   │ suggestions       │ guide panel
──────────────┼───────────────────┼───────────────────┼──────────────────
Emotion       │ Reassured: no     │ Strategic: knows  │ Prepared: fully
              │ missing items     │ exactly where to  │ guided, just
              │                   │ improve           │ submit externally
──────────────┼───────────────────┼───────────────────┼──────────────────
System        │ Client API →      │ Client API →      │ Client API →
              │ AI Gateway →      │ AI Gateway →      │ doc export
              │ KraftData         │ KraftData Scoring │ (python-docx /
              │ Compliance        │ Simulator Agent   │ reportlab) →
              │ Checker Agent     │                   │ S3 download
──────────────┼───────────────────┼───────────────────┼──────────────────
UX Req        │ UX-P01            │ UX-P02            │ UX-P03
```

**UX Requirements surfaced**:

- **UX-P01**: Compliance report must show pass/fail per checklist item with direct links to the proposal section that needs fixing. Red/amber/green status badges.
- **UX-P02**: Scoring simulator scorecard must show per-criterion predicted score vs. maximum, with inline AI suggestions. Allow "re-score" after edits to see improvement.
- **UX-P03**: Export dialog must let user choose format (PDF/DOCX), include/exclude sections, and display a "Submission Guide" panel with portal-specific instructions (URL, form name, deadline countdown).
- **UX-P04** (supplementary): Proposal editor must support real-time SSE streaming — text appears progressively as the AI generates it, with a "Stop generation" button.
- **UX-P05** (supplementary): Bid/No-Bid scorecard must show: strategic fit score, win probability estimate, resource availability check, margin analysis, and an overall recommendation (Bid / No-Bid / Conditional). User override requires a text justification that is logged in the audit trail.
- **UX-P06** (supplementary): Document upload must show virus scan progress, file size validation (100MB limit), and supported format indicators. Multi-file upload with drag-and-drop.
- **UX-P07** (supplementary): Calendar events for tracked opportunities must appear in the sidebar — deadline, clarification period, site visit dates. Google/Outlook sync status indicator.
- **UX-P08** (supplementary): Version history panel on proposals showing diff between versions, with author attribution and timestamp.

---

### Journey 4: Enterprise — Multi-Client White-Label Operations

**Persona**: Nikolai (Enterprise User)

```
Phase         │ Configure         │ Onboard Client    │ Operate            │ Analyze
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Action        │ Set up white-     │ Create team       │ Run concurrent     │ Review cross-
              │ label subdomain   │ members for       │ proposals for      │ client analytics
              │ + branding +      │ client project,   │ multiple clients   │ + lessons learned
              │ API key           │ assign roles      │ across EU          │ + competitor view
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Touchpoint    │ Admin settings    │ Team management   │ Proposal dashboard │ Analytics hub
              │ + API docs        │ panel             │ (multi-tender)     │ + pipeline
              │                   │                   │                    │ forecasting
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Emotion       │ Professional:     │ Efficient: quick  │ In control:        │ Strategic:
              │ clients see       │ client setup      │ parallel bid mgmt  │ data-driven
              │ branded portal    │                   │ across projects    │ portfolio mgmt
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
System        │ Client API:       │ Client API: RBAC  │ All services at    │ Client API reads
              │ whitelabel_config │ + company + team   │ full capacity,     │ analytics views,
              │ stored in DB,     │ member CRUD       │ unlimited usage    │ AI Gateway runs
              │ subdomain DNS     │                   │ (no metering       │ Market Intel +
              │ via Cloudflare    │                   │ limits)            │ Competitor agents
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
UX Req        │ UX-E01            │ UX-E02            │ UX-E03             │ UX-E04
```

**UX Requirements surfaced**:

- **UX-E01**: White-label setup wizard: subdomain, logo upload, primary/accent colors, email sender domain. Preview mode before activation.
- **UX-E02**: Team management with role assignment (bid manager, technical writer, financial analyst, legal reviewer, read-only). Bulk invite via CSV or email list.
- **UX-E03**: Multi-tender dashboard showing all active bids with status (draft, in review, submitted, won, lost), deadlines as a Gantt-style timeline, and assignment by team member.
- **UX-E04**: Cross-client analytics with filterable views: win rate by sector, average score by CPV, revenue pipeline forecast, competitor overlap analysis. Exportable as PDF/DOCX reports.
- **UX-E05** (supplementary): API documentation page (auto-generated OpenAPI spec) accessible from the Enterprise settings panel. Includes API key management (create, rotate, revoke) with usage analytics.
- **UX-E06** (supplementary): Lessons Learned feed — when a bid outcome is recorded, the platform prompts the user to run the Lessons Learned Agent. Insights are stored and automatically referenced by the Proposal Generator Workflow for future bids.

---

### Journey 5: Platform Admin — Compliance & Operations

**Persona**: Dimitar (Platform Administrator)

```
Phase         │ Configure         │ Monitor           │ Intervene          │ Analyze
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Action        │ Assign compliance │ Check crawler     │ Regulation change  │ Review platform
              │ frameworks to new │ health dashboards │ detected → update  │ analytics: signup
              │ ingested opps     │ + agent quality   │ framework → re-    │ funnel, churn,
              │ (auto-suggested)  │ eval scores       │ validate affected  │ agent quality
              │                   │                   │ opportunities      │
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Touchpoint    │ Admin: Compliance │ Admin: Operations │ Admin: Compliance  │ Admin: Platform
              │ assignment queue  │ dashboard         │ framework editor   │ analytics
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
Emotion       │ Efficient: auto-  │ Confident: all    │ Urgent but in     │ Informed: clear
              │ suggestion handles│ systems green     │ control — can bulk │ view of business
              │ 90% of cases      │                   │ re-validate        │ health
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
System        │ Admin API: reads  │ Admin API: reads  │ Admin API: updates │ Admin API: reads
              │ pipeline opps →   │ Grafana metrics + │ compliance_frame-  │ aggregated
              │ AI Gateway runs   │ KraftData eval    │ works → publishes  │ analytics from
              │ Framework         │ scores            │ framework_assigned │ client + pipeline
              │ Suggestion Agent  │                   │ event → Client API │ schemas (read-
              │ → admin confirms  │                   │ refreshes          │ only)
──────────────┼───────────────────┼───────────────────┼────────────────────┼──────────────────
UX Req        │ UX-A01            │ UX-A02            │ UX-A03             │ UX-A04
```

**UX Requirements surfaced**:

- **UX-A01**: Compliance assignment queue: list of newly ingested opportunities without a framework. Each row shows the auto-suggested framework with a "Confirm" button and an "Override" dropdown. Bulk confirm for batches.
- **UX-A02**: Operations dashboard: crawler last-run time + status (green/amber/red), agent avg latency, eval quality scores, Redis Stream consumer lag, Celery queue depth. Alert badges for anomalies.
- **UX-A03**: Framework editor with versioning. When a framework is updated, show impacted opportunities count. "Re-validate all" button triggers batch compliance re-check via AI Gateway.
- **UX-A04**: Platform analytics: signup funnel (free → trial → paid), tier distribution, churn rate, agent usage heatmap (which agents are most/least used), revenue MRR/ARR. All filterable by date range.
- **UX-A05** (supplementary): Audit log viewer with full-text search, filterable by user, action type, entity, and date range. Export to CSV.
- **UX-A06** (supplementary): Tenant management view: all companies with subscription status, usage metrics, last login. Ability to impersonate a user for support (with audit logging).

---

## 3. End-to-End Workflows & Service-Level Dataflows

### Workflow 1: Opportunity Ingestion Pipeline

**Trigger**: Celery Beat schedule (configurable: immediate, daily, weekly per crawler)

```mermaid
sequenceDiagram
    participant Beat as Data Pipeline<br/>(Celery Beat)
    participant Worker as Data Pipeline<br/>(Celery Worker)
    participant GW as AI Gateway
    participant KD as KraftData
    participant DB as PostgreSQL
    participant Redis as Redis Streams
    participant Notif as Notification Svc
    participant Client as Client API

    Beat->>Worker: trigger crawl_opportunities task
    Worker->>Worker: AOP/TED/EU Grant crawler<br/>fetches raw data (HTTP)
    Worker->>GW: POST /teams/{normalization_team}/run<br/>{raw_data, source}
    GW->>KD: POST /client/api/v1/teams/{teamId}/run
    KD-->>GW: normalized opportunity data
    GW-->>Worker: normalized result
    Worker->>GW: POST /agents/{enrichment_agent}/run<br/>{normalized_data}
    GW->>KD: POST /client/api/v1/agents/{agentId}/run
    KD-->>GW: enriched data (summary, CPV, tags)
    GW-->>Worker: enriched result
    Worker->>DB: UPSERT pipeline.opportunities
    Worker->>Redis: publish "opportunities.ingested"<br/>{opportunity_ids, source}
    Redis-->>Notif: consume event (notification-svc group)
    Redis-->>Client: consume event (client-api group)
    Notif->>Notif: queue alert matching task
    Client->>Client: invalidate opportunity cache
```

**DB writes**: `pipeline.opportunities` (upsert), `pipeline.crawler_runs` (log), `gateway.agent_executions` (log), `shared.audit_log`

**Error handling**: If KraftData is unavailable (circuit breaker open), the worker stores raw data in `pipeline.enrichment_queue` for retry. Existing opportunity data remains available in Client API.

---

### Workflow 2: Alert Digest Generation & Delivery

**Trigger**: Celery Beat schedule (per user preference: immediate on `opportunities.ingested` event, or daily/weekly batch)

```mermaid
sequenceDiagram
    participant Redis as Redis Streams
    participant Notif as Notification Svc<br/>(Celery Worker)
    participant DB as PostgreSQL
    participant SG as SendGrid

    Redis->>Notif: consume "opportunities.ingested"<br/>{opportunity_ids}
    Notif->>DB: SELECT client.alert_preferences<br/>WHERE schedule = 'immediate'
    Notif->>DB: SELECT pipeline.opportunities<br/>WHERE id IN (ingested_ids)
    Notif->>Notif: match_opportunity()<br/>CPV codes, regions, budget range,<br/>keywords per user preference
    Notif->>SG: send digest email<br/>(matched opps per user)
    Notif->>DB: INSERT notification.alert_log<br/>{user_id, opp_ids, channel, status}
    Notif->>DB: INSERT notification.email_log
```

**Daily/weekly digests**: Celery Beat fires `generate_daily_digest` at 08:00 UTC and `generate_weekly_digest` on Monday 08:00 UTC. These tasks query all users with the respective schedule preference and aggregate matching opportunities since last digest.

**DB reads**: `client.alert_preferences`, `pipeline.opportunities`, `client.users` (email)
**DB writes**: `notification.alert_log`, `notification.email_log`

---

### Workflow 3: User Registration → Trial Activation

**Trigger**: User clicks "Sign Up" on eusolicit.com

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend<br/>(Next.js)
    participant API as Client API
    participant DB as PostgreSQL
    participant Stripe as Stripe
    participant Redis as Redis Streams

    User->>FE: Click "Sign Up" (email/password or Google OAuth2)
    FE->>API: POST /api/v1/auth/register<br/>{email, password, company_name}
    API->>DB: INSERT client.users, client.companies
    API->>Stripe: POST /v1/customers<br/>{email, company}
    Stripe-->>API: customer_id
    API->>DB: UPDATE client.companies<br/>SET stripe_customer_id
    API->>API: Generate JWT (RS256)
    API-->>FE: {access_token, refresh_token}
    FE->>FE: Redirect to onboarding wizard

    Note over FE,API: Onboarding Wizard (3 steps)
    User->>FE: Step 1: Company profile (sector, size, regions)
    FE->>API: PUT /api/v1/companies/{id}/profile
    API->>DB: UPDATE client.companies

    User->>FE: Step 2: CPV preferences + region interests
    FE->>API: POST /api/v1/alerts/preferences
    API->>DB: INSERT client.alert_preferences
    API->>Redis: publish "alert_preference.updated"

    User->>FE: Step 3: "Start 14-day Pro Trial" or "Stay Free"
    FE->>API: POST /api/v1/subscriptions/trial
    API->>Stripe: POST /v1/subscriptions<br/>{trial_period_days: 14, price: pro_monthly}
    Stripe-->>API: subscription_id (trialing)
    API->>DB: INSERT client.subscriptions<br/>{tier: professional, status: trialing}
    API->>Redis: publish "subscription.changed"<br/>{company_id, old_tier: free, new_tier: professional}
    API-->>FE: trial activated
    FE->>FE: Redirect to dashboard
```

**DB writes**: `client.users`, `client.companies`, `client.alert_preferences`, `client.subscriptions`, `shared.audit_log`

---

### Workflow 4: AI Summary Generation (Tier-Gated + Usage-Metered)

**Trigger**: User clicks "Generate AI Summary" on opportunity detail page

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant Redis as Redis
    participant GW as AI Gateway
    participant KD as KraftData

    User->>FE: Click "Generate AI Summary"
    FE->>API: POST /api/v1/opportunities/{id}/summary

    Note over API: Tier Gate check
    API->>API: TierGate.check_opportunity_access()<br/>region, CPV, budget vs user tier
    alt Tier denied
        API-->>FE: 403 {error, upgrade_url}
        FE->>FE: Show upgrade prompt modal
    end

    Note over API: Usage Gate check
    API->>Redis: INCR usage:ai_summaries:{company}:{period}
    alt Limit exceeded
        API->>Redis: DECR (rollback)
        API-->>FE: 429 {usage: {used, limit}, upgrade_url}
        FE->>FE: Show "limit reached" with upgrade CTA
    end

    API->>GW: POST /agents/{executive_summary}/run<br/>{opportunity_id, context}
    GW->>KD: POST /client/api/v1/agents/{agentId}/run
    KD-->>GW: {summary, key_risks, requirements_count}
    GW->>GW: Log to gateway.agent_executions
    GW-->>API: summary result
    API->>API: Cache summary in opportunity record
    API-->>FE: {summary, risks, usage: {used: 4, limit: 10}}
    FE->>FE: Render summary panel with usage counter
```

**DB reads**: `client.subscriptions`, `client.tier_access_policies`, `pipeline.opportunities`
**DB writes**: `gateway.agent_executions`, `shared.usage_meters`, `shared.audit_log`
**Redis**: Usage counter atomic INCR/DECR

---

### Workflow 5: Proposal Generation (SSE Streaming)

**Trigger**: User clicks "Generate Proposal Draft" on opportunity detail page (Professional/Enterprise)

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant Redis as Redis
    participant GW as AI Gateway
    participant KD as KraftData

    User->>FE: Click "Generate Proposal Draft"
    FE->>API: POST /api/v1/opportunities/{id}/proposals/generate

    Note over API: Tier Gate (Pro+) + Usage Gate (proposal_drafts)
    API->>Redis: INCR usage:proposal_drafts:{company}:{period}

    API->>GW: POST /workflows/{proposal_generator}/run-stream<br/>{opportunity_id, company_profile, template_id}
    GW->>KD: POST /client/api/v1/workflows/{workflowId}/run-stream

    Note over GW,FE: SSE stream proxied through AI Gateway → Client API → Frontend
    loop SSE chunks
        KD-->>GW: SSE: data chunk
        GW-->>API: SSE: data chunk
        API-->>FE: SSE: data chunk
        FE->>FE: Append text to Tiptap editor<br/>progressively
    end

    KD-->>GW: SSE: [DONE]
    GW->>GW: Log execution (gateway.agent_executions)
    GW-->>API: SSE: [DONE]
    API->>API: Store proposal draft
    API->>API: INSERT client.proposals,<br/>client.proposal_versions
    API-->>FE: SSE: {proposal_id, version: 1}
    FE->>FE: Enable editing, show "Save" button
```

**DB writes**: `client.proposals`, `client.proposal_versions`, `gateway.agent_executions`, `shared.usage_meters`, `shared.audit_log`
**Real-time UX**: Text streams into the editor character-by-character. "Stop generation" button sends abort signal. Progress indicator shows workflow stage (e.g., "Analyzing requirements... Drafting technical approach... Building timeline...").

---

### Workflow 6: Bid/No-Bid Decision

**Trigger**: User clicks "Run Bid/No-Bid Analysis" on opportunity detail page

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant GW as AI Gateway
    participant KD as KraftData
    participant DB as PostgreSQL

    User->>FE: Click "Bid/No-Bid Analysis"
    FE->>API: POST /api/v1/opportunities/{id}/bid-decision
    API->>GW: POST /agents/{bid_nobid}/run<br/>{opportunity, company_profile, past_bids}
    GW->>KD: POST /client/api/v1/agents/{agentId}/run
    KD-->>GW: {recommendation, score_breakdown,<br/>key_risks, win_probability}
    GW-->>API: decision result
    API-->>FE: scorecard data
    FE->>FE: Render scorecard modal:<br/>Strategic fit, Win probability,<br/>Resource availability, Margin

    alt User accepts recommendation
        User->>FE: Click "Accept: Bid" / "Accept: No-Bid"
        FE->>API: POST /api/v1/bid-decisions<br/>{opportunity_id, decision, ai_recommendation}
    else User overrides
        User->>FE: Click "Override" → enter justification
        FE->>API: POST /api/v1/bid-decisions<br/>{opportunity_id, decision, override: true,<br/>justification: "..."}
    end
    API->>DB: INSERT client.bid_decisions
    API->>DB: INSERT shared.audit_log<br/>(override flagged if applicable)
    API-->>FE: {decision_id, status: recorded}
```

**Audit trail**: Override decisions are prominently logged with the user's justification, AI recommendation, and score breakdown. The audit log entry includes before (AI recommendation) and after (user decision) values.

---

### Workflow 7: Compliance Check Against Assigned Framework

**Trigger**: User clicks "Run Compliance Check" on a proposal

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant Redis as Redis
    participant GW as AI Gateway
    participant KD as KraftData
    participant DB as PostgreSQL

    User->>FE: Click "Run Compliance Check"
    FE->>API: POST /api/v1/proposals/{id}/compliance-check

    Note over API: Usage Gate (compliance_checks)
    API->>Redis: INCR usage:compliance_checks:{company}:{period}

    API->>DB: SELECT admin.compliance_frameworks<br/>WHERE opportunity_id = opp_id
    Note over API: Load assigned framework for this opportunity

    API->>GW: POST /agents/{compliance_checker}/run<br/>{proposal, framework, opportunity}
    GW->>KD: POST /client/api/v1/agents/{agentId}/run
    KD-->>GW: {checklist: [{item, status, section_ref, fix_suggestion}]}
    GW-->>API: compliance result
    API->>DB: Store result in client.proposals (compliance_report JSONB)
    API-->>FE: {pass_count, fail_count, items: [...]}
    FE->>FE: Render compliance report:<br/>pass/fail per item, link to proposal section
```

**Key detail**: The compliance check validates against the specific framework assigned to the opportunity by the admin — not a generic ruleset. If no framework is assigned, the UI shows "Awaiting compliance framework assignment" with no check available.

---

### Workflow 8: Calendar Sync (Google Calendar)

**Trigger**: User connects Google Calendar (Professional/Enterprise) + periodic Celery Beat sync

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant Redis as Redis Streams
    participant Notif as Notification Svc
    participant Google as Google Calendar API

    Note over User,API: One-time OAuth connection
    User->>FE: Click "Connect Google Calendar"
    FE->>API: GET /api/v1/calendar/connect/google
    API-->>FE: Redirect to Google OAuth2 consent
    User->>Google: Authorize
    Google-->>API: OAuth callback with auth code
    API->>Google: Exchange code for tokens
    API->>API: Encrypt + store tokens
    API->>API: INSERT client.calendar_connections
    API->>Redis: publish "calendar.connected"<br/>{connection_id, user_id, provider: google}
    API-->>FE: Connection successful

    Note over Notif: Periodic sync (every 15 min)
    Notif->>Notif: Celery Beat: sync_calendars task
    Notif->>API: Read client.calendar_connections<br/>(via DB read-only access)
    Notif->>API: Read pipeline.opportunities<br/>tracked by user
    Notif->>Google: Create/Update calendar events<br/>(deadline, clarification, site visit)
    Notif->>Notif: UPDATE notification.sync_log
```

**DB reads**: `client.calendar_connections`, `client.calendar_events`, `pipeline.opportunities`
**DB writes**: `client.calendar_events`, `notification.sync_log`
**Token management**: Notification Service refreshes OAuth tokens before expiry; stores encrypted tokens in `client.calendar_connections`.

---

### Workflow 9: Stripe Subscription Lifecycle

**Trigger**: User upgrades, downgrades, or cancels subscription

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant Stripe as Stripe
    participant DB as PostgreSQL
    participant Redis as Redis Streams
    participant Notif as Notification Svc

    Note over User,Stripe: Upgrade: Free → Starter
    User->>FE: Click "Upgrade to Starter"
    FE->>API: POST /api/v1/subscriptions/checkout<br/>{tier: starter}
    API->>Stripe: POST /v1/checkout/sessions
    Stripe-->>API: checkout_url
    API-->>FE: redirect to Stripe Checkout
    User->>Stripe: Enter payment → confirm
    Stripe->>API: Webhook: checkout.session.completed
    API->>DB: INSERT client.subscriptions<br/>{tier: starter, status: active}
    API->>Redis: publish "subscription.changed"<br/>{company_id, old: free, new: starter}
    Redis-->>Notif: consume → send welcome email
    API-->>FE: subscription active

    Note over User,Stripe: Self-service management
    User->>FE: Click "Manage Subscription"
    FE->>API: POST /api/v1/subscriptions/portal
    API->>Stripe: POST /v1/billing_portal/sessions
    Stripe-->>API: portal_url
    API-->>FE: redirect to Stripe Customer Portal
    Note over User,Stripe: User can upgrade, downgrade,<br/>cancel, update payment method

    Note over Stripe,API: Usage metering sync
    Notif->>Notif: Celery Beat: daily usage sync
    Notif->>Redis: Read usage counters
    Notif->>Stripe: POST /v1/subscription_items/{id}/usage_records
```

---

### Workflow 10: Admin — Compliance Framework Assignment

**Trigger**: New opportunities ingested (auto-suggestion) or manual admin action

```mermaid
sequenceDiagram
    participant Redis as Redis Streams
    participant Admin as Admin API
    participant GW as AI Gateway
    participant KD as KraftData
    participant DB as PostgreSQL
    participant Client as Client API

    Note over Redis,Admin: New opportunities trigger
    Redis->>Admin: consume "opportunities.ingested"
    Admin->>DB: SELECT pipeline.opportunities<br/>WHERE compliance_framework_id IS NULL

    loop For each unassigned opportunity
        Admin->>GW: POST /agents/{framework_suggestion}/run<br/>{country, source, funding_type}
        GW->>KD: POST /client/api/v1/agents/{agentId}/run
        KD-->>GW: {suggested_framework_id, confidence}
        GW-->>Admin: suggestion result
        Admin->>DB: UPDATE pipeline.opportunities<br/>SET suggested_framework = result
    end

    Note over Admin: Admin reviews suggestion queue
    Admin->>Admin: Admin confirms or overrides
    Admin->>DB: INSERT admin.compliance_frameworks mapping
    Admin->>Redis: publish "compliance_framework.assigned"<br/>{opportunity_id, framework_id}
    Redis-->>Client: consume → refresh opportunity detail cache
    Admin->>DB: INSERT shared.audit_log
```

**Important**: Auto-suggestion runs automatically on ingestion, but final assignment always requires admin confirmation. This is a human-in-the-loop gate for regulatory correctness.

---

### Workflow 11: EU Grant Eligibility & Application Support

**Trigger**: User views an EU grant opportunity and clicks "Check Eligibility"

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant GW as AI Gateway
    participant KD as KraftData

    User->>FE: Click "Check Grant Eligibility"
    FE->>API: POST /api/v1/grants/{id}/eligibility
    API->>GW: POST /agents/{grant_eligibility}/run<br/>{grant_programme, company_profile}
    GW->>KD: POST /client/api/v1/agents/{agentId}/run
    KD-->>GW: {eligible: true/false, criteria_match: [...],<br/>gaps: [...], co_financing_rate}
    GW-->>API: eligibility result
    API-->>FE: render eligibility report

    Note over User,FE: If eligible, proceed to application support
    User->>FE: Click "Build Budget"
    FE->>API: POST /api/v1/grants/{id}/budget
    API->>GW: POST /agents/{budget_builder}/run<br/>{grant_programme, project_description}
    GW->>KD: Agent execution
    KD-->>GW: {budget_table, cost_categories,<br/>overhead_calc, co_financing_split}
    GW-->>API: budget result
    API-->>FE: render budget builder UI

    User->>FE: Click "Generate Logframe"
    FE->>API: POST /api/v1/grants/{id}/logframe
    API->>GW: POST /agents/{logframe_generator}/run<br/>{project_narrative, grant_programme}
    GW->>KD: Agent execution
    KD-->>GW: {logframe, work_packages,<br/>gantt_chart, deliverables}
    GW-->>API: logframe result
    API-->>FE: render logframe + Gantt view
```

---

### Workflow 12: Bid Outcome Recording & Lessons Learned Loop

**Trigger**: User records bid outcome after award decision

```mermaid
sequenceDiagram
    participant User as Browser
    participant FE as Frontend
    participant API as Client API
    participant GW as AI Gateway
    participant KD as KraftData
    participant DB as PostgreSQL

    User->>FE: Navigate to bid → "Record Outcome"
    FE->>FE: Outcome form: Won / Lost / Withdrawn<br/>+ optional: evaluator score, feedback
    User->>FE: Submit outcome
    FE->>API: POST /api/v1/bid-outcomes<br/>{opportunity_id, result, score, feedback}
    API->>DB: INSERT client.bid_outcomes
    API->>DB: INSERT shared.audit_log
    API-->>FE: outcome recorded

    Note over FE,API: Lessons Learned prompt
    FE->>FE: "Would you like to run Lessons Learned analysis?"
    User->>FE: Click "Yes"
    FE->>API: POST /api/v1/bid-outcomes/{id}/lessons-learned
    API->>GW: POST /agents/{lessons_learned}/run<br/>{proposal, outcome, feedback, past_outcomes}
    GW->>KD: POST /client/api/v1/agents/{agentId}/run
    KD-->>GW: {insights: [...], improvement_recs: [...],<br/>patterns: [...]}
    GW->>KD: Upload insights to Historical Bids Store<br/>POST /client/api/v1/storage-resources/{id}/files
    GW-->>API: lessons learned result
    API-->>FE: render insights panel
    FE->>FE: Display: "What worked", "What didn't",<br/>"Recommendations for next bid"
```

**Feedback loop**: The insights uploaded to KraftData's Historical Bids Store are automatically referenced by the Proposal Generator Workflow on future bids, closing the institutional memory loop.

---

## 4. Supplementary Business Requirements

The following requirements emerged during user journey mapping and are not explicitly covered in the Requirements Brief v4. They are recommended additions.

### 4.1 Onboarding & Activation

| ID | Requirement | Rationale | Priority |
|---|---|---|---|
| BR-S01 | First-login onboarding wizard (3 steps: company profile, CPV/region prefs, trial prompt) | Without guided setup, users won't configure alerts and won't see value quickly. High churn risk. | **Must-have** |
| BR-S02 | Deep links in email digests must authenticate via magic link or session token | Users abandon if forced to login after clicking an email link. Reduces friction to engagement. | **Must-have** |
| BR-S03 | Empty-state designs for every major view (no opportunities, no proposals, no outcomes) | Empty screens cause confusion. Guide users to the next action. | **Must-have** |

### 4.2 Upgrade & Conversion

| ID | Requirement | Rationale | Priority |
|---|---|---|---|
| BR-S04 | Contextual upgrade prompts with tier comparison at every paywall touchpoint | Generic "upgrade" buttons convert poorly. Show exactly what the user gains. | **Must-have** |
| BR-S05 | Trial expiry warning sequence: 3 days, 1 day, expired — via email + in-app banner | Users forget about trials. Timely reminders prevent churn at conversion point. | **Must-have** |
| BR-S06 | Downgrade path must clearly show what the user will lose (features, data retention) | Transparent downgrade reduces support tickets and builds trust. | **Should-have** |

### 4.3 Proposal & Workflow UX

| ID | Requirement | Rationale | Priority |
|---|---|---|---|
| BR-S07 | "Stop generation" button for SSE-streamed AI outputs | Users must be able to abort long AI generations. Essential for UX control. | **Must-have** |
| BR-S08 | Proposal version diff view (side-by-side or inline) with author attribution | Bid managers need to track what changed between iterations. | **Should-have** |
| BR-S09 | Compliance check unavailable state when no framework assigned to opportunity | Prevents confusing "no results" when the real issue is missing admin config. | **Must-have** |
| BR-S10 | Scoring simulator "re-score" after edits to show improvement | The value of the simulator is iterative improvement, not one-shot scoring. | **Should-have** |

### 4.4 Enterprise & Multi-User

| ID | Requirement | Rationale | Priority |
|---|---|---|---|
| BR-S11 | Multi-tender Gantt-style dashboard for Enterprise bid managers | Operating 10+ concurrent bids without a timeline view is unmanageable. | **Should-have** |
| BR-S12 | Bulk team invite via CSV or email list | Enterprise onboards entire consulting teams; one-by-one is impractical. | **Should-have** |
| BR-S13 | White-label preview mode before activation | Branding mistakes on live subdomains damage client trust. | **Must-have** |

### 4.5 Admin & Operations

| ID | Requirement | Rationale | Priority |
|---|---|---|---|
| BR-S14 | Compliance framework versioning with impact analysis | Regulatory changes affect live validations; admin needs to see blast radius. | **Must-have** |
| BR-S15 | Bulk framework confirmation for auto-suggested assignments | Processing 50+ new opportunities individually is impractical. | **Must-have** |
| BR-S16 | Admin user impersonation with full audit logging | Support team needs to see what the user sees without sharing credentials. | **Should-have** |

### 4.6 Data & Content

| ID | Requirement | Rationale | Priority |
|---|---|---|---|
| BR-S17 | Opportunity status lifecycle: Open → Closing Soon → Closed → Awarded | Users need to see deadline urgency at a glance. "Closing Soon" drives action. | **Must-have** |
| BR-S18 | Lessons Learned prompt after every bid outcome recording | Closing the feedback loop is the core differentiator; must be prompted, not buried. | **Must-have** |
| BR-S19 | iCal subscription URL with copy-to-clipboard and QR code | Calendar subscription is a high-value, low-effort feature — make it frictionless. | **Should-have** |

---

## 5. UX Requirements Registry

Consolidated list of all UX requirements identified in this document, cross-referenced to journeys.

| ID | Requirement | Journey | Persona |
|---|---|---|---|
| UX-F01 | Public landing pages with SSR for SEO metadata | J1 | Free |
| UX-F02 | Free-tier locked content indicators (blur, lock icons) | J1 | Free |
| UX-F03 | Contextual upgrade prompt on gated content | J1 | Free |
| UX-F04 | Zero-friction trial activation (≤3 clicks, no card) | J1 | Free |
| UX-F05 | First-login onboarding wizard (company, prefs, trial) | J1 | Free |
| UX-S01 | Hierarchical CPV sector picker with search | J2 | Starter |
| UX-S02 | Mobile-responsive email digest with deep links | J2 | Starter |
| UX-S03 | Invisible tier enforcement when within bounds | J2 | Starter |
| UX-S04 | AI summary usage counter with upgrade CTA at limit | J2 | Starter |
| UX-S05 | iCal URL with copy-to-clipboard and QR code | J2 | Starter |
| UX-P01 | Compliance report with pass/fail per item + section links | J3 | Professional |
| UX-P02 | Scoring simulator scorecard with re-score capability | J3 | Professional |
| UX-P03 | Export dialog with format choice + submission guide | J3 | Professional |
| UX-P04 | SSE streaming in proposal editor with stop button | J3 | Professional |
| UX-P05 | Bid/No-Bid scorecard with override justification | J3 | Professional |
| UX-P06 | Document upload with scan progress + drag-and-drop | J3 | Professional |
| UX-P07 | Calendar sidebar with sync status indicator | J3 | Professional |
| UX-P08 | Proposal version history with diff view | J3 | Professional |
| UX-E01 | White-label setup wizard with preview mode | J4 | Enterprise |
| UX-E02 | Team management with role assignment + bulk invite | J4 | Enterprise |
| UX-E03 | Multi-tender Gantt-style dashboard | J4 | Enterprise |
| UX-E04 | Cross-client analytics with export | J4 | Enterprise |
| UX-E05 | API documentation + key management panel | J4 | Enterprise |
| UX-E06 | Lessons Learned prompt on bid outcome recording | J4 | Enterprise |
| UX-A01 | Compliance assignment queue with bulk confirm | J5 | Admin |
| UX-A02 | Operations dashboard (crawlers, agents, queues) | J5 | Admin |
| UX-A03 | Framework editor with versioning + re-validate | J5 | Admin |
| UX-A04 | Platform analytics (funnel, churn, MRR, agent usage) | J5 | Admin |
| UX-A05 | Audit log viewer with search, filter, export | J5 | Admin |
| UX-A06 | Tenant management with impersonation | J5 | Admin |

---

## 6. Screen Inventory (Recommended)

Based on the journeys above, the following screens are needed for the MVP frontend.

### 6.1 Public (Unauthenticated)

| Screen | Purpose | SSR |
|---|---|---|
| Landing page | Product marketing, SEO entry point | Yes |
| Opportunity catalogue (public) | Free-tier browsable list with limited metadata | Yes |
| Login | Email/password + Google OAuth2 | No |
| Register | Account creation + onboarding wizard | No |
| Pricing page | Tier comparison with CTAs | Yes |

### 6.2 Client App (Authenticated — all tiers)

| Screen | Purpose | Tier Gate |
|---|---|---|
| Dashboard | Overview: tracked opps, upcoming deadlines, recent alerts, usage | All |
| Opportunity list | Searchable, filterable list with tier-appropriate data | All (gated detail) |
| Opportunity detail | Full tender data, documents, AI actions, compliance status | Paid |
| AI Summary panel | Executive summary with usage counter | Starter+ |
| Proposal editor | Tiptap rich text with SSE streaming, version history | Pro+ |
| Proposal list | All drafts with status, deadline, assignment | Pro+ |
| Bid/No-Bid scorecard | AI recommendation modal with override | Pro+ |
| Compliance report | Pass/fail checklist with section links | Pro+ |
| Scoring simulator | Per-criterion scorecard with re-score | Pro+ |
| EU Grant tools | Eligibility, budget builder, logframe generator | Pro+ |
| Export dialog | PDF/DOCX format selection + submission guide | Pro+ |
| Alert preferences | CPV, region, budget, schedule configuration | All |
| Calendar settings | iCal URL, Google/Outlook connection management | All (sync: Pro+) |
| Analytics dashboard | Market intel, ROI, competitor, pipeline, usage | Pro+ (limited: Starter) |
| Bid outcomes | Record and review past bid results | All |
| Lessons Learned | AI analysis of bid outcomes | Pro+ |
| Company profile | Credentials, CVs, certifications, team | All |
| Team management | Invite, roles, permissions | Pro+ (>1 user) |
| Subscription settings | Current plan, usage, upgrade/downgrade, Stripe portal | All |
| White-label settings | Subdomain, branding, email domain | Enterprise |
| API settings | API keys, docs, usage analytics | Enterprise |

### 6.3 Admin App (admin.eusolicit.com)

| Screen | Purpose |
|---|---|
| Admin dashboard | KPIs: active tenants, MRR, crawler status, agent health |
| Compliance assignment queue | Unassigned opps with auto-suggestions, bulk confirm |
| Framework editor | CRUD compliance frameworks with versioning |
| Crawler management | Enable/disable, scheduling, last-run status |
| Tenant list | All companies with subscription, usage, last login |
| Tenant detail | Company deep-dive with impersonation |
| Audit log | Full-text searchable, filterable, exportable |
| Platform analytics | Signup funnel, churn, tier distribution, agent usage |
| White-label management | All configured subdomains, status |
| Platform settings | Global config, rate limits, feature flags |

---

*Document version: 1.0 | Date: 2026-04-04 | Status: Draft*
