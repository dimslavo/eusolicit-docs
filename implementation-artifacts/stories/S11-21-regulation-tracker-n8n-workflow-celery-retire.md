# S11.21: Regulation Tracker N8N workflow + Celery retire

**Epic:** [E11: EU Grant Specialization & Compliance](https://github.com/aifork-search/eusolicit-docs/blob/main/planning-artifacts/epics/E11-grants-compliance.md)

**Type:** `backend` + `workflow`
**Points:** 3
**Last Updated:** 2026-05-16
**Status:** `proposed`

## Description

As part of the migration to the SirmaAI architecture, replace the Celery Beat-scheduled regulation tracker with a `regulation-tracker-v1` N8N workflow template. The user-facing admin dashboard surface will remain unchanged, but the underlying scheduling and execution mechanism will shift from a project-specific Celery task to a standardized, organization-shared N8N cron workflow.

## Acceptance Criteria

- [ ] The existing Celery Beat task configuration for the Regulation Tracker agent is identified and removed from the codebase (likely in `services/data-pipeline` or a shared Celery configuration).
- [ ] A new N8N workflow, `regulation-tracker-v1`, is designed and implemented.
- [ ] The N8N workflow is configured to run on a recurring cron schedule (e.g., weekly), matching the behavior of the old Celery task.
- [ ] The workflow authenticates with the `ai-gateway` (or `sirmaai-gateway`) and successfully invokes the `regulation-tracker` logical agent, passing the necessary `company_id` or context for a platform-wide check.
- [ ] The agent's output (a list of detected regulatory changes) is correctly parsed and stored in the `admin.regulatory_changes` PostgreSQL table.
- [ ] The data schema and format of the records inserted into `regulatory_changes` are identical to the previous implementation, ensuring no breaking change for the admin dashboard.
- [ ] Existing integration tests for the Regulation Tracker admin dashboard (`GET /admin/regulatory-changes`) continue to pass without modification to the frontend or the API endpoint itself.
- [ ] The old Celery task code is fully removed, and any associated dependencies are cleaned up if no longer needed.
- [ ] Documentation for scheduling system maintenance is updated to refer to N8N instead of Celery Beat for this task.

## Implementation Plan

1.  **Locate & Analyze:** Find the current Celery Beat task definition for the Regulation Tracker. It's likely in `eusolicit-app/services/data-pipeline/` or a related service. Note its schedule and the exact agent invocation it performs.
2.  **Deactivate Celery Task:** Comment out or remove the Celery Beat schedule entry to prevent it from running during the migration.
3.  **Create N8N Workflow:**
    -   Create a new workflow in N8N named `regulation-tracker-v1`.
    -   Add a Cron trigger node, configured to the same schedule as the old Celery task.
    -   Add an HTTP Request node to call the `sirmaai-gateway` to invoke the `regulation-tracker` agent. Ensure authentication is handled correctly (e.g., using an API key stored in N8N credentials).
    -   Add a PostgreSQL node (or another HTTP Request node to call a `data-pipeline` endpoint) to insert the parsed results from the agent into the `admin.regulatory_changes` table.
4.  **Test Workflow:** Manually trigger the N8N workflow and verify:
    -   It completes successfully.
    -   The `regulation-tracker` agent is invoked (check `ai-gateway` logs).
    -   New records appear in the `admin.regulatory_changes` table.
5.  **Clean Up:** Once the N8N workflow is confirmed to be working, permanently remove the old Celery task code, its schedule definition, and any related helper functions.
6.  **Verification:** Run the `eusolicit-app` test suite, especially any tests related to the admin compliance dashboard, to ensure no regressions have been introduced. `make test-service SVC=admin-api` should be relevant.

## Definition of Done

- All acceptance criteria met.
- `make lint` and `make type-check` pass.
- Relevant integration tests pass.
- The `sprint-status.yaml` file is updated to reflect this story's completion.
