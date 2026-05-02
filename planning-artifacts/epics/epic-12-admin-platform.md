# Epic 12: Admin Platform

System operators securely manage platform-wide operations, monitor KPIs, and curate data to ensure ongoing health.

### Story 12.1: Tenant Management
As an Operator,
I manage companies and tiers,
So that I can support customers effectively.

**Acceptance Criteria:**
**Given** I am logged into the internal admin portal
**When** I access tenant management
**Then** I can list, suspend, reactivate, and adjust tiers
**And** the admin portal is protected by a VPN/IP allowlist and actions are audited

### Story 12.2: KPI Dashboards
As a leader,
I want live business metrics,
So that I can monitor the platform's performance and unit economics.

**Acceptance Criteria:**
**Given** I am in the admin portal
**When** I view the dashboards
**Then** Recharts charts display MRR, active companies, AI cost ratio, and crawler health
**And** the data is refreshed via scheduled materialized views
