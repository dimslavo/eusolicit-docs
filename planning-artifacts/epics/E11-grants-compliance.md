# E11: EU Grant Specialization & Compliance

**Sprint**: 11–12 | **Points**: 55 | **Dependencies**: E04, E06, E07 | **Milestone**: MVP

## Goal

Deliver the EU grant application toolkit and the platform-wide compliance framework system, completing the MVP milestone. Users gain access to dedicated KraftData agents -- Grant Eligibility, Budget Builder, Consortium Finder, Logframe Generator, and Reporting Template Generator -- that transform a project idea into a submission-ready grant package with EU-compliant budgets, partner suggestions, logical frameworks, and post-award reporting templates. A structured ESPD profile system with AI-assisted auto-fill streamlines procurement tender participation. On the admin side, platform administrators manage compliance frameworks (national, EU, and programme-level) with structured validation rules, assign them per opportunity, review AI-suggested framework matches, and monitor regulatory changes via the Regulation Tracker Agent. All agent invocations flow through the AI Gateway (E04) to KraftData, and the Compliance Checker Agent integrated in E07 is reused here for framework-aware validation.

## Acceptance Criteria

- [ ] Grant Eligibility Agent maps a company profile against active EU programmes and returns matched programmes with eligibility scores
- [ ] Budget Builder Agent generates an EU-compliant budget with cost categories, overhead, and co-financing from project description and parameters
- [ ] Consortium Finder Agent searches the Consortium Partners Store and returns ranked partner suggestions with past participation history
- [ ] Logframe Generator Agent produces logical frameworks, work packages, Gantt chart data, and deliverable tables from project narrative
- [ ] Reporting Template Generator Agent pre-fills periodic report templates from awarded project data
- [ ] ESPD profiles can be created, edited, listed, and deleted with structured data mapped to ESPD XML schema (Parts II-V)
- [ ] ESPD Auto-Fill Agent maps company profile data to ESPD fields and returns a pre-filled ESPD downloadable as XML and PDF
- [ ] Admins can create, edit, activate/deactivate, and delete compliance frameworks with structured validation rules
- [ ] Admins can assign one or more compliance frameworks per opportunity, supporting hybrid national+EU scenarios
- [ ] Framework Suggestion Agent auto-suggests applicable frameworks when new opportunities are ingested; admins confirm or override
- [ ] Regulation Tracker Agent runs on a Celery Beat schedule, surfaces regulatory changes in the admin dashboard, and admins can acknowledge/dismiss changes
- [ ] All agent calls go through the AI Gateway with proper error handling, loading states, and timeout management
- [ ] Frontend provides dedicated pages for grant tools, ESPD management, and compliance administration with consistent UX

## Stories

### S11.01: ESPD Profile & Compliance Framework DB Schema + Migrations
**Points**: 2 | **Type**: backend

Create the `espd_profiles` table in the client schema with columns: id (UUID PK), company_id (FK), profile_name, espd_data (JSONB storing exclusion grounds, selection criteria, economic capacity, technical capacity mapped to ESPD Parts II-V), created_at, updated_at. Create `compliance_frameworks` in the admin schema with columns: id (UUID PK), name, description, country, regulation_type (enum: national/eu/programme), rules (JSONB for structured validation rules), is_active (boolean, default true), created_by, created_at, updated_at. Create `platform_settings` in the admin schema with columns: id (UUID PK), key (unique), value (JSONB), updated_at. Add `compliance_framework_id` (FK to admin.compliance_frameworks, nullable) to the existing pipeline `opportunities` table -- or if hybrid assignment is needed, create an `opportunity_compliance_frameworks` join table (opportunity_id, framework_id, assigned_by, assigned_at). Add indexes on company_id for espd_profiles, country and regulation_type for frameworks, and key for platform_settings. Write Alembic migrations and seed sample data for dev.

---

### S11.02: ESPD Profile CRUD API
**Points**: 2 | **Type**: backend

Implement REST endpoints for ESPD profiles: `POST /espd-profiles` (create with profile_name and structured espd_data), `GET /espd-profiles` (list all for current company), `GET /espd-profiles/:id` (detail view), `PATCH /espd-profiles/:id` (update profile_name or espd_data sections), `DELETE /espd-profiles/:id`. Enforce company-scoped RLS so users only access their own profiles. Validate espd_data structure against expected ESPD schema sections (Parts II through V). Return appropriate error codes for missing or cross-company access. Write unit and integration tests for each endpoint.

---

### S11.03: ESPD Auto-Fill Agent Integration
**Points**: 3 | **Type**: backend

Create `POST /espd-profiles/:id/auto-fill` that accepts an opportunity_id, loads the company ESPD profile and opportunity data, and sends them to the ESPD Auto-Fill Agent via AI Gateway. The agent maps company data to ESPD fields contextualised by opportunity requirements. Parse the agent response into a pre-filled ESPD data structure. Store the result as a new or updated profile snapshot. Implement `POST /espd-profiles/:id/export` with format param (`xml` or `pdf`): for XML, generate ESPD-compliant XML following the EU ESPD schema; for PDF, render a formatted ESPD document using reportlab. Return as streaming download. Write tests with mocked agent responses and validate exported XML structure.

---

### S11.04: Grant Eligibility Agent Integration
**Points**: 2 | **Type**: backend

Create `POST /grants/eligibility-check` that takes optional filter params (programme type, funding range) and sends the current company profile to the Grant Eligibility Agent via AI Gateway. The agent maps company attributes (size, sector, country, legal form, turnover) against active EU programmes (Horizon Europe, Digital Europe, structural funds, CAP, Interreg). Parse the response into a structured list of matched programmes with eligibility_score (0-100), programme_name, call_reference, requirements_summary, and gap_analysis. Return the results directly (stateless call, not persisted). Write tests with mocked agent responses covering full-match, partial-match, and no-match scenarios.

---

### S11.05: Budget Builder Agent Integration
**Points**: 3 | **Type**: backend

Create `POST /grants/budget-builder` that accepts project_description, duration_months, consortium_size, overhead_rate, target_programme, and total_requested_funding. Send to the Budget Builder Agent via AI Gateway. The agent generates an EU-compliant budget following programme-specific cost category rules (personnel, subcontracting, travel, equipment, other direct costs, indirect costs). Parse the response into a structured budget object with cost_categories (each with name, amount, justification), totals, overhead_calculation, co_financing_split (EU contribution vs own contribution), and per-partner breakdown if consortium. Return as JSON. Optionally persist budget snapshots linked to a proposal or project. Write tests validating budget arithmetic consistency (totals match line items, overhead correctly applied).

---

### S11.06: Consortium Finder Agent Integration
**Points**: 2 | **Type**: backend

Create `POST /grants/consortium-finder` that accepts project_description, required_capabilities (list), target_countries (optional list), and max_results. Send to the Consortium Finder Agent via AI Gateway, which searches the Consortium Partners Store (historical EU grant participation data). Parse the response into ranked partner suggestions, each with organisation_name, country, organisation_type, relevant_capabilities, past_projects (list of project names and programmes), collaboration_score, and contact_info. Return as paginated JSON. Write tests with mocked agent responses including edge cases (no matches, single-country filter, capability overlap ranking).

---

### S11.07: Logframe Generator & Reporting Template Agent Integrations
**Points**: 3 | **Type**: backend

Implement two endpoints:
1. `POST /grants/logframe-generate` -- accepts project_narrative (text) and target_programme. Sends to the Logframe Generator Agent via AI Gateway. Parse response into: logical_framework (objectives, activities, indicators, means_of_verification as structured table), work_packages (list with WP number, title, lead, description, deliverables, person-months), gantt_data (tasks with start/end month, dependencies suitable for chart rendering), deliverable_table (deliverable number, title, WP, type, dissemination level, due month). Return as JSON.
2. `POST /grants/reporting-template` -- accepts project_id (for won grants). Loads project data (milestones, budget, consortium from local DB). Sends to the Reporting Template Generator Agent via AI Gateway. Returns a pre-filled periodic report template as structured JSON. Implement `POST /grants/reporting-template/export` with format `docx` using python-docx to generate a downloadable report document.

Write tests for both endpoints with mocked agent responses.

---

### S11.08: Compliance Framework CRUD API (Admin)
**Points**: 2 | **Type**: backend

Implement admin-scoped REST endpoints: `POST /admin/compliance-frameworks` (create with name, description, country, regulation_type, rules JSONB), `GET /admin/compliance-frameworks` (list with filters on country, regulation_type, is_active; paginated), `GET /admin/compliance-frameworks/:id`, `PATCH /admin/compliance-frameworks/:id` (update any field), `DELETE /admin/compliance-frameworks/:id` (soft-delete by setting is_active to false, or hard-delete if unused). Validate rules JSONB structure against a defined schema (rule_id, criterion, check_type, threshold, description). Prevent deletion of frameworks currently assigned to active opportunities. Enforce admin-only access via role-based middleware. Write tests covering CRUD, validation, and deletion guard.

---

### S11.09: Framework Assignment & Auto-Suggestion API (Admin)
**Points**: 3 | **Type**: backend

Implement `POST /admin/opportunities/:id/compliance-frameworks` to assign one or more frameworks to an opportunity (supporting hybrid national+EU assignments). Implement `GET /admin/opportunities/:id/compliance-frameworks` to list assigned frameworks. Implement `DELETE /admin/opportunities/:id/compliance-frameworks/:fid` to remove an assignment.

Implement the auto-suggestion flow: when a new opportunity is ingested (hook into the pipeline ingestion event or provide `POST /admin/compliance-frameworks/suggest` for manual trigger), invoke the Framework Suggestion Agent via AI Gateway with opportunity metadata (country, source, funding_type, programme). Parse the response into suggested framework IDs with confidence scores. Store suggestions in a `framework_suggestions` queue table (opportunity_id, framework_id, confidence, status: pending/accepted/rejected, reviewed_by, reviewed_at). Implement `GET /admin/framework-suggestions` (list pending), `PATCH /admin/framework-suggestions/:id` (accept/reject with optional override framework_id). On acceptance, auto-assign the framework to the opportunity. Write tests for assignment CRUD, suggestion generation, and acceptance flow.

---

### S11.10: Regulation Tracker Agent & Platform Settings API (Admin)
**Points**: 2 | **Type**: backend

Implement the scheduled Regulation Tracker: create a Celery Beat task that triggers the Regulation Tracker Agent via AI Gateway on a configurable schedule (default weekly). The agent monitors changes to ZOP (Bulgarian public procurement law), EU directives, and programme rules. Parse results into regulatory_changes records (source, change_type, summary, affected_frameworks list, severity, detected_at). Store in a `regulatory_changes` table in the admin schema. Implement `GET /admin/regulatory-changes` (list with filters on status: new/acknowledged/dismissed, severity, date range), `PATCH /admin/regulatory-changes/:id` (acknowledge or dismiss, with optional notes). Link acknowledged changes to affected compliance frameworks for admin review.

Implement `GET /admin/platform-settings` and `PATCH /admin/platform-settings/:key` for managing configuration (regulation tracker schedule, auto-suggestion enabled/disabled, etc.). Enforce admin-only access. Write tests for the Celery task with mocked agent, CRUD endpoints, and settings management.

---

### S11.11: EU Grant Tools Frontend Page -- Eligibility & Budget Panels
**Points**: 3 | **Type**: frontend

Build the EU Grant Tools page at `/grants/tools` with a tabbed or card-based layout. Implement two panels:
1. **Grant Eligibility Panel**: trigger button ("Check Eligibility"), loading spinner during agent call, results displayed as a list of matched programmes sorted by eligibility score. Each programme card shows name, score (colour-coded: green >80, amber 50-80, red <50), call reference, and an expandable "View Details" section with requirements summary and gap analysis.
2. **Budget Builder Panel**: input form with fields for project description (textarea), duration (number), consortium size (number), overhead rate (percentage slider), target programme (dropdown). Submit triggers the agent. Results displayed as an editable table with EU cost categories as rows, columns for amount and justification, calculated totals row, co-financing split visualisation (stacked bar), and per-partner breakdown accordion if applicable. Table cells are editable for manual adjustments with recalculated totals.

Wire up React Query mutations and cache. Handle loading, error, and empty states.

---

### S11.12: EU Grant Tools Frontend -- Consortium Finder & Logframe Panels
**Points**: 3 | **Type**: frontend

Add two more panels to the Grant Tools page:
1. **Consortium Finder Panel**: input form with project description (textarea), required capabilities (tag input), target countries (multi-select), max results (number). Submit triggers the agent. Results displayed as partner cards in a grid: organisation name, country flag, organisation type badge, matched capabilities (highlighted chips), collaboration score bar, past projects list (collapsible), contact info. Cards are selectable to build a shortlist.
2. **Logframe Display Panel**: input form with project narrative (textarea) and target programme (dropdown). Results displayed as: structured table for the logical framework (objectives -> activities -> indicators -> means of verification), work package list (expandable cards with WP details), Gantt chart (horizontal bar chart using Recharts with tasks, durations, dependencies), deliverable table (sortable columns).

Add a **Reporting Template** tab: select a won project from dropdown, trigger generation, display pre-filled template with editable fields, and "Download DOCX" button.

---

### S11.13: ESPD Profile Management & Auto-Fill Frontend
**Points**: 3 | **Type**: frontend

Build the ESPD section at `/espd`:
1. **ESPD Profile List Page**: table listing all profiles with name, created date, last updated, and action buttons (edit, delete, set as default). "Create New Profile" button. Empty state with explanation of ESPD purpose.
2. **ESPD Profile Editor Page**: structured multi-step form mapped to ESPD Parts II-V. Part II: information about the economic operator (name, address, contact, registration). Part III: exclusion grounds (checkboxes with detail fields for each ground). Part IV: selection criteria (economic/financial standing, technical/professional ability with sub-fields). Part V: reduction of candidates. Each section has inline validation and help tooltips explaining the ESPD field. Save draft and submit actions.
3. **ESPD Auto-Fill Page**: select a saved profile from dropdown, select an opportunity from searchable dropdown, trigger auto-fill. Show a side-by-side preview: original profile on left, auto-filled result on right with changed fields highlighted. Accept/reject individual field changes. Download buttons for XML and PDF export. Loading and error states throughout.

---

### S11.14: Compliance Admin -- Framework Management Frontend
**Points**: 3 | **Type**: frontend

Build the compliance admin section at `/admin/compliance`:
1. **Framework List Page**: data table with columns for name, country, regulation type (badge), active status (toggle), created date, and action buttons (edit, deactivate, delete). Filters for country (dropdown), regulation type (multi-select), and status (active/inactive). Search bar for framework name. Pagination. "Create Framework" button.
2. **Framework Editor Page**: form with fields for name, description, country (searchable dropdown), regulation_type (radio: national/EU/programme). Rules editor: a dynamic form where admin adds validation rules, each with rule_id (auto-generated), criterion name, check_type (dropdown: contains/regex/threshold/boolean), threshold value (conditional on check_type), and description. Rules are displayed as an editable list with add/remove/reorder. Preview panel shows how a sample proposal would be validated against the current rules. Save and cancel actions.

---

### S11.15: Compliance Admin -- Assignment, Suggestions & Regulation Tracker Frontend
**Points**: 3 | **Type**: frontend

Build three admin sub-pages:
1. **Framework Assignment Page** (`/admin/compliance/assign`): two-column layout. Left: opportunity selector (searchable table/list with opportunity name, source, country). Right: assigned frameworks panel showing current assignments with remove button, and framework selector (dropdown with search) to add new assignments. Support multiple assignments per opportunity. Show assignment history with timestamps and assigned_by.
2. **Auto-Suggestion Queue** (`/admin/compliance/suggestions`): table of pending framework suggestions showing opportunity name, suggested framework, confidence score (colour-coded bar), and actions (Accept, Override with framework picker, Dismiss). Batch actions for processing multiple suggestions. Filter by confidence threshold and date range.
3. **Regulation Tracker Dashboard** (`/admin/compliance/regulations`): feed of detected regulatory changes, each card showing source, change type badge (new/amended/repealed), summary text, severity indicator (low/medium/high), detected date, and affected frameworks (linked chips). Actions: Acknowledge (with optional notes field), Dismiss. Filter by status (new/acknowledged/dismissed), severity, and date range. Acknowledged changes show a link to review and update affected frameworks.

---

### S11.16: E2E Integration Testing & Agent Error Handling Hardening
**Points**: 2 | **Type**: fullstack

Write end-to-end tests covering the critical user journeys:
1. Company runs grant eligibility check -> views matched programmes -> triggers budget builder -> views budget.
2. Company creates ESPD profile -> triggers auto-fill for an opportunity -> previews result -> exports as XML.
3. Admin creates compliance framework -> assigns to opportunity -> user's proposal is validated against it (reuses E07 compliance checker).
4. New opportunity ingested -> framework suggestion generated -> admin accepts -> framework assigned automatically.
5. Regulation tracker fires -> admin views changes -> acknowledges and reviews affected framework.

Harden agent error handling across all E11 endpoints: standardise timeout handling (30s default, configurable), retry logic (1 retry with exponential backoff for transient failures), graceful degradation messages when agents are unavailable, and consistent error response format. Verify error states render correctly in all frontend panels. Write tests for timeout, retry, and failure scenarios with mocked agent errors.
