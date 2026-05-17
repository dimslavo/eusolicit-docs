---
title: "E11 Endpoint Internal Call-Path Swap"
epic: 11
story_id: "11-22"
status: "ready-for-dev"
points: 5
---

### Description

This story covers the internal refactoring of the 8 grant and compliance agent endpoints to use the new `sirmaai-gateway` service instead of the legacy `ai-gateway`. The public-facing API contracts of these endpoints will remain unchanged.

The core task is to replace all instances of `AiGatewayClient.run_agent("logical_name", payload)` with the new invocation pattern: `SirmaaiGatewayClient.call_agent(logical_name, company_id, payload)`. This ensures that all agent-related traffic is routed through the SirmaAI infrastructure, leveraging the per-tenant Project isolation and agent versioning capabilities introduced in the SirmaAI pivot.

### Acceptance Criteria

1.  **Internal Call Path Refactoring**:
    *   All 8 backend endpoints related to grant and compliance agents (ESPD Auto-Fill, Grant Eligibility, etc.) must be updated.
    *   The internal call to `AiGatewayClient.run_agent()` must be replaced with a call to `SirmaaiGatewayClient.call_agent()`.
    *   The `logical_name` of the agent and the `company_id` for tenant context must be passed correctly to the new service.

2.  **Integration Test Updates**:
    *   All existing integration tests for the 8 affected endpoints must be updated.
    *   Mocks for `AiGatewayClient` must be replaced with mocks for `SirmaaiGatewayClient`.
    *   Tests must be updated to mock responses based on SirmaAI Project agent IDs instead of logical agent names.

3.  **No Public Contract Changes**:
    *   A diff of the OpenAPI specification before and after the changes must show no modifications to the public request/response schemas of the 8 affected endpoints.
    *   All existing E2E tests for these features must continue to pass without modification.

4.  **Cross-Tenant Isolation**:
    *   A negative integration test must be added to verify that an agent call for Company A executes only within Company A's SirmaAI Project and cannot access data or agents from Company B.

5.  **Configuration & Dependencies**:
    *   The `sirmaai-gateway` client must be correctly injected into the services that host the grant and compliance endpoints.
    *   All legacy `ai-gateway` client dependencies that are no longer needed should be removed.

### Dev Notes

*   This is primarily a backend refactoring task. No frontend changes are expected.
*   The 8 endpoints to be refactored are those created in stories S11.03, S11.04, S11.05, S11.06, S11.07 (two endpoints), S11.09, and S11.10.
*   The `company_id` is crucial for the `sirmaai-gateway` to resolve the correct SirmaAI Project and its associated agents. Ensure this is being passed correctly from the user's session or context.
*   Pay close attention to the mocking strategy in the integration tests. The new mocks should simulate the behavior of the `sirmaai-gateway` and the underlying SirmaAI platform.

### Files to be Created

*   None

### Files to be Modified

*   `services/client-api/src/client_api/services/espd_service.py` (or similar)
*   `services/client-api/src/client_api/services/grant_service.py` (or similar)
*   `services/admin-api/src/admin_api/services/compliance_service.py` (or similar)
*   `services/client-api/tests/integration/test_espd_*.py`
*   `services/client-api/tests/integration/test_grant_*.py`
*   `services/admin-api/tests/integration/test_compliance_*.py`
