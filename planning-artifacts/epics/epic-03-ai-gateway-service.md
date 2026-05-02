# Epic 3: AI Gateway Service

Internal services communicate reliably and safely with KraftData AI via a central, rate-limited, SSE-streaming gateway.

### Story 3.1: Agent Registry & Resilient Calls
As an internal service,
I want to call KraftData only through the gateway,
So that calls are resilient and rate-limited.

**Acceptance Criteria:**
**Given** an outbound call to KraftData
**When** the call is initiated
**Then** it is wrapped in circuit_breaker(retry(http_factory)) and must carry an X-Caller-Service header
**And** it uses a YAML-driven agent registry and handles 503 errors gracefully

### Story 3.2: Streaming Generation
As a frontend,
I want to receive SSE chunks reliably,
So that I can display AI responses in real-time.

**Acceptance Criteria:**
**Given** an AI generation request
**When** the response is streaming
**Then** the run-stream endpoint returns a StreamingResponse with no-cache and no buffering
**And** UsageGate is decremented before streaming begins, with appropriate idle and total timeouts
