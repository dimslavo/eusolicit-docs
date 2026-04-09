# Story 11.7: Logframe Generator & Reporting Template Agent Integrations

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **company admin or bid manager**,
I want to generate a logical framework (logframe) from a project narrative and receive AI-generated periodic report templates for awarded projects,
so that I can produce structured EU grant planning artifacts (logframe, Gantt chart, work packages, deliverables) and pre-filled periodic reports ready for submission.

## Acceptance Criteria

### Logframe Generator Endpoint

1. **AC1** — `POST /api/v1/grants/logframe-generate` accepts a JSON request body with:
   - `project_narrative` (str, required, `min_length=1` — full project description)
   - `target_programme` (str or null, optional — e.g. `"horizon_europe"`, `"digital_europe"`, `"interreg"`)
   Returns HTTP 200 with a `LogframeResponse`. Any authenticated user may call this endpoint — no minimum role beyond a valid JWT with `company_id`.

2. **AC2** — The AI Gateway agent payload sent to logical agent name `logframe-generator` (URL: `{AIGW_BASE_URL}/agents/logframe-generator/run`) must be structured as:
   ```json
   {
     "project_narrative": "<string>",
     "target_programme": "<string or null>",
     "company_id": "<company UUID as string>"
   }
   ```
   The request must include the header `X-Caller-Service: client-api`. Timeout is 30 seconds (configurable via `AIGW_TIMEOUT_SECONDS`). The client-api does NOT retry — the AI Gateway (E04) handles retry and circuit-breaker logic.

3. **AC3** — The agent response is parsed into a `LogframeResponse` with:
   - `logical_framework` (list of `LogicalFrameworkRow`, may be empty): each row has:
     - `level` (str or null — e.g. `"Overall Objective"`, `"Specific Objective"`, `"Expected Result"`, `"Activity"`)
     - `description` (str, required)
     - `indicators` (list[str], may be empty)
     - `means_of_verification` (list[str], may be empty)
     - `assumptions` (list[str], may be empty)
   - `work_packages` (list of `WorkPackage`, may be empty): each has:
     - `wp_number` (int, required)
     - `title` (str, required)
     - `lead` (str or null — lead organisation name)
     - `description` (str or null)
     - `deliverables` (list[str], may be empty)
     - `person_months` (float or null)
   - `gantt_data` (list of `GanttTask` **or null** — null when key is absent from agent response, see AC4): each task has:
     - `task_title` (str, required)
     - `wp_number` (int or null)
     - `start_month` (int, required — 1-indexed from project start)
     - `end_month` (int, required — 1-indexed from project start)
     - `dependencies` (list[str], may be empty — other task_title refs or WP refs)
   - `deliverable_table` (list of `DeliverableItem`, may be empty): each has:
     - `deliverable_number` (str, required — e.g. `"D1.1"`, `"D2.3"`)
     - `title` (str, required)
     - `wp_number` (int or null)
     - `delivery_type` (str or null — e.g. `"report"`, `"software"`, `"dataset"`, `"prototype"`)
     - `dissemination_level` (str or null — `"PU"` public, `"CO"` confidential, `"EU"` restricted)
     - `due_month` (int or null)

4. **AC4** — Logframe graceful degradation for `gantt_data` (E11-R-008): if `gantt_data` key is **absent** from the agent response, the parser MUST return `gantt_data: null` in the response (not raise an error, not return an empty list — explicitly `null` to signal absence to the frontend Gantt chart). Other missing list fields (`logical_framework`, `work_packages`, `deliverable_table`) default to `[]` when absent. A present but empty `gantt_data: []` is a valid result and is NOT the same as null.

5. **AC5** — AI Gateway error handling for logframe-generate: if the AI Gateway times out (> 30 s) or returns HTTP 5xx, the endpoint returns HTTP 503:
   ```json
   {"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"}
   ```
   Raw gateway error is never forwarded. No data is persisted on agent failure.

6. **AC6** — Unauthenticated requests (missing or expired JWT) return HTTP 401. All authenticated roles (`read_only`, `contributor`, `reviewer`, `bid_manager`, `admin`) may call this endpoint without restriction. Endpoint is stateless — no database writes.

### Reporting Template Generator Endpoints

7. **AC7** — `POST /api/v1/grants/reporting-template` accepts a JSON body with:
   - `project_id` (UUID, required — identifies the awarded/won project in the local DB)
   Returns HTTP 200 with a `ReportingTemplateResponse`. Returns 404 if `project_id` not found or not owned by the caller's company (company RLS).

8. **AC8** — Service loads project data from DB before calling agent:
   - Query local DB for the project by `project_id`, enforcing company RLS (`company_id` from JWT — WHERE clause must include `company_id = :company_id`)
   - If not found: raise HTTP 404 with `{"detail": "Project not found"}`
   - Build agent payload from loaded project data and send to `reporting-template-generator` at `{AIGW_BASE_URL}/agents/reporting-template-generator/run`:
   ```json
   {
     "project_id": "<UUID as string>",
     "project_title": "<string or null>",
     "milestones": [...],
     "budget_summary": {...},
     "consortium": [...],
     "company_id": "<company UUID as string>"
   }
   ```
   Header: `X-Caller-Service: client-api`. Timeout: 30 seconds.

9. **AC9** — The agent response is parsed into a `ReportingTemplateResponse` with:
   - `project_id` (str — echoed from request)
   - `project_title` (str or null)
   - `reporting_period` (str or null — e.g. `"Month 1-6"`, `"Q1 2025"`)
   - `report_type` (str or null — e.g. `"periodic"`, `"final"`)
   - `milestones` (list of `ReportingTemplateMilestone`, may be empty): each has `milestone_id` (str or null), `title` (str), `due_date` (str or null), `status` (str or null — `"complete"`, `"in progress"`, `"pending"`), `description` (str or null)
   - `sections` (list of `ReportingTemplateSection`, may be empty — pre-filled narrative sections): each has `section_title` (str), `content` (str or null)
   - `budget_overview` (dict or null — flexible structure from agent)
   - `consortium_summary` (list[dict], may be empty)

10. **AC10** — AI Gateway error handling for reporting-template: timeout or HTTP 5xx → HTTP 503 with standard `AGENT_UNAVAILABLE` body (same as AC5). The 404 for unknown `project_id` (AC8) is not intercepted — it propagates as-is.

11. **AC11** — `POST /api/v1/grants/reporting-template/export` accepts a JSON body with:
    - `project_id` (UUID, required)
    Returns a DOCX file download with:
    - HTTP 200
    - `Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document`
    - `Content-Disposition: attachment; filename="reporting-template-{project_id}.docx"`
    - Non-empty body (valid `.docx` generated by `python-docx`)
    Returns 404 if project not found, 503 if agent unavailable.

12. **AC12** — The DOCX generated by `python-docx` MUST include at minimum:
    - Document heading (level 0): `project_title` or `"Periodic Report"` if null
    - Paragraph: reporting period and report type (if present)
    - Each `sections` entry as a Heading 2 + paragraph with `content`
    - Milestones table (columns: Milestone ID, Title, Due Date, Status) if milestones non-empty
    - Budget overview section (if `budget_overview` not null)
    - Consortium summary section (if `consortium_summary` non-empty)

## Tasks / Subtasks

- [x] **Task 1: Add Story 11.7 schemas to `src/client_api/schemas/grants.py`** (AC: 1, 3, 4, 7, 9, 11)
  - [x] 1.1 After the existing S11.6 schemas, append story banner:
    ```python
    # ---------------------------------------------------------------------------
    # Story 11.7 — Logframe Generator & Reporting Template Agent Integrations
    # ---------------------------------------------------------------------------
    ```
  - [x] 1.2 Define `LogicalFrameworkRow(BaseModel)`:
    - `level: str | None = None`
    - `description: str`
    - `indicators: list[str] = Field(default_factory=list)`
    - `means_of_verification: list[str] = Field(default_factory=list)`
    - `assumptions: list[str] = Field(default_factory=list)`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.3 Define `WorkPackage(BaseModel)`:
    - `wp_number: int`
    - `title: str`
    - `lead: str | None = None`
    - `description: str | None = None`
    - `deliverables: list[str] = Field(default_factory=list)`
    - `person_months: float | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.4 Define `GanttTask(BaseModel)`:
    - `task_title: str`
    - `wp_number: int | None = None`
    - `start_month: int`
    - `end_month: int`
    - `dependencies: list[str] = Field(default_factory=list)`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.5 Define `DeliverableItem(BaseModel)`:
    - `deliverable_number: str`
    - `title: str`
    - `wp_number: int | None = None`
    - `delivery_type: str | None = None`
    - `dissemination_level: str | None = None`
    - `due_month: int | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.6 Define `LogframeRequest(BaseModel)`:
    - `project_narrative: str = Field(..., min_length=1)`
    - `target_programme: str | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.7 Define `LogframeResponse(BaseModel)`:
    - `logical_framework: list[LogicalFrameworkRow]`
    - `work_packages: list[WorkPackage]`
    - `gantt_data: list[GanttTask] | None`  ← `None` = absent from agent; `[]` = agent returned empty
    - `deliverable_table: list[DeliverableItem]`
  - [x] 1.8 Define `ReportingTemplateMilestone(BaseModel)`:
    - `milestone_id: str | None = None`
    - `title: str`
    - `due_date: str | None = None`
    - `status: str | None = None`
    - `description: str | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.9 Define `ReportingTemplateSection(BaseModel)`:
    - `section_title: str`
    - `content: str | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.10 Define `ReportingTemplateRequest(BaseModel)`:
    - `project_id: UUID`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.11 Define `ReportingTemplateExportRequest(BaseModel)`:
    - `project_id: UUID`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.12 Define `ReportingTemplateResponse(BaseModel)`:
    - `project_id: str`
    - `project_title: str | None = None`
    - `reporting_period: str | None = None`
    - `report_type: str | None = None`
    - `milestones: list[ReportingTemplateMilestone] = Field(default_factory=list)`
    - `sections: list[ReportingTemplateSection] = Field(default_factory=list)`
    - `budget_overview: dict | None = None`
    - `consortium_summary: list[dict] = Field(default_factory=list)`
    - `model_config = ConfigDict(extra="ignore")`

- [x] **Task 2: Implement Logframe Generator service in `src/client_api/services/grants_service.py`** (AC: 1–6)
  - [x] 2.1 Add imports at top of existing `grants_service.py` (after S11.6 imports):
    - From `client_api.schemas.grants`: `LogicalFrameworkRow`, `WorkPackage`, `GanttTask`, `DeliverableItem`, `LogframeRequest`, `LogframeResponse`
  - [x] 2.2 Add story banner comment:
    ```python
    # ---------------------------------------------------------------------------
    # Story 11.7 — Logframe Generator
    # ---------------------------------------------------------------------------
    ```
  - [x] 2.3 Define `_parse_logical_framework_row(raw: dict) -> LogicalFrameworkRow | None`:
    - Extract: `level = raw.get("level")`, `description = raw.get("description", "")`, `indicators = raw.get("indicators") or []`, `means_of_verification = raw.get("means_of_verification") or []`, `assumptions = raw.get("assumptions") or []`
    - Return `LogicalFrameworkRow(...)`
    - Wrap in `try/except (KeyError, ValueError, TypeError)`: `log.warning("grants.logframe.parse_lf_row.error", raw=raw)`; return `None`
  - [x] 2.4 Define `_parse_work_package(raw: dict) -> WorkPackage | None`:
    - Extract: `wp_number = int(raw.get("wp_number", 0))`, `title = raw.get("title", "")`, `lead = raw.get("lead")`, `description = raw.get("description")`, `deliverables = raw.get("deliverables") or []`, `person_months = float(raw["person_months"]) if raw.get("person_months") is not None else None`
    - Wrap in `try/except`: log warning, return `None`
  - [x] 2.5 Define `_parse_gantt_task(raw: dict) -> GanttTask | None`:
    - Extract: `task_title = raw.get("task_title", "")`, `wp_number = int(raw["wp_number"]) if raw.get("wp_number") is not None else None`, `start_month = int(raw.get("start_month", 1))`, `end_month = int(raw.get("end_month", 1))`, `dependencies = raw.get("dependencies") or []`
    - Wrap in `try/except`: log warning, return `None`
  - [x] 2.6 Define `_parse_deliverable_item(raw: dict) -> DeliverableItem | None`:
    - Extract all fields with `.get()` and safe defaults
    - Wrap in `try/except`: log warning, return `None`
  - [x] 2.7 Implement `async def generate_logframe(request: LogframeRequest, current_user: CurrentUser, gw_client: AiGatewayClient) -> LogframeResponse`:
    - Build payload:
      ```python
      payload = {
          "project_narrative": request.project_narrative,
          "target_programme": request.target_programme,
          "company_id": str(current_user.company_id),
      }
      ```
    - Call agent with error handling (AC5):
      ```python
      try:
          agent_response = await gw_client.run_agent("logframe-generator", payload)
      except (AiGatewayTimeoutError, AiGatewayUnavailableError):
          log.warning("grants.logframe.agent_error", company_id=str(current_user.company_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      ```
    - Parse `logical_framework`: `raw_lf = agent_response.get("logical_framework", [])` → map through `_parse_logical_framework_row` → filter `None`
    - Parse `work_packages`: same pattern with `_parse_work_package`
    - Parse `gantt_data` **(AC4 — CRITICAL)**:
      ```python
      raw_gantt = agent_response.get("gantt_data")  # NO default — None signals absence
      if raw_gantt is None:
          gantt_data = None  # Key absent → null in response
      else:
          gantt_data = [t for t in (_parse_gantt_task(r) for r in raw_gantt) if t is not None]
      ```
      Do NOT use `agent_response.get("gantt_data", [])` — that would mask absence as empty list.
    - Parse `deliverable_table`: same pattern with `_parse_deliverable_item`
    - Log: `log.info("grants.logframe.completed", company_id=str(current_user.company_id), wp_count=len(work_packages), gantt_present=gantt_data is not None)`
    - Return `LogframeResponse(logical_framework=lf, work_packages=wps, gantt_data=gantt_data, deliverable_table=deliverables)`

- [x] **Task 3: Implement Reporting Template service in `src/client_api/services/grants_service.py`** (AC: 7–12)
  - [x] 3.1 Add imports:
    - From `client_api.schemas.grants`: `ReportingTemplateMilestone`, `ReportingTemplateSection`, `ReportingTemplateRequest`, `ReportingTemplateExportRequest`, `ReportingTemplateResponse`
    - `from docx import Document`
    - `import io`
    - `from uuid import UUID` (if not already imported)
    - `from sqlalchemy.ext.asyncio import AsyncSession` (if not already imported)
  - [x] 3.2 Add story banner:
    ```python
    # ---------------------------------------------------------------------------
    # Story 11.7 — Reporting Template Generator
    # ---------------------------------------------------------------------------
    ```
  - [x] 3.3 Define `async def _load_project_data(session: AsyncSession, project_id: UUID, company_id: UUID) -> dict | None`:
    - **VERIFY the exact table before implementing** — search `src/client_api/models/` for `Proposal`, `Project`, or `AwardedProject` models; also check Alembic migrations for `proposals` or `projects` table in the `client` schema. The table must have `id` and `company_id` columns.
    - Query with company RLS:
      ```python
      result = await session.execute(
          select(ProjectModel).where(
              ProjectModel.id == project_id,
              ProjectModel.company_id == company_id,
          )
      )
      row = result.scalar_one_or_none()
      if row is None:
          return None
      ```
    - Load related data (milestones, budget, consortium) — may require additional queries or joined loads depending on the ORM model
    - Return dict: `{"project_title": row.title, "milestones": [...], "budget_summary": {...}, "consortium": [...]}`
    - Log if not found: `log.warning("grants.reporting_template.project_not_found", project_id=str(project_id))`
  - [x] 3.4 Define `_parse_reporting_template_milestone(raw: dict) -> ReportingTemplateMilestone | None`:
    - Extract all fields with `.get()` and safe defaults
    - `title = raw.get("title", "")` (required field — default to empty string if absent)
    - Wrap in `try/except`: log warning, return `None`
  - [x] 3.5 Define `_parse_reporting_template_section(raw: dict) -> ReportingTemplateSection | None`:
    - `section_title = raw.get("section_title", "")`, `content = raw.get("content")`
    - Wrap in `try/except`: log warning, return `None`
  - [x] 3.6 Define `_parse_reporting_template_response(agent_response: dict, project_id: UUID) -> ReportingTemplateResponse`:
    - Parse all fields with safe defaults:
      ```python
      milestones = [m for m in (_parse_reporting_template_milestone(r) for r in agent_response.get("milestones", [])) if m]
      sections = [s for s in (_parse_reporting_template_section(r) for r in agent_response.get("sections", [])) if s]
      ```
    - Return `ReportingTemplateResponse(project_id=str(project_id), project_title=agent_response.get("project_title"), reporting_period=agent_response.get("reporting_period"), report_type=agent_response.get("report_type"), milestones=milestones, sections=sections, budget_overview=agent_response.get("budget_overview"), consortium_summary=agent_response.get("consortium_summary") or [])`
  - [x] 3.7 Implement `async def generate_reporting_template(request: ReportingTemplateRequest, current_user: CurrentUser, gw_client: AiGatewayClient, session: AsyncSession) -> ReportingTemplateResponse`:
    - Load project: `project_data = await _load_project_data(session, request.project_id, current_user.company_id)`
    - If None: `raise HTTPException(status_code=404, detail="Project not found")`
    - Build agent payload (AC8):
      ```python
      payload = {
          "project_id": str(request.project_id),
          "project_title": project_data.get("project_title"),
          "milestones": project_data.get("milestones", []),
          "budget_summary": project_data.get("budget_summary"),
          "consortium": project_data.get("consortium", []),
          "company_id": str(current_user.company_id),
      }
      ```
    - Call agent with error handling (AC10):
      ```python
      try:
          agent_response = await gw_client.run_agent("reporting-template-generator", payload)
      except (AiGatewayTimeoutError, AiGatewayUnavailableError):
          log.warning("grants.reporting_template.agent_error", project_id=str(request.project_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      ```
    - Parse: `return _parse_reporting_template_response(agent_response, request.project_id)`
    - Log: `log.info("grants.reporting_template.completed", project_id=str(request.project_id), sections_count=len(result.sections))`
  - [x] 3.8 Define `def build_report_docx(template: ReportingTemplateResponse) -> bytes`:
    ```python
    doc = Document()
    doc.add_heading(template.project_title or "Periodic Report", 0)
    if template.reporting_period:
        doc.add_paragraph(f"Reporting Period: {template.reporting_period}")
    if template.report_type:
        doc.add_paragraph(f"Report Type: {template.report_type}")
    for section in template.sections:
        doc.add_heading(section.section_title, 2)
        doc.add_paragraph(section.content or "")
    if template.milestones:
        doc.add_heading("Milestones", 1)
        table = doc.add_table(rows=1, cols=4)
        hdr = table.rows[0].cells
        hdr[0].text, hdr[1].text, hdr[2].text, hdr[3].text = "ID", "Title", "Due Date", "Status"
        for m in template.milestones:
            row = table.add_row().cells
            row[0].text = m.milestone_id or ""
            row[1].text = m.title
            row[2].text = m.due_date or ""
            row[3].text = m.status or ""
    if template.budget_overview:
        doc.add_heading("Budget Overview", 1)
        for k, v in template.budget_overview.items():
            doc.add_paragraph(f"{k}: {v}")
    if template.consortium_summary:
        doc.add_heading("Consortium", 1)
        for partner in template.consortium_summary:
            org = partner.get("organisation", "")
            role = partner.get("role", "")
            country = partner.get("country", "")
            doc.add_paragraph(f"{org} ({role}, {country})")
    buffer = io.BytesIO()
    doc.save(buffer)
    buffer.seek(0)
    return buffer.read()
    ```
  - [x] 3.9 Implement `async def export_reporting_template_docx(request: ReportingTemplateExportRequest, current_user: CurrentUser, gw_client: AiGatewayClient, session: AsyncSession) -> bytes`:
    - Reuse reporting template logic: build a `ReportingTemplateRequest(project_id=request.project_id)` and call `generate_reporting_template` — this handles 404 and 503
    - Call `build_report_docx(template)` and return bytes
    - Note: `doc.save()` is synchronous but fast for typical reports; no Celery offload needed for MVP (E11-R-011 deferred)

- [x] **Task 4: Add 3 endpoints to `src/client_api/api/v1/grants.py`** (AC: 1, 5, 6, 7, 10, 11)
  - [x] 4.1 Add to imports in existing `grants.py`:
    - From `client_api.schemas.grants`: `LogframeRequest`, `LogframeResponse`, `ReportingTemplateRequest`, `ReportingTemplateExportRequest`, `ReportingTemplateResponse`
    - `from fastapi import Response` (if not already imported)
    - `from sqlalchemy.ext.asyncio import AsyncSession`
    - `get_async_session` from the dependency module — **check where other DB-using endpoints import it from** (e.g. look at ESPD CRUD router from S11.02 for the correct import path)
  - [x] 4.2 Implement `POST /logframe-generate` (stateless — no DB session needed):
    ```python
    @router.post("/logframe-generate", response_model=LogframeResponse)
    async def logframe_generate(
        body: LogframeRequest,
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
    ) -> LogframeResponse | JSONResponse:
        """Generate logframe, work packages, Gantt data, and deliverables.

        Stateless — no DB read/write. Requires authentication (all roles, AC6).
        AC4: gantt_data is null (not []) when agent response omits the key.
        AC5: AI Gateway errors → 503 AGENT_UNAVAILABLE.
        """
        try:
            return await grants_service.generate_logframe(body, current_user, gw_client)
        except HTTPException as exc:
            if exc.status_code == 503 and isinstance(exc.detail, dict):
                return JSONResponse(status_code=503, content=exc.detail)
            raise
    ```
  - [x] 4.3 Implement `POST /reporting-template/export` **(define BEFORE /reporting-template to avoid any ambiguous routing)**:
    ```python
    @router.post("/reporting-template/export")
    async def export_reporting_template(
        body: ReportingTemplateExportRequest,
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
        session: Annotated[AsyncSession, Depends(get_async_session)],
    ) -> Response | JSONResponse:
        """Generate and download a DOCX periodic report for an awarded project.

        Loads project from DB (company RLS). Returns 404 if not found.
        AC11/AC12: DOCX generated via python-docx. 503 if agent unavailable.
        Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document
        """
        try:
            docx_bytes = await grants_service.export_reporting_template_docx(
                body, current_user, gw_client, session
            )
            return Response(
                content=docx_bytes,
                media_type="application/vnd.openxmlformats-officedocument.wordprocessingml.document",
                headers={
                    "Content-Disposition": f'attachment; filename="reporting-template-{body.project_id}.docx"'
                },
            )
        except HTTPException as exc:
            if exc.status_code == 503 and isinstance(exc.detail, dict):
                return JSONResponse(status_code=503, content=exc.detail)
            raise  # 404 and others propagate as-is
    ```
  - [x] 4.4 Implement `POST /reporting-template`:
    ```python
    @router.post("/reporting-template", response_model=ReportingTemplateResponse)
    async def reporting_template(
        body: ReportingTemplateRequest,
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
        session: Annotated[AsyncSession, Depends(get_async_session)],
    ) -> ReportingTemplateResponse | JSONResponse:
        """Generate pre-filled periodic report template for an awarded project.

        Loads project data from DB (company RLS). Returns 404 if not found.
        AC10: AI Gateway errors → 503 AGENT_UNAVAILABLE.
        """
        try:
            return await grants_service.generate_reporting_template(
                body, current_user, gw_client, session
            )
        except HTTPException as exc:
            if exc.status_code == 503 and isinstance(exc.detail, dict):
                return JSONResponse(status_code=503, content=exc.detail)
            raise
    ```
  - [x] 4.5 **No changes to `main.py`** — the `/grants` router was already registered in S11.04.

- [x] **Task 5: Write tests in `tests/api/test_logframe_generator.py`** (E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-004, E11-P1-005, E11-R-008)
  - [x] 5.1 Module docstring listing epic test IDs covered + link to story file
  - [x] 5.2 Guard: `respx = pytest.importorskip("respx", reason="respx required for AI Gateway mocking")`
  - [x] 5.3 Module-level constants:
    ```python
    AIGW_TEST_BASE_URL = "http://test-aigw.local:8000"
    AIGW_LOGFRAME_URL = f"{AIGW_TEST_BASE_URL}/agents/logframe-generator/run"
    ENDPOINT = "/api/v1/grants/logframe-generate"
    AGENT_UNAVAILABLE_CODE = "AGENT_UNAVAILABLE"
    AGENT_UNAVAILABLE_MESSAGE = "AI features are temporarily unavailable. Please try again."
    ```
  - [x] 5.4 Define fixture responses:
    ```python
    # MOCK_FULL_LOGFRAME_RESPONSE: all 4 fields populated; gantt_data key present
    MOCK_FULL_LOGFRAME_RESPONSE = {
        "logical_framework": [
            {
                "level": "Overall Objective",
                "description": "Improve digital health outcomes in rural EU communities",
                "indicators": ["30% improvement in health access by Month 24"],
                "means_of_verification": ["WHO health access reports", "Project KPI dashboard"],
                "assumptions": ["Partner institutions remain operational"],
            },
            {
                "level": "Specific Objective",
                "description": "Deploy AI triage system in 5 rural health clinics",
                "indicators": ["5 clinics active", "1000 patients/month by Month 18"],
                "means_of_verification": ["Clinic operational reports"],
                "assumptions": ["Regulatory approval granted"],
            },
        ],
        "work_packages": [
            {
                "wp_number": 1,
                "title": "Project Management and Coordination",
                "lead": "Sofia Tech University",
                "description": "Overall project coordination, reporting, and dissemination",
                "deliverables": ["D1.1 Project Management Plan", "D1.2 Progress Reports"],
                "person_months": 12.0,
            },
            {
                "wp_number": 2,
                "title": "AI Triage System Development",
                "lead": "Berlin IoT GmbH",
                "description": "Development and validation of AI triage algorithm",
                "deliverables": ["D2.1 System Specification", "D2.2 Prototype", "D2.3 Validated System"],
                "person_months": 24.0,
            },
        ],
        "gantt_data": [
            {"task_title": "WP1: Project Kick-off", "wp_number": 1, "start_month": 1, "end_month": 2, "dependencies": []},
            {"task_title": "WP2: AI Development Phase 1", "wp_number": 2, "start_month": 2, "end_month": 12, "dependencies": ["WP1: Project Kick-off"]},
        ],
        "deliverable_table": [
            {"deliverable_number": "D1.1", "title": "Project Management Plan", "wp_number": 1, "delivery_type": "report", "dissemination_level": "CO", "due_month": 2},
            {"deliverable_number": "D2.1", "title": "AI System Specification", "wp_number": 2, "delivery_type": "report", "dissemination_level": "PU", "due_month": 4},
        ],
    }

    # MOCK_PARTIAL_LOGFRAME_RESPONSE: gantt_data key intentionally ABSENT
    # (tests graceful degradation AC4 / E11-P1-005 / E11-R-008)
    MOCK_PARTIAL_LOGFRAME_RESPONSE = {
        "logical_framework": [
            {"level": "Overall Objective", "description": "Digitalise SME supply chains",
             "indicators": ["500 SMEs onboarded by Month 18"], "means_of_verification": ["Platform logs"], "assumptions": []},
        ],
        "work_packages": [
            {"wp_number": 1, "title": "Platform Development", "lead": "Tech Lead GmbH",
             "description": "Core development", "deliverables": ["D1.1 Platform MVP"], "person_months": 18.0},
        ],
        # "gantt_data" key intentionally ABSENT — parser must return gantt_data: null (not [])
        "deliverable_table": [
            {"deliverable_number": "D1.1", "title": "Platform MVP", "wp_number": 1,
             "delivery_type": "software", "dissemination_level": "PU", "due_month": 12},
        ],
    }
    ```
  - [x] 5.5 Session-scoped `aigw_env_setup` fixture — same pattern as `test_consortium_finder.py`:
    - Set `CLIENT_API_AIGW_BASE_URL = AIGW_TEST_BASE_URL` in `os.environ`
    - Clear `get_settings.cache_clear()` and `get_ai_gateway_client.cache_clear()` LRU caches
    - Restore on teardown
  - [x] 5.6 Function-scoped `logframe_client_and_session` fixture — same pattern as `consortium_client_and_session` in `test_consortium_finder.py`:
    - Create `httpx.AsyncClient(app=app, base_url="http://test")`
    - Register company user via `POST /api/v1/auth/register`
    - Verify email: `UPDATE client.users SET email_verified = TRUE WHERE id = :id`
    - Login via `POST /api/v1/auth/login` → extract `access_token`
    - Yield `(client, session, access_token)`
    - Teardown: close client, rollback session
  - [x] 5.7 **`TestAC1AC2HappyPath`** — Smoke + payload verification (E11-P0-009):
    - `test_logframe_generate_returns_200_with_complete_structure` — mock `AIGW_LOGFRAME_URL` with `MOCK_FULL_LOGFRAME_RESPONSE`; POST `{"project_narrative": "AI health tech for rural clinics", "target_programme": "horizon_europe"}`; assert 200; `logical_framework` non-empty; `work_packages` non-empty; `gantt_data` is not None; `deliverable_table` non-empty
    - `test_logframe_agent_payload_has_required_fields` — intercept agent call with `respx.MockRouter`; parse `request.content` JSON; assert `project_narrative` (str), `target_programme` (str), `company_id` (str) all present
    - `test_logframe_x_caller_service_header_sent` — assert captured headers contain `x-caller-service: client-api`
    - `test_logframe_null_target_programme_forwarded` — POST without `target_programme`; assert captured payload `target_programme` is null; assert 200
  - [x] 5.8 **`TestAC3AC4ResponseParsing`** — Parsing + graceful degradation (E11-P1-004, E11-P1-005, E11-R-008):
    - `test_all_four_logframe_fields_present_in_response` — mock `MOCK_FULL_LOGFRAME_RESPONSE`; POST; assert response contains `logical_framework`, `work_packages`, `gantt_data`, `deliverable_table` all present (E11-P1-004)
    - `test_logical_framework_rows_have_required_fields` — assert each row has `description` (str), `indicators` (list), `means_of_verification` (list)
    - `test_work_packages_have_required_fields` — assert each WP has `wp_number` (int), `title` (str), `deliverables` (list), `person_months`
    - `test_gantt_tasks_have_required_fields` — assert each task has `task_title` (str), `start_month` (int), `end_month` (int)
    - `test_deliverables_have_required_fields` — assert each has `deliverable_number` (str), `title` (str)
    - `test_gantt_data_absent_returns_null_not_error` — mock `MOCK_PARTIAL_LOGFRAME_RESPONSE` (no `gantt_data` key); POST; assert 200; `response.json()["gantt_data"] is None` — **NOT an empty list** (AC4, E11-P1-005, E11-R-008)
    - `test_gantt_data_present_empty_list_not_null` — mock response with `"gantt_data": []` (key present, value empty); POST; assert 200; `response.json()["gantt_data"] == []` — not null
  - [x] 5.9 **`TestAC5AgentErrorHandling`** (E11-P0-008, E11-P0-010):
    - `test_gateway_timeout_returns_503_agent_unavailable` — mock `AIGW_LOGFRAME_URL` to raise `httpx.ReadTimeout`; POST; assert 503; `body["code"] == AGENT_UNAVAILABLE_CODE`; `body["message"] == AGENT_UNAVAILABLE_MESSAGE` (E11-P0-010)
    - `test_gateway_500_returns_503_not_500` — mock to return HTTP 500; POST; assert 503; `body["code"] == AGENT_UNAVAILABLE_CODE` (E11-P0-008)
    - `test_gateway_503_returns_503_with_standard_body` — mock to return HTTP 503; POST; assert 503; standard error body
  - [x] 5.10 **`TestAC6Authorization`**:
    - `test_unauthenticated_returns_401` — POST without `Authorization` header; assert 401
  - [x] 5.11 **Input validation**:
    - `test_missing_project_narrative_returns_422` — POST without `project_narrative`; assert 422
    - `test_empty_project_narrative_returns_422` — POST with `project_narrative=""`; assert 422 (`min_length=1`)

- [x] **Task 6: Write tests in `tests/api/test_reporting_template.py`** (E11-P0-008, E11-P0-009, E11-P1-006, E11-P1-007, E11-P2-003)
  - [x] 6.1 Module docstring + `respx = pytest.importorskip("respx", ...)`
  - [x] 6.2 Module-level constants:
    ```python
    AIGW_TEST_BASE_URL = "http://test-aigw.local:8000"
    AIGW_REPORTING_URL = f"{AIGW_TEST_BASE_URL}/agents/reporting-template-generator/run"
    ENDPOINT = "/api/v1/grants/reporting-template"
    EXPORT_ENDPOINT = "/api/v1/grants/reporting-template/export"
    AGENT_UNAVAILABLE_CODE = "AGENT_UNAVAILABLE"
    AGENT_UNAVAILABLE_MESSAGE = "AI features are temporarily unavailable. Please try again."
    DOCX_MIME_TYPE = "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
    ```
  - [x] 6.3 Define fixture response:
    ```python
    MOCK_REPORTING_TEMPLATE_RESPONSE = {
        "project_title": "AI Health Tech for Rural Communities",
        "reporting_period": "Month 1-6",
        "report_type": "periodic",
        "milestones": [
            {"milestone_id": "MS1", "title": "Project Kick-off Workshop", "due_date": "Month 2", "status": "complete", "description": "Initial consortium meeting"},
            {"milestone_id": "MS2", "title": "System Prototype Available", "due_date": "Month 6", "status": "in progress", "description": "First working prototype"},
        ],
        "sections": [
            {"section_title": "Executive Summary", "content": "This periodic report covers Month 1-6 activities of the AI Health Tech project."},
            {"section_title": "Work Progress", "content": "WP1 and WP2 are on schedule. D1.1 submitted successfully."},
        ],
        "budget_overview": {"total_budget": 500000.0, "spent_to_date": 125000.0, "remaining": 375000.0, "burn_rate_percent": 25.0},
        "consortium_summary": [
            {"organisation": "Sofia Tech University", "role": "coordinator", "country": "BG"},
            {"organisation": "Berlin IoT GmbH", "role": "partner", "country": "DE"},
        ],
    }
    ```
  - [x] 6.4 Session-scoped `aigw_env_setup` fixture — same pattern as other test files
  - [x] 6.5 Function-scoped `reporting_client_and_session` fixture:
    - Register company user → verify email → login → extract `access_token` and `company_id`
    - **Seed a test project in DB**: insert a row into the appropriate project/proposals table with the test `company_id`. **Verify the exact table name from the codebase before writing seed SQL** (check `src/client_api/models/` for Proposal/Project model)
    - Yield `(client, session, access_token, project_id_str)`
    - Teardown: rollback session, close client
  - [x] 6.6 **`TestAC7AC8HappyPath`** (E11-P0-009, E11-P1-006):
    - `test_reporting_template_returns_200_with_json_structure` — seed project; mock `AIGW_REPORTING_URL` with `MOCK_REPORTING_TEMPLATE_RESPONSE`; POST `{"project_id": project_id_str}`; assert 200; response has `project_id` (str), `milestones` (list), `sections` (list) (E11-P1-006)
    - `test_reporting_template_milestones_have_required_fields` — assert each milestone has `title` (str)
    - `test_reporting_template_sections_have_required_fields` — assert each section has `section_title` (str)
    - `test_reporting_template_agent_payload_includes_project_data` — intercept agent call; assert payload contains `project_id`, `company_id`, `milestones`, `budget_summary`, `project_title`
    - `test_reporting_template_x_caller_service_header_sent` — assert captured headers `x-caller-service: client-api`
  - [x] 6.7 **`TestAC11DOCX`** (E11-P1-007, E11-P2-003):
    - `test_export_returns_docx_content_type` — seed project; mock AIGW; POST to `EXPORT_ENDPOINT` with `{"project_id": project_id_str}`; assert 200; `response.headers["content-type"].startswith(DOCX_MIME_TYPE)` (E11-P1-007)
    - `test_export_has_content_disposition_attachment` — assert `"content-disposition"` header contains `"attachment"` and filename ending in `.docx`
    - `test_export_body_is_non_empty` — assert `len(response.content) > 0`
    - `test_export_is_valid_word_document` — parse `response.content` with `Document(io.BytesIO(response.content))` from `python-docx` without exception; assert `len(doc.paragraphs) > 0` (E11-P2-003)
  - [x] 6.8 **`TestAC10AgentErrorHandling`** (E11-P0-008, E11-P0-010):
    - `test_reporting_template_gateway_timeout_returns_503` — seed project; mock AIGW to raise `httpx.ReadTimeout`; POST; assert 503; `body["code"] == AGENT_UNAVAILABLE_CODE` (E11-P0-010)
    - `test_reporting_template_gateway_500_returns_503` — mock HTTP 500; POST; assert 503; `body["code"] == AGENT_UNAVAILABLE_CODE` (E11-P0-008)
    - `test_export_gateway_timeout_returns_503` — mock AIGW timeout on export; POST to `EXPORT_ENDPOINT`; assert 503
  - [x] 6.9 **`TestAC8NotFound`**:
    - `test_unknown_project_id_returns_404` — POST with random UUID not in DB; assert 404; `response.json()["detail"] == "Project not found"`
    - `test_export_unknown_project_id_returns_404` — same for `EXPORT_ENDPOINT`
  - [x] 6.10 **`TestAuthorization`**:
    - `test_unauthenticated_returns_401` — POST to `ENDPOINT` without auth; assert 401
    - `test_unauthenticated_export_returns_401` — POST to `EXPORT_ENDPOINT` without auth; assert 401
  - [x] 6.11 **Input validation**:
    - `test_missing_project_id_returns_422` — POST `{}` to `ENDPOINT`; assert 422
    - `test_invalid_project_id_format_returns_422` — POST `{"project_id": "not-a-uuid"}`; assert 422

## Dev Notes

### Architecture Context

This story adds 3 new endpoints and 2 new agent types to the established `/grants` resource group. The files to edit are `grants.py` (schemas), `grants_service.py` (service), `api/v1/grants.py` (router). **`main.py` requires NO changes** — the `/grants` router was registered in S11.04.

### Critical Difference from S11.04–S11.06: DB Session Required for Reporting Template

S11.07 introduces the **first DB-reading endpoints** in the grants group. Unlike S11.04/05/06 (fully stateless):
- `POST /reporting-template` and `POST /reporting-template/export` inject `AsyncSession`
- `POST /logframe-generate` remains **stateless** (no `AsyncSession`)
- The `get_async_session` dependency must be imported in `api/v1/grants.py` — check where S11.02 (ESPD CRUD) imports it from for the correct path

### CRITICAL: Verify DB Table Before Implementing `_load_project_data`

Before writing any SQL/ORM query for the reporting template service:
1. Search `src/client_api/models/` for `Proposal`, `Project`, or `AwardedProject` SQLAlchemy model
2. Check Alembic migrations for `proposals` or `projects` table in the `client` schema
3. The E07 implementation stories (if created) may contain the schema
4. The table MUST have `id` (UUID PK) and `company_id` (UUID FK) columns for RLS enforcement

### CRITICAL: Verify python-docx Installed

Before implementing the DOCX export: confirm `python-docx` is in `requirements.txt` or `pyproject.toml` of `client-api`. If not present, add it. The package is named `python-docx` on PyPI but imported as `from docx import Document`.

### File Modification Checklist

| File | Action |
|------|--------|
| `src/client_api/schemas/grants.py` | **Edit** — append 12 new schema classes after S11.6 schemas |
| `src/client_api/services/grants_service.py` | **Edit** — append logframe service (6 functions) + reporting template service (6 functions) |
| `src/client_api/api/v1/grants.py` | **Edit** — add 3 new routes + imports (including `AsyncSession`, `get_async_session`) |
| `src/client_api/main.py` | **No change** |
| `tests/api/test_logframe_generator.py` | **Create new** |
| `tests/api/test_reporting_template.py` | **Create new** |

### Key Patterns from S11.06 to Follow Exactly

1. **HTTP 503 shape**: catch `AiGatewayTimeoutError` and `AiGatewayUnavailableError`; raise `HTTPException(503, detail={"message": "...", "code": "AGENT_UNAVAILABLE"})`. Router converts to `JSONResponse` for flat JSON body.

2. **UUID serialization**: always `str(current_user.company_id)` and `str(request.project_id)` — httpx JSON does not handle `uuid.UUID`.

3. **Parser defensive coding**: all parse helpers use `.get()` with safe defaults + `try/except (KeyError, ValueError, TypeError)` + `log.warning(...)` on bad shape.

4. **`gantt_data` null vs empty list** (CRITICAL — AC4, E11-R-008):
   - Use `agent_response.get("gantt_data")` with NO default argument
   - `None` returned by `.get()` = key absent in agent response → set `gantt_data = None`
   - `[]` returned = key present but empty → parse to empty list `[]`
   - These are semantically different: `null` tells the frontend not to render the Gantt chart; `[]` tells it the chart is empty but present

5. **DOCX export response**: return `fastapi.Response` (not `StreamingResponse`) with `content=bytes`, `media_type=DOCX_MIME_TYPE`, `headers={"Content-Disposition": ...}`.

6. **Route ordering in `grants.py`**: define `POST /reporting-template/export` BEFORE `POST /reporting-template` to prevent any ambiguous matching in FastAPI's routing logic.

### Agent URL Reference

| Agent | Logical Name | URL Pattern |
|-------|-------------|-------------|
| Logframe Generator | `logframe-generator` | `{AIGW_BASE_URL}/agents/logframe-generator/run` |
| Reporting Template Generator | `reporting-template-generator` | `{AIGW_BASE_URL}/agents/reporting-template-generator/run` |

### DOCX MIME Type (exact string — no abbreviation)

```
application/vnd.openxmlformats-officedocument.wordprocessingml.document
```

### Epic Test ID Coverage

| Epic Test ID | Priority | Description | Test File | Test Methods |
|-------------|----------|-------------|-----------|--------------|
| **E11-P0-008** | P0 | Agent error body shape on 503/5xx — logframe | `test_logframe_generator.py` | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` |
| **E11-P0-008** | P0 | Agent error body shape — reporting-template | `test_reporting_template.py` | `test_reporting_template_gateway_500_returns_503`, `test_export_gateway_timeout_returns_503` |
| **E11-P0-009** | P0 | Deterministic fixture responses in CI (smoke gate) | `test_logframe_generator.py` | `test_logframe_generate_returns_200_with_complete_structure` |
| **E11-P0-009** | P0 | Deterministic fixture responses — reporting-template | `test_reporting_template.py` | `test_reporting_template_returns_200_with_json_structure` |
| **E11-P0-010** | P0 | 30s timeout enforced — logframe | `test_logframe_generator.py` | `test_gateway_timeout_returns_503_agent_unavailable` |
| **E11-P0-010** | P0 | 30s timeout enforced — reporting-template | `test_reporting_template.py` | `test_reporting_template_gateway_timeout_returns_503` |
| **E11-P1-004** | P1 | All 4 logframe output fields present | `test_logframe_generator.py` | `test_all_four_logframe_fields_present_in_response` |
| **E11-P1-005** | P1 | `gantt_data` absent → null, not 500 | `test_logframe_generator.py` | `test_gantt_data_absent_returns_null_not_error` |
| **E11-P1-006** | P1 | Project data loaded from DB; template returned as JSON | `test_reporting_template.py` | `test_reporting_template_returns_200_with_json_structure` |
| **E11-P1-007** | P1 | DOCX export content-type + download succeeds | `test_reporting_template.py` | `test_export_returns_docx_content_type` |
| **E11-P2-003** | P2 | DOCX export is a valid Word document | `test_reporting_template.py` | `test_export_is_valid_word_document` |

### Risk Coverage

- **E11-R-001** (AI Gateway error handling): AC5, AC10 + `TestAC5AgentErrorHandling`, `TestAC10AgentErrorHandling` — all agent error paths → 503
- **E11-R-008** (Logframe parser field completeness): AC4 + `test_gantt_data_absent_returns_null_not_error` + `test_gantt_data_present_empty_list_not_null` — null vs `[]` distinction enforced
- **E11-R-011** (DOCX performance): noted as low risk; in-memory `io.BytesIO` generation is MVP-appropriate; Celery offload deferred

### `LogframeResponse` JSON Shape Example

```json
{
  "logical_framework": [
    {
      "level": "Overall Objective",
      "description": "Improve digital health outcomes in rural EU communities",
      "indicators": ["30% improvement in health access by Month 24"],
      "means_of_verification": ["WHO health access reports"],
      "assumptions": ["Partner institutions remain operational"]
    }
  ],
  "work_packages": [
    {
      "wp_number": 1,
      "title": "Project Management",
      "lead": "Sofia Tech University",
      "description": "Overall coordination",
      "deliverables": ["D1.1 Management Plan"],
      "person_months": 12.0
    }
  ],
  "gantt_data": [
    {"task_title": "WP1: Project Kick-off", "wp_number": 1, "start_month": 1, "end_month": 2, "dependencies": []}
  ],
  "deliverable_table": [
    {"deliverable_number": "D1.1", "title": "Project Management Plan", "wp_number": 1, "delivery_type": "report", "dissemination_level": "CO", "due_month": 2}
  ]
}
```

When `gantt_data` absent from agent response:
```json
{"logical_framework": [...], "work_packages": [...], "gantt_data": null, "deliverable_table": [...]}
```

### `ReportingTemplateResponse` JSON Shape Example

```json
{
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "project_title": "AI Health Tech for Rural Communities",
  "reporting_period": "Month 1-6",
  "report_type": "periodic",
  "milestones": [
    {"milestone_id": "MS1", "title": "Project Kick-off Workshop", "due_date": "Month 2", "status": "complete", "description": "Initial meeting"}
  ],
  "sections": [
    {"section_title": "Executive Summary", "content": "This report covers Month 1-6 activities."},
    {"section_title": "Work Progress", "content": "WP1 and WP2 on schedule."}
  ],
  "budget_overview": {"total_budget": 500000.0, "spent_to_date": 125000.0, "remaining": 375000.0, "burn_rate_percent": 25.0},
  "consortium_summary": [
    {"organisation": "Sofia Tech University", "role": "coordinator", "country": "BG"}
  ]
}
```

### Project Structure Notes

- All new schemas appended to existing `src/client_api/schemas/grants.py` — do NOT create a new schema file
- All new service functions appended to existing `src/client_api/services/grants_service.py` — do NOT create a new service file
- All new routes added to existing `src/client_api/api/v1/grants.py` router — do NOT create a new router
- Test files: `tests/api/test_logframe_generator.py` (new) and `tests/api/test_reporting_template.py` (new)
- Mirror the module-level organization pattern from `tests/api/test_consortium_finder.py` exactly

### Test Count Summary

| File | Tests | Assertions |
|------|-------|------------|
| `test_logframe_generator.py` | ~15 | ~25 |
| `test_reporting_template.py` | ~14 | ~22 |
| **Total** | **~29** | **~47** |

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E11-grants-compliance.md#S11.07]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-11.md#P0, P1, P2]
- [Source: eusolicit-docs/implementation-artifacts/11-6-consortium-finder-agent-integration.md — patterns to mirror]
- [Source: eusolicit-docs/implementation-artifacts/11-5-budget-builder-agent-integration.md — DB session pattern reference]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

- Added `response_model=None` to `POST /reporting-template/export` FastAPI decorator to avoid FastAPIError when return type is `Response | JSONResponse`.
- Used `migration_role` (not `client_api_role`) to run Alembic migration 010 — `client_api_role` lacks `CREATE` on the `client` schema.
- Created `Proposal` ORM model and migration 010 for `client.proposals` table (no existing awarded-project model found in `src/client_api/models/`).

### Completion Notes List

- ✅ Task 1: Added 12 new Pydantic schema classes to `src/client_api/schemas/grants.py` (LogicalFrameworkRow, WorkPackage, GanttTask, DeliverableItem, LogframeRequest, LogframeResponse, ReportingTemplateMilestone, ReportingTemplateSection, ReportingTemplateRequest, ReportingTemplateExportRequest, ReportingTemplateResponse). All use `ConfigDict(extra="ignore")`.
- ✅ Task 2: Implemented Logframe Generator in `grants_service.py` — 5 helper parsers + `generate_logframe`. AC4 implemented correctly: `agent_response.get("gantt_data")` with no default — `None` = absent (→ null), `[]` = present-but-empty.
- ✅ Task 3: Implemented Reporting Template service in `grants_service.py` — `_load_project_data` (company RLS), 4 parsers, `generate_reporting_template`, `build_report_docx` (python-docx), `export_reporting_template_docx`. Added `python-docx>=1.1` to `pyproject.toml`.
- ✅ Bonus: Created `Proposal` ORM model + Alembic migration `010_proposals.py` (client.proposals table with id, company_id, title, milestones, budget_summary, consortium).
- ✅ Task 4: Added 3 new endpoints to `api/v1/grants.py` — `POST /logframe-generate` (stateless), `POST /reporting-template/export` (registered before /reporting-template to avoid routing ambiguity, `response_model=None`), `POST /reporting-template` (DB session injected). No changes to `main.py`.
- ✅ All 35 ATDD tests pass: 17/17 `test_logframe_generator.py` + 18/18 `test_reporting_template.py`.
- ✅ Zero regressions: 69/69 previously-passing grants tests (S11.04/05/06) still pass.

### File List

- `eusolicit-app/services/client-api/pyproject.toml`
- `eusolicit-app/services/client-api/src/client_api/schemas/grants.py`
- `eusolicit-app/services/client-api/src/client_api/services/grants_service.py`
- `eusolicit-app/services/client-api/src/client_api/api/v1/grants.py`
- `eusolicit-app/services/client-api/src/client_api/models/proposal.py`
- `eusolicit-app/services/client-api/src/client_api/models/__init__.py`
- `eusolicit-app/services/client-api/alembic/versions/010_proposals.py`

## Senior Developer Review

**REVIEW: Approve**

**Date:** 2026-04-09
**Reviewer:** Claude Opus 4.6 (BMad Adversarial Code Review — 3-layer)

### Verdict

All 12 Acceptance Criteria are correctly implemented and tested. 35/35 ATDD tests pass. Zero regressions across 69 prior grants tests (S11.04/05/06). Code follows established project patterns. Approved with advisory hardening notes below.

### AC Compliance Matrix

| AC | Status | Notes |
|----|--------|-------|
| AC1 | PASS | Endpoint, request schema (min_length=1), auth gate all correct |
| AC2 | PASS | Agent payload structure verified; X-Caller-Service header sent by AiGatewayClient |
| AC3 | PASS | LogframeResponse schema matches spec exactly; all 4 output fields present |
| AC4 | PASS | **Critical path verified:** `agent_response.get("gantt_data")` with no default; `None` = absent, `[]` = empty. Two dedicated tests enforce the distinction |
| AC5 | PASS | AiGatewayTimeoutError/UnavailableError caught; 503 body shape exact (`message` + `code`) |
| AC6 | PASS | `Depends(get_current_user)` enforces 401; no role restriction |
| AC7 | PASS | Endpoint accepts UUID project_id; returns ReportingTemplateResponse or 404 |
| AC8 | PASS | `_load_project_data` enforces company RLS (`WHERE company_id = :company_id`); payload built from DB data |
| AC9 | PASS | ReportingTemplateResponse schema matches spec; parsed via dedicated helpers |
| AC10 | PASS | Same 503 error handling as AC5 |
| AC11 | PASS | DOCX Content-Type and Content-Disposition correct; route ordering correct (export before template) |
| AC12 | PASS | build_report_docx includes heading, period, type, sections as H2, milestones table, budget overview, consortium |

### Epic Test Coverage

All required epic test IDs verified present: E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-004, E11-P1-005, E11-P1-006, E11-P1-007, E11-P2-003.

### Positive Observations

- **AC4 implementation is exemplary** — the comment `# noqa: SIM910 — intentional: no default` documents the design decision clearly.
- **Parser defensive coding** is thorough: all 4 parse helpers include `isinstance(raw, dict)` guard + `try/except (KeyError, ValueError, TypeError)` + structured logging.
- **Route ordering** (`/reporting-template/export` before `/reporting-template`) is correct and documented.
- **python-docx** properly declared in `pyproject.toml`.
- **Proposal model** and migration 010 correctly add `client.proposals` with indexed `company_id` for RLS.

### Advisory Findings (Non-Blocking)

These are hardening improvements for a future iteration. They follow the same pattern gap present in S11.04-S11.06 and are not regressions.

| # | Severity | Finding | Location |
|---|----------|---------|----------|
| A1 | MEDIUM | Missing `isinstance(raw_gantt, list)` guard before iterating at line 745. If agent returns a non-list for `gantt_data`, individual `_parse_gantt_task` calls safely filter garbage (each has `isinstance(raw, dict)` check), producing `[]` instead of the correct `None`. Silent semantic degradation. | `grants_service.py:745` |
| A2 | LOW | Same pattern for `raw_lf`, `raw_wp`, `raw_del` — no `isinstance(..., list)` before iterating. Safe default `[]` prevents crash but non-list agent response would silently degrade. Pre-existing in S11.06. | `grants_service.py:734-749` |
| A3 | LOW | `run_agent()` in `ai_gateway_client.py` returns `response.json()` without validating it's a `dict`. If agent returns JSON array/scalar, callers crash on `.get()`. Pre-existing pattern across all stories. | `ai_gateway_client.py:80` |
| A4 | LOW | `build_report_docx` calls `budget_overview.items()` directly. Protected upstream by Pydantic `dict | None` type validation, but a direct `isinstance` guard would be more resilient. | `grants_service.py:950` |
| A5 | LOW | Migration 010: `created_at`/`updated_at` columns are `nullable=True` despite having `server_default=sa.func.now()`. Functionally harmless but semantically inconsistent. | `010_proposals.py` |

### Recommendation

**Approve as-is.** Advisory findings A1-A5 are suitable for a cross-cutting hardening ticket (covering S11.04-S11.07 agent response validation) rather than blocking this story.

## Change Log

- 2026-04-09: Story 11.7 implemented — Logframe Generator (POST /logframe-generate) and Reporting Template Generator (POST /reporting-template, POST /reporting-template/export). Added Proposal model + migration 010. All 35 ATDD tests pass.
- 2026-04-09: Senior Developer Review — APPROVED. All 12 ACs pass. 5 advisory hardening notes (non-blocking).
