# ATDD Checklist for Story 11.23: KB-Grounded ESPD + Grant Tests

**Story:** [11.23: KB-Grounded ESPD + Grant Tests](https://github.com/example/eusolicit/issues/1123)
**Epic:** E11 Amendment - AI Agent Compliance and Security
**Generated:** 2026-05-17
**Test Type:** Backend Integration (`pytest`)

This document outlines the failing acceptance tests required to satisfy the story's requirements. These tests should be implemented in `services/client-api/tests/integration/test_kb_grounded_grant_compliance.py`.

## Test Plan

The tests will verify that the ESPD Auto-Fill and Grant Eligibility SirmaAI agents correctly consume documents from the tenant's Knowledge Base (KB) and surface citations.

-   **Test Level:** Integration (Service + Database + AI Agent mocks)
-   **Test Framework:** `pytest`
-   **Fixtures:**
    -   `db_session`: For database interactions.
    -   `respx_mock`: To mock outbound calls from SirmaAI agents to the actual AI providers.
    -   `tenant_kb_factory`: A new factory to create and manage KB files for a tenant.
    -   `espd_template_artefact`: A new fixture providing a synthetic ESPD template file.
    -   `company_profile_artefact`: A new fixture providing a synthetic company profile file.

---

## Acceptance Criteria & Failing Tests

### AC1: ESPD Auto-Fill with KB Artefact

> Integration test suite at `services/client-api/tests/integration/test_kb_grounded_grant_compliance.py` covering: ESPD Auto-Fill with uploaded ESPD-template artefact in KB → output cites the template.

-   [ ] **Test Case 1.1: `test_espd_autofill_with_kb_template_cites_source`**
    -   **Given** a tenant with an ESPD template uploaded to their Knowledge Base.
    -   **When** the ESPD Auto-Fill agent is executed for that tenant.
    -   **Then** the agent's response should contain a `kb_citations` list.
    -   **And** the citation must reference the `file_id` of the uploaded ESPD template.

```python
# tests/integration/test_kb_grounded_grant_compliance.py

import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

# Placeholder for real factories and fixtures
from tests.fixtures import CompanyFactory, UserFactory, SirmaAIKBFileFactory

@pytest.mark.integration
async def test_espd_autofill_with_kb_template_cites_source(
    client: AsyncClient,
    db_session: AsyncSession,
    user_factory: UserFactory,
    company_factory: CompanyFactory,
    sirmaai_kb_file_factory: SirmaAIKBFileFactory,
    respx_mock, # To mock the AI provider call
):
    """
    Verify that the ESPD Auto-Fill agent cites the KB template it used.
    """
    pytest.fail("Test not implemented: AC1")
    # 1. GIVEN: Create a company and a user
    # 2. GIVEN: Upload a synthetic ESPD template to the tenant's KB using sirmaai_kb_file_factory
    #    - kb_file = await sirmaai_kb_file_factory(company_id=company.id, file_path="path/to/espd_template.txt")
    # 3. GIVEN: Mock the AI provider to return a response indicating grounding was used.
    # 4. WHEN: Make a request to the client-api endpoint that triggers the ESPD Auto-Fill agent.
    # 5. THEN: Assert the API response status is 200.
    # 6. THEN: Assert the response body contains a `kb_citations` array.
    # 7. AND: Assert that the `file_id` in the citation matches kb_file.id.
```

### AC2: Grant Eligibility with KB Artefact

> Grant Eligibility with uploaded company-profile artefact in KB → output cites profile sectors.

-   [ ] **Test Case 2.1: `test_grant_eligibility_with_kb_profile_cites_source`**
    -   **Given** a tenant with a company profile document uploaded to their Knowledge Base.
    -   **When** the Grant Eligibility agent is executed for that tenant.
    -   **Then** the agent's response should contain a `kb_citations` list.
    -   **And** the citation must reference the `file_id` of the uploaded company profile.

```python
# tests/integration/test_kb_grounded_grant_compliance.py

@pytest.mark.integration
async def test_grant_eligibility_with_kb_profile_cites_source(
    client: AsyncClient,
    db_session: AsyncSession,
    user_factory: UserFactory,
    company_factory: CompanyFactory,
    sirmaai_kb_file_factory: SirmaAIKBFileFactory,
    respx_mock,
):
    """
    Verify that the Grant Eligibility agent cites the company profile from the KB.
    """
    pytest.fail("Test not implemented: AC2")
    # 1. GIVEN: Create a company and a user.
    # 2. GIVEN: Upload a synthetic company profile to the tenant's KB.
    #    - kb_file = await sirmaai_kb_file_factory(company_id=company.id, file_path="path/to/company_profile.txt")
    # 3. GIVEN: Mock the AI provider to return a response indicating grounding.
    # 4. WHEN: Make a request to the client-api endpoint that triggers the Grant Eligibility agent.
    # 5. THEN: Assert the API response status is 200.
    # 6. THEN: Assert the response body contains a `kb_citations` array.
    # 7. AND: Assert that the `file_id` in the citation matches kb_file.id.
```

### AC3: ESPD Auto-Fill without KB Artefact

> ESPD Auto-Fill with NO ESPD templates in KB → output proceeds but surfaces `caveat="no_kb_template"`.

-   [ ] **Test Case 3.1: `test_espd_autofill_without_kb_template_returns_caveat`**
    -   **Given** a tenant with an empty Knowledge Base.
    -   **When** the ESPD Auto-Fill agent is executed.
    -   **Then** the agent's response should proceed without error.
    -   **And** the response should contain a `caveat` field with the value `"no_kb_template"`.
    -   **And** the `kb_citations` list should be empty.

```python
# tests/integration/test_kb_grounded_grant_compliance.py

@pytest.mark.integration
async def test_espd_autofill_without_kb_template_returns_caveat(
    client: AsyncClient,
    db_session: AsyncSession,
    user_factory: UserFactory,
    company_factory: CompanyFactory,
    respx_mock,
):
    """
    Verify ESPD agent returns a caveat when no KB template is available.
    """
    pytest.fail("Test not implemented: AC3")
    # 1. GIVEN: Create a company and a user, ensure KB is empty for this tenant.
    # 2. GIVEN: Mock the AI provider to return a standard response.
    # 3. WHEN: Trigger the ESPD Auto-Fill agent.
    # 4. THEN: Assert the API response status is 200.
    # 5. THEN: Assert the response body contains `caveat: "no_kb_template"`.
    # 6. AND: Assert the response body's `kb_citations` is an empty list.
```

### AC4: Grant Eligibility without KB Artefact

> Grant Eligibility with NO profile in KB → output uses the structured profile fields only + surfaces appropriate caveat.

-   [ ] **Test Case 4.1: `test_grant_eligibility_without_kb_profile_returns_caveat`**
    -   **Given** a tenant with an empty Knowledge Base but with a structured company profile (in the database).
    -   **When** the Grant Eligibility agent is executed.
    -   **Then** the agent's response should proceed without error, using the structured data.
    -   **And** the response should contain a `caveat` field indicating no KB profile was used.
    -   **And** the `kb_citations` list should be empty.

```python
# tests/integration/test_kb_grounded_grant_compliance.py

@pytest.mark.integration
async def test_grant_eligibility_without_kb_profile_returns_caveat(
    client: AsyncClient,
    db_session: AsyncSession,
    user_factory: UserFactory,
    company_factory: CompanyFactory,
    respx_mock,
):
    """
    Verify Grant Eligibility agent returns a caveat when no KB profile is available.
    """
    pytest.fail("Test not implemented: AC4")
    # 1. GIVEN: Create a company with structured profile data (e.g., sectors) and a user.
    # 2. GIVEN: Ensure the tenant's KB is empty.
    # 3. GIVEN: Mock the AI provider to confirm it receives the structured data.
    # 4. WHEN: Trigger the Grant Eligibility agent.
    # 5. THEN: Assert the API response status is 200.
    # 6. THEN: Assert the response body contains an appropriate caveat (e.g., `caveat: "no_kb_profile"`).
    # 7. AND: Assert the response body's `kb_citations` is an empty list.
```

### AC5: New Test Fixtures

> Test fixtures for KB artefacts: small synthetic ESPD template + synthetic company profile artefact committed at `services/client-api/tests/data/fixtures/kb_artefacts/`.

-   [ ] **Fixture 5.1: Create `espd_template.txt` fixture file.**
    -   A small, synthetic ESPD-like text file should be created at `services/client-api/tests/data/fixtures/kb_artefacts/espd_template.txt`.
-   [ ] **Fixture 5.2: Create `company_profile.txt` fixture file.**
    -   A small, synthetic company profile text file should be created at `services/client-api/tests/data/fixtures/kb_artefacts/company_profile.txt`.
-   [ ] **Fixture 5.3: Implement `sirmaai_kb_file_factory` fixture.**
    -   A pytest fixture/factory should be created to easily upload these files to the `sirmaai_kb_files` table for a given tenant during test setup.

---

This ATDD checklist serves as the "red" phase of the TDD cycle. The next step is to implement the test file and fixtures, making these tests pass ("green" phase).
